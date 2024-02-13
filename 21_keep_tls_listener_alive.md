- etcd/client/pkg/transport
- a net.TCPConn can be kept alive

```go
type keepAliveConn struct {
	*net.TCPConn
}

// SetKeepAlive sets keepalive
func (l *keepAliveConn) SetKeepAlive(doKeepAlive bool) error {
	return l.TCPConn.SetKeepAlive(doKeepAlive)
}

// SetKeepAlivePeriod sets keepalive period
func (l *keepAliveConn) SetKeepAlivePeriod(d time.Duration) error {
	return l.TCPConn.SetKeepAlivePeriod(d)
}
```

- a keepAliveConn can be wrapped in a keep-alive listener
```go
type keepaliveListener struct{ net.Listener }

func (kln *keepaliveListener) Accept() (net.Conn, error) {
	c, err := kln.Listener.Accept()
	if err != nil {
		return nil, err
	}

	kac, err := createKeepaliveConn(c)
	if err != nil {
		return nil, fmt.Errorf("create keepalive connection failed, %w", err)
	}
	// detection time: tcp_keepalive_time + tcp_keepalive_probes + tcp_keepalive_intvl
	// default on linux:  30 + 8 * 30
	// default on osx:    30 + 8 * 75
	if err := kac.SetKeepAlive(true); err != nil {
		return nil, fmt.Errorf("SetKeepAlive failed, %w", err)
	}
	if err := kac.SetKeepAlivePeriod(30 * time.Second); err != nil {
		return nil, fmt.Errorf("SetKeepAlivePeriod failed, %w", err)
	}
	return kac, nil
}

func createKeepaliveConn(c net.Conn) (*keepAliveConn, error) {
	tcpc, ok := c.(*net.TCPConn)
	if !ok {
		return nil, ErrNotTCP
	}
	return &keepAliveConn{tcpc}, nil
}
```

- a keep-alive listener is wrapped in a tls-keep-alive-listener
```go
type tlsKeepaliveListener struct {
	net.Listener
	config *tls.Config
}

// Accept waits for and returns the next incoming TLS connection.
// The returned connection c is a *tls.Conn.
func (l *tlsKeepaliveListener) Accept() (c net.Conn, err error) {
	c, err = l.Listener.Accept()
	if err != nil {
		return
	}
	c = tls.Server(c, l.config)
	return c, nil
}

// NewListener creates a Listener which accepts connections from an inner
// Listener and wraps each connection with Server.
// The configuration config must be non-nil and must have
// at least one certificate.
func newTLSKeepaliveListener(inner net.Listener, config *tls.Config) net.Listener {
	l := &tlsKeepaliveListener{}
	l.Listener = inner
	l.config = config
	return l
}

// how it is used
	kal := &keepaliveListener{
		Listener: l,
	}

	if scheme == "https" {
		if tlscfg == nil {
			return nil, fmt.Errorf("cannot listen on TLS for given listener: KeyFile and CertFile are not presented")
		}
		return newTLSKeepaliveListener(kal, tlscfg), nil
	}
```