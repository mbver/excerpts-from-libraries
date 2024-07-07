- sometimes we want the tcp listener accepts only 1 connection then reject all the later incoming connection. for example, in testing. close the listener immediately right after accepting the first incoming request.

github.com/hashicorp/serf/command/agent/rpc_client.go ccbafea
```go
	var conn net.Conn
    var l net.Listener
	var lClosed bool
	var lock sync.Mutex

	l, err := net.Listen("tcp", "127.0.0.1:0")

    go func() {
        var err error
		conn, err = l.Accept()

        lock.Lock()
        if !lClosed {
            l.Close()
            lClosed = true
        }
        lock.Unlock()
    }
```