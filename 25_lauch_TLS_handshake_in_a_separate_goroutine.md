- etcd/client/pkg/transport/listener_tls.go
- tlsListener wrapped the tls.listener struct. the tls.listener can accept the connection but will do handshake for the first Read or Write, not when the connection is initiated.
- the tlsListener will launch a control loop that
    - accept the incoming connection request
    - launch a separate goroutine to do tls handshake
    - push the new connection to a channel
- the tlsListener then pick the connection in the channel and return it
- how it works 
    - a tlsListener is listening on an addr of the server
    - the incoming request comming on that server.
        - the tlsListener.Accept is called and block waiting on the connc channel
    - the Accept method of tls.listener waiting in acceptLoop will advance independently
        - if multiples listeners are wrapped layer by layer, their accept method can advance independently!
        - normally, these Accept methods are chained together, but for tlsListener, it is separated from the inner Accept!
    - the acceptLoop lauch a separate goroutine to handle the tls handshake for the connection
        - whenever the handshake done, the goroutine send the initiated connection to connc channel
    - the blocking Accept of tlsListener picks up the new conn and return

* it seems unnecessary do the handshake for each connection. we can just return the connection and then the first read/write will trigger the tls handshake! but is it faster?

```go
type tlsListener struct {
	net.Listener
	connc            chan net.Conn
	donec            chan struct{}
	err              error
	handshakeFailure func(*tls.Conn, error)
	check            tlsCheckFunc
}

func newTLSListener(l net.Listener, tlsinfo *TLSInfo, check tlsCheckFunc) (net.Listener, error) {
    ...

	tlsl := &tlsListener{
		Listener:         tls.NewListener(l, tlscfg),
		connc:            make(chan net.Conn),
		donec:            make(chan struct{}),
		handshakeFailure: hf,
		check:            check,
	}
	go tlsl.acceptLoop()
	return tlsl, nil
}

func (l *tlsListener) Accept() (net.Conn, error) {
	select {
	case conn := <-l.connc:
		return conn, nil
	case <-l.donec:
		return nil, l.err
	}
}

func (l *tlsListener) acceptLoop() {
	var wg sync.WaitGroup
	var pendingMu sync.Mutex

	pending := make(map[net.Conn]struct{})
	ctx, cancel := context.WithCancel(context.Background())
	defer func() {
		cancel()
		pendingMu.Lock()
		for c := range pending {
			c.Close()
		}
		pendingMu.Unlock()
		wg.Wait()
		close(l.donec)
	}()

	for {
		conn, err := l.Listener.Accept()
		if err != nil {
			l.err = err
			return
		}

		pendingMu.Lock()
		pending[conn] = struct{}{}
		pendingMu.Unlock()

		wg.Add(1)
		go func() {
			defer func() {
				if conn != nil {
					conn.Close()
				}
				wg.Done()
			}()

			tlsConn := conn.(*tls.Conn)
			herr := tlsConn.Handshake() // take long time
			pendingMu.Lock()
			delete(pending, conn)
			pendingMu.Unlock()

			if herr != nil {
				l.handshakeFailure(tlsConn, herr)
				return
			}
			if err := l.check(ctx, tlsConn); err != nil {
				l.handshakeFailure(tlsConn, err)
				return
			}

			select {
			case l.connc <- tlsConn:
				conn = nil
			case <-ctx.Done():
			}
		}()
	}
}

```
- the acceptLoop will exit whenever there's an error of inner listener's Accept
    - if it exit, it will try call context.Cancel. 
        - for all the blocking `select`, they will advance immediately
        - for l.`check` that use ctx, they will fail
    - it will not cancel the `Handshake` in progress. it will wait until `check`
    - these separate goroutines when canceled will call wg.Done
    - the acceptLoop is waiting for all wg.Done in its wg.Wait
    - the donc will close, which allows l.Accept to exit immediately
- the acceptLoop also delete all the pending conn that is not doing handshake yet
- the code illustrate the usage of
    - context
    - mutex
    - waitgroup
    - channel