- net/net.go Buffers
    - Buffers has Read and WriteTo methods
    - we want to assert it satisfy io.WriteTo and io.Reader interfaces
    - this is done via type assertion
    - the variable obtained by the assertion is unused
    - we use nil as input to type assertion
```go
type Buffers [][]byte
// _ is the variable obtained after type assertion. it is unused
var (
	_ io.WriterTo = (*Buffers)(nil)
	_ io.Reader   = (*Buffers)(nil)
)
...
```