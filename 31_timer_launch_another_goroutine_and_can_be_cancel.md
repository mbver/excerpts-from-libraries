- github.com/hashicorp/state.go
- a timer can be used to launch another goroutine that do something after timeout. at sometime, we want to cancel that goroutine before timeout, use timer.Stop.
- setAckHandler is called when we setup a handler for a ping msg, with a unique SeqNo. When the response arrives, we will call invokeAckHandler for that ping msg, with that SeqNo. We want to cancel the goroutine in setAckHandler. Use timer.Stop to cancel it.
```go
func (m *Memberlist) setAckHandler(seqNo uint32, handler func(), timeout time.Duration) {
	// Add the handler
	ah := &ackHandler{handler, nil}
	m.ackLock.Lock()
	m.ackHandlers[seqNo] = ah
	m.ackLock.Unlock()

	// delete ackHandler after timeout.
	// use timer.Stop to cancel.
	ah.timer = time.AfterFunc(timeout, func() {
		m.ackLock.Lock()
		delete(m.ackHandlers, seqNo)
		m.ackLock.Unlock()
	})
}

// find ackHandler for seqNo. delete it in map.
// cancel timer. run the handler.
func (m *Memberlist) invokeAckHandler(seqNo uint32) {
	m.ackLock.Lock()
	ah, ok := m.ackHandlers[seqNo]
	delete(m.ackHandlers, seqNo)
	m.ackLock.Unlock()
	if !ok {
		return
	}
	ah.timer.Stop() // cancel timer
	ah.handler()
}
```