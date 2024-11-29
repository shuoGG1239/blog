---
title: mongodb官方备份工具, MongoDump源码分析
date: 2023/04/15
categories: 
- db
tags:
- go
- mongodb
---

#### 简介
* 因为工作需要魔改该模块, 所以详细看了源码, 这里顺便做下笔记
* 本文前几小节讲MongoDump的运转流程, 后几小节讲代码方面的技术细节
* 源码版本: 100.6.1
* 源码仓库: [https://github.com/mongodb/mongo-tools/blob/100.6.1/mongodump](https://github.com/mongodb/mongo-tools/blob/100.6.1/mongodump)


#### 0. 备份总览: MongoDump.Dump
* 备份逻辑就在MongoDump的Dump方法 (mongodump/mongodump.go/MongoDump.Dump)
* Dump主要分4步:
    1. 各种初始化工作
    2. 备份metaData, index, users, roles, version 等基础数据
    3. 备份collections
    4. 备份oplog

* 后面会经常提到`Intent`, 这是MongoDump自己的一个抽象概念, 可以简单理解为备份任务单元, 例如一个collection的备份对应一个Intent, oplog的备份对应一个Intent等等; 在阅读源码时你可以将`Intent`在脑海里替换成`Task`. 关于`Intent`详见本文后面章节

* 核心逻辑见以下源码及注释(为了方便阅读, 这里我删减了些不关键的逻辑):

```go
// Dump是MongoDump的一个方法
type MongoDump struct {
    ToolOptions   *options.ToolOptions
    InputOptions  *InputOptions
    OutputOptions *OutputOptions

    SkipUsersAndRoles bool

    ProgressManager progress.Manager

    SessionProvider *db.SessionProvider // 就是mongoClient
    manager         *intents.Manager    // 备份单元管理, 核心组件
    query           bson.D
    oplogCollection string
    oplogStart      primitive.Timestamp
    oplogEnd        primitive.Timestamp
    isMongos        bool
    storageEngine   storageEngineType
    authVersion     int
    archive         *archive.Writer    // InputOptions.Output非"-"时往这里写入
    shutdownIntentsNotifier *notifier
    OutputWriter io.Writer             // InputOptions.Output为"-"时往这里写入

    Logger *log.ToolLogger
}

// OutputOptions出现频繁所以贴一下
type OutputOptions struct {
	Out                        string   `long:"out" value-name:"<directory-path>" short:"o" description:"output directory, or '-' for stdout (default: 'dump')"`
	Gzip                       bool     `long:"gzip" description:"compress archive or collection output with Gzip"`
	Oplog                      bool     `long:"oplog" description:"use oplog for taking a point-in-time snapshot"`
	Archive                    string   `long:"archive" value-name:"<file-path>" optional:"true" optional-value:"-" description:"dump as an archive to the specified path. If flag is specified without a value, archive is written to stdout"`
	DumpDBUsersAndRoles        bool     `long:"dumpDbUsersAndRoles" description:"dump user and role definitions for the specified database"`
	ExcludedCollections        []string `long:"excludeCollection" value-name:"<collection-name>" description:"collection to exclude from the dump (may be specified multiple times to exclude additional collections)"`
	ExcludedCollectionPrefixes []string `long:"excludeCollectionsWithPrefix" value-name:"<collection-prefix>" description:"exclude all collections from the dump that have the given prefix (may be specified multiple times to exclude additional prefixes)"`
	NumParallelCollections     int      `long:"numParallelCollections" short:"j" description:"number of collections to dump in parallel" default:"4" default-mask:"-"`
	ViewsAsCollections         bool     `long:"viewsAsCollections" description:"dump views as normal collections with their produced data, omitting standard collections"`
}

func (dump *MongoDump) Dump() (err error) {
    defer dump.SessionProvider.Close()
    /* 1. 阶段1: 各种初始化工作 */

    // 检查下dump.ToolOptions.Namespace.DB和dump.ToolOptions.Namespace.Collection是否存在
    exists, err := dump.verifyCollectionExists()
    if err != nil {
        return fmt.Errorf("error verifying collection info: %v", err)
    }
    if !exists {
        return nil
    }

    // 初始化shutdownIntentsNotifier; 本质就是一个shutdown chan
    dump.shutdownIntentsNotifier = newNotifier()

    // 初始化dump.query; 是针对指定了过滤条件的情况, 一般不会用到
    if dump.InputOptions.HasQuery() {
        content, err := dump.InputOptions.GetQuery()
        if err != nil {
            return err
        }
        var query bson.D
        err = bson.UnmarshalExtJSON(content, false, &query)
        if err != nil {
            return fmt.Errorf("error parsing query as Extended JSON: %v", err)
        }
        dump.query = query
    }

    // 连源MongoDB获取authSchemaVersion, 版本小于3则不支持备份用户和角色, 直接返回错误
    if !dump.SkipUsersAndRoles && dump.OutputOptions.DumpDBUsersAndRoles {
        // dump.SessionProvider就是mongoClient
        dump.authVersion, err = auth.GetAuthVersion(dump.SessionProvider)
        if err == nil {
            err = auth.VerifySystemAuthVersion(dump.SessionProvider)
        }
        if err != nil {
            return fmt.Errorf("error getting auth schema version for dumpDbUsersAndRoles: %v", err)
        }
        if dump.authVersion < 3 {
            return fmt.Errorf("backing up users and roles is only supported for "+
                "deployments with auth schema versions >= 3, found: %v", dump.authVersion)
        }
    }

    // 初始化dump.archive, dump.archive是个高度封装的Writer
    if dump.OutputOptions.Archive != "" {
        var archiveOut io.WriteCloser
        // 根据dump.OutputOptions.Archive获取对应输出文件的ioWriter, 当值为"-"时输出将到OutputWriter而不是文件
        archiveOut, err = dump.getArchiveOut()
        if err != nil {
            return err
        }
        dump.archive = &archive.Writer{
            Out: archiveOut,
            Mux: archive.NewMultiplexer(archiveOut, dump.shutdownIntentsNotifier),
        }
        go dump.archive.Mux.Run()
        // 备份结束后的一些释放工作
        defer func() {
            // The Mux runs until its Control is closed
            close(dump.archive.Mux.Control)
            muxErr := <-dump.archive.Mux.Completed
            archiveOut.Close()
            if muxErr != nil {
                if err != nil {
                    err = fmt.Errorf("archive writer: %v / %v", err, muxErr)
                } else {
                    err = fmt.Errorf("archive writer: %v", muxErr)
                }
                dump.Logger.Logvf(log.DebugLow, "%v", err)
            } else {
                dump.Logger.Logvf(log.DebugLow, "mux completed successfully")
            }
        }()
    }

    // 源Mongodb连通性检测
    session, err := dump.SessionProvider.GetSession()
    if err != nil {
        return fmt.Errorf("error getting a client session: %v", err)
    }
    err = session.Ping(context.Background(), nil)
    if err != nil {
        return fmt.Errorf("error connecting to host: %v", err)
    }

    // 创建备份Intents
    switch {
    case dump.ToolOptions.DB == "" && dump.ToolOptions.Collection == "":
        err = dump.CreateAllIntents()
    case dump.ToolOptions.DB != "" && dump.ToolOptions.Collection == "":
        err = dump.CreateIntentsForDatabase(dump.ToolOptions.DB)
    case dump.ToolOptions.DB != "" && dump.ToolOptions.Collection != "":
        err = dump.CreateCollectionIntent(dump.ToolOptions.DB, dump.ToolOptions.Collection)
    }
    if err != nil {
        return fmt.Errorf("error creating intents to dump: %v", err)
    }

    // 如果需要备份Oplog, 则创建备份Oplog的Intents
    if dump.OutputOptions.Oplog {
        err = dump.CreateOplogIntents()
        if err != nil {
            return err
        }
    }

    // 如果需要备份Users和Roles, 则创建备份Users和Role的Intents
    if !dump.SkipUsersAndRoles && dump.OutputOptions.DumpDBUsersAndRoles && dump.ToolOptions.DB != "admin" {
        err = dump.CreateUsersRolesVersionIntentsForDB(dump.ToolOptions.DB)
        if err != nil {
            return err
        }
    }

    /* 2. 阶段2: 备份metaData, index, users, roles, version 等基础数据 */
    err = dump.DumpMetadata() // intent.MetadataFile.Write(json.Marshal(metadata))
    if err != nil {
        return fmt.Errorf("error dumping metadata: %v", err)
    }

    if dump.OutputOptions.Archive != "" {
        serverVersion, err := dump.SessionProvider.ServerVersion()
        if err != nil {
            dump.Logger.Logvf(log.Always, "warning, couldn't get version information from server: %v", err)
            serverVersion = "unknown"
        }
        dump.archive.Prelude, err = archive.NewPrelude(dump.manager, dump.OutputOptions.NumParallelCollections, serverVersion, dump.ToolOptions.VersionStr)
        if err != nil {
            return fmt.Errorf("creating archive prelude: %v", err)
        }
        err = dump.archive.Prelude.Write(dump.archive.Out)
        if err != nil {
            return fmt.Errorf("error writing metadata into archive: %v", err)
        }
    }

    // 备份users, roles
    if !dump.SkipUsersAndRoles {
        if dump.ToolOptions.DB == "admin" || dump.ToolOptions.DB == "" {
            err = dump.DumpUsersAndRoles()
            if err != nil {
                return fmt.Errorf("error dumping users and roles: %v", err)
            }
        }
        if dump.OutputOptions.DumpDBUsersAndRoles {
            dump.Logger.Logvf(log.Always, "dumping users and roles for %v", dump.ToolOptions.DB)
            if dump.ToolOptions.DB == "admin" {
                dump.Logger.Logvf(log.Always, "skipping users/roles dump, already dumped admin database")
            } else {
                err = dump.DumpUsersAndRolesForDB(dump.ToolOptions.DB)
                if err != nil {
                    return fmt.Errorf("error dumping users and roles: %v", err)
                }
            }
        }
    }

    // 设置dump.oplogStart 和 dump.oplogCollection
    if dump.OutputOptions.Oplog {
        // set dump.oplogCollection, "oplog.rs"或"oplog.$main"
        err := dump.determineOplogCollectionName()
        if err != nil {
            return fmt.Errorf("error finding oplog: %v", err)
        }
        dump.Logger.Logvf(log.Info, "getting most recent oplog timestamp")
        dump.oplogStart, err = dump.getOplogCopyStartTime()
        if err != nil {
            return fmt.Errorf("error getting oplog start: %v", err)
        }
    }

    /* 3. 阶段3: 备份collections */
    if err := dump.DumpIntents(); err != nil {
        return err
    }

    /* 4. 阶段4: 备份oplog */
    if dump.OutputOptions.Oplog {
        dump.oplogEnd, err = dump.getCurrentOplogTime()
        if err != nil {
            return fmt.Errorf("error getting oplog end: %v", err)
        }

        // 确认oplog文件是否发生了翻转(Roll over), oplog本身是个环形队列
        exists, err := dump.checkOplogTimestampExists(dump.oplogStart)
        if !exists {
            return fmt.Errorf(
                "oplog overflow: mongodump was unable to capture all new oplog entries during execution")
        }
        if err != nil {
            return fmt.Errorf("unable to check oplog for overflow: %v", err)
        }
        dump.Logger.Logvf(log.Always, "writing captured oplog to %v", dump.manager.Oplog().Location)

        // 备份oplog
        err = dump.DumpOplogBetweenTimestamps(dump.oplogStart, dump.oplogEnd)
        if err != nil {
            return fmt.Errorf("error dumping oplog: %v", err)
        }

        // 再次确认oplog文件是否发生了翻转
        dump.Logger.Logvf(log.DebugLow, "checking again if oplog entry %v still exists", dump.oplogStart)
        exists, err = dump.checkOplogTimestampExists(dump.oplogStart)
        if !exists {
            return fmt.Errorf(
                "oplog overflow: mongodump was unable to capture all new oplog entries during execution")
        }
        if err != nil {
            return fmt.Errorf("unable to check oplog for overflow: %v", err)
        }
    }

    // 备份完成
    dump.Logger.Logvf(log.DebugLow, "finishing dump")
    return err
}
```


#### 1. 备份metadata
* 备份metadata的逻辑比较简单, 就是将`Metadata` jsonMarshal后写入`intent.MetadataFile (io)`

* 源码逻辑如下:
```go
type Metadata struct {
    Options        bson.M   `bson:"options,omitempty"`
    Indexes        []bson.D `bson:"indexes"`
    UUID           string   `bson:"uuid,omitempty"`
    CollectionName string   `bson:"collectionName"`
    Type           string   `bson:"type,omitempty"`
}

func (dump *MongoDump) dumpMetadata(intent *intents.Intent, buffer resettableOutputBuffer) (err error) {
    // 1. 填充Metadata, 值取自入参`intent`
    meta := Metadata{
        Indexes: []bson.D{},
    }
    meta.Options = intent.Options
    meta.UUID = intent.UUID
    meta.CollectionName = intent.C
    if intent.Type != "" {
        meta.Type = intent.Type
    }

    session, err := dump.SessionProvider.GetSession()
    if err != nil {
        return err
    }

    // 获取源端的index并set进meta.Indexes
    if dump.OutputOptions.ViewsAsCollections || intent.IsView() {
        dump.Logger.Logvf(log.DebugLow, "not dumping indexes metadata for '%v' because it is a view", intent.Namespace())
    } else {
        // get the indexes
        indexesIter, err := db.GetIndexes(session.Database(intent.DB).Collection(intent.C))
        if err != nil {
            return err
        }
        if indexesIter == nil {
            dump.Logger.Logvf(log.Always, "the collection %v appears to have been dropped after the dump started", intent.Namespace())
            return nil
        }
        defer indexesIter.Close(context.Background())

        ctx := context.Background()
        for indexesIter.Next(ctx) {
            indexOpts := &bson.D{}
            err := indexesIter.Decode(indexOpts)
            if err != nil {
                return fmt.Errorf("error converting index: %v", err)
            }

            meta.Indexes = append(meta.Indexes, *indexOpts)
        }

        if err := indexesIter.Err(); err != nil {
            return fmt.Errorf("error getting indexes for collection `%v`: %v", intent.Namespace(), err)
        }
    }

    // 2. 把Metadata写入intent.MetadataFile
    /* 后面就是将meta jsonMarshal后写入intent.MetadataFile 而已*/
    jsonBytes, err := bson.MarshalExtJSON(meta, true, false)
    if err != nil {
        return fmt.Errorf("error marshalling metadata json for collection `%v`: %v", intent.Namespace(), err)
    }

    err = intent.MetadataFile.Open()
    if err != nil {
        return err
    }
    defer func() {
        closeErr := intent.MetadataFile.Close()
        if err == nil && closeErr != nil {
            err = fmt.Errorf("error writing metadata for collection `%v` to disk: %v", intent.Namespace(), closeErr)
        }
    }()

    var f io.Writer
    f = intent.MetadataFile
    if buffer != nil {
        buffer.Reset(f)
        f = buffer
        defer func() {
            closeErr := buffer.Close()
            if err == nil && closeErr != nil {
                err = fmt.Errorf("error writing metadata for collection `%v` to disk: %v", intent.Namespace(), closeErr)
            }
        }()
    }
    _, err = f.Write(jsonBytes)
    if err != nil {
        err = fmt.Errorf("error writing metadata for collection `%v` to disk: %v", intent.Namespace(), err)
    }
    return
}
```

#### 2. 备份collections
* 源码如下, 就是从intent的manager中不断取intent分配给n条线程进行备份

* 一个intent对应一个collection的备份任务

```go
// 并发备份collections, NumParallelCollections条线程
func (dump *MongoDump) DumpIntents() error {
	resultChan := make(chan error)

    // 线程数 = jobs = dump.OutputOptions.NumParallelCollections
	jobs := dump.OutputOptions.NumParallelCollections
	if numIntents := len(dump.manager.Intents()); jobs > numIntents {
		jobs = numIntents
	}

    // 设置intents的Pop顺序策略
	if jobs > 1 {
		dump.manager.Finalize(intents.LongestTaskFirst)
	} else {
		dump.manager.Finalize(intents.Legacy)
	}

    // 多线程从dump.manager中Pop出Intent, 进行dump
	for i := 0; i < jobs; i++ {
		go func(id int) {
			buffer := dump.getResettableOutputBuffer()
			dump.Logger.Logvf(log.DebugHigh, "starting dump routine with id=%v", id)
			for {
				intent := dump.manager.Pop()
				if intent == nil {
					dump.Logger.Logvf(log.DebugHigh, "ending dump routine with id=%v, no more work to do", id)
					resultChan <- nil
					return
				}
				if intent.BSONFile != nil {
					err := dump.DumpIntent(intent, buffer)
					if err != nil {
						resultChan <- err
						return
					}
				}
				dump.manager.Finish(intent)
			}
		}(i)
	}
    // 等待所有intents dump完
	for i := 0; i < jobs; i++ {
		if err := <-resultChan; err != nil {
			return err
		}
	}

	return nil
}
```



#### 3. 备份oplog
* 备份oplog的逻辑比较简单, 将查询oplog的结果写入oplog对应的intent.BSONFile

* 如果对oplog不熟悉可以看下官方文档: [https://www.mongodb.com/zh-cn/docs/manual/core/replica-set-oplog/](https://www.mongodb.com/zh-cn/docs/manual/core/replica-set-oplog/)


```go
// 查询start与end之间的oplog, 写入 dump.manager.Oplog().BSONFile
func (dump *MongoDump) DumpOplogBetweenTimestamps(start, end primitive.Timestamp) error {
	session, err := dump.SessionProvider.GetSession()
	if err != nil {
		return err
	}
	queryObj := bson.M{"$and": []bson.M{
		{"ts": bson.M{"$gte": start}},
		{"ts": bson.M{"$lte": end}},
	}}
	oplogQuery := &db.DeferredQuery{
        // "local.oplog.rs"(replset)或"local.oplog.$main"(m/s)
		Coll:      session.Database("local").Collection(dump.oplogCollection),
		Filter:    queryObj,
		LogReplay: true,
	}
    // 执行上面的`oplogQuery`, 将结果写入`dump.manager.Oplog().BSONFile` (dump.manager.Oplog()是oplog的intent)
	oplogCount, err := dump.dumpValidatedQueryToIntent(oplogQuery, dump.manager.Oplog(), dump.getResettableOutputBuffer(), oplogDocumentValidator)
	if err == nil {
		dump.Logger.Logvf(log.Always, "\tdumped %v oplog %v",
			oplogCount, util.Pluralize(int(oplogCount), "entry", "entries"))
	}
	return err
}

// 把`query`的查询结果写入`intent.BSONFile` (这是一个公共函数, 上边引用到了就顺便贴下, 不关键)
func (dump *MongoDump) dumpValidatedQueryToIntent(
	query *db.DeferredQuery, intent *intents.Intent, buffer resettableOutputBuffer, validator documentValidator) (dumpCount int64, err error) {

	err = intent.BSONFile.Open()
	if err != nil {
		return 0, err
	}
	defer func() {
		closeErr := intent.BSONFile.Close()
		if err == nil && closeErr != nil {
			err = fmt.Errorf("error writing data for collection `%v` to disk: %v", intent.Namespace(), closeErr)
		}
	}()
	// don't dump any data for views being dumped as views
	if intent.IsView() && !dump.OutputOptions.ViewsAsCollections {
		return 0, nil
	}

	total, err := dump.getCount(query, intent)
	if err != nil {
		return 0, err
	}

	dumpProgressor := progress.NewCounter(total)
	if dump.ProgressManager != nil {
		dump.ProgressManager.Attach(intent.Namespace(), dumpProgressor)
		defer dump.ProgressManager.Detach(intent.Namespace())
	}

	var f io.Writer
	f = intent.BSONFile
	if buffer != nil {
		buffer.Reset(f)
		f = buffer
		defer func() {
			closeErr := buffer.Close()
			if err == nil && closeErr != nil {
				err = fmt.Errorf("error writing data for collection `%v` to disk: %v", intent.Namespace(), closeErr)
			}
		}()
	}

	cursor, err := query.Iter()
	if err != nil {
		return
	}
    // 将cursor查询结果写入f
	err = dump.dumpValidatedIterToWriter(cursor, f, dumpProgressor, validator)
	dumpCount, _ = dumpProgressor.Progress()
	if err != nil {
		err = fmt.Errorf("error writing data for collection `%v` to disk: %v", intent.Namespace(), err)
	}
	return
}
```


#### 备份单元: Intent
* 备份任务单元, 可以简单理解1个collection的备份任务就叫intent, 拆分是为了多线程执行

* Intent相关的结构和关键方法:

```go
type Intent struct {
	// Destination namespace info
	DB string
	C  string // collection

	// File locations as absolute paths
	BSONFile     file
	BSONSize     int64
	MetadataFile file

	// Indicates where the intent will be read from or written to
	Location         string
	MetadataLocation string

	// Collection options
	Options bson.M

	// UUID (for MongoDB 3.6+) as a big-endian hex string
	UUID string

	// File/collection size, for some prioritizer implementations.
	// Units don't matter as long as they are consistent for a given use case.
	Size int64

	// Either view or timeseries. Empty string "" is a regular collection.
	Type string
}

// 查询collection(intent.C) 的数据, 写入intent.BSONFile, 仅此而已
func (dump *MongoDump) DumpIntent(intent *intents.Intent, buffer resettableOutputBuffer) error {
	session, err := dump.SessionProvider.GetSession()
	if err != nil {
		return err
	}
	intendedDB := session.Database(intent.DB)
	var coll *mongo.Collection
	if intent.IsTimeseries() {
		coll = intendedDB.Collection("system.buckets." + intent.C)
	} else {
		coll = intendedDB.Collection(intent.C)
	}

	isView := true
	collInfo, err := db.GetCollectionInfo(coll)
	if err != nil {
		return err
	} else if collInfo != nil {
		isView = collInfo.IsView()
	}
	// 推断并设置dump.storageEngine
	if dump.storageEngine == storageEngineUnknown && !isView {
		if err != nil {
			return err
		}
		dump.storageEngine = storageEngineModern
		isMMAPV1, err := db.IsMMAPV1(intendedDB, intent.C)
		if err != nil {
			dump.Logger.Logvf(log.Always,
				"failed to determine storage engine, an mmapv1 storage engine could result in"+
					" inconsistent dump results, error was: %v", err)
		} else if isMMAPV1 {
			dump.storageEngine = storageEngineMMAPV1
		}
	}

	findQuery := &db.DeferredQuery{Coll: coll}
	switch {
	case len(dump.query) > 0:
		if intent.IsTimeseries() {
			metaKey, ok := intent.Options["timeseries"].(bson.M)["metaField"].(string)
			if !ok {
				return fmt.Errorf("could not determine the metaField for %s", intent.Namespace())
			}
			for i, predicate := range dump.query {
				splitPredicateKey := strings.SplitN(predicate.Key, ".", 2)
				if splitPredicateKey[0] != metaKey {
					return fmt.Errorf("cannot process query %v for timeseries collection %s. "+
						"mongodump only processes queries on metadata fields for timeseries collections.", dump.query, intent.Namespace())
				}
				if len(splitPredicateKey) > 1 {
					dump.query[i].Key = "meta." + splitPredicateKey[1]
				} else {
					dump.query[i].Key = "meta"
				}

			}
		}
		findQuery.Filter = dump.query
	case dump.storageEngine == storageEngineMMAPV1 && !dump.InputOptions.TableScan &&
		!isView && !intent.IsSpecialCollection() && !intent.IsOplog():
		autoIndexId, found := intent.Options["autoIndexId"]
		if !found || autoIndexId == true {
			findQuery.Hint = bson.D{{"_id", 1}}
		}
	}

	var dumpCount int64
	if dump.OutputOptions.Out == "-" {
        // 初始化阶段有 "intent.BSONFile = &stdoutFile{Writer: dump.OutputWriter}", 可以搜下源码
        // 所以这里虽然也是写到intent.BSONFile, 但实际写到dump.OutputWriter了
		dump.Logger.Logvf(log.Always, "writing %v to stdout", intent.DataNamespace())
		dumpCount, err = dump.dumpQueryToIntent(findQuery, intent, buffer)
		if err == nil {
			// on success, print the document count
			dump.Logger.Logvf(log.Always, "dumped %v %v", dumpCount, docPlural(dumpCount))
		}
		return err
	}

    //  将findQuery查到的写入intent.BSONFile
	if dumpCount, err = dump.dumpQueryToIntent(findQuery, intent, buffer); err != nil {
		return err
	}
	return nil
}

// 将query查到的写入intent.BSONFile
func (dump *MongoDump) dumpQueryToIntent(
	query *db.DeferredQuery, intent *intents.Intent, buffer resettableOutputBuffer) (dumpCount int64, err error) {
	return dump.dumpValidatedQueryToIntent(query, intent, buffer, nil)
}

func (dump *MongoDump) dumpValidatedQueryToIntent(
	query *db.DeferredQuery, intent *intents.Intent, buffer resettableOutputBuffer, validator documentValidator) (dumpCount int64, err error) {

	err = intent.BSONFile.Open()
	if err != nil {
		return 0, err
	}
	defer func() {
		closeErr := intent.BSONFile.Close()
		if err == nil && closeErr != nil {
			err = fmt.Errorf("error writing data for collection `%v` to disk: %v", intent.Namespace(), closeErr)
		}
	}()
	// don't dump any data for views being dumped as views
	if intent.IsView() && !dump.OutputOptions.ViewsAsCollections {
		return 0, nil
	}

	total, err := dump.getCount(query, intent)
	if err != nil {
		return 0, err
	}

	dumpProgressor := progress.NewCounter(total)
	if dump.ProgressManager != nil {
		dump.ProgressManager.Attach(intent.Namespace(), dumpProgressor)
		defer dump.ProgressManager.Detach(intent.Namespace())
	}

	var f io.Writer
	f = intent.BSONFile
	if buffer != nil {
		buffer.Reset(f)
		f = buffer
		defer func() {
			closeErr := buffer.Close()
			if err == nil && closeErr != nil {
				err = fmt.Errorf("error writing data for collection `%v` to disk: %v", intent.Namespace(), closeErr)
			}
		}()
	}

	cursor, err := query.Iter()
	if err != nil {
		return
	}
    // 将cursor查到的东西写入f
	err = dump.dumpValidatedIterToWriter(cursor, f, dumpProgressor, validator)
	dumpCount, _ = dumpProgressor.Progress()
	if err != nil {
		err = fmt.Errorf("error writing data for collection `%v` to disk: %v", intent.Namespace(), err)
	}
	return
}

```


#### 多路读写模块: archive.Writer/Reader
* 这是MongoDump唯一比较复杂的模块, 因为只讲备份, 所以只讲`archive.Writer`, `archive.Reader`是恢复时用到, 原理一样

* 多路读写核心`Multiplexer`, 里面有个核心组件`MuxIn`, 这东西其实就是上面提到的`BSONFile`的实现, 每次`BSONFile.Open`就相当于`New`一个`NuxIn`然后塞给`Multiplexer`管理, `MuxIn`就是多路读写里面的`路`

* 多路读写怎么实现? 其实本质是`多路In, 一路Out`, 上面提到的`MuxIn`是实现`多路In`, 而`一路Out`的关键逻辑在`Multiplexer.formatBody`, 这里可以看看下面的源码, 其实就是利用写入header和namespace来做数据隔离, 配合`Multiplexer`的`select channel`这样就实现了多路读写. 这个思想是值得学习的

* 概念那么多是不是看了头晕? 我们将所有概念都关联起来捋一下:
    - 在1次备份中, 只有1个`archive.Writer`, 也意味着只有1个`Multiplexer`, 1个`Multiplexer`管理了`n`个`MuxIn`, `n`又等于`Intent`的个数, `Intent`有多少个? `Intent`的个数为`len(collections) + 1 + 1`, 这里的两个`1`分别是`metadata`和`oplog`

* `Multiplexer`源码的几个核心方法:
```go
type Writer struct {
	Out     io.WriteCloser
	Prelude *Prelude
	Mux     *Multiplexer
}

type Multiplexer struct {
	Out       io.WriteCloser
	Control   chan *MuxIn
	Completed chan error
	shutdownInputs notifier
	// ins and selectCases are correlating slices
	ins              []*MuxIn
	selectCases      []reflect.SelectCase
	currentNamespace string
}

type notifier interface {
	Notify()
}

func NewMultiplexer(out io.WriteCloser, shutdownInputs notifier) *Multiplexer {
	mux := &Multiplexer{
		Out:            out,
		Control:        make(chan *MuxIn),
		Completed:      make(chan error),
		shutdownInputs: shutdownInputs,
		ins: []*MuxIn{
			nil, // There is no MuxIn for the Control case
		},
	}
    // 反射实现channel select, 非常少见的玩法!
	mux.selectCases = []reflect.SelectCase{
		{
			Dir:  reflect.SelectRecv,
			Chan: reflect.ValueOf(mux.Control),
			Send: reflect.Value{},
		},
	}
	return mux
}

// 核心事件循环: 处理MuxIn的增删事件和来自MuxIn的写数据事件
func (mux *Multiplexer) Run() {
	var err, completionErr error
	for {
		// select的反射玩法, 学到了
		index, value, notEOF := reflect.Select(mux.selectCases)
		EOF := !notEOF
		if index == 0 { // index 0 为 mux.Control, 用于接收新的MuxIn
			if EOF {
				log.Logvf(log.DebugLow, "Mux finish")
				mux.Out.Close()
				if completionErr != nil {
					mux.Completed <- completionErr
				} else if len(mux.selectCases) != 1 {
					mux.Completed <- fmt.Errorf("Mux ending but selectCases still open %v",
						len(mux.selectCases))
				} else {
					mux.Completed <- nil
				}
				return
			}
			muxIn, ok := value.Interface().(*MuxIn)
			if !ok {
				mux.Completed <- fmt.Errorf("non MuxIn received on Control chan") // one for the MuxIn.Open
				return
			}
			log.Logvf(log.DebugLow, "Mux open namespace %v", muxIn.Intent.DataNamespace())
			mux.selectCases = append(mux.selectCases, reflect.SelectCase{
				Dir:  reflect.SelectRecv,
				Chan: reflect.ValueOf(muxIn.writeChan),
				Send: reflect.Value{},
			})
			mux.ins = append(mux.ins, muxIn)
		} else { // index > 0 为 MuxIn.writeChan, 用于接收MuxIn.Write的data
			if EOF {
				mux.ins[index].writeCloseFinishedChan <- struct{}{}

				err = mux.formatEOF(index, mux.ins[index])
				if err != nil {
					mux.shutdownInputs.Notify()
					mux.Out = &nopCloseNopWriter{}
					completionErr = err
				}
				log.Logvf(log.DebugLow, "Mux close namespace %v", mux.ins[index].Intent.DataNamespace())
				mux.currentNamespace = ""
				mux.selectCases = append(mux.selectCases[:index], mux.selectCases[index+1:]...)
				mux.ins = append(mux.ins[:index], mux.ins[index+1:]...)
			} else {
				bsonBytes, ok := value.Interface().([]byte)
				if !ok {
					mux.Completed <- fmt.Errorf("multiplexer received a value that wasn't a []byte")
					return
				}
				// format bsonBytes, 然后 mux.Out.Write(bsonBytes)
				err = mux.formatBody(mux.ins[index], bsonBytes)
				if err != nil {
					mux.shutdownInputs.Notify()
					mux.Out = &nopCloseNopWriter{}
					completionErr = err
				}
			}
		}
	}
}


// 核心逻辑, 这个里的header用于隔离不同namespace的数据, 已达到多路的效果, 恢复的时候也是根据header来恢复的
// mux.Out.Write header和bsonBytes, 这里的Out其实就是dump.archive.Out
func (mux *Multiplexer) formatBody(in *MuxIn, bsonBytes []byte) error {
	var err error
	var length int
	defer func() {
		in.writeLenChan <- length
	}()
	if in.Intent.DataNamespace() != mux.currentNamespace {
		// Handle the change of which DB/Collection we're writing docs for
		// If mux.currentNamespace then we need to terminate the current block
		if mux.currentNamespace != "" {
			l, err := mux.Out.Write(terminatorBytes)
			if err != nil {
				return err
			}
			if l != len(terminatorBytes) {
				return io.ErrShortWrite
			}
		}
		header, err := bson.Marshal(NamespaceHeader{
			Database:   in.Intent.DB,
			Collection: in.Intent.DataCollection(),
		})
		if err != nil {
			return err
		}
		l, err := mux.Out.Write(header)
		if err != nil {
			return err
		}
		if l != len(header) {
			return io.ErrShortWrite
		}
	}
	mux.currentNamespace = in.Intent.DataNamespace()
	length, err = mux.Out.Write(bsonBytes)
	if err != nil {
		return err
	}
	return nil
}
```

* `MuxIn`源码:
```go
type MuxIn struct {
	writeChan              chan []byte
	writeLenChan           chan int
	writeCloseFinishedChan chan struct{}
	buf                    []byte
	hash                   hash.Hash64
	Intent                 *intents.Intent
	Mux                    *Multiplexer
}

func (muxIn *MuxIn) Read([]byte) (int, error) {
	return 0, nil
}

func (muxIn *MuxIn) Pos() int64 {
	return 0
}

// 关闭muxIn内部的所有chan, 最后multiplexer会收到关闭信号并返回formatEOF, 同时multiplexer也会发信号muxIn.writeCloseFinishedChan
func (muxIn *MuxIn) Close() error {
	// the mux side of this gets closed in the mux when it gets an eof on the read
	log.Logvf(log.DebugHigh, "MuxIn close %v", muxIn.Intent.DataNamespace())
	if bufferWrites {
		muxIn.writeChan <- muxIn.buf
		length := <-muxIn.writeLenChan
		if length != len(muxIn.buf) {
			return io.ErrShortWrite
		}
		muxIn.buf = nil
	}
	close(muxIn.writeChan)
	close(muxIn.writeLenChan)
	<-muxIn.writeCloseFinishedChan
	return nil
}

// 初始化muxIn, 然后把自己发给 muxIn.Mux
func (muxIn *MuxIn) Open() error {
	log.Logvf(log.DebugHigh, "MuxIn open %v", muxIn.Intent.DataNamespace())
	muxIn.writeChan = make(chan []byte)
	muxIn.writeLenChan = make(chan int)
	muxIn.writeCloseFinishedChan = make(chan struct{})
	muxIn.buf = make([]byte, 0, bufferSize)
	muxIn.hash = crc64.New(crc64.MakeTable(crc64.ECMA))
	if bufferWrites {
		muxIn.buf = make([]byte, 0, db.MaxBSONSize)
	}
	muxIn.Mux.Control <- muxIn
	return nil
}

// buf写入muxIn.buf, 满了就把muxIn.buf写入muxIn.writeChan, 然后清空muxIn.buf
func (muxIn *MuxIn) Write(buf []byte) (int, error) {
	if bufferWrites { // 固定true
		if len(muxIn.buf)+len(buf) > cap(muxIn.buf) {
			muxIn.writeChan <- muxIn.buf
			length := <-muxIn.writeLenChan
			if length != len(muxIn.buf) {
				return 0, io.ErrShortWrite
			}
			muxIn.buf = muxIn.buf[:0]
		}
		muxIn.buf = append(muxIn.buf, buf...)
	} else {
		muxIn.writeChan <- buf
		length := <-muxIn.writeLenChan
		if length != len(buf) {
			return 0, io.ErrShortWrite
		}
	}
	muxIn.hash.Write(buf)
	return len(buf), nil
}
```