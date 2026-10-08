# Python 进阶技巧学习笔记

**本书定位一句话**：本笔记依据某 Python 进阶书的章节结构整理，把 26 个独立的小主题串成一套"从会写到写好"的查漏补缺清单。

## 一、`*args` 与 `**kwargs`：让函数"什么参数都吃得下"

`*args` 把多余的位置参数收进一个元组，`**kwargs` 把多余的关键字参数收进一个字典。它们真正的价值不是炫技，而是**写通用包装器**：装饰器、代理函数、框架回调都靠这对组合把参数原样透传下去。

```python
def logger(fn):
    def wrapper(*args, **kwargs):
        print(f"调用 {fn.__name__}，参数={args}，关键字={kwargs}")
        return fn(*args, **kwargs)
    return wrapper

@logger
def add(a, b=0):
    return a + b

print(add(3, b=4))
```

调用时的逆操作是"解包"：`f(*[1, 2])` 等价于 `f(1, 2)`，`f(**{"a": 1})` 等价于 `f(a=1)`。注意定义顺序只能是：位置参数 → 默认参数 → `*args` → 关键字参数 → `**kwargs`。

## 二、生成器：用"暂停"换内存

迭代器是实现了 `__iter__` 和 `__next__` 的对象；可迭代对象是能产出迭代器的东西（如列表）；生成器是**用 `yield` 写成的、懒惰产出值的函数**。它的设计动机很直接：列表一次性把所有元素装进内存，生成器一次只算一个，处理大文件、数据流时内存占用是常数级的。

```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1

for x in countdown(3):
    print(x)

# 生成器表达式：列表推导式把 [] 换成 () 即可
total = sum(i * i for i in range(10 ** 6))  # 不占内存
print(total)
```

## 三、map / filter / reduce：函数式三件套

`map` 做一对一变换，`filter` 做筛选，`reduce`（在 `functools` 里）把序列折叠成一个值。Python 3 里它们都返回迭代器，同样是惰性的。实际工作中，简单场景用推导式可读性更高；这三者的主场是**已经有一个现成函数、想把它套到整个序列上**的时候。

```python
from functools import reduce

nums = [1, 2, 3, 4, 5]
print(list(map(str, nums)))            # ['1','2','3','4','5']
print(list(filter(lambda x: x % 2, nums)))  # [1,3,5]
print(reduce(lambda a, b: a * b, nums))    # 120
```

## 四、set：去重与集合运算的利器

`set` 是无序不重复元素的集合，底层是哈希表，成员判断是 O(1)。它最自然的用途是**去重**和**集合运算**（交/并/差），比如"两个用户群体的重叠部分"。

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
print(a & b)   # 交集 {3, 4}
print(a | b)   # 并集
print(a - b)   # 差集 {1, 2}
print(list(set([1, 2, 2, 3])))  # 去重
```

## 五、三元运算符与"一行式"

`x if 条件 else y` 是把简单分支压缩成表达式。它的边界是**只用于赋值或返回这种"选一个值"的场景**，逻辑一复杂就拆回 `if` 语句，否则可读性崩盘。

```python
score = 85
level = "优秀" if score >= 80 else "继续努力"
print(level)
```

## 六、装饰器：不改源码给函数"加外挂"

装饰器的本质是**高阶函数**：接收一个函数，返回一个增强后的函数。`@deco` 只是 `f = deco(f)` 的语法糖。带参数的装饰器是"返回装饰器的函数"，多套一层；类也可以当装饰器（实现 `__call__`）。容易被忽略的细节是**用 `functools.wraps` 保留原函数的元信息**，否则文档工具和调试器看到的全是 `wrapper`。

```python
import functools
import time

def timer(unit="秒"):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            start = time.time()
            result = fn(*args, **kwargs)
            print(f"{fn.__name__} 耗时 {time.time() - start:.4f}{unit}")
            return result
        return wrapper
    return deco

@timer(unit="s")
def slow():
    time.sleep(0.01)

slow()
print(slow.__name__)  # 仍是 slow，而不是 wrapper
```

## 七、Global、Return 与对象变动：作用域的坑

函数里**读取**全局变量没问题，**赋值**就会创建局部变量——除非声明 `global`。更隐蔽的是可变对象：`def f(lst=[])` 的默认空列表只创建一次，多次调用会累积，这是 Python 最著名的新手陷阱。根因是**默认参数在函数定义时求值一次**，而不是每次调用。

```python
def append_bug(item, box=[]):
    box.append(item)
    return box

def append_ok(item, box=None):
    box = [] if box is None else box
    box.append(item)
    return box

print(append_bug(1))  # [1]
print(append_bug(2))  # [1, 2] —— 意外累积！
print(append_ok(1))   # [1]
print(append_ok(2))   # [2]
```

## 八、`__slots__`：给实例"瘦身"

普通实例的属性存在 `__dict__` 字典里，每个对象都有一份字典开销。当你要创建**成千上万个小对象**（如一条条数据记录）时，`__slots__ = ("x", "y")` 告诉解释器"这个类只有这几个属性"，改用固定偏移量存储，省内存也更快。代价是实例不能再随意挂新属性。

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y

p = Point(1, 2)
print(p.x, p.y)
print(hasattr(p, "__dict__"))  # False：没有属性字典了
```

## 九、虚拟环境与容器：工程化的两块基石

虚拟环境（`python -m venv`）给每个项目一套独立的第三方包，避免版本打架；`collections` 模块则是内置容器的"加强版"：`namedtuple` 给元组起名字、`defaultdict` 自动给缺失的键兜底、`Counter` 一行统计词频、`deque` 两端高效增删。

```python
from collections import namedtuple, defaultdict, Counter, deque

User = namedtuple("User", ["name", "age"])
u = User("小明", 20)
print(u.name, u[1])

groups = defaultdict(list)
groups["a"].append(1)
print(dict(groups))

print(Counter("abracadabra").most_common(2))

dq = deque([1, 2, 3])
dq.appendleft(0)
print(list(dq))
```

## 十、enumerate 与对象自省

`enumerate` 在循环里同时给下标和元素，替代手写计数器。对象自省是"运行时看自己"：`type()` 看类型、`id()` 看身份、`dir()` 列属性，`inspect` 模块能拿到函数签名和源码。写通用工具、调试框架代码时全靠它们。

```python
import inspect

for i, ch in enumerate("abc", start=1):
    print(i, ch)

def demo(a, b=2, *args, **kw):
    pass

print(inspect.signature(demo))
```

## 十一、推导式与 lambda：压缩表达

列表/集合/字典推导式把"循环+条件+收集"压成一行，**可读上限是两层嵌套**，超过就写回普通循环。`lambda` 是匿名小函数，只适合当参数传给 `sorted`、`key=`、`filter` 这类地方；需要复用或超过一行就老老实实用 `def`。

```python
squares = [x * x for x in range(10) if x % 2 == 0]
word_len = {w: len(w) for w in ["apple", "pear"]}
print(squares)
print(word_len)

students = [("a", 80), ("b", 95)]
print(sorted(students, key=lambda s: s[1], reverse=True))
```

## 十二、异常：try / except / else / finally 的完整语义

`except` 可以并列多个异常类型；`else` 只在**没有异常时**执行——适合放"成功了才做的事"，和 `try` 块里的主逻辑分开；`finally` 无论如何都执行，负责释放资源。自定义异常继承 `Exception` 即可，抛出用 `raise`。

```python
def divide(a, b):
    try:
        r = a / b
    except (ZeroDivisionError, TypeError) as e:
        print("参数有问题：", e)
    else:
        print("计算成功，结果是", r)
    finally:
        print("收尾：本次调用结束")

divide(10, 2)
divide(10, 0)
```

## 十三、For-Else：循环里的"没找到"分支

`for...else` 中 `else` 的含义是"循环**没有被 break 打断**才执行"，专门表达"遍历完都没找到"的逻辑，省掉一个标志变量。质数判断是它的经典用例。

```python
def is_prime(n):
    if n < 2:
        return False
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            break
    else:
        return True
    return False

print([n for n in range(2, 20) if is_prime(n)])
```

## 十四、open 函数与 C 扩展

`open` 务必用 `with` 语句包裹，自动关闭文件；读写文本**显式指定 `encoding="utf-8"`**，跨平台不踩坑。C 扩展（`ctypes` 直接调系统库、`SWIG`、手写 C API）是把 Python 的易用和 C 的速度拼起来的三条路，`ctypes` 零编译成本，最适合"调一个现成的动态库函数"。

```python
import ctypes

with open("/tmp/demo.txt", "w", encoding="utf-8") as f:
    f.write("你好，Python\n")

with open("/tmp/demo.txt", encoding="utf-8") as f:
    print(f.read().strip())

print("C int 占字节数：", ctypes.sizeof(ctypes.c_int))
```

## 十五、协程、函数缓存与上下文管理器

协程是"能暂停、能接收外部数据的生成器"（`yield` 既是产出也是入口），早期 Python 拿它做协作式并发。`functools.lru_cache` 给纯函数加记忆化，递归函数（如斐波那契）加速是指数级的。上下文管理器（`with` 背后的协议）统一了"申请—释放"模式，自己实现只需 `__enter__`/`__exit__`，或用 `contextlib` 更省事。

```python
from functools import lru_cache
from contextlib import contextmanager

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(30))  # 递归+缓存，瞬间算完

@contextmanager
def tag(name):
    print(f"<{name}>")
    yield
    print(f"</{name}>")

with tag("body"):
    print("内容")

# 协程：yield 双向通信
def averager():
    total = count = 0
    while True:
        x = yield total / count if count else 0
        total += x
        count += 1

avg = averager()
next(avg)          # 预激：推进到第一个 yield
print(avg.send(10))
print(avg.send(20))
```

## 常见坑速查

1. 可变默认参数只求值一次 → 用 `None` 兜底。
2. 装饰器丢元信息 → 记得 `functools.wraps`。
3. `except` 裸捕获（`except:`）会吞掉 `KeyboardInterrupt`，尽量写明异常类型。
4. 生成器只能消费一次，第二次遍历是空的。
5. `open` 不写 `encoding`，在 Windows 上大概率乱码。

## 练习建议

- 把常用的计时、重试、缓存逻辑各写成一个装饰器，攒成自己的工具箱。
- 用生成器实现一个"逐行读大文件并统计词频"的脚本，文件大于内存也照跑。
- 给任意一个"申请—释放"资源（计时器、临时目录、数据库连接）写上下文管理器。

本文为学习笔记（编纂），用自己的话重写；
