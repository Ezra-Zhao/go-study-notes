# Python 数据分析知识手册学习笔记

**本书定位一句话**：本笔记依据某 Python 数据分析手册的章节结构整理，沿着"基础语法 → Jupyter 环境 → NumPy → Pandas → 可视化 → 实战项目"这条主线，讲透数据分析师的日常工具链。

## 一、Python 基础：为数据处理打底

数据分析 80% 的时间在"整理数据"，而整理数据的核心操作就是**列表和字典的变换**。字典合并在 Python 3.9+ 有了 `|` 运算符；列表推导式配条件，一行完成"过滤+变换"。

```python
# 字典合并：3.9+ 用 |，低版本用 {**a, **b}
base = {"name": "小明", "city": "北京"}
extra = {"age": 20, "city": "上海"}
merged = base | extra
print(merged)  # 后者覆盖前者：city 变上海

# 列表推导式：清洗一列原始数据
raw = [" 12 ", "N/A", " 34", "", "56 "]
clean = [int(x.strip()) for x in raw if x.strip().isdigit()]
print(clean)  # [12, 34, 56]
```

### 内置时间模块：datetime / time / calendar

数据分析天天和日期打交道：`datetime` 做解析和运算，`time` 做计时，`calendar` 查日历。记住一个转换链：**字符串 → datetime 对象 → 时间戳/格式化字符串**，`strptime` 解析、`strftime` 格式化。

```python
from datetime import datetime, timedelta
import calendar

s = "2026-10-07 19:00:00"
dt = datetime.strptime(s, "%Y-%m-%d %H:%M:%S")  # 字符串 -> 对象
print(dt + timedelta(days=7))                    # 日期运算
print(dt.strftime("%Y年%m月%d日"))               # 对象 -> 字符串
print(calendar.month(2026, 10))                  # 打印月历
```

## 二、Jupyter Notebook 与 Anaconda：分析师的工作台

Jupyter 的价值是**"代码—结果—图表"同屏**：一段代码、一段文字说明、一张图交错出现，天然适合探索性分析。几个提效习惯：`Shift+Enter` 运行当前格并跳到下一格；变量名后加 `?` 看文档（如 `pd.read_csv?`）；`%%time` 给整格代码计时。Anaconda 则解决环境问题：`conda create -n 分析 python=3.12` 给每个项目独立的包版本，避免"在我机器上能跑"的灾难。习惯是：**一个项目一个环境，环境配置导出成 yml 随代码提交**。注意手册里提到的"指定路径安装"，本质是给 C 盘空间不足的人一条活路：`conda create --prefix D:\envs\分析` 把环境建到别的盘，重装系统也不丢。

## 三、NumPy：数值计算的底盘

NumPy 的核心洞察是：**Python 循环慢，C 循环快**。`ndarray` 把数据存成连续的 C 数组，向量化操作一次调 C 循环，百万级数据比纯 Python 快几十倍。`random` 模块做抽样和模拟；`meshgrid` 把一维坐标展成二维网格，是画等高线、曲面前提的"坐标纸"。

```python
import numpy as np

a = np.arange(12).reshape(3, 4)   # 3行4列
print(a)
print("列均值：", a.mean(axis=0))  # 按列求均值，向量化

# random：可复现的随机数（设种子）
rng = np.random.default_rng(42)
samples = rng.normal(loc=0, scale=1, size=5)
print("正态抽样：", samples.round(3))

# meshgrid：生成二维坐标网格
x = np.array([0, 1, 2])
y = np.array([0, 1])
xx, yy = np.meshgrid(x, y)
print(xx)
print("网格上 z = x + y：\n", xx + yy)
```

## 四、Pandas：表格数据的瑞士军刀

Pandas 的 `DataFrame` 就是"内存里的 Excel 表"。掌握五个高频动作，日常分析够用一大半：**日期处理**（`to_datetime` 统一解析）、**按指定顺序排序**（`Categorical` 自定义顺序）、**按条件造新列**（`np.where` / `apply`）、**分组聚合**（`groupby`）。

```python
import pandas as pd

df = pd.DataFrame({
    "date": ["2026-10-05", "2026-10-06", "2026-10-07", "2026-10-07"],
    "city": ["北京", "上海", "北京", "广州"],
    "sales": [120, 200, 150, 90],
})
df["date"] = pd.to_datetime(df["date"])          # 日期处理
df["level"] = np.where(df["sales"] >= 150, "高", "低")  # 按条件造新列

# 按指定 list 排序：城市按"北京 上海 广州"的自定义顺序
order = ["北京", "上海", "广州"]
df["city"] = pd.Categorical(df["city"], categories=order, ordered=True)
df = df.sort_values("city")
print(df[["city", "sales", "level"]])

# apply：对每行做复杂逻辑
df["desc"] = df.apply(lambda r: f'{r["city"]}{r["level"]}销量', axis=1)
print(df["desc"].tolist())

# groupby：分组聚合，一行出报表
report = df.groupby("city", observed=True)["sales"].agg(["sum", "mean"]).round(1)
print(report)

# 日期处理进阶：从日期列派生年/月/星期，透视表一眼看结构
df["month"] = df["date"].dt.month
df["weekday"] = df["date"].dt.day_name()
pivot = df.pivot_table(index="city", columns="level", values="sales",
                       aggfunc="sum", fill_value=0, observed=True)
print(pivot)
```

## 五、可视化：Matplotlib 与 Seaborn

可视化的铁律是**"一张图只讲一件事"**。Matplotlib 是底层画布，什么都能画，代码稍长；Seaborn 站在它肩上，统计图表（热力图、分布图）一行出图。分析师的常用组合：**探索阶段用 Seaborn 快速看分布，汇报阶段用 Matplotlib 精修细节**。

```python
import matplotlib
matplotlib.use("Agg")  # 无界面环境：只存图不弹窗
import matplotlib.pyplot as plt

months = ["1月", "2月", "3月", "4月", "5月", "6月"]
sales_a = [120, 135, 128, 150, 162, 170]
sales_b = [90, 95, 110, 105, 118, 125]

plt.rcParams["font.sans-serif"] = ["DejaVu Sans"]
fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(months, sales_a, marker="o", label="产品A")
ax.bar(months, sales_b, alpha=0.6, label="产品B")
ax.set_title("半年销量趋势")
ax.set_ylabel("销量")
ax.legend()
fig.savefig("/tmp/sales_trend.png", dpi=100)
print("图片已保存：/tmp/sales_trend.png")

# Seaborn 热力图：相关性矩阵一眼看关系
import seaborn as sns
corr = df[["sales"]].assign(noise=rng.normal(size=len(df))).corr()
fig2, ax2 = plt.subplots()
sns.heatmap(corr, annot=True, ax=ax2)
fig2.savefig("/tmp/corr_heatmap.png", dpi=100)
print("热力图已保存：/tmp/corr_heatmap.png")
```

Bokeh 的核心抽象是 `ColumnDataSource`：它把绘图需要的所有列数据包成一个"数据源对象"，图表只引用列名而不直接持有数据。这样做的好处是**数据与视图解耦**——同一份数据源可以同时驱动折线图、散点图和数据表，改一处数据，所有图表联动更新；配合 Bokeh 的布局组件（行列排布、选项卡），不用写前端代码就能拼出可交互的数据看板。这是它区别于 Matplotlib"一张图一套数据"的根本设计差异。

Plotly Express 的哲学是"一行代码一张好图"：长表（tidy data）输进去，`px.line(df, x="date", y="sales", color="city")` 直接出可交互的折线图，还能 `write_image` 导出 jpeg 插进报告。它的代价是包体积大、首次加载慢，适合**交付给别人看的最终图表**，而不适合探索阶段反复试错。

```python
import plotly.express as px

trend = pd.DataFrame({
    "month": ["1月", "2月", "3月", "4月"] * 2,
    "city": ["北京"] * 4 + ["上海"] * 4,
    "sales": [120, 135, 128, 150, 90, 95, 110, 105],
})
fig = px.line(trend, x="month", y="sales", color="city",
              markers=True, title="两城销量趋势（交互图）")
fig.write_html("/tmp/trend.html")
print("交互图已保存：/tmp/trend.html")
```

## 六、项目实战：分析的通用套路

手册里的实战（持仓数据、UFO 目击、世界杯、福布斯榜单）看似主题各异，套路是同一套，值得背下来：

1. **读数据**：`pd.read_csv` / `read_excel`，先 `df.head()`、`df.info()`、`df.describe()` 摸家底；
2. **洗数据**：处理缺失值（删还是填，看缺失比例）、统一日期格式、去重；
3. **看分布**：分组计数、画直方图/柱状图，找异常值；
4. **找关系**：分组聚合对比、相关性热力图；
5. **讲结论**：一张图配一句话，图越多越要克制。

```python
# 迷你实战：模拟一份"城市目击事件"数据走完全流程
events = pd.DataFrame({
    "city": ["北京", "上海", "北京", "广州", "上海", "北京", None, "广州"],
    "count": [3, 5, 2, 4, 6, 1, 9, 2],
})
print("缺失情况：\n", events.isna().sum())

cleaned = events.dropna().copy()          # 1. 删缺失
report = (cleaned.groupby("city")["count"]  # 2. 分组聚合
          .agg(total="sum", avg="mean").round(2)
          .sort_values("total", ascending=False))
print(report)
print("结论：", f"{report.index[0]}目击总量最高（{report.iloc[0]['total']}起）")
```

## 七、学习资料与进阶路线

手册附的学习资料思路是对的：**官方文档 > 经典书 > 碎片文章**。给自己的路线可以这样排：先把 Pandas 官方"10 分钟入门"跑通，再系统学《利用 Python 进行数据分析》这类实战书，碎片文章只用来查具体函数的冷门用法。**学完一个知识点，当天就用真实数据（哪怕是自己的记账本）练一遍**，否则等于没学。

## 常见坑速查

1. `df` 切片后赋值报 `SettingWithCopyWarning` → 用 `.loc` 或先 `.copy()`。
2. `read_csv` 中文乱码 → 试 `encoding="utf-8-sig"` 或 `"gbk"`。
3. 日期列是字符串就做运算 → 先 `pd.to_datetime` 转成日期类型。
4. `groupby` 后索引变层级 → 需要时 `reset_index()` 拿回普通列。
5. Jupyter 里变量越积越多结果对不上 → 定期 `Kernel → Restart & Run All` 重跑全流程验证。

## 练习建议

- 找一份真实 CSV（如个人记账、运动记录），走完"读—洗—看—找—讲"五步，输出一份一页纸报告。
- 用 `groupby` + `agg` 做三张不同维度的聚合表，体会"分组键即分析视角"。
- 同一份数据分别用 Matplotlib 和 Seaborn 画，对比代码量和效果。
- 刻意制造几个脏数据（缺失、重复、格式混乱的日期），练习"洗数据"三板斧：`isna` 定位、`dropna`/`fillna` 处理、`to_datetime` 统一。

本文为学习笔记（编纂），用自己的话重写；
