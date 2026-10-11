# Python 数据可视化学习笔记：从环境搭建到图表实战

> 参考材料：《Python 数据可视化编程实战》一书的大纲提纲。下面是按该书思路，用自己的话重新组织、代码全部自己手写的学习笔记。

---

## 1. 准备工作环境

可视化这套东西，核心就两个库：`matplotlib` 负责画图，`numpy` 负责数值计算。大部分场景装这两个就够了，科学计算再加 `scipy`。

```bash
pip install matplotlib numpy scipy pillow requests
```

几点实务经验：

- 尽量用虚拟环境，避免把全局 Python 搞乱：
  ```bash
  python3 -m venv viz-env && source viz-env/bin/activate
  ```
- 在无图形界面的服务器上跑绘图代码，画完直接存文件就行。用 Agg 后端（无头渲染），不依赖显示器：
  ```python
  import matplotlib
  matplotlib.use("Agg")
  import matplotlib.pyplot as plt
  ```
- 版本问题偶尔会坑人（比如旧版 `boxplot` 不认 `tick_labels` 参数），写代码前先 `python -c "import matplotlib; print(matplotlib.__version__)"` 看一眼。

---

## 2. 了解数据：导入、导出与清洗

画图之前先把数据弄进 Python。几种常见来源：

### 2.1 CSV

```python
import csv

with open("scores.csv") as f:
    rows = list(csv.DictReader(f))  # 每行是字典，键为表头

for r in rows:
    print(r["name"], int(r["score"]))
```

数值型数据更省事的办法是用 `numpy.loadtxt`，或者装了 pandas 直接 `read_csv`。

### 2.2 JSON

```python
import json

with open("config.json") as f:
    data = json.load(f)   # 读进来是 list/dict
```

### 2.3 Excel / 关系型数据库

Excel 可以用 `openpyxl` 读；数据库用 `sqlite3`（标准库自带）查出来再转成列表。思路都一样：取出来 → 整理成 Python 的列表/字典/numpy 数组 → 再画图。

### 2.4 异常值清理

真实数据里常混着离群点，不清掉会把整张图拉偏。四分位距（IQR）法简单有效：

```python
import numpy as np

data = np.concatenate([np.random.default_rng(5).normal(50, 5, 200), [500.0]])

q1, q3 = np.percentile(data, [25, 75])
iqr = q3 - q1
mask = (data >= q1 - 1.5 * iqr) & (data <= q3 + 1.5 * iqr)
cleaned = data[mask]   # 离群点被剔除
```

### 2.5 噪声平滑

传感器数据抖得厉害时，一个最简单的办法是移动平均（滑动窗口求均值）：

```python
noisy = np.random.default_rng(5).normal(0, 1, 100)
kernel = np.ones(5) / 5
smoothed = np.convolve(noisy, kernel, mode="same")
```

窗口越大曲线越平滑，但代价是会抹掉细节。画图前先想清楚：你是想看趋势，还是想保留尖峰。

---

## 3. 绘制并定制化图表

这是最常用的一章。记住一个总原则：先搭框架，再往上面加修饰。

### 3.1 柱状图

适合对比几个类别的数值大小：

```python
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

categories = ["A", "B", "C"]
values = [5, 7, 4]

fig, ax = plt.subplots()
ax.bar(categories, values)
fig.savefig("bar.png")
plt.close(fig)
```

堆积柱状图就是在 `bar` 里加 `bottom` 参数，把第二组数据"摞"在第一组上面，适合看总量和构成。

### 3.2 线形图

连续数据的趋势用线画。正弦余弦是最经典的练习：

```python
import numpy as np

x = np.linspace(0, 2 * np.pi, 200)

fig, ax = plt.subplots()
ax.plot(x, np.sin(x), label="sin")
ax.plot(x, np.cos(x), label="cos")
ax.legend()
fig.savefig("lines.png")
plt.close(fig)
```

### 3.3 坐标轴、刻度、网格、注解

一张图从"能用"到"好看"，全靠这些细节：

```python
x = np.arange(10)

fig, ax = plt.subplots()
ax.plot(x, x ** 2, marker="o")
ax.set_xlabel("x")
ax.set_ylabel("x^2")
ax.set_title("square curve")
ax.set_xticks([0, 2, 4, 6, 8])            # 自定义刻度位置
ax.grid(True, linestyle="--", alpha=0.5)   # 网格线，半透明虚线最耐看
ax.annotate("steep rise",                  # 文字注解 + 箭头
            xy=(8, 64), xytext=(5, 80),
            arrowprops=dict(arrowstyle="->"))
fig.savefig("custom.png")
plt.close(fig)
```

记住一个小诀窍：网格用浅色虚线，千万别抢数据的风头。

### 3.4 散点图

两个变量之间的关系（有没有相关性、聚没聚类），看散点图最直观。还能用点的大小和颜色再编码两个维度：

```python
rng = np.random.default_rng(7)
x = rng.normal(size=60)
y = rng.normal(size=60)
sizes = rng.uniform(20, 200, size=60)
colors = rng.uniform(0, 1, size=60)

fig, ax = plt.subplots()
sc = ax.scatter(x, y, s=sizes, c=colors, cmap="viridis", alpha=0.7)
fig.colorbar(sc)   # 颜色条，解释颜色代表什么
fig.savefig("scatter.png")
plt.close(fig)
```

### 3.5 直方图

看数据分布（正态还是偏态、有没有双峰）用直方图：

```python
data = np.random.default_rng(3).normal(loc=100, scale=15, size=2000)

fig, ax = plt.subplots()
ax.hist(data, bins=40, edgecolor="black")
ax.axvline(data.mean(), color="red", linestyle="--")  # 标出均值
fig.savefig("hist.png")
plt.close(fig)
```

`bins` 个数多试几个：太少会糊掉细节，太多全是毛刺。

### 3.6 误差条

做实验的人绕不开。误差条把"不确定度"画在图上，比光给数字诚实得多：

```python
x = np.arange(5)
y = np.array([2.0, 3.5, 3.0, 4.5, 5.0])
err = np.array([0.3, 0.5, 0.4, 0.6, 0.5])

fig, ax = plt.subplots()
ax.errorbar(x, y, yerr=err, fmt="-o", capsize=4)
fig.savefig("errorbar.png")
plt.close(fig)
```

### 3.7 饼图

占比结构图。注意两点：类别超过 6 个就别用饼图了；小扇区标签会挤成一团，考虑把小的合并成"其他"：

```python
sizes = [30, 25, 20, 15, 10]
labels = ["alpha", "beta", "gamma", "delta", "epsilon"]

fig, ax = plt.subplots()
ax.pie(sizes, labels=labels, autopct="%1.1f%%", startangle=90)
ax.axis("equal")   # 必须加，否则饼会被拉成椭圆
fig.savefig("pie.png")
plt.close(fig)
```

---

## 4. 多子图与高级图表

### 4.1 subplots：一张画布多张图

对比几张相关图表时，把它们拼在一张图里最方便：

```python
x = np.linspace(0, 10, 100)

fig, axes = plt.subplots(2, 2, figsize=(8, 6))  # 2 行 2 列
axes[0, 0].plot(x, np.sin(x)); axes[0, 0].set_title("sin")
axes[0, 1].plot(x, np.cos(x)); axes[0, 1].set_title("cos")
axes[1, 0].bar(["x", "y"], [1, 2])
axes[1, 1].hist(x, bins=20)
fig.tight_layout()  # 自动调间距，防止标题重叠
fig.savefig("subplots.png")
plt.close(fig)
```

`tight_layout()` 是救命的：子图多了标题和坐标轴标签很容易互相打架。

### 4.2 等高线图

二维标量场（比如地形高度、温度分布）的经典画法：

```python
x = np.linspace(-3, 3, 100)
y = np.linspace(-3, 3, 100)
X, Y = np.meshgrid(x, y)          # 生成网格点
Z = np.sin(X) * np.cos(Y)         # 每个网格点上的值

fig, ax = plt.subplots()
cs = ax.contour(X, Y, Z, levels=12)
ax.clabel(cs, inline=True, fontsize=8)  # 线上标注数值
fig.savefig("contour.png")
plt.close(fig)
```

### 4.3 极坐标图

方向性数据（风向、雷达扫描）用极坐标画：

```python
theta = np.linspace(0, 2 * np.pi, 200)
r = 1 + 0.3 * np.sin(5 * theta)   # 五瓣花形

fig = plt.figure()
ax = fig.add_subplot(111, projection="polar")
ax.plot(theta, r)
fig.savefig("polar.png")
plt.close(fig)
```

### 4.4 文件系统树可视化（概念）

大纲里提到用图表展示文件系统树。思路很简单：把"目录结构"看成树形数据，每个节点的大小对应磁盘占用，然后用树状图或旭日图画出来。实现上一般配合 `os.walk` 扫描目录、统计大小，再交给专门的树状图库绘制。matplotlib 本身没有现成的 treemap，知道这个思路就行。

---

## 5. 3D 可视化入门

matplotlib 的 3D 功能靠 `mpl_toolkits.mplot3d` 提供，画简单的 3D 图足够，复杂渲染还是得找专业工具（如 OpenGL 相关的库）：

```python
from mpl_toolkits.mplot3d import Axes3D  # noqa: F401  (注册 3d 投影)

fig = plt.figure()
ax = fig.add_subplot(111, projection="3d")

xs = np.arange(4); ys = np.arange(3)
xpos, ypos = np.meshgrid(xs, ys)
xpos = xpos.flatten(); ypos = ypos.flatten()
zpos = np.zeros_like(xpos)
dx = dy = 0.5 * np.ones_like(zpos)
dz = np.linspace(1, 12, len(xpos))

ax.bar3d(xpos, ypos, zpos, dx, dy, dz, shade=True)
fig.savefig("bar3d.png")
plt.close(fig)
```

3D 图的一个忠告：好看但难读。汇报材料里能不用 3D 就不用，2D 的柱状图配分组颜色，传达的信息只多不少。动画的原理也类似：循环里不断更新数据、每帧存一张图，最后用 ffmpeg 拼起来。

---

## 6. 图像与地图图表（概念）

大纲提到了几个方向，这里记概念和思路，不做完整代码演示：

- **PIL（Pillow）**：Python 的图像处理库。读图、裁剪、缩放、加水印都可以。它和 matplotlib 配合：PIL 负责把图像读成数组，`ax.imshow()` 负责显示。
- **地图**：把数据点画在地图底图上。常用做法是找一个底图瓦片或矢量边界，然后把经纬度坐标映射上去。注意坐标系转换是坑点最多的地方，先确认你的经纬度是 WGS84 还是别的。
- **验证码（CAPTCHA）**：验证码图像的生成和识别，本质上是"用代码画图 + 用图像处理读图"，属于图像处理的趣味应用，和数据可视化关系不大，了解即可。

---

## 7. 用正确的图表理解数据

选错图表比不画图更糟——它会误导。这一节记几个关键概念：

### 7.1 对数图

数据横跨好几个数量级（比如 1 到 100000），普通坐标轴会把小数值压成一条线。对数坐标把每个数量级拉平：

```python
x = np.logspace(0, 3, 100)   # 10 的 0 次方到 10 的 3 次方

fig, ax = plt.subplots()
ax.loglog(x, x ** 2, base=10)
ax.set_xlabel("log x")
ax.set_ylabel("log y")
fig.savefig("loglog.png")
plt.close(fig)
```

幂律关系（y 等于 x 的平方这种）在双对数坐标下会变成直线——这是判断"数据是不是幂律分布"的土办法。

### 7.2 箱线图

一张图看五个统计量：最小值、下四分位、中位数、上四分位、最大值，外加离群点。对比几组数据的分布差异时非常好用：

```python
groups = [np.random.default_rng(9).normal(loc=m, scale=1.5, size=200)
          for m in (0, 2, 4)]

fig, ax = plt.subplots()
ax.boxplot(groups, showmeans=True)
ax.set_xticklabels(["g1", "g2", "g3"])
fig.savefig("boxplot.png")
plt.close(fig)
```

### 7.3 颜色表（colormap）

用颜色表示数值时，选色是门学问。两个原则：

- 连续数据用顺序色（sequential），比如 viridis，从深到浅单调变化；
- 围绕零点的正负数据用发散色（diverging），比如蓝-白-红。

千万别用彩虹色（jet）表示连续数据——人眼对彩虹色的亮度变化不敏感，会产生虚假的"断层"。

```python
x = np.linspace(0, 1, 50)
fig, ax = plt.subplots()
for i, name in enumerate(["viridis", "plasma", "cividis"]):
    ax.plot(x, x + i * 0.4, color=plt.get_cmap(name)(0.6), label=name)
ax.legend()
fig.savefig("colormap.png")
plt.close(fig)
```

### 7.4 互相关与自相关（概念 + 演示）

- **自相关**：信号和它自己延迟后的版本比，找出周期性。比如一个正弦信号，它的自相关函数在延迟一个周期处会出现峰值：

  ```python
  x = np.linspace(0, 4 * np.pi, 400)
  sig = np.sin(x)
  ac = np.correlate(sig - sig.mean(), sig - sig.mean(), mode="full")
  ac = ac[ac.size // 2:] / ac.max()
  lags = np.arange(len(ac))

  fig, ax = plt.subplots()
  ax.stem(lags[::10], ac[::10], basefmt=" ")
  fig.savefig("autocorr.png")
  plt.close(fig)
  ```

- **互相关**：两个不同信号比，找出它们的延迟/相似程度。`np.correlate` 同样适用。
- **谱图 / 火柴杆图 / 矢量场**：谱图是把信号的频率成分随时间展开（音频分析常用）；火柴杆图（stem 图）适合画离散序列；矢量场用箭头表示每个点的方向和大小（流体、电磁场）。这些都是"数据是什么形态就用什么图"的实例。

### 7.5 甘特图（概念）

项目管理里画任务时间线的图。matplotlib 没有现成函数，一般用 `ax.barh` 横向条形图模拟：y 轴是任务名，x 轴是时间，条的长度就是工期。

---

## 8. pyplot 与 OO 接口的区别

这是 matplotlib 里最容易让新手迷糊的一点，其实很简单：

- **pyplot 接口**（`plt.plot(...)`）：Matlab 风格，函数式写法。背后维护了一个"当前图形/当前坐标轴"的全局状态，你调用的函数都作用在这个隐式对象上。写小脚本、交互式探索时最顺手。
- **OO 接口**（`fig, ax = plt.subplots(); ax.plot(...)`）：显式拿到 Figure 和 Axes 对象，方法都挂在对象上。画多子图、写库代码、做复杂定制时必须用它，否则全局状态会把你绕晕。

两者完全等价，下面这两种写法画出来一模一样：

```python
# pyplot 风格
import matplotlib.pyplot as p
x = np.arange(5)
p.figure(); p.plot(x, x); p.close("all")

# OO 风格
fig, ax = plt.subplots()
ax.plot(x, x)
plt.close(fig)
```

建议：入门时两种都认识，正式写代码一律用 OO 风格。养成好习惯，后面画复杂图时少踩很多坑。

---

## 速查：画图前问自己四个问题

1. 数据是什么类型？（类别 / 连续 / 分布 / 占比 / 方向）
2. 想传达什么信息？（对比 / 趋势 / 相关 / 分布）
3. 观众是谁？（自己探索用糙一点没关系，给人看就把标题、标签、图例写全）
4. 有没有离群点或量级跨度大的数据？（先清洗，必要时用对数轴）

本文为学习笔记（编纂），用自己的话重写；
