---
title: go module笔记与源码分析
date: 2021/06/09
categories: 
- 后端
tags:
- go
---

#### 简介
* 零零散散的关于go module的笔记, 通过源码来理解这些点


#### go mod源码位置
* 仓库位置: [https://github.com/golang/go/tree/go1.16.6/src/cmd/go/internal/modcmd](https://github.com/golang/go/tree/go1.16.6/src/cmd/go/internal/modcmd)
* 相对位置:
    * "cmd/go/internal/modcmd"
    * "cmd/go/internal/modfetch"
    * "cmd/go/internal/modget"
    * "cmd/go/internal/modload"


#### 关于版本选择
* 同一个大版本取最大版本号
    + main依赖了(A1.1.0, B1.2.0), B1.2.0依赖了(A1.0.0), 则最终构建阶段大家都用A1.1.0编译
* 版本从信息从`/@v/list`获取, 如果为空则取走`@latest`获取最新版本号, 在拿最新版本号取拉包
* [版本规则](https://golang.google.cn/doc/modules/version-numbers)

* 核心逻辑
```go
func (p *proxyRepo) Versions(prefix string) ([]string, error) {
    data, err := p.getBytes("@v/list") // 获取list文件内容
    if err != nil {
        return nil, p.versionError("", err)
    }
    var list []string
    for _, line := range strings.Split(string(data), "\n") {
        f := strings.Fields(line)
        if len(f) >= 1 && semver.IsValid(f[0]) && strings.HasPrefix(f[0], prefix) && !IsPseudoVersion(f[0]) {
            list = append(list, f[0])
        }
    }
    SortVersions(list) // 对list里面的版本进行排序
    return list, nil
}

// IsPseudoVersion reports whether v is a pseudo-version.
func IsPseudoVersion(v string) bool {
    return strings.Count(v, "-") >= 2 && semver.IsValid(v) && pseudoVersionRE.MatchString(v)
}
```

#### go private
* private时不会走proxy和checksumDB
```go
// go/internal/modfetch/sumdb.go
func useSumDB(mod module.Version) bool {
    return cfg.GOSUMDB != "off" && !module.MatchPrefixPatterns(cfg.GONOSUMDB, mod.Path) // cfg.GONOSUMDB里面包含了GOPRIVATE
}

// cmd/go/internal/cfg/cfg.go
GONOSUMDB  = envOr("GONOSUMDB", GOPRIVATE)
```

#### commit version
* go get github.com/pingcap/parser@659821e
* go get github.com/pingcap/parser@latest
* go get github.com/pingcap/parser@feature-lstest


#### @latest
* v0.0.5 > v0.0.5-alpha > v0.0.4
```go
// 版本比较的源码: src/cmd/vendor/golang.org/x/mod/semver/semver.go
func Compare(v, w string) int {
    pv, ok1 := parse(v)
    pw, ok2 := parse(w)
    if !ok1 && !ok2 {
        return 0
    }
    if !ok1 {
        return -1
    }
    if !ok2 {
        return +1
    }
    if c := compareInt(pv.major, pw.major); c != 0 {
        return c
    }
    if c := compareInt(pv.minor, pw.minor); c != 0 {
        return c
    }
    if c := compareInt(pv.patch, pw.patch); c != 0 {
        return c
    }
    return comparePrerelease(pv.prerelease, pw.prerelease)
}
```

#### indirect
* main依赖了A, 但是A依赖了B但go.mod里没有`require B`, 则A的go.mod会自动加上`B indirect`
    + 此时main的go.mod强行require上B, 则`B indirect`将消失


#### exclude
* 跳过某个版本 (之后一般gomod会自动使用比跳过版本更高的版本)
* 例如: "require github.com/google/uuid v1.1.0", 最后tidy后require里自动变成"github.com/google/uuid v1.1.1"
* 跟replace一样, 仅main module时生效
* 不大实用, 一般直接用replace即可


#### url
* GET $base/$module/@v/list
* https://goproxy.io/github.com/gin-gonic/gin/@v/v1.4.0.mod
* https://goproxy.io/github.com/gin-gonic/gin/@v/
* https://goproxy.io/github.com/gin-gonic/gin/@v/list //latest是根据这个list拉取最大的版本
* https://goproxy.io/github.com/gin-gonic/gin/@latest


#### go.sum
* https://sum.golang.org/lookup/golang.org/x/sync@v0.0.0-20181221193216-37e7f081c4d4
* checksum源码: go/src搜索 checkModSum(mod, hash)
* checksumDB源码: go/src搜索 checkSumDB(mod, h)
