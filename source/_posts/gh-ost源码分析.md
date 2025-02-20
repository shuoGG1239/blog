---
title: gh-ost源码分析
date: 2024/05/22
categories: 
- db
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

#### 源码结构
* /cmd包: 入口
    - /cmd/gh-ost/cmd.go: 入口

* /base包: 相当于config, 处理配置信息和日志工具
    - /base/context.go: MigrationContext的定义和各种Get/Set方法, 不重要

    - /base/default_logger.go: 标准输出logger封装, 不重要

    - /base/load_map.go: 解析k1=v1,k2=v2到map, 不重要

    - /base/utils.go: 标准工具包, 不重要

* /sql包: 相当于sql_parser, 处理sql解析的工具包
    - /sql/builder.go: 构建带`/* gh-ost xxx.tbl */`的sql

    - /sql/encoding.go: charset定义, 不重要

    - /sql/parser.go: AlterTable特供parser (gh-ost没有引用其他sqlparser包, 很干净但也仅支持了alter table)

    - /sql/types.go: Column相关结构的封装, 不重要

* /mysql包: 相当于mysql相关的util包
    - /mysql/binlog.go: BinlogCoordinates结构(logfile,logPos), 不重要

    - /mysql/connection.go: mysql的ConnectionConfig结构, 不重要

    - /mysql/instance_key.go: InstanceKey结构(host,port), 不重要

    - /mysql/instance_key_map.go: 给上面的InstanceKey套了层map, 不重要

    - /mysql/utils.go: 不重要

* /binlog包: 仅仅是对replication.BinlogSyncer的封装, 最后将replication.RowsEvent封装成BinlogEntry塞到EventsStreamer.eventsChannel里面
    - /binlog/binlog_dml_event.go: BinlogDMLEvent结构, 不重要

    - /binlog/binlog_entry.go: BinlogEntry结构, 不重要

    - /binlog/binlog_reader.go: BinlogReader接口定义, 不重要

    - /binlog/gomysql_reader.go: 基于go-mysql/replication封装了自己的binlogReader

* /logic包: 核心流程都在这个包
    - /logic/server.go: 提供接口, 用于动态设置一些运行参数和查看任务状态 (不重要)

    - /logic/hooks.go: hook执行器. 按规定的执行程序名字, 将执行程序(如sh脚本)放入指定目录, 后面会根据事件执行这些执行程序. (不重要)

    - /logic/streamer.go: 在上面binlog包的基础上再封装一层listener, listener处理上面提到的EventsStreamer.eventsChannel接收的BinlogEntry

    - /logic/inspect.go: 连接slave, 获取实例的基础信息如表结构, 表大小等, 检查改表是否符合迁移条件

    - /logic/throttler.go: 限流器, 调用throttle()实现限流 (符合限流条件时该函数会卡住否则不卡)
        + 根据多种条件（如HTTP,freno状态、复制延迟、系统负载等）判断是否需要进行限流

    - /logic/applier.go
        - gho和ghc的处理包括cutOver, 提供实现. 调用都在Migrator
        - ApplyDMLEventQueries(dmlEvents [](*binlog.BinlogDMLEvent)): 将binlogEvent转为query, 然后在_gho表执行

    - /logic/migrator.go: 主流程. 上述各个模块提供的方法会在migrator中使用, 完成整个改表流程


#### 改表流程源码
* 改表流程直接从`/logic/migrator.go`的`func (this *Migrator) Migrate()`开始看, 核心逻辑如下
```go
func (this *Migrator) Migrate() (err error) {
    // 初始化binlogReader, 用于读取binlog
    if err := this.initiateStreaming(); err != nil {
        return err
    }
    // 初始化Applier, 影子表的操作基本都在applier中进行
    if err := this.initiateApplier(); err != nil {
        return err
    }
    // 若支持instant改表则直接用instant改表完成大表改表操作
    if this.migrationContext.AttemptInstantDDL {
        this.migrationContext.Log.Infof("Attempting to execute alter with ALGORITHM=INSTANT")
        if err := this.applier.AttemptInstantDDL(); err == nil {
            this.migrationContext.Log.Infof("Success! table %s.%s migrated instantly", sql.EscapeName(this.migrationContext.DatabaseName), sql.EscapeName(this.migrationContext.OriginalTableName))
            return nil
        } else {
            this.migrationContext.Log.Infof("ALGORITHM=INSTANT not supported for this operation, proceeding with original algorithm: %s", err)
        }
    }

    // 抓binlogEvent, 最终在this.executeWriteFuncs消费
    if err := this.addDMLEventsListener(); err != nil {
        return err
    }

    // 确定row copy的边界值, 其实就是通过"select uk order by uk limit 1 desc/asc"来获取
    if err := this.applier.ReadMigrationRangeValues(); err != nil {
        return err
    }
    go this.executeWriteFuncs()   // 消费来自this.iterateChunks的chunkCopy任务和this.addDMLEventsListener的binlogEvent
    go this.iterateChunks()       // 生产chunk copy任务, 在self.executeWriteFuncs消费
    this.consumeRowCopyComplete() // 等待row copy完成, 信号来自this.iterateChunks

    // cutOver: rename gho表
    var retrier func(func() error, ...bool) error
    if err := retrier(this.cutOver); err != nil {
        return err
    }
    // 清理阶段: drop ghc表
    if err := this.finalCleanup(); err != nil {
        return nil
    }
    return nil
}
```