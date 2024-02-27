- a peer has 2 handlers, each runs an infinite loop. a peer also has a stop method that can be invoked outside. whenever error occurs, we want the loop to exit and notify the peer to stop. whenever stop is called, all handlers must be stop.
- stop is designed to be called multiple times. it close a quit channel to signal handler to stop. before closing, it must check if the channel is closed. using `select` and default to examine it.
```go

// Outbound message handler. Outbound messages are handled here
func (p *Peer) HandleOutbound() {
	defer p.Stop()
out:
	for {
		select {
		case msg := <-p.outputQueue:
			err := ...
			if err != nil {
				log.Println(err)

				break out
			}
		case <-p.quit:
			break out
		}
	}
}

// Inbound handler. Inbound messages are received here and passed to the appropriate methods
func (p *Peer) HandleInbound() {
	defer p.Stop()

out:
	for {
		// Wait for a message from the peer
		msg, err := ...
		if err != nil {
			log.Println(err)
			break out
		}
		...
	}
}

func (p *Peer) Stop() {
	...
	select {
	case <-p.quit:
	default:
        close(p.quit)
	}
}
```