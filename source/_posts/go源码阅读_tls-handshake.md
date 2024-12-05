---
title: "go源码阅读:TLS handshake"
date: 2023/04/25
categories: 
- 后端
tags:
- go
- tls
---

#### tls handshake
* tls/handshake_server.go: serverHandshakeState

* tls/common.go: Conn.PeerCertificates

* tls/handshake_client.go: doFullHandshake:
    - handshake读取到的certificateMsg最终会解析后赋值给peerCertificates
    - [rfc5246-Handshake Protocol](https://www.rfc-editor.org/rfc/rfc5246#section-7.4)

* tls/handshake_client.go: clientHandshake
    - makeClientHello
    - loadSession : 会生成sessionID
    - writeRecord : 将ClientHello发送给server
    - readHandshake: 等server回复

* tls/handshake_server.go: serverHandshake
    - readClientHello
    - processClientHello
    - pickCipherSuite
    - doFullHandshake
    - establishKeys
    - readFinished
    - sendSessionTicket
    - sendFinished

* tls.Conn: handlePostHandshakeMessage/handleKeyUpdate: KeyUpdate指的就是tls最后的那个对称密钥(变量名叫`trafficSecret`)


#### tls/conn.go
* tls.Conn实现了net.Conn接口, 在Write和Read包裹了一层handShake, 后面的读写也是带了加密的(readRecordOrCCS)


#### net/conn.go
```go
type Conn interface {
    Read(b []byte) (n int, err error)
    Write(b []byte) (n int, err error)
    Close() error
    LocalAddr() Addr
    RemoteAddr() Addr
    SetDeadline(t time.Time) error
    SetReadDeadline(t time.Time) error
    SetWriteDeadline(t time.Time) error
}
```