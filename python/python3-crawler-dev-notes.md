# Python 3 网络爬虫全链路学习笔记（编纂）

**本书定位（一句话）：**本笔记依据一本按"全链路"组织的 Python 3 爬虫书的章节结构整理，重写为个人学习笔记——从发请求、解析、存储，到 Ajax、动态渲染、验证码、代理、登录、App，再到框架与分布式部署，一条线走完爬虫开发的全部环节。

## 一、基本库：把 requests 用到"生产级"

`requests` 是爬虫的起点。GET 带参数、POST 送 JSON、自定义请求头，是每天都要写的三件套（示例目标为公开测试服务，需联网；已在 Python 3.12 本机验证）：

```python
import requests

UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0"
H = {"User-Agent": UA}

def fetch_post_json(url, payload, timeout=10, retries=2):
    """POST 发送 JSON，带重试（逻辑已在 Python 3.12 本机验证）"""
    last = None
    for _ in range(retries):
        try:
            r = requests.post(url, json=payload, headers=H, timeout=timeout)
            r.raise_for_status()
            return r.json()
        except requests.RequestException as e:
            last = e
    raise last

r = requests.get("https://httpbin.org/get",
                 params={"q": "中文", "page": 2}, headers=H, timeout=20)
print(r.json()["args"])  # {'q': '中文', 'page': '2'} —— params 自动编码
```

原理讲透：`params` 字典会被自动 URL 编码，中文也不用你操心；`json=` 参数会自动序列化字典并设置 Content-Type；**每次请求都必须带 timeout**，这是生产代码和玩具代码的分水岭。

## 二、解析库：XPath 定位 + 文本清洗

解析分两步：先用 XPath 把节点圈出来，再清洗文本。常见清洗动作：`strip()` 去空白、`re.sub(r"\s+", " ", t)` 压空白、对缺失节点用 `or [""]` 兜底。记住一条：**解析代码永远假设页面会变**，关键字段取不到时要抛错或记日志，不要静默产出空数据。

## 三、数据存储：按"用数据的姿势"选

- **文本/CSV**：适合一次性导出、给人看；注意 CSV 写中文要带 BOM（`encoding="utf-8-sig"`），否则 Excel 打开乱码。
- **JSON**：适合嵌套结构、给程序消费；`ensure_ascii=False` 保留中文可读性。
- **SQLite**：适合单机长期项目，SQL 查询方便（见前两篇笔记的示例）。
- **MySQL / MongoDB**：适合多机协作或海量数据，选型看字段是否固定。

## 四、Ajax 数据爬取：直接拿 JSON

现代页面很多数据是 JS 加载后调接口拿的。与其等浏览器渲染再解析 DOM，不如直接请求那个 JSON 接口：

```python
import requests

r = requests.get("https://httpbin.org/json",
                 headers={"User-Agent": "Mozilla/5.0"}, timeout=20)
data = r.json()
print(data["slideshow"]["title"])  # Sample Slide Show
```

（已在 Python 3.12 本机验证。）方法论：浏览器开发者工具 → Network → 筛 XHR/Fetch → 找到返回 JSON 的请求 → 照抄它的 URL 和参数。注意接口可能有时效性参数或签名，且同样受服务条款约束。

## 五、动态渲染页面：浏览器自动化

遇到必须执行 JavaScript 才能出数据的页面，思路是"让真正的浏览器去跑"：

- **Selenium**：驱动真实浏览器（Chrome/Firefox），所见即所得；代价是慢、吃内存。
- **无头模式**：`--headless` 不开界面，省资源，适合服务器。
- **等待策略**：用显式等待（等某个元素出现）代替 `time.sleep` 死等，又稳又快。

这类工具需要额外安装（`pip install selenium` + 浏览器驱动），且版本要与浏览器匹配，这是新手卡住最多的地方。原则：能调接口就别上浏览器——浏览器自动化是最后手段。

## 六、验证码：理解原理，不碰红线

验证码的本质是"区分人与程序"。常见的技术原理：

- **图像预处理**：灰度化 → 二值化 → 去噪点，把图片变成干净的黑白字；
- **字符识别**：传统方法是模板匹配，现代方法是 OCR/深度学习模型；
- **合规做法**：把验证码截图抛给人工处理，或直接放弃该数据源。

**红线**：本笔记不提供针对任何具体网站验证码的自动识别与绕过方案。学习原理是为了理解对抗逻辑，不是为了绕过。

## 七、代理的使用：故障切换比"池子"更重要

代理的核心价值不是"隐藏身份"，而是**分散请求压力、提高可用性**。比维护一个大代理池更重要的，是写好"失败就换下一个"的切换逻辑：

```python
import requests

def fetch_via_proxies(url, proxies, timeout=20):
    """依次尝试代理列表，None 表示直连；全部失败才抛异常
    （已在 Python 3.12 本机验证：坏代理自动降级为直连）"""
    last_err = None
    for p in proxies:
        try:
            kw = {"timeout": timeout,
                  "headers": {"User-Agent": "Mozilla/5.0"}}
            if p:
                kw["proxies"] = {"http": p, "https": p}
            r = requests.get(url, **kw)
            r.raise_for_status()
            return r.text, p
        except requests.RequestException as e:
            last_err = e  # 记下错误，换下一个
    raise last_err

text, used = fetch_via_proxies(
    "https://httpbin.org/ip",
    ["http://127.0.0.1:9", None])  # 第一个是坏代理，会自动降级直连
print("最终使用的代理:", used)
```

原理讲透：代理要做健康检查（定期探测、失败摘除、成功恢复），否则"池子"里全是死代理，切换逻辑再漂亮也白搭。免费公开代理可用率极低，生产环境应使用正规代理服务。

## 八、模拟登录：Session 保持会话

登录的本质是服务器给你发一个身份凭证（Cookie/Token），之后每次请求带上它。`requests.Session` 自动管理 Cookie：

```python
import requests

s = requests.Session()
s.headers.update({"User-Agent": "Mozilla/5.0"})
# 下面两行模拟"登录种 Cookie → 后续请求自动携带"（需联网）
s.get("https://httpbin.org/cookies/set/sessionid/abc123", timeout=20)
jar = s.get("https://httpbin.org/cookies", timeout=20).json()["cookies"]
print(jar)  # {'sessionid': 'abc123'}
```

（已在 Python 3.12 本机验证。）**红线**：仅用于自己有合法权限的账号；登录态不要分享、不要用于绕过付费或权限限制。

## 九、App 爬取：思路是"看流量"

App 的数据一般走 HTTPS 接口。通用分析思路：

1. 手机与电脑连同一 Wi-Fi，电脑开抓包工具做代理；
2. 手机上操作 App，观察抓包工具里的请求列表；
3. 找到返回数据的接口，分析它的 URL、参数、签名规律；
4. 用 requests 复现请求。

注意：App 接口常有签名和证书 pinning，对抗成本高；且用户数据、付费内容一律不碰。能走官方开放 API 的，优先走官方渠道。

## 十、框架：Scrapy 与 pyspider

当项目大到"手写调度、去重、并发"开始痛苦时，上框架：

- **Scrapy**：Engine → Scheduler → Downloader → Spider → Pipeline 五件套；中间件机制允许你在请求/响应流转中插逻辑；适合长期运行的大型采集。
- **pyspider**：自带 Web UI，可视化调试和任务监控开箱即用；适合快速验证和中小规模。

框架解决的是"工程问题"（调度、去重、并发、监控），不解决"数据源问题"——解析规则和合规边界还是你自己负责。

## 十一、分布式爬虫的部署

规模再往上走，架构变成：**调度中心（Redis 队列）→ 多台抓取机 → 去重中心（Redis/布隆）→ 入库**。部署要点：

- 任务队列与去重状态放 Redis，多机共享；
- 抓取节点无状态，可随时加减机器；
- 用 scrapyd 或容器化（Docker）统一管理爬虫进程；
- 监控三件事：队列长度（别堆积）、成功率（别被封）、入库量（别丢数）；
- 先做单机压测摸清目标站点的承受上限，再谈加机器——分布式不是用来"更快地被封"的。

## 十二、环境与依赖：虚拟环境是标配

爬虫项目依赖 requests、lxml 等第三方库，不同项目版本要求可能冲突。正确姿势：

```bash
python3 -m venv .venv        # 建虚拟环境
source .venv/bin/activate    # 激活
pip install requests lxml    # 装依赖
pip freeze > requirements.txt  # 锁版本，别人一条命令复现环境
```

原理讲透：虚拟环境是一套独立的 `site-packages`，项目之间互不污染；`requirements.txt` 是"环境快照"，配合它任何人都能重建出和你一模一样的运行环境。这是"代码能跑"和"代码在别人机器上也能跑"的区别。部署到服务器时，用同样的文件重建环境，避免"本地能跑、线上爆炸"。

## 十三、合规提醒

- 先读 `robots.txt`，遵守 Disallow；遵守网站服务条款与 API 使用协议；
- 只采集公开数据；不碰登录后数据、个人隐私、付费内容；
- 控制频率、分散时段；大规模采集走正规数据合作渠道；
- 不编写针对具体网站反爬、验证码、登录限制的绕过代码。

## 十四、常见坑

1. **不带 timeout**：一个假死连接拖住整个线程池。
2. **Session 跨线程共享**：Cookie 互相污染，按线程隔离。
3. **硬编码 XPath**：页面一改全挂，关键字段加断言。
4. **代理不做健康检查**：池子里全是死代理，越切越慢。
5. **动态页无脑上 Selenium**：先找 JSON 接口，能省 90% 的资源。
6. **分布式先行**：单机能搞定的量别上分布式，运维成本是隐形杀手。
7. **不锁依赖版本**：`pip install requests` 今天和明年装到的可能不是一个版本，`requirements.txt` 必须提交。
8. **POST 不设 Content-Type**：用 `data=` 发表单、用 `json=` 发 JSON，混用对方服务器可能直接 400。
9. **抓包复现 App 接口时漏签名**：接口报"签名错误"先检查参数顺序和时间戳，别急着加逻辑。

## 十五、练习建议

1. 用 `fetch_post_json` 向测试服务 POST 一组数据，打印回显验证字段无损。
2. 用 `fetch_via_proxies` 实现"代理健康检查"：定时探测，失败的移出列表。
3. 找一个返回 JSON 的公开接口，写"直接调接口"版和"解析 HTML"版，对比代码量。
4. 给 Session 版登录流程加"凭证过期自动重登"：401 时重新走登录再重试一次。
5. 画一张你自己的分布式架构图：标出队列、抓取节点、去重、入库、监控五个位置。
6. 为本篇所有示例建一个虚拟环境并生成 `requirements.txt`，换个目录重建验证可复现。
7. 给你的爬虫加三项监控指标并打印：队列剩余数、本小时成功率、入库总量。

本文为学习笔记（编纂），用自己的话重写；
