# Python 入门笔记：从第一行代码到面向对象

> 学习路线：认识 Python → 基本语法 → 流程控制 → 函数 → 模块 → 数据结构 → 实战小项目 → 面向对象 → 文件与异常。

## 1. 认识 Python

Python 是一门解释型、面向对象、动态类型的高级语言，由 Guido van Rossum 创造。它的语法贴近自然语言，适合作为第一门编程语言，也广泛用于自动化脚本、数据处理和 Web 后端。

运行 Python 有两种姿势：

- **交互解释器**：终端输入 `python3`，看到 `>>>` 提示符后逐行执行，适合试小想法。
- **脚本文件**：把代码写进 `.py` 文件，用 `python3 xxx.py` 一次跑完，适合正式程序。

```python
# hello.py —— 第一个脚本
print("你好，Python！")
```

```bash
python3 hello.py
```

几点基本规矩：

- `#` 开头的是单行注释，解释器会忽略。
- Python **区分大小写**，`name` 和 `Name` 是两个变量。
- 变量名只能以字母或下划线开头，后面可跟字母、数字、下划线。
- 遇到不懂的内置用法，进解释器打 `help(len)` 看文档。

## 2. 数字、字符串与格式化

### 数字

整数、浮点数都直接写，科学计数法用 `e`：

```python
count = 42            # int
price = 9.99          # float
big = 1.5e6           # 1500000.0，科学计数法
print(count, price, big)
```

### 字符串

单引号、双引号、三引号都可以；三引号适合写多行文本。字符串一旦创建就**不可变**——所有"修改"操作返回的都是新串。

```python
s1 = '单引号'
s2 = "双引号"
poem = """三引号
可以换行写诗"""
print(s1, s2)
print(poem)
```

转义字符处理特殊符号，`r` 前缀表示"所见即所得"的原始字符串（写正则和 Windows 路径时常用）：

```python
print("第一行\n第二行\t缩进")   # \n 换行，\t 制表符
print("反斜杠要写成 \\\\")
print(r"C:\new\test")            # 原始字符串，\n 不会被转义
```

### 格式化输出

`str.format()` 是最常用的插值方式，大括号是占位符：

```python
name = "小林"
days = 7
print("姓名：{}，坚持了 {} 天".format(name, days))
print("姓名：{0}，坚持了 {1} 天；再说一遍，{0}".format(name, days))  # 位置参数可复用
print("姓名：{n}，天数：{d}".format(n=name, d=days))                  # 关键字参数

pi = 3.1415926
print("圆周率保留两位小数：{:.2f}".format(pi))
print("右对齐占 10 格：{:>10}".format(name))
print("居中并用 * 填充：{:*^10}".format(name))
```

`print()` 默认以空格分隔、换行结尾；`end` 参数可以改结尾：

```python
print("a", "b", "c", sep="-")   # a-b-c
print("不换行", end=" ")
print("接到同一行")
```

## 3. 运算符与表达式

比较运算返回布尔值，Python 支持链式比较，读起来像数学：

```python
score = 85
print(60 <= score < 90)   # True，相当于 60 <= score and score < 90
print(score == 85, score != 100)
```

布尔运算 `not` / `and` / `or` 有短路特性：`and` 遇到 False 就停，`or` 遇到 True 就停：

```python
logged_in = True
has_ticket = False
print(logged_in and has_ticket)  # False
print(logged_in or has_ticket)   # True
print(not has_ticket)            # True
```

## 4. 控制流：for、while 与输入

`for` 配 `range()` 是最常见的计数循环；`while True` 配 `break` 写交互循环：

```python
for i in range(1, 6):      # 1 到 5，不含 6
    print("第", i, "次打卡")
```

```python
# 交互循环：输入 quit 退出，并统计每次输入的长度
while True:
    text = input("说点什么（输入 quit 退出）：")
    if text == "quit":
        break
    print("你输入了", len(text), "个字符")
```

> 上面这段是交互式的，跑的时候需要手动输入；验证时可以用管道喂输入：`echo quit | python3 loop.py`。

## 5. 函数与作用域

函数把一段逻辑打包，`def` 定义、`return` 返回：

```python
def circle_area(radius):
    return 3.14159 * radius * radius

print(circle_area(2))   # 12.56636
```

函数里的变量默认是局部的，不会污染外面；同名时局部会"遮住"全局。想在函数里改全局变量，用 `global` 声明：

```python
total = 0

def add_wrong(n):
    total = total + n   # 报错！函数内 total 被当成局部变量

def add_right(n):
    global total
    total = total + n

add_right(5)
add_right(3)
print(total)   # 8
```

## 6. 模块与命令行参数

一个 `.py` 文件就是一个模块，`import` 拿来即用。`sys.argv` 能拿到命令行参数，写小工具时很有用（`argv[0]` 是脚本自己的名字）：

```python
# args_demo.py
import sys

print("脚本名：", sys.argv[0])
print("收到的参数：", sys.argv[1:])
```

```bash
python3 args_demo.py 早上好 8点
# 脚本名： args_demo.py
# 收到的参数： ['早上好', '8点']
```

模块搜索按 `sys.path` 的顺序找；自己写的模块和脚本放同一目录最省心。

## 7. 数据结构：列表与元组

列表可变、方括号；元组不可变、圆括号。索引都从 0 开始：

```python
todos = ["起床", "跑步", "读书"]
todos.append("写代码")     # 末尾追加
todos.sort()              # 原地排序，返回 None
print(todos[0])           # 取第一个
del todos[0]              # 按下标删除
print(len(todos), todos)
```

```python
point = (10, 20)          # 元组：坐标这种"一组值"适合用它
print(len(point))
# point[0] = 99           # TypeError：元组不可改
```

记住：`sort()` 是原地操作返回 `None`，别写成 `todos = todos.sort()`。

## 8. 实战：写个备份小脚本

把零散知识点串起来——用 `os` 和 `time` 给指定目录做时间戳备份：

```python
# backup_demo.py
import os
import time

def backup(src_dir, dst_dir):
    stamp = time.strftime("%Y%m%d_%H%M%S")
    if not os.path.isdir(src_dir):
        print("源目录不存在：", src_dir)
        return
    os.makedirs(dst_dir, exist_ok=True)
    count = 0
    for name in os.listdir(src_dir):
        src = os.path.join(src_dir, name)
        if os.path.isfile(src):
            dst = os.path.join(dst_dir, f"{stamp}_{name}.bak")
            with open(src, "rb") as f_in, open(dst, "wb") as f_out:
                f_out.write(f_in.read())
            count += 1
    print(f"备份完成，共 {count} 个文件 -> {dst_dir}")

if __name__ == "__main__":
    backup("/tmp/demo_src", "/tmp/demo_backup")
```

```bash
mkdir -p /tmp/demo_src && echo hello > /tmp/demo_src/a.txt
python3 backup_demo.py
ls /tmp/demo_backup
```

这个例子一次练到了：模块导入、函数、循环、分支、文件读写（`open/read/write/close`，用 `with` 自动关闭更稳）。

## 9. 面向对象：一句话入门

类是对象的蓝图；`self` 指对象自己；`__init__` 是构造方法；子类继承父类：

```python
class Pet:
    def __init__(self, name):
        self.name = name

    def greet(self):
        print(self.name, "摇了摇尾巴")

class Dog(Pet):                      # 继承
    def bark(self):
        print(self.name, "：汪汪！")

d = Dog("豆豆")
d.greet()
d.bark()
```

## 10. 异常处理

可能出错的代码用 `try/except` 包住；自己发现问题用 `raise` 主动抛：

```python
def safe_div(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        print("除数不能为 0，返回 None")
        return None

print(safe_div(10, 2))   # 5.0
print(safe_div(10, 0))   # 提示 + None

def must_be_adult(age):
    if age < 18:
        raise ValueError("未满 18 岁")
    return True

try:
    must_be_adult(15)
except ValueError as e:
    print("捕获到异常：", e)
```

## 11. 标准库速览

- `sys`：解释器相关（命令行参数、退出、模块路径）。
- `os`：操作系统交互（目录、文件、环境变量）。
- `time` / `datetime`：时间戳、日期格式化。
- `json`：结构化数据的序列化与解析。
- `re`：正则表达式，文本提取利器。

学完这些，就可以去啃"标准库文档 + 一个小项目"了：比如写个批量重命名工具、日志分析脚本。下一步建议：装饰器、生成器、虚拟环境与 pip。

本文为学习笔记（编纂），用自己的话重写；
