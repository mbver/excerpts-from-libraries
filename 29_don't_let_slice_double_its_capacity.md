- when a slice reaches its capacity, next append will double its size. that's can be a problem for memory management. expand the slice by fixed increment
- goyacc:
```go
	j := pstate[nstate+1]
	if j >= len(statemem) {
		asm := make([]Item, j+STATEINC)
		copy(asm, statemem)
		statemem = asm
	}
	statemem[j].pitem = p
```