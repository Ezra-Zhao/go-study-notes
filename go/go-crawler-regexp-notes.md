# Go 爬虫第四讲：用正则从网页里抠数据

> 拿到网页 HTML 只是拿到了原材料，正则表达式是把目标数据从一堆标签里抠出来的镊子。

## 一、思路就两步

1. **看结构**：浏览器按 F12 打开开发者工具，找到目标数据所在的标签，看它的特征（标签名、class、id）；
2. **写正则**：按这个特征写匹配模式，把标签里的文本抠出来。

## 二、核心技巧：否定尖括号

网页里的数据一般长这样：`<span class="title">复仇者联盟</span>`。想要"复仇者联盟"这几个字，模式可以写成：

```
<span class="title">([^<]+)</span>
```

`([^<]+)` 是精髓：匹配"尖括号之前的所有字符"——正好把标签里的纯文本抠出来，标签本身不会混进来。

## 三、Go 侧的做法

- 用标准库 `regexp` 包；
- 正则**编译一次、反复复用**（`regexp.MustCompile` 放在包级变量里，别在循环里反复编译）；
- `FindAllStringSubmatch` 一次把页面里所有匹配项全抓出来，返回二维切片，`m[i][1]` 就是第 i 个匹配的括号分组内容。

## 四、适用边界

正则适合结构简单的页面，写起来快、依赖少。但如果 HTML 结构复杂、嵌套很深，正则会越写越脆——这时改用专门的 HTML 解析库（如 `golang.org/x/net/html` 或 `goquery`）更稳。工具没有高低，只有合不合适。

## 五、可运行示例：从榜单页提取名称和评分

```go
package main

import (
	"fmt"
	"regexp"
)

var (
	titleRe = regexp.MustCompile(`<span class="title">([^<]+)</span>`)
	scoreRe = regexp.MustCompile(`<span class="rating_num"[^>]*>([^<]+)</span>`)
	linkRe  = regexp.MustCompile(`<a href="([^"]+)"[^>]*class="cover"`)
)

func main() {
	// 模拟从网页上拿到的 HTML 片段
	html := `
	<div class="item"><a href="/movie/1" class="cover"><img></a>
	<span class="title">星际穿越</span><span class="rating_num" property="v:average">9.3</span></div>
	<div class="item"><a href="/movie/2" class="cover"><img></a>
	<span class="title">盗梦空间</span><span class="rating_num" property="v:average">9.2</span></div>
	<div class="item"><a href="/movie/3" class="cover"><img></a>
	<span class="title">泰坦尼克号</span><span class="rating_num">9.1</span></div>`

	titles := titleRe.FindAllStringSubmatch(html, -1)
	scores := scoreRe.FindAllStringSubmatch(html, -1)
	links := linkRe.FindAllStringSubmatch(html, -1)

	fmt.Printf("提取到 %d 条记录：\n", len(titles))
	ok := len(titles) == 3 && len(scores) == 3 && len(links) == 3
	for i := range titles {
		score := "?"
		if i < len(scores) {
			score = scores[i][1]
		}
		link := "?"
		if i < len(links) {
			link = links[i][1]
		}
		fmt.Printf("  %d. %s  评分 %s  链接 %s\n", i+1, titles[i][1], score, link)
	}
	if ok {
		fmt.Println("PASS：三类字段全部提取成功")
	} else {
		fmt.Println("FAIL：提取结果不完整")
	}
}
```

运行输出：

```
提取到 3 条记录：
  1. 星际穿越  评分 9.3  链接 /movie/1
  2. 盗梦空间  评分 9.2  链接 /movie/2
  3. 泰坦尼克号  评分 9.1  链接 /movie/3
PASS：三类字段全部提取成功
```

注意 `scoreRe` 里的 `[^>]*`：它容忍了标签里多出来的属性（比如 `property="v:average"`），这就是写正则时留一点"弹性"的好处——页面小改版不至于全军覆没。

## 六、本讲小结

- 两步走：F12 看结构定位标签特征，再写正则匹配；
- 核心技巧 `([^<]+)`：抠出标签内文本；
- Go 做法：`regexp` 包，编译一次复用，`FindAllStringSubmatch` 批量提取；
- 边界：页面结构简单用正则，复杂了换专用解析库。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go regexp 标准库文档与公开的正则表达式教程。
