---
title: Redis数据迁移: RedisShake源码阅读
date: 2022/07/09
categories: 
- db
tags:
- go
- redis
---
#### 简述
* 近期用到了RedisShake做数据迁移, 源码代码量不多于是看了一遍, 本篇为阅读源码的笔记
* 本篇重点讲解RedisShake的数据迁移功能, 其他几个功能dump,decode,restore,rump只简单提及

#### 源码信息
* 源码版本: v1.6
* 源码仓库: [https://github.com/tair-opensource/RedisShake](https://github.com/tair-opensource/RedisShake)

#### 一. 进程入口:main
* `func main`为进程的入口, 核心源码如下: (为了只关注核心逻辑, 无关紧要的代码片段会用`// ... + 注释`代替)
* 简单来讲做了3件事情
    1. 初始化和校验配置参数
    2. 加进程锁 
        + 使用了第三方工具`github.com/nightlyone/lockfile`实现的
        + 简单讲就是在`conf.Options.PidPath`目录下建一个`{conf.Options.Id}.pid`文件来实现进程锁; 所以你可以扫描该目录下有多少个pid文件来确认当前有多少RedisShake进程在运行
    3. 跑数据迁移

```go
// main/main.go
func main() {
    // ... 各种初始化和校验, 如配置文件, 入参等 

    // 加进程锁, 进程id为`conf.Options.Id`, 入参时指定
    if err = utils.WritePidById(conf.Options.Id, conf.Options.PidPath); err != nil {
        crash(fmt.Sprintf("write pid failed. %v", err), -5)
    }

    // 根据需求选择对应的命令, 我们这里只关注数据迁移, 所以走run.CmdSync
    var runner base.Runner
    switch *tp {
    case conf.TypeDecode:
        runner = new(run.CmdDecode)  // 见decode.go
    case conf.TypeRestore: 
        runner = new(run.CmdRestore) // 见restore.go
    case conf.TypeDump:
        runner = new(run.CmdDump)    // 见dump.go
    case conf.TypeSync:
        runner = new(run.CmdSync)    // 见sync.go
    case conf.TypeRump:
        runner = new(run.CmdRump)    // 见rump.go
    }

    // 运行迁移任务
    runner.Main()
}

```

#### 二. 数据迁移入口: CmdSync
* 数据迁移的具体实现在 `sync.go/CmdSync.Main`, 核心源码如下:

* 简单来讲做了2件事情
    1. 建n个dbSyncer, 分配给m条线程
        - 默认情况下`n = m = len(SourceAddressList)`, 即和源地址相关, 但和源架构无关
        - 对于迁移目标为cluster架构, 所有dbSyncer用同一个dst, 即TargetAddressList
        - 对于迁移目标为standalone架构, 用roundRobin将TargetAddressList分配给各个DbSyncer

    2. 用dbSyncer完成数据迁移, 所以dbSyncer才是数据迁移的核心, 下小节讲解

```go
// sync.go
// 这个Main()就是上小节的`runner.Main()`
func (cmd *CmdSync) Main() {
    syncChan := make(chan syncNode, total)
    // dbSyncer是数据迁移的核心, 相当于worker
    cmd.dbSyncers = make([]*dbSyncer, total)

    for i, source := range conf.Options.SourceAddressList {
        var target []string
        if conf.Options.TargetType == conf.RedisTypeCluster {
            target = conf.Options.TargetAddressList
        } else {
            // round-robin pick
            pick := utils.PickTargetRoundRobin(len(conf.Options.TargetAddressList))
            target = []string{conf.Options.TargetAddressList[pick]}
        }

        nd := syncNode{
            id:             i,
            source:         source,
            sourcePassword: conf.Options.SourcePasswordRaw,
            target:         target,
            targetPassword: conf.Options.TargetPasswordRaw,
        }
        syncChan <- nd
    }

    var wg sync.WaitGroup
    wg.Add(len(conf.Options.SourceAddressList))

    for i := 0; i < int(conf.Options.SourceRdbParallel); i++ {
        go func() {
            for {
                nd, ok := <-syncChan
                if !ok {
                    break
                }

                ds := NewDbSyncer(nd.id, nd.source, nd.sourcePassword, nd.target, nd.targetPassword, conf.Options.HttpProfile+i)
                cmd.dbSyncers[nd.id] = ds
                go ds.sync()

                <-ds.waitFull // 阻塞直至全量同步完成

                wg.Done()
            }
        }()
    }

    wg.Wait() // 阻塞直至全量同步完成
    close(syncChan)

    // 永远阻塞
    select {}
}
```


#### 三. 数据迁移核心: dbSyncer
* 核心方法为3个:
    1. sendPSyncCmd: 读取全量+增量数据(pSync), 返回reader
    2. syncRDBFile: 从reader读取全量数据, 并同步(restore key)
    3. syncCommand: 从reader读取增量数据, 并同步(回放cmd)

* 这三个核心方法的具体逻辑如下:
```go
// sync.go

// 不阻塞
// 跑Psync, 跑一条全量+增量同步的goroutine, 用于不断读取全量+增量的数据,
// 全量和增量同步获取的实体数据最终都会Pipe到piper, 即return的那个pipe.Reader
func (ds *dbSyncer) sendPSyncCmd(master, auth_type, passwd string, tlsEnable bool) (pipe.Reader, int64) {

    // 1. 执行pSync
    c := utils.OpenNetConn(master, auth_type, passwd, tlsEnable)
    utils.SendPSyncListeningPort(c, conf.Options.HttpProfile)
    br := bufio.NewReaderSize(c, utils.ReaderBufferSize)
    bw := bufio.NewWriterSize(c, utils.WriterBufferSize)

    // 2. 首次pSync获取runid, offset, nsize, 后面的rdb数据会写入bw, 我们只需关注从br读取即可
    runid, offset, wait := utils.SendPSyncFullsync(br, bw)
    ds.targetOffset.Set(offset)
    var nsize int64 // nsize为rdb的大小
    for nsize == 0 {
        // 获取nsize为耗时操作, 通过wait管道通知获取(变量名换成waitForRdbSize好些)
        select {
        case nsize = <-wait:
            if nsize == 0 {
                log.Infof("dbSyncer[%v] +", ds.id)
            }
        case <-time.After(time.Second):
            log.Infof("dbSyncer[%v] -", ds.id)
        }
    }

    // br -> pipew -> piper(这个返回出去,作为ds.syncRDBFile的reader)
    piper, pipew := pipe.NewSize(utils.ReaderBufferSize)

    go func() {
        defer pipew.Close()
        p := make([]byte, 8192)
        // 3. 全量: 读取psync的数据直至读了nsize的数据, 最终写到piper(return出去那个)
        for rdbsize := int(nsize); rdbsize != 0; {
            // br -> pipew
            rdbsize -= utils.Iocopy(br, pipew, p, rdbsize)
        }

        // 4. 增量: 不断从psync读数据, 最终写到piper(return出去那个), 是一个死循环(除非异常)
        for {
            n, err := ds.pSyncPipeCopy(c, br, bw, offset, pipew) // 正常会在这里永远阻塞
            if err != nil {
                log.PanicErrorf(err, "dbSyncer[%v] psync runid = %s, offset = %d, pipe is broken",
                    ds.id, runid, offset)
            }
            // ... 后面是失败重试相关的操作, 这里不展示
        }
    }()
    return piper, nsize
}

// 阻塞至全量同步完成
// 从reader(这个就是上面sendPSyncCmd返回的reader)读出BinEntry, 再OpenRedisConn(target), 然后RestoreRdbEntry到target的redis节点
func (ds *dbSyncer) syncRDBFile(reader *bufio.Reader, target []string, auth_type, passwd string, nsize int64, tlsEnable bool) {
    pipe := utils.NewRDBLoader(reader, &ds.rbytes, base.RDBPipeSize)
    wait := make(chan struct{})
    go func() {
        defer close(wait)
        var wg sync.WaitGroup
        wg.Add(conf.Options.Parallel)
        // restore的时候可以并发, 从pipe里面取, 因为是全量的(按key进行restore), 所以没有顺序之分, 可以并发执行
        for i := 0; i < conf.Options.Parallel; i++ {
            go func() {
                defer wg.Done()
                c := utils.OpenRedisConn(target, auth_type, passwd, conf.Options.TargetType == conf.RedisTypeCluster,
                    tlsEnable)
                defer c.Close()
                var lastdb uint32 = 0
                for e := range pipe { // e是BinEntry, 全量同步的单位数据
                    // filterDB控制src, targetDB控制dst
                    if filter.FilterDB(int(e.DB)) {
                    } else {
                        // 这里执行selectDB选择同步的目标DB
                        // selectDB, 写这么多只是为了防止重复selectDB浪费性能
                        if conf.Options.TargetDB != -1 {
                            if conf.Options.TargetDB != int(lastdb) {
                                lastdb = uint32(conf.Options.TargetDB)
                                utils.SelectDB(c, uint32(conf.Options.TargetDB))
                            }
                        } else { // 如果不指定targetDB, 则源DB是啥, targetDB就是啥
                            if e.DB != lastdb {
                                lastdb = e.DB
                                utils.SelectDB(c, lastdb)
                            }
                        }

                        // 根据BinEntry的Key进行过滤
                        if filter.FilterKey(string(e.Key)) == true { // key白名单
                            ds.ignore.Incr()
                            continue
                        } else {
                            slot := int(utils.KeyToSlot(string(e.Key)))
                            if filter.FilterSlot(slot) == true { // slot白名单
                                ds.ignore.Incr()
                                continue
                            }
                        }
                        utils.RestoreRdbEntry(c, e) // restore 到 target
                    }
                }
            }()
        }

        wg.Wait()
    }()

    // 这会阻塞至<-wait信号, 即全量同步完成
    for done := false; !done; {
        select {
        case <-wait:
            done = true
        case <-time.After(time.Second):
        }
        // ... 后面都是统计和打日志逻辑, 这里不展示
    }
}


// 永远阻塞
func (ds *dbSyncer) syncCommand(reader *bufio.Reader, target []string, auth_type, passwd string, tlsEnable bool) {

    c := utils.OpenRedisConnWithTimeout(target, auth_type, passwd, readeTimeout, writeTimeout, isCluster, tlsEnable)
    defer c.Close()

    // ... 一大段FakeSlaveOffset相关逻辑, 给不支持pSync的用的, 这里不展示
    // ... 一大段统计相关逻辑, 这里不展示

    go func() {
        var (
            lastdb        int32 = 0
            bypass              = false
            isselect            = false
            scmd          string
            argv, newArgv [][]byte
            err           error
            reject        bool
        )

        decoder := redis.NewDecoder(reader)
        // 1. 读取reader, 解析出cmdDetail, 然后发送到sendBuf
        for {
            ignoresentinel:= false
            ignorecmd := false
            isselect = false
            resp := redis.MustDecodeOpt(decoder)

            // 这里是我精简后的代码
            // 根据scmd做一些过滤逻辑, 以及对当scmd为Select db时, 对target db的一些处理
            if scmd, argv, err = redis.ParseArgs(resp); err != nil {
            } else {
                if scmd != "ping" {
                    if strings.EqualFold(scmd, "select") {
                        s := string(argv[0])
                        n, err := strconv.Atoi(s)
                        bypass = filter.FilterDB(n)
                        isselect = true
                    } else if filter.FilterCommands(scmd) {
                        ignorecmd = true
                    }
                    if strings.EqualFold(scmd, "publish") && strings.EqualFold(string(argv[0]), "__sentinel__:hello"){
                        ignoresentinel = true
                    }
                    if bypass || ignorecmd || ignoresentinel{
                        ds.nbypass.Incr()
                        continue
                    }
                }
                newArgv, reject = filter.HandleFilterKeyWithCommand(scmd, argv)
                if bypass || ignorecmd || reject {
                    continue
                }
            }
            if isselect && conf.Options.TargetDB != -1 {
                if conf.Options.TargetDB != int(lastdb) {
                    lastdb = int32(conf.Options.TargetDB)
                    ds.sendBuf <- cmdDetail{Cmd: "SELECT", Args: [][]byte{[]byte(strconv.FormatInt(int64(lastdb), 10))}}
                }
                continue
            }
            ds.sendBuf <- cmdDetail{Cmd: scmd, Args: newArgv}
        }
    }()

    // 2. 从sendBuf读取出cmd, 回放到target (默认5000个cmd flush一次)
    go func() {
        var noFlushCount uint
        var cachedSize uint64

        for item := range ds.sendBuf {
            length := len(item.Cmd)
            data := make([]interface{}, len(item.Args))
            for i := range item.Args {
                data[i] = item.Args[i]
                length += len(item.Args[i])
            }
            err := c.Send(item.Cmd, data...) // 回放cmd
            noFlushCount += 1

            if noFlushCount >= conf.Options.SenderCount || cachedSize >= conf.Options.SenderSize ||
                    len(ds.sendBuf) == 0 { // 5000 ds in a batch
                err := c.Flush()
                noFlushCount = 0
                cachedSize = 0
            }
        }
    }()

    // 3. 永远阻塞, 每1s做一次统计
    for lstat := ds.Stat(); ; {
        time.Sleep(time.Second)
        nstat := ds.Stat()
        var b bytes.Buffer
        fmt.Fprintf(&b, "dbSyncer[%v] sync: ", ds.id)
        fmt.Fprintf(&b, " +forwardCommands=%-6d", nstat.forward-lstat.forward)
        fmt.Fprintf(&b, " +filterCommands=%-6d", nstat.nbypass-lstat.nbypass)
        fmt.Fprintf(&b, " +writeBytes=%d", nstat.wbytes-lstat.wbytes)
        log.Info(b.String())
        lstat = nstat
    }
}
```

#### 四. 总结
* 简单来说, 这个迁移原理就是假装为slave (利用pSync), 接收源redis的rdb然后restore到目标redis实现全量同步, 
然后继续接收源redis的增量cmd然后回放到目标redis实现增量同步

* 附官方架构图cluster版
![cluster](https://tair-opensource.github.io/RedisShake/architecture-c2c.svg)

* 附官方架构图standalone版
![standalone](https://tair-opensource.github.io/RedisShake/architecture-s2s.svg)