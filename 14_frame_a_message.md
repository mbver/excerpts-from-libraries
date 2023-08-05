- if a message to be compressed, it should be compressed once and the compressed data is cached. whenever the data is needed, it is taken from the cache
- the frame function use a fake net.Conn connection (pconn). pconn writes to its buffer, instead of the network connection. the data written to it (compressed) will be retrieved and store in the cache.
- the function that turn the websocket Conn's writer into a compressionWriter is `compressNoContextTakeover`. the Conn's writer is a messageWriter mw, that has Conn as a field mw.c. mw.c now has pconn as a field mw.c.conn.
- the compressWriter has a field fw where it execute Write. but that field somehow turn into a TruncWriter. so TruncWriter execute Write for compressWriter. TruncWriter refer to the messageWriter as a field cw.fw.w
- cw.fw.w Write method is invoked, which will invoke flushFrame. flushFrame will invoke mw.c.write, where c is the websocket connection. mw.c.write will invoke c.writeBuffs. c.writeBuffs turns data in to a [][]byte net.Buffers b and invoke b.WriteTo on c.conn. and c.conn is pconn. pconn.Write just write the data into its pconn.buf
- the summary of flow:
    - c Conn; c.conn = pconn; c.writer = mw
    - mw.c = c
    - frame -> c.WriteMessage -> NextWriter -> compressNoContextTakeover (mw) -> cw{fw:{w:mw}} -> cw.Write -> cw.fw.w.Write -> mw.flushFrame -> c.write -> c.writeBuffs -> net.Buffers.WriteTo(c.conn) -> pconn.Write -> pconn.buf
- the flow of compression is very complicated and requires another investigation

- gorilla/websocket/prepared.go frame
```go
func (pm *PreparedMessage) frame(key prepareKey) (int, []byte, error) {
	pm.mu.Lock()
	frame, ok := pm.frames[key]
	if !ok {
		frame = &preparedFrame{}
		pm.frames[key] = frame
	}
	pm.mu.Unlock()

	var err error
	frame.once.Do(func() {
		// Prepare a frame using a 'fake' connection.
		// TODO: Refactor code in conn.go to allow more direct construction of
		// the frame.
		mu := make(chan struct{}, 1)
		mu <- struct{}{}
		var nc prepareConn
		c := &Conn{
			conn:                   &nc,
			mu:                     mu,
			isServer:               key.isServer,
			compressionLevel:       key.compressionLevel,
			enableWriteCompression: true,
			writeBuf:               make([]byte, defaultWriteBufferSize+maxFrameHeaderSize),
		}
		if key.compress {
			c.newCompressionWriter = compressNoContextTakeover
		}
		err = c.WriteMessage(pm.messageType, pm.data)
		frame.data = nc.buf.Bytes()
	})
	return pm.messageType, frame.data, err
}
```

- gorilla/websocket/conn.go WriteMessage
```go
func (c *Conn) WriteMessage(messageType int, data []byte) error {

	if c.isServer && (c.newCompressionWriter == nil || !c.enableWriteCompression) {
		// Fast path with no allocations and single frame.

		var mw messageWriter
		if err := c.beginMessage(&mw, messageType); err != nil {
			return err
		}
		n := copy(c.writeBuf[mw.pos:], data)
		mw.pos += n
		data = data[n:]
		return mw.flushFrame(true, data)
	}

	w, err := c.NextWriter(messageType) // frame follow this path
	if err != nil {
		return err
	}
	if _, err = w.Write(data); err != nil {
		return err
	}
	return w.Close()
}
```

- gorilla/websocket/conn.go NextWriter
```go
func (c *Conn) NextWriter(messageType int) (io.WriteCloser, error) {
	var mw messageWriter
	if err := c.beginMessage(&mw, messageType); err != nil {
		return nil, err
	}
	c.writer = &mw
	if c.newCompressionWriter != nil && c.enableWriteCompression && isData(messageType) {
		w := c.newCompressionWriter(c.writer, c.compressionLevel)
		mw.compress = true
		c.writer = w
	}
	return c.writer, nil
}
```

- gorilla/websocket/compression.go compressNoContextTakeover
```go
func compressNoContextTakeover(w io.WriteCloser, level int) io.WriteCloser {
	p := &flateWriterPools[level-minCompressionLevel]
	tw := &truncWriter{w: w}
	fw, _ := p.Get().(*flate.Writer)
	if fw == nil {
		fw, _ = flate.NewWriter(tw, level)
	} else {
		fw.Reset(tw)
	}
	return &flateWriteWrapper{fw: fw, tw: tw, p: p}
}
```
- gorilla/websocket/compression.go truncWrite Write
```go
type truncWriter struct {
	w io.WriteCloser
	n int
	p [4]byte
}

func (w *truncWriter) Write(p []byte) (int, error) {
	n := 0

	// fill buffer first for simplicity.
	if w.n < len(w.p) {
		n = copy(w.p[w.n:], p)
		p = p[n:]
		w.n += n
		if len(p) == 0 {
			return n, nil
		}
	}

	m := len(p)
	if m > len(w.p) {
		m = len(w.p)
	}

	if nn, err := w.w.Write(w.p[:m]); err != nil {
		return n + nn, err
	}

	copy(w.p[:], w.p[m:])
	copy(w.p[len(w.p)-m:], p[len(p)-m:])
	nn, err := w.w.Write(p[:len(p)-m])
	return n + nn, err
}
```

- gorilla/websocket/conn.go Conn.write
```go
func (c *Conn) write(frameType int, deadline time.Time, buf0, buf1 []byte) error {
	<-c.mu
	defer func() { c.mu <- struct{}{} }()

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
		err = c.writeBufs(buf0, buf1) // frame follows this path
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

- gorilla/websocket/conn.go Conn->writeBufs
```go
func (c *Conn) writeBufs(bufs ...[]byte) error {
	b := net.Buffers(bufs)
	_, err := b.WriteTo(c.conn)
	return err
}
```

- net/net.go Buffers->WriteTo
```go
func (v *Buffers) WriteTo(w io.Writer) (n int64, err error) {
	if wv, ok := w.(buffersWriter); ok {
		return wv.writeBuffers(v)
	}
	for _, b := range *v {
		nb, err := w.Write(b)
		n += int64(nb)
		if err != nil {
			v.consume(n)
			return n, err
		}
	}
	v.consume(n)
	return n, nil
}
```

- gorilla/websocket/conn.go messageWriter->Write
```go
func (w *messageWriter) Write(p []byte) (int, error) {
	if w.err != nil {
		return 0, w.err
	}

	if len(p) > 2*len(w.c.writeBuf) && w.c.isServer {
		// Don't buffer large messages.
		err := w.flushFrame(false, p)
		if err != nil {
			return 0, err
		}
		return len(p), nil
	}

	nn := len(p)
	for len(p) > 0 {
		n, err := w.ncopy(len(p))
		if err != nil {
			return 0, err
		}
		copy(w.c.writeBuf[w.pos:], p[:n])
		w.pos += n
		p = p[n:]
	}
	return nn, nil
}
```
- gorilla/websocket/conn.go messageWriter->flushFrame
```go
func (w *messageWriter) flushFrame(final bool, extra []byte) error {
	c := w.c
	length := w.pos - maxFrameHeaderSize + len(extra)

	// Check for invalid control frames.
	if isControl(w.frameType) &&
		(!final || length > maxControlFramePayloadSize) {
		return w.endMessage(errInvalidControlFrame)
	}

	b0 := byte(w.frameType)
	if final {
		b0 |= finalBit
	}
	if w.compress {
		b0 |= rsv1Bit
	}
	w.compress = false

	b1 := byte(0)
	if !c.isServer {
		b1 |= maskBit
	}

	// Assume that the frame starts at beginning of c.writeBuf.
	framePos := 0
	if c.isServer {
		// Adjust up if mask not included in the header.
		framePos = 4
	}

	switch {
	case length >= 65536:
		c.writeBuf[framePos] = b0
		c.writeBuf[framePos+1] = b1 | 127
		binary.BigEndian.PutUint64(c.writeBuf[framePos+2:], uint64(length))
	case length > 125:
		framePos += 6
		c.writeBuf[framePos] = b0
		c.writeBuf[framePos+1] = b1 | 126
		binary.BigEndian.PutUint16(c.writeBuf[framePos+2:], uint16(length))
	default:
		framePos += 8
		c.writeBuf[framePos] = b0
		c.writeBuf[framePos+1] = b1 | byte(length)
	}

	if !c.isServer {
		key := newMaskKey()
		copy(c.writeBuf[maxFrameHeaderSize-4:], key[:])
		maskBytes(key, 0, c.writeBuf[maxFrameHeaderSize:w.pos])
		if len(extra) > 0 {
			return w.endMessage(c.writeFatal(errors.New("websocket: internal error, extra used in client mode")))
		}
	}

	// Write the buffers to the connection with best-effort detection of
	// concurrent writes. See the concurrency section in the package
	// documentation for more info.

	if c.isWriting {
		panic("concurrent write to websocket connection")
	}
	c.isWriting = true

	err := c.write(w.frameType, c.writeDeadline, c.writeBuf[framePos:w.pos], extra)

	if !c.isWriting {
		panic("concurrent write to websocket connection")
	}
	c.isWriting = false

	if err != nil {
		return w.endMessage(err)
	}

	if final {
		w.endMessage(errWriteClosed)
		return nil
	}

	// Setup for next frame.
	w.pos = maxFrameHeaderSize
	w.frameType = continuationFrame
	return nil
}
```

- gorilla/websocket/prepared.go prepareConn, Write, SetWriteDeadline
```go
type prepareConn struct {
	buf bytes.Buffer
	net.Conn
}

func (pc *prepareConn) Write(p []byte) (int, error)        { return pc.buf.Write(p) }
func (pc *prepareConn) SetWriteDeadline(t time.Time) error { return nil }
```
