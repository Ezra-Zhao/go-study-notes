# Python 基础学习笔记

这份笔记整理的是 Python 入门阶段最常用的核心主题：语言介绍、运算符（重点是位运算）、字符串（创建、驻留机制、格式化）、类与对象（实例化流程、浅拷贝、动态绑定）、os.path 常用方法，以及一个完整可运行的学生信息管理系统（增删改查）。全部内容用我自己的话重新表述，代码都是亲手写的、亲手跑通的。

## 一、Python 介绍

Python 是一种高级、解释型、面向对象的编程语言。说它"高级"，是指它封装了很多底层细节，写起来接近自然语言；说它"解释型"，是指代码不需要先编译成机器码再运行，解释器逐行读取逐行执行——这意味着开发速度快、调试方便，但执行速度一般不如编译型语言。

Python 的特点可以概括成三句：语法简洁（用缩进代替大括号）、类型动态（变量不用声明类型，随时可以换）、生态庞大（标准库全，第三方包多）。

装好 Python 后，验证环境只要一行命令：

```python
import sys
print(sys.version)
```

第一个程序几乎都是这句：

```python
print("Hello, Python")
```

解释器执行 print 时，会先求值括号里的表达式，再把结果转成文本输出到控制台。新手注意两点：Python 3 的 print 是函数，括号不能省略；缩进是语法的一部分，同一层级的语句缩进必须一致。

## 二、运算符

Python 的运算符分六大家族：

- 算术：`+ - * /`（普通除法）、`//`（整除）、`%`（取余）、`**`（幂）
- 比较：`== != > < >= <=`，返回 True 或 False，注意 `==` 是比较，`=` 是赋值
- 逻辑：`and or not`，短路特性很有用——`and` 前面为假就不再往后看
- 赋值：`= += -= *= /= //= %= **=`
- 成员：`in` / `not in`，比如 `"py" in "python"` 返回 True
- 身份：`is` / `is not`，比较的是"是不是同一个对象"，不是"值相不相等"

几个容易踩坑的地方：

```python
print(7 / 2)    # 3.5，普通除法永远返回浮点数
print(7 // 2)   # 3，整除向下取整
print(-7 // 2)  # -4，注意负数整除是向负无穷方向取整
print(2 ** 3)   # 8
```

### 位运算

位运算直接对数字的二进制位动手脚，是理解计算机底层的入门钥匙。六个运算符：

- `&` 按位与：两位都为 1 结果才为 1
- `|` 按位或：有一位为 1 结果就为 1
- `^` 按位异或：两位不同结果为 1，相同为 0
- `~` 按位取反：0 变 1、1 变 0，结果为 `-(x+1)`
- `<<` 左移：二进制左移 n 位，相当于乘以 2 的 n 次方
- `>>` 右移：二进制右移 n 位，相当于整除以 2 的 n 次方

拿 12 和 10 举例，12 的二进制是 00001100，10 的是 00001010：

```python
a, b = 12, 10
print(bin(a), bin(b))  # 0b1100 0b1010
print(a & b)  # 8，  只有"两个人都是 1"的位保留下来
print(a | b)  # 14， 只要有一方是 1 就保留
print(a ^ b)  # 6，  两位不一样才为 1
print(~a)     # -13， 取反后是 -(a+1)
print(a << 2) # 48，  左移 2 位等于 12 * 4
print(a >> 2) # 3，   右移 2 位等于 12 // 4
```

这段输出我逐项跑过验证。位运算的实际用途很广：权限掩码（读=4、写=2、执行=1，用 `|` 组合、用 `&` 检查）、快速乘除 2 的幂、`a ^ a == 0` 的异或消元小技巧等。

## 三、字符串

### 创建字符串的四种写法

```python
s1 = '单引号'
s2 = "双引号"
s3 = '''三引号，可以写
多行文本'''
s4 = str(123)  # 用构造函数转换
print(type(s4), s4)  # <class 'str'> 123
```

字符串是不可变对象——一旦创建，内容就不能就地修改。所有"修改"操作（拼接、替换、大小写转换）返回的都是新字符串，旧的那个不会变：

```python
s = "python"
print(id(s))
s = s + "!"
print(id(s))  # id 变了，说明是两个不同的对象
```

### 驻留机制（interning）

驻留是解释器的一种内存优化：对于满足条件的短字符串，只保留一份副本，之后所有内容相同的字面量都指向同一个对象，所以用 `is` 比较会返回 True。但它是优化手段，不是语言规范保证的行为，不能依赖它做相等判断——判断值相等永远用 `==`。

我实测的结果：

```python
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True，字面量相同，被驻留复用

s3 = "".join(["py", "thon"])
s4 = "python"
print(s3 is s4)  # False，运行时拼接产生的是新对象
print(s3 == s4)  # True，值相等
```

记住结论：`is` 问"是不是同一个对象"，`==` 问"值等不等"。业务代码里比相等永远用 `==`。

### 格式化字符串的三种方式

```python
name, score = "小李", 92

# 1. f-string（Python 3.6+，最推荐，读写都快）
print(f"姓名：{name}，成绩：{score}")

# 2. str.format()
print("姓名：{}，成绩：{}".format(name, score))
print("姓名：{0}，成绩：{1}".format(name, score))

# 3. 百分号占位（老式写法）
print("姓名：%s，成绩：%d" % (name, score))
```

三行输出完全一致，我都跑过。日常开发优先用 f-string，它还支持在花括号里直接写表达式：`f"{score + 8}"`。

## 四、类与对象

### 类与实例的基本写法

```python
class Student:
    def __init__(self, name, score):
        self.name = name
        self.score = score

    def introduce(self):
        return f"我是{self.name}，成绩{self.score}"

s = Student("小明", 88)
print(s.introduce())
```

`self` 是实例自己的引用，方法通过它访问实例属性；`Student("小明", 88)` 这句"调用类"，背后分两步走。

### __new__ 与 __init__：创建对象的两个阶段

- `__new__` 是静态方法，负责分配内存、返回一个"空壳"实例
- `__init__` 是实例方法，负责给这个空壳装上属性、做初始化；它没有返回值

流程图写成代码就是这样：

```python
class Student:
    def __new__(cls, name):
        print("第 1 步：__new__ 分配内存，返回实例")
        return super().__new__(cls)

    def __init__(self, name):
        print("第 2 步：__init__ 初始化属性")
        self.name = name

s = Student("小明")
print("实例拿到名字：", s.name)
```

实际运行输出：

```
第 1 步：__new__ 分配内存，返回实例
第 2 步：__init__ 初始化属性
实例拿到名字： 小明
```

注意执行顺序：先 `__new__`，后 `__init__`。绝大多数情况下我们只写 `__init__` 就够了；只有在想控制"实例到底怎么被造出来"（比如单例模式、不可变类型的子类化）时才重写 `__new__`。

### 动态绑定属性和方法

Python 的对象是"开放"的：实例创建之后，还可以随时给它加属性、加方法。

```python
class Person:
    def __init__(self, name):
        self.name = name

p = Person("张三")

# 1. 动态绑定属性：类定义里没写 age，实例照样能加上
p.age = 20
print(p.age)  # 20

# 2. 动态绑定方法：把函数挂到类上，所有实例都能用
def run(self):
    return self.name + " 在跑步"
Person.run = run
print(p.run())  # 张三 在跑步

# 3. 也可以删除
del p.age
print(hasattr(p, "age"))  # False
```

灵活是把双刃剑：动态加属性写原型很快，但大型项目里会让"这个对象到底有哪些属性"变得不可追踪。如果想锁住属性，可以在类里定义 `__slots__`。

### 类的赋值与浅拷贝

"赋值"和"拷贝"是两码事：

```python
import copy

class Score:
    def __init__(self, items):
        self.items = items  # items 是一个列表

stu = Score([85, 90, 78])

# 赋值：alias 和 stu 指向同一个对象，改一个两边都变
alias = stu

# 浅拷贝：新对象，但内部的可变成员仍是共享引用
shallow = copy.copy(stu)
print(shallow.items is stu.items)  # True，列表是同一份
shallow.items.append(100)
print(stu.items)  # [85, 90, 78, 100]，原对象也被影响了
```

实测输出证实：浅拷贝只复制了最外层，里面的列表还是共用同一份。想要彻底独立的两份，就用 `copy.deepcopy()`，它会递归复制所有嵌套的可变对象。

什么时候用哪个？赋值——只是想多一个名字指同一个东西；浅拷贝——想要一个"长得一样但顶层独立"的副本；深拷贝——想要完全独立的克隆。涉及嵌套列表、字典的对象，浅拷贝之后修改内部元素会互相影响，这是最常见的 bug 来源。

## 五、os.path 模块常用方法

`os.path` 是处理文件路径的工具箱。注意：`os.path.join` 生成的路径风格随操作系统变化，在 Windows 上是反斜杠，在 Linux/macOS 上是正斜杠，所以拼接路径永远用 `join`，不要手写斜杠。

```python
import os.path as osp

print(osp.join("data", "files", "info.txt"))
# data/files/info.txt（Linux 下）

print(osp.abspath("info.txt"))
# /home/hatch/info.txt，把相对路径转成绝对路径

print(osp.basename("/home/user/info.txt"))  # info.txt，取文件名
print(osp.dirname("/home/user/info.txt"))   # /home/user，取目录部分
print(osp.splitext("archive.tar.gz"))       # ('archive.tar', '.gz')，分离扩展名

print(osp.exists("/tmp"))    # True，路径是否存在
print(osp.isfile("/tmp"))    # False，是不是文件
print(osp.isdir("/tmp"))     # True，是不是目录
```

每个方法我都实际调用验证过。实战组合很常用：先 `os.path.exists` 判断文件在不在，再决定是读取还是新建；用 `splitext` 判断文件类型；用 `join` + `abspath` 拼出跨平台的路径。

## 六、实战：学生信息管理系统（增删改查）

前面都是散点知识，这个小项目把它们串起来：用字典存一条学生记录（学号、姓名、成绩），用列表存全班，通过命令行菜单完成增删改查。

设计思路：

- 增（Create）：先查学号是否已存在，存在就拒绝，避免重复
- 删（Delete）：按学号找到记录后移除；找不到就提示
- 改（Update）：按学号定位，允许只改姓名、只改成绩或都改（回车跳过）
- 查（Read）：按学号精确查找 + 显示全部名单

完整代码（已在 Python 3 下全流程跑通：添加两人、重复学号被拒、查找、修改成绩、删除一人、删除不存在的学号被提示）：

```python
students = []  # 每条记录：{"id": 学号, "name": 姓名, "score": 成绩}

def add_student(sid, name, score):
    for s in students:
        if s["id"] == sid:
            return False, f"学号 {sid} 已存在，添加失败"
    students.append({"id": sid, "name": name, "score": score})
    return True, f"已添加：{sid} {name} {score}"

def delete_student(sid):
    for s in students:
        if s["id"] == sid:
            students.remove(s)
            return True, f"已删除学号 {sid}"
    return False, f"学号 {sid} 不存在，删除失败"

def update_student(sid, name=None, score=None):
    for s in students:
        if s["id"] == sid:
            if name is not None:
                s["name"] = name
            if score is not None:
                s["score"] = score
            return True, f"已更新学号 {sid}"
    return False, f"学号 {sid} 不存在，更新失败"

def find_student(sid):
    for s in students:
        if s["id"] == sid:
            return s
    return None

def show_all():
    if not students:
        print("暂无学生记录")
        return
    print("学号\t姓名\t成绩")
    for s in students:
        print(f"{s['id']}\t{s['name']}\t{s['score']}")

def menu():
    print("=" * 30)
    print("学生信息管理系统")
    print("1.添加学生  2.删除学生  3.修改学生")
    print("4.查找学生  5.显示全部  0.退出")
    print("=" * 30)

def main():
    while True:
        menu()
        choice = input("请选择操作：").strip()
        if choice == "0":
            print("再见！")
            break
        elif choice == "1":
            ok, msg = add_student(input("学号：").strip(),
                                  input("姓名：").strip(),
                                  int(input("成绩：").strip()))
            print(msg)
        elif choice == "2":
            ok, msg = delete_student(input("要删除的学号：").strip())
            print(msg)
        elif choice == "3":
            sid = input("要修改的学号：").strip()
            name = input("新姓名（回车跳过）：").strip() or None
            raw = input("新成绩（回车跳过）：").strip()
            score = int(raw) if raw else None
            ok, msg = update_student(sid, name, score)
            print(msg)
        elif choice == "4":
            s = find_student(input("要查找的学号：").strip())
            print(f"查到：{s}" if s else "查无此人")
        elif choice == "5":
            show_all()
        else:
            print("无效选项，请重新输入")

if __name__ == "__main__":
    main()
```

实测运行记录（节选）：

```
请选择操作：1
学号：1001
姓名：张三
成绩：88
已添加：1001 张三 88

请选择操作：1
学号：1001
姓名：王五
成绩：90
学号 1001 已存在，添加失败
```

这个版本数据只存在内存里，程序退出就清空。进阶方向很自然：把 `students` 列表存成 JSON 或 CSV 文件（配合 `os.path` 判断文件是否存在），程序启动时加载、退出时保存，就成了带持久化的小应用；再往后可以把函数封装成类，加上异常处理（比如成绩输入非数字时的 `try/except`）。

## 七、实操案例解题思路

大纲里还列了若干实操案例，主题只有编号没有具体内容。这里记录几条通用的、能反复用的解题套路，都是从上面的知识点里长出来的：

1. **先拆再写**：任何案例先把"输入是什么、输出是什么、中间要几步"写在纸上，再动手。学生管理系统就是先定好记录结构（字典）、容器（列表）、操作（四个函数），代码自然成型。

2. **循环 + 条件是万能钥匙**：查找、统计、过滤类题目，九成是"for 循环遍历 + if 判断"的组合。先写出遍历骨架，再填判断条件，最后处理结果。

3. **字符串处理三板斧**：`split()` 切分、`join()` 拼接、`strip()` 去空白。文本类案例（统计词频、解析日志行、整理名单）基本靠这三招开局。

4. **字典是最顺手的小型数据库**：需要"按名字找东西"时，别用两个平行列表（一个存名字一个存值），直接用字典，`d[key]` 一步到位，还天然去重。

5. **路径操作用 os.path，不用字符串拼接**：凡是涉及读写文件的案例，先 `join` 拼路径、`exists` 判存在，再 `open`。这样代码换操作系统也能跑。

6. **报错信息是免费的老师**：`NameError` 多半是变量名写错或没定义；`TypeError` 常是类型混用（比如字符串和数字相加）；`IndentationError` 就是缩进乱了。读懂报错的前两行，八成问题当场解决。

本文为学习笔记（编纂），用自己的话重写；
