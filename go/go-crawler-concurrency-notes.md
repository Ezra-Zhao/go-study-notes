# Go 爬虫第六讲：channel + goroutine 并发抓取框架

> 列表页之间没有依赖关系，一个页面抓完才能抓下一个纯属浪费时间。goroutine 负责抓、channel 负责收结果、WaitGroup 负责等全部结束——这就是并发抓取的完整框架。

## 一、分工

- **生产者**：每个 `goroutine` 抓一页，把解析出的条目发进 `channel`；
- **消费者**：主流程用 `range` 从 `channel` 里取结果，处理（打印、入库）；
- **协调者**：`sync.WaitGroup` 等所有生产者干完，然后**关 channel**，消费者自然结束，不会死锁。

生产者和消费者经 channel 解耦：生产者只管发，消费者只管收，谁也不用等谁。

## 二、关 channel 的时机是关键

顺序不能错：

1. 所有抓取 goroutine `wg.Done()`；
2. 另起一个 goroutine 做 `wg.Wait()`，等完就 `close(out)`；
3. 主流程 `for it := range out` 消费，channel 一关，循环自然退出。

如果关早了——还有 goroutine 往里发——程序直接 panic。记住：**全部抓完才关**。

## 三、实战提醒

- 循环里加延时、控制并发数，别对目标网站造成压力（呼应第三讲"有礼貌的爬虫"）；
- 单个页面抓取失败只记日志、跳过，别让一个坏页面拖死整批任务。

## 四、可运行示例：并发抓取 4 个分页并汇总

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"regexp"
	"sync"
	"time"
)

var itemRe = regexp.MustCompile(`<li>([^<]+)</li>`)

type Item struct {
	Page  int
	Title string
}

func httpGet(url string) ([]byte, error) {
	client := &http.Client{Timeout: 5 * time.Second}
	req, _ := http.NewRequest(http.MethodGet, url, nil)
	req.Header.Set("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0")
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

func crawlPage(page int, url string, out chan<- Item, wg *sync.WaitGroup) {
	defer wg.Done()
	time.Sleep(100 * time.Millisecond) // 礼貌延时：别压垮对方
	body, err := httpGet(url)
	if err != nil {
		fmt.Printf("第 %d 页失败，跳过: %v\n", page, err)
		return
	}
	for _, m := range itemRe.FindAllStringSubmatch(string(body), -1) {
		out <- Item{Page: page, Title: m[1]}
	}
}

func main() {
	// 本地测试服务器：/list?page=N 返回该页的 2 个条目
	ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		p := r.URL.Query().Get("page")
		fmt.Fprintf(w, "<ul><li>第%s页-条目A</li><li>第%s页-条目B</li></ul>", p, p)
	}))
	defer ts.Close()

	var wg sync.WaitGroup
	out := make(chan Item, 100)

	// 协调者：等全部抓完再关 channel
	go func() {
		wg.Wait()
		close(out)
	}()

	// 生产者：每页一个 goroutine
	for page := 1; page <= 4; page++ {
		wg.Add(1)
		go crawlPage(page, fmt.Sprintf("%s/list?page=%d", ts.URL, page), out, &wg)
	}

	// 消费者：range 取结果，channel 关闭后自然结束
	count := 0
	for it := range out {
		fmt.Printf("  第%d页: %s\n", it.Page, it.Title)
		count++
	}
	if count == 8 {
		fmt.Println("PASS：4 页 × 2 条 = 8 条，全部收齐，无死锁")
	} else {
		fmt.Printf("FAIL：只收到 %d 条\n", count)
	}
}
```

运行输出（各页到达顺序可能不同，这是并发的正常表现）：

```
  第2页: 第2页-条目A
  第2页: 第2页-条目B
  第1页: 第1页-条目A
  ...
PASS：4 页 × 2 条 = 8 条，全部收齐，无死锁
```

输出顺序不固定恰恰证明了并发在工作：谁先抓完谁先交结果。

## 五、本讲小结与全系列收束

- 分工：goroutine 抓、channel 收、WaitGroup 等、抓完关 channel；
- 关键顺序：`wg.Wait()` 之后才 `close`，消费者 `range` 自然结束；
- 实战：加延时、控并发、单页失败只跳过；
- 全系列链条打通：**找分页规律 → 发请求 → 正则提取 → 并发加速**。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go 并发编程公开资料与 net/http 标准库文档。
