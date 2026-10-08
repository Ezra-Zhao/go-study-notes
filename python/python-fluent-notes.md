# 流畅的 Python：写出地道 Python 代码的学习笔记

**本书定位一句话**：本笔记依据某 Python 进阶书的章节结构整理，围绕"Python 数据模型"这一核心思想，讲透从数据结构、函数、面向对象到并发、元编程的地道写法。

## 第一部分 序幕：Python 数据模型

Python 的一切"魔法"都来自**特殊方法**（双下划线方法，如 `__len__`、`__getitem__`）。设计哲学是：解释器不直接操作对象，而是调用对象身上的协议方法。`len(x)` 其实是 `x.__len__()`，`x[0]` 是 `x.__getitem__(0)`。理解这一点，你就能让自定义类无缝接入 `for` 循环、`in` 判断、`sorted` 等所有内置机制——**写更少的代码，复用更多的语言基础设施**。

```python
from collections import namedtuple

Card = namedtuple("Card", ["rank", "suit"])

class Deck:
    ranks = [str(n) for n in range(2, 11)] + list("JQKA")
    suits = "黑桃 红桃 方块 梅花".split()

    def __init__(self):
        self._cards = [Card(r, s) for s in self.suits for r in self.ranks]

    def __len__(self):
        return len(self._cards)

    def __getitem__(self, pos):
        return self._cards[pos]

deck = Deck()
print(len(deck))          # 52：len 直接可用
print(deck[0], deck[-1])  # 索引、负索引免费获得
print(deck[:3])           # 切片也免费获得
for card in deck[:2]:     # 迭代自动工作
    print(card)
print(Card("A", "黑桃") in deck)  # in 判断自动工作
```

只实现了两个特殊方法，就"继承"了整个序列协议——这就是数据模型的威力。

## 第二部分 数据结构：序列、字典、文本

### 序列：切片与拆包

Python 序列的精髓在**切片**和**拆包**。切片 `s[a:b:c]` 左闭右开，支持负索引和步长；拆包 `a, b = b, a` 让交换变量不需要临时变量。`*` 还能收集剩余部分：`first, *rest = [1, 2, 3, 4]`。

```python
s = "Pythonista"
print(s[::2])        # Ptoit：隔一个取一个
print(s[::-1])       # 反转字符串

a, b = 1, 2
a, b = b, a          # 交换，无需临时变量
print(a, b)

head, *tail = [1, 2, 3, 4]
print(head, tail)    # 1 [2, 3, 4]
```

### 字典和集合：哈希的艺术

字典的键必须是**可哈希**的（不可变对象）。它的查找是 O(1)，代价是内存。处理"键不存在"有三招：`d.get(k, 默认值)`、`d.setdefault(k, 默认值)`、以及 `collections.defaultdict`。理解哈希才能理解为什么**字典在 Python 3.7+ 能保持插入顺序**（实现改用紧凑数组+稀疏索引）以及为什么**不能在遍历字典时增删键**。

```python
from collections import defaultdict

# 词频统计：defaultdict 让缺失的键自动从 0 开始
freq = defaultdict(int)
for ch in "abracadabra":
    freq[ch] += 1
print(dict(freq))

# 集合推导式去重并过滤
unique_vowels = {ch for ch in "abracadabra" if ch in "aeiou"}
print(unique_vowels)
```

### 文本和字节序列：编码是翻译

`str` 是给人看的字符，`bytes` 是给机器存的字节，**编码就是两者之间的翻译规则**。`encode` 把字符翻成字节，`decode` 把字节翻回字符。90% 的乱码问题都源于"编解码用的不是同一套规则"。处理网络数据、文件读写时，**第一时间想清楚你手里的是 str 还是 bytes**。

```python
text = "你好，Python"
data = text.encode("utf-8")     # str -> bytes
print(data, type(data))
print(data.decode("utf-8"))     # bytes -> str
print(data.decode("gbk", errors="replace"))  # 规则错了就乱码
```

## 第三部分 把函数视作对象

### 一等函数：函数也是值

在 Python 里函数是**一等公民**：可以赋值给变量、塞进列表、当参数传、从函数里返回。高阶函数（`map`、`filter`、`sorted(key=...)`）的本质就是"把行为当参数传"。想通了这一点，很多设计模式可以直接用函数实现，不用写类。

```python
def make_multiplier(n):
    def multiply(x):
        return x * n
    return multiply  # 返回函数：闭包

double = make_multiplier(2)
triple = make_multiplier(3)
print(double(5), triple(5))

words = ["apple", "pear", "fig", "banana"]
print(sorted(words, key=len))  # 把 len 当行为传进去
```

### 用一等函数实现设计模式：策略模式

经典的策略模式要写一堆策略类；在 Python 里，**策略就是一组普通函数**，用字典按名字分发。代码量少一个数量级，而且新增策略就是多写一个函数、字典里加一行。

```python
def discount_vip(amount):
    return amount * 0.8

def discount_new(amount):
    return amount * 0.9

def discount_none(amount):
    return amount

strategies = {"vip": discount_vip, "new": discount_new, "none": discount_none}

def checkout(amount, kind="none"):
    return strategies.get(kind, discount_none)(amount)

print(checkout(100, "vip"))
print(checkout(100, "new"))
```

### 函数装饰器和闭包：nonlocal 的舞台

闭包 = **函数 + 它记住的外部变量**。`nonlocal` 让内层函数能修改外层（非全局）变量，这是写计数器、缓存装饰器的关键。装饰器叠加时顺序是**从下往上执行、从上往下包装**，配合 `functools.wraps` 保留元信息。

```python
import functools

def count_calls(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        wrapper.calls += 1
        return fn(*args, **kwargs)
    wrapper.calls = 0
    return wrapper

def make_counter():
    n = 0
    def inc():
        nonlocal n   # 不写这行，n += 1 会报 UnboundLocalError
        n += 1
        return n
    return inc

@count_calls
def greet(name):
    return f"你好，{name}"

print(greet("小明"), greet("小红"))
print("调用次数：", greet.calls)

c = make_counter()
print(c(), c(), c())
```

## 第四部分 面向对象惯用法

### 对象引用、可变性与垃圾回收

Python 变量是**贴在对象上的标签**，赋值从不拷贝。函数传参传的是引用，所以可变对象（列表、字典）在函数里改了外面也看得见。需要独立副本时：`copy.copy` 浅拷贝（只拷一层）、`copy.deepcopy` 深拷贝（递归全拷）。垃圾回收靠**引用计数为主、标记清除为辅**，循环引用由后者兜底。

```python
import copy

a = [1, [2, 3]]
b = a            # 只是多了个标签，同一个对象
c = copy.copy(a)      # 浅拷贝：外层新，内层仍共享
d = copy.deepcopy(a)  # 深拷贝：彻底独立

a[1].append(99)
print("b:", b)   # 跟着变
print("c:", c)   # 内层跟着变
print("d:", d)   # 不变
```

### 符合 Python 风格的对象：repr 是门面

一个"像 Python 原生类型"的类至少做到：`__repr__` 返回能重建对象的字符串（调试器的门面）、`__str__` 给人看、`__eq__` 定义相等、`__hash__` 与相等保持一致（定义了 `__eq__` 却不定义 `__hash__`，对象会变得不可哈希——这是常见坑）。

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

    def __str__(self):
        return f"({self.x}, {self.y})"

    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)

    def __hash__(self):
        return hash((self.x, self.y))

    def __add__(self, other):   # 运算符重载：v1 + v2
        return Vector(self.x + other.x, self.y + other.y)

    def __abs__(self):          # abs(v) 求模长
        return (self.x ** 2 + self.y ** 2) ** 0.5

v1 = Vector(3, 4)
v2 = Vector(1, 2)
print(repr(v1))        # Vector(3, 4)
print(v1 + v2)         # (4, 6)
print(abs(v1))         # 5.0
print(len({v1, v2, Vector(3, 4)}))  # 2：相等且哈希一致，去重成功
```

### 接口：从协议到抽象基类

Python 的接口是**鸭子类型**：不看你是什么类，看你会什么（有没有 `__len__`、`__iter__`）。`collections.abc` 里的抽象基类（如 `Sequence`、`Mapping`）把"会什么"形式化了，还附赠混入方法——继承 `Sequence` 并实现 `__len__` 和 `__getitem__`，`in`、`index`、`count` 全都有了。**优先用协议/ABC 而不是死板的继承层级**。

### 继承的优缺点：组合优先

多重继承的方法解析顺序（MRO）用 C3 算法，`super()` 不是"调父类"而是"按 MRO 找下一个"。菱形继承一不小心就调错方法。经验法则：**想复用行为用组合（has-a），只有真正的"是一种"（is-a）关系才用继承**；混入类（Mixin）只加小功能、不单独实例化。

```python
class LoggerMixin:
    def log(self, msg):
        print(f"[{type(self).__name__}] {msg}")

class Service(LoggerMixin):
    def run(self):
        self.log("启动服务")

Service().run()
print(Service.__mro__)  # 查看方法解析顺序
```

## 第五部分 控制流程

### 可迭代对象、迭代器与生成器

`iter(x)` 拿迭代器，`next(it)` 取下一个，取完抛 `StopIteration`——`for` 循环就是这套协议的语法糖。生成器函数（`yield`）是**最省事的迭代器写法**，解释器替你维护状态机。数据管道用生成器串起来：读文件 → 清洗 → 统计，每一步都是惰性的，内存占用恒定。

```python
def gen_squares(n):
    for i in range(n):
        yield i * i

# 生成器管道：惰性求值
data = gen_squares(10 ** 6)
evens = (x for x in data if x % 2 == 0)
print(sum(x for x in evens if x < 100))  # 只算需要的那部分
```

### 上下文管理器和 else 块

`with` 背后的 `__enter__`/`__exit__` 协议统一了"申请—释放"模式；`__exit__` 返回真还能吞掉异常。`else` 块配 `try`（无异常才执行）、配 `for`/`while`（没被 break 才执行）；注意 `with` 语句从未支持过 `else` 子句。

### 协程：yield 的双向通道

`yield` 不只是产出值，`gen.send(x)` 还能把值**送进**暂停的生成器，`yield from` 可以委托给子生成器。早期异步编程就建立在这套机制上：事件循环驱动一堆协程，谁阻塞就切走。

```python
def averager():
    total = count = 0
    while True:
        x = yield round(total / count, 2) if count else 0.0
        total += x
        count += 1

avg = averager()
next(avg)            # 预激到第一个 yield
print(avg.send(10))  # 10.0
print(avg.send(20))  # 15.0
print(avg.send(30))  # 20.0
```

### 用期物与 asyncio 处理并发

`concurrent.futures` 把"线程池/进程池"统一成**期物（Future）**接口：提交任务拿回期物，结果就绪再取。I/O 密集用线程池，CPU 密集用进程池（绕开 GIL）。`asyncio` 则是单线程的协作式并发，`async/await` 让异步代码写出同步的样子，适合**大量 I/O 等待**的场景（爬虫、API 聚合）。

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

def fetch(n):
    return n * n

with ThreadPoolExecutor(max_workers=4) as pool:
    print(list(pool.map(fetch, range(8))))  # 线程池并行

async def hello(name, delay):
    await asyncio.sleep(delay)
    return f"你好，{name}"

async def main():
    results = await asyncio.gather(
        hello("小明", 0.1), hello("小红", 0.05)
    )
    print(results)

asyncio.run(main())
```

## 第六部分 元编程：在运行时"写代码"

### 动态属性和特性

`__getattr__` 只在**常规查找失败**时触发，适合做懒加载、代理；`@property` 把方法伪装成属性，让"取值时顺手做计算/校验"成为可能，且调用方无感。这是"先简单、后复杂"的设计：先用普通属性，以后需要逻辑了再换成 property，接口不变。

```python
class LazyData:
    def __init__(self):
        self._cache = {}

    def __getattr__(self, name):
        # 只有找不到的属性才到这里
        print(f"懒加载：{name}")
        value = f"<数据:{name}>"
        self._cache[name] = value
        setattr(self, name, value)  # 挂到实例属性上，下次直接命中
        return value

    @property
    def count(self):
        return len(self._cache)

d = LazyData()
print(d.users)   # 触发 __getattr__
print(d.users)   # 已在实例字典，直接命中，不再触发
print(d.count)   # 像属性一样调用
```

### 属性描述符：可复用的属性逻辑

描述符是实现了 `__get__`/`__set__`/`__delete__` 的类，**把"属性的存取逻辑"抽出来复用**。`property` 只是描述符的一种语法糖。当多个属性要做同样的校验（非空、范围、类型）时，写一个描述符类比写十个 property 干净得多。

```python
class Positive:
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        return obj.__dict__[self.name]

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name} 必须是正数")
        obj.__dict__[self.name] = value

class Product:
    price = Positive()   # 两个属性复用同一套校验
    stock = Positive()

p = Product()
p.price = 99
print(p.price)
try:
    p.stock = -5
except ValueError as e:
    print("拦截：", e)
```

### 类元编程：type 与元类

`type` 既能查类型，也能**动态造类**：`type("名", (父类,), {属性})`。元类（metaclass）是"类的类"，类定义语句执行时会被元类拦截——框架用它做**类注册表、自动校验、API 自动生成**。威力大、晦涩，日常业务代码能不用就不用；但读懂框架源码必须懂它。

```python
# 用 type 动态创建一个类
Animal = type("Animal", (), {"speak": lambda self: "叫声"})
print(Animal().speak())

# 元类：自动给类注册名字
registry = {}

class RegistryMeta(type):
    def __new__(mcs, name, bases, attrs):
        cls = super().__new__(mcs, name, bases, attrs)
        if name != "Base":
            registry[name] = cls
        return cls

class Base(metaclass=RegistryMeta):
    pass

class Dog(Base):
    pass

class Cat(Base):
    pass

print(sorted(registry))  # ['Cat', 'Dog']：定义类时自动注册
```

## 常见坑速查

1. 定义了 `__eq__` 没定义 `__hash__` → 对象变不可哈希，塞不进 set/dict 键。
2. `__getattr__` 里访问不存在的属性会无限递归 → 内部用 `self.__dict__` 或 `object.__getattribute__`。
3. 闭包里想改外层变量忘了 `nonlocal` → 报 `UnboundLocalError`。
4. 在遍历字典时增删键 → `RuntimeError`；先 `list(d)` 快照再改。
5. `asyncio` 里调阻塞函数（如 `time.sleep`）→ 整个事件循环卡死，用 `await asyncio.sleep`。

## 练习建议

- 给自定义类实现 `__len__`/`__getitem__`/`__repr__`/`__eq__` 四件套，体会"协议"思维。
- 用字典分发函数重写一段 if-elif 策略代码，对比行数。
- 写一个带校验的描述符，用在 3 个以上属性上。
- 用 `ThreadPoolExecutor` 并行下载 10 张图片（或模拟 I/O），对比串行耗时。

本文为学习笔记（编纂），用自己的话重写；
