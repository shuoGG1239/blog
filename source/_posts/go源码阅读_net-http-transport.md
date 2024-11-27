---
title: go源码阅读:net/http的Transport
date: 2022/03/20
categories: 
- 后端
tags:
- go
---

#### 简介
* `http.Transport`这样的高频使用模块, 源码肯定得看看
* go源码版本: v1.16
* 源码仓库: [https://github.com/golang/go/blob/go1.16.6/src/net/http/transport.go](https://github.com/golang/go/blob/go1.16.6/src/net/http/transport.go)


#### http.Transport的关键逻辑
* Transport是RoundTripper接口的实现: func RoundTrip(req *Request) (*Response, error)

* Transport对外只提供方法: func RoundTrip(req *Request) (*Response, error)

* Transport的内部对象idleLRU connLRU写得不错, 简单实现了LRU

* http.Client就是在Tranport上简单封装一层

* Transport就是一个连接池, 池子里面放着persistConn连接对象(idleConn map[connectMethodKey][]*persistConn)

* queueForIdleConn: 根据请求的connectMethodKey从t.idleConn获取一个[]*persistConn切片， 并从切片中，根据算法获取一个有效的空闲连接。如果未获取到空闲连接，则将wantConn结构体放入t.idleConnWait[w.key]等待队列

* 连接释放逻辑在 (t *Transport) tryPutIdleConn(pconn *persistConn)
    - 哪些情况才回去调 tryPutIdleConn:
        1. 大部分的异常情况
        2. responseBody read完: 代码详细见 case bodyEOF := <-waitForBodyRead

* dialConnFor: 会调用t.dialConn获取一个真正的*persistConn。并将这个连接传递给w, 如果w已经获取到了连接，则会传递失败，此时调用t.putOrCloseIdleConn将连接放回空闲连接池。

* dialConn: 
    1. 调用t.dial(ctx, "tcp", cm.addr())创建TCP连接并将其赋予刚new的persistConn
    2. 如果是https的请求，则对请求建立安全的tls传输通道
    3. 为persistConn创建读写buffer，如果用户没有自定义读写buffer的大小，读写bufffer的大小默认为4096
    4. 执行go pconn.readLoop()和go pconn.writeLoop()开启读写循环然后返回连接

* dialConn里面这段代码是开启http2的核心
```go
// 当client和server都支持http2时，s.NegotiatedProtocol的值为"h2"且s.NegotiatedProtocolIsMutual的值为true
// pconn.tlsState是在pconn.addTLS中附加的
// 所以是否支持http2是在tls握手时得知的, 因此http2时强制要求https
if s := pconn.tlsState; s != nil && s.NegotiatedProtocolIsMutual && s.NegotiatedProtocol != "" {
    if next, ok := t.TLSNextProto[s.NegotiatedProtocol]; ok {
        alt := next(cm.targetAddr, pconn.conn.(*tls.Conn))
        if e, ok := alt.(erringRoundTripper); ok {
            // pconn.conn was closed by next (http2configureTransports.upgradeFn).
            return nil, e.RoundTripErr()
        }
        return &persistConn{t: t, cacheKey: pconn.cacheKey, alt: alt}, nil
    }
}
```


#### 关于persistConn
* persistConn是在给"conn net.Conn"包一层

* readLoop: for循环, 不停等待新的requestAndChan(由roundTrip发起), response塞回requestAndChan(rc.ch <- responseAndError{res: resp}); 确认读完之后调tryPutIdleConn放回Transport的连接池
    - 只有当调用方完整的读取了响应，该连接才能够被复用。
    - 因此在http1.1中，1个连接上的请求，只有等前一个请求处理完之后才能继续下一个请求。
    - 如果前面的请求处理较慢， 则后面的请求必须等待， 这就是http1.1中的线头阻塞
    - 所以就算你不关心response的body, 也必须把body读完以保持连接的复用, 可以如下处理
        + io.CopyN(ioutil.Discard, resp.Body, 2 << 10)
        + resp.Body.Close()

* writeLoop: for循环, 不停等待新的writeRequest, 写完发信号给pc.writeErrCh和wr.ch, 出错了会关闭该persistConn并结束writeLoop的循环, 否则继续等待新的writeRequest

* roundTrip: 在for循环里面等待本次roundTrip的各种信号, 如来自writeLoop的写完成信号, pcClosed, cancelChan, response结果信号等, 收到response或者出错则结束循环. 只有在出错的时候才会pc.close; roundTrip里面没有处理连接池的逻辑



#### 关于http.Response
* http/response.go: 核心函数 func ReadResponse(r *bufio.Reader, req *Request) (*Response, error)
    - 关于HeaderTimeout, 源码可看出是读完所有header才算header读结束了(blank line), HeaderTimeout是header读结束的timeout
    - response.wroteHeader: writeHeader里面会将wroteHeader置为true
    - func (cw *chunkWriter) writeHeader(p []byte)的最后一行是: w.conn.bufw.Write(crlf)
    - response的flush本质就是writeHeader(nil), 也就是最后也会w.conn.bufw.Write(crlf)


#### 关于http.DefaultTransport
```go
var DefaultTransport RoundTripper = &Transport{
    Proxy: ProxyFromEnvironment, // "HTTP_PROXY","http_proxy","HTTPS_PROXY","https_proxy","NO_PROXY","no_proxy"
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
    }).DialContext,
    ForceAttemptHTTP2:     true,
    MaxIdleConns:          100,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
```
