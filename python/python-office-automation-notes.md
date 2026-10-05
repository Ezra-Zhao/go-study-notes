# Python 办公自动化笔记：Excel / Word / PPT / 邮件

> 思路：把重复的点鼠标变成可复用的脚本。四件套——openpyxl 管 Excel，python-docx 管 Word，python-pptx 管 PPT，yagmail 管发邮件。

```bash
pip install openpyxl python-docx python-pptx yagmail keyring schedule
```

> 说明：原文档中有一短章标题未能辨认，此处略去，不编造。

## 1. openpyxl：读写 Excel

openpyxl 只认新格式：`.xlsx`、`.xlsm`、`.xltx`、`.xltm`。老式的 `.xls` 它打不开。

先备一张测试表（下面代码自己造，不依赖外部文件）：

```python
from openpyxl import Workbook, load_workbook

# --- 造一张测试表 ---
wb = Workbook()
ws = wb.active
ws.title = "成绩"
ws.append(["姓名", "语文", "数学"])
ws.append(["小赵", 90, 95])
ws.append(["小钱", 88, 92])
wb.save("/tmp/demo.xlsx")

# --- 重新打开并读取 ---
from openpyxl import load_workbook
wb2 = load_workbook(filename="/tmp/demo.xlsx")
print("所有表名：", wb2.sheetnames)
sheet = wb2["成绩"]                    # 按名字取表
print("数据范围：", sheet.dimensions)   # 如 A1:C3
print("活动表：", wb2.active.title)

cell = sheet["A1"]                     # 按坐标取格子
print("A1 的值：", cell.value)
for row in sheet.iter_rows(values_only=True):
    print(row)
```

几个核心概念：

- **行**用 1 基数字（第 1 行），**列**用字母（A 列），交叉点就是**格子（cell）**，一堆格子组成**表（sheet）**。
- `cell.value` 读写格子内容；`sheet["B2"] = 100` 直接写。

### 样式：边框与填充

```python
from openpyxl import load_workbook
from openpyxl.styles import Border, Side, PatternFill

wb = load_workbook("/tmp/demo.xlsx")
ws = wb["成绩"]

thin = Side(style="thin", color="000000")          # 细边框
red_thick = Side(style="thick", color="FF0000")    # 红色粗边框
ws["A1"].border = Border(left=thin, right=thin, top=red_thick, bottom=thin)

ws["A1"].fill = PatternFill(fill_type="solid", fgColor="FFFF00")  # 黄底
wb.save("/tmp/demo_styled.xlsx")
print("样式已写入 /tmp/demo_styled.xlsx")
```

`fill_type="solid"` 是纯色填充；想要渐变可以用 `GradientFill`，传起始和终止颜色。日常报表里，给表头加粗边框 + 浅色底纹，表格立刻像样。

## 2. python-docx：生成 Word

```python
from docx import Document
from docx.shared import Pt

doc = Document()
doc.add_heading("月度工作汇报", level=1)          # 一级标题
doc.add_heading("一、本月完成", level=2)          # 二级标题
doc.add_paragraph("完成了 3 个自动化脚本，节省约 6 小时人工。")

p = doc.add_paragraph()
run = p.add_run("重点：")
run.bold = True                                  # 加粗
run.font.size = Pt(14)                           # 字号
p.add_run("下月计划接入邮件自动发送。")

doc.add_paragraph("待办事项", style="List Bullet")  # 项目符号列表
doc.save("/tmp/report.docx")
print("已生成 /tmp/report.docx")
```

要点：`Document()` 新建（也可传路径打开已有文档），`add_heading` 加标题，`add_paragraph` 加段落，`add_run` 在段落里加"文本片段"以便单独设样式，最后 `save()` 落盘。

## 3. python-pptx：生成 PPT

```python
from pptx import Presentation
from pptx.util import Inches, Pt

prs = Presentation()                       # 默认 16:9
slide = prs.slides.add_slide(prs.slide_layouts[5])  # 空白版式

# 标题文本框
box = slide.shapes.add_textbox(Inches(0.5), Inches(0.3), Inches(9), Inches(1))
box.text_frame.text = "季度数据总览"
box.text_frame.paragraphs[0].runs[0].font.size = Pt(32)

# 表格：3 行 4 列
table_shape = slide.shapes.add_table(3, 4, Inches(0.5), Inches(1.5), Inches(9), Inches(3))
table = table_shape.table
headers = ["指标", "Q1", "Q2", "Q3"]
data = [["营收(万)", 120, 150, 180], ["成本(万)", 80, 85, 90]]
for j, h in enumerate(headers):
    table.cell(0, j).text = h
for i, row in enumerate(data, start=1):
    for j, val in enumerate(row):
        table.cell(i, j).text = str(val)
table.columns[0].width = Inches(2.5)

prs.save("/tmp/deck.pptx")
print("已生成 /tmp/deck.pptx")
```

要点：

- `slide_layouts[i]` 是版式模板（0 通常是标题页，5/6 常为空白），`add_slide` 新建一页。
- 想看某版式有哪些占位符，遍历 `slide.placeholders`，看 `placeholder_format.type`（如 TITLE、BODY、PICTURE）。
- `add_table(行, 列, 左, 上, 宽, 高)` 加表格，再逐格填 `cell.text`；`add_textbox` 加自由文本框，用 `text_frame` 调对齐、换行和填充色。

## 4. 自动收发邮件：yagmail + keyring + schedule

发邮件最省事的是 `yagmail`（对 SMTP 的极简封装）。**密码千万别写死在代码里**，用 `keyring` 走系统密钥环：

```python
import yagmail

# 首次使用：把密码存进系统密钥环（只需做一次，交互式输入）
# import keyring
# keyring.set_password("my-mail", "zhaoguangyi", "你的授权码")

# 发送时从密钥环取（下面用占位演示构造，不真正发送）
import keyring
try:
    pwd = keyring.get_password("my-mail", "zhaoguangyi")
    print("从密钥环取到密码：", "是" if pwd else "否（需先 set_password）")
except Exception as e:
    print("keyring 不可用：", e)

# yagmail.SMTP("你的邮箱", pwd)   # 构造发送器
# yag.send("收件人@example.com", "主题", "正文")          # 发送
# yag.send("收件人@example.com", "带附件", "见附件", attachments="/tmp/report.docx")
print("yagmail 构造与发送写法如上，需真实邮箱授权码才能实际发出")
```

> 注意：多数邮箱（163、QQ、Gmail）发信要用"授权码"而非登录密码，去邮箱设置里单独申请。

定时执行用 `schedule`，写法接近自然语言：

```python
import schedule

jobs_run = []

def morning_report():
    jobs_run.append("已执行晨报任务")
    print("执行一次晨报任务")

schedule.every().day.at("08:30").do(morning_report)  # 每天 08:30
schedule.every().monday.do(morning_report)            # 每周一

print("已注册任务数：", len(schedule.jobs))
schedule.run_all()          # 立刻把所有任务跑一遍（用于测试）
print(jobs_run)

# 正式用法是死循环：while True: schedule.run_pending(); time.sleep(60)
```

把四件套串起来，就是一条完整的办公流水线：定时拉数据 → 写 Excel → 生成 Word 汇报 → 转成 PPT → 邮件发出，全程无人值守。

本文为学习笔记（编纂），用自己的话重写；
