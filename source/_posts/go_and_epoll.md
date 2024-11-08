---
title: golang/net包与epoll
date: 2020/06/07
categories: 
- 后端
tags:
- go
---
#### net包与epoll
* linux下go的网络包底层如tcp也是采用epoll来实现, 你可以从`Accept`方法一路追下去, 追到尽头你会看到`internal/poll/fd_poll_runtime.go`里面这些在runtime实现的方法:
```go
func runtime_pollServerInit()
func runtime_pollOpen(fd uintptr) (uintptr, int)
func runtime_pollClose(ctx uintptr)
func runtime_pollWait(ctx uintptr, mode int) int
func runtime_pollWaitCanceled(ctx uintptr, mode int) int
func runtime_pollReset(ctx uintptr, mode int) int
func runtime_pollSetDeadline(ctx uintptr, d int64, mode int)
func runtime_pollUnblock(ctx uintptr)
func runtime_isPollServerDescriptor(fd uintptr) bool
```
* 此时到`src/runtime/netpoll.go`就能看到上述这些方法的实现, 再往下追下去就可以看到各个平台的具体实现了, 如`netpoll_epoll.go` `netpoll_kqueue.go` `netpoll_windows.go`, 看到`netpoll_epoll.go`里面的`epollcreate, epollctl, epollwait`了吧, 多么熟悉的几个函数!

#### goroutine与epoll
* 虽然net包底层用epoll实现了, 但是实际我们在用tcp还是开goroutine来serve
* net包就是推荐我们用goroutine来玩tcp, 应对大部分场景妥妥的
* 面对比较变态的场景并发量贼高时, goroutine尽管只有消耗2k~8k的栈空间, 连接一多还是耗不起, 此时就只能用一些黑魔法来使用epoll了
* 具体怎么玩可以参照 [https://github.com/mailru/easygo](https://github.com/mailru/easygo)