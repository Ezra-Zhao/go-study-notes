# Python 核心编程学习笔记

**本书定位**：本笔记依据某本 Python 核心编程书的章节结构整理，分"语言核心"与"高级主题"两大部分，适合想把 Python 从"会用"练到"懂原理"的人。

## 第一部分：语言核心

### 一、一切皆对象

Python 里没有"原始类型"和"对象"的区分：整数、字符串、函数、类本身，全是对象。每个对象有三要素：**身份**（`id`，内存地址的抽象）、**类型**（`type`）、**值**。

两个最容易混的概念：

- `==` 比的是值是否相等，`is` 比的是不是同一个对象。
- 小整数（-5~256）和短字符串会被解释器缓存复用，所以 `a = 100; b = 100; a is b` 可能是 True，但这只是实现细节，**永远不要依赖它**，比较值一律用 `==`。

```python
a = [1, 2]
b = [1, 2]
c = a
print(a == b, a is b)   # True False：值相等但不是同一对象
print(a is c)           # True：c 和 a 指向同一块内存
c.append(3)
print(a)                # [1, 2, 3]：a 也被改了，因为是同一个对象
```

### 二、数字：精确与近似

整数精度无限（只受内存限制），但浮点数是二进制近似：`0.1 + 0.2` 并不精确等于 `0.3`。算钱、做金融必须用 `decimal.Decimal`，做科学计算用 `fractions.Fraction` 或 numpy。复数、八/十六进制字面量、二进制位运算都是开箱即用的。

```python
from decimal import Decimal
print(0.1 + 0.2 == 0.3)              # False：浮点近似
print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))  # True
print(bin(10), hex(255), 0b1010 + 5)  # 0b1010 0xff 15
```

### 三、序列：字符串、列表、元组

序列的通用操作是**索引、切片、拼接、重复、成员判断**。切片 `s[i:j:k]` 是左闭右开，`s[::-1]` 反转字符串是 Python 式惯用法。

核心区别：

- **列表可变、元组不可变**。元组不可变带来两个好处：可做字典键、可做集合元素；函数返回多个值时本质就是打包成元组。
- **字符串也不可变**，所有"修改"字符串的方法返回的都是新字符串。循环里 `s += x` 拼接大量文本是性能陷阱，正确做法是攒进列表最后 `"".join()`。

```python
words = ["a", "b", "c"]
print(words[1:], words[::-1])        # ['b', 'c'] ['c', 'b', 'a']
t = (1, "x")
print(t[0], len(t))                  # 1 2
# 高效拼接
parts = []
for i in range(5):
    parts.append(str(i))
print("".join(parts))                # 01234
print([x * x for x in range(6) if x % 2 == 0])  # 列表推导：[0, 4, 16]
```

**深浅拷贝**是序列章节最常见的暗坑：`b = a[:]` 或 `b = a.copy()` 只复制最外层，内层的嵌套列表依然是同一对象，改 `b[0][0]` 会连带改掉 `a`。需要完全独立时用 `copy.deepcopy`。

```python
import copy
a = [[1, 2], [3, 4]]
b = a.copy()          # 浅拷贝：外层新，内层共享
c = copy.deepcopy(a)  # 深拷贝：彻底独立
b[0][0] = 99
c[1][1] = 88
print(a)  # [[99, 2], [3, 4]] —— a 被 b 连带改了，c 没影响它
print(c)  # [[1, 2], [3, 88]]
```

### 四、映射与集合

字典是 Python 的灵魂数据结构：哈希表实现，平均 O(1) 查找。`dict.get(k, 默认值)` 避免 KeyError；`setdefault` 和 `collections.defaultdict` 处理"键不存在时初始化"的场景。

集合（set）管"去重"和"关系运算"：交并差一目了然，做数据清洗、求共同好友这类任务比列表循环快得多。

```python
from collections import defaultdict, Counter
cnt = Counter("abracadabra")
print(cnt.most_common(2))            # [('a', 5), ('b', 2)]
groups = defaultdict(list)
for k, v in [("x", 1), ("y", 2), ("x", 3)]:
    groups[k].append(v)
print(dict(groups))                  # {'x': [1, 3], 'y': [2]}
print({1, 2, 3} & {2, 3, 4})         # {2, 3} 交集
```

### 五、条件、循环与迭代器

`for` 循环的本质是不断调 `next()` 直到 `StopIteration`。任何实现了 `__iter__` / `__next__` 的对象都能被 for 循环。**生成器**（含 `yield` 的函数）是"懒"序列：一次只算一个，内存占用恒定，处理大文件、大数据流就靠它。

```python
def fib():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

g = fib()
print([next(g) for _ in range(8)])   # [0, 1, 1, 2, 3, 5, 8, 13]

# 生成器表达式：处理大文件不占内存
total = sum(int(line) for line in ["1\n", "2\n", "3\n"])
print(total)                         # 6
```

### 六、文件与输入输出

读写文件只记一个范式：`with open(...) as f`，退出代码块自动关闭，再也不用操心 `close()`。文本模式注意 `encoding="utf-8"` 显式声明，跨平台不踩坑。二进制模式（`rb`/`wb`）处理图片、压缩包等非文本数据。

```python
import json, os
data = {"name": "demo", "tags": ["a", "b"]}
with open("/tmp/demo.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False)
with open("/tmp/demo.json", encoding="utf-8") as f:
    print(json.load(f))
os.remove("/tmp/demo.json")
```

### 七、错误与异常

异常是 Python 的常规控制流，不是"出了大事"。结构是 `try / except / else / finally`：`else` 只在没异常时跑，`finally` 不管有没有异常都跑（放清理代码）。

自定义异常就两行：继承 `Exception` 写个类。`raise` 主动抛，`raise ... from ...` 保留异常链。**裸 `except:` 是坏习惯**，它会吞掉 KeyboardInterrupt 和拼写错误，永远捕获具体的异常类型。

```python
class AgeError(Exception):
    pass

def check(age):
    if age < 0:
        raise AgeError(f"年龄不能为负：{age}")
    return age

for v in [20, -3]:
    try:
        print("合法年龄：", check(v))
    except AgeError as e:
        print("捕获：", e)
    else:
        print("无异常，继续")
    finally:
        print("---")
```

### 八、函数：装饰器与闭包

函数是一等公民：可赋值、可传参、可返回。**闭包**是"函数 + 它记住的外部变量"，装饰器是"接收函数、返回新函数"的语法糖（`@deco` 等价于 `f = deco(f)`）。

两个经典坑：

1. **可变默认参数**：`def f(x, lst=[])` 里的 `[]` 只创建一次，多次调用会累积。改成 `lst=None` 再在函数里 `lst = lst or []`。
2. **循环闭包延迟绑定**：循环里定义的 lambda 记住的是变量名不是当时的值，全跑完都指向最后一个。用默认参数 `lambda i=i: ...` 立刻绑定。

```python
import functools, time

def timer(fn):
    @functools.wraps(fn)
    def wrapper(*a, **kw):
        t0 = time.time()
        r = fn(*a, **kw)
        print(f"{fn.__name__} 耗时 {time.time()-t0:.3f}s")
        return r
    return wrapper

@timer
def work():
    time.sleep(0.1)
    return 42

print(work())

# 坑1：可变默认参数
def bad(x, box=[]):
    box.append(x)
    return box
print(bad(1), bad(2))   # [1] [1, 2] —— box 被复用了！

def good(x, box=None):
    box = box if box is not None else []
    box.append(x)
    return box
print(good(1), good(2))  # [1] [2] —— 每次独立

# 坑2：循环闭包
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])  # [0, 1, 2]，不加 i=i 会全是 2
```

### 九、模块：代码的组织方式

一个 `.py` 文件就是一个模块。`import` 只执行一次，后续导入复用缓存。`if __name__ == "__main__":` 让文件既能被导入又能直接运行。包就是含 `__init__.py` 的目录（3.3+ 之后命名空间包可省略，但显式写更清楚）。

导入时的执行顺序坑：**循环导入**（A 导入 B，B 又导入 A）会导致半初始化模块。解法是把导入挪到函数内部延迟执行，或重构拆分模块。

另外两个工程细节：`__all__` 列表控制 `from 模块 import *` 时导出哪些名字，不写的话以下划线开头的私有名字会被排除；包内模块互相引用用相对导入（`from .util import double`），搬包时不依赖顶层包名，复用性更好。

```python
import sys, os
os.makedirs("/tmp/mypkg", exist_ok=True)
with open("/tmp/mypkg/util.py", "w") as f:
    f.write("def double(n):\n    return n * 2\n")
sys.path.insert(0, "/tmp/mypkg")
import util
print(util.double(21))   # 42
sys.path.remove("/tmp/mypkg")
```

### 十、面向对象编程

类是"数据 + 操作数据的函数"的打包。掌握 OOP 抓四件事：

- **`__init__` 初始化状态**，`self` 就是实例本身。
- **魔术方法**定义对象的"语法行为"：`__len__` 让 `len()` 可用，`__getitem__` 让 `[]` 可用，`__str__` 决定 `print` 长什么样。
- **继承是"是一个"关系**，`super()` 调父类方法；多用组合（"有一个"）少用深继承，继承层次超过三层就要警惕。
- **`@property`** 把方法伪装成属性，对外接口稳定、内部可加校验逻辑。

```python
class Bag:
    def __init__(self):
        self._items = []
    def add(self, x):
        self._items.append(x)
    def __len__(self):
        return len(self._items)
    def __getitem__(self, i):
        return self._items[i]
    def __str__(self):
        return f"Bag({self._items})"
    @property
    def total(self):
        return sum(self._items)

class CountBag(Bag):
    def add(self, x):          # 重写：只收数字
        if not isinstance(x, (int, float)):
            raise TypeError("只收数字")
        super().add(x)

b = CountBag()
b.add(3); b.add(4)
print(b, len(b), b[0], b.total)  # Bag([3, 4]) 2 3 7
```

### 十一、执行环境

理解程序"在哪跑、怎么跑"：`sys.argv` 拿命令行参数，`sys.path` 决定 import 去哪找模块，`os.environ` 读环境变量，`__file__` 告诉你当前脚本在磁盘上的位置（写相对路径资源文件时靠它定位）。

虚拟环境（venv）给每个项目独立的包空间，避免版本打架——这是工程实践的底线，今天所有 Python 项目都应该跑在 venv 里。标准流程三步：`python3 -m venv .venv` 建环境，`source .venv/bin/activate` 激活，`pip install` 装包。配合 `pip freeze > requirements.txt` 锁定版本，换机器一键还原。

## 第二部分：高级主题

### 十二、正则表达式

正则处理"有规律的文本"：提取、校验、替换。记住三件套：`.` 任意字符（除换行）、`*` 零或多次、`+` 一或多次；`^$` 锚定首尾；`()` 分组捕获。**原始字符串 `r"..."`** 写正则，反斜杠不用双重转义。

忠告：正则适合"模式清晰"的文本；嵌套结构（HTML、JSON）用专用解析器，不要硬上正则。

```python
import re
text = "联系：zhang@example.com 或 138-0000-1111"
emails = re.findall(r"[\w.]+@[\w.]+", text)
print(emails)                        # ['zhang@example.com']
m = re.search(r"(\d{3})-(\d{4})-(\d{4})", text)
print(m.groups())                    # ('138', '0000', '1111')
print(re.sub(r"\d{3}-\d{4}-\d{4}", "***", text))
```

### 十三、多线程编程

线程适合 **IO 密集**任务（网络请求、文件读写）：一个线程等 IO 时，GIL 会释放给别的线程。**CPU 密集**任务（纯计算）受 GIL 限制，多线程跑不满多核，要用 `multiprocessing` 或 C 扩展。

`concurrent.futures.ThreadPoolExecutor` 是现代写法：扔任务进去，`as_completed` 收结果，不用手工管线程生命周期。线程间共享数据要加锁（`threading.Lock`），否则计数器这类操作会丢数——这是并发 bug 里最常见的一类。

```python
import concurrent.futures, time

def fetch(i):
    time.sleep(0.1)   # 模拟 IO 等待
    return i * i

t0 = time.time()
with concurrent.futures.ThreadPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(fetch, range(8)))
print(results, f"耗时 {time.time()-t0:.2f}s")  # 8 个任务约 0.2s 而非 0.8s
```

### 十四、网络与 Web 编程入门

标准库自带网络全家桶：`socket` 打地基，`http.client` / `urllib` 做 HTTP 客户端，`http.server` 搭简易服务端，`ftplib` / `smtplib` / `poplib` / `imaplib` 覆盖文件传输和邮件协议。原理相通：都是"连上去、按协议格式收发文本或字节"。

Web 开发方向，历史上 CGI 是"每个请求起一个进程跑脚本"的古老方案，今天已被 WSGI/ASGI + 框架取代，了解其"请求进、响应出"的思想即可，不必深究。

### 十五、数据库编程

Python 访问关系数据库的统一接口叫 **DB-API**：`connect → cursor → execute → fetch` 四步走，换数据库（SQLite/MySQL/PostgreSQL）基本只换驱动和连接串。ORM（对象关系映射）再包一层，让你用类和对象操作代替手写 SQL。

防注入铁律：**永远用参数化查询**（占位符传参），不要字符串拼接 SQL——拼接是 SQL 注入的万恶之源。

```python
import sqlite3
conn = sqlite3.connect(":memory:")
cur = conn.cursor()
cur.execute("CREATE TABLE user(id INTEGER PRIMARY KEY, name TEXT)")
cur.execute("INSERT INTO user(name) VALUES(?)", ("alice",))  # 参数化，防注入
cur.execute("INSERT INTO user(name) VALUES(?)", ("bob",))
print(cur.execute("SELECT * FROM user WHERE name=?", ("bob",)).fetchall())
conn.close()
```

### 十六、扩展与混合编程

Python 慢在解释执行，热路径可以用 C/C++ 写扩展模块提速；反过来也可以把 Python 解释器嵌入 C 程序。Cython 是折中路线：写接近 Python 的语法，编译成 C 扩展。对大多数人，记住"性能不够时先 profile，找到热点再优化"——90% 的性能问题靠算法和缓存解决，轮不到写 C。

还有一条少有人走但很有价值的路：**给 CPython 本体做贡献**。从修文档 typo、补测试用例开始，既能深入理解解释器，又是含金量极高的公开履历。标准库的 `Tools/` 目录里藏着不少官方小工具，读它们的源码是学习"Python 式写法"的捷径。

## 常见坑（速查）

1. **可变默认参数** `def f(x, lst=[])`：默认值只创建一次，用 `None` 哨兵。
2. **闭包延迟绑定**：循环里 `lambda: i` 全指向最终值，加默认参数 `i=i` 立刻绑定。
3. **`is` vs `==`**：比较值永远用 `==`，`is` 只用于 `None` 判断（`x is None`）。
4. **深浅拷贝**：`copy()` 只拷一层，嵌套结构修改会互相影响，用 `copy.deepcopy`。
5. **裸 except**：吞掉所有异常包括拼写错误，永远捕获具体类型。
6. **字符串拼接**：循环里 `+=` 拼大文本慢，用列表攒 + `"".join()`。
7. **GIL**：CPU 密集任务多线程不加速，用多进程。
8. **忘记 `encoding="utf-8"`**：跨平台读写文本文件显式声明编码。
9. **循环导入**：A↔B 互相 import，延迟导入或重构拆分。
10. **SQL 拼接**：永远参数化查询，杜绝注入。

## 练习建议

1. 用生成器写一个"惰性读大文件"工具：逐行统计词频，对比一次性 `read()` 的内存占用（`tracemalloc` 可量化）。
2. 给上面的 `timer` 装饰器加参数（比如 `@timer(unit="ms")`），体会"装饰器工厂"三层嵌套的写法。
3. 实现一个带 `__len__` / `__iter__` / `__contains__` 的自定义容器类，用 `for` 和 `in` 测试它。
4. 用 `ThreadPoolExecutor` 并发请求 20 个 URL（可用 httpbin.org），对比串行耗时，观察线程数对速度的影响曲线。
5. 用 `sqlite3` 建一个通讯录小应用：增删改查 + 参数化查询，再故意写一个拼接版 SQL，亲手验证注入是怎么发生的（本地练习，切勿用于真实系统）。
6. 每天读 10 分钟标准库文档（`collections`、`itertools`、`functools` 优先），把"重复造的轮子"逐个换成标准库实现。
7. 挑一个你写过的 200 行以上脚本，用本文的坑清单逐条审查一遍，亲手改掉至少三个问题。

本文为学习笔记（编纂），用自己的话重写；
