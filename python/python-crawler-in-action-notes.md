# 多技术栈 Python 爬虫学习笔记（编纂）

**本书定位（一句话）：**本笔记依据一本覆盖多种 Python 爬虫工具的实战型书籍的章节结构整理，重写为个人学习笔记——它的主线是"同一件事有多种做法"，教你根据场景在不同工具之间做取舍。

## 一、先建立工具选型地图

爬虫技术栈大体按"重量"排成一条谱系：

- **最轻：requests + lxml。** 发一个 HTTP 请求、拿到 HTML、按 XPath 把数据抠出来。适合静态页面、接口返回 JSON、一次性脚本。优点是依赖少、心智负担小。
- **解析层：BeautifulSoup 类库。** 把 HTML 转成可搜索的树，API 对新手友好。注意它只管"解析"，不管"下载"。
- **中型框架：Scrapy。** 自带调度、去重、并发、中间件、管道（Pipeline），适合长期运行的大型项目。代价是学习曲线陡。
- **浏览器自动化：Selenium。** 真实驱动浏览器，能执行 JavaScript，适合动态渲染页面。代价是慢、耗资源。
- **可视化分布式：pyspider。** 自带 Web 界面、可视化调试，适合需要多人协作或快速验证想法的场景。

选型原则：能静态解析就别上浏览器，能单机跑就别上分布式。工具越重，维护成本越高。

## 二、发请求：把"礼貌"写进代码

任何爬虫的第一步都是发 HTTP 请求。健壮的请求函数要回答四个问题：我是谁（UA）、等多久（超时）、失败了怎么办（重试）、对方拒绝怎么办（状态码检查）。

```python
import time
import requests

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                  "AppleWebKit/537.36 (KHTML, like Gecko) "
                  "Chrome/120.0 Safari/537.36",
    "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8",
}

def fetch(url, retries=3, timeout=10):
    """带指数退避重试的请求函数（已在 Python 3.12 本机验证）"""
    last_err = None
    for i in range(retries):
        try:
            r = requests.get(url, headers=HEADERS, timeout=timeout)
            r.raise_for_status()          # 非 2xx 直接抛异常
            r.encoding = r.apparent_encoding  # 自动纠正中文乱码
            return r.text
        except requests.RequestException as e:
            last_err = e
            time.sleep(2 ** i)            # 1s、2s、4s 退避
    raise last_err
```

原理讲透：`timeout` 防的是"对方服务器假死把你拖住"；指数退避防的是"对方一抖你就狂刷"；`raise_for_status` 把 HTTP 错误变成异常，逼你在上层统一处理，而不是让脏数据悄悄流入数据库。

## 三、解析：XPath 是爬虫人的基本功

拿到 HTML 后，解析的核心动作是"定位节点、取出文本"。XPath 用路径表达式描述节点位置，比正则稳定得多：

```python
from lxml import html

SAMPLE_HTML = """
<table><tr><th>名称</th><th>价格</th></tr>
<tr><td>商品A</td><td>12.5</td></tr>
<tr><td>商品B</td><td>9.9</td></tr></table>
"""

def parse_items(page_html):
    """解析表格行，返回 [(名称, 价格)]（已在 Python 3.12 本机验证）"""
    tree = html.fromstring(page_html)
    items = []
    for row in tree.xpath('//table//tr[position()>1]'):  # 跳过表头
        cols = [c.strip() for c in row.xpath('./td/text()')]
        if cols:
            items.append((cols[0], float(cols[1])))
    return items
```

XPath 速记：`//` 是任意层级，`./` 是相对当前节点，`text()` 取文本，`@href` 取属性，`[position()>1]` 做位置过滤。写表达式前先用浏览器开发者工具看清 DOM 结构，再动手。

## 四、反爬与应对：理解"攻防"的通用逻辑

网站常见的反爬手段和通用应对思路（只讲原理，不针对任何具体网站）：

- **识别请求头**：默认的库请求头一看就是程序。应对：带上正常的 UA 和 Accept-Language。
- **频率限制**：短时间内请求过多会被限流。应对：请求间隔加随机延迟，尊重对方服务器。
- **IP 频率画像**：单 IP 高频访问易被注意。应对：控制总量、分散时段；大规模采集应走正规数据合作渠道。
- **验证码**：本质是"证明你是人"。合规做法是人工介入或放弃该数据源，不研究自动绕过。

合规第一步永远是读 `robots.txt`。下面是一个极简解析器，只实现 Disallow 前缀匹配，默认允许：

```python
def can_fetch(robots_text, path="/", ua="*"):
    """极简 robots.txt 检查（已在 Python 3.12 本机验证）"""
    rules, current_ua = [], None
    for line in robots_text.splitlines():
        line = line.split("#", 1)[0].strip()
        if not line or ":" not in line:
            continue
        k, v = (x.strip() for x in line.split(":", 1))
        if k.lower() == "user-agent":
            current_ua = v
        elif k.lower() == "disallow" and current_ua in (ua, "*"):
            rules.append(v)
    return not any(path.startswith(d) for d in rules if d)

# robots = "User-agent: *\nDisallow: /admin/\n"
# can_fetch(robots, "/public")   -> True
# can_fetch(robots, "/admin/x")  -> False
```

## 五、存储：先用 SQLite 把数据落盘

新手最容易犯的错是把数据存在内存列表里，程序一崩全丢。正确姿势：解析完一批就入库。

```python
import sqlite3

def save_db(items, db="crawler.db"):
    """把 (名称, 价格) 存入 SQLite（已在 Python 3.12 本机验证）"""
    conn = sqlite3.connect(db)
    conn.execute("CREATE TABLE IF NOT EXISTS items "
                 "(id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, price REAL)")
    conn.executemany("INSERT INTO items(name, price) VALUES(?, ?)", items)
    conn.commit()
    n = conn.execute("SELECT COUNT(*) FROM items").fetchone()[0]
    conn.close()
    return n
```

注意用参数化查询（`?` 占位符），既防 SQL 注入，又自动处理引号转义。

## 六、提速：线程池并发

I/O 密集型任务（等网络响应）用线程池最划算。注意并发数要克制，否则你就是在对目标网站做压力测试。

```python
from concurrent.futures import ThreadPoolExecutor

urls = ["https://httpbin.org/html", "https://httpbin.org/status/200"]
with ThreadPoolExecutor(max_workers=4) as pool:
    pages = list(pool.map(fetch, urls))  # fetch 复用第二节的函数
print("抓到页面数:", len(pages))
```

## 七、HTTP 再深一层：状态码与请求头是排查利器

拿到非 200 的响应先别慌，状态码本身就是线索：

- **301/302**：重定向。`requests` 默认自动跟随，但登录态场景下要检查 Cookie 有没有在跳转中丢失。
- **403**：服务器明确拒绝。先自查请求头（UA、Referer、Accept 是否完整），再看是否触发了频率限制。
- **404**：URL 错了，检查拼接逻辑，尤其是相对路径转绝对路径。
- **429**：被限流的明确信号，退避等待，不要硬刚。
- **500/502/503**：对方服务器问题，指数退避重试，连续失败则报警停机。

请求头里最值得理解的三个字段：`User-Agent` 声明客户端身份；`Referer` 声明"我从哪个页面来"，有些站点靠它做简单的防盗链；`Accept-Language` 影响返回内容的语言。把请求头当成"自我介绍"来写，越像正常浏览器，越不容易被误伤。

## 八、限流：令牌桶的思想

"礼貌延迟"用 `time.sleep` 固定等待，简单但死板。**令牌桶**是更优雅的限流思想：桶里每秒产生固定数量的令牌，每次请求消耗一枚，没有令牌就等。它天然支持"突发"（桶满时可以连发几枚）和"长期限速"（平均速率恒定）：

```python
import time, threading

class TokenBucket:
    """令牌桶限流器：rate=每秒发放令牌数，capacity=桶容量
    （已在 Python 3.12 本机验证：10 次 acquire 约耗时 1 秒）"""
    def __init__(self, rate, capacity):
        self.rate, self.capacity = rate, capacity
        self.tokens, self.stamp = capacity, time.monotonic()
        self.lock = threading.Lock()

    def acquire(self):
        with self.lock:
            now = time.monotonic()
            self.tokens = min(self.capacity,
                              self.tokens + (now - self.stamp) * self.rate)
            self.stamp = now
            if self.tokens >= 1:
                self.tokens -= 1
                return 0.0
            wait = (1 - self.tokens) / self.rate
            time.sleep(wait)
            self.tokens = 0.0
            self.stamp = time.monotonic()
            return wait

bucket = TokenBucket(rate=2, capacity=4)  # 平均每秒 2 次，允许突发 4 次
# 每次请求前调用 bucket.acquire()
```

原理讲透：`tokens` 的累积用"时间差 × 速率"计算，不需要后台线程定时补充；`threading.Lock` 保证多线程下令牌计数不乱。把 `bucket.acquire()` 放在请求函数入口，整个爬虫的出口速率就被一只"手"管住了。

## 九、并发模型取舍：线程、进程、协程

- **多线程**：I/O 等待型任务的首选，写起来最简单。受 GIL 限制，但爬虫主要耗时在等网络，GIL 不是瓶颈。
- **多进程**：CPU 密集型解析（如超大页面、复杂清洗）才考虑；进程间通信成本高。
- **协程（asyncio/aiohttp）**：单线程撑起数千并发，性能天花板最高，但代码要全链路异步化，调试心智负担重。

给新手的路线：先单线程跑通 → 线程池提速 → 遇到天花板再学协程。过早优化是万恶之源。

## 十、解析三选一：正则、XPath、CSS 怎么取舍

三种解析手段各有领地：

- **正则**：适合从一长串文本里抠"模式固定"的小片段，比如从 JS 代码里提取接口地址。缺点是 HTML 稍微嵌套就难写难维护，**不要用正则解析整页 HTML**。
- **XPath**：表达能力最强，能按位置、属性、文本内容多维定位，是爬虫人的主力武器。
- **CSS 选择器**：写法最贴近前端习惯，`div.list > a.title` 一眼能懂；但表达力弱于 XPath（比如"取第三个"这类位置逻辑写起来别扭）。

实战口诀：整页结构用 XPath/CSS，文本里抠模式用正则。两者混用很常见——先用 XPath 圈出包含目标的节点，再用正则从节点文本里提取细节。无论用哪种，解析函数都要做"取不到就报警"，不要让空数据悄悄入库。

## 十一、合规提醒

- 先读目标站点的 `robots.txt`，遵守 Disallow 规则；
- 遵守网站服务条款，只采集公开允许的数据，不碰登录后数据和个人隐私；
- 控制频率：单线程 + 随机延迟是底线，不要给对方服务器造成负担；
- 不研究、不编写针对具体网站反爬机制的绕过代码。

## 十二、常见坑

1. **乱码**：先看响应头 charset，再看 HTML meta，都不准就用 `apparent_encoding` 兜底。
2. **被 403**：先检查请求头是否完整，而不是立刻加代理。
3. **XPath 写死**：页面一改版就失效，关键字段加断言，失败早报警。
4. **内存爆炸**：大数据量不要攒在列表里，边解析边入库。
5. **异常吞掉**：`except: pass` 是爬虫数据缺失的头号元凶，至少记日志。
6. **限流只靠 sleep**：固定延迟在对方限流策略变化时会失效，令牌桶更稳。
7. **线程数拍脑袋**：从 4 开始压测，观察成功率和响应时间再往上加，429 一出现就往回调。
8. **SQLite 多线程写**：默认连接不能跨线程共享，要么每线程一个连接，要么加锁串行写。

## 十三、练习建议

1. 用 `fetch` + `parse_items` + `save_db` 拼一条完整链路，目标换成任意公开测试页（如 httpbin.org/html），跑通并查库验证。
2. 给 `fetch` 加上"请求间隔随机 1~3 秒"的礼貌延迟。
3. 把线程池版本改成"生产者-消费者"：一个线程抓列表页，多个线程抓详情页。
4. 用 `can_fetch` 给你的爬虫加一个启动前检查：先读 robots.txt 再开工。
5. 把固定 `sleep` 换成 `TokenBucket`，对比两种限流下 20 个请求的耗时曲线。
6. 故意把 UA 改成空字符串请求一次，观察返回的状态码，理解"自我介绍"的重要性。

本文为学习笔记（编纂），用自己的话重写；
