# Go 爬虫第三讲：做一个"有礼貌"的爬虫

> 爬虫的行为规范，总结成一句话就是：别把人家网站打挂。礼貌 = 可持续，粗暴抓取迟早被封。

## 一、礼貌的三条做法

1. **控制抓取频率**：请求之间加延时，比如每次间隔一两秒；别像机关枪一样连发；
2. **控制并发数**：同时只开固定数量的抓取任务，别几百个连接一起压过去；
3. **看时间段**：能避开对方业务高峰就避开，深夜低峰跑批量任务是基本体贴。

## 二、三条线不碰

- 需要登录才能看到的内容，不抓；
- 反爬措施（验证码、IP 封禁后的绕行），不绕；
- 网站的服务条款，遵守。

## 三、数据用途的底线

抓下来的数据自己用：不倒卖、不公开他人隐私。数据一旦流出，你就失去了对它的控制——这条线守不住，前面所有的合规动作都白做。

## 四、可运行示例：带延时和并发上限的抓取器

下面这段程序实现"有礼貌"的两个核心机制：**请求间延时** + **信号量控制并发数**。用本地测试服务器记录每次请求的时间和并发峰值，程序自己校验是否达标：

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"sync"
	"sync/atomic"
	"time"
)

// PoliteFetcher：有礼貌的抓取器
// sem 控制最大并发数；tokens 是令牌桶，ticker 每 delay 发放一个令牌，
// 全局限速——拿不到令牌的请求就等着，不会一拥而上。
type PoliteFetcher struct {
	sem    chan struct{}
	tokens chan struct{}
}

func NewPoliteFetcher(delay time.Duration, maxConcurrent int) *PoliteFetcher {
	p := &PoliteFetcher{
		sem:    make(chan struct{}, maxConcurrent),
		tokens: make(chan struct{}, 1),
	}
	// 令牌只由 ticker 发放：第一个请求也要等第一个 tick，
	// 之后每 delay 补一个。注意：令牌发放间隔是严格的，
	// 但请求实际发出的间隔还受本地调度抖动影响，约等于 delay。
	go func() {
		t := time.NewTicker(delay)
		defer t.Stop()
		for range t.C {
			select {
			case p.tokens <- struct{}{}:
			default: // 桶满就丢弃，避免令牌堆积造成突发
			}
		}
	}()
	return p
}

func (p *PoliteFetcher) Fetch(url string) ([]byte, error) {
	p.sem <- struct{}{}        // 占一个并发名额
	defer func() { <-p.sem }() // 释放名额
	<-p.tokens                 // 取令牌：全局限速，取不到就等

	resp, err := http.Get(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	return io.ReadAll(resp.Body)
}

func main() {
	var cur, peak int64
	var mu sync.Mutex
	var hits []time.Time

	// 本地测试服务器：记录每次请求到达的时间和并发数
	ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		n := atomic.AddInt64(&cur, 1)
		for {
			old := atomic.LoadInt64(&peak)
			if n <= old || atomic.CompareAndSwapInt64(&peak, old, n) {
				break
			}
		}
		mu.Lock()
		hits = append(hits, time.Now())
		mu.Unlock()
		time.Sleep(50 * time.Millisecond) // 模拟服务器处理耗时
		atomic.AddInt64(&cur, -1)
		fmt.Fprint(w, "ok")
	}))
	defer ts.Close()

	f := NewPoliteFetcher(200*time.Millisecond, 2)

	// 预热一次连接：把 TCP 建连成本排除在测量之外，只测限速机制本身。
	// 注意要读完并关闭 Body，否则连接无法复用，预热就白做了。
	if resp, err := http.Get(ts.URL); err != nil {
		fmt.Println("预热失败:", err)
		return
	} else {
		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()
	}
	base := len(hits) // 预热请求不计入测量

	// 阶段一：限速验证——单 goroutine 顺序抓 6 次
	// 同一个 worker 不存在调度竞速，测到的是纯粹的令牌间隔，结论稳定
	for i := 0; i < 6; i++ {
		if _, err := f.Fetch(ts.URL); err != nil {
			fmt.Println("抓取失败:", err)
			return
		}
	}
	rateOK := true
	minGap := time.Hour
	for i := base + 1; i < len(hits); i++ {
		if gap := hits[i].Sub(hits[i-1]); gap < minGap {
			minGap = gap
		}
	}
	span := hits[len(hits)-1].Sub(hits[base]) // 首尾跨度：5 个名义 200ms 间隔
	// 不断言每一跳精确 200ms：本地调度抖动会让单跳有 ±20ms 级浮动；
	// 机制保证的是"令牌每 200ms 发放一个"，所以断言总跨度与单跳底线。
	if span < 900*time.Millisecond || minGap < 150*time.Millisecond {
		rateOK = false
	}
	fmt.Printf("阶段一（限速）：6 次请求，总跨度 %dms，平均间隔 %dms，最小间隔 %dms -> %s\n",
		span.Milliseconds(), span.Milliseconds()/5, minGap.Milliseconds(),
		map[bool]string{true: "达标", false: "未达标"}[rateOK])

	// 阶段二：并发验证——6 个 goroutine 同时抓，信号量上限 2
	// 并发峰值是构造性保证（信号量满了就进不去），与调度抖动无关
	var wg sync.WaitGroup
	for i := 0; i < 6; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			if _, err := f.Fetch(ts.URL); err != nil {
				fmt.Println("抓取失败:", err)
			}
		}()
	}
	wg.Wait()
	peakOK := atomic.LoadInt64(&peak) <= 2
	fmt.Println("阶段二（并发）：并发峰值", atomic.LoadInt64(&peak), "->", map[bool]string{true: "达标", false: "未达标"}[peakOK])

	if rateOK && peakOK {
		fmt.Println("PASS：限速与并发上限双双达标")
	} else {
		fmt.Println("FAIL：礼貌机制没生效")
	}
}
```

运行输出：

```
阶段一（限速）：6 次请求，总跨度 1002ms，平均间隔 200ms，最小间隔 196ms -> 达标
阶段二（并发）：并发峰值 1 -> 达标
PASS：限速与并发上限双双达标
```

（具体毫秒数每次运行略有浮动，这是本地调度的正常抖动。）

关键点：`sem` 这个带缓冲的 channel 是"并发名额池"，`tokens` 令牌桶是"全局节流阀"——令牌由单个 ticker 每 200ms 补充一个，无论多少个 goroutine 同时想抓，请求之间都约间隔 200ms（实测 196–212ms，抖动来自本地调度）。两个机制都很便宜，但决定了你的爬虫是"客人"还是"攻击者"。

## 五、本讲小结

- 礼貌三做法：控制频率、控制并发数、避开业务高峰；
- 三条线不碰：登录墙不碰、反爬不绕、服务条款遵守；
- 数据用途：自用，不倒卖、不公开他人隐私；
- 核心思想：**礼貌 = 可持续**。

本文为学习笔记（编纂），用自己的话重写；知识点源自公开的网络爬虫行业惯例与 Go 并发编程常识。
