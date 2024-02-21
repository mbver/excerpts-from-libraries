- to remove an element in the middle of the slice:
    - move the elements on the right of that element back by 1
    - reduce the length of slice to 1

- go/src/pkg/net b0f39cc27c
```go
func (p *pollster) DelFD(fd int, mode int) {
	// pollServer is locked.
    ...
	i := 0
	for i < len(p.waitEvents) {
		if fd == int(p.waitEvents[i].Fd) {
			copy(p.waitEvents[i:], p.waitEvents[i+1:])
			p.waitEvents = p.waitEvents[:len(p.waitEvents)-1]
		} else {
			i++
		}
	}
}
```