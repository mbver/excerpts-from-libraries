- components of a websocket mostly created from an underlying net.Conn from a http.ResponseWriter
- through Hijack() method, we obtain the net.Conn and a bufio.ReadWriter(brw)
- we can reuse: brw.Reader
- we can also reuse brw.Writer's buffer
- the Conn represents a websocket connection and uses the above 3 components

- gorilla/websocket/server.go Upgrade
```go
netConn, brw, err := h.Hijack()
var br *bufio.Reader
if ... {
    // Reuse hijacked buffered reader as connection reader.
    br = brw.Reader
}
buf := bufioWriterBuffer(netConn, brw.Writer)
var writeBuf []byte
if ... {
    // Reuse hijacked write buffer as connection buffer.
    writeBuf = buf
}
c := newConn(netConn, true, ..., br, writeBuf)
```
- gorilla/websocket/conn.go newConn
```go
func newConn(conn net.Conn, ...,  br *bufio.Reader, writeBuf []byte) *Conn {

	if br == nil {
        ...
		br = bufio.NewReaderSize(conn, readBufferSize)
	}
    ...

	if writeBuf == nil && ... {
		writeBuf = make([]byte, writeBufferSize)
	}

	mu := make(chan struct{}, 1)
	mu <- struct{}{}
	c := &Conn{
		br:                     br,
		conn:                   conn,
		mu:                     mu,
		writeBuf:               writeBuf,
        ...
	}
	return c
}
```
- gorilla/websocket/conn.go Conn (type)
```go
type Conn struct {
	conn        net.Conn

	// Write fields
	mu            chan struct{} // used as mutex to protect write to conn
	writeBuf      []byte        // frame is constructed in this buffer.
	writer        io.WriteCloser // the current writer returned to the application

	writeErrMu sync.Mutex
	writeErr   error

	// Read fields
	reader  io.ReadCloser // the current reader returned to the application
	br      *bufio.Reader
    ....
}
```

