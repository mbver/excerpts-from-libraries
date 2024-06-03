- we want to sleep but want to exit sleep early whenever a stop signal arrives? instead of using time.Sleep, we can use time.After. then select on both channels.
- github.com/hashicorp/memberlist/state.go

```go
	select {
	case <-time.After(randStagger):
	case <-stop:
		return
	}
```