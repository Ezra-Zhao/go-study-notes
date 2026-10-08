# Python 网络编程学习笔记

**本书定位**：本笔记依据某本 Python 网络编程书的章节结构整理，从底层套接字一路讲到应用层协议的 Python 实现，适合已经熟悉 Python 语法、想系统掌握网络通信编程的人。

## 一、客户端 / 服务器模型

网络程序几乎都是"一方提需求、一方给服务"的关系。理解这个模型时抓住三件事就够了：

- **地址四元组**：一次 TCP 连接由（源 IP、源端口、目标 IP、目标端口）唯一确定。端口就是同一台机器上区分不同服务的门牌号，0~1023 是知名端口（如 80、443）。
- **角色不对称**：服务器先 bind 再 listen 等待；客户端主动 connect 发起。UDP 没有"连接"的概念，双方都是对等收发。
- **字节流 vs 数据报**：TCP 是可靠字节流，数据像水管里的水连续送到；UDP 是独立数据报，一个包一个包地发，可能丢、可能乱序，但快。

## 二、UDP：无连接的数据报

UDP 的调用流程极短：创建套接字 → 直接 `sendto` / `recvfrom`。没有握手，没有连接状态，服务器也无需 `listen`。

适用场景：DNS 查询、视频会议、游戏位置同步——丢一两个包无所谓，但延迟必须低。

调用模式（本机沙箱禁用了 UDP 数据报，下面的代码在普通机器上可运行，本机未执行验证，故此处只讲模式不附可运行示例）：服务器 `socket(AF_INET, SOCK_DGRAM)` 后直接 `bind`；`recvfrom(1024)` 返回（数据，发送方地址）二元组；回复时用该地址 `sendto` 回去。客户端无需 `connect`，直接 `sendto(数据, (服务器IP, 端口))`。全程无握手、无连接状态。

核心区别一句话：TCP 先建连接再说话，UDP 拿起就喊、喊完不确认。

## 三、TCP：可靠的字节流

TCP 编程是四个经典动作：服务器 `bind → listen → accept`，客户端 `connect`，双方 `send/recv`，最后 `close`。

关键原理：

- **三次握手**在 `connect` / `accept` 里由内核完成，应用层看不到细节，只需知道连接建立后通道就是可靠有序的。
- **recv 不保证一次收完**：TCP 是字节流没有消息边界，`recv(4096)` 返回"当前可用的"字节，可能是半条消息。所以收消息必须自己定协议：要么固定长度，要么先发 4 字节长度头再发正文，要么用分隔符。
- **粘包的本质**：发送方两次 `send` 的数据，接收方可能一次全收到。处理粘包是每个 TCP 程序员的必修课，标准做法是"长度前缀 + 循环收满"。

```python
import socket, threading, struct

def server():
    srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(("127.0.0.1", 13000))
    srv.listen(1)
    conn, _ = srv.accept()
    with conn:
        while True:
            head = recvn(conn, 4)
            if not head:
                break
            (n,) = struct.unpack("!I", head)
            body = recvn(conn, n)
            msg = body.decode()
            resp = msg.upper().encode()
            conn.sendall(struct.pack("!I", len(resp)) + resp)
    srv.close()

def recvn(conn, n):
    buf = b""
    while len(buf) < n:
        chunk = conn.recv(n - len(buf))
        if not chunk:
            return b"" if not buf else buf[:0] or b""
        buf += chunk
    return buf

t = threading.Thread(target=server, daemon=True)
t.start()

cli = socket.create_connection(("127.0.0.1", 13000))
for m in ["hello", "tcp", "bye"]:
    b = m.encode()
    cli.sendall(struct.pack("!I", len(b)) + b)
    head = cli.recv(4)
    (n,) = struct.unpack("!I", head)
    print("收到:", cli.recv(n).decode())
cli.close()
```

注意上面 `recvn` 的写法：循环收直到凑满 n 字节，对端关闭时返回空字节串，这是处理"消息边界"的最小可用模式。

## 四、套接字与 DNS

程序里写的是域名，真正连的是 IP，中间靠 DNS 翻译。Python 里用 `socket.getaddrinfo` 做解析，它一次性返回所有可用的地址族组合，写兼容 IPv4/IPv6 的代码就靠它。

实践经验：不要自己拼 DNS 报文去查（除非在学协议），直接调系统解析。需要控制超时时，用 `socket.setdefaulttimeout` 或对套接字单独 `settimeout`，因为 DNS 解析本身可能阻塞很久。

```python
import socket

# 解析一个域名，列出所有地址
for fam, stype, proto, canon, addr in socket.getaddrinfo("localhost", 80):
    print(socket.AddressFamily(fam).name, addr[0])

# 反向解析：IP -> 主机名
print(socket.gethostbyaddr("127.0.0.1"))
```

## 五、网络数据与网络错误

网络字节序是大端（big-endian），`struct` 的 `!` 前缀就是"网络字节序"，打包协议头时必须用它，否则两台机器解析出的数字对不上。

错误处理要分三类想：

1. **可重试的**：超时（`socket.timeout`）、连接被拒（`ConnectionRefusedError`）——适合退避重试。
2. **要断开的**：对端关闭（`recv` 返回 `b""`）、连接重置（`ConnectionResetError`）——清理资源、关闭连接。
3. **编程错误**：地址已占用、非法参数——修代码而不是重试。

生产级代码里，给每次连接设超时是铁律：不设超时的 `recv` 会永远卡住，一个慢客户端就能拖死整个服务。

## 六、TLS / SSL

TLS 给 TCP 套了一层加密和身份认证。客户端验证服务器证书、协商出会话密钥，之后的数据全加密。Python 标准库的 `ssl` 模块可以直接把普通套接字包装成 TLS 套接字。

记住两个概念就够用：**证书**证明"我是谁"（由 CA 签发），**握手**协商出"我们怎么加密说话"。自己写客户端时，永远用系统默认的 CA  bundle 做校验，不要图省事关掉证书验证——关掉的那一刻，中间人攻击就畅通无阻了。

## 七、服务器架构：并发模型

一个服务器要同时服务多个客户端，经典有四种模型：

| 模型 | 原理 | 适合 |
|---|---|---|
| 多进程 | 每个连接 fork 一个进程 | CPU 密集、利用多核，内存开销大 |
| 多线程 | 每个连接一个线程 | IO 密集、写法直观，受 GIL 限制 |
| select/poll | 单线程轮询就绪事件 | 连接数中等，跨平台兼容好 |
| asyncio | 单线程协程事件循环 | 高并发 IO 密集，Python 3.5+ 主流方案 |

现代 Python 网络服务的默认答案是 **asyncio**：一个线程里跑几千个连接，代码写起来像同步一样直白，性能足够好。

```python
import asyncio

async def handle(reader, writer):
    while True:
        data = await reader.read(1024)
        if not data:
            break
        writer.write(data.upper())
        await writer.drain()
    writer.close()
    await writer.wait_closed()

async def main():
    srv = await asyncio.start_server(handle, "127.0.0.1", 14000)
    async with srv:
        await srv.serve_forever()

async def client():
    reader, writer = await asyncio.open_connection("127.0.0.1", 14000)
    writer.write(b"hello asyncio")
    await writer.drain()
    print("收到:", await reader.read(1024))
    writer.close()
    await writer.wait_closed()

async def demo():
    task = asyncio.create_task(main())
    await asyncio.sleep(0.2)
    await client()
    task.cancel()

asyncio.run(demo())
```

## 八、缓存与消息队列

网络服务的性能瓶颈往往不在计算而在重复劳动。两层思想：

- **缓存**：把贵的结果存下来，下次直接给。HTTP 有整套缓存语义（ETag、Last-Modified、Cache-Control），客户端按规则决定用缓存还是重新要。
- **消息队列**：生产者和消费者解耦，削峰填谷。发邮件、跑耗时任务这种"不用立刻出结果"的事，扔进队列慢慢做，前端立刻返回。

缓存最朴素的实现就是"记住算过的结果"。下面是一个带过期时间的内存缓存装饰器，原理和 HTTP 缓存、DNS 缓存是同一件事：用空间换时间，用过期换新鲜度。

```python
import time, functools

def ttl_cache(seconds):
    store = {}
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*a):
            now = time.time()
            if a in store:
                val, ts = store[a]
                if now - ts < seconds:
                    print("命中缓存")
                    return val
            val = fn(*a)
            store[a] = (val, now)
            return val
        return wrapper
    return deco

@ttl_cache(2)
def slow_add(x, y):
    time.sleep(0.5)
    return x + y

t0 = time.time(); print(slow_add(1, 2)); print("耗时", round(time.time() - t0, 2))
t0 = time.time(); print(slow_add(1, 2)); print("耗时", round(time.time() - t0, 2))
time.sleep(2.1)
t0 = time.time(); print(slow_add(1, 2)); print("耗时", round(time.time() - t0, 2))
```

## 九、HTTP 客户端

绝大多数时候用 `urllib` 或第三方库就够了，不需要手拼 HTTP 报文。手写 HTTP 客户端的价值在于理解协议：一行请求行（`GET /path HTTP/1.1`）、若干头（Host 必填）、空行、再到可选的 body。响应同理：状态行 + 头 + 空行 + body。

```python
import urllib.request, json

# 用标准库发 GET 请求并解析 JSON
req = urllib.request.Request(
    "https://httpbin.org/get",
    headers={"User-Agent": "study-notes/1.0"},
)
with urllib.request.urlopen(req, timeout=10) as r:
    print(r.status)
    data = json.loads(r.read().decode())
    print("你的出口 IP:", data["origin"])
```

注意：`urlopen` 默认会校验 HTTPS 证书，这正是我们想要的；`timeout` 参数一定要给。

## 十、HTTP 服务器

理解 HTTP 服务器只需抓住请求处理流水线：**解析请求行和头 → 路由到处理函数 → 构造状态行+头+正文 → 发回**。Python 标准库的 `http.server` 模块已经实现了这套流水，写个小工具服务几十行就够。

```python
import threading, time
from http.server import BaseHTTPRequestHandler, HTTPServer

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        body = b'{"ok": true}'
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)
    def log_message(self, *a):
        pass

srv = HTTPServer(("127.0.0.1", 15000), H)
t = threading.Thread(target=srv.serve_forever, daemon=True)
t.start()
time.sleep(0.2)

import urllib.request
with urllib.request.urlopen("http://127.0.0.1:15000/", timeout=5) as r:
    print(r.status, r.read().decode())
srv.shutdown()
```

要点：`Content-Length` 头必须和正文长度一致，否则客户端不知道正文在哪里结束；`log_message` 重写掉可以让示例输出干净。

## 十一、万维网

万维网 = HTTP + URL + HTML 三件套。URL 的结构 `协议://主机:端口/路径?查询#片段` 每个部分都有含义，写爬虫或 API 客户端时天天打交道：路径定位资源，查询串传参，片段只给浏览器用、不会发到服务器。HTML 解析用标准库的 `html.parser` 或第三方库，不要用正则表达式去解析 HTML——正则处理不了嵌套标签，这是条血泪经验。

## 十二、电子邮件协议族

邮件系统是三个协议分工：

- **SMTP**：只负责"寄信"，客户端把信推给服务器（发件用它）。
- **POP3**：把信从服务器"搬下来"，搬完服务器上一般就删了，适合单设备。
- **IMAP**：在服务器上"管理信"，多设备同步、文件夹操作都靠它。

Python 标准库 `smtplib` / `poplib` / `imaplib` 三件套对应这三个协议。现代实践：发信用带 TLS 的 SMTP（587 端口 STARTTLS），收信用 IMAP；密码用应用专用密码，不要把主密码写进代码。

## 十三、Telnet 与 SSH

Telnet 是明文协议，密码在网线里裸奔，今天只剩调试和古董设备还在用，**永远不要用它传敏感信息**。SSH 是加密替代品：`paramiko` 之类的库可以实现远程执行命令、传文件。记住：能走 SSH 的地方就不要走 Telnet。

## 十四、FTP

FTP 是老牌文件传输协议，特点是"控制连接 + 数据连接"双通道，主动/被动模式经常是防火墙排错的坑。标准库 `ftplib` 封装得很好。现代替代方案是 SFTP（走 SSH 通道，单端口、加密），新项目优先选 SFTP。

## 十五、RPC：像调本地函数一样调远程

RPC 的核心思想是把"网络调用"伪装成"本地函数调用"：客户端调 `add(1, 2)`，框架负责序列化参数、发请求、等结果、反序列化。Python 标准库自带极简实现 `xmlrpc`，理解原理足够：

```python
import threading, time
from xmlrpc.server import SimpleXMLRPCServer
from xmlrpc.client import ServerProxy

def serve():
    srv = SimpleXMLRPCServer(("127.0.0.1", 16000), logRequests=False)
    srv.register_function(lambda a, b: a + b, "add")
    srv.serve_forever()

t = threading.Thread(target=serve, daemon=True)
t.start()
time.sleep(0.2)

proxy = ServerProxy("http://127.0.0.1:16000/")
print("1 + 2 =", proxy.add(1, 2))
```

生产环境更多用 gRPC、REST 或消息队列，但"序列化 + 传输 + 反序列化"这个三段式是所有 RPC 的共同骨架。

## 十六、常见坑（血泪清单）

1. **recv 不设超时**：一个不说话的客户端能卡住你的线程 forever，所有套接字都要 `settimeout`。
2. **以为 send 一次就发完**：`send` 返回实际发送字节数，循环发或直接用 `sendall`。
3. **忘记 SO_REUSEADDR**：调试时反复重启服务器会报"地址已占用"，开发阶段加上它。
4. **TCP 当消息队列用**：没有消息边界概念，必须自己定"长度前缀"协议。
5. **关掉 TLS 证书校验**：图省事一时爽，中间人攻击火葬场。
6. **阻塞的 DNS 解析**：`getaddrinfo` 可能卡几十秒，关键路径上要设超时或异步解析。
7. **文件描述符泄漏**：连接不用了一定要 `close`，长跑服务泄漏 fd 会把自己拖死。

## 十七、练习建议

1. 把本文的 TCP 回声服务器改成"聊天室"：多个客户端连上来，一人说话所有人收到（提示：服务器维护一个连接列表，`sendall` 广播）。
2. 给 UDP 示例加上"序列号 + 超时重传"，体会 TCP 可靠性机制是怎么一层层搭出来的。
3. 用 `http.server` 写一个文件浏览器：访问根路径列出目录，点击文件名下载文件。
4. 把 asyncio 回声服务器改成支持 1000 个并发客户端，用另一个脚本压测，观察内存和 CPU。
5. 用 `xmlrpc` 写一个远程计算器服务，再试试用 JSON over HTTP 自己实现一遍，对比两种方式的优劣。

本文为学习笔记（编纂），用自己的话重写；
