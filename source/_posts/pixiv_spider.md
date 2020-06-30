---
title: go-爬点Pixiv画师图
date: 2020/06/21
---
#### 背景
* 刚好需要某个画师的插画, 故写了个简单无需登录的爬图工具
* 根据Pixiv画师ID, 爬完直接保存在当前目录的

#### 使用
```golang
// Fetch完后自动建一个画师ID的目录然后图片就放里面
cli := PixivClient{
    Cli:       http.Client{},
    Proxy:     defaultProxy(), // 不需要代理则nil即可
}
ctx, _ := context.WithTimeout(context.Background(), time.Second*600)
cli.Fetch(ctx, "4462", -1) // 画师ID:4462
```

#### 完整实现
* 代码很简单, 就100多行

```golang
package pixiv

import (
    "context"
    "encoding/json"
    "fmt"
    "golang.org/x/net/proxy"
    "io"
    "io/ioutil"
    "math/rand"
    "net/http"
    "os"
    "strconv"
    "strings"
    "sync"
    "time"
)

type PixivClient struct {
    Cli       http.Client
    Proxy     proxy.Dialer // 不为nil则走proxy
    RandSleep bool         // 随机sleep, 防止爬太快
}

// limit为-1则无限
func (pcli PixivClient) Fetch(ctx context.Context, userId string, limit int) {
    wg := sync.WaitGroup{}
    tasksDone := make(chan struct{})
    illusts, err := pcli.ListIllustByUser(userId)
    if err != nil {
        return
    }
    for i, illust := range illusts {
        if i > limit && limit != -1 {
            break
        }
        wg.Add(1)
        go func(imgUrl, imgId string) {
            defer wg.Done()
            if pcli.RandSleep {
                randSleep(time.Second * 5)
            }
            err := pcli.fetchOne(userId, imgUrl, imgId)
            if err != nil {
                fmt.Printf("fetchOne(%s, %s): %s\n", userId, imgId, err.Error())
                return
            }
            fmt.Printf("%s finish!\n", imgId)
        }(illust.ImageUrls.Large, strconv.Itoa(illust.Id))
    }
    go func() {
        wg.Wait()
        tasksDone <- struct{}{}
    }()
    select {
    case <-tasksDone:
    case <-ctx.Done():
        fmt.Println(ctx.Err())
    }
}

type illustInfo struct {
    Id        int `json:"id"`
    ImageUrls struct {
        Large string `json:"large"`
    } `json:"image_urls"`
}

func (pcli PixivClient) ListIllustByUser(userId string) ([]illustInfo, error) {
    type pagination struct {
        Pages   int `json:"pages"`
        Current int `json:"current"`
        PerPage int `json:"per_page"`
        Total   int `json:"total"`
    }
    type imjad struct {
        Response   []illustInfo `json:"response"`
        Pagination pagination   `json:"pagination"`
    }
    pcli.Cli.Transport = nil // 国内的api无需代理
    getFn := func(pageNo int) (*imjad, error) {
        u := fmt.Sprintf("https://api.imjad.cn/pixiv/v1/?type=member_illust&id=%s&page=%d", userId, pageNo)
        req, _ := http.NewRequest("GET", u, nil)
        req.Header.Set("user-agent",
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36")
        resp, err := pcli.Cli.Do(req)
        if err != nil {
            return nil, err
        }
        data, err := ioutil.ReadAll(resp.Body)
        if err != nil {
            return nil, err
        }
        var im imjad
        err = json.Unmarshal(data, &im)
        if err != nil {
            return nil, err
        }
        return &im, nil
    }
    im, err := getFn(1)
    if err != nil {
        return nil, err
    }
    illusts := make([]illustInfo, 0)
    for p := 1; p <= im.Pagination.Pages; p++ {
        im, err := getFn(p)
        if err != nil {
            return nil, err
        }
        illusts = append(illusts, im.Response...)
    }
    return illusts, nil
}

func (pcli PixivClient) fetchOne(userId, imgUrl, imgId string) error {
    u := imgUrl
    cli := pcli.Cli
    if pcli.Proxy != nil {
        cli.Transport = pcli.socks5Transport()
    }
    req, _ := http.NewRequest("GET", u, nil)
    req.Header.Set("user-agent",
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36")
    req.Header.Set("Referer", "http://www.pixiv.net/")
    resp, err := cli.Do(req)
    if err != nil {
        return err
    }
    _ = os.Mkdir(userId, os.ModePerm)
    idx := strings.LastIndexFunc(imgUrl, func(r rune) bool {
        if r == '/' {
            return true
        }
        return false
    })
    f, err := os.Create(fmt.Sprintf("%s/%s", userId, imgUrl[idx+1:]))
    if err != nil {
        return err
    }
    defer func() { _ = f.Close() }()
    if _, err = io.Copy(f, resp.Body); err != nil {
        return err
    }
    return nil
}

func (pcli PixivClient) socks5Transport() *http.Transport {
    httpTransport := &http.Transport{}
    httpTransport.Dial = pcli.Proxy.Dial
    return httpTransport
}

func randSleep(max time.Duration) {
    t := rand.Int63n(int64(max))
    time.Sleep(time.Duration(t))
}

func defaultProxy() proxy.Dialer {
    d, err := proxy.SOCKS5("tcp", "127.0.0.1:1080", nil, proxy.Direct)
    if err != nil {
        panic(err)
    }
    return d
}

```