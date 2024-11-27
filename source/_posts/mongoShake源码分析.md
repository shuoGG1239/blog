---
title: Mongodb数据迁移: MongoShake源码阅读
date: 2023/04/09
categories: 
- db
tags:
- go
- mongodb
---
#### 简述
* 近期用到了MongoShake做数据迁移, 顺便看看源码, 本篇为阅读源码的笔记

#### 源码信息
* 源码版本: v2.8.1
* 源码仓库: [https://github.com/alibaba/MongoShake](https://github.com/alibaba/MongoShake)

#### 一. 进程入口:main
* `func main`为进程的入口, 核心源码如下: (为了只关注核心逻辑, 无关紧要的代码片段会用`// ... + 注释`代替)
* 简单来讲做了3件事情
    1. 初始化和校验配置参数
    2. 加进程锁 
        + 使用了第三方工具`github.com/nightlyone/lockfile`实现的
        + 简单讲就是在`conf.Options.PidPath`目录下建一个`{conf.Options.Id}.pid`文件来实现进程锁; 所以你可以扫描该目录下有多少个pid文件来确认当前有多少RedisShake进程在运行
    3. 跑数据迁移


#### 留坑,以后填