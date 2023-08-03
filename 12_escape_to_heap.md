- returning a pointer can make the variable escape from stack to heap
- it is easier to destroy the variables in stack than in heap
- variables escaping to heap can create more work for garbage collectors and affect performance
- run `go run -gcflags="-m -l" main.go` to see optimization decision made by compiler. -m is for optimization
```go
package main

// returnValue returns a value over the call stack
func returnValue() int {
	n := 42
	return n
}

// returnPointer returns a pointer to n, and n is moved to the heap
func returnPointer() *int {
	n := 42 // --- line 11 ---
	return &n
}

// returnSlice returns a slice that escapes to the heap,
// because the slice header includes a pointer to the data.
func returnSlice() []int {
	slice := []int{42} // --- line 18 ---
	return slice
}

// returnArray returns an array that does not escape to the heap,
// because arrays need no header for tracking length and capacity
// and are always copied by value.
func returnArray() [1]int {
	return [1]int{42}
}

// largeArray creates a ridiculously large array that escapes to the heap,
// even though the array itself is not returned
// and thus does not outlive the function.
func largeArray() int {
	var largeArray [100000000]int // --- line 33 ---
	largeArray[42] = 42
	return largeArray[42] // if the array is small, this is passed by value!
    // for size of 1.000.000, it doesn't escape. but for 2.000.000, it doesß
}

// returnFunc() returns a function that escapes to the heap
func returnFunc() func() int {
	f := func() int { // --- line 40 ---
		return 42
	}
	return f
}

func main() {
	a := returnValue()
	p := *returnPointer()
	s := returnSlice()
	arr := returnArray()
	la := largeArray()
	f := returnFunc()()

	// Consume the variables to avoid compiler warnings.
	// I don't use Printf/ln because this produces a lot
	// of extra escape messages.
	if a+p+s[0]+arr[0]+la+f == 0 {
		return
	}
}
```

- bytes/buffer.go growSlice
- if we extend a slice b by b = append(b, make([]byte, n)...), it will escape to heap
- so we must create another slice with bigger length, then assign b to it

```go
// growSlice grows b by n, preserving the original content of b.
// If the allocation fails, it panics with ErrTooLarge.
func growSlice(b []byte, n int) []byte {
	defer func() {
		if recover() != nil {
			panic(ErrTooLarge)
		}
	}()
	// TODO(http://golang.org/issue/51462): We should rely on the append-make
	// pattern so that the compiler can call runtime.growslice. For example:
	//	return append(b, make([]byte, n)...)
	// This avoids unnecessary zero-ing of the first len(b) bytes of the
	// allocated slice, but this pattern causes b to escape onto the heap.

	// ----> in the future, this will be fixed. and we will use the above code!!!!
	
    // Instead use the append-make pattern with a nil slice to ensure that
	// we allocate buffers rounded up to the closest size class.
	// ----> for now, create a slice manually so b will not escape to heap!

    c := len(b) + n // ensure enough space for n elements
	if c < 2*cap(b) {
		// The growth rate has historically always been 2x. In the future,
		// we could rely purely on append to determine the growth rate.
		c = 2 * cap(b)
	}
	b2 := append([]byte(nil), make([]byte, c)...)
	copy(b2, b)
	return b2[:len(b)]
}
```