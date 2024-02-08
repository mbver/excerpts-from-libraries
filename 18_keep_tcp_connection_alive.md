- When the `http handler` receives the request, it will not return after processing the request. instead, it creates a `closeNotifier` that notified when the connection should be closed. the http handler blocks waiting on that channel and keeps the connection alive. whenever the close notifier is triggered, the handler will safely exit and the tcp connection is closed automatically.
- etcd/server/etcdserver/rafthttp/http.go
```go
func (h *streamHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	...
    w.WriteHeader(http.StatusOK)
	w.(http.Flusher).Flush()

	c := newCloseNotifier()
	conn := &outgoingConn{
		t:       t,
		Writer:  w,
		Flusher: w.(http.Flusher),
		Closer:  c,
		localID: h.tr.ID,
		peerID:  from,
	}
	p.attachOutgoingConn(conn)
	<-c.closeNotify()
}
```