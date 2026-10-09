# Go 爬虫第二讲：动手写代码前的法律红线

> 爬虫的第一课不是代码，是红线。红线看清楚了，技术才能长久地用。

## 一、网站的"告示牌"：robots.txt

几乎每个网站根目录下都有一个 `robots.txt`，这是站长贴在门口的告示牌：写明哪些路径允许抓取、哪些不允许（`Disallow`）。

动手前的第一动作：**先读它**。告示牌上说不让进的地方，就别进。这不是法律条文的字面强制，而是一个基本的行业约定——尊重它，你的爬虫才不会一开始就站到对立面。

## 二、个人信息的三条原则

按《网络安全法》的精神，自己归纳成三句话，贴在工位上：

1. **收集要经过同意**：涉及个人信息的内容，事先要经过本人同意才能收集；
2. **限定范围和用途**：只收集为提供服务所必需的信息，不超范围收集、不挪作他用；
3. **妥善保管**：拿到手的信息要防止泄露、篡改和丢失，存下来了就要负责。

## 三、另外三条硬线

- **遵守网站服务条款**：人家条款里写明不许抓，就别抓；
- **不碰登录墙**：需要登录才能看到的内容不抓，不绕过反爬措施；
- **数据自用**：抓下来的数据自己用，不倒卖、不公开他人隐私。

## 四、可运行示例：读懂一个网站的 robots.txt

下面这段程序抓取目标网站的 `robots.txt` 并逐行列出规则，遇到 `Disallow` 的路径就标出来提醒。示例用本地测试服务器代替真实网站：

```go
package main

import (
	"bufio"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"
)

func checkRobots(base string) {
	resp, err := http.Get(base + "/robots.txt")
	if err != nil {
		fmt.Println("robots.txt 获取失败:", err)
		return
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		fmt.Println("该网站没有 robots.txt（状态码:", resp.Status, "）")
		return
	}
	fmt.Println("robots.txt 规则：")
	sc := bufio.NewScanner(resp.Body)
	for sc.Scan() {
		line := strings.TrimSpace(sc.Text())
		if line == "" || strings.HasPrefix(line, "#") {
			continue
		}
		mark := ""
		if strings.HasPrefix(strings.ToLower(line), "disallow") {
			mark = "   <-- 禁止抓取的路径，动手时避开"
		}
		fmt.Println(" ", line, mark)
	}
}

func main() {
	// 本地测试服务器：模拟一个网站的 robots.txt
	ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "User-agent: *\nDisallow: /admin/\nDisallow: /user/\nAllow: /public/\n")
	}))
	defer ts.Close()

	checkRobots(ts.URL)
}
```

运行输出：

```
robots.txt 规则：
  User-agent: *
  Disallow: /admin/    <-- 禁止抓取的路径，动手时避开
  Disallow: /user/    <-- 禁止抓取的路径，动手时避开
  Allow: /public/
```

动手前跑一遍这个小工具，十秒钟，心里就有数了。

## 五、本讲小结

- 先读 `robots.txt`：网站门口的告示牌，`Disallow` 的路径不进；
- 个人信息三原则：经同意、限范围、妥保管；
- 三条硬线：遵守服务条款、不碰登录墙、不绕反爬；数据自用，不倒卖、不公开他人隐私；
- 结论：技术是中立的，用在正道上才有价值；合规是爬虫能长期使用的前提。

本文为学习笔记（编纂），用自己的话重写；知识点源自公开的 robots 协议说明与《网络安全法》公开条文精神。
