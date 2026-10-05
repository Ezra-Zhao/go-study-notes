# Go 网络爬虫入门：从正则到并发抓取

> 学这一章，是为了写合规的数据采集工具：技术本身是中立的，用在正道上才有价值。

## 一、爬虫是什么

网络爬虫（Web Crawler）就是一段按照既定规则自动浏览网页、把公开信息抓回来整理好的程序。合规的用途很多：

- 给自己的网站做可用性监控，定时检查页面是否正常；
- 对公开数据做聚合分析，比如公开的商品价格、学术文献目录；
- 搜索引擎的网页索引，本质上也是爬虫。

一句话：爬虫是"替人翻网页的助手"，关键看你让它翻什么、怎么翻。

## 二、为什么用 Go 写爬虫

- **标准库齐全**：`net/http` 发请求、`regexp` 做提取，开箱即用，不用装一堆第三方包；
- **并发轻量**：`goroutine` 启动成本极低，几百个页面同时抓也不费劲，配合 `channel` 收结果，代码比多线程模型清爽得多；
- **部署简单**：编译成单个二进制文件，丢到服务器上就能跑。

## 三、合规红线（先看这节再动手）

### 1. 先看 robots.txt

每个网站根目录下一般都有 `robots.txt`，这是站长贴在门口的"告示牌"，写明哪些路径允许抓、哪些不允许（`Disallow`）。动手前先读它，告示牌上说不让进的地方就别进。

### 2. 个人信息保护的三条原则

按《网络安全法》的精神，自己归纳成三句话：

1. **收集要经过同意**：涉及个人信息的内容，事先要经过本人同意才能收集；
2. **限定范围和用途**：只收集为提供服务所必需的信息，不超范围、不挪作他用；
3. **妥善保管**：收集到的信息要防止泄露、篡改和丢失。

### 3. 做个"有礼貌"的爬虫

- 控制抓取频率，请求之间加延时，别把人家网站打挂；
- 遵守网站的服务条款；
- 不抓取需要登录才能看到的内容，不绕过反爬措施；
- 抓下来的数据自己用，不倒卖、不公开他人隐私。

## 四、分页规律分析

抓列表类页面，第一步是找分页规律。方法是：手动翻两页，观察地址栏 URL 的变化。常见的规律是：

```
第 1 页：https://example.com/list?start=0
第 2 页：https://example.com/list?start=25
第 3 页：https://example.com/list?start=50
```

看出规律了吗——"下一页 = 上一页 + 25"。找到这个步长，程序里用循环拼 URL 就能把所有页面走完：

```go
for start := 0; start < 250; start += 25 {
    url := fmt.Sprintf("https://example.com/list?start=%d", start)
    // 抓取 url ...
}
```

## 五、正则提取思路

拿到网页 HTML 后，第二步是定位想要的数据。方法是：浏览器按 F12 看元素结构，找到目标数据所在的标签特征，然后用正则表达式匹配。

比如某榜单页，条目名称在 `<span class="title">` 里，评分在 `<span class="rating_num">` 里：

```go
var (
    titleRe = regexp.MustCompile(`<span class="title">([^<]+)</span>`)
    scoreRe = regexp.MustCompile(`<span class="rating_num"[^>]*>([^<]+)</span>`)
)

titles := titleRe.FindAllStringSubmatch(html, -1)
scores := scoreRe.FindAllStringSubmatch(html, -1)
for i := range titles {
    fmt.Println(titles[i][1], scores[i][1])
}
```

`([^<]+)` 是核心技巧：匹配"尖括号之前的所有字符"，正好把标签里的文本抠出来。提醒一句：正则适合结构简单的页面；HTML 结构复杂时，建议改用专门的解析库（如 `golang.org/x/net/html` 或 `goquery`），比正则更稳。

## 六、带超时和 UA 的 HTTP 封装

直接裸调 `http.Get` 有两个毛病：没有超时（对方服务器 hang 住你就一直等），默认 UA 一看就是程序。自己封装一个：

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "time"
)

const userAgent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36"

func httpGet(url string) ([]byte, error) {
    client := &http.Client{Timeout: 5 * time.Second}
    req, err := http.NewRequest(http.MethodGet, url, nil)
    if err != nil {
        return nil, err
    }
    req.Header.Set("User-Agent", userAgent)
    resp, err := client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("unexpected status: %s", resp.Status)
    }
    return io.ReadAll(resp.Body)
}
```

要点：`http.Client` 设 `Timeout` 防 hang 死；`User-Agent` 设成常见浏览器标识；检查状态码；`defer resp.Body.Close()` 别漏。

## 七、channel 并发抓取框架

页面之间没有依赖关系，正好并发。`goroutine` 负责抓，`channel` 负责收结果，`WaitGroup` 负责等全部结束：

```go
package main

import (
    "fmt"
    "regexp"
    "sync"
)

var (
    titleRe = regexp.MustCompile(`<span class="title">([^<]+)</span>`)
    scoreRe = regexp.MustCompile(`<span class="rating_num"[^>]*>([^<]+)</span>`)
)

type Item struct {
    Title string
    Score string
}

func crawlPage(start int, out chan<- Item, wg *sync.WaitGroup) {
    defer wg.Done()
    url := fmt.Sprintf("https://example.com/list?start=%d", start)
    body, err := httpGet(url) // 复用第六节的封装
    if err != nil {
        fmt.Printf("page %d failed: %v\n", start, err)
        return
    }
    html := string(body)
    titles := titleRe.FindAllStringSubmatch(html, -1)
    scores := scoreRe.FindAllStringSubmatch(html, -1)
    for i := range titles {
        it := Item{Title: titles[i][1]}
        if i < len(scores) {
            it.Score = scores[i][1]
        }
        out <- it
    }
}

func main() {
    var wg sync.WaitGroup
    out := make(chan Item, 100)

    go func() {
        wg.Wait()
        close(out) // 所有页面抓完才关 channel
    }()

    for start := 0; start < 250; start += 25 {
        wg.Add(1)
        go crawlPage(start, out, &wg)
    }

    for it := range out {
        fmt.Printf("%s\t%s\n", it.Title, it.Score)
    }
}
```

这个框架的精髓：生产者（抓取 goroutine）和消费者（打印循环）通过 channel 解耦；`wg.Wait()` 之后关 channel，消费者用 `range` 自然结束，不会死锁。实际使用时记得在循环里加延时、控制并发数，别对目标网站造成压力。

## 八、小结

爬虫技术链条很短：**找分页规律 → 发请求 → 正则提取 → 并发加速**。Go 的标准库把每一步都包圆了。真正拉开差距的不是代码技巧，而是第三节的合规意识——先看 robots.txt、尊重个人信息、做个有礼貌的爬虫，工具才能长久地用在正道上。

本文为学习笔记（编纂），知识点源自公开的 Go 标准库文档与网络安全常识；
