- brw is created by hijacking the w http.ResponseWriter. maybe its original brw.Writer.wr is netConn
- bufioWriterBuffer need netConn to put it back to brw.Writer
    - the writeHook has the Write method.
        - when Write method is called on a slice, it grabs that slice
    - first, put writeHook to brw.Writer.wr
    - then call WriteByte(0), which also invoke Flush, which invoke Write on the writeHook
        - a byte 0 is written to brw.Writer.buf, n = 1
    - then Flush is called again
        - writeHook grab the brw.Write.buf, nothing new from previous step!
    - put netConn back to brw.Writer.wr

- it seems that, we just need to call Flush once, no need to call WriteByte, then the br.Write.buf is grabbed by the writeHook!. Not sure why it has to be more complicated.

```go
// gorilla/websocket/server.go Upgrade
h, ok := w.(http.Hijacker)
netConn, brw, err := h.Hijack() // netConn is net.Conn, brw: *bufio.ReadWriter
buf := bufioWriterBuffer(netConn, brw.Writer)
...
// gorilla/websocket/server.go bufioWriterBuffer
type writeHook struct {
	p []byte
}

func (wh *writeHook) Write(p []byte) (int, error) {
	wh.p = p
	return len(p), nil
}

// bufioWriterBuffer grabs the buffer from a bufio.Writer.
func bufioWriterBuffer(originalWriter io.Writer, bw *bufio.Writer) []byte {
	var wh writeHook
	bw.Reset(&wh)
	bw.WriteByte(0) // this will set b.n to 1
	bw.Flush() // this will set the b.n back to 0 again
    // wh.Write is invoked 2 times, seems unnecessary?
	bw.Reset(originalWriter)

	return wh.p[:cap(wh.p)]
}

// bufio/bufio.go WriteByte
func (b *Writer) WriteByte(c byte) error {
	if b.err != nil {
		return b.err
	}
	if b.Available() <= 0 && b.Flush() != nil {
		return b.err
	}
	b.buf[b.n] = c
	b.n++
	return nil
}

// bufio/bufio.go Flush
func (b *Writer) Flush() error {
	if b.err != nil {
		return b.err
	}
	if b.n == 0 {
		return nil
	}
	n, err := b.wr.Write(b.buf[0:b.n])
	if n < b.n && err == nil {
		err = io.ErrShortWrite
	}
	if err != nil {
		if n > 0 && n < b.n {
			copy(b.buf[0:b.n-n], b.buf[n:b.n])
		}
		b.n -= n
		b.err = err
		return err
	}
	b.n = 0
	return nil
}
```

