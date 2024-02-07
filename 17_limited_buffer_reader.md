- when read from a request body, the amount of bytes to read from connection is limited to avoid accidental timeout due to possible `blocking in underlying implementation`
- etcd/pkg/ioutil/reader.go
```go
func NewLimitedBufferReader(r io.Reader, n int) io.Reader {
	return &limitedBufferReader{
		r: r,
		n: n,
	}
}

type limitedBufferReader struct {
	r io.Reader
	n int
}

// read at most r.n bytes
func (r *limitedBufferReader) Read(p []byte) (n int, err error) {
	np := p
	if len(np) > r.n {
		np = np[:r.n]
	}
	return r.r.Read(np)
}
```
- etcd/server/etcdserver/rafthttp/http.go
```go
	limitedr := pioutil.NewLimitedBufferReader(r.Body, connReadLimitByte)
	b, err := ioutil.ReadAll(limitedr)
```