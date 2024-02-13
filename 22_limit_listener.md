- etcd/client/pkg/transport/limit_listen.go
- a limit-listener accepts at most n simultaneous connections
```go
type limitListener struct {
	net.Listener
	sem chan struct{}
}

func (l *limitListener) acquire() { l.sem <- struct{}{} }
func (l *limitListener) release() { <-l.sem }

func (l *limitListener) Accept() (net.Conn, error) {
	l.acquire()
	c, err := l.Listener.Accept()
	if err != nil {
		l.release()
		return nil, err
	}
	return &limitListenerConn{Conn: c, release: l.release}, nil
}

func LimitListener(l net.Listener, n int) net.Listener {
	return &limitListener{l, make(chan struct{}, n)}
}
```

- the limitListenerConn helps to release on semaphore when it is closed
```go
type limitListenerConn struct {
	net.Conn
	releaseOnce sync.Once
	release     func()
}

func (l *limitListenerConn) Close() error {
	err := l.Conn.Close()
	l.releaseOnce.Do(l.release) // if close is called multiple times, release is called just once!
	return err
}
```

