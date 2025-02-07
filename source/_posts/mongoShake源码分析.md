---
title: "Mongodb数据迁移:MongoShake源码阅读"
date: 2023/04/09
categories: 
- db
tags:
- go
- mongodb
---
#### 简述
* 近期用到了MongoShake做数据迁移, 顺便看看源码, 本篇为阅读源码的笔记
* 本文只讲数据迁移这块相关的


#### 原理及架构
* 架构 ![mongoshake-arch](https://github.com/alibaba/MongoShake/blob/develop/resources/dataflow.png)


#### 源码信息
* 源码版本: v2.8.1
* 源码仓库: [https://github.com/alibaba/MongoShake](https://github.com/alibaba/MongoShake)
* 本文贴的源码片段为了只关注核心逻辑, 无关紧要的代码段会干掉或注释代替


#### 一. 进程入口:
* `collector`的`func main`为进程的入口, 核心源码如下:
* 简单来讲做了3件事情
    1. 初始化和校验配置参数
    2. 加进程锁 
        + 使用了第三方工具`github.com/nightlyone/lockfile`实现的
        + 简单讲就是在`conf.Options.LogDirectory`目录下建一个`{conf.Options.Id}.pid`文件来实现进程锁; 所以你可以扫描该目录下有多少个pid文件来确认当前有多少MongoShake进程在运行
    3. 开始跑数据迁移(startup)


```golang
// cmd/collector/collector.go
// 初始化ReplicationCoordinator然后Run
func main() {
    var err error
    defer handleExit()
    defer LOG.Close()
    defer utils.Goodbye()

    // argument options
    configuration := flag.String("conf", "a.conf", "configure file absolute path")
    verbose := flag.Int("verbose", 0, "where log goes to: 0 - file，1 - file+stdout，2 - stdout")
    version := flag.Bool("version", false, "show version")
    flag.Parse()

    var file *os.File
    if file, err = os.Open(*configuration); err != nil {
        crash(fmt.Sprintf("Configure file open failed. %v", err), -1)
    }
    defer file.Close()

    // read fcv and do comparison
    if _, err := conf.CheckFcv(*configuration, utils.FcvConfiguration.FeatureCompatibleVersion); err != nil {
        crash(err.Error(), -5)
    }

    configure := nimo.NewConfigLoader(file)
    configure.SetDateFormat(utils.GolangSecurityTime)
    if err := configure.Load(&conf.Options); err != nil {
        crash(fmt.Sprintf("Configure file %s parse failed. %v", *configuration, err), -2)
    }

    // verify collector options and revise
    if err = SanitizeOptions(); err != nil {
        crash(fmt.Sprintf("Conf.Options check failed: %s", err.Error()), -4)
    }

    conf.Options.Version = utils.BRANCH

    utils.Mkdirs(conf.Options.LogDirectory)
    // get exclusive process lock and write pid
    if utils.WritePidById(conf.Options.LogDirectory, conf.Options.Id) {
        startup()
    }
}

func startup() {
    // leader election at the beginning
    selectLeader()

    // ReplicationCoordinator即迁移任务, 包含全量和增量
    coordinator := &coordinator.ReplicationCoordinator{
        MongoD: make([]*utils.MongoSource, len(conf.Options.MongoUrls)),
    }

    // init
    for i, src := range conf.Options.MongoUrls {
        coordinator.MongoD[i] = new(utils.MongoSource)
        coordinator.MongoD[i].URL = src
        if len(conf.Options.IncrSyncOplogGIDS) != 0 {
            coordinator.MongoD[i].Gids = conf.Options.IncrSyncOplogGIDS
        }
    }
    if conf.Options.MongoSUrl != "" {
        coordinator.MongoS = &utils.MongoSource{
            URL:         conf.Options.MongoSUrl,
            ReplicaName: "mongos",
        }
        coordinator.RealSourceFullSync = []*utils.MongoSource{coordinator.MongoS}
        coordinator.RealSourceIncrSync = []*utils.MongoSource{coordinator.MongoS}
        if conf.Options.IncrSyncMongoFetchMethod == utils.VarIncrSyncMongoFetchMethodOplog {
            coordinator.RealSourceIncrSync = coordinator.MongoD
        }
    } else {
        coordinator.RealSourceFullSync = coordinator.MongoD
        coordinator.RealSourceIncrSync = coordinator.MongoD
    }

    if conf.Options.MongoCsUrl != "" {
        coordinator.MongoCS = &utils.MongoSource{
            URL: conf.Options.MongoCsUrl,
        }
    }

    // start mongodb replication
    if err := coordinator.Run(); err != nil {
        // initial or connection established failed
        LOG.Critical(fmt.Sprintf("run replication failed: %v", err))
        crash(err.Error(), -6)
    }

    // if the sync mode is "document", mongoshake should exit here.
    if conf.Options.SyncMode == utils.VarSyncModeFull {
        return
    }

    // do not exit
    select {}
}
```

* 上面的`startup`里面的`coordinator.Run()`会根据`syncMode`走`全量`,`增量`或`全量+增量`
```golang
    switch syncMode {
    case utils.VarSyncModeAll: // 全量+增量
        if conf.Options.FullSyncReaderOplogStoreDisk {
            LOG.Info("run parallel document oplog")
            if err := coordinator.parallelDocumentOplog(fullBeginTs); err != nil {
                return err
            }
        } else {
            LOG.Info("run serialize document oplog")
            if err := coordinator.serializeDocumentOplog(fullBeginTs); err != nil {
                return err
            }
        }
    case utils.VarSyncModeFull: // 全量
        if err := coordinator.startDocumentReplication(); err != nil {
            return err
        }
    case utils.VarSyncModeIncr: // 增量
        if err := coordinator.startOplogReplication(int64(0), int64(0), startTsMap); err != nil {
            return err
        }
    default:
        LOG.Critical("unknown sync mode %v", conf.Options.SyncMode)
        return errors.New("unknown sync mode " + conf.Options.SyncMode)
    }
```

#### 二. 全量同步
* 根据上面可知道, 全量同步的入口在`ReplicationCoordinator.startDocumentReplication`, 主要分为3步:
    1. 获取同步源的所有表, 删除同步目标对应的表
        + 上面说的`表`实际在源码中是ns(namespace), 格式为f'{database}.{collection}', 后面会经常提到

    2. 同步所有索引数据 (Option为后台索引时是放在这一步, 如果是前台索引则是同步db后才执行同步)
        + 查询源库getIndexes, 然后在目标库createIndex

    3. 同步db(collection)数据
        + 多线程同步, 线程数取决源数据地址和架构
        + mongos只算一个地址即一条线程, 多线程是针对mongod的情况
        + dbSyncer是同步的核心, 一条线程对应一个dbSyncer, 负责单个db实例的全量同步工作, 所以通常情况下只有1个dbSyncer

```golang
// collector/coordinator/full.go
func (coordinator *ReplicationCoordinator) startDocumentReplication() error {

    fromIsSharding := coordinator.SourceIsSharding()

    var shardingChunkMap sharding.ShardingChunkMap
    var err error
    // init orphan sharding chunk map if source is mongod(get data directly from mongod)
    if fromIsSharding && coordinator.MongoS == nil {
        LOG.Info("source is mongod, need to fetching chunk map")
        shardingChunkMap, err = fetchChunkMap(fromIsSharding)
        if err != nil {
            LOG.Critical("fetch chunk map failed[%v]", err)
            return err
        }
    } else {
        LOG.Info("source is replica or mongos, no need to fetching chunk map")
    }

    filterList := filter.NewDocFilterList()
    // get all namespace need to sync
    nsSet, _, err := utils.GetAllNamespace(coordinator.RealSourceFullSync, filterList.IterateFilter,
        conf.Options.MongoSslRootCaFile)
    if err != nil {
        return err
    }
    LOG.Info("all namespace: %v", nsSet)

    var ckptMap map[string]utils.TimestampNode
    if conf.Options.SpecialSourceDBFlag != utils.VarSpecialSourceDBFlagAliyunServerless && len(coordinator.MongoD) > 0 {
        // get current newest timestamp
        ckptMap, err = getTimestampMap(coordinator.MongoD, conf.Options.MongoSslRootCaFile)
        if err != nil {
            return err
        }
    }

    // create target client
    toUrl := conf.Options.TunnelAddress[0]
    var toConn *utils.MongoCommunityConn
    if !conf.Options.FullSyncExecutorDebug {
        if toConn, err = utils.NewMongoCommunityConn(toUrl, utils.VarMongoConnectModePrimary, true,
            utils.ReadWriteConcernLocal, utils.ReadWriteConcernDefault, conf.Options.TunnelMongoSslRootCaFile); err != nil {
            return err
        }
        defer toConn.Close()
    }

    // create namespace transform
    trans := transform.NewNamespaceTransform(conf.Options.TransformNamespace)

    // drop target collection if possible
    if err := docsyncer.StartDropDestCollection(nsSet, toConn, trans); err != nil {
        return err
    }

    // enable shard if sharding -> sharding
    shardingSync := docsyncer.IsShardingToSharding(fromIsSharding, toConn)
    if shardingSync {
        var connString string
        if len(conf.Options.MongoSUrl) > 0 {
            connString = conf.Options.MongoSUrl
        } else {
            connString = conf.Options.MongoCsUrl
        }
        if err := docsyncer.StartNamespaceSpecSyncForSharding(connString, toConn, trans); err != nil {
            return err
        }
    }

    // 同步所有索引数据
    // fetch all indexes
    var indexMap map[utils.NS][]bson.D // Ns即namespace, 说白了就是 f'{database}.{collection}'
    if conf.Options.FullSyncCreateIndex != utils.VarFullSyncCreateIndexNone {
        if indexMap, err = fetchIndexes(coordinator.RealSourceFullSync, filterList.IterateFilter); err != nil {
            return fmt.Errorf("fetch index failed[%v]", err)
        }

        // print
        LOG.Info("index list below: ----------")
        for ns, index := range indexMap {
            // LOG.Info("collection[%v] -> %s", ns, utils.MarshalStruct(index))
            LOG.Info("collection[%v] -> %v", ns, index)
        }
        LOG.Info("index list above: ----------")

        // 后台索引, 就是简单的查询源库getIndexes, 然后在目标库createIndex
        if conf.Options.FullSyncCreateIndex == utils.VarFullSyncCreateIndexBackground {
            if err := docsyncer.StartIndexSync(indexMap, toUrl, trans, true); err != nil {
                return fmt.Errorf("create background index failed[%v]", err)
            }
        }
    }

    // global qps limit, all dbsyncer share only 1 Qos
    qos := utils.StartQoS(0, int64(conf.Options.FullSyncReaderDocumentBatchSize), &utils.FullSentinelOptions.TPS)

    // 同步db数据, 配置文件填了几个源地址就开几条线程(mongos只算一个地址即一条线程, 多线程是针对mongod的情况)
    // start sync each db
    var wg sync.WaitGroup
    var replError error
    for i, src := range coordinator.RealSourceFullSync {
        var orphanFilter *filter.OrphanFilter
        if conf.Options.FullSyncExecutorFilterOrphanDocument && shardingChunkMap != nil {
            dbChunkMap := make(sharding.DBChunkMap)
            if chunkMap, ok := shardingChunkMap[src.ReplicaName]; ok {
                dbChunkMap = chunkMap
            } else {
                LOG.Warn("document syncer %v has no chunk map", src.ReplicaName)
            }
            orphanFilter = filter.NewOrphanFilter(src.ReplicaName, dbChunkMap)
        }

        dbSyncer := docsyncer.NewDBSyncer(i, src.URL, src.ReplicaName, toUrl, trans, orphanFilter, qos, fromIsSharding)
        dbSyncer.Init()
        LOG.Info("document syncer-%d do replication for url=%v", i, src.URL)

        wg.Add(1)
        nimo.GoRoutine(func() {
            defer wg.Done()
            if err := dbSyncer.Start(); err != nil {
                LOG.Critical("document replication for url=%v failed. %v",
                    utils.BlockMongoUrlPassword(src.URL, "***"), err)
                replError = err
            }
            dbSyncer.Close()
        })
    }

    wg.Wait()
    if replError != nil {
        return replError
    }

    // 完成同步后创建前台索引, 后台索引是同步数据前创建
    // create index if == foreground
    if conf.Options.FullSyncCreateIndex == utils.VarFullSyncCreateIndexForeground {
        if err := docsyncer.StartIndexSync(indexMap, toUrl, trans, false); err != nil {
            return fmt.Errorf("create forground index failed[%v]", err)
        }
    }

    // update checkpoint after full sync
    // do not update checkpoint when source is "aliyun_serverless"
    if conf.Options.SyncMode != utils.VarSyncModeFull && conf.Options.SpecialSourceDBFlag != utils.VarSpecialSourceDBFlagAliyunServerless {
        // need merge to one when from mongos and fetch_mothod=="change_stream"
        if coordinator.MongoS != nil && conf.Options.IncrSyncMongoFetchMethod == utils.VarIncrSyncMongoFetchMethodChangeStream {
            var smallestNew int64 = math.MaxInt64
            for _, val := range ckptMap {
                if smallestNew > val.Newest {
                    smallestNew = val.Newest
                }
            }
            ckptMap = map[string]utils.TimestampNode{
                coordinator.MongoS.ReplicaName: {
                    Newest: smallestNew,
                },
            }
        }

        /*
        eg:
        map[map[shard1_servers:{Oldest:7329804871119405057 Newest:7431129274255409154}
                shard2_servers:{Oldest:7398473718780919810 Newest:7431129274255409153}
                shard3_servers:{Oldest:7417453196441813035 Newest:7431129278550376449}]]
        
        这个长数字其实时间戳, 只不过是 `ts << 32 + cnt`, oplog就是这样的格式的
        */
        LOG.Info("try to set checkpoint with map[%v]", ckptMap)
        if err := docsyncer.Checkpoint(ckptMap); err != nil {
            return err
        }
    }

    LOG.Info("document syncer sync end")
    return nil
}
```

##### 全量同步的核心: dbSyncer
* `dbSyncer`负责单个db的全量同步工作, 同步逻辑如下的`Start`:
    - 将同步任务细分为`nsList`, `ns`即namespace, 其实就是`collection`
    - 开n条线程, n=`conf.Options.FullSyncReaderCollectionParallel`
    - 将上面细分的任务`nsList`丢给这些线程去运行, 即1个`collection`的同步是一个任务单元, 所以多个`collection`是做到了并行同步的
    - 任务单元的执行逻辑在`dbSyncer.collectionSync`

```golang
// 每个collection为1个任务(nsList), 分配给n条线程执行
func (syncer *DBSyncer) Start() (syncError error) {
    syncer.startTime = time.Now()
    var wg sync.WaitGroup

    filterList := filter.NewDocFilterList()

    // get all namespace (就是f'{database}.{collection}')
    nsList, _, err := utils.GetDbNamespace(syncer.FromMongoUrl, filterList.IterateFilter,
        conf.Options.MongoSslRootCaFile)
    if err != nil {
        return err
    }

    collExecutorParallel := conf.Options.FullSyncReaderCollectionParallel
    namespaces := make(chan utils.NS, collExecutorParallel)

    wg.Add(len(nsList))

    nimo.GoRoutine(func() {
        for _, ns := range nsList {
            namespaces <- ns
        }
    })

    // run collection sync in parallel
    var nsDoneCount int32 = 0
    for i := 0; i < collExecutorParallel; i++ {
        collExecutorId := GenerateCollExecutorId()
        nimo.GoRoutine(func() {
            for {
                ns, ok := <-namespaces
                if !ok {
                    break
                }
                // 每个ns本质就是一个collection
                toNS := utils.NewNS(syncer.nsTrans.Transform(ns.Str()))

                LOG.Info("%s collExecutor-%d sync ns %v to %v begin", syncer, collExecutorId, ns, toNS)
                err := syncer.collectionSync(collExecutorId, ns, toNS) // from collection to collection
                atomic.AddInt32(&nsDoneCount, 1)

                if err != nil {
                    LOG.Critical("%s collExecutor-%d sync ns %v to %v failed. %v",
                        syncer, collExecutorId, ns, toNS, err)
                    syncError = fmt.Errorf("document syncer sync ns %v to %v failed. %v", ns, toNS, err)
                } else {
                    process := int(atomic.LoadInt32(&nsDoneCount)) * 100 / len(nsList)
                    LOG.Info("%s collExecutor-%d sync ns %v to %v successful. db syncer-%d progress %v%%",
                        syncer, collExecutorId, ns, toNS, syncer.id, process)
                }
                wg.Done()
            }
            LOG.Info("%s collExecutor-%d finish", syncer, collExecutorId)
        })
    }

    wg.Wait()
    close(namespaces)

    return syncError
}
```

* `collectionSync`的逻辑如下, 在`collection`这个粒度下还在`splitter`中做了细分任务, 这个是根据`splitKeys`去做分割的, 一般很少用到
    - `splitter`负责从源读数据, 数据reader入队到`splitter.readerChan`
    - `splitSync`那上面出队的`reader`进行读数据, 这里就是读数据的终点了
    - 读到的真实数据(`BSON`)会交给`colExecutor`去同步给目标
    - `colExecutor`也是一个`生产-消费`模型, 上面读到的数据会推给`colExecutor.docBatch (chan []*bson.Raw)`, 在`colExecutor.Start()`中处理队列数据
    - `colExecutor`又又又细分了`DocExecutor`, `DocExecutor`拿到`docBatch`里面的数据后, 才真正的进行同步`exec.doSync(docs)`, 这里就是写数据的终点了
    - 写数据用的`BulkWrite`去批量写入

```golang
// 单个collection的全量同步, splitter负责从FromMongoUrl-ns读, colExecutor负责往ToMongoUrl-toNS写
// start sync single collection
func (syncer *DBSyncer) collectionSync(collExecutorId int, ns utils.NS, toNS utils.NS) error {
    // writer
    colExecutor := NewCollectionExecutor(collExecutorId, syncer.ToMongoUrl, toNS, syncer, conf.Options.TunnelMongoSslRootCaFile)
    if err := colExecutor.Start(); err != nil {
        return fmt.Errorf("start collectionSync failed: %v", err)
    }

    // splitter reader: splitter.Run()会生成n个reader(DocumentReader), 往splitter.readerChan灌;
    // 每个reader相当于一个任务, 在下面走多线程执行, 属于多个reader往一个writer(colExecutor)写的模式;
    // 全量数据可以放心并发无序写入;
    splitter := NewDocumentSplitter(syncer.FromMongoUrl, conf.Options.MongoSslRootCaFile, ns)
    if splitter == nil {
        return fmt.Errorf("create splitter failed")
    }
    defer splitter.Close()

    // run in several pieces
    var wg sync.WaitGroup
    wg.Add(conf.Options.FullSyncReaderParallelThread)
    for i := 0; i < conf.Options.FullSyncReaderParallelThread; i++ {
        go func() {
            defer wg.Done()
            for {
                reader, ok := <-splitter.readerChan
                if !ok || reader == nil {
                    break
                }
                // 从reader里面读取docs, 然后把bson结果往chan colExecutor.docBatch灌
                if err := syncer.splitSync(reader, colExecutor, collectionMetric); err != nil {
                    LOG.Crashf("%v", err)
                }
            }
        }()
    }
    wg.Wait()
    LOG.Info("%s all readers finish, wait all writers finish", syncer)

    // close writer
    if err := colExecutor.Wait(); err != nil {
        return fmt.Errorf("close writer failed: %v", err)
    }
    // set collection finish
    collectionMetric.CollectionStatus = StatusFinish
    return nil
}


// 从reader读docs数据, 然后写到colExecutor
func (syncer *DBSyncer) splitSync(reader *DocumentReader, colExecutor *CollectionExecutor, collectionMetric *CollectionMetric) error {
	bufferSize := conf.Options.FullSyncReaderDocumentBatchSize
	buffer := make([]*bson.Raw, 0, bufferSize)
	bufferByteSize := 0
	for {
		// 里面就是docCursor.Next(), 只是做了层简单的封装
		doc, err := reader.NextDoc()
		if err != nil {
			return fmt.Errorf("splitter reader[%v] get next document failed: %v", reader, err)
		} else if doc == nil {
			atomic.AddUint64(&collectionMetric.FinishCount, uint64(len(buffer)))
			colExecutor.Sync(buffer) // colExecutor.docBatch <- buffer
			syncer.replMetric.AddSuccess(uint64(len(buffer))) // only used to calculate the tps which is extract from "success"
			break
		}

		syncer.replMetric.AddGet(1)

		if bufferByteSize+len(doc) > MAX_BUFFER_BYTE_SIZE || len(buffer) >= bufferSize {
			atomic.AddUint64(&collectionMetric.FinishCount, uint64(len(buffer)))
			colExecutor.Sync(buffer)
			syncer.replMetric.AddSuccess(uint64(len(buffer))) // only used to calculate the tps which is extract from "success"
			buffer = make([]*bson.Raw, 0, bufferSize)
			bufferByteSize = 0
		}

		// transform dbref for document
		if len(conf.Options.TransformNamespace) > 0 && conf.Options.IncrSyncDBRef {
			var docData bson.D
			if err := bson.Unmarshal(doc, &docData); err != nil {
				LOG.Error("splitter reader[%v] do bson unmarshal %v failed. %v", reader, doc, err)
			} else {
				docData = transform.TransformDBRef(docData, reader.ns.Database, syncer.nsTrans)
				if v, err := bson.Marshal(docData); err != nil {
					LOG.Warn("splitter reader[%v] do bson marshal %v failed. %v", reader, docData, err)
				} else {
					doc = v
				}
			}
		}

		buffer = append(buffer, &doc)
		bufferByteSize += len(doc)
	}

	LOG.Info("splitter reader finishes: %v", reader)
	reader.Close()
	// reader.CloseMgo()
	return nil
}

// 接上面collectionSync的colExecutor.Start()
func (colExecutor *CollectionExecutor) Start() error {
	var err error
	if !conf.Options.FullSyncExecutorDebug {
		writeConcern := utils.ReadWriteConcernDefault
		if conf.Options.FullSyncExecutorMajorityEnable {
			writeConcern = utils.ReadWriteConcernMajority
		}
		if colExecutor.conn, err = utils.NewMongoCommunityConn(colExecutor.mongoUrl,
			utils.VarMongoConnectModePrimary, true,
			utils.ReadWriteConcernDefault, writeConcern,
			colExecutor.sslRootFile); err != nil {
			return err
		}
	}

	parallel := conf.Options.FullSyncReaderWriteDocumentParallel
	colExecutor.docBatch = make(chan []*bson.Raw, parallel)

	executors := make([]*DocExecutor, parallel)
	for i := 0; i != len(executors); i++ {
		executors[i] = NewDocExecutor(GenerateDocExecutorId(), colExecutor, colExecutor.conn, colExecutor.syncer)
		go executors[i].start()
	}
	colExecutor.executors = executors
	return nil
}

// 接上面的go executors[i].start()
func (exec *DocExecutor) start() {
	if !conf.Options.FullSyncExecutorDebug {
		defer exec.conn.Close()
	}

	for {
		// 读取源端的数据
		docs, ok := <-exec.colExecutor.docBatch
		if !ok {
			break
		}

		if exec.error == nil {
            // 里面就是往目标
			if err := exec.doSync(docs); err != nil {
				exec.error = err
				// since v2.4.11: panic directly if meets error
				LOG.Crashf("%s sync failed: %v", exec, err)
			}
		}

		exec.colExecutor.wg.Done()
		// atomic.AddInt64(&exec.colExecutor.batchCount, -1)
	}
}

// 终于到头了, 这里便是连接目标库进行BulkWrite
func (exec *DocExecutor) doSync(docs []*bson.Raw) error {
    // 省略大量逻辑
	opts := options.BulkWrite().SetOrdered(false)
	res, err := exec.conn.Client.Database(ns.Database).Collection(ns.Collection).BulkWrite(nil, models, opts)
    // 省略大量逻辑
}
```

* 看完了全量同步流程, 很明显有个缺陷, 全量同步期间源端有数据更新, 两边数据就不一致了, 当然这个限制条件官方也是有指出的

#### 三. 增量同步
* 增量同步逻辑在`OplogSyncer`
* `OplogSyncer`和`DbSyncer`类似
    - 多线程同步, 线程数取决源数据地址和架构
    - mongos只算一个地址即一条线程, 多线程是针对mongod的情况
    - 一条线程对应一个OplogSyncer, 负责单个db实例的增量同步工作, 所以通常情况下只有1个OplogSyncer

* 依旧是`生产-消费`模型
    - worker是消费者, Batcher是生产者

```golang
// collector/coordinator/incr.go
// 增量同步
func (coordinator *ReplicationCoordinator) startOplogReplication(oplogStartPosition interface{},
    fullSyncFinishPosition int64,
    startTsMap map[string]int64) error {

    // prepare all syncer. only one syncer while source is ReplicaSet
    // otherwise one syncer connects to one shard
    LOG.Info("start incr replication")
    for i, src := range coordinator.RealSourceIncrSync {
        var syncerTs interface{}
        if val, ok := oplogStartPosition.(int64); ok && val == 0 {
            if v, ok := startTsMap[src.ReplicaName]; !ok {
                return fmt.Errorf("replia[%v] not exists on startTsMap[%v]", src.ReplicaName, startTsMap)
            } else {
                syncerTs = v
            }
        } else {
            syncerTs = oplogStartPosition // fullBeginTs
        }

        LOG.Info("RealSourceIncrSync[%d]: %s, startTimestamp[%v]", i, src, syncerTs)
        syncer := collector.NewOplogSyncer(src.ReplicaName, syncerTs, fullSyncFinishPosition, src.URL,
            src.Gids)
        // syncerGroup http api registry
        syncer.Init()
        coordinator.syncerGroup = append(coordinator.syncerGroup, syncer)
    }
    // set to group 0 as a leader
    coordinator.syncerGroup[0].SyncGroup = coordinator.syncerGroup

    // prepare worker routine and bind it to syncer
    for i := 0; i < conf.Options.IncrSyncWorker; i++ {
        syncer := coordinator.syncerGroup[i%len(coordinator.syncerGroup)]
        w := collector.NewWorker(syncer, uint32(i))
        if !w.Init() {
            return errors.New("worker initialize error")
        }
        w.SetInitSyncFinishTs(fullSyncFinishPosition)
        syncer.Bind(w)
        go w.StartWorker() // worker是消费者, Batcher是生产者(在syncer.Start()里面)
    }

    for _, syncer := range coordinator.syncerGroup {
        go syncer.Start()
    }
    return nil
}
```


* 附oplog实例及字段意义
> [oplog各字段的解析](https://github.com/alibaba/MongoShake/wiki/oplog%E5%90%84%E5%AD%97%E6%AE%B5%E7%9A%84%E8%A7%A3%E6%9E%90#%E5%87%A0%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84%E6%93%8D%E4%BD%9C%E5%AF%B9%E5%BA%94%E7%9A%84oplog%E4%B8%BE%E4%BE%8B)
```json
{
    "ts" : Timestamp(1582277156, 1), // 操作的时间戳，64位表示，高32位是时间戳，低32位是计数累加
    "t" : NumberLong(1), // 对应raft协议里面的term，每次发生节点down掉，新节点加入，主从切换，term都会自增
    "h" : NumberLong(0), // 操作的全局唯一id的hash结果
    "v" : 2,             // oplog的版本字段
    "op" : "i",          // "i"表示插入，"d"表示删除，"u"表示更新，"c"表示DDL操作，"n"表示心跳
    "ns" : "zz.test",    // 命名空间。操作发生在哪个表上面
    "ui" : UUID("20d9f949-cfc7-496e-a80e-32ba633701a8"), // 表的uuid
    "wall" : ISODate("2020-02-21T09:25:56.570Z"),
    "o" : {              // 具体的操作指令字段, 跟"op"一对
        "_id" : ObjectId("5e4fa224a6717632d6ee2e85"),
        "kick" : 1
    }
}
```