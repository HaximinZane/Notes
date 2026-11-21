# Python `csv` 库超详细教程：从一行文本，到可靠的表格读写、校验和批量处理

## 零基础先导：CSV 不是“用逗号随便切一下”的字符串

CSV 是 Comma-Separated Values（逗号分隔值）这一类文本表格格式。它适合不同软件交换**行列数据**，例如学生名单、实验数据、导出的交易记录。`csv` 则是 Python 自带的读写工具，不是 Excel 软件，也不是能自动识别所有业务类型的数据库。

假设文件内容为：

```csv
name,note
Alice,"喜欢 Python, C++"
Bob,"第二行"
```

第二行的备注里有逗号；如果直接 `line.split(",")`，会把一个字段切成两个。专业解析器会考虑引号和转义规则。

### 最小读法，逐行看懂

```python
import csv

with open("people.csv", "r", encoding="utf-8", newline="") as f:
    reader = csv.DictReader(f)       # 第一行作为字段名
    for row in reader:              # 一行对应一个字典
        print(row["name"], row["note"])
```

`with open` 负责打开/关闭文件；`csv.DictReader` 负责按 CSV 规则解析；`for` 逐行取得字典。`newline=""` 有助于让 csv 模块自己正确处理行结束符（尤其是包含字段内换行或 Windows 写文件时）。**CSV 本身没有 Python int 类型**：即使文件里是 `95`，读入后通常仍是字符串，需要按业务需要 `int(row["score"])` 并处理转换错误。

写文件则先思考：列名是什么、字段顺序是什么、编码是什么，之后再学 `DictWriter`。不要先把数百行“数据清洗完整工程”当成 `csv` 的基本语法。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

这一篇讲 Python 标准库中的 `csv`。

我们从一个非常常见的问题开始：老师给你一份成绩表，后缀是 `.csv`，你希望读取每名学生的姓名、学号和成绩，再计算平均分，生成一份新表。

第一眼看，CSV 似乎只是“每行几个字符串，中间用逗号隔开”。

但当姓名中有逗号、备注里有换行、学号以零开头，或者 Windows 上写出的文件莫名出现空白行时，简单字符串分割就不够用了。

本文按照“看到一个现象，提出一个问题，运行一个小例子，再解释每一步”的顺序展开。

常规代码面向 Python 3.11 及以上；少数新版本功能会单独标注。

所有创建、覆盖和清理操作都应在专门练习目录或临时目录中执行。

## 第一章：CSV 究竟是什么？为什么还需要专门的库？

### 1.1 从一份最简单的成绩表看起

假设文本文件内容是：

```csv
student_id,name,score
001,小林,95
002,小周,88
003,小陈,100
```

第一行描述每一列的名称。

后面的每一行描述一名学生。

逗号用来区分字段，换行通常用来区分记录。

CSV 来自 Comma-Separated Values，可以理解为“逗号分隔的值”。

它适合在电子表格、数据库和程序之间交换简单的二维表格。

CSV 文件本质上通常是文本，而不是 Excel 工作簿。

`.xlsx` 有更复杂的文件结构、工作表和单元格类型；`.csv` 通常只描述一张表里的文本字段。

### 1.2 那我直接 `split(",")` 不就可以了吗？

如果数据永远不包含特殊字符，当然能分开最简单的行。

```python
line = "001,小林,95"
print(line.split(","))
```

输出：

```text
['001', '小林', '95']
```

但现在，备注字段本身也包含逗号：

```csv
001,小林,"喜欢 Python,也喜欢数据库",95
```

我们希望得到四个字段。

手工分割却会把备注内部的逗号也当成分隔符。

这不是再加一次 `strip()` 就能修复的问题，因为你需要识别哪些逗号属于字段内容，哪些逗号属于表格结构。

CSV 用引号等规则解决这种歧义。

`csv` 模块负责按这些规则解析和生成文本，你负责解释字段的业务含义。

### 1.3 为什么不同 CSV 还可能有不同规则？

有些工具用逗号，有些工具用分号，有些导出制表符分隔文本。

引号、转义、行结束形式等细节也可能不同。

Python 把一组相关格式约定称为 dialect，也就是“方言”。

这里的“方言”不是另一门编程语言，而是一份 CSV 格式配置。

同一套解析器根据配置处理不同来源，避免你为每个工具重写一个分割器。[官方背景说明](https://docs.python.org/3.14/library/csv.html)

### 1.4 `csv` 是标准库，不需要安装

```python
import csv
```

直接导入即可。

不要执行 `pip install csv` 来寻找一个名字相同的第三方包。

如果你把自己的练习脚本也命名为 `csv.py`，导入时可能遮蔽标准库。

遇到奇怪的成员缺失，可以检查：

```python
import csv
print(csv.__file__)
```

脚本文件建议命名为 `read_students.py` 或 `csv_practice.py`。

## 第二章：先搭好练习环境，避免文件状态影响观察

### 2.1 文件放在哪里？

假设项目结构是：

```text
csv_practice/
├── main.py
├── input/
│   └── students.csv
└── output/
```

你准备从 `input/students.csv` 读取，向 `output/report.csv` 写入。

如果终端工作目录就在 `csv_practice`，普通相对路径可以这样用：

```python
from pathlib import Path

input_path = Path("input") / "students.csv"
output_path = Path("output") / "report.csv"
```

### 2.2 换一个目录启动，为什么就找不到文件？

你的脚本可能位于：

```text
D:\project\csv_practice\main.py
```

而终端位于：

```text
C:\Users\Alice
```

然后运行：

```powershell
python D:\project\csv_practice\main.py
```

脚本里的 `Path("input/students.csv")` 通常仍按终端启动时的工作目录解释。

它可能寻找：

```text
C:\Users\Alice\input\students.csv
```

如果你想固定读取普通脚本旁边的资源，可以明确以脚本目录为基准：

```python
from pathlib import Path

base_dir = Path(__file__).resolve().parent
input_path = base_dir / "input" / "students.csv"
```

交互式解释器和部分 Notebook 没有 `__file__`，这类环境应明确指定练习根目录。

### 2.3 先由程序生成一份已知输入

为了排除编辑器编码和复制文本的影响，先用标准库写一份小表：

```python
import csv
from pathlib import Path

root = Path("csv_practice")
input_path = root / "input" / "students.csv"
input_path.parent.mkdir(parents=True, exist_ok=True)

with input_path.open("w", encoding="utf-8", newline="") as target:
    writer = csv.writer(target)
    writer.writerow(["student_id", "name", "score"])
    writer.writerow(["001", "小林", "95"])
    writer.writerow(["002", "小周", "88"])
    writer.writerow(["003", "小陈", "100"])
```

这里的写入会覆盖同名练习文件。

后面读取示例默认使用这份文件，除非特别说明换了输入。

如果你希望拒绝覆盖，可以把 `"w"` 改为 `"x"`，目标已存在时就会报错。

### 2.4 为什么每个例子都强调输入状态？

文件处理具有持续状态。

第一次运行创建的目录，第二次运行仍在。

第一次写出的内容，第二次可能被追加或覆盖。

如果你删除某行代码后程序仍然成功，可能只是上一次运行留下了条件。

练习时要知道自己使用的是哪份输入、是否已经存在输出，以及当前代码是否会修改它。

## 第三章：第一个 `csv.reader()`，每一层都是什么？

### 3.1 最小读取程序

```python
import csv
from pathlib import Path

path = Path("csv_practice/input/students.csv")

with path.open("r", encoding="utf-8", newline="") as source:
    reader = csv.reader(source)
    for row in reader:
        print(row)
```

通常输出：

```text
['student_id', 'name', 'score']
['001', '小林', '95']
['002', '小周', '88']
['003', '小陈', '100']
```

### 3.2 `open()` 已经读文件了，为什么还要 `reader()`？

`open()` 创建可读取文本的流。

它知道怎样从文件中取得字符，但不知道哪些逗号被引号保护，也不知道字段如何组织。

`csv.reader(source)` 接收这个文本流，按 CSV 规则产生一条条记录。

你可以理解成两个相互配合的层次：

```text
磁盘字节
    │ 文本文件流：解码、换行接口
    ▼
字符输入
    │ CSV 解析器：分隔符、引号、记录边界
    ▼
一条记录的字段列表
```

### 3.3 `reader` 是列表吗？

不是。

它是可以迭代的读取器，随着遍历不断取得下一条记录。

```python
reader = csv.reader(source)
```

这行并不等于立刻把全部文件读入一个列表。

如果写：

```python
rows = list(reader)
```

才会消费剩余记录并保存成列表。

小文件这样做方便，大文件则要考虑内存。

### 3.4 每个 `row` 是什么？

默认情况下，一条记录对应一个字段列表。

```python
['001', '小林', '95']
```

你可以像普通列表一样访问：

```python
student_id = row[0]
name = row[1]
score_text = row[2]
```

这些索引操作属于 Python 列表语法。

CSV 读取器负责把字段分出来，并不负责告诉你第三列一定是分数。

### 3.5 为什么 `95` 读出来仍是字符串？

因为 CSV 通常没有明确的列类型声明。

同样的 `001`，可能是编号，也可能被人理解为整数一。

读取器如果擅自把所有看起来像数字的文本转成数值，就可能丢掉有意义的前导零。

因此，默认返回字符串，让你按照业务规则选择转换。

```python
score = int(score_text)
```

而学号继续保留为：

```python
student_id = "001"
```

### 3.6 第一行会自动当标题跳过吗？

普通 `reader()` 不会。

它按相同规则返回第一条记录，所以你看到标题也出现在输出中。

如果格式约定首行是标题，可以明确消费一次：

```python
with path.open(encoding="utf-8", newline="") as source:
    reader = csv.reader(source)
    header = next(reader, None)

    print("标题：", header)
    for row in reader:
        print("数据：", row)
```

`next(reader, None)` 在没有记录时返回默认值 `None`，避免空文件直接触发 `StopIteration`。

但拿到标题后，还应检查它是否符合预期，不能无条件丢掉一个可能本来就是数据的首行。

## 第四章：为什么 CSV 打开时总写 `newline=""`？

### 4.1 先区分文件换行与字段内部换行

CSV 一条记录通常写在一行上。

但如果字段内容本来包含换行，它可以放在引号保护的字段内部。

例如：

```csv
id,note
1,"第一行备注
第二行备注"
```

从物理文件看，这里不止两行文本。

从 CSV 记录看，仍然是一条标题记录和一条数据记录。

因此，CSV 解析器需要自己识别哪些换行属于字段内容，哪些表示记录结束。

### 4.2 普通文本流可能转换换行

文本 `open()` 默认的 `newline=None` 会进行通用换行处理。

Windows 文本写入时，`\n` 还可能被转成系统常用行分隔形式。

CSV writer 本身也负责生成记录结束符。

如果两个层次都处理换行，就可能产生重复转换或让字段内部换行解释不符合预期。

### 4.3 `newline=""` 把相关控制交给 CSV 工具

常见正确组合：

```python
with open("table.csv", "r", encoding="utf-8", newline="") as source:
    reader = csv.reader(source)
```

写入也一样：

```python
with open("table_out.csv", "w", encoding="utf-8", newline="") as target:
    writer = csv.writer(target)
```

这里 `newline=""` 不表示文件不能有换行。

它表示文本层不额外执行相应换行替换，让 CSV 模块按格式管理边界。[官方打开方式说明](https://docs.python.org/3.14/library/csv.html#csv.reader)

### 4.4 我在 Linux 不写也没出错，是不是可以省略？

简单单行字段的数据，在某些环境下确实看不出差别。

但这只说明当前样本没有暴露边界情况。

代码以后可能在 Windows 上运行，也可能遇到带换行的字段。

按照库的推荐方式打开，能把意图写清楚，减少环境依赖。

### 4.5 想让输出统一使用 LF 呢？

文本层保持 `newline=""`，然后明确设置 writer 的记录结束符：

```python
import csv

with open("lf_table.csv", "w", encoding="utf-8", newline="") as target:
    writer = csv.writer(target, lineterminator="\n")
    writer.writerow(["name", "score"])
    writer.writerow(["小林", 95])
```

这样由 CSV writer 决定记录结束形式。

`lineterminator` 主要控制写入，不能直接当成 reader 的“只按这个字符串分行”开关。

读取器对换行识别有自己的规则。

## 第五章：编码、中文、BOM 与 Excel

### 5.1 `csv` 不会自动知道文件编码

`csv.reader()` 接收的是文本输入。

把字节转成字符串的编码，由你提供的文本流决定。

```python
with open("students.csv", encoding="utf-8", newline="") as source:
    reader = csv.reader(source)
```

这表示输入内容应按 UTF-8 解码。

它并不表示所有后缀为 `.csv` 的文件天然都是 UTF-8。

### 5.2 从旧工具导出的中文文件怎么办？

如果生成方明确使用 GBK，可以：

```python
with open("legacy_students.csv", encoding="gbk", newline="") as source:
    reader = csv.reader(source)
    for row in reader:
        print(row)
```

不要遇到解码错误就随意加 `errors="ignore"`。

那可能让姓名、标识或备注中的部分字节悄悄丢失。

应先确认来源编码，再决定是否需要有记录的容错处理。

### 5.3 第一列名字前面为什么多了 `\ufeff`？

某些 UTF-8 文件开头带有 BOM。

如果按普通 UTF-8 读成文本，开头标记可能保留为 `\ufeff`。

于是标题可能是：

```python
["\ufeffstudent_id", "name", "score"]
```

你再按 `row["student_id"]` 查字典，就可能找不到键。

来源是这种文件时，可以使用：

```python
with open("from_excel.csv", encoding="utf-8-sig", newline="") as source:
    reader = csv.DictReader(source)
```

`utf-8-sig` 读取时会处理开头 UTF-8 BOM，没有 BOM 的普通 UTF-8 内容通常也能读取。

### 5.4 写给 Excel，要不要一律使用 `utf-8-sig`？

部分 Excel 版本或打开方式更容易识别带 BOM 的 UTF-8，所以这种写法常见：

```python
with open("for_excel.csv", "w", encoding="utf-8-sig", newline="") as target:
    writer = csv.writer(target)
    writer.writerow(["姓名", "学号"])
    writer.writerow(["小林", "001"])
```

但它不是所有数据交换场景的唯一正确编码。

程序间协议若约定普通 UTF-8，应遵守协议。

对接人类表格工具时，应在目标软件和实际导入方式中验证。

### 5.5 编码正确，学号为什么还是变成了 `1`？

因为显示工具可能自动推断单元格类型。

CSV 里保存的是文本字段 `001`，Excel 直接打开时可能把它识别成数字一。

给字段加引号也不保证所有表格工具都禁用类型推断。

更可靠的做法是通过导入向导指定该列为文本，或在需要单元格类型控制时使用真正的工作簿格式。

这属于消费工具的解释行为，不能让 CSV writer 单独保证。

## 第六章：引号、逗号和转义，逐个看实际结果

### 6.1 字段包含逗号时会怎样？

```python
import csv
from io import StringIO

with StringIO(newline="") as buffer:
    writer = csv.writer(buffer, lineterminator="\n")
    writer.writerow(["001", "小林", "喜欢 Python,也喜欢数据库"])
    text = buffer.getvalue()

print(text, end="")
```

通常输出：

```csv
001,小林,"喜欢 Python,也喜欢数据库"
```

引号告诉读取器：里面的逗号属于字段内容。

你不需要先自己给字符串加双引号，再交给 writer。

### 6.2 字段本身有双引号呢？

```python
row = ["001", '他说："今天学习 CSV"']
```

常见默认规则会在被引号包裹的字段中，把内部引号写成两个引号。

文本看起来可能是：

```csv
001,"他说：""今天学习 CSV"""
```

解析回来仍然是原先一个普通双引号字符。

这属于 CSV 转义规则，与 Python 源代码里反斜杠转义是不同层次。

### 6.3 我手工先 `.replace('"', '""')` 可以吗？

通常不应在使用正常 writer 之前再手工做一次 CSV 转义。

writer 会根据格式规则处理字段。

你先改内容，它可能再转义一次，结果就多出不属于原数据的引号。

正确流程是把原始字段值交给 writer，让它负责输出格式。

### 6.4 字段里有换行呢？

```python
import csv
from io import StringIO

row = ["001", "第一行备注\n第二行备注"]

with StringIO(newline="") as buffer:
    writer = csv.writer(buffer, lineterminator="\n")
    writer.writerow(row)
    buffer.seek(0)
    restored = next(csv.reader(buffer))

print(restored == row)
print(repr(restored[1]))
```

输出：

```text
True
'第一行备注\n第二行备注'
```

一条逻辑记录可以跨多个物理文本行。

因此，不能先 `read().splitlines()`，再把每个物理行独立当成一条 CSV 记录解析。

### 6.5 如果我要观察真实输出字节呢？

先写文件，再用二进制方式读取：

```python
with open("quoted.csv", "rb") as source:
    payload = source.read()

print(repr(payload))
```

这能看见真实 `\r\n` 或 `\n` 字节，也能帮助区分“编辑器显示方式”和“文件里实际保存了什么”。

但不要把 CSV 解析器直接交给二进制流，它需要字符串输入。

## 第七章：`DictReader`——能不能按列名访问？

### 7.1 不想反复记 `row[0]`、`row[1]` 怎么办？

表格列多时，数字索引很容易混淆。

使用 `DictReader`：

```python
import csv

with open("csv_practice/input/students.csv", encoding="utf-8", newline="") as source:
    reader = csv.DictReader(source)

    for row in reader:
        print(row["student_id"], row["name"], row["score"])
```

通常输出：

```text
001 小林 95
002 小周 88
003 小陈 100
```

每条记录变成按标题名称组织的字典。

现代 Python 的 `DictReader` 返回普通 `dict`，不要依赖历史版本曾使用的其他具体映射类型。

### 7.2 第一行去哪了？

没有主动提供 `fieldnames` 时，`DictReader` 把第一条记录作为字段名称。

它通常不会把这条标题再作为数据返回。

因此，与普通 `reader()` 相比，循环中少了一条标题记录。

### 7.3 怎么查看解析出的列名？

```python
with open("csv_practice/input/students.csv", encoding="utf-8", newline="") as source:
    reader = csv.DictReader(source)
    print(reader.fieldnames)
```

结果类似：

```text
['student_id', 'name', 'score']
```

访问这个属性可能触发读取标题，所以它不是你手工写入的一份永远独立于流位置的配置。

读取器仍然关联同一个输入流，离开文件上下文后不能指望继续正常取得新记录。

### 7.4 文件没有标题，能不能自己指定？

可以。

假设输入是：

```csv
001,小林,95
002,小周,88
```

那么：

```python
import csv
from io import StringIO

text = "001,小林,95\n002,小周,88\n"

with StringIO(text, newline="") as source:
    reader = csv.DictReader(source, fieldnames=["student_id", "name", "score"])
    for row in reader:
        print(row)
```

你提供字段名后，第一条记录就按数据处理。

如果文件本来有标题，你又手工提供同样字段名，原标题会变成一条普通数据记录。

所以要先知道来源有没有标题，不能一律“为了保险”重复传一次。

### 7.5 一行比标题多字段，会发生什么？

假设标题三列，数据四列：

```csv
id,name,score
001,小林,95,extra
```

默认 `DictReader` 会把多余字段放在 `restkey` 指定的键下面；默认键是 `None`。

可以明确命名：

```python
reader = csv.DictReader(source, restkey="_extra_fields")
```

于是检查：

```python
if row.get("_extra_fields") is not None:
    raise ValueError("这一条记录的字段数量超出标题定义")
```

`restkey` 是收纳机制，不等于自动修复了格式不匹配。

### 7.6 一行少字段呢？

缺少部分字段时，使用 `restval` 填充，默认是 `None`。

注意，实际空字段与缺失字段可以不同。

```csv
id,name,score
001,小林,
002,小周
```

第一条记录明确提供了第三个空字符串字段。

第二条记录只提供两列，第三列可能由 `restval` 补成 `None`。

如果你的业务需要区分“明确留空”和“这一列根本没提供”，就应保留这种差别。

### 7.7 标题重复会怎么样？

假设：

```csv
id,name,name
001,小林,小周
```

字典无法同时保留两个完全相同键的独立值。

使用 `DictReader` 时，后面的同名列可能覆盖前面映射到同名键的内容。

正式导入前应验证标题唯一：

```python
fieldnames = reader.fieldnames

if fieldnames is None:
    raise ValueError("文件为空，没有标题")
if len(fieldnames) != len(set(fieldnames)):
    raise ValueError("标题中存在重复列名")
```

### 7.8 标题前后有空格呢？

`" score"` 与 `"score"` 是不同字符串。

你可以根据契约要求精确匹配，也可以主动清理标题。

但清理前要考虑：两个原本不同的标题清理后是否发生冲突。

例如 `"name"` 与 `" name "`，清理后都变成 `"name"`。

如果选择规范化，就应在规范化之后再次检查唯一性。

## 第八章：`DictWriter`——字典怎样按固定列顺序输出？

### 8.1 最小示例

```python
import csv

records = [
    {"name": "小林", "score": 95, "student_id": "001"},
    {"name": "小周", "score": 88, "student_id": "002"},
]

with open("students_out.csv", "w", encoding="utf-8", newline="") as target:
    writer = csv.DictWriter(target, fieldnames=["student_id", "name", "score"])
    writer.writeheader()
    writer.writerows(records)
```

输出列顺序由 `fieldnames` 决定。

它不依赖每个字典碰巧以什么顺序插入键。

### 8.2 为什么 `fieldnames` 必须明确提供？

因为写入器需要知道输出表格有哪些列，以及它们的顺序。

如果不同记录拥有不同键，完全依赖第一条记录猜测，会让后面的字段容易丢失或导致布局变化。

明确字段列表也相当于明确导出契约。

### 8.3 `writeheader()` 会自动调用吗？

不会。

创建 `DictWriter` 只创建对象，不自动写任何记录。

你调用：

```python
writer.writeheader()
```

才会输出一条标题记录。

忘记这一行，文件依然可能是合法 CSV，只是没有标题。

接收方若默认用 `DictReader` 读取，就可能把第一名学生的数据误当列名。

### 8.4 某个字典缺少键怎么办？

`restval` 决定缺失项的写入值，默认空字符串。

```python
writer = csv.DictWriter(
    target,
    fieldnames=["id", "name", "note"],
    restval="",
)
```

缺少 `note` 时写空字段。

这是一种输出政策，不代表你的业务允许这个字段缺失。

必填字段应在写入前校验，不要让 writer 的补空值掩盖问题。

### 8.5 字典多出键怎么办？

默认 `extrasaction="raise"`，遇到不在 `fieldnames` 中的键会报错。

```python
writer = csv.DictWriter(target, fieldnames=["id", "name"], extrasaction="raise")
```

这通常有利于发现字段拼错或导出模型意外变化。

如果明确只导出白名单字段，可以使用 `extrasaction="ignore"`。

但它会忽略额外数据，所以应把这种选择写在导出规则里。

### 8.6 `None` 写出去是什么？

普通 CSV writer 会把 `None` 写成空字段，其他非字符串对象通常经 `str()` 转换。

因此：

```python
[None, "", 0, False]
```

写入后，前两项可能无法仅靠默认 CSV 文本区分。

`0` 和 `False` 也只是相应文本表现，不自动保留 Python 类型标签。

如果需要严格恢复空值、类型或嵌套结构，应设计明确编码约定，或选择 JSON、数据库等更适合的数据表示。

## 第九章：数据类型转换，哪些列应该转，哪些不应该？

### 9.1 学号与数量不能按外观混为一类

```text
001
```

如果它是学生编号，前导零可能属于标识。

```text
001 件
```

如果它是数量，程序可能只关心数值一。

CSV 不知道业务含义，因此转换规则应以列契约为依据。

### 9.2 整数转换应保留位置上下文

```python
try:
    score = int(row["score"])
except ValueError as error:
    raise ValueError(f"第 {record_number} 条记录的 score 不是整数") from error
```

相比让 `int()` 原始报错直接冒出去，这里告诉用户哪一条记录出错。

但请注意逻辑记录号不一定等于物理文本行号，字段内部换行会让两者不同。

### 9.3 转换成功，还要检查范围

```python
if not 0 <= score <= 100:
    raise ValueError("分数必须在 0 到 100 之间")
```

`int("1000")` 可以成功，但不符合这个练习的成绩范围。

类型转换与业务校验分别处理不同问题。

### 9.4 金额能不能直接用 float？

可以表示近似数值，但二进制浮点不精确表示所有十进制小数。

如果 CSV 列表达金额，常使用 `Decimal` 并明确小数位和舍入规则：

```python
from decimal import Decimal, InvalidOperation

try:
    amount = Decimal(row["amount"])
except InvalidOperation as error:
    raise ValueError("amount 不是合法十进制金额") from error

if not amount.is_finite():
    raise ValueError("amount 必须是有限数值")
```

`Decimal` 也能表示某些特殊值，因此有限性仍需主动检查。

### 9.5 布尔字段怎么办？

不要直接：

```python
bool("false")
```

它得到 `True`，因为非空字符串在布尔判断中为真。

可以明确定义允许文本：

```python
def parse_bool(text):
    normalized = text.strip().casefold()
    if normalized in {"true", "1", "yes"}:
        return True
    if normalized in {"false", "0", "no"}:
        return False
    raise ValueError(f"无法识别的布尔文本：{text!r}")
```

允许哪些形式是协议选择。

不要不加说明地同时接受几十种拼法，让数据错误难以发现。

### 9.6 日期字段呢？

CSV 读取后先是字符串。

如果约定 `YYYY-MM-DD`：

```python
from datetime import date

birthday = date.fromisoformat(row["birthday"])
```

无效日期会失败，例如不存在的二月三十日。

如果来源是 `11/02/2026`，还要明确月日在前还是日在前，不能凭字符串外观猜测。

## 第十章：分隔符、引号和方言参数

### 10.1 制表符分隔是不是也能用 `csv`？

可以。

```python
reader = csv.reader(source, delimiter="\t")
```

或者使用内置 `excel-tab` 方言。

常见 `.tsv` 就是制表符分隔值。

虽然模块名称包含逗号，它提供的抽象可以处理多种单字符分隔格式。

### 10.2 `delimiter` 是一个字符

```python
reader = csv.reader(source, delimiter=";")
```

表示分号分隔。

不能把 `delimiter="||"` 当成通用多字符分隔器。

那属于另一类文本格式，需要不同解析策略。

### 10.3 `quotechar`

默认通常是双引号。

```python
reader = csv.reader(source, delimiter=";", quotechar='"')
```

它决定什么字符用于保护包含特殊内容的字段。

不要因为 Python 源代码里用单引号包住 `quotechar` 的值，就误以为 CSV 也采用单引号作为字段引号。

外层是 Python 字符串语法，值本身才是 CSV 配置。

### 10.4 `doublequote` 与 `escapechar`

默认常用双写引号表达字段内部引号。

某些来源使用反斜杠转义，可以显式配置：

```python
reader = csv.reader(source, doublequote=False, escapechar="\\")
```

这必须与来源格式一致。

不能随意同时更改参数，希望解析器自动理解任何混合规则。

### 10.5 `skipinitialspace`

```python
reader = csv.reader(source, skipinitialspace=True)
```

它用于忽略分隔符后某些紧接的空格。

它不等价于对每个字段执行 `.strip()`，也不会自动清除字段末尾所有空白。

例如，引号内有意义的空格应当继续保留。

### 10.6 `strict=True` 是业务完整校验吗？

```python
reader = csv.reader(source, strict=True)
```

这会对解析器识别的某些不良 CSV 输入抛出 `csv.Error`。

它不自动检查每行列数是否与标题一致，不验证学号唯一，也不验证分数范围。

解析严格性、表格结构校验和业务约束仍然是三层工作。

## 第十一章：`quoting`，何时给字段加引号？

### 11.1 `QUOTE_MINIMAL`

常见默认策略：只在字段包含分隔符、引号或相关换行等需要保护的内容时加引号。

```python
writer = csv.writer(target, quoting=csv.QUOTE_MINIMAL)
```

它让普通数字和简单文本保持简洁，遇到特殊内容自动加保护。

### 11.2 `QUOTE_ALL`

```python
writer = csv.writer(target, quoting=csv.QUOTE_ALL)
```

所有字段都加引号。

这有助于得到统一布局，但不表示表格软件一定把所有字段按文本类型导入。

引号是 CSV 结构规则，不是跨软件的强制单元格类型声明。

### 11.3 `QUOTE_NONNUMERIC`

写入时对非数值字段加引号；读取时会把未加引号字段转换为浮点数。

```python
import csv
from io import StringIO

with StringIO('"001","小林",95\n', newline="") as source:
    row = next(csv.reader(source, quoting=csv.QUOTE_NONNUMERIC))

print(row)
```

通常得到：

```text
['001', '小林', 95.0]
```

如果未加引号的文本不是合法浮点形式，读取会失败。

所以它只适合双方明确采用这类格式的情况，不适合作为任意 CSV 的自动类型推断器。

### 11.4 `QUOTE_NONE`

不使用引号保护时，特殊字符可能需要 `escapechar`。

```python
writer = csv.writer(target, quoting=csv.QUOTE_NONE, escapechar="\\")
```

如果没有合适转义设置，写入含分隔符等内容可能抛 `csv.Error`。

接收方也必须理解这种转义约定。

### 11.5 新版本的 `QUOTE_NOTNULL` 与 `QUOTE_STRINGS`

Python 3.12 增加这两个常量，分别为区分空值、字符串与其他值提供额外策略。

它们的读写表现与普通 `QUOTE_ALL`、`QUOTE_NONNUMERIC` 有区别，使用前要核对目标解释器版本以及接收方是否采用相同约定。

特别需要注意，Python 3.12 的读取行为曾存在相关问题，Python 3.13 才修复相应识别行为。

本文通用实践采用稳定的基础策略，并通过显式字段协议处理空值，避免把新版模式当成所有环境都支持的通用 CSV 类型系统。

### 11.6 不要把所有策略当成可随意切换的美化选项

缩进 JSON 通常主要影响布局，CSV 的 quoting 策略却可能参与数值与空值解释。

修改策略后，应重新做读写往返验证，确认目标消费者理解一致。

数据格式参数是契约的一部分。

## 第十二章：注册方言，什么时候比重复传参数更合适？

### 12.1 同一格式在多个地方使用

假设某个内部系统统一输出分号、双引号、LF 记录结束符。

你可以每次创建 writer 都写一遍参数。

```python
writer = csv.writer(target, delimiter=";", quotechar='"', lineterminator="\n")
```

如果多个模块共享这套约定，可以给它取一个明确名字。

```python
import csv

csv.register_dialect(
    "school_export",
    delimiter=";",
    quotechar='"',
    quoting=csv.QUOTE_MINIMAL,
    lineterminator="\n",
)
```

然后使用：

```python
writer = csv.writer(target, dialect="school_export")
```

这让调用位置表达“按学校导出格式写”，而不是到处重复几个缺少背景的字符参数。

### 12.2 方言名称会永久写进系统吗？

不会。

注册影响当前 Python 进程中的 CSV 方言注册表。

程序重新启动后，要由代码再次注册。

也不会自动让其他电脑的软件认识这个名字。

真正交换的是实际 CSV 文本，双方仍需要共享格式约定。

### 12.3 查看和移除

```python
print(csv.list_dialects())

dialect = csv.get_dialect("school_export")
print(dialect.delimiter)

csv.unregister_dialect("school_export")
```

`get_dialect()` 得到的是相应只读方言描述，不适合拿来直接修改全局配置。

需要变体时，创建 reader 或 writer 时覆盖具体参数，或注册另一个明确名称。

### 12.4 用类描述格式

```python
import csv


class SchoolDialect(csv.Dialect):
    delimiter = ";"
    quotechar = '"'
    doublequote = True
    escapechar = None
    lineterminator = "\n"
    quoting = csv.QUOTE_MINIMAL
    skipinitialspace = False
    strict = True
```

可以把类传给相应构造器。

类写法适合把配置集中放在一个模块中。

参数很少、只用一次时，直接关键字参数往往更容易理解，没有必要为每一张表都建立一套类层次。

### 12.5 我能只覆盖一项吗？

可以。

```python
reader = csv.reader(source, dialect="excel", delimiter=";")
```

在方言基础上，显式格式参数会覆盖对应项。

这种方式很方便，但调用者要清楚最终使用的规则，不要让多层配置来源使分隔符究竟是什么变得难以追踪。

## 第十三章：`Sniffer` 能自动识别格式吗？

### 13.1 它根据样本猜测，不拥有生成方知识

```python
import csv

with open("unknown.csv", encoding="utf-8", newline="") as source:
    sample = source.read(4096)
    dialect = csv.Sniffer().sniff(sample, delimiters=",;\t")
    source.seek(0)
    reader = csv.reader(source, dialect=dialect)

    for row in reader:
        print(row)
```

`sniff()` 分析一段文本，尝试推断分隔符等配置。

`delimiters` 限定候选字符，减少不合理猜测空间。

### 13.2 为什么要 `seek(0)`？

因为为了取样，我们已经从同一个文件流读走了前面一段。

后面的 reader 会从当前位置继续。

如果不回到开头，最前面的记录可能被跳过，甚至从一条记录中间开始解析。

这是文件流位置问题，与 Sniffer 本身是否识别成功是两个方面。

### 13.3 样本恰好截在引号字段中间怎么办？

取样可能不包含完整结构。

文件很小、字段特别长、分隔符出现次数很少，都可能影响推断。

你可以扩大合理样本、结合来源信息或要求用户明确选择格式。

不能因为工具名称叫 Sniffer，就把推断结果当成绝对正确的格式声明。

### 13.4 `has_header()` 是否一定判断对标题？

```python
has_header = csv.Sniffer().has_header(sample)
```

它是一种启发式判断，可能误判。

例如第一条真实数据恰好全是文本，后面多数列看起来像数字，就可能被当成标题。

正式数据导入若约定“必须有标题”，最好直接检查约定标题，而不是先猜测有没有。

### 13.5 识别失败时该怎么办？

捕获 `csv.Error`，提供明确提示：

```python
try:
    dialect = csv.Sniffer().sniff(sample, delimiters=",;\t")
except csv.Error as error:
    raise ValueError("无法根据样本可靠判断分隔格式，请明确指定") from error
```

如果选择回退到逗号格式，应让调用者知道发生了回退。

不应静默猜一个格式，再把后面列数不符误报成“用户忘写分数”。

## 第十四章：空行、空字段与缺失字段有什么不同？

### 14.1 三份输入先分别看

第一份是一条物理空行：

```text

```

第二份是一条包含两个空字段的记录：

```csv
,
```

第三份是一条只提供学号的记录：

```csv
001
```

它们在 CSV 结构上并不相同。

普通 reader 可能分别给出空列表、两个空字符串和一个字符串字段。

### 14.2 做一个可运行实验

```python
import csv
from io import StringIO

text = "\n,\n001\n"

with StringIO(text, newline="") as source:
    for row in csv.reader(source):
        print(repr(row))
```

输出：

```text
[]
['', '']
['001']
```

如果你要跳过真正的空记录，可以检查 `if not row`。

如果你写 `if not any(row)`，则会把所有字段都为空字符串的记录也跳过。

是否接受这种扩大，需要业务明确决定。

### 14.3 `DictReader` 会怎样处理空行？

`DictReader` 会跳过一些空行，不像普通 reader 一样把空列表直接交给循环。

但 `,` 形成的空字段记录仍然可以产生字典。

这意味着两种读取器的“循环次数”可能不同。

做精确行数统计时，应先定义你统计的是物理文本行、CSV 逻辑记录，还是非空业务记录。

### 14.4 空分数应该变成零吗？

不一定。

空字符串可能表示尚未录入，而零表示真实成绩为零。

如果你写：

```python
score = int(row["score"] or "0")
```

就把两种含义合并了。

对于成绩系统，可能更合适的是拒绝缺失分数，或者用 `None` 表示未录入，最后在统计中单独计算。

默认值要根据语义选择，不能只为了让转换不报错。

### 14.5 如果缺失值需要可逆保存呢？

可以规定一个专用文本标记，例如 `\N`，但还必须规定真实字符串本身等于这个标记时怎样转义。

如果你不设计这种冲突规则，所谓专用标记仍然可能与合法内容混淆。

数据模型复杂时，JSON 或数据库的明确空值支持更容易保持语义。

## 第十五章：解析错误与业务错误，分别在哪里处理？

### 15.1 三层错误

读取 CSV 可能发生文件访问错误。

内容可能无法按指定编码解码。

文本可能不符合指定 CSV 规则。

即使这些都成功，记录仍然可能缺列、重复编号或分数超范围。

你需要让错误提示区分这些原因。

### 15.2 基本结构

```python
import csv

try:
    with open("students.csv", encoding="utf-8", newline="") as source:
        reader = csv.reader(source, strict=True)
        try:
            for row in reader:
                print(row)
        except csv.Error as error:
            raise ValueError(
                f"CSV 解析失败，已读取到源文本第 {reader.line_num} 行附近：{error}"
            ) from error
except UnicodeDecodeError as error:
    print("文件编码不符合 UTF-8：", error)
except OSError as error:
    print("文件访问失败：", error)
except ValueError as error:
    print(error)
```

### 15.3 `reader.line_num` 是第几条记录？

它记录已经从源迭代器读取的文本行数，不等于返回过的逻辑记录数。

如果一条备注跨三行，解析这一条记录就可能推进三行。

所以提示中写“源文本第几行附近”更准确，不能直接命名为学生序号。

### 15.4 为什么不捕获后继续当前 reader？

某些语法错误会让解析器无法确认当前记录在哪里结束。

此时你不能总是假设“跳过这一行就能恢复正常”。

对于业务错误，例如某个完整解析出的分数不是整数，可以按记录跳过并记录原因。

对于 CSV 结构损坏，通常应该终止这份文件的解析，要求修正来源，或使用明确的恢复策略。

## 第十六章：完整实践一——读取成绩、校验标题和记录

### 16.1 输入规则

我们继续使用：

```csv
student_id,name,score
001,小林,95
002,小周,88
003,小陈,100
```

要求标题顺序固定，学号非空且唯一，姓名非空，成绩是零到一百的整数。

错误应包含文件和记录位置。

### 16.2 完整实现

```python
import csv
from pathlib import Path


EXPECTED_COLUMNS = ["student_id", "name", "score"]


def read_students(path):
    path = Path(path)
    students = []
    known_ids = set()

    with path.open(encoding="utf-8-sig", newline="") as source:
        reader = csv.reader(source, strict=True)
        header = next(reader, None)

        if header is None:
            raise ValueError(f"{path}：文件为空")
        if header != EXPECTED_COLUMNS:
            raise ValueError(f"{path}：标题应为 {EXPECTED_COLUMNS!r}，实际为 {header!r}")

        for record_number, row in enumerate(reader, start=1):
            location = f"{path} 第 {record_number} 条数据记录"

            if len(row) != len(EXPECTED_COLUMNS):
                raise ValueError(f"{location}：需要三列，实际 {len(row)} 列")

            student_id, name, score_text = row
            student_id = student_id.strip()
            name = name.strip()
            score_text = score_text.strip()

            if not student_id:
                raise ValueError(f"{location}：student_id 不能为空")
            if student_id in known_ids:
                raise ValueError(f"{location}：重复学号 {student_id!r}")
            if not name:
                raise ValueError(f"{location}：name 不能为空")

            try:
                score = int(score_text)
            except ValueError as error:
                raise ValueError(f"{location}：score 必须是整数") from error

            if not 0 <= score <= 100:
                raise ValueError(f"{location}：score 超出 0 到 100 范围")

            known_ids.add(student_id)
            students.append({
                "student_id": student_id,
                "name": name,
                "score": score,
            })

    return students


students = read_students("csv_practice/input/students.csv")
for student in students:
    print(student)
```

### 16.3 为什么这里使用普通 reader？

因为我们想先严格验证整条标题和每条记录的列数。

`DictReader` 很方便，但它的多列收纳、缺列填充值和重复标题映射行为需要额外处理。

普通 reader 加明确校验，对固定结构的导入程序同样清楚。

### 16.4 为什么学号不转成整数？

它是标识，保留前导零。

之后写回 CSV，也能保留 `001` 这个文本值。

要特别提醒：目标表格软件是否继续按文本展示，是另一层导入政策，不能仅靠这里保证。

### 16.5 为什么先验证再加入集合？

只有完整记录合法，才把学号加入 `known_ids` 并添加结果。

这样“已经接收的记录”与“正在验证但可能失败的记录”分工更清楚。

本例遇到第一条错误就终止，因此即使提前加入也未必立刻产生可见差异，但良好的步骤安排更容易扩展为逐条报告模式。

### 16.6 这里的 `.strip()` 会不会改变数据？

会，所以我们必须把它写进规则。

这个练习规定学号、姓名和分数字段两端空白不属于正式值。

如果你处理的备注、源代码或缩进文本需要保留空白，就不应照搬这种清理。

“清洗数据”也是修改数据，不能当作没有语义影响的固定步骤。

## 第十七章：完整实践二——计算成绩并导出排序报告

### 17.1 在上一章结果上继续

这里假设已经调用 `read_students()`，得到由合法字典组成的列表。

我们要按分数从高到低输出，同分按学号排序，再增加等级列。

### 17.2 完整实现

```python
import csv
from pathlib import Path


def score_grade(score):
    if score >= 90:
        return "优秀"
    if score >= 80:
        return "良好"
    if score >= 60:
        return "及格"
    return "待提高"


def export_score_report(path, students):
    path = Path(path)
    path.parent.mkdir(parents=True, exist_ok=True)

    sorted_students = sorted(
        students,
        key=lambda student: (-student["score"], student["student_id"]),
    )

    with path.open("w", encoding="utf-8-sig", newline="") as target:
        writer = csv.DictWriter(
            target,
            fieldnames=["rank", "student_id", "name", "score", "grade"],
            lineterminator="\n",
        )
        writer.writeheader()

        for rank, student in enumerate(sorted_students, start=1):
            writer.writerow({
                "rank": rank,
                "student_id": student["student_id"],
                "name": student["name"],
                "score": student["score"],
                "grade": score_grade(student["score"]),
            })


students = read_students("csv_practice/input/students.csv")
export_score_report("csv_practice/output/report.csv", students)
```

### 17.3 这是什么排名规则？

这里 `rank` 只是排序后从一开始的行序号。

同分学生仍然会得到不同序号。

如果你希望同分并列排名、后面跳号或不跳号，需要另外设计排名算法。

不要因为列名叫 rank，就让读者误以为所有常见竞赛排名规则都已经实现。

### 17.4 平均分要放在哪？

可以单独生成摘要文件，而不是在同一表格最后突然插入一条不同结构的说明。

```python
count = len(students)
average = sum(student["score"] for student in students) / count if count else None
```

如果没有学生，平均分不存在，不应除以零。

使用 `None` 在内存中表达“无结果”，导出时再明确选择空字段或人类可读提示。

### 17.5 为什么不直接给每个原字典增加 rank？

可以修改，但这会改变调用者传进来的数据。

这里构造新的输出字典，避免让导出过程顺便污染原始业务列表。

输入模型与输出模型可以相近，但不必完全相同。

## 第十八章：完整实践三——逐行转换大文件，边读边写

### 18.1 如果记录有几百万条，还要先 `list(reader)` 吗？

如果任务只是把某些列转换后导出，不需要全局排序，可以逐条处理。

输入读取一条，验证一条，写出一条。

这样不需要把全部记录同时保存在列表中。

### 18.2 示例：把分号分隔输入转成标准逗号输出

```python
import csv
from pathlib import Path


def convert_delimited_file(source_path, target_path):
    source_path = Path(source_path)
    target_path = Path(target_path)

    if source_path.resolve() == target_path.resolve():
        raise ValueError("输入输出不能使用同一路径")

    target_path.parent.mkdir(parents=True, exist_ok=True)
    count = 0

    with source_path.open(encoding="utf-8-sig", newline="") as source:
        with target_path.open("x", encoding="utf-8", newline="") as target:
            reader = csv.reader(source, delimiter=";", strict=True)
            writer = csv.writer(target, delimiter=",", lineterminator="\n")

            for row in reader:
                writer.writerow(row)
                count += 1

    return count
```

### 18.3 为什么目标用 `x`？

这个练习要求生成新文件，不覆盖已有目标。

排他创建把这个意图放在真正打开目标的步骤中。

但如果转换到一半失败，已经创建的目标可能只含部分数据。

“拒绝覆盖”和“完整发布”是两个不同要求，后面会继续处理。

### 18.4 记录含换行，也能这样转换吗？

可以，只要输入方言与内容一致，reader 会按逻辑记录产生字段列表，writer 再按目标规则编码。

你不应在中间先把每条记录重新用 `",".join(row)` 连接，那会丢失 writer 对引号和特殊字符的处理。

### 18.5 为什么无法顺便进行全局排序？

要知道谁排第一，通常需要看到全部可比较记录，或者利用外部排序、数据库索引等机制。

纯粹逐条读取写出保持了输入顺序。

如果你一边说“完全不保存所有记录”，一边在末尾 `sorted(all_rows)`，内存优势就已经被后面的收集步骤改变了。

### 18.6 流式处理仍有边界

某一条字段特别长，当前记录本身就可能很大。

你保存的错误列表、分组字典或去重集合，也可能随记录数量增长。

因此，需要观察整个处理链条，而不是只因为使用了 `for row in reader` 就宣称内存恒定。

## 第十九章：完整实践四——出错时保留旧报告，成功后再替换

### 19.1 直接写正式文件，有什么问题？

假设报告原来有一千条合法记录。

你用 `w` 打开后，文件立即截断。

新一轮处理到第三百条时遇到非法数据，程序退出，正式报告可能只剩前面三百条。

`with` 正确关闭了文件，但没有恢复旧版本。

如果你希望“新结果全部成功后，才让正式报告切到新版本”，可以先写同目录临时文件。

### 19.2 一个可以复用的保存函数

```python
import csv
import os
import tempfile
from pathlib import Path


def write_csv_replacing(path, fieldnames, records):
    path = Path(path).absolute()
    path.parent.mkdir(parents=True, exist_ok=True)
    temp_path = None

    try:
        with tempfile.NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            newline="",
            dir=path.parent,
            prefix=path.name + ".",
            suffix=".tmp",
            delete=False,
        ) as target:
            temp_path = Path(target.name)
            writer = csv.DictWriter(
                target,
                fieldnames=fieldnames,
                extrasaction="raise",
                lineterminator="\n",
            )
            writer.writeheader()
            writer.writerows(records)
            target.flush()
            os.fsync(target.fileno())

        temp_path.replace(path)
    finally:
        if temp_path is not None:
            temp_path.unlink(missing_ok=True)
```

### 19.3 为什么临时文件放在目标目录？

成功替换通常需要源与目标位于同一文件系统，才能利用相应路径替换语义。

临时目录若位于另一个磁盘，最后改名可能因跨文件系统失败。

把临时文件放在目标同目录，能减少这种常见差异。

### 19.4 为什么先关闭，再 `replace()`？

退出 `with` 后，临时输出的流已经关闭。

这既让 Python 缓冲得到正常处理，也减少 Windows 文件共享限制对替换的影响。

`delete=False` 是为了让临时文件在关闭后继续存在，供后续替换使用。

### 19.5 `writerows(records)` 接收生成器可以吗？

可以。

它会逐条消费可迭代对象。

如果生成器中途抛异常，临时输出可能不完整，但正式路径还没有执行替换。

这正是把“生成”和“发布”分开的实际价值。

### 19.6 为什么还要 `finally`？

处理失败时可能留下临时文件。

`finally` 尝试清理这个尚未发布的产物。

如果替换已经成功，原临时名字不再存在，`missing_ok=True` 允许这种情况。

清理本身也可能因权限等原因失败，程序不应假设所有错误路径都绝对无异常。

### 19.7 这个函数能提供数据库一样的事务吗？

不能这样扩大理解。

它针对单个结果文件，减少读者看到半成品的风险。

它不协调多个写入者，不保留旧文件全部元数据，也没有同步父目录元数据或提供多文件同时提交。

如果两个任务同时生成报告，后完成的替换仍可能覆盖前一个结果。

对真实数据库文件，应使用数据库自己的备份和事务机制，不能把它当普通 CSV 随便替换。

## 第二十章：完整实践五——合法记录和错误报告分别导出

### 20.1 遇到一条坏数据，是否一定停止？

有些导入任务要求全部正确才继续。

有些清洗任务允许把坏记录隔离，让人工稍后修复。

两种政策都可能合理，但必须明确。

下面选择第二种：只有已经完整解析的记录才能进入业务校验；CSV 结构损坏则停止，不假设能随意恢复。

### 20.2 输入示例

```csv
student_id,name,score
001,小林,95
002,小周,excellent
003,小陈,120
004,小吴,88
```

希望输出两条合法记录，同时报告第二、第三条数据记录为什么失败。

### 20.3 完整实现

```python
import csv
from pathlib import Path


def clean_students(source_path, accepted_path, rejected_path):
    source_path = Path(source_path)
    accepted_path = Path(accepted_path)
    rejected_path = Path(rejected_path)

    resolved = {p.resolve() for p in (source_path, accepted_path, rejected_path)}
    if len(resolved) != 3:
        raise ValueError("输入、合法输出和错误报告必须使用不同路径")

    accepted_path.parent.mkdir(parents=True, exist_ok=True)
    rejected_path.parent.mkdir(parents=True, exist_ok=True)
    accepted_count = 0
    rejected_count = 0
    seen_ids = set()

    with source_path.open(encoding="utf-8-sig", newline="") as source:
        with accepted_path.open("x", encoding="utf-8", newline="") as accepted:
            with rejected_path.open("x", encoding="utf-8", newline="") as rejected:
                reader = csv.reader(source, strict=True)
                header = next(reader, None)
                if header != ["student_id", "name", "score"]:
                    raise ValueError("标题不符合输入格式")

                good_writer = csv.writer(accepted, lineterminator="\n")
                bad_writer = csv.writer(rejected, lineterminator="\n")
                good_writer.writerow(header)
                bad_writer.writerow(["record_number", "source_line_end", "reason", "raw_fields"])

                for record_number, row in enumerate(reader, start=1):
                    try:
                        if len(row) != 3:
                            raise ValueError("必须恰好有三列")

                        student_id, name, score_text = (field.strip() for field in row)
                        if not student_id or not name:
                            raise ValueError("学号和姓名不能为空")
                        if student_id in seen_ids:
                            raise ValueError("学号重复")

                        score = int(score_text)
                        if not 0 <= score <= 100:
                            raise ValueError("分数超出 0 到 100")
                    except ValueError as error:
                        rejected_count += 1
                        bad_writer.writerow([
                            record_number,
                            reader.line_num,
                            str(error),
                            repr(row),
                        ])
                        continue

                    seen_ids.add(student_id)
                    good_writer.writerow([student_id, name, score])
                    accepted_count += 1

    return accepted_count, rejected_count
```

### 20.4 为什么错误报告保留两种位置？

`record_number` 是数据记录序号，不包含标题。

`source_line_end` 是解析完这条记录后读取器到达的物理行号。

多行字段会让两者不相同。

同时保留有助于人工定位，但它还不是精确字符偏移。

### 20.5 为什么 `repr(row)` 只作为诊断？

它是 Python 显示形式，便于人查看字段中的转义字符。

如果后续程序需要可靠恢复原字段，应该把它作为 JSON 编码，或把字段分别放在明确列中。

诊断文本与可机器交换的数据格式不能混为一谈。

### 20.6 两份结果能同时原子发布吗？

上面代码使用排他创建，避免覆盖已有报告，但中途失败仍可能留下部分输出。

如果要求合法表和错误表必须属于同一次完整处理，可以先在独立临时目录生成，再发布带任务标识的一组结果，或使用更完整的事务系统。

不能因为单个文件可以替换，就假设任意两个文件的替换也自动是同一个事务。

## 第二十一章：完整实践六——合并多份 CSV，标题只能出现一次

### 21.1 最常见的错误合并方式

你有三份 CSV，每份第一行都是标题。

如果直接把文本字节拼起来，中间会出现多次标题。

更严重的是，前一个文件没有末尾换行时，后一个标题可能直接粘在最后一条数据后面。

正确思路是逐份解析、校验标题、逐条重新写出。

### 21.2 完整实现

```python
import csv
from pathlib import Path


def merge_csv_files(source_paths, target_path, expected_header):
    source_paths = [Path(path) for path in source_paths]
    target_path = Path(target_path)

    if not source_paths:
        raise ValueError("至少需要一份输入文件")

    target_resolved = target_path.resolve()
    if any(path.resolve() == target_resolved for path in source_paths):
        raise ValueError("输出不能与输入使用同一路径")

    target_path.parent.mkdir(parents=True, exist_ok=True)
    count = 0

    with target_path.open("x", encoding="utf-8", newline="") as target:
        writer = csv.writer(target, lineterminator="\n")
        writer.writerow(expected_header)

        for source_path in source_paths:
            with source_path.open(encoding="utf-8-sig", newline="") as source:
                reader = csv.reader(source, strict=True)
                header = next(reader, None)

                if header != expected_header:
                    raise ValueError(f"{source_path}：标题不一致")

                for record_number, row in enumerate(reader, start=1):
                    if len(row) != len(expected_header):
                        raise ValueError(
                            f"{source_path} 第 {record_number} 条数据记录列数不一致"
                        )
                    writer.writerow(row)
                    count += 1

    return count


paths = sorted(Path("csv_practice/parts").glob("*.csv"))
merge_csv_files(paths, "csv_practice/output/merged.csv", ["student_id", "name", "score"])
```

### 21.3 为什么先排序文件名？

目录扫描不承诺固定顺序。

如果输出行顺序影响阅读和后续比较，应明确输入文件顺序。

文件名 `part01.csv`、`part02.csv`、`part10.csv` 更容易按普通字符串排序得到预期序号。

如果使用 `part1.csv`、`part10.csv`、`part2.csv`，结果可能与你想的自然数字顺序不同。

### 21.4 三份文件列相同，但排列不同呢？

本例严格要求顺序一致。

你也可以用 `DictReader` 按列名重排，但要验证标题唯一、字段集合相同，并明确多余列和缺列政策。

不要只比较 `set(header)` 就忽略重复列名，因为集合会去重，让 `id,name,name` 与另一份标题的差异被掩盖。

### 21.5 学号重复要在这里处理吗？

文件合并层只验证结构，不自动知道学号是否全局唯一。

如果任务要求唯一，应在跨文件共享的集合中检查，并记录第一次出现的来源。

同一个处理流程可以包含多层验证，但每层的职责要说清楚，不能让“成功合并”被误解为“所有业务约束都已通过”。

## 第二十二章：完整实践七——按类别分组统计，而不保存全部记录

### 22.1 输入是一张销售流水表

```csv
category,amount
book,12.50
course,99.00
book,7.50
course,20.00
```

我们只需要每类数量和金额总和，不需要按原样保留所有流水。

### 22.2 实现

```python
import csv
from collections import Counter, defaultdict
from decimal import Decimal, InvalidOperation


def summarize_sales(path):
    counts = Counter()
    totals = defaultdict(lambda: Decimal("0"))

    with open(path, encoding="utf-8-sig", newline="") as source:
        reader = csv.DictReader(source)
        if reader.fieldnames != ["category", "amount"]:
            raise ValueError("销售流水标题不符合约定")

        for number, row in enumerate(reader, start=1):
            if None in row or row["category"] is None or row["amount"] is None:
                raise ValueError(f"第 {number} 条记录列数不符合要求")

            category = row["category"].strip()
            if not category:
                raise ValueError(f"第 {number} 条记录类别为空")

            try:
                amount = Decimal(row["amount"])
            except InvalidOperation as error:
                raise ValueError(f"第 {number} 条记录金额不合法") from error

            if not amount.is_finite() or amount < 0:
                raise ValueError(f"第 {number} 条记录金额必须为非负有限值")

            counts[category] += 1
            totals[category] += amount

    return counts, dict(totals)
```

### 22.3 为什么需要 `Decimal("0")`？

我们希望金额始终使用十进制数值语义。

从字符串构造 Decimal，可以避免先经由 float 引入已有近似值。

`defaultdict` 第一次访问某个类别时，调用工厂创建初始零值。

### 22.4 类别数量也可能越来越多

这个程序不保存全部流水，但保存每个类别的累计状态。

如果类别只有十几种，内存很小。

如果每条记录都带一个全新的类别，分组字典仍然可能与输入数量一起增长。

这就是为什么“流式”需要结合你保留的状态来理解。

### 22.5 输出汇总

```python
counts, totals = summarize_sales("sales.csv")

with open("sales_summary.csv", "w", encoding="utf-8", newline="") as target:
    writer = csv.writer(target, lineterminator="\n")
    writer.writerow(["category", "count", "total_amount"])

    for category in sorted(counts):
        writer.writerow([category, counts[category], format(totals[category], "f")])
```

`format(..., "f")` 避免使用科学记数法展示金额。

它不自动执行币种换算，也不自动决定财务舍入规则。

如果金额必须保留两位小数，应在业务验证中确定输入精度和舍入政策，而不是最后随手截取字符串。

## 第二十三章：完整实践八——根据学号连接两张表

### 23.1 为什么不能直接把两张表按行拼起来？

第一张表保存学生基本信息：

```csv
student_id,name
001,小林
002,小周
003,小陈
```

第二张表保存成绩：

```csv
student_id,score
003,100
001,95
002,88
```

两张表排序不同。

如果把第一行学生与第一行成绩直接对应，就会把小林配成一百分。

正确的连接依据是学号，而不是当前行号。

### 23.2 先为一张较小的表建立索引

```python
import csv


def load_name_index(path):
    index = {}

    with open(path, encoding="utf-8-sig", newline="") as source:
        reader = csv.DictReader(source)
        if reader.fieldnames != ["student_id", "name"]:
            raise ValueError("姓名表标题不符合约定")

        for number, row in enumerate(reader, start=1):
            if None in row or any(value is None for value in row.values()):
                raise ValueError(f"姓名表第 {number} 条记录列数不符合约定")

            student_id = row["student_id"].strip()
            name = row["name"].strip()
            if not student_id or not name:
                raise ValueError(f"姓名表第 {number} 条记录缺少学号或姓名")
            if student_id in index:
                raise ValueError(f"姓名表存在重复学号：{student_id}")

            index[student_id] = name

    return index
```

### 23.3 用另一张表逐条查询

```python
import csv


def join_scores(names_path, scores_path, output_path):
    names = load_name_index(names_path)
    seen_scores = set()

    with open(scores_path, encoding="utf-8-sig", newline="") as source:
        with open(output_path, "x", encoding="utf-8", newline="") as target:
            reader = csv.DictReader(source)
            if reader.fieldnames != ["student_id", "score"]:
                raise ValueError("成绩表标题不符合约定")

            writer = csv.DictWriter(
                target,
                fieldnames=["student_id", "name", "score"],
                lineterminator="\n",
            )
            writer.writeheader()

            for number, row in enumerate(reader, start=1):
                if None in row or any(value is None for value in row.values()):
                    raise ValueError(f"成绩表第 {number} 条记录列数不符合约定")

                student_id = row["student_id"].strip()
                if student_id not in names:
                    raise ValueError(f"成绩表存在未登记学号：{student_id}")
                if student_id in seen_scores:
                    raise ValueError(f"成绩表存在重复学号：{student_id}")

                score = int(row["score"])
                if not 0 <= score <= 100:
                    raise ValueError(f"学号 {student_id} 的分数超出范围")

                writer.writerow({
                    "student_id": student_id,
                    "name": names[student_id],
                    "score": score,
                })
                seen_scores.add(student_id)

    missing = set(names) - seen_scores
    return sorted(missing)
```

### 23.4 返回的缺失名单表示什么？

它表示姓名表中存在，但成绩表里没有记录的学号。

这里选择把这些学生作为缺失名单返回，而没有自动给零分。

原因是“没有提供成绩”和“成绩确实为零”具有不同含义。

你也可以选择在输出中保留空分数，但必须明确说明这是外连接政策，而不是随意补零。

### 23.5 为什么需要检查两张表各自的重复？

如果姓名表学号重复，字典覆盖会悄悄丢掉前一个名字。

如果成绩表学号重复，输出可能出现一名学生多条成绩，但业务原本要求一人一条。

连接算法本身不会自动知道数据唯一性约束，必须在建立索引和消费记录时检查。

### 23.6 两张表都特别大怎么办？

把一张表整个放进字典会占用内存。

可以使用数据库做连接，或者先按连接键排序，再进行归并连接。

这类选择与数据规模、存储条件和排序成本有关，不应把一个方便的小表字典例子当成所有规模的最佳方案。

## 第二十四章：CSV 与 JSON 互转，怎样避免丢失类型？

### 24.1 从 CSV 到 JSON，默认得到什么？

```python
import csv
import json

with open("students.csv", encoding="utf-8-sig", newline="") as source:
    records = list(csv.DictReader(source))

print(json.dumps(records, ensure_ascii=False, indent=2))
```

输出中分数通常仍然是字符串。

这只是容器结构转换，没有自动完成业务类型推断。

### 24.2 要得到数字字段，应该在哪一步转换？

在验证模型时转换。

```python
converted = []

for number, row in enumerate(records, start=1):
    score = int(row["score"])
    if not 0 <= score <= 100:
        raise ValueError(f"第 {number} 条记录分数越界")
    converted.append({
        "student_id": row["student_id"],
        "name": row["name"],
        "score": score,
    })
```

学号保持字符串，分数转成整数，选择来自列语义。

### 24.3 JSON 数组太大，还要全部放列表吗？

如果目标允许 JSON Lines，可以逐条转换逐条写：

```python
import csv
import json

with open("students.csv", encoding="utf-8-sig", newline="") as source:
    with open("students.jsonl", "x", encoding="utf-8", newline="\n") as target:
        reader = csv.DictReader(source)
        for row in reader:
            target.write(json.dumps(row, ensure_ascii=False, allow_nan=False) + "\n")
```

这生成每行一个 JSON 值的记录格式，不是整个文件一份 JSON 数组。

接收方必须采用相应逐行解析方式。

### 24.4 JSON 中有嵌套对象怎么办？

CSV 是二维表格，嵌套结构不能无约定地摊进去。

例如：

```json
{"name":"小林","address":{"city":"上海","street":"示例路"}}
```

你可以把结构摊平成 `address_city`、`address_street` 两列，也可以把整个 address 编码为一个 JSON 文本字段。

两种方式都需要定义恢复规则。

直接 `str(address)` 会得到 Python 字典显示形式，不是标准 JSON，也不适合作为跨语言交换约定。

### 24.5 空值与空字符串互转要特别小心

JSON 有 `null`，CSV 默认空字段通常无法区分原先是 `None` 还是空字符串。

如果互转必须可逆，应使用明确标记、额外类型列或更合适的格式。

格式转换会改变能表达的结构范围，不能仅检查“文件已经生成”就判断转换完全保真。

## 第二十五章：把 CSV 导入 SQLite，为什么有时更实用？

### 25.1 当你需要反复筛选、排序和连接

单次扫描 CSV 很方便。

但如果同一份数据要反复按不同条件查询，数据库能提供索引、约束和 SQL 聚合等能力。

这里使用内存数据库练习，不接触真实项目数据。

### 25.2 完整导入示例

```python
import csv
import sqlite3


connection = sqlite3.connect(":memory:")

try:
    connection.execute("""
        CREATE TABLE students (
            student_id TEXT PRIMARY KEY,
            name TEXT NOT NULL,
            score INTEGER NOT NULL CHECK (score BETWEEN 0 AND 100)
        )
    """)

    with open("students.csv", encoding="utf-8-sig", newline="") as source:
        reader = csv.DictReader(source)
        if reader.fieldnames != ["student_id", "name", "score"]:
            raise ValueError("标题不符合约定")

        with connection:
            for number, row in enumerate(reader, start=1):
                if None in row or any(value is None for value in row.values()):
                    raise ValueError(f"第 {number} 条记录列数错误")

                student_id = row["student_id"].strip()
                name = row["name"].strip()
                score = int(row["score"])
                if not student_id or not name:
                    raise ValueError(f"第 {number} 条记录标识或姓名为空")

                connection.execute(
                    "INSERT INTO students(student_id, name, score) VALUES (?, ?, ?)",
                    (student_id, name, score),
                )

    for row in connection.execute(
        "SELECT student_id, name, score FROM students WHERE score >= ? ORDER BY score DESC",
        (90,),
    ):
        print(row)
finally:
    connection.close()
```

### 25.3 为什么 SQL 使用问号？

它是参数占位符，值通过第二个参数单独传入。

不要用 f-string 把 CSV 中的姓名直接拼进 SQL 字符串。

参数化既处理引号等内容，也能避免把字段内容误当 SQL 代码。

### 25.4 `(90,)` 末尾逗号为什么必须有？

这是一个单元素元组。

`(90)` 只是给整数加括号，没有形成序列。

SQL 参数接口需要对应参数集合，所以写成 `(90,)`。

这是 Python 元组语法与数据库接口结合的典型小细节。

### 25.5 `with connection` 结束就自动关闭吗？

不会。

连接上下文管理主要处理适用的事务提交和回滚。

我们仍在外层 `finally` 调用 `connection.close()`，明确释放连接。

这里采用常见的默认兼容事务用法；更细的 Python 3.12+ `autocommit` 控制应在 SQLite 专门教材中继续理解。

## 第二十六章：空值之外，还要考虑电子表格公式解释

### 26.1 为什么一个普通文本字段也可能有额外含义？

有些电子表格软件会把以等号等字符开头的内容解释为公式。

CSV 里只是字符串，但导入软件可能进一步解释。

因此，把外部用户输入导出给电子表格使用时，需要考虑目标工具的行为。

### 26.2 CSV 引号能解决吗？

引号解决的是 CSV 字段边界。

它不保证电子表格软件永远把字段当普通文本。

所以，“我已经用 writer 正确转义”与“目标软件不会执行公式含义”是两层要求。

### 26.3 应当怎样设计输出？

根据目标工具和使用方式，明确文本导入政策。

可以通过指定文本列、采用支持单元格类型的工作簿格式，或根据已验证规则生成面向展示的安全文本副本。

有些做法会主动改变字段，例如添加前导字符，这时不能再声称输出与原始数据逐字相同。

### 26.4 不要无条件修改所有负数

以减号开头的字段可能只是合法负金额或负温度。

如果你把所有这样的数据都任意改写，会损害数值语义。

安全展示与原始数据保存可以分成两个产物，分别说明用途。

对程序间交换的原始数据，按协议保真；对特定人类工具的展示，按工具行为进行明确处理。

## 第二十七章：字段大小、内存和资源限制

### 27.1 为什么特别长的字段会抛错？

CSV 解析器对字段大小有相应限制。

可以查询：

```python
import csv

current_limit = csv.field_size_limit()
print(current_limit)
```

实际值依实现和运行环境而定，不应在教程里把某个数字当作永远固定。

### 27.2 能不能直接改成特别大？

可以设置新的合理上限：

```python
previous = csv.field_size_limit(2_000_000)
try:
    # 在这里执行你明确需要的 CSV 读取。
    pass
finally:
    csv.field_size_limit(previous)
```

返回值可用于恢复原设置。

但这是进程内共享配置，其他同时运行的读取任务也可能受到影响。

线程或复杂服务中，不要在不知道影响范围时反复修改全局限制。

### 27.3 为什么不直接设成系统最大整数？

输入可能不可信，极长字段可能消耗大量内存。

放宽限制应由真实业务需求支撑，例如备注最多允许多少字符，而不是为了“任何输入都不报错”。

此外，不同平台底层整数范围也可能影响可接受设置。

### 27.4 文件大小与记录数也要分别限制

一份小文件可以包含很多很短记录，一份大文件也可能只有一个极长字段。

如果应用要求输入最多十万条记录，应在逻辑记录循环中计数。

如果请求体或文件总量有上限，应在输入层限制原始字节。

不同限制保护不同资源，不能只设置一个字段上限就宣称已经控制全部开销。

### 27.5 大文件能不能随机跳到第十万条记录？

普通 CSV 没有内建记录索引。

字段可能跨行，也可能使用变长编码，所以不能简单用“平均行长乘序号”定位。

通常要从头解析到目标位置，或提前建立经过验证的偏移索引。

频繁随机查询时，数据库可能更适合。

## 第二十八章：怎样验证一个 CSV 处理程序确实正确？

### 28.1 只看到文件生成，还不够

一个脚本运行结束，没有异常，输出文件也出现了。

这只能说明某一条执行路径完成。

你还要检查：标题是否正确、记录数量是否符合预期、前导零是否保留、特殊字符能否往返、错误数据有没有被无声丢弃。

### 28.2 用临时目录做可重复实验

```python
import csv
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "roundtrip.csv"
    expected = [
        ["id", "name", "note"],
        ["001", "小林", "包含,逗号"],
        ["002", "小周", '包含"双引号"'],
        ["003", "小陈", "包含\n换行"],
        ["004", "  有空格  ", ""],
    ]

    with path.open("w", encoding="utf-8", newline="") as target:
        csv.writer(target).writerows(expected)

    with path.open(encoding="utf-8", newline="") as source:
        actual = list(csv.reader(source))

    assert actual == expected
    print("所有字段完成读写往返")
```

这里的 `assert` 用于本地测试，不替代正式程序必须执行的输入校验。

测试完成离开上下文，临时目录会安排清理，不会污染真实资料。

### 28.3 为什么比较解析结果，而不是只比较文件字符串？

同样字段可以采用不同但合法的引号布局和记录结束符。

如果测试目标是“内容能正确往返”，比较字段结构更合适。

如果协议要求字节级固定输出，就另外比较二进制字节，并明确编码、BOM、行结束符和 quoting 策略。

两类测试都可能有价值，关键是说明验证的性质。

### 28.4 哪些输入特别值得测试？

空文件。

只有标题、没有数据的文件。

标题重复、缺列和多列。

空字符串、缺失字段、只有空格的姓名。

带逗号、引号、CRLF 和字段内部换行的备注。

重复标识和越界分数。

错误编码、损坏引号和非常长字段。

这些例子分别触及不同边界，远比把同一条正常记录重复一千遍更能发现问题。

### 28.5 失败时还要检查旧结果

如果采用临时文件替换，应先写一个可识别的旧报告，再故意让新记录生成器中途抛异常。

然后重新读取正式路径，确认旧报告没有被替换。

同时检查临时文件是否按约定清理。

这验证的是保存流程的失败边界，不是单纯检查 writer 会不会输出逗号。

## 第二十九章：练习一到四——从格式理解开始

### 29.1 练习一：为什么不能直接 `split()`？

给定这一条 CSV 文本：

```csv
001,"Lin, Alice","说了""你好"""
```

要求正确取得三个字段。

先自己用 `split(",")` 观察错误，再用读取器解析。

参考：

```python
import csv
from io import StringIO

text = '001,"Lin, Alice","说了""你好"""\n'

with StringIO(text, newline="") as source:
    row = next(csv.reader(source))

print(row)
```

预期：

```text
['001', 'Lin, Alice', '说了"你好"']
```

注意看：字段内部逗号没有变成额外分隔，双写引号恢复成一个普通引号字符。

这道题的重点是区分内容字符与结构字符。

### 29.2 练习二：保留学号前导零

读取 `student_id,name,score` 三列，将分数转换成整数，学号保持原样。

参考核心逻辑：

```python
student_id = row["student_id"]
name = row["name"]
score = int(row["score"])
```

不要对整行每个字段统一 `int()`。

更不要把“数字外观”当成“数值语义”。

自己验证 `001`、`000` 和 `A001` 三种学号，确认它们都按标识文本处理。

如果业务规定学号必须只含某些字符，应该单独验证格式，而不是借整数转换顺便验证。

### 29.3 练习三：空文件不应误当成功导入

要求：文件为空时给出明确错误；只有标题时返回空学生列表。

```python
header = next(reader, None)

if header is None:
    raise ValueError("输入文件为空，没有标题")
if header != ["student_id", "name", "score"]:
    raise ValueError("标题不符合约定")

students = []
for row in reader:
    students.append(row)
```

这两个输入的差别是：后者具有合法表结构，只是当前没有数据。

是否允许零条数据，由任务决定。

不要用一个笼统的 `if not students` 同时解释所有情况。

### 29.4 练习四：写出一个含内部换行的备注

要求生成一条数据记录，其中备注有两段文字。

```python
import csv
from io import StringIO

with StringIO(newline="") as target:
    writer = csv.writer(target, lineterminator="\n")
    writer.writerow(["id", "note"])
    writer.writerow(["001", "第一段\n第二段"])
    text = target.getvalue()

print(text, end="")

with StringIO(text, newline="") as source:
    records = list(csv.reader(source))

assert len(records) == 2
assert records[1][1] == "第一段\n第二段"
```

物理显示可能占多行，但逻辑记录只有标题和一条数据。

自己比较 `len(text.splitlines())` 与 `len(records)`，理解为什么它们不一定相同。

## 第三十章：练习五到八——校验、转换和错误定位

### 30.1 练习五：缺列与空字段分开处理

准备：

```csv
id,name,score
001,小林,
002,小周
```

要求分别报告“分数明确为空”和“记录缺少 score 字段”。

参考：

```python
value = row["score"]

if value is None:
    print("缺少 score 字段")
elif value == "":
    print("score 字段明确为空")
else:
    print("提供了分数字段：", value)
```

这段逻辑假设已经用正常 `DictReader` 读取，且标题已经验证。

如果你在前面把所有 `None` 都替换成空字符串，就失去了这项区分信息。

### 30.2 练习六：拒绝重复标题

给定标题：

```text
['id', 'name', 'name']
```

不要先转换成字典再检查，因为那时信息可能已经被覆盖。

参考：

```python
from collections import Counter

counts = Counter(fieldnames)
duplicates = [name for name, count in counts.items() if count > 1]

if duplicates:
    raise ValueError(f"重复标题：{duplicates}")
```

如果你还会清理标题两端空格，应在清理之后再次检查重复。

练习输入 `name` 和 ` name `，说明为什么“原始标题不同”不保证“规范化标题不同”。

### 30.3 练习七：为坏分数加上位置

要求把：

```text
invalid literal for int()
```

变成包含文件名、逻辑记录号和列名的错误。

参考：

```python
try:
    score = int(row["score"])
except ValueError as error:
    raise ValueError(
        f"{path} 第 {record_number} 条记录的 score 无法转换为整数"
    ) from error
```

不要为了显示简短提示就丢掉异常原因。

`from error` 保留原始失败链，方便开发者排查。

对外展示时仍应避免泄露不该公开的完整输入内容。

### 30.4 练习八：将 TSV 转为 CSV

输入以制表符分隔，字段内部仍可能有引号和换行。

参考：

```python
import csv

with open("input.tsv", encoding="utf-8", newline="") as source:
    with open("output.csv", "x", encoding="utf-8", newline="") as target:
        reader = csv.reader(source, delimiter="\t")
        writer = csv.writer(target, delimiter=",", lineterminator="\n")
        writer.writerows(reader)
```

这里转换结构格式，没有主动转换分数字段或日期类型。

如果需求只是不改变内容地更换分隔格式，保留字段字符串反而更合适。

## 第三十一章：练习九到十二——把方法连成一个小工具

### 31.1 练习九：按成绩筛选及格记录

要求：验证每条输入，再只输出分数不低于六十的记录。

不要把格式错误当成“不及格”静默跳过。

参考处理顺序：

```python
score = int(row["score"])
if not 0 <= score <= 100:
    raise ValueError("分数超出范围")

if score >= 60:
    writer.writerow({
        "student_id": row["student_id"],
        "name": row["name"],
        "score": score,
    })
```

先判断输入是否有效，再判断有效记录是否符合筛选条件。

这两类判断混在一起，错误统计和业务统计就会失真。

### 31.2 练习十：只保留某些列，但不能悄悄漏掉必填列

假设源表有十列，你只想导出 `id,name,email`。

可以使用字段白名单构造新字典：

```python
selected_columns = ["id", "name", "email"]

if not set(selected_columns).issubset(reader.fieldnames or []):
    raise ValueError("输入缺少需要导出的列")

for row in reader:
    output_row = {name: row[name] for name in selected_columns}
    writer.writerow(output_row)
```

这里仍需要根据输入契约验证重复标题和不规则列数。

明确选择列，比把原字典全部写出再期待工具自动判断哪些字段应该保密更容易审阅。

### 31.3 练习十一：输出可供人工查看的错误表

要求每条错误包含来源文件、记录号、字段名和简短原因。

不要把完整原始行无条件放进日志，因为它可能包含不该公开的个人信息。

参考结构：

```python
error_record = {
    "source": str(path),
    "record_number": record_number,
    "field": "score",
    "message": "必须是 0 到 100 的整数",
}
```

用 `DictWriter` 输出固定列顺序。

如果同一记录有多个错误，可以选择每个错误一行，或一条记录合并错误列表，但要明确对应方式。

### 31.4 练习十二：验证失败时不破坏旧报告

先建立旧结果：

```python
from pathlib import Path

path = Path("old_report.csv")
path.write_text("old\nkeep\n", encoding="utf-8")
```

创建一个中途失败的记录生成器：

```python
def broken_records():
    yield {"id": "001", "name": "小林"}
    raise ValueError("模拟第二条数据生成失败")
```

调用前面 `write_csv_replacing()`，捕获预期错误后，重新读取正式文件。

```python
try:
    write_csv_replacing(path, ["id", "name"], broken_records())
except ValueError:
    pass

assert path.read_text(encoding="utf-8") == "old\nkeep\n"
```

这道题检查的是失败语义。

如果你把实现改成直接 `w` 打开正式文件，这个断言就很可能失败。

## 第三十二章：再回答一些真正写脚本时会问的问题

### 32.1 `writer.writerow("Alice")` 为什么把字母分开了？

因为字符串本身可迭代。

`writerow()` 期待一条记录的字段序列，你给一个字符串，它就可能把每个字符当成一个字段。

如果只有一个字段，应写：

```python
writer.writerow(["Alice"])
```

或单元素元组：

```python
writer.writerow(("Alice",))
```

这与数据库参数中的单元素元组问题很相似，都是容器形状必须正确。

### 32.2 `writerows()` 和 `writerow()` 只是名字长短不同吗？

不是。

`writerow()` 接受一条记录。

`writerows()` 接受多条记录组成的可迭代对象。

```python
writer.writerow(["001", "小林", 95])
writer.writerows([["002", "小周", 88], ["003", "小陈", 100]])
```

如果给 `writerows()` 传一条平坦字符串列表，它可能把每个字符串分别当成一条记录，再按字符迭代，结果完全不同。

### 32.3 `writerow()` 返回的是写了多少条吗？

它返回底层流 `write()` 调用的返回值。

普通文本流中通常与本次生成文本的字符数量有关，不是固定返回一。

`writerows()` 通常不提供记录计数返回值。

需要条数时，应在你的记录循环中明确累计。

### 32.4 为什么离开 `with` 后，reader 还在却不能继续读？

读取器对象仍然存在，但它依赖的文件流已经关闭。

这和文件对象变量仍在、底层资源却已关闭的情况一样。

想把内容带出去，可以在 `with` 内转成列表，前提是规模适合内存。

想流式输出，就明确设计生成器生命周期或把读取循环留在上下文内。

### 32.5 空白行能不能表示注释？

CSV 标准库没有一个普遍适用的“以 `#` 开头就是注释”的默认规则。

如果你在读取前逐物理行过滤 `#`，可能误删一个多行引号字段内部的合法内容。

只有当来源格式明确约定注释，且你能保证不会与字段内容冲突时，才设计对应预处理。

不要把配置文件常见注释习惯直接套到所有 CSV。

### 32.6 能不能在文件末尾追加记录？

可以使用 `a` 打开，再创建 writer。

但你要确认已有文件的编码、方言、列顺序和末尾记录边界都符合要求。

如果旧文件最后一条记录没有结束符，盲目追加可能把新记录粘在后面。

多个写入者同时追加也不能仅靠 writer 保证业务记录事务性。

### 32.7 用 `DictWriter` 追加时，要不要再写标题？

如果已经有同结构标题，通常不要重复 `writeheader()`。

但判断文件是否存在并不足以判断是否具有合法标题，还要考虑空文件、损坏文件和其他进程的修改。

个人脚本可以按明确规则检查后追加；重要长期日志往往采用专门的记录格式或日志系统更容易管理。

### 32.8 CSV 能保存公式、格式、颜色和多个工作表吗？

CSV 通常只保存表格字段文本。

字体、列宽、颜色和多工作表结构不属于这种格式的基本表达范围。

某个表格软件可能把字段解释成公式，但那是消费行为，不等于 CSV 提供了完整工作簿模型。

如果需求本来是生成带格式的 Excel 工作簿，应选择相应工作簿工具，而不是不断往 CSV 中塞私有约定。

### 32.9 为什么输出顺序有时变了？

看看上游数据来源是否用了集合、未排序目录遍历或数据库无 `ORDER BY` 查询。

CSV writer 按你提供记录的顺序写出，不会自动按学号排序。

列顺序由字段列表或你提供的字段序列决定，行顺序由你消费记录的顺序决定。

### 32.10 `errors="replace"` 能否让所有坏文件继续处理？

它只处理文本解码层的问题，并可能改变原字符。

CSV 结构损坏、字段缺失和业务错误不会因此自动修复。

即使得到可读文本，替代字符也可能让关键标识不再正确。

容错必须说明允许损失什么、如何记录问题，以及结果是否还能用于原业务。

## 第三十三章：常用接口在实际流程中各放在哪里？

先决定路径和来源编码。

然后打开文本流，读 CSV 结构。

再校验标题、列数和业务类型。

之后完成筛选、转换、连接或统计。

最后按明确输出契约写入，并考虑失败时是否应保留旧结果。

对应关系可以整理为：

| 环节 | 常用接口 | 需要自己决定的内容 |
| --- | --- | --- |
| 路径构造 | `Path(...)`、`/` | 相对哪个基准 |
| 文本流 | `open(..., newline="", encoding=...)` | 编码、创建或覆盖策略 |
| 顺序字段读取 | `csv.reader()` | 标题与列数验证 |
| 按列名读取 | `csv.DictReader()` | 重复标题、缺列、多列政策 |
| 顺序字段写入 | `csv.writer()` | 分隔、引号、结束符 |
| 字典写入 | `csv.DictWriter()` | 字段白名单与顺序 |
| 写一条 | `writerow()` | 一条记录的字段序列 |
| 写多条 | `writerows()` | 可迭代记录集合 |
| 写标题 | `writeheader()` | 是否已有合法标题 |
| 猜测格式 | `csv.Sniffer()` | 推断失败和误判政策 |
| 格式复用 | `register_dialect()` | 共享格式名称与范围 |
| 解析定位 | `line_num`、`csv.Error` | 物理行与逻辑记录区别 |
| 资源边界 | `field_size_limit()` | 合理上限与共享影响 |

表格适合最后复习。

真正写程序时，先从输入和输出契约出发，不必为了用完所有 API 而增加不需要的复杂度。

官方参考：[Python csv 模块](https://docs.python.org/3.14/library/csv.html)、[Python 3.12 新 quoting 模式的读取限制说明](https://docs.python.org/3.12/library/csv.html#csv.QUOTE_NOTNULL)、[CSV 格式背景 RFC 4180](https://www.rfc-editor.org/rfc/rfc4180)。本篇的成绩、销售和文件处理案例为独立教学设计，字段规则只属于相应练习；真实交换应与生成方和接收方确认格式契约。
