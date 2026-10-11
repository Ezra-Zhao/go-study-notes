# Python 数据科学工具栈学习笔记

这份笔记整理的是 Python 数据科学常用的五个工具/领域：交互式环境、数组计算、表格数据处理、绘图、机器学习。内容按「用什么、为什么、怎么用」三层组织，代码均为自己手写并在本地实际运行验证过。

---

## 一、IPython：超越普通 Python 的交互环境

IPython 是一个增强型的 Python 交互式 shell，Jupyter Notebook 的内核就基于它。它适合边写边试、边看结果边调整的探索式工作方式。

常用的几个能力：

- **即时帮助**：在对象后面加问号可以查看文档说明，例如 `len?` 会显示内置函数 len 的用法简介；加两个问号可以看到更多细节。
- **Tab 补全**：输入对象名加点再按 Tab，会列出它可用的属性和方法，记不住函数名时很好用。
- **魔术命令**：以 `%` 开头的行内命令和以 `%%` 开头的整段命令，是 IPython 特有的效率工具。
- **性能分析**：内置的计时工具能快速比较不同写法的速度，对优化循环很有帮助。

以下为概念示例（需在 IPython 或 Jupyter 里运行，普通 python3 解释器不支持魔术命令）：

```python
# %timeit：对一条语句计时，多次运行取平均
# %timeit sum(range(1000))

# %%timeit：对整个代码块计时
# %%timeit
# total = 0
# for i in range(1000):
#     total += i

# %run：直接运行一个外部脚本文件
# %run my_script.py

# %matplotlib inline：让画图结果直接显示在 Notebook 里
# %matplotlib inline
```

理解要点：交互环境的核心价值是「短反馈循环」——写一行、看一行结果，数据探索的效率远高于反复改脚本再全量运行。

---

## 二、NumPy：数组计算的基础

NumPy 的核心是多维数组对象 `ndarray`。数据科学里的大部分数值运算都建立在它之上，特点是整块数据一起算（向量化），不用写显式循环，速度快、写法简洁。

### 2.1 创建数组

```python
import numpy as np

# 一维数组
a = np.array([3, 1, 4, 1, 5])

# 二维数组：2 行 3 列
m = np.array([[1, 2, 3],
              [4, 5, 6]])
print(m.shape)   # (2, 3)，查看数组形状
print(m[0, 2])   # 取第 1 行第 3 列的元素，结果是 3

# 常用快捷创建方式
print(np.zeros(3))        # 全零
print(np.ones((2, 2)))    # 全一
print(np.arange(4))       # 等差序列 0,1,2,3
```

### 2.2 数组运算与广播

数组和标量做运算时，会自动作用到每个元素上，这叫向量化。形状不同的数组相遇时，NumPy 会按规则自动「拉伸」较小的那一方，这叫广播。

```python
import numpy as np

a = np.array([3, 1, 4, 1, 5])
print(a * 2)    # [ 6  2  8  2 10]，每个元素乘 2
print(a + 10)   # [13 11 14 11 15]，每个元素加 10

# 广播示例：一行向量加一列向量，自动扩展成 3x4 的加法表
row = np.arange(4)                 # [0, 1, 2, 3]
col = np.arange(3).reshape(3, 1)   # 变成 3 行 1 列
print(row + col)
# [[0 1 2 3]
#  [1 2 3 4]
#  [2 3 4 5]]
```

理解要点：广播规则是「从右往左对齐维度，缺失或为 1 的维度可以扩展」。写代码时心里先想好结果的形状，广播才不会出错。

### 2.3 聚合、布尔索引与排序

```python
import numpy as np

rng = np.random.default_rng(42)
x = rng.normal(0, 1, size=1000)   # 1000 个标准正态随机数
print(x.mean(), x.std())         # 均值约 -0.03，标准差约 0.99
print((x > 2).sum())             # 大于 2 的元素个数，约 19 个

# 布尔索引：用条件直接筛元素
print(x[x > 2][:5])              # 取出大于 2 的前 5 个

# 排序
print(np.sort(np.array([5, 2, 9, 1])))       # [1 2 5 9]
print(np.sort(np.array([5, 2, 9, 1]))[:3])   # 取最小的 3 个：[1 2 5]
```

理解要点：`ndarray` 是同类型数据的紧凑存储，这是它快的原因；凡是能用整块运算表达的逻辑，就不要写 Python 层面的 for 循环。

---

## 三、Pandas：表格数据的处理利器

Pandas 建立在 NumPy 之上，专门处理带标签的表格数据。两个核心对象：

- **Series**：一维带索引的数组，像一列数据。
- **DataFrame**：二维带行列标签的表格，像一张 Excel 表。

### 3.1 Series 与基本取值

```python
import pandas as pd

s = pd.Series([10, 20, 30], index=['a', 'b', 'c'])
print(s['b'])        # 20，按标签取值
print(s[s > 15])     # 布尔过滤，留下 b 和 c 两行
```

### 3.2 DataFrame 的分组与透视

分组聚合是数据分析里最常用的操作：先按某一列分组，再对每组做统计。透视表则是把分组结果摊平成行列交叉的表格。

```python
import pandas as pd

df = pd.DataFrame({
    'city':  ['北京', '上海', '北京', '上海', '北京'],
    'month': [1, 1, 2, 2, 3],
    'sales': [120, 150, 130, 160, 140],
})

# 按城市分组，求每城平均销售额
print(df.groupby('city')['sales'].mean())
# 北京 130.0，上海 155.0

# 透视表：行=城市，列=月份，值=销售额求和
print(df.pivot_table(index='city', columns='month',
                     values='sales', aggfunc='sum'))
```

### 3.3 表的合并

```python
import pandas as pd

left = pd.DataFrame({'id': [1, 2, 3], 'name': ['张', '李', '王']})
right = pd.DataFrame({'id': [2, 3, 4], 'score': [88, 92, 75]})

# 按 id 内连接：只保留两边都有的 id（2 和 3）
print(left.merge(right, on='id', how='inner'))
```

理解要点：`merge` 的 `how` 参数决定保留哪些行——inner 取交集、left 保留左表全部、outer 取并集。做之前先想清楚「丢行」是否可接受。

### 3.4 时间序列

Pandas 对日期索引有专门支持：重采样（按周/月聚合）和滑动窗口（移动平均）是看趋势的两把刷子。

```python
import pandas as pd
import numpy as np

idx = pd.date_range('2026-01-01', periods=60, freq='D')
ts = pd.Series(np.random.default_rng(7).normal(0, 1, 60).cumsum(),
               index=idx)

print(ts.resample('W').mean().head(3))   # 按周重采样求均值
print(ts.rolling(7).mean().tail(3))     # 7 天滑动平均，看趋势更平滑
```

---

## 四、Matplotlib：把数据画出来

Matplotlib 有两种接口：

- **MATLAB 风格接口**（`plt.plot` 等直接调用）：写起来快，适合简单快速出图。
- **面向对象接口**（先拿 `Figure` 和 `Axes` 对象再操作）：显式、可控，多子图和复杂排版时更清晰，推荐日常使用。

### 4.1 折线图与图例

```python
import matplotlib
matplotlib.use('Agg')   # 无显示环境时用 Agg 后端，只存文件不弹窗
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 2 * np.pi, 100)
fig, ax = plt.subplots()          # 创建画布和坐标轴
ax.plot(x, np.sin(x), label='sin')
ax.plot(x, np.cos(x), label='cos')
ax.legend()                       # 显示图例
ax.set_xlabel('x')
ax.set_ylabel('y')
fig.savefig('/tmp/sine.png')      # 保存成图片文件
```

### 4.2 散点图、直方图与多子图

```python
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import numpy as np

rng = np.random.default_rng(0)
fig, axes = plt.subplots(1, 2)    # 一行两列两个子图

axes[0].scatter(rng.normal(0, 1, 200), rng.normal(0, 1, 200), s=8)
axes[0].set_title('scatter')      # 散点图看两个变量的关系

axes[1].hist(rng.normal(0, 1, 1000), bins=30)
axes[1].set_title('hist')         # 直方图看单个变量的分布

fig.tight_layout()                # 自动调整子图间距防重叠
fig.savefig('/tmp/scatter_hist.png')
```

理解要点：画图前先问「我想看什么」——看趋势用折线、看关系用散点、看分布用直方图；一张图只讲一件事，图例和坐标轴标签不要省。

---

## 五、机器学习入门

机器学习的标准流程：准备数据 → 划分训练集/测试集 → 选模型训练 → 评估。评估时不能只在训练集上看好坏，要用交叉验证估计模型在新数据上的表现，防止过拟合（模型把训练数据的噪声也背下来了）。

### 5.1 分类算法对比：朴素贝叶斯、SVM、决策树、随机森林

用经典的鸢尾花数据集演示，四种分类器在同一划分下训练和评估：

```python
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.naive_bayes import GaussianNB
from sklearn.svm import SVC
from sklearn.tree import DecisionTreeClassifier
from sklearn.ensemble import RandomForestClassifier

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.25, random_state=0)

for name, model in [
    ('GaussianNB', GaussianNB()),
    ('SVC', SVC()),
    ('DecisionTree', DecisionTreeClassifier(random_state=0)),
    ('RandomForest', RandomForestClassifier(n_estimators=50, random_state=0)),
]:
    model.fit(X_train, y_train)
    scores = cross_val_score(model, X_train, y_train, cv=5)
    print(name,
          '测试集准确率 =', round(model.score(X_test, y_test), 3),
          '五折交叉验证 =', round(scores.mean(), 3))
```

本地运行结果（四舍五入到三位小数）：

- GaussianNB：测试集 1.0，交叉验证 0.956
- SVC：测试集 0.974，交叉验证 0.964
- DecisionTree：测试集 0.974，交叉验证 0.964
- RandomForest：测试集 0.974，交叉验证 0.955

几点理解：

- **朴素贝叶斯**：基于贝叶斯定理，假设特征之间相互独立，训练极快，适合文本分类这类高维稀疏数据。
- **SVM（支持向量机）**：找一条间隔最大的分界线，配合核函数能处理非线性边界，小样本上表现稳。
- **决策树**：一连串「是/否」判断，可解释性强，但单棵树容易过拟合。
- **随机森林**：很多棵决策树投票，方差小、更稳，是日常任务里省心的默认选择之一。

### 5.2 线性回归

回归预测的是连续数值。线性回归假设目标和特征成直线关系，用最小二乘法求出最佳的斜率和截距。

```python
from sklearn.linear_model import LinearRegression
import numpy as np

rng = np.random.default_rng(1)
xr = rng.uniform(0, 10, 200).reshape(-1, 1)
yr = 3 * xr.ravel() + 5 + rng.normal(0, 1, 200)  # 真实关系：y = 3x + 5 加噪声

lr = LinearRegression().fit(xr, yr)
print('斜率 =', round(float(lr.coef_[0]), 3))       # 约 2.986
print('截距 =', round(float(lr.intercept_), 3))     # 约 4.981
```

拟合出的斜率和截距接近真实值 3 和 5，说明模型把线性关系找了回来。理解要点：线性回归简单但假设很强，数据不是直线关系时要先做特征变换或换模型。

### 5.3 无监督学习：PCA 降维与 k-means 聚类

无监督学习没有标签可用。PCA 把高维数据压缩到低维、尽量保留信息；k-means 把数据分成 k 个簇，簇内尽量紧凑。

```python
from sklearn.datasets import load_iris
from sklearn.decomposition import PCA
from sklearn.cluster import KMeans
import numpy as np

X, y = load_iris(return_X_y=True)

X2 = PCA(n_components=2, random_state=0).fit_transform(X)  # 4 维降到 2 维
labels = KMeans(n_clusters=3, n_init=10, random_state=0).fit_predict(X2)
print('三个簇的大小：', np.bincount(labels).tolist())  # [61, 50, 39]
```

理解要点：k-means 的 k 要自己定，选 k 前可以多试几个值看簇内误差下降的拐点；PCA 降维后画二维散点图，是检查聚类效果最直观的办法。

### 5.4 几个延伸概念

- **特征工程**：模型上线前最花功夫的一步——把原始字段加工成模型好用的特征（缺失值填充、类别编码、数值缩放、从日期里拆出星期几）。特征做得好不好，常常比换模型更影响结果。
- **高斯混合模型（GMM）**：k-means 的"软"版本。它假设每个簇是一个高斯分布，数据点以一定概率属于每个簇，而不是非此即彼。簇的形状是椭圆形、边界模糊时，GMM 比 k-means 更合适。
- **核密度估计（KDE）**：不假设分布形状，直接用数据点"堆"出一条平滑的密度曲线。画分布、找双峰时比直方图更细腻，代价是带宽参数要调。
- **流形学习**：PCA 是线性的，数据如果躺在一个"卷起来"的曲面上（比如瑞士卷形状），PCA 会把它压坏。流形学习（如 t-SNE、UMAP）保留局部邻域关系，把这种弯曲结构展开到低维，常用于高维数据的可视化探索。

---

本文为学习笔记（编纂），用自己的话重写；
