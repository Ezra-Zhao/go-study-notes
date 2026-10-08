# 项目驱动的 Python 爬虫学习笔记（编纂）

**本书定位（一句话）：**本笔记依据一本"项目驱动"思路的爬虫书的章节结构整理，重写为个人学习笔记——它不按工具讲，而按"基础→中级→深入"三层递进，每一层都落到能跑的项目上。

## 一、基础篇：一条能跑通的最小链路

爬虫的最小闭环只有四步：**取种子 URL → 下载页面 → 提取链接 → 入库去重**。先把这四步跑通，再谈优化。下面是一个从种子页出发、带去重和入库的迷你爬虫（种子页用公开测试服务，需联网；已在 Python 3.12 本机验证）：

```python
import sqlite3, time
import requests
from lxml import html
from urllib.parse import urljoin

HEADERS = {"User-Agent": "Mozilla/5.0 (compatible; StudyBot/1.0; +https://example.com)"}

def crawl_seed(seed, db="mini.db", max_pages=6):
    conn = sqlite3.connect(db)
    conn.execute("CREATE TABLE IF NOT EXISTS seen(url TEXT PRIMARY KEY)")
    conn.execute("CREATE TABLE IF NOT EXISTS results(url TEXT, title TEXT)")
    visited, queue, got = set(), [seed], 0
    while queue and got < max_pages:
        url = queue.pop(0)
        if url in visited:
            continue
        visited.add(url)
        conn.execute("INSERT OR IGNORE INTO seen(url) VALUES(?)", (url,))
        try:
            r = requests.get(url, headers=HEADERS, timeout=15)
            r.raise_for_status()
        except requests.RequestException as e:
            print("跳过:", url, type(e).__name__)
            continue
        tree = html.fromstring(r.content)
        title = (tree.xpath("//title/text()") or [""])[0].strip()
        conn.execute("INSERT INTO results(url, title) VALUES(?, ?)", (url, title))
        got += 1
        for href in tree.xpath("//a/@href"):
            nxt = urljoin(url, href)   # 相对链接转绝对
            if nxt not in visited:
                queue.append(nxt)
        time.sleep(1)                  # 礼貌延迟
    conn.commit()
    n = conn.execute("SELECT COUNT(*) FROM results").fetchone()[0]
    conn.close()
    return n
```

原理讲透：`visited` 集合是内存去重，`seen` 表是磁盘去重——程序重启后，已抓过的 URL 不会重抓。`urljoin` 处理相对路径，这是新手最容易漏的一步。`time.sleep(1)` 是底线礼貌，生产环境应加随机抖动。

## 二、中级篇之一：存储选型

三种常见存储的取舍：

- **SQLite**：零配置、单文件，适合单机项目和原型验证，上面的代码就是例子。
- **MySQL**：适合结构化数据、多人协作、需要复杂查询的场景；注意建表时选好字符集（utf8mb4），否则中文和 emoji 会乱码。
- **MongoDB**：文档型，适合字段不固定的页面数据（如不同详情页字段差异大），存 JSON 原生友好。

选型口诀：字段固定用关系型，字段多变用文档型，单机原型用 SQLite。

## 三、中级篇之二：动态网站的两种思路

页面数据分两种：服务端渲染好的 HTML，和页面加载后由 JavaScript 再请求的 JSON 接口。

- **思路一（解析 HTML）**：适合传统服务端渲染页，用 XPath/CSS 直接抠。
- **思路二（直接调接口）**：打开浏览器开发者工具的 Network 面板，找到返回 JSON 的数据接口，直接请求它——省去解析 HTML，数据还干净。

第二种思路往往效率高一个数量级，但要注意：接口可能有签名或时效参数，且同样受服务条款约束。

## 四、中级篇之三：登录态与会话保持

需要登录才能看的数据，技术上靠 Cookie 维持会话。`requests.Session` 会自动管理 Cookie，登录一次、后续请求自动带上：

```python
import requests

s = requests.Session()
s.headers.update({"User-Agent": "Mozilla/5.0"})
s.get("https://httpbin.org/cookies/set/sessionid/abc123", timeout=20)  # 模拟登录种 Cookie，需联网
jar = s.get("https://httpbin.org/cookies", timeout=20).json()["cookies"]
print(jar)  # {'sessionid': 'abc123'} —— Session 自动携带了 Cookie
```

（已在 Python 3.12 本机验证。）原理：Session 对象在多次请求间共享 CookieJar，相当于浏览器"记住登录状态"的机制。**红线**：只用于自己有合法权限的账号和数据，不碰他人账号、不绕过登录限制。

## 五、中级篇之四：多端爬取思路

同一份数据可能存在于多个终端，思路可以发散：

- **PC 网页端**：传统 HTML，解析成熟。
- **手机网页端**：页面更简洁，有时反爬更松，但结构可能不同，要单独适配解析规则。
- **客户端（含移动 App）**：数据走 API 接口，抓包分析后直接调接口效率最高，但接口签名和加密是常见门槛。

核心思想：不要死磕一种入口，换个终端可能就是另一片天地。但无论哪个端，合规边界一致。

## 六、深入篇之一：大规模去重

URL 量到百万级时，`set()` 存内存会爆炸。**布隆过滤器**用固定内存换"可能误判、绝不错杀"：

```python
import hashlib

class SimpleBloom:
    """极简布隆过滤器（已在 Python 3.12 本机验证）"""
    def __init__(self, size=2 ** 20):
        self.size = size
        self.bits = bytearray(size // 8)

    def _hashes(self, s):
        b = s.encode("utf-8")
        h1 = int(hashlib.md5(b).hexdigest(), 16)
        h2 = int(hashlib.sha1(b).hexdigest(), 16)
        return h1 % self.size, h2 % self.size

    def add(self, s):
        for h in self._hashes(s):
            self.bits[h // 8] |= 1 << (h % 8)

    def __contains__(self, s):
        return all(self.bits[h // 8] & (1 << (h % 8)) for h in self._hashes(s))
```

原理讲透：一个元素被 k 个哈希函数映射到 k 个 bit 位；查询时 k 位全为 1 才判"见过"。误判率可控（调大位数组、调优 k），但"判没见过"一定准确——这正是去重需要的单向保证。生产环境可用 Redis 的位图或现成布隆模块做分布式去重。

## 七、深入篇之二：分布式爬虫

单机不够用时，架构拆成三件套：**任务队列 → 抓取节点 → 去重中心**。队列可用 Redis 的 List，多个抓取进程从同一队列取 URL，去重中心用 Redis Set 或布隆过滤器。多进程通信的最小模型如下（已在 Python 3.12 本机验证）：

```python
from multiprocessing import Process, Queue

def worker(q, out):
    while True:
        item = q.get()
        if item is None:      # 毒丸：通知退出
            break
        out.put(item.upper()) # 模拟"抓取并处理"

if __name__ == "__main__":
    q, out = Queue(), Queue()
    p = Process(target=worker, args=(q, out))
    p.start()
    q.put("task-a"); q.put(None); p.join()
    print(out.get())  # TASK-A
```

原理：Queue 是进程安全的，`None` 作为"毒丸"优雅停工。真实分布式把 Queue 换成 Redis，把单机函数换成 Scrapy 爬虫，骨架不变。框架层面，Scrapy-Redis 是经典方案，pyspider 则自带分布式调度。

## 八、URL 规范化：去重之前先"洗"URL

去重最大的隐形 bug 是"同一个页面，URL 长得不一样"：大小写、尾部斜杠、参数顺序、锚点（# 后面）都会让字符串比较失效。入库去重之前，先做规范化：

- 统一转小写（域名部分）；
- 去掉 `#` 锚点（锚点不影响服务端内容）；
- 排序查询参数（`?b=2&a=1` 和 `?a=1&b=2` 是同一个页面）；
- 去掉追踪参数（如 `utm_source` 这类不影响内容的营销参数）；
- 统一尾部斜杠。

规范化后的 URL 再送进布隆过滤器或 `seen` 表，去重率会明显提升。这一步的投入产出比极高，但新手经常跳过。

## 九、Scrapy 数据流：五件套一次讲清

Scrapy 的核心是五个组件构成的一条流水线，理解数据流向就理解了框架：

1. **Engine（引擎）**：总调度，负责在各组件之间搬运数据；
2. **Scheduler（调度器）**：URL 队列，决定下一个抓谁，自带去重；
3. **Downloader（下载器）**：真正发请求拿响应的模块；
4. **Spider（爬虫）**：你写解析规则的地方，输入响应、输出数据或新请求；
5. **Pipeline（管道）**：数据清洗、去重、入库的最后一公里。

数据流转：Spider 产出 Request → Engine → Scheduler 排队 → Engine → Downloader 下载 → Response 回 Engine → Spider 解析 → Item 进 Pipeline 入库。**中间件**（Middleware）就是插在这条流水线上的"钩子"，比如在请求发出去之前统一加代理、在响应回来之后统一处理异常。理解了这条链，Scrapy 的文档就不再是天书。

## 十、增量抓取：只抓变化的部分

全量重抓是资源浪费，成熟爬虫都做增量。三种常用策略：

- **时间戳比对**：列表页带发布时间，只抓"上次抓取时间之后"的新条目。需要持久化一个"水位线"（watermark）。
- **内容哈希**：对详情页正文算哈希，哈希不变说明没更新，跳过解析和入库，省 CPU 和数据库写入。
- **站点信号**：很多站点提供 sitemap 或 RSS，直接消费这些"官方变更通知"，比自己轮询高效得多。

增量抓取的前提是"状态可持久化"：水位线、哈希表都要落盘（SQLite 就够），程序重启不丢失。把"全量"和"增量"做成两种模式：第一次全量建库，之后每天增量更新——这是生产爬虫的标准形态。

## 十一、合规提醒

- 先读 `robots.txt`，遵守 Disallow；遵守网站服务条款；
- 只采集公开数据，不采集登录后数据、个人隐私与付费内容；
- 控制频率、分散时段，不给目标服务器造成负担；
- 登录态仅用于自己有合法权限的场景；不研究针对具体网站的绕过去重/反爬细节。

## 十二、常见坑

1. **相对链接没转绝对**：`urljoin` 是标配，漏了它详情页全是 404。
2. **去重只做内存**：程序一重启全重抓，磁盘去重（seen 表）必须有。
3. **布隆误判当 bug**：误判是设计特性，不是 bug；关键业务用精确去重兜底。
4. **Session 混用**：多线程共享一个 Session 可能互相污染 Cookie，按线程隔离。
5. **分布式先行**：单机一天能抓完的量，不要上分布式——运维成本远超收益。
6. **URL 没规范化就去重**：同一个页面多种写法，去重形同虚设。
7. **广度优先无深度限制**：种子站外链一扩散，队列指数爆炸，先设深度上限。
8. **Pipeline 里做重型清洗**：管道是入库前的最后一公里，耗时操作放前面异步做，别堵住入库。

## 十三、练习建议

1. 跑通 `crawl_seed`，把 `max_pages` 调大，观察去重表 `seen` 的增长。
2. 给迷你爬虫加"深度限制"：记录每个 URL 的深度，超过 2 层不再扩展。
3. 用 `SimpleBloom` 替换 `visited` 集合，对比内存占用（可用 `tracemalloc` 观察）。
4. 把单进程版改成"一生产者 + N 消费者"的多进程版，用毒丸优雅退出。
5. 选一个公开 API（如返回 JSON 的测试接口），写"直接调接口"版本，对比解析 HTML 的代码量。
6. 给 `crawl_seed` 加 URL 规范化函数：去掉锚点、排序参数，统计去重率的变化。
7. 照着第九节的数据流，在纸上画出 Scrapy 一次"请求→入库"的完整路径，标出每个组件的输入输出。

本文为学习笔记（编纂），用自己的话重写；
