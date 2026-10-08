# Python 编程：以解决计算问题为导向的学习笔记

> 资料定位：本笔记依据某 Python 入门资料的章节结构整理，围绕"用 Python 解决实际计算问题"这一主线，把计算思维、基础语法、数值计算和文件操作串成一条完整的学习路径。

## 一、计算思维：先把问题想清楚，再动手写代码

**什么是计算。** 计算的本质，是把一个现实问题翻译成"输入 → 一系列明确步骤 → 输出"。写程序之前，先问自己三个问题：输入是什么、输出应该是什么、中间每一步有没有歧义。凡是说不清楚中间步骤的问题，代码一定写不顺。

**什么是算法。** 算法就是解决问题的有限步骤序列，三个硬性要求：每一步含义明确、步骤数量有限、执行后一定能停下来。评价算法好坏主要看两点：对不对（正确性）、快不快（步数随输入规模增长的速度）。同样的需求，算法选得好，效果天差地别：

```python
def sum_loop(n):
    """循环累加：步数随 n 线性增长"""
    total = 0
    for i in range(1, n + 1):
        total += i
    return total


def sum_formula(n):
    """等差数列求和公式：一步到位"""
    return n * (n + 1) // 2


print(sum_loop(100))    # 5050
print(sum_formula(100))  # 5050，算 1 亿也几乎不耗时
```

这个例子说明：**编程的大头功夫花在"想"上，而不是"写"上**。语言只是表达工具，算法才是解法本身。

**复杂度：用计时直观感受。** 上面两种写法的差距，拿秒表一测就明白：

```python
import time


def sum_loop(n):
    total = 0
    for i in range(1, n + 1):
        total += i
    return total


def sum_formula(n):
    return n * (n + 1) // 2


n = 1_000_000
t0 = time.perf_counter(); r1 = sum_loop(n); t1 = time.perf_counter()
r2 = sum_formula(n); t2 = time.perf_counter()
print(f"循环累加耗时: {t1 - t0:.4f} 秒")
print(f"公式耗时: {t2 - t1:.6f} 秒")  # 快了近万倍
```

循环版的耗时随 n 线性增长，n 翻 10 倍耗时也翻 10 倍；公式版 n 再大也几乎不耗时。衡量算法的"快"，看的就是"耗时随规模怎么涨"，这叫时间复杂度。写程序时多问一句"数据量大了这段代码还撑得住吗"，能避开大量性能坑。

**计算模型：人机分工。** 人脑擅长模糊判断、联想和权衡，计算机擅长精确、高速地重复执行。编程就是把人的思路拆成机器能一步步执行的指令。现代计算机普遍采用"存储程序"的体系结构：指令和数据都以二进制形式存放在内存里，CPU 按顺序取指、执行。理解这一点，就能明白为什么变量需要"类型"（告诉机器这段内存该怎么解读），为什么循环本质上是"跳回去再执行一遍"。

## 二、Python 基础：变量、赋值与内存

**变量是贴在对象上的标签。** `x = [1, 2, 3]` 的真实含义是：创建一个列表对象，再把名字 `x` 指向它。赋值操作绑定的是引用，而不是复制一份数据。这个心智模型能解释 Python 里大量"灵异现象"：

```python
a = [1, 2, 3]
b = a          # b 和 a 指向同一个列表对象
b.append(4)
print(a)       # [1, 2, 3, 4] —— a 也被"连累"了

c = a.copy()   # 想要独立副本时，显式复制
c.append(5)
print(a)       # [1, 2, 3, 4] 不受影响
print(c)       # [1, 2, 3, 4, 5]
```

**可变与不可变是分水岭。** 列表、字典、集合是可变的，可以原地修改；整数、字符串、元组是不可变的，"修改"实际是创建新对象。函数传参时，可变对象在函数内被改会影响外部，这是新手 bug 的重灾区，记住"传的是引用"即可防患。

**内存管理不用你操心细节，但要懂代价。** Python 有自动垃圾回收，不再被引用的对象会被清理。你不需要手动释放内存，但要知道：频繁创建大对象、循环引用、把大列表常驻内存，都是有代价的。用 `id()` 可以查看对象身份，`is` 比较的是身份、`==` 比较的是值，别混用：

```python
x = [1, 2]
y = [1, 2]
z = x
print(x == y)  # True：值相等
print(x is y)  # False：是两个不同的对象
print(x is z)  # True：z 和 x 指向同一个对象
```

判断"是不是同一个东西"用 `is`（比如 `x is None`），判断"值相不相等"用 `==`。写 `if x == None` 也能跑，但风格上 `is None` 才是正解。

## 三、数值数据：算圆周率、格式化输出、进制与字符

**用算 π 理解"数值方法"。** 圆周率算不尽，但可以用无穷级数逼近。莱布尼茨级数说：π/4 = 1 − 1/3 + 1/5 − 1/7 + …，项数越多越接近真值：

```python
def estimate_pi_leibniz(terms):
    total = 0.0
    sign = 1.0
    for k in range(terms):
        total += sign / (2 * k + 1)
        sign = -sign
    return 4 * total


print(f"1000项: {estimate_pi_leibniz(1000):.6f}")      # 3.140593
print(f"100000项: {estimate_pi_leibniz(100000):.6f}")  # 3.141583
```

注意它收敛得很慢——10 万项才 4 位有效数字。这引出数值计算的核心观念：**方法本身有效率之分**，选对方法比堆算力重要。

**蒙特卡洛：用随机解决确定性问题。** 往边长为 1 的正方形里随机撒点，落在内切四分之一圆里的点的比例 ≈ π/4。随机算法的思想在模拟、金融、物理中无处不在：

```python
import random


def estimate_pi_monte_carlo(points, seed=42):
    random.seed(seed)  # 固定种子，结果可复现
    inside = 0
    for _ in range(points):
        x, y = random.random(), random.random()
        if x * x + y * y <= 1.0:
            inside += 1
    return 4 * inside / points


print(f"200000点: {estimate_pi_monte_carlo(200_000):.4f}")
```

**数字格式化输出。** 数据算出来还要给人看，f-string 是首选：

```python
amount = 1234567.891
ratio = 0.8765
print(f"千分位: {amount:,.2f}")   # 1,234,567.89
print(f"百分比: {ratio:.1%}")     # 87.6%
print(f"十六进制: {255:#x}")      # 0xff
print(f"二进制: {255:#b}")        # 0b11111111
print(f"|{'name':<10}|{'score':>6}|")  # 左对齐 / 右对齐
```

冒号后面的格式规格是"迷你语言"，记住几个常用的（`,.2f`、`.1%`、`#x`、`<10`、`>6`）就能应付九成报表需求。

**十六进制与 ASCII 码。** 计算机底层只认二进制，十六进制是给人看的"压缩版二进制"（1 位十六进制 = 4 位二进制）。字符与数字的桥梁是编码表：

```python
print(ord("A"))          # 65：字符 -> 码点
print(chr(65))           # A：码点 -> 字符
print(hex(255))          # 0xff
print(int("ff", 16))     # 255：十六进制字符串转整数

text = "Hi, 你好"
raw = text.encode("utf-8")   # str -> bytes，存文件/发网络都用 bytes
print(raw.hex())             # 看看它真实的字节面目
print(raw.decode("utf-8"))   # bytes -> str，要配对
```

记住链条：**字符 --编码--> 字节 --解码--> 字符**，编码解码必须用同一套规则，否则就是乱码。

**进制转换实战：颜色值。** 网页里的 `#ff8800` 这类颜色，本质就是三个十六进制字节拼成的 RGB。会了 `int(x, 16)` 和格式化，互转就是几行代码：

```python
def hex_to_rgb(h):
    h = h.lstrip("#")
    return tuple(int(h[i:i + 2], 16) for i in (0, 2, 4))


def rgb_to_hex(r, g, b):
    return f"#{r:02x}{g:02x}{b:02x}"


print(hex_to_rgb("#ff8800"))      # (255, 136, 0)
print(rgb_to_hex(255, 136, 0))    # #ff8800
```

`{r:02x}` 的意思是"转十六进制，不足两位前面补 0"——做颜色、做 MAC 地址、做定宽编号都用得上。进制转换的套路都是通的：`int(s, base)` 进来，格式化字符串出去。

## 四、文件与目录操作：os.path 与目录遍历

**路径处理用 os.path，别手拼字符串。** 不同系统的分隔符不一样（`/` vs `\`），`os.path.join` 帮你屏蔽差异：

```python
import os

p = os.path.join("data", "2026", "report.txt")
print(p)                          # data/2026/report.txt
print(os.path.splitext(p))        # ('data/2026/report', '.txt') 拆后缀
print(os.path.basename(p))        # report.txt 取文件名
print(os.path.exists(p))          # 路径是否存在
print(os.path.isdir("data"))      # 是不是目录
```

**文件读写：with 是标配。** 读文件三步：打开、读取、关闭。用 `with` 包裹，块结束自动关闭，就算中途抛异常也不会泄漏：

```python
import os
import tempfile

with tempfile.TemporaryDirectory() as tmp:
    p = os.path.join(tmp, "scores.txt")
    with open(p, "w", encoding="utf-8") as fh:  # 写
        fh.write("张三 88\n李四 92\n")
    with open(p, "r", encoding="utf-8") as fh:  # 读
        lines = [line.strip().split() for line in fh if line.strip()]
    print(lines)  # [['张三', '88'], ['李四', '92']]
    print("总分:", sum(int(s) for _, s in lines))
```

`encoding="utf-8"` 每次都写上，中文不乱码。读大文件别 `read()` 一把梭，用 `for line in fh` 逐行读，内存稳如泰山。

**目录遍历：os.walk 一把梭。** 它生成 `(当前目录, 子目录列表, 文件列表)` 三元组，递归下钻全自动。下面这个"按后缀统计文件数"的小工具是练手好例子：

```python
import os
import tempfile


def count_by_suffix(root):
    stats = {}
    for dirpath, dirnames, filenames in os.walk(root):
        for name in filenames:
            suffix = os.path.splitext(name)[1].lower() or "(无后缀)"
            stats[suffix] = stats.get(suffix, 0) + 1
    return stats


# 用临时目录造一份测试数据，验证逻辑
with tempfile.TemporaryDirectory() as tmp:
    os.makedirs(os.path.join(tmp, "sub", "deep"))
    for rel in ["a.py", "b.py", "c.txt", "sub/d.py",
                "sub/e.md", "sub/deep/f.txt", "README"]:
        with open(os.path.join(tmp, rel), "w", encoding="utf-8") as fh:
            fh.write("x")
    print(count_by_suffix(tmp))
    # {'.py': 3, '.txt': 2, '.md': 1, '(无后缀)': 1}
```

自己手写一遍递归版遍历再对比 `os.walk`，对"递归"会有直观体感：递归 = 函数自己调自己，但必须有明确的终止条件，否则就是无限循环。

## 常见坑

1. **浮点数不是精确的**：`0.1 + 0.2` 不等于 `0.3`，这是二进制浮点表示的固有问题。涉及金额或精确比较时，用容差（`abs(a-b) < 1e-9`）或 `decimal` 模块。
2. **除法**：Python 3 里 `/` 永远是真除法，`7/2` 得 `3.5`；要整除用 `//`。
3. **`range` 左闭右开**：`range(5)` 是 0~4，没有 5。循环越界 bug 一半来自这里。
4. **遍历时别改结构**：`os.walk` 进行中删除/新增目录会产生意外行为；要改，先收集完路径再处理。
5. **文件要关闭**：永远用 `with open(...)`，它保证块结束自动关闭，不怕异常时泄漏。
6. **默认参数别用可变对象**：`def f(x=[])` 里的 `[]` 只创建一次，多次调用会"记住"上次的内容，用 `None` 代替。

## 练习建议

1. 分别用莱布尼茨级数和蒙特卡洛估算 π，记录"项数/点数—误差—耗时"三列数据，画个简单对比，体会收敛速度的差异。
2. 给自己写个"磁盘体检"脚本：遍历下载目录，按后缀统计文件数和总大小，用 f-string 输出对齐的表格。
3. 把上面的 `count_by_suffix` 改写成纯递归版本，不用 `os.walk`，对比两种写法的可读性。
4. 用 f-string 做一份成绩单：姓名左对齐 10 宽、分数保留 1 位小数、再加一列百分比，打印出来看看效果。

本文为学习笔记（编纂），用自己的话重写；
