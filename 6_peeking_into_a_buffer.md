- the method try to look into the buffer to get n bytes from it
    - if it doesn't have enough bytes or some error happens, we get ErrBufferFull or other error
    - on the surface, it may look simple, but several cases can happen and many things can go wrong
    - the logic seems weird at first but it handle all these cases
    - part of the complication is from the fill() method, which may get more bytes but not enough or can create an error
    - the readErr method is called to get err from buffer, but also reset it to nil
- a notable technique used to offset the bytes in buffer to the beginning: using copy on that slice!

- bufio/bufio.go Peek
```go
// Peek returns the next n bytes without advancing the reader. The bytes stop
// being valid at the next read call. If Peek returns fewer than n bytes, it also returns
// ErrBufferFull, telling n is larger than b's buffer size.
//
// Calling Peek prevents a UnreadByte or UnreadRune call from succeeding
// until the next read operation.
func (b *Reader) Peek(n int) ([]byte, error) {
	if n < 0 {
		return nil, ErrNegativeCount
	}

	b.lastByte = -1 // prevent calling UnreadByte. check for it in bufio
	b.lastRuneSize = -1 // prevent calling UnreadRune, check in bufio

	for b.w-b.r < n && b.w-b.r < len(b.buf) && b.err == nil {
        // b.w - b.r: the bytes written in buffer, to be read. smaller than n? then need to add more byte
        // b.w-b.r < len(b.buf) ==> avoid the panic if we try to fill a full buffer!
        // b.err == nil: we have no error, so can add data
		b.fill()
        // b.fill reset b.r to 0 and b.w to b.w-b.r
        // b.fill try to read some new data into b.buf
        // 3 cases can happen
        // 1. b.w-b.r > n => read enough new data, have enough bytes
        // 2. b.w-b.r < n => some new data read, and no error created by fill
        // 3. bw-b.r is the same => no new data read and b.err = errNoProgress
	}

	if n > len(b.buf) {
        // in this case, no matter the read is successful or not, can never read enough bytes
		return b.buf[b.r:b.w], ErrBufferFull
	}

	// 0 <= n <= len(b.buf)
    // we handle several case: 1.enough bytes: b.w - b.r < n; 2.fill is success, but not enough bytes; 3. fill creates an error
	var err error
	if avail := b.w - b.r; avail < n {
        // handle the cases not enough bytes
		n = avail // n will be used later to return, 
		err = b.readErr() // get err. err can be errNoProgress or nil or any err happens before
		if err == nil {
			err = ErrBufferFull
		}
	}
    // handle every case. if ok, n is the same, err is nil. if not ok, n is smaller, err is not nil
	return b.buf[b.r : b.r+n], err
}
```

- bufio/bufio.go fill
```go
func (b *Reader) fill() {
	// Slide existing data to beginning.
	if b.r > 0 {
		copy(b.buf, b.buf[b.r:b.w]) // offset the bytes using copy on the same slice!
		b.w -= b.r // reset b.w to b.w-b.r
		b.r = 0 // reset b.r to 0
	}

	if b.w >= len(b.buf) { // b.w is at len(b.buf)
		panic("bufio: tried to fill full buffer")
	}

	// Read new data: try a limited number of times.
	for i := maxConsecutiveEmptyReads; i > 0; i-- {
		n, err := b.rd.Read(b.buf[b.w:]) // try to read data from b.rd into b.buf
		if n < 0 {
			panic(errNegativeRead)
		}
		b.w += n // advance b.w depending on the data read
		if err != nil {
			b.err = err
			return
		}
		if n > 0 { // whenever some data is read, return
			return
		}
	}
	b.err = io.ErrNoProgress // if can not read any data, assign an error to b.err
}
```

- bufio/bufio.go readErr
```go
func (b *Reader) readErr() error {
	err := b.err
	b.err = nil // reading will reset b.err to nil
	return err
}
```