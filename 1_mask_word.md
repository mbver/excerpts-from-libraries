1. obtaining bit patterns
- a word has the same size as an uintptr
- if we have an array of 8 bytes, we can turn it into an uintptr
    - obtain the address of array
    - cast address into an unsafe.Pointer
    - turn the unsafe.Pointer into *uintptr (pointer to an uintptr)
    - access the value at that *uintptr (an uintptr of 8 bytes!)
- the obtained uintptr represents the bit pattern of array. if the system is little endian, the number represented byt the bit pattern is reversed

```go
func main() {
	var k [8]byte
	key := [4]byte{0, 1, 2, 3}
	for i := range k {
		k[i] = key[i&3]
	}
	fmt.Println("k:", k)
	fmt.Print("reverse bit pattern of k: ") // the system is little endian, so bit pattern will be reversed
	for i := len(k) - 1; i >= 0; i-- {
		fmt.Printf("%08b", k[i])
	}
	fmt.Println()
	fmt.Printf("address of k: %p\n", &k)
	x := unsafe.Pointer(&k) // turn address into an unsafe.Pointer
	fmt.Printf("x : %T, %v\n", x, x)
	y := (*uintptr)(x) // turn unsafe.Pointer to *uintptr. meaning this address is to an uinptr
    // an uintptr size is 8 bytes.
	fmt.Printf("y: %T, %v\n", y, y)
	z := *y // obtaining value of uintptr from the pointer
	fmt.Printf("z: %T, %v\n", z, z)
	fmt.Printf("*(*uintptr)(unsafe.Pointer(&k)): %064b\n", z) // bit pattern will be reversed
}
```