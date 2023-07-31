- net/net.go Buffers consume
    - whenever n bytes from Buffers is read or written, we call consume so the next read or write can start on the first unused byte
    - consume is called only once, when error happens or the read/write finish
```go
// consume means n bytes in the buffers has been used, advanced to the next first unused byte
// the next first unused byte is determined by: what is the first slice of buffers, and what is the first byte of that slice
// change the first slice of buffers, buffers has to be a pointer, so we can manipulate the slice under it
func (v *Buffers) consume(n int64) {
	for len(*v) > 0 {
		ln0 := int64(len((*v)[0]))
		if ln0 > n { // enough, gonna return
			(*v)[0] = (*v)[0][n:] // change the slice element, don't need pointer
			return
		}
		n -= ln0
		(*v)[0] = nil // for garbage collection
		*v = (*v)[1:] // change the slice, need to be a pointer
	}
}

// here's how consume is called
func (v *Buffers) WriteTo(w io.Writer) (n int64, err error) {
	if wv, ok := w.(buffersWriter); ok {
		return wv.writeBuffers(v)
	}
	for _, b := range *v {
		nb, err := w.Write(b)
		n += int64(nb)
		if err != nil {
			v.consume(n) // consume is called only once, when an error happens or the loop finished
			return n, err
		}
	}
	v.consume(n)
	return n, nil
}

```