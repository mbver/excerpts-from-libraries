- gorilla/websocket/conn.go
- what it does:
    - write data to a connection with a deadline
- steps:
    - check message type. only allows: close, ping, pong
    - building data buffer:
        - first 2 bytes: message type, data len
        - data itself
            - if conn is on client side, add key and mask data before appending to buf
    - check if deadline exceeded
    - make sure to write a signal after finished
    - check if there's a writeErr in the connection
    - set deadline
    - write buffered data to the connection
    - check error and if not nil, assign it to c.writeErr

```go   
func (c *Conn) WriteControl(messageType int, data []byte, deadline time.Time) error {
	if !isControl(messageType) { // CloseMessage is a control message. pass!
		return errBadWriteOpCode
	}
	if len(data) > maxControlFramePayloadSize {
		return errInvalidControlFrame
	}

	b0 := byte(messageType) | finalBit // finalBit=1>>7 = 128
	b1 := byte(len(data)) // data length, max is 255 if converted to byte
	if !c.isServer { // isServer is true
		b1 |= maskBit // maskBit = 1>>7 = 128
	}

	buf := make([]byte, 0, maxFrameHeaderSize+maxControlFramePayloadSize) // we fix the capacity while length is 0
	buf = append(buf, b0, b1) // buf[0] = b0, buf[1] = b1

	if c.isServer {
		buf = append(buf, data...) // isServer = true
	} else {
		key := newMaskKey() // key is an array of 4 bytes
		buf = append(buf, key[:]...) // convert key to slice first, then spread it
		buf = append(buf, data...)
		maskBytes(key, 0, buf[6:]) // 2 bytes for b0, b1, 4 bytes for key, data start from 6
	}

	d := 1000 * time.Hour
	if !deadline.IsZero() { // deadline is not 0 too
		d = deadline.Sub(time.Now())
		if d < 0 {
			return errWriteTimeout
		}
	}

	timer := time.NewTimer(d)
	select {
	case <-c.mu:
		timer.Stop() // c.mu already has a struct{}, this will stop
	case <-timer.C:
		return errWriteTimeout
	}
	defer func() { c.mu <- struct{}{} }() // send a signal to c.u after finishing

	c.writeErrMu.Lock() // why we need a lock?
	err := c.writeErr
	c.writeErrMu.Unlock()
	if err != nil {
		return err
	}

	c.conn.SetWriteDeadline(deadline) // it is defined in the interface. conn is in net package
	_, err = c.conn.Write(buf) // in net.Conn interface
	if err != nil {
		return c.writeFatal(err) // assign err to c.writeErr
	}
	if messageType == CloseMessage {
		c.writeFatal(ErrCloseSent) // assign err to c.writeErr. do nothing else
	}
	return err
}
```