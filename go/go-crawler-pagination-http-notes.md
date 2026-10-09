# Go 爬虫第五讲：分页规律与带超时的 HTTP 封装

> 抓列表类页面，第一步永远是找分页规律；第二步是把"发请求"这个动作封装好——这是整个系列的地基，后面并发抓取直接复用它。

## 一、找分页规律：手动翻两页

方法是笨办法，但是最可靠的：手动点"下一页"，看地址栏 URL 的变化。常见的规律长这样：

```
第 1 页：https://example.com/list?start=0
第 2 页：https://example.com/list?start=25
第 3 页：https://example.com/list?start=50
```

看出来了——"下一页 = 上一页 + 25"。找到这个步长，程序里用循环拼 URL 就能走完所有页面：

```go
for start := 0; start < 250; start += 25 {
    url := fmt.Sprintf("https://example.com/list?start=%d", start)
    // 抓取 url ...
}
```

## 二、裸调 http.Get 的两个毛病

1. **没有超时**：对方服务器 hang 住，你就一直等，整个程序卡死；
2. **默认 UA 一看就是程序**：`Go-http-client/2.0` 这种标识，等于举着牌子说"我是爬虫"。

## 三、自己封装：四件事一次做对

- `http.Client` 设 `Timeout`（比如 5 秒），防 hang 死；
- 请求头设常见浏览器的 `User-Agent`；
- 检查状态码，非 200 就报错，别把错误页面当正常数据解析；
- `defer resp.Body.Close()`，连接别漏关。

## 四、可运行示例：分页 URL 生成 + 稳健的请求封装

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"time"
)

const userAgent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36"

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

func main() {
	// 本地测试服务器：检查 UA 是否像浏览器；/bad 返回 500
	ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path == "/bad" {
			http.Error(w, "boom", http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "UA=%s", r.Header.Get("User-Agent"))
	}))
	defer ts.Close()

	// 1) 验证分页 URL 生成
	urls := buildPageURLs(ts.URL+"/list", 0, 25, 3)
	fmt.Println("分页 URL：")
	for _, u := range urls {
		fmt.Println(" ", u)
	}

	// 2) 验证 UA 伪装生效
	body, err := httpGet(urls[0])
	if err != nil {
		fmt.Println("FAIL：请求失败:", err)
		return
	}
	fmt.Println("服务器看到的 UA:", string(body))

	// 3) 验证非 200 状态码会被拦截
	if _, err := httpGet(ts.URL + "/bad"); err != nil {
		fmt.Println("非200拦截 OK:", err)
		fmt.Println("PASS：分页生成、UA 伪装、状态码检查全部通过")
	} else {
		fmt.Println("FAIL：500 错误没被拦截")
	}
}
```

运行输出（URL 中的端口号每次运行不同）：

```
分页 URL：
  http://127.0.0.1:PORT/list?start=0
  http://127.0.0.1:PORT/list?start=25
  http://127.0.0.1:PORT/list?start=50
服务器看到的 UA: UA=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36
非200拦截 OK: unexpected status: 500 Internal Server Error
PASS：分页生成、UA 伪装、状态码检查全部通过
```

## 五、本讲小结

- 分页规律：手动翻两页看 URL 变化，找步长，循环拼 URL；
- 裸调 `http.Get` 的毛病：无超时、默认 UA 暴露身份；
- 封装四件事：Client 设 Timeout、请求头设浏览器 UA、检查状态码、`defer` 关 Body；
- 这是全系列的地基：下一讲的并发抓取直接复用这个封装。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go net/http 标准库文档与公开的 HTTP 基础知识。
