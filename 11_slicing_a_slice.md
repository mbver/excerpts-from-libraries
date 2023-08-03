- a nil slice (uninitialized slice) has a length and capacity of 0, and the pointer is nil
- but it can be sliced! but slicing it will also generate a nil slice!
```go
	var x []byte
	fmt.Println(x == nil)
	x = x[:0]
	fmt.Println(x == nil)
```