---
title: gh-ost源码分析
date: 2024/05/22
categories: 
- 后端
tags:
- go
- mysql
---
#### 简述
* 之前用到了gh-ost做大表改表工具, 回过来看看源码, 本篇为阅读源码的笔记

#### 源码信息
* 源码版本: 
* 源码仓库: [https://github.com/github/gh-ost](https://github.com/github/gh-ost)

#### gh-ost原理
* gh-ost 首先连接到主库上，根据 alter 语句创建幽灵表，然后作为一个"备库"连接到其中一个真正的备库上(默认设置,想连到master也行)

* 一边在主库上拷贝已有的数据到幽灵表，一边从备库上拉取增量数据的 binlog，然后不断的把 binlog 应用回主库

* 图中 cut-over 是最后一步，锁住主库的源表，等待 binlog 应用完毕，然后替换 gh-ost 表为源表

* gh-ost 在执行中，会在原本的 binlog event 里面增加以下 hint 和心跳包，用来控制整个流程的进度，检测状态等


#### gh-ost的改表流程
1. 检查有没有外键和触发器。
2. 检查表的主键信息。
3. 检查是否主库或从库，是否开启log_slave_updates，以及binlog信息  
4. 检查gho和del结尾的临时表是否存在
5. 创建ghc结尾的表，存数据迁移的信息，以及binlog信息等    
6. 初始化stream的连接,添加binlog的监听
7. 创建gho结尾的临时表，执行DDL在gho结尾的临时表上
8. 开启事务，按照主键id把源表数据写入到gho结尾的表上，再提交，以及binlog apply。
9. lock源表，rename 表：rename 源表 to 源_del表，gho表 to 源表。(这个过程叫cut-over)
10. 清理ghc表。


#### 关于v1.1.6修复的时区问题
* 在_gho表执行sql的session和binlog读取时指定的时区不一致导致
    - 从binlogEvent读取的时间结构体是带了时区的, 该时区是由BinlogParser.timestampStringLocation指定, 
    在转换成query时会用timestamp结合时区生成时间String
    
    - 强调: 不管是mysql底层还是binlog中, timestamp是不带时区的, 就是4个bytes; 

* github.com/go-mysql-org/go-mysql在时区强制指定utc, 新版本变成可配置并默认为系统时区, 所以是go-mysql没考虑兼容性导致

#### 源码
* /base包:
    - 相当于config, 处理配置信息和日志工具

* /sql包:
    - 相当于sql_parser, 处理sql解析的工具包

* /mysql包:
    - 相当于mysql相关的util包

* /binlog包:
    - 仅仅是对replication.BinlogSyncer的封装, 最后将replication.RowsEvent封装成BinlogEntry塞到EventsStreamer.eventsChannel里面

* /logic/server.go
    - 提供接口, 主要用于动态设置一些运行参数

* /logic/hooks.go:
    - 执行hook. 按规定的执行程序名字, 将执行程序放入指定目录, 后面会根据事件执行这些执行程序

* /logic/streamer.go:
    - 在上面binlog包的基础上再封装一层listener, listener处理上面提到的EventsStreamer.eventsChannel接收的BinlogEntry

* /logic/inspect.go:
    - 连接slave, 获取实例的基础信息如表结构, 表大小等, 检查改表是否符合迁移条件

* /logic/throttler.go:
    - 限流器, 调用throttle()会卡住以实现限流

* /logic/applier.go
    - gho和ghc的处理包括cutOver, 提供实现. 调用都在Migrator
    - ApplyDMLEventQueries(dmlEvents [](*binlog.BinlogDMLEvent)): 将binlogEvent转为query, 然后在_gho表执行

* /logic/migrator.go
    - 主流程. 上述各个模块提供的方法会在migrator中使用, 完成整个改表流程