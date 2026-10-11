# Python 数据分析基础学习笔记

这份笔记是我从一本 274 页的 Python 数据分析入门教材提炼的学习大纲，用自己的话重写。覆盖的是一套完整的数据分析链路：脚本基础、文件读写、表格处理、数据库、统计建模、图表，最后到定时自动跑。每个环节都自己手写代码并在 Python 3 上跑通，数据全是自造的练习数据。

## 1. Python 基础：从脚本跑起来

先把"一个分析脚本长什么样"摸清楚。习惯上建一个 `analyze.py`，然后 `python3 analyze.py` 直接跑。脚本里最常见的就是几件事：数据结构装数据、控制流做筛选汇总、正则和日期处理脏数据、文本读写存结果。

```python
import re
from datetime import datetime

records = [
    {"name": "A", "sales": 120.5, "date": "2026-01-05"},
    {"name": "B", "sales": 89.0,  "date": "2026-01-12"},
    {"name": "A", "sales": 55.0,  "date": "2026-02-03"},
]

# 字典做分组汇总是最顺手的套路
total = {}
for r in records:
    total[r["name"]] = total.get(r["name"], 0) + r["sales"]
print(total)  # {'A': 175.5, 'B': 89.0}

# 正则：日期字符串拆成年月日
pat = re.compile(r"^(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})$")
m = pat.match("2026-03-15")
print(m.groupdict())  # {'year': '2026', 'month': '03', 'day': '15'}

# 日期解析与格式化
dt = datetime.strptime("2026-03-15 08:30", "%Y-%m-%d %H:%M")
print(dt.strftime("%Y年%m月%d日"))
```

文本文件读写用 `with open(...)` 最稳，自动关文件：

```python
with open("demo.txt", "w", encoding="utf-8") as f:
    f.write("第一行\n第二行\n第三行\n")

with open("demo.txt", encoding="utf-8") as f:
    lines = [line.strip() for line in f if line.strip()]
print(lines)
```

几点体会：列表套字典装记录行是最常用的内存结构；`dict.get(key, 0)` 做累加计数不用写 if；正则的命名分组 `(?P<name>...)` 让解析结果可读性强很多。

## 2. CSV 文件处理：pandas 读写与筛选统计

CSV 是分析里见得最多的格式。用标准库 `csv` 写文件、pandas 读和分析，配合起来最顺。注意 `open` 时加 `newline=""`，否则 Windows 下会多出空行。

```python
import csv
import pandas as pd

rows = [
    {"store": "north", "month": "01", "sales": 1200},
    {"store": "south", "month": "01", "sales": 950},
    {"store": "north", "month": "02", "sales": 1350},
    {"store": "south", "month": "02", "sales": 1010},
]
with open("sales.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.DictWriter(f, fieldnames=["store", "month", "sales"])
    w.writeheader()
    w.writerows(rows)

df = pd.read_csv("sales.csv")

# 按列筛选并求和
print(df[df["store"] == "north"]["sales"].sum())  # 2550

# 分组统计，一行顶一堆循环
print(df.groupby("store")["sales"].agg(["sum", "mean"]))
```

多个 CSV 文件合并很常见，用 `pd.concat`：

```python
df2 = pd.DataFrame({"store": ["east"], "month": ["01"], "sales": [700]})
df2.to_csv("sales2.csv", index=False)

combined = pd.concat(
    [pd.read_csv("sales.csv"), pd.read_csv("sales2.csv")],
    ignore_index=True,
)
print(len(combined))  # 5
```

记住的几个点：`groupby` + `agg` 是表格分析的半壁江山；合并多个文件前先确认列名一致，不一致先对齐；`ignore_index=True` 让合并后索引重新编号，省得后面出怪事。

## 3. Excel 文件：多工作表、多工作簿

Excel 的难点在于结构比 CSV 多一层：工作簿里有多个工作表。写可以用 openpyxl，读和分析继续用 pandas。先列出工作表名再动手，是个好习惯。

```python
import openpyxl
import pandas as pd

wb = openpyxl.Workbook()
ws1 = wb.active
ws1.title = "Q1"
ws1.append(["store", "sales"])
ws1.append(["north", 1200])
ws1.append(["south", 950])

ws2 = wb.create_sheet("Q2")
ws2.append(["store", "sales"])
ws2.append(["north", 1350])
ws2.append(["south", 1010])
wb.save("sales.xlsx")

xls = pd.ExcelFile("sales.xlsx")
print(xls.sheet_names)  # ['Q1', 'Q2']

# 逐个工作表读出来再拼接
q1 = xls.parse("Q1")
q2 = xls.parse("Q2")
allq = pd.concat([q1, q2], ignore_index=True)
print(allq["sales"].sum())  # 4510
```

多工作簿的场景就是把上面的 `pd.ExcelFile` 换成遍历目录里的一堆 xlsx，每个文件读出来 `concat`。核心思想和多 CSV 文件合并一样，区别只在多了一层工作表。

## 4. 数据库：sqlite3 实操

数据量大到内存放不下，或者要反复查，sqlite3 是零成本的选择：标准库自带，不用装服务。建表、插入、聚合查询，一个脚本全搞定。

```python
import sqlite3

conn = sqlite3.connect("demo.db")
cur = conn.cursor()
cur.execute("DROP TABLE IF EXISTS orders")
cur.execute("CREATE TABLE orders(id INTEGER PRIMARY KEY, store TEXT, amount REAL)")
cur.executemany(
    "INSERT INTO orders(store, amount) VALUES (?, ?)",
    [("north", 120.0), ("south", 95.0), ("north", 55.0), ("east", 70.0)],
)
conn.commit()

# 分组聚合：每个店的笔数和总额，按总额倒序
cur.execute(
    "SELECT store, COUNT(*), SUM(amount) FROM orders "
    "GROUP BY store ORDER BY SUM(amount) DESC"
)
for row in cur.fetchall():
    print(row)

# 带条件的聚合
cur.execute("SELECT store, AVG(amount) FROM orders WHERE amount > 60 GROUP BY store")
print(cur.fetchall())
conn.close()
```

输出里 `('north', 2, 175.0)` 排第一，因为总额最高。体会：插入一定要用 `?` 占位符（`executemany` 传元组），既防注入又快；`GROUP BY` 配合聚合函数，SQL 一句话能替代几十行 Python。

## 5. 应用：大文件查找与分类统计

这是个典型的小应用：遍历目录，把大文件揪出来。`os.walk` 递归走目录树，`os.path.getsize` 取大小，排序后取前 N 个。

```python
import os

root = "/data"
big = []
for dirpath, _, filenames in os.walk(root):
    for fn in filenames:
        p = os.path.join(dirpath, fn)
        big.append((p, os.path.getsize(p)))

big.sort(key=lambda x: x[1], reverse=True)
for path, size in big[:10]:
    print(f"{size:>12}  {path}")
```

分类统计的思路同理：按扩展名或日期分组，累加大小和个数。这个模式稍加改装就能做"某个目录下各类日志各占多少空间"之类的日常运维小工具。

## 6. 图表：matplotlib 基础

分析结果最后要落到图上。matplotlib 的基本流程：准备数据、画图、存文件。服务器上无显示器时，`matplotlib.use("Agg")` 切到无头后端，直接存 PNG。

```python
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

labels = ["north-1", "north-2", "south-1", "south-2"]
values = [1200, 1350, 950, 1010]

plt.figure()
plt.bar(labels, values)
plt.title("sales")
plt.savefig("chart.png")
```

pandas 的 DataFrame 自带 `.plot()`，内部也是调用 matplotlib，画折线图、柱状图一行代码就够。选图有个简单原则：看趋势用折线，看对比用柱状，看构成用饼图。默认字体对中文不友好，正式出图前要配中文字体。

## 7. 描述性统计与建模

这是最硬核的一章，分四步走：描述性统计先看清数据长什么样，然后用假设检验和回归回答"有没有关系、关系多强"，最后用分类和聚类做预测和分组。

先看描述性统计和 t 检验。t 检验回答的是"两组数据的均值差异是不是偶然"。比如比较两种包装下日销量有没有显著差异：

```python
import numpy as np
from scipy import stats

a = np.array([5.1, 5.4, 5.0, 5.6, 5.2])
b = np.array([6.1, 6.3, 5.9, 6.5, 6.0])

t = stats.ttest_ind(a, b)
print(round(float(t.pvalue), 5))  # 0.00036，远小于 0.05，差异显著
```

p 值小于 0.05 通常认为差异显著，这是个约定俗成的阈值，不是自然定律。

相关性看两个变量一起变化的程度：

```python
xs = np.array([1., 2, 3, 4, 5])
ys = np.array([2.1, 4.0, 5.9, 8.2, 10.1])
print(round(float(np.corrcoef(xs, ys)[0, 1]), 4))  # 0.9994，强正相关
```

最小二乘回归是把直线拟合成数据。公式是正规方程 beta = (X'X)^-1 X'y，自己手写一遍再和 `np.polyfit` 对结果，能确认理解是对的：

```python
X = np.vstack([np.ones_like(xs), xs]).T
beta = np.linalg.inv(X.T @ X) @ X.T @ ys
beta2 = np.polyfit(xs, ys, 1)
print(np.allclose(beta[::-1], beta2))  # True
```

逻辑斯蒂回归处理"是/否"这类分类问题，比如客户会不会流失。原理是把线性输出压进 sigmoid 函数变成概率，再用梯度下降优化。下面是纯 numpy 手写实现：

```python
rng = np.random.default_rng(0)
n = 200
Xl = rng.normal(0, 1, (n, 2))
true_w = np.array([0.8, -1.2])
prob = 1 / (1 + np.exp(-(Xl @ true_w - 0.2)))
yl = (rng.random(n) < prob).astype(float)

w = np.zeros(2)
b_ = 0.0
for _ in range(2000):
    z = Xl @ w + b_
    p = 1 / (1 + np.exp(-z))
    w -= 0.5 * Xl.T @ (p - yl) / n   # 权重梯度
    b_ -= 0.5 * (p - yl).mean()      # 偏置梯度

pred = ((1 / (1 + np.exp(-(Xl @ w + b_)))) > 0.5).astype(float)
print("准确率:", round(float((pred == yl).mean()), 3))  # 0.745
print("学到的权重:", np.round(w, 3))  # [ 0.929 -1.301]，接近真实 0.8/-1.2
```

聚类不需要标签，K-means 把相似的点自动分成几组。同样是 numpy 手写，迭代两步走：分配点到最近中心、按组重算中心。

```python
pts = np.vstack([rng.normal(0, 1, (60, 2)), rng.normal(6, 1, (60, 2))])
centers = pts[rng.choice(len(pts), 2, replace=False)]
for _ in range(20):
    d = ((pts[:, None, :] - centers[None, :, :]) ** 2).sum(axis=2)
    labels = d.argmin(axis=1)
    centers = np.array([pts[labels == k].mean(axis=0) for k in range(2)])
print(np.round(centers, 2))  # [[-0.02 -0.07] [ 5.99  5.9 ]]，正好落在两个数据团中心
```

这一章的经验：建模前先画图看数据，别一上来就套模型；回归看残差，分类看准确率和混淆矩阵；聚类结果要回头检查中心点有没有业务含义。

## 8. 按计划自动运行脚本：cron 示例

分析脚本写好了，最后一步是让它每天自动跑。Linux 上用 cron，每分钟粒度。编辑 crontab：

```bash
crontab -e
```

加一行，每天凌晨两点跑分析脚本，输出记日志：

```cron
0 2 * * * /usr/bin/python3 /home/user/analyze.py >> /home/user/analyze.log 2>&1
```

五个星号依次是分、时、日、月、星期。日志重定向别忘了，不然报错了你都不知道。另外一个经验：脚本里用绝对路径，别用相对路径，cron 的工作目录和你登录时不一样，相对路径最容易翻车。

## 9. 接下来可以往哪走

教材最后指了几个方向：标准库继续深挖（`itertools`、`collections`、`argparse`），PyPI 上的科学计算栈（NumPy、SciPy、Scikit-Learn），还有数据结构（栈、队列、图、树）。按我的理解，分析这条线上下一步最值得投的是 SQL 再扎实一点，加上 pandas 的时间序列处理，基本能应付日常八成的数据活。

本文为学习笔记（编纂），用自己的话重写；
