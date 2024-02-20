- when a separate goroutine needs to write a channel before it can exit, make the channel buffered. the write can succeed immediately

- etcd/client/pkg/transport/listener_test.go
```go
	chClientErr := make(chan error, 1)
	go func() {
		_, err := cli.Get("https://" + ln.Addr().String())
		chClientErr <- err
	}()

	chAcceptErr := make(chan error, 1)
	chAcceptConn := make(chan net.Conn, 1)
	go func() {
		conn, err := ln.Accept()
		if err != nil {
			chAcceptErr <- err
		} else {
			chAcceptConn <- conn
		}
	}()

	select {
	case <-chClientErr:
		if acceptExpected {
			t.Errorf("accepted for good client address: skipClientSANVerify=%t, goodClientHost=%t", skipClientSANVerify, goodClientHost)
		}
	case acceptErr := <-chAcceptErr:
		t.Fatalf("unexpected Accept error: %v", acceptErr)
	case conn := <-chAcceptConn:
		defer conn.Close()
		if _, ok := conn.(*tls.Conn); !ok {
			t.Errorf("failed to accept *tls.Conn")
		}
		if !acceptExpected {
			t.Errorf("accepted for bad client address: skipClientSANVerify=%t, goodClientHost=%t", skipClientSANVerify, goodClientHost)
		}
	}
```