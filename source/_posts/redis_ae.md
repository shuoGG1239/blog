---
title: Redis源码阅读-事件模型ae
date: 2020/05/05
---
#### 源码文件
* src/ae.c


#### 入口函数
* `src/ae.c`下的`void aeMain(aeEventLoop *eventLoop)`函数; 推荐从这个函数开始阅读
```c
/*
 * 事件处理器的主循环
 */
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        // 如果有需要在事件处理前执行的函数，那么运行它
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        // 开始处理事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}

```
* 我们着重看下`aeMain`里面`aeProcessEvents(eventLoop, AE_ALL_EVENTS)`做了什么; 这里我们留意一下里面的`aeApiPoll`函数, 该函数用于`获取可执行的事件`, 获取之后在下面的for循环中处理事件, 执行事件处理器` fe->rfileProc(eventLoop,fd,fe->clientData,mask)`
*  `aeApiPoll`函数是ae模块提供的一个接口, 在`ae_epoll.c` `ae_kqueue.c` `ae_select.c` `ae_evport.c`都做了相应的具体实现, 也是所谓`IO多路复用`各平台的具体实现, 目的为了兼容不同平台
* 备注: 也许你会好奇为啥`IO多路复用`没有`iocp`的实现难道windows就没人权吗, 其实redis的官方版本是不支持windows的, windows版本在`https://github.com/microsoftarchive/redis`由微软团队自己维护, 里面就有`ae_wsiocp.c`即`iocp`版的实现
```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
        ... ...
        // 处理文件事件
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 从已就绪数组中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;
            // 读事件
            if (fe->mask & mask & AE_READABLE) {
                // rfired 确保读/写事件只能执行其中一个
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
        ... ...
}

```
* 通常说的redis的`reactor模型(反应堆)`其实说的就是`aeMain`的大循环中`aeProcessEvents`做的那些事情: `监听网络连接的FD的文件事件---> 获取事件---> 执行事件回调`
* 剩下具体细节不多赘述, 顺着思路看源码即可

#### 参考
* [Redis 和 IO 多路复用](https://www.cnblogs.com/john8169/p/9780484.html)
* [redis的事件模型详解(结合Reactor设计模式)](https://blog.csdn.net/gdj0001/article/details/80268836)