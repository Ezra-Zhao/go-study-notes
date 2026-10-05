# Python 语言入门笔记：核心语法与外围生态学习路线

> 这是一份按"语言核心 → 外围工具"组织的 Python 系统学习笔记：第一部分打地基（类型、语句、函数、模块、类、异常），第二部分学用 Python 干实事（内置工具、文件、网络、框架）。

## 第一部分：语言核心

### 第 1 章 · 起步：为什么选 Python

选 Python 的三条实在理由：**写得快**（代码量少、试错成本低）、**质量稳**（可读性强、团队好维护）、**能集成**（C 扩展、调系统命令、衔接各种组件）。

运行程序两种方式：交互解释器适合试代码，脚本文件适合正式跑。一个 `.py` 文件就是一个**模块**，这是 Python 组织代码的基本单位。

```python
# module_demo.py
GREETING = "你好"

def hello(name):
    return GREETING + "，" + name + "！"

if __name__ == "__main__":
    print(hello("世界"))
```

```bash
python3 module_demo.py # 输出：你好，世界！
```

### 第 2 章 · 类型与操作符

Python 程序 = 模块 → 语句 → 表达式。内置类型是地基，记住"身份、类型、值"三属性：`id()` 看身份，`type()` 看类型，`==` 比值。

```python
# 数字：int / float / 复数
print(0b1010, 0o17, 0xFF) # 二进制、八进制、十六进制字面量：10 15 255
print(3 + 4j) # 复数也是一等公民

# 字符串：去空格、转大写、替换
s = "  Python 入门  "
clean = s.strip()
print(clean) # Python 入门
print(clean.upper()) # PYTHON 入门
print(clean.replace("入门", "进阶"))

# 列表：可变序列，切片赋值一次换一段
nums = [1, 2, 3]
nums[1:2] = [20, 30]
print(nums) # [1, 20, 30, 3]

# 字典：get 带默认值，不抛错
cfg = {"host": "localhost", "port": 8080}
print(cfg.get("timeout", 30)) # 30

# 元组：不可变
t = (1, "a")
print(len(t))
```

```python
# 文件：open / read / write，用 with 自动关闭
with open("/tmp/note.txt", "w") as f:
    f.write("第一行\n第二行\n")
with open("/tmp/note.txt") as f:
    print(f.read())
```

常见陷阱：可变对象做默认参数、浮点精度、`is` 与 `==` 混用。

### 第 3 章 · 基本语句

赋值不只是 `=`：支持序列解包，一行换两个变量：

```python
x, y = 1, 2
x, y = y, x + y # 交换与递推，一行搞定
print(x, y) # 2 3

# if 条件测试
score = 78
if score >= 90:
    level = "优"
elif score >= 60:
    level = "良"
else:
    level = "待努力"
print(level) # 良
```

```python
# while 与 for；else 在循环没被 break 打断时执行
n = 1
while n <= 3:
    print("while:", n)
    n += 1

for ch in "abc":
    print("for:", ch)
else:
    print("循环正常结束")
```

### 第 4 章 · 函数：作用域与参数

函数解决复用和抽象。作用域规则 LEGB：局部 → 外层函数 → 全局 → 内置。参数形式很灵活：

```python
def report(title, *items, sep="、", **meta):
    body = sep.join(str(i) for i in items)
    extra = "；".join(k + "=" + str(v) for k, v in meta.items())
    result = "【" + title + "】" + body
    if extra:
        result += "（" + extra + "）"
    return result

print(report("水果", "苹果", "梨", sep="|", 产地="山东"))
```

```python
add = lambda a, b: a + b # lambda：一次性小函数
print(add(2, 3)) # 5
print(sorted(["bb", "a", "ccc"], key=lambda s: len(s))) # 按长度排序
```

### 第 5 章 · 模块：名字空间与导入

模块文件就是独立的名字空间，天然隔离变量。导入有三种姿势：

```python
import math # 全量导入，用 math.sqrt 调用
from math import sqrt # 只拿要的，直接 sqrt 调用
from math import sqrt as sq # 重命名防冲突
print(sq(16)) # 4.0
```

`from x import *` 会污染名字空间，生产代码里别用。包（package）就是带 `__init__.py` 的目录，`import pkg.mod` 按点号寻址。

### 第 6 章 · 类：继承与操作符重载

类把数据和行为打包。继承形成名字空间搜索树（子类 → 父类 → …），方法解析按 MRO 来：

```python
class Counter:
    def __init__(self):
        self.n = 0

    def tick(self):
        self.n += 1
        return self

    def __add__(self, other): # 操作符重载：+ 号的行为自己定
        c = Counter()
        c.n = self.n + other.n
        return c

    def __repr__(self):
        return "Counter(" + str(self.n) + ")"

a, b = Counter(), Counter()
a.tick().tick()
b.tick()
print(a + b) # Counter(3)
print(Counter.__mro__) # 方法解析顺序
```

设计建议：先想清楚"这个类代表什么、能做什么"，再写代码；组合（has-a）往往比继承（is-a）更稳。

### 第 7 章 · 异常：捕获模式

```python
def read_config(path):
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        print("配置文件缺失，用默认配置")
        return ""
    except OSError as e:
        print("读取失败：", e)
        raise # 处理不了就继续往上抛
    else:
        print("读取成功") # 没异常才走 else
    finally:
        print("收尾工作") # 有没有异常都执行

print(repr(read_config("/tmp/no_such_file.cfg")))
```

```python
class AppError(Exception):
    """自定义异常：继承 Exception，别继承 BaseException"""
    pass

try:
    raise AppError("业务规则被违反")
except AppError as e:
    print("捕获自定义异常：", e)
```

## 第二部分：外围层

### 第 8 章 · 内置工具

```python
data = [3, 1, 4, 1, 5]
print(list(enumerate(data))) # 带下标遍历
print(list(zip("abc", data))) # 配对
print(list(map(str, data))) # 映射
print(list(filter(lambda x: x > 2, data))) # 过滤

import itertools, collections
print(list(itertools.islice(range(100), 5))) # 惰性切片
print(dict(collections.Counter("abracadabra"))) # 计数器
```

标准库按需查文档即可，记住几个大类：文件（os/shutil/pathlib）、数据（json/csv/re）、网络（urllib/socket）、并发（threading/asyncio）。

### 第 9 章 · 用 Python 干实事

```python
import os
from pathlib import Path

# 批量处理文件：把目录下所有.log 改名加日期前缀
demo = Path("/tmp/logs_demo")
demo.mkdir(exist_ok=True)
for i in range(3):
    (demo / ("app" + str(i) + ".log")).write_text("log line " + str(i) + "\n")

for p in demo.glob("*.log"):
    new_name = "20261005_" + p.name
    p.rename(p.parent / new_name)
print(sorted(x.name for x in demo.iterdir()))
```

这一章的思路是"组合拳"：数据结构操作 + 文件操作 + 调外部程序（`subprocess`）+ 简单网络任务，拼出真正的工具脚本。

### 第 10 章 · 框架和应用

这一章是综合案例：比如用 Tkinter 写个表格数据编辑器、通过系统接口操控其他软件、把 Python 嵌进已有系统。思路比代码重要——**先定义好模块边界和数据流，再动手**。

> 年代注：书中部分集成技术（如 COM 接口、JPython）年代较早，现代开发中直接用到的机会很少，理解"Python 如何与外部系统集成"的思路即可；Tkinter 仍是标准库自带的 GUI 方案，写小工具够用。

## 第三部分：附录与学习建议

- **资源**：官方文档（docs.python.org）是最好的手册；标准库文档按需查。
- **平台差异**：路径分隔符、换行符、编码（Windows 上多留心），写跨平台代码用 `os.path` / `pathlib`。
- **练习**：每章后的练习一定要动手做，看懂和会写是两回事。

学习路线建议：核心语法（本笔记第一部分）→ 写 3–5 个自动化小脚本（第二部分思路）→ 选一个方向深入（Web / 数据 / 运维 / AI）。遇到问题先读报错，再查文档，最后问人。

本文为学习笔记（编纂），用自己的话重写；
