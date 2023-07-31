- a buffered channel can be used as mutex
    - Conn has a field called mu, a buffered channel of size 1
    - it is sent a signal when conn is created
    - whenever we need to write to Conn, mu will be read
    - the read is blocked if mu is empty, meaning some goroutine is trying to write
    - after finished writing, we send a signal back to mu, make other goroutine to be able to proceed
    - should use defer for resending signal

```go
type Conn struct {

    ...
	// Write fields
	mu            chan struct{} // used as mutex to protect write to conn
    ...
}

func newConn(...) *Conn {
    ...
	mu := make(chan struct{}, 1)
	mu <- struct{}{}
	c := &Conn{
        ...
		mu: mu,
        ...
	}
    ...
	return c
}

func (c *Conn) write(frameType int, deadline time.Time, buf0, buf1 []byte) error {
	<-c.mu // locking
	defer func() { c.mu <- struct{}{} }() // unlocking

	c.writeErrMu.Lock()
	err := c.writeErr
	c.writeErrMu.Unlock()
	if err != nil {
		return err
	}

	c.conn.SetWriteDeadline(deadline)
	if len(buf1) == 0 {
		_, err = c.conn.Write(buf0)
	} else {
		err = c.writeBufs(buf0, buf1)
	}
	if err != nil {
		return c.writeFatal(err)
	}
	if frameType == CloseMessage {
		c.writeFatal(ErrCloseSent)
	}
	return nil
}

```
