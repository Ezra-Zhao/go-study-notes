# Python 网络数据采集学习笔记

> 资料定位：本笔记依据某网络数据采集资料的章节结构整理，覆盖从"发出第一个请求"到"解析、存储、清洗与合规"的完整链路。代码示例只用 Python 标准库（urllib / re / json / csv / sqlite3 / html.parser），全部可在本机离线验证；凡是需要真实联网的示例均已标注"需联网环境"。
>
> 合规声明：采集必须遵守目标网站的 robots.txt 与服务条款，控制请求频率，不碰付费墙后内容与个人隐私数据。本笔记只讲技术原理与合规做法。

## 一、初见网络爬虫：请求与响应的循环

**先理解 HTTP 再谈爬虫。** 浏览器打开网页，本质是发一个 HTTP 请求、收到一个 HTTP 响应。响应分三部分：状态码（200 成功、404 不存在、403 被拒绝、500 服务器错）、响应头（内容类型、编码等元信息）、正文（HTML 文本）。爬虫干的活和浏览器一样，只是把"渲染给人看"换成"提取数据存下来"。

**爬虫 = 请求 → 解析 → 提取 → 存储的循环。** 先请求拿到 HTML，再解析出想要的字段，然后存进文件或数据库，最后顺着页面里的链接去抓下一页。整个链条里，"解析"和"礼貌"是最需要下功夫的两环。

下面是发请求的最小形态（**需联网环境**，逻辑已核对，直接跑需要能访问外网）：

```python
import urllib.request

# 需联网环境：真实请求目标网站时，带上 UA 头是基本礼貌
req = urllib.request.Request(
    "https://example.com",
    headers={"User-Agent": "Mozilla/5.0 (学习用途的采集脚本)"},
)
with urllib.request.urlopen(req, timeout=10) as resp:
    print(resp.status)                       # 200
    print(resp.headers.get_content_charset())  # 告诉你正文是什么编码
    html = resp.read().decode("utf-8", errors="replace")
```

三个细节决定成败：**超时必须设**（`timeout=10`，否则请求 hang 住整个程序卡死）、**UA 头要带**（不带会被很多站点直接拒绝）、**解码用 errors="replace" 保底**（坏字节不至于让程序崩掉）。

## 二、复杂 HTML 解析：正则与结构化提取

**正则适合"模式固定"的小片段。** 比如从一堆 HTML 里抠标题和链接，正则又快又直接。但有两个铁律：用**非贪婪** `.*?`（贪婪的 `.*` 会一口吞掉半个页面），涉及跨行匹配时加 `re.S` 让 `.` 能匹配换行：

```python
import re

html = """
<html><head><title>今日热点</title></head>
<body>
<h2>新闻一</h2><a href="/news/1">详情</a>
<h2>新闻二</h2><a href="/news/2">详情</a>
</body></html>
"""

title = re.search(r"<title>(.*?)</title>", html, re.S).group(1)
links = re.findall(r'<a\s+href="([^"]+)">', html)   # 括号分组 = 只取括号里的部分
headings = re.findall(r"<h2>(.*?)</h2>", html, re.S)
print(title)     # 今日热点
print(links)     # ['/news/1', '/news/2']
print(headings)  # ['新闻一', '新闻二']
```

**整页结构用解析器。** 正则写复杂了会变成天书，大段 HTML 应该按"标签树"来走。标准库的 `html.parser` 就能写个小解析器，比如收集页面里所有链接：

```python
from html.parser import HTMLParser


class LinkCollector(HTMLParser):
    def __init__(self):
        super().__init__()
        self.links = []

    def handle_starttag(self, tag, attrs):
        if tag == "a":
            for name, value in attrs:
                if name == "href":
                    self.links.append(value)


collector = LinkCollector()
collector.feed(html)
print(collector.links)  # ['/news/1', '/news/2']
```

经验法则：**抠零散小字段用正则，走整页结构用解析器**。另外 `filter` + `lambda` 组合很适合做"按条件筛标签"这类一次性过滤，写起来比循环清爽。

## 三、采集策略：从单页到全站的礼貌爬法

**广度优先（BFS）是爬全站的标准姿势。** 维护一个待抓队列和一个"已访问"集合：取出一个 URL，抓下来，提取其中的站内链接入队，循环直到队列清空。已访问集合用来去重，不然 A 链 B、B 链 A 会让你死循环。

**URL 规范化是去重的前提。** `https://EXAMPLE.com/a?b=2&a=1` 和 `https://example.com/a?a=1&b=2#top` 是同一个页面，不规范化就会重复抓：

```python
from urllib.parse import urljoin, urlparse, parse_qsl, urlencode, urlunparse


def normalize_url(url):
    p = urlparse(url)
    query = urlencode(sorted(parse_qsl(p.query)))  # 参数排序，去 fragment
    return urlunparse((p.scheme, p.netloc.lower(), p.path, "", query, ""))


print(normalize_url("HTTPS://Example.COM/a?b=2&a=1#x"))
# https://example.com/a?a=1&b=2
print(urljoin("https://example.com/news/", "../about"))
# https://example.com/about（相对链接转绝对链接）
```

**礼貌三件套**：请求之间 `sleep` 几秒（别打垮人家服务器）、限制抓取深度（别把整站翻个底朝天）、只跟站内链接（`urljoin` 转绝对后比对域名）。下面是个离线可跑的 BFS 框架，用字典模拟"已抓取的页面"：

```python
import re
from urllib.parse import urljoin, urlparse


def crawl(start_url, fake_web, max_depth=2):
    seen, queue, order = set(), [(start_url, 0)], []
    while queue:
        url, depth = queue.pop(0)
        url = normalize_url(url)
        if url in seen or depth > max_depth:
            continue
        seen.add(url)
        order.append(url)
        page = fake_web.get(url, "")
        for href in re.findall(r'href="([^"]+)"', page):
            nxt = urljoin(url, href)
            if urlparse(nxt).netloc == urlparse(start_url).netloc:
                queue.append((nxt, depth + 1))
    return order


fake_web = {
    "https://example.com/": '<a href="/a">a</a><a href="/b">b</a>'
                            '<a href="https://other.com/x">站外</a>',
    "https://example.com/a": '<a href="/c">c</a>',
    "https://example.com/b": "no links",
    "https://example.com/c": "end",
}
print(crawl("https://example.com/", fake_web))
# ['https://example.com/', '.../a', '.../b', '.../c']，站外链接被过滤
```

真实采集时，在循环里加 `time.sleep(2)` 并处理请求异常，这个骨架就能直接上岗。

## 四、用 API 拿数据：能调接口就别爬页面

很多网站的数据来自后端 API 接口，返回的是干净的 JSON。**能调 API 就别解析 HTML**：结构稳定、字段齐全、还没反爬。找接口的方法：浏览器开发者工具看 Network 面板，刷新页面，找返回 JSON 的请求。

JSON 处理是标准库 `json` 的主场：

```python
import json

api_response = """
{"status": "ok", "page": 2, "items": [
  {"id": 101, "title": "Python 入门", "price": "¥1,234.56"},
  {"id": 102, "title": "数据采集实战", "price": "¥99.00"}
]}
"""
data = json.loads(api_response)
print(data["status"])                        # ok
print([item["title"] for item in data["items"]])
```

注意两点：API 一般分页（`page` 参数循环拉到底）、一般限流（返回 429 就是让你慢点，加 sleep 重试）。

## 五、存储数据：CSV 与 SQLite

**CSV 存表格数据。** 标准库 `csv` 模块够用：写时注意 `newline=""`（否则 Windows 下多空行）、编码用 `utf-8-sig`（Excel 打开中文不乱码）：

```python
import csv, io

buf = io.StringIO()
writer = csv.writer(buf)
writer.writerow(["id", "title", "price"])
for item in data["items"]:
    writer.writerow([item["id"], item["title"], item["price"]])
print(buf.getvalue())
```

**SQLite 存结构化数据。** 标准库 `sqlite3` 零依赖，建表、批量插入、条件查询一条龙，几万条数据内完全够用：

```python
import sqlite3

conn = sqlite3.connect(":memory:")  # 内存库演示，落盘把路径换成文件名
conn.execute("CREATE TABLE items (id INTEGER PRIMARY KEY, title TEXT, price REAL)")
conn.executemany(
    "INSERT INTO items (id, title, price) VALUES (?, ?, ?)",
    [(101, "Python 入门", 1234.56), (102, "数据采集实战", 99.0)],
)
conn.commit()
print(conn.execute("SELECT title FROM items WHERE price < 1000").fetchall())
# [('数据采集实战',)]
conn.close()
```

占位符 `?` 传参既能防 SQL 注入，又省了拼字符串的麻烦。媒体文件（图片/视频）直接按二进制存文件，数据库里只记路径；需要 MySQL 这类服务端数据库时再引入第三方驱动。

## 六、读取文档：编码是第一关

抓下来的不一定是 HTML，可能是纯文本、CSV、PDF、Word。**第一步永远是先搞定编码**：看响应头的 charset 声明，拿不到就按常见编码试，`errors="replace"` 保底：

```python
def smart_decode(raw_bytes):
    for encoding in ("utf-8", "gbk"):
        try:
            return raw_bytes.decode(encoding), encoding
        except UnicodeDecodeError:
            continue
    return raw_bytes.decode("utf-8", errors="replace"), "utf-8(replace)"


print(smart_decode("你好, world".encode("utf-8")))  # ('你好, world', 'utf-8')
print(smart_decode("你好".encode("gbk")))            # ('你好', 'gbk')
```

中文站 utf-8 和 gbk 覆盖九成情况。纯文本按行处理、CSV 用 `csv` 模块，PDF/Word 需要专用解析库（概念上知道"有专门工具干这个"即可，按需再学）。

## 七、数据清洗：脏数据不洗没法用

爬下来的数据一定是脏的：价格带货币符号和逗号、前后有多余空白、日期格式五花八门。清洗函数的写法要点：**输入输出明确、可单元测试**：

```python
import re


def clean_price(raw):
    cleaned = re.sub(r"[^\d.]", "", raw)  # 只留数字和小数点
    return float(cleaned) if cleaned else 0.0


assert clean_price("¥1,234.56") == 1234.56
assert clean_price("$99.00") == 99.0
assert clean_price("") == 0.0
print("清洗函数自测通过")
```

清洗清单：去重（按唯一键）、去空值、统一格式（日期转 ISO、价格转数字）、`strip()` 去空白。每个清洗函数配几个 `assert`，数据 pipeline 的可靠性就立住了。

## 八、自然语言处理入门：马尔可夫链

**思想一句话**：统计"当前词后面常跟什么词"，然后按这个概率随机游走，就能生成"像那么回事"的文本。这是理解更复杂语言模型的一个好起点，纯 Python 几十行就能实现：

```python
import random


def build_markov(words):
    model = {}
    for a, b, c in zip(words, words[1:], words[2:]):
        model.setdefault((a, b), []).append(c)  # (前两字 -> 可能的第三字)
    return model


def generate(model, start, length=12, seed=7):
    random.seed(seed)
    a, b = start
    out = [a, b]
    for _ in range(length):
        nxt = model.get((a, b))
        if not nxt:
            break
        c = random.choice(nxt)
        out.append(c)
        a, b = b, c
    return "".join(out)


chars = list("今天天气很好我们去公园散步今天天气很好适合放风筝")
model = build_markov(chars)
print(generate(model, ("今", "天")))  # 今天天气很好我们去公园散步今
```

固定种子让输出可复现，方便验证。语料越大、阶数越高，生成的文本越通顺——但本质还是"统计接龙"，别指望它理解语义。

## 九、表单、登录与 Cookie：概念篇

有些数据在登录后才能看。原理：登录是往服务器 POST 一组用户名密码，服务器返回 Cookie（或 token），之后每个请求带上它，服务器就认得你了。标准库可以用 `http.cookiejar` + `urllib.request` 的 Cookie 处理器自动管理 Cookie（**需联网环境**）：

```python
import http.cookiejar
import urllib.request

# 需联网环境：概念演示，目标地址需换成真实登录接口
jar = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))
login_data = urllib.parse.urlencode({"user": "demo", "pwd": "demo"}).encode()
# opener.open("https://example.com/login", data=login_data, timeout=10)
# 之后再用 opener.open 抓登录后的页面，Cookie 会自动带上
print("Cookie 处理器已就绪，jar 当前为空:", len(jar) == 0)
```

红线：**只用自己的账号登录，只采公开或已授权的数据**；把账号密码、Cookie 写进公开代码是最蠢的泄露方式。

## 十、动态页面：数据不在 HTML 里是常态

现代网站大量用 JavaScript 动态加载：你看到的页面是 JS 跑起来之后的样子，而爬虫拿到的初始 HTML 可能是个空壳。排查思路按顺序来：

1. **先找 Ajax 接口**：开发者工具 Network 面板里，数据往往来自某个返回 JSON 的接口，直接调它（回到第四节）；
2. **看页面内嵌数据**：有的站点把 JSON 直接塞在 `<script>` 标签里，正则抠出来就是；
3. **最后才上重武器**：用无头浏览器真实渲染页面再抓，成本最高，按需使用。

记住判断句式："页面源码里搜不到想要的文字" = 数据是动态加载的，别再对着空 HTML 写解析器了。

## 十一、图像与验证码：概念篇

验证码存在的意义就是"不让你用机器过"，看见它先想想"这个数据我是不是该拿"。技术上，简单验证码的思路是：灰度化 → 二值化降噪 → 字符分割 → 识别（OCR）。复杂验证码（滑块、点选）本质是对抗自动化，**尊重它的设计意图**，不要研究绕过登录验证的手段——那是另一条赛道，且多半违法。

## 十二、合规与道德：爬虫的红线

1. **读 robots.txt**：站点根目录下的这个文本文件声明了"不欢迎爬虫的路径"，`Disallow` 的就别去。下面是个离线可验证的解析小函数：
   ```python
   def allowed_by_robots(robots_txt, path):
       disallows, in_wildcard = [], False
       for line in robots_txt.splitlines():
           line = line.strip()
           if not line or line.startswith("#"):
               continue
           if line.lower().startswith("user-agent:"):
               in_wildcard = line.split(":", 1)[1].strip() == "*"
           elif in_wildcard and line.lower().startswith("disallow:"):
               rule = line.split(":", 1)[1].strip()
               if rule:
                   disallows.append(rule)
       return not any(path.startswith(d) for d in disallows)


   robots = "User-agent: *\nDisallow: /admin/\nDisallow: /private\n"
   assert allowed_by_robots(robots, "/news/1") is True
   assert allowed_by_robots(robots, "/admin/dashboard") is False
   print("robots.txt 解析自测通过")
   ```
2. **控制频率**：请求间隔几秒是底线，别把人家小网站打挂了；
3. **遵守服务条款与版权法**：条款明确禁止采集的，别硬来；
4. **个人数据别碰**：姓名、电话、住址这类信息，采了就是麻烦；
5. **伪装 UA 可以，伪造身份不行**：改个请求头属于常规操作，绕过付费墙、盗用他人 Cookie 属于越界。

## 常见坑

1. **正则贪婪匹配**：`.*` 一口吞掉半个页面，提取 HTML 片段一律用非贪婪 `.*?`。
2. **忘记设超时**：`urlopen` 不带 `timeout`，一个 hang 住的请求卡死整个程序。
3. **编码猜错**：先看响应头 charset，再试 utf-8/gbk，最后 `errors="replace"` 保底，别硬 `decode("utf-8")`。
4. **相对链接没转绝对**：`/news/1` 直接请求会 404，先 `urljoin` 拼成完整 URL。
5. **频率太高被封 IP**：`sleep` + 随机抖动，被 429/403 先停手看原因，别硬刚。
6. **BFS 不去重**：没有 `seen` 集合，循环链接让你抓到天荒地老。
7. **敏感信息进代码**：账号、密码、Cookie、API Key 一律走环境变量，别写死在脚本里。

## 练习建议

1. 用 `HTMLParser` 写一个标签计数器：输入一段 HTML，输出每种标签各出现几次。
2. 给第三节的 BFS 框架加上"最大页数"限制和"站外链接过滤"（已内置），再试一个带循环链接的模拟站点，验证不会死循环。
3. 用 `sqlite3` 存 100 条模拟商品数据（标题+价格），练习按价格区间查询和按价格排序。
4. 手写 robots.txt 解析（第十二节已有），找 5 条真实站点的 robots.txt 规则来测试你的函数。
5. 找一个返回 JSON 的公开 API（**需联网环境**），用 `json.loads` 解析并存进 SQLite，走通"请求→解析→存储"全链路。

本文为学习笔记（编纂），用自己的话重写；
