when write-to/read-from a channel, we may be blocked. if we want to receive/send or quit immediately, use select with default
```go
    // quit immediately if can not send right away
	select {
	case <-ch:
		t.Fatalf("should not get leave")
	default:
	}
    // quit immediately if can not receive right away
    select {
	case ch <- n:
	default:
	}
```