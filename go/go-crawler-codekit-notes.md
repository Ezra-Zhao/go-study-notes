# Go 爬虫配套代码包：全系列可运行实现总览

> 前面六讲把链条拆开了：分页规律、HTTP 封装、正则提取、并发框架、合规与礼貌。这一篇把它们拼成一个能直接跑的小工具包——标准库为主，不依赖第三方包，每段都可独立验证。

## 一、工具包的组成

一个文件，四个函数，分工如下：

| 函数 | 职责 | 对应哪一讲 |
|---|---|---|
| `buildPageURLs` | 按步长拼出所有分页 URL | 第五讲 |
| `httpGet` | 带超时、浏览器 UA、状态码检查的请求封装 | 第五讲 |
| `extractItems` | 正则批量提取标题和评分 | 第四讲 |
| `crawlPages` | goroutine 并发抓取 + channel 汇总 + 礼貌延时 | 第六讲（+第三讲） |

`main` 函数把它们串起来：生成 URL → 并发抓取 → 提取 → 打印汇总。

## 二、完整可运行代码

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

const userAgent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36"

var (
	titleRe = regexp.MustCompile(`<span class="title">([^<]+)</span>`)
	scoreRe = regexp.MustCompile(`<span class="rating_num"[^>]*>([^<]+)</span>`)
)

// buildPageURLs：按步长拼出所有分页 URL
func buildPageURLs(base string, start, step, pages int) []string {
	urls := make([]string, 0, pages)
	for i := 0; i < pages; i++ {
		urls = append(urls, fmt.Sprintf("%s?start=%d", base, start+i*step))
	}
	return urls
}

// httpGet：带超时、UA、状态码检查的请求封装
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

// extractItems：从一页 HTML 里批量提取（标题，评分）
func extractItems(html string) [][2]string {
	titles := titleRe.FindAllStringSubmatch(html, -1)
	scores := scoreRe.FindAllStringSubmatch(html, -1)
	var items [][2]string
	for i := range titles {
		score := "?"
		if i < len(scores) {
			score = scores[i][1]
		}
		items = append(items, [2]string{titles[i][1], score})
	}
	return items
}

// crawlPages：并发抓取多页，结果经 channel 汇总
func crawlPages(urls []string) [][2]string {
	var wg sync.WaitGroup
	out := make(chan [2]string, 200)
	go func() {
		wg.Wait()
		close(out)
	}()
	for _, u := range urls {
		wg.Add(1)
		go func(url string) {
			defer wg.Done()
			time.Sleep(100 * time.Millisecond) // 礼貌延时
			body, err := httpGet(url)
			if err != nil {
				fmt.Println("  跳过失败页面:", err)
				return
			}
			for _, it := range extractItems(string(body)) {
				out <- it
			}
		}(u)
	}
	var all [][2]string
	for it := range out {
		all = append(all, it)
	}
	return all
}

func main() {
	// 本地测试服务器：3 个分页，每页 2 条记录
	ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		p := r.URL.Query().Get("start")
		fmt.Fprintf(w, `<div><span class="title">第%s页-电影A</span><span class="rating_num">9.%s</span></div>`+
			`<div><span class="title">第%s页-电影B</span><span class="rating_num" property="v:average">8.%s</span></div>`,
			p, p, p, p)
	}))
	defer ts.Close()

	urls := buildPageURLs(ts.URL+"/list", 0, 25, 3)
	fmt.Println("待抓取", len(urls), "页")
	items := crawlPages(urls)
	fmt.Println("共提取", len(items), "条：")
	for _, it := range items {
		fmt.Printf("  %s  评分 %s\n", it[0], it[1])
	}
	if len(items) == 6 {
		fmt.Println("PASS：工具包端到端跑通")
	} else {
		fmt.Println("FAIL：条目数量不对")
	}
}
```

运行输出（顺序可能不同）：

```
待抓取 3 页
共提取 6 条：
  第0页-电影A  评分 9.0
  第0页-电影B  评分 8.0
  第25页-电影A  评分 9.25
  ...
PASS：工具包端到端跑通
```

## 三、使用与扩展建议

- 把 `ts.URL` 换成真实目标的列表页地址，把正则换成 F12 看到的真实标签特征，就能干活（先读第二讲的合规红线和第三讲的礼貌规范）；
- 想存结果：把 `crawlPages` 返回的切片写进 CSV 或数据库，别只打印；
- 页面结构复杂了：`extractItems` 换成 `goquery` 这类解析库，框架不用动——这就是"生产者/消费者解耦"的好处。

## 四、小结

全系列技术链条，一张图：

**找分页规律（buildPageURLs）→ 稳健发请求（httpGet）→ 正则提取（extractItems）→ 并发加速（crawlPages）**，全程合规先行、礼貌抓取。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go 标准库文档与公开的网络编程常识。
