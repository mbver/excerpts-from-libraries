- the ListenerOptions contains multiple internal fields that can be populated later
- ListenerOptions is not a list of ListenerOption
- ListenerOption is a function that sets a corresponding field of ListenerOptions
- the caller is passed with []ListenerOption
- should be called OptionSetter? or SetOption
```go
type ListenerOptions struct {
	Listener     net.Listener
	ListenConfig net.ListenConfig

	socketOpts       *SocketOpts
	tlsInfo          *TLSInfo
	skipTLSInfoCheck bool
	writeTimeout     time.Duration
	readTimeout      time.Duration
}

// these functions create ListenerOption, that sets a field of ListenerOptions
func WithTimeout(read, write time.Duration) ListenerOption {
	return func(lo *ListenerOptions) {
		lo.writeTimeout = write
		lo.readTimeout = read
	}
}

func WithSocketOpts(s *SocketOpts) ListenerOption {
	return func(lo *ListenerOptions) { lo.socketOpts = s }
}

func WithTLSInfo(t *TLSInfo) ListenerOption {
	return func(lo *ListenerOptions) { lo.tlsInfo = t }
}

func WithSkipTLSInfoCheck(skip bool) ListenerOption {
	return func(lo *ListenerOptions) { lo.skipTLSInfoCheck = skip }
}
```

- create []ListenerOption
```go
    opts: []ListenerOption{
        WithSocketOpts(&SocketOpts{ReuseAddress: true, ReusePort: true}),
        WithTLSInfo(tlsInfo),
    },

    opts: []ListenerOption{
        WithSocketOpts(&SocketOpts{ReusePort: true}),
        WithTLSInfo(tlsInfo),
        WithTimeout(5*time.Second, 5*time.Second),
    },

    opts: []ListenerOption{
        WithSocketOpts(&SocketOpts{ReusePort: true}),
        WithSkipTLSInfoCheck(true),
    },
```

- use ListenerOptions
```go
    ln, err := NewListenerWithOpts("127.0.0.1:0", test.scheme, test.opts...)

    // turn it into ListenerOptions
    lnOpts := newListenOpts(opts...)

    func newListenOpts(opts ...ListenerOption) *ListenerOptions {
        lnOpts := &ListenerOptions{}
        lnOpts.applyOpts(opts)
        return lnOpts
    }

    func (lo *ListenerOptions) applyOpts(opts []ListenerOption) {
        for _, opt := range opts {
            opt(lo)
        }
    }

```

