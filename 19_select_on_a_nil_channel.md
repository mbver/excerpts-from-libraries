- selecting on a nil channel will be blocked until its value is assigned. this technique is used when we want to control the order of processing in select statement. 
    - if chan B already has data to be read, but we want processing signal from chan A first, then use a nil tmp chan, select on chan A and tmp chan. after processing signal of chan A, assign tmp chan to chan B
- etcd/server/etcdserver/api/rafthttp/stream.go
```go
func (cw *streamWriter) run() {
	var (
		msgc       chan raftpb.Message
		enc        encoder
		flusher    http.Flusher
	)
    ...
	for {
		select {
        ...
		case m := <-msgc: // waits until msgc is assigned cw.msgc
			err := enc.encode(&m)
			if err == nil {
                ...
				continue
			}
            ...
		case conn := <-cw.connc: // process signal on connc before reading in cw.msgc
            ...
			t = conn.t
			switch conn.t {
			case streamTypeMsgAppV2:
				enc = newMsgAppV2Encoder(conn.Writer, cw.fs)
			case streamTypeMessage:
				enc = &messageEncoder{w: conn.Writer}
            ...
			flusher = conn.Flusher
			cw.closer = conn.Closer
            // enc, flusher, cw.closer needs to be populated before processing messages
			if closed {
				if cw.lg != nil {
					cw.lg.Warn(
						"closed TCP streaming connection with remote peer",
						zap.String("stream-writer-type", t.String()),
						zap.String("local-member-id", cw.localID.String()),
						zap.String("remote-peer-id", cw.peerID.String()),
					)
				}
			}
			..., msgc = ..., cw.msgc
	}}}
}
```