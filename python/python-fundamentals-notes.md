# Python 基础笔记：命名、数据类型与异常

> 主线：变量怎么起名 → 数据长什么样（数字/字符串/序列/字典/集合）→ 输出怎么排版 → 出错了怎么办。

## 1. 变量命名规则

变量名是给数据起的代号，规则只有三条：

1. 只能含字母、数字、下划线，且**不能以数字开头**；
2. 区分大小写；
3. 不能和关键字重名（`if`、`for`、`class` 等）。

```python
user_name = "小赵"      # 下划线风格（推荐）
userName = "小赵"       # 驼峰风格（也常见，一个项目内统一即可）
# 2nd = "x"             # 非法：数字开头
# class = "x"           # 非法：关键字

import keyword
print(keyword.kwlist[:5])   # 看看都有哪些关键字：['False', 'None', 'True', 'and', 'as']
```

## 2. 三种常用数据类型

- **逻辑型** `bool`：只有 `True` / `False`，配合 `and`、`or`、`not` 做判断。
- **数值型**：`int` 整数、`float` 小数。
- **字符型** `str`：文本。

```python
is_raining = False
temperature = 26.5
city = "深圳"
print(is_raining and temperature > 30)   # False
print(not is_raining)                   # True
```

## 3. 数字的坑：整除、取余与浮点精度

```python
print(7 // 2)    # 3，整除（向下取整）
print(7 % 2)     # 1，取余
print(4.2 + 2.1) # 6.300000000000001 —— 浮点精度陷阱！
```

计算机用二进制存小数，很多十进制小数存不精确。算钱、做科学计算时用 `Decimal`：

```python
from decimal import Decimal
print(Decimal("4.2") + Decimal("2.1"))   # 6.3，精确
```

> 记住：`Decimal("4.2")` 传字符串，别传浮点数 `Decimal(4.2)`，否则精度损失已经发生了。

## 4. 序列：索引、切片与元组

列表、元组、字符串都是序列，共享一套操作：索引、切片、求长度、成员判断、遍历。

```python
s = "hello"
print(s[0], s[-1])     # h o（-1 表示最后一个）
print(s[1:4])          # ell（左闭右开）
print(len(s))          # 5
print("e" in s)        # True

nums = [10, 20, 30, 40]
print(nums[::2])       # [10, 30]，步长为 2
```

元组是**不可变**的序列，适合放"一组固定值"（如坐标、配置项）。注意单元素元组的逗号：

```python
t1 = (1,)    # 这才是单元素元组
t2 = (1)     # 这只是括号里的整数 1
print(type(t1), type(t2))

t = (5, 6, 7, 5)
print(t.index(6))        # 1，找下标
print(t.count(5))        # 2，计数
# t[0] = 9               # TypeError：元组不可改
```

元组为什么存在？三点实在的好处：可作字典的键、可一次返回多个值、不可变带来性能与安全。

```python
def min_max(data):
    return min(data), max(data)   # 一次返回两个值，打包成元组

lo, hi = min_max([3, 1, 9, 4])
print(lo, hi)   # 1 9（解包）
```

## 5. 字符串：拼接、分割与驻留

```python
a = "123"
print(a + "456")        # 123456：字符串之间可以 +
# print(a + 456)        # TypeError：字符串不能和整数直接相加

line = "apple,banana,pear"
fruits = line.split(",")          # 切成列表
print(fruits)
print(" | ".join(fruits))         # 拼回去：apple | banana | pear
```

性能提示：多次拼接用 `"".join(列表)`，别在循环里反复 `+`（每次 `+` 都产生新串，越拼越慢）。

## 6. 列表：增删改查

```python
langs = ["go", "python", "rust"]

langs.append("java")        # 末尾加
langs.insert(1, "c")        # 下标 1 处插入
print(langs.index("rust"))  # 找下标，找不到抛 ValueError
print(langs.count("go"))    # 计数

langs.remove("c")           # 按值删（只删第一个）
last = langs.pop()          # 弹出最后一个并返回
del langs[0]                # 按下标删
print(langs, "| 弹出了:", last)

nums = [3, 1, 2]
nums.sort()                 # 原地排序，返回 None
print(nums)
print(sorted([3, 1, 2], reverse=True))  # 返回新列表，原列表不动
nums.reverse()              # 原地翻转
nums.clear()                # 清空
print(nums)
```

`sort()` vs `sorted()`：前者改原列表、返回 `None`；后者返回新列表、原列表不动。别写 `x = x.sort()`。

## 7. 字典：键值对的增删改查

```python
score = {"语文": 90, "数学": 95}

score["英语"] = 88          # 新增/修改都用 []
score["数学"] = 98
print(score.pop("语文"))    # 90，删除并返回值
print(score.pop("物理", 0)) # 0，键不存在时返回默认值，不抛错
print(score)

for k, v in score.items():  # 遍历键值对
    print(k, "->", v)
print(list(score.keys()), list(score.values()))
```

空字典 `{}` 别和空集合搞混（见下节）；`popitem()` 删最后一个键值对，空字典调用抛 `KeyError`。

## 8. 格式化输出：两种写法

新式 `format()`（推荐）和旧式 `%` 都要认识，老代码里 `%` 很多：

```python
# 新式
print("{1} 年 {0} 月".format(10, 2026))     # 2026 年 10 月（位置参数可调序）
print("{:_<10}".format("左"))              # 左________  左对齐
print("{:_>10}".format("右"))              # ________右  右对齐
print("{:_^10}".format("中"))              # ____中_____  居中
print("{:.2f}".format(3.14159))            # 3.14
print("{:+d}".format(42))                 # +42，显式符号

# 旧式（% 写法）
print("%s 今年 %d 岁" % ("小王", 30))
print("%.2f" % 3.14159)                   # 3.14
print("%5.2f" % 3.14159)                  # ' 3.14'，总宽 5
```

格式符要和类型匹配：`{:d}` 只能给整数，给字符串会抛 `ValueError`。

## 9. 集合：去重与关系运算

集合三大用途：去重、成员测试、集合运算（交/并/差）。元素必须**可哈希**（数字、字符串、元组可以；列表、字典、集合不行）。

```python
print({1, 2, 2, 3})        # {1, 2, 3}，自动去重
empty = set()              # 空集合必须用 set()，{} 是空字典
print({i for i in [1, 2, 2, 3]})   # 集合推导式，同样去重

s = {1, 2, 3}
s.add(3)                   # 已存在，加了也白加
s.discard(99)              # 不存在也不报错（静默）
try:
    s.remove(99)           # 不存在抛 KeyError
except KeyError:
    print("remove 缺失元素会抛 KeyError")

a = {1, 2, 3, 4}
b = {3, 4, 5}
print(a & b)               # {3, 4} 交集
print(a | b)               # {1, 2, 3, 4, 5} 并集
print(a - b)               # {1, 2} 差集
print(a ^ b)               # {1, 2, 5} 对称差
print(a.issubset({1, 2, 3, 4, 5}))  # True，子集判断
```

注意 `==` 比的是内容相等，`is` 比的是"是不是同一个对象"；小整数和短字符串有缓存机制，别用 `is` 比值。

真值测试：`0`、`0.0`、`""`、`[]`、`{}`、`set()`、`None` 都是 `False`，其余为 `True`：

```python
print(bool(0), bool(""), bool([]), bool(None))  # False False False False
print(bool("x"), bool([0]))                     # True True（非空即真）
```

## 10. 异常：读懂报错

报错信息（traceback）**从下往上看**，最后一行是真正的异常类型和原因。常见异常速查：

```python
def demo_errors():
    cases = []
    try:
        5 / 0
    except ZeroDivisionError as e:
        cases.append(("除零", type(e).__name__))
    try:
        {}["k"]
    except KeyError as e:
        cases.append(("缺键", type(e).__name__))
    try:
        [1][9]
    except IndexError as e:
        cases.append(("越界", type(e).__name__))
    try:
        int("abc")
    except ValueError as e:
        cases.append(("转 int 失败", type(e).__name__))
    try:
        import no_such_module_xyz
    except ModuleNotFoundError as e:
        cases.append(("模块不存在", type(e).__name__))
    try:
        assert 5 > 10, "断言失败"
    except AssertionError as e:
        cases.append(("断言", type(e).__name__))
    for name, typ in cases:
        print(f"{name}: {typ}")

demo_errors()
```

异常体系的根是 `BaseException`，我们日常打交道的都是它的子类 `Exception`；自己定义异常时继承 `Exception` 即可，别去碰 `BaseException`。

调试习惯：先看最后一行定类型，再往上找自己代码的行号。`NameError` 多半是拼写错变量名，`TypeError` 多半是类型对不上（比如字符串和数字相加）。

本文为学习笔记（编纂），用自己的话重写；
