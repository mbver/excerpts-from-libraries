- hashicorp/memberlist/util
- the compound message is encoded into a byte sequence:
    - num of msgs + len1 + len2 + .. lenN + msg1 + msg2 + .. msgN
- everytime we extract a value in the sequence we, slice it off the buffer slice.
```go
func decodeCompoundMessage(buf []byte) (trunc int, parts [][]byte, err error) {
	if len(buf) < 1 {
		err = fmt.Errorf("missing compound length byte")
		return
	}
	numParts := uint8(buf[0])
	buf = buf[1:] // slice numParts off

	// Check we have enough bytes
	if len(buf) < int(numParts*2) {
		err = fmt.Errorf("truncated len slice")
		return
	}

	// Decode the lengths
	lengths := make([]uint16, numParts)
	for i := 0; i < int(numParts); i++ {
		lengths[i] = binary.BigEndian.Uint16(buf[i*2 : i*2+2])
	}
	buf = buf[numParts*2:] // slice lengths off

	// Split each message
	for idx, msgLen := range lengths {
		if len(buf) < int(msgLen) {
			trunc = int(numParts) - idx
			return
		}

		// Extract the slice, seek past on the buffer
		slice := buf[:msgLen]
		buf = buf[msgLen:] // slice msgI off
		parts = append(parts, slice)
	}
	return
}
```