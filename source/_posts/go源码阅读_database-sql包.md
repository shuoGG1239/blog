---
title: go源码阅读:database/sql包
date: 2022/03/27
categories: 
- 后端
tags:
- go
---

#### 简介
* `database/sql`最主要还是实现了连接池逻辑
* go源码版本: v1.16
* 源码仓库: [https://github.com/golang/go/tree/go1.16.6/src/database/sql](https://github.com/golang/go/tree/go1.16.6/src/database/sql)

#### sql.DB的一些关键逻辑
* Open会返回DB对象并开启一条connectionOpener线程
    - connectionOpener主要处理下面提到的"当前连接数大于maxOpen会陷入等待"的连接资源请求

* DB的核心方法: conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error)
    - 优先返回连接池里的连接
    - 当前连接数大于maxOpen会陷入等待, 等待取决于ctx, 也即由调用方控制
    - 其余情况则返回一个新连接

* DB的连接池: freeConn []*driverConn

* DB的核心方法: putConnDBLocked(conn, err) bool
    - 如果当前连接数大于maxOpen则直接返回false, 上层看到false则直接关闭该连接conn
    - 优先满足正在等待连接资源请求的(connRequests是一个map,可以看出请求连接资源并无优先级的说法)
    - 如果MaxIdleConns < len(freeConn), 即连接池满了, 则直接返回false, 上层看到false则直接关闭该连接conn
    - 否则丢到连接池freeConn中


* 清理线程(connectionCleaner):
    - SetConnMaxLifetime,SetConnMaxIdleTime时才会去起唯一的一条清理线程
    - 清理线程定期清理连接池freeConn的连接, 根据maxLifetime和createAt清理, 也根据maxIdleTime和returnedAt清理
    - 有趣的是清理线程并不理会MaxOpen和MaxIdleConns是多少, 只关注MaxLifetime和MaxIdleTime, 反正过期了就清理


* sql执行方法如Query,Exec,Ping等, 都会先去调conn获取连接, 再用其连接执行sql, 最后将putConnDBLocked(conn), (query是等rows全部scan完close再putConnDBLocked(conn))


* 关于resetSession最终的去处, 总的来说就是reset了个寂寞, 最终居然只是`conn.SetReadDeadline(time.Time{})`? 
```go
// go-sql-driver/mysql/connection.go里面的
// Write packet buffer 'data'
func (mc *mysqlConn) writePacket(data []byte) error {
    pktLen := len(data) - 4

    if pktLen > mc.maxAllowedPacket {
        return ErrPktTooLarge
    }

    // Perform a stale connection check. We only perform this check for
    // the first query on a connection that has been checked out of the
    // connection pool: a fresh connection from the pool is more likely
    // to be stale, and it has not performed any previous writes that
    // could cause data corruption, so it's safe to return ErrBadConn
    // if the check fails.
    if mc.reset {
        mc.reset = false
        conn := mc.netConn
        if mc.rawConn != nil {
            conn = mc.rawConn
        }
        var err error
        // If this connection has a ReadTimeout which we've been setting on
        // reads, reset it to its default value before we attempt a non-blocking
        // read, otherwise the scheduler will just time us out before we can read
        if mc.cfg.ReadTimeout != 0 {
            err = conn.SetReadDeadline(time.Time{})
        }
        if err == nil && mc.cfg.CheckConnLiveness {
            err = connCheck(conn)
        }
        if err != nil {
            errLog.Print("closing bad idle connection: ", err)
            mc.Close()
            return driver.ErrBadConn
        }
    }
    ... ...
}
```


#### 关于statement
1. 关于query带?的sql的逻辑: func (db *DB) queryDC:
    * 对于带了args的query, 且InterpolateParams=false, 则driver会返回driver.ErrSkip, 此时会走到si, err = ctxDriverPrepare(ctx, dc.ci, query), 做完prepare与ctxDriverStmtQuery, 将si放到Rows里面, Rows读完Close后si会跟着Close
