- a `fallthrough` statement helps to merge 2 cases in a switch statement. after `fallthrough` from the 1st case, control is transfered to the case right below it.
- etcd/client/pkg/transport/listener.go
    - if opts has sockets opts, we set the ListenConfig, if opts has timeout config, we set Lister. if it has both, we set both. so to avoid duplicating, we merge the case having 1 option with case having both options.
    - we can put the Timeout first and fallthrough it instead of the socket case
```go
func newListener(addr, scheme string, opts ...ListenerOption) (net.Listener, error) {
    ...
	lnOpts := newListenOpts(opts...)

	switch {
	case lnOpts.IsSocketOpts():
		// new ListenConfig with socket options.
		config, err := newListenConfig(lnOpts.socketOpts)
		if err != nil {
			return nil, err
		}
		lnOpts.ListenConfig = config
		// check for timeout
		fallthrough
	case lnOpts.IsTimeout(), lnOpts.IsSocketOpts():
		// timeout listener with socket options.
		ln, err := newKeepAliveListener(&lnOpts.ListenConfig, addr)
		if err != nil {
			return nil, err
		}
		lnOpts.Listener = &rwTimeoutListener{
			Listener:     ln,
			readTimeout:  lnOpts.readTimeout,
			writeTimeout: lnOpts.writeTimeout,
		}
	case lnOpts.IsTimeout():
		ln, err := newKeepAliveListener(nil, addr)
		if err != nil {
			return nil, err
		}
		lnOpts.Listener = &rwTimeoutListener{
			Listener:     ln,
			readTimeout:  lnOpts.readTimeout,
			writeTimeout: lnOpts.writeTimeout,
		}
    ...
}
}
```

- etcd/client/v3/client.go
- the DeadlineExceeded case will be handled exactly the same with the Canceled case
- a if a || b statement will be enough but not as readable (due to more indentations)
```go
func toErr(ctx context.Context, err error) error {
    ....
	if ev, ok := status.FromError(err); ok {
		code := ev.Code()
		switch code {
		case codes.DeadlineExceeded:
			fallthrough
		case codes.Canceled:
			if ctx.Err() != nil {
				err = ctx.Err()
			}
		}
	}
	return err
}
```

- etcd/server/proxy/tcpproxy/userspace.go
- the case Priority < bestPr will reset w, weighted and unweighted. after that, these values will be handled exactly the same as the case Priority == bestPr becase bestPr is now set to Priority
```go
func (tp *TCPProxy) pick() *remote {
	for _, r := range tp.remotes {
		switch {
		case !r.isActive():
		case r.srv.Priority < bestPr:
			bestPr = r.srv.Priority
			w = 0
			weighted = nil
			unweighted = nil
			fallthrough
		case r.srv.Priority == bestPr:
			if r.srv.Weight > 0 {
				weighted = append(weighted, r)
				w += int(r.srv.Weight)
			} else {
				unweighted = append(unweighted, r)
			}
		}
	}
}

```

- etcd/client/v3/leasing
- err is nil but resp.Succeeded is not sure to be true. but if err is not nil, resp.Succeeded is sure to be false. so in case err != nil, we try to evict the key first and the handle exactly the same as resp.Succeeded is false.
```go
func (lkv *leasingKV) tryModifyOp(ctx context.Context, op v3.Op) (*v3.TxnResponse, chan<- struct{}, error) {
	key := string(op.KeyBytes())
	wc, rev := lkv.leases.Lock(key)
	cmp := v3.Compare(v3.CreateRevision(lkv.pfx+key), "<", rev+1)
	resp, err := lkv.kv.Txn(ctx).If(cmp).Then(op).Commit()
	switch {
	case err != nil:
		lkv.leases.Evict(key)
		fallthrough
	case !resp.Succeeded:
		if wc != nil {
			close(wc)
		}
		return nil, nil, err
	}
	return resp, wc, nil
}
```
