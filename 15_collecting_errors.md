- whenever an error happens, the function should return immediately. sometimes, it is better to collect all possible errors and decide to return later, especially when many things can go wrong at the same time. that helps debugging and keep the flow very clean

- gorilla/websocket/conn.go advanceFrame
```go
func (c *Conn) advanceFrame() (int, error) {
    ....

	var errors []string

	p, err := c.read(2)
	if err != nil {
		return noFrame, err
	}

	frameType := int(p[0] & 0xf)
	final := p[0]&finalBit != 0
	rsv1 := p[0]&rsv1Bit != 0
	rsv2 := p[0]&rsv2Bit != 0
	rsv3 := p[0]&rsv3Bit != 0
	mask := p[1]&maskBit != 0
	c.setReadRemaining(int64(p[1] & 0x7f))

	c.readDecompress = false
	if rsv1 {
		if c.newDecompressionReader != nil {
			c.readDecompress = true
		} else {
			errors = append(errors, "RSV1 set") // collecting errors
		}
	}

	if rsv2 {
		errors = append(errors, "RSV2 set") // collecting errors
	}

	if rsv3 {
		errors = append(errors, "RSV3 set") // collecting errors
	}

	switch frameType {
	case CloseMessage, PingMessage, PongMessage:
		if c.readRemaining > maxControlFramePayloadSize {
			errors = append(errors, "len > 125 for control") // collecting errors
		}
		if !final {
			errors = append(errors, "FIN not set on control") // collecting errors
		}
	case TextMessage, BinaryMessage:
		if !c.readFinal {
			errors = append(errors, "data before FIN") // collecting errors
		}
		c.readFinal = final
	case continuationFrame:
		if c.readFinal {
			errors = append(errors, "continuation after FIN") // collecting errors
		}
		c.readFinal = final
	default:
		errors = append(errors, "bad opcode "+strconv.Itoa(frameType)) // collecting errors
	}

	if mask != c.isServer {
		errors = append(errors, "bad MASK") // collecting errors
	}

	if len(errors) > 0 {
		return noFrame, c.handleProtocolError(strings.Join(errors, ", ")) // return after all errors collected
    }
}
```