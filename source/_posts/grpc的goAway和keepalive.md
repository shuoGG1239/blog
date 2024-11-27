---
title: grpc的goAway和keepalive
date: 2023/08/02
categories: 
- 后端
tags:
- go
- grpc
---

#### 简介
* 虽是Http2的东西, 但也可以通过grpc的源码来侧面加深下理解


#### GoAway
* 告诉客户端, 服务端准备关闭了, 本连接不要发新请求过来了 (发一半的请求还是会处理完的)
* 当client收到这个包之后就会主动关闭连接。下次需要发送数据时，就会重新建立连接
* 流程: client收到Goaway -> client主动关闭http2连接 -> channel变为IDLE -> 用户发起新请求 -> 创建新连接 -> channel变为CONNECTING
* 另外GoAway是实现优雅关闭的基石, 因为是client主动关闭(不同于服务端关闭), 可以避免很多无效的请求 
* 源码: 
    - google.golang.org/grpc/clientconn.go: errConnDrain (drain是接收到goaway后的连接状态)
    - google.golang.org/grpc/internal/transport/http2_client.go: func (t *http2Client) Close(err error)
    - google.golang.org/grpc/internal/transport/http2_client.go: func (t *http2Client) handleGoAway(f *http2.GoAwayFrame)
        * 里面的 t.onClose(t.goAwayReason)的onClose在"grpc/clientconn.go/addrConn createTransport"里面定义的:

```go
func (ac *addrConn) createTransport(ctx context.Context, addr resolver.Address, copts transport.ConnectOptions, connectDeadline time.Time) error {
    addr.ServerName = ac.cc.getServerName(addr)
    hctx, hcancel := context.WithCancel(ctx) // hctx会深入到http2Client,见newHTTP2Client的信号"<-newClientCtx.Done()"

    // 客户端收到goAway的反馈: 关闭连接 + state变IDLE
    onClose := func(r transport.GoAwayReason) {
        ac.mu.Lock()
        defer ac.mu.Unlock()
        // adjust params based on GoAwayReason
        ac.adjustParams(r)
        if ctx.Err() != nil {
            // Already shut down or connection attempt canceled.  tearDown() or
            // updateAddrs() already cleared the transport and canceled hctx
            // via ac.ctx, and we expected this connection to be closed, so do
            // nothing here.
            return
        }
        hcancel() // 关闭http2连接!
        if ac.transport == nil {
            // We're still connecting to this address, which could error.  Do
            // not update the connectivity state or resolve; these will happen
            // at the end of the tryAllAddrs connection loop in the event of an
            // error.
            return
        }
        ac.transport = nil
        // Refresh the name resolver on any connection loss.
        ac.cc.resolveNow(resolver.ResolveNowOptions{})
        // Always go idle and wait for the LB policy to initiate a new
        // connection attempt.
        ac.updateConnectivityState(connectivity.Idle, nil)
    }
    connectCtx, cancel := context.WithDeadline(ctx, connectDeadline)
    defer cancel()
    copts.ChannelzParentID = ac.channelzID

    newTr, err := transport.NewClientTransport(connectCtx, ac.cc.ctx, addr, copts, onClose)
    ... ...
}
```

#### 关于keepalive
* keepalive在server端是默认开启的, client端默认关闭
* keepalive.ServerParameters.MaxConnectionIdle:
    - 如果一个client空闲超过15s, 发送一个 GOAWAY

* [grpc-keepalive-guide](https://github.com/grpc/grpc/blob/master/doc/keepalive.md)

* [RPC客户端长连接机制实现及keepalive分析](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/)
* 源码:
    - 结构: google.golang.org/grpc/internal/transport/http2_server.go: http2Server.kp
    - 结构: google.golang.org/grpc/internal/transport/http2_client.go: http2Client.kp
    - 实现: google.golang.org/grpc/internal/transport/http2_server.go: func (t *http2Server) keepalive()
    - 实现: google.golang.org/grpc/internal/transport/http2_client.go: func (t *http2Client) keepalive()


