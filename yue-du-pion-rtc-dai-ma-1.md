---
description: 看一下整体的收包流程
---

# 阅读pion RTC 代码1

```go
// srtp/v2/session.go
func (s *session) start(localMasterKey, localMasterSalt, remoteMasterKey, remoteMasterSalt []byte, profile ProtectionProfile, child streamSession) error {
	var err error
	s.localContext, err = CreateContext(localMasterKey, localMasterSalt, profile, s.localOptions...)
	if err != nil {
		return err
	}

	s.remoteContext, err = CreateContext(remoteMasterKey, remoteMasterSalt, profile, s.remoteOptions...)
	if err != nil {
		return err
	}

	go func() {
		defer func() {
			close(s.newStream)

			s.readStreamsLock.Lock()
			s.readStreamsClosed = true
			s.readStreamsLock.Unlock()
			close(s.closed)
		}()

		// 这里是最底层的逻辑
		// 从 nextConn 读取数据 然后解密
		// 推测数据从child 传到上层
		b := make([]byte, 8192)
		for {
			var i int
			i, err = s.nextConn.Read(b)
			if err != nil {
				if err != io.EOF {
					s.log.Error(err.Error())
				}
				return
			}

			if err = child.decrypt(b[:i]); err != nil {
				s.log.Info(err.Error())
			}
		}
	}()

	close(s.started)

	return nil
}

```

这里需要弄明白两个东西

1. nextConn 是什么
2. child 解密做了什么更重要的是 解密之后的数据传到哪了

经过向上溯源，可以看到 nextConn 是 iceTransport生成的endpoint，可以暂时理解为从ice层获取数据

```go
// func (t *DTLSTransport) Start(remoteParameters DTLSParameters) error
t.srtpEndpoint = t.iceTransport.NewEndpoint(mux.MatchSRTP)
t.srtcpEndpoint = t.iceTransport.NewEndpoint(mux.MatchSRTCP)
t.remoteParameters = remoteParameters

// func (t *DTLSTransport) startSRTP() error  
srtpSession, err := srtp.NewSessionSRTP(t.srtpEndpoint, srtpConfig)
if err != nil {
	return fmt.Errorf("%w: %v", errFailedToStartSRTP, err)
}
```

接下来，看一下`child.decrypt()` 做了啥

```go
func (s *SessionSRTP) decrypt(buf []byte) error {
	h := &rtp.Header{}
	if err := h.Unmarshal(buf); err != nil {
		return err
	}
	
	// 这里找ssrc对应的stream，ssrc是sdp提前协商的
	// 找不到会创建新的，并通知上层有sdp里面没有协商的ssrc
	r, isNew := s.session.getOrCreateReadStream(h.SSRC, s, newReadStreamSRTP)
	if r == nil {
		return nil // Session has been closed
	} else if isNew {
		s.session.newStream <- r // Notify AcceptStream
	}
	
	// down_cast
	readStream, ok := r.(*ReadStreamSRTP)
	if !ok {
		return errFailedTypeAssertion
	}

	// 解密
	decrypted, err := s.remoteContext.decryptRTP(buf, buf, h)
	if err != nil {
		return err
	}

	// 传递数据给ssrc stream处理
	_, err = readStream.write(decrypted)
	if err != nil {
		return err
	}

	return nil
}

// 这里不是简单的write到buffer
// 其实同时通知了在读的用户
func (r *ReadStreamSRTP) write(buf []byte) (n int, err error) {
	n, err = r.buffer.Write(buf)

	if errors.Is(err, packetio.ErrFull) {
		// Silently drop data when the buffer is full.
		return len(buf), nil
	}

	return n, err
}

```
