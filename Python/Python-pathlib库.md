# Python `pathlib` 库超详细教程：从路径对象，到目录遍历、文件管理与项目实践

## 零基础先导：`Path` 是“路径的表示”，不是“已经打开的文件”

在 `os.path` 中，你经常传入字符串：`"data/report.txt"`。`pathlib` 则把路径变成有方法和属性的 `Path` 对象，让拼接、查看文件名、创建目录等操作更直观。

```python
from pathlib import Path

folder = Path("data")            # ① 路径对象，尚未创建目录
target = folder / "report.txt"   # ② / 这里是重载的路径拼接运算符
print(target.name)               # ③ report.txt，仅提取名字
print(target.suffix)             # ④ .txt，仅提取扩展名
print(target.exists())           # ⑤ 查询路径目前是否存在
```

重点：①②③④ 都不意味着磁盘上真的创建了文件；⑤ 才查询文件系统。要创建目录，可使用 `folder.mkdir(parents=True, exist_ok=True)`。要写文件，则选择 `target.write_text(..., encoding="utf-8")` 或 `open`。

`Path.cwd()` 返回当前工作目录；`Path(__file__).parent` 与脚本文件所在目录有关，两者**不一定相同**。`Path.resolve()` 也不是“文件必然存在”的证明。在 Windows 与 Linux 间处理路径时，留意不同系统的分隔符、路径规则与大小写行为；不要为了一个固定路径字符串而混用 Windows 的 `C:\...` 与 Linux 的 `/home/...`。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

适合已经了解字符串、列表、循环、函数，刚开始系统学习文件与目录处理的读者。

你可能已经在代码里见过：

```python
from pathlib import Path

path = Path("data") / "report.txt"
```

为什么一个“路径”要做成对象？为什么中间的 `/` 能连接目录和文件名？构造这行代码之后，文件到底创建了没有？

把这些基础问题讲明白，后面的 `name`、`parent`、`glob()`、`resolve()`、`mkdir()` 和 `open()` 才不会变成一串孤立的名字。

本文以 Python 3.11 及以上的常用写法为基础。`Path.walk()`、复制移动新接口等涉及较新版本的地方会明确标注。示例使用独立的练习目录，不应直接指向重要资料目录。

## 第一章：`pathlib` 是什么，为什么需要它？

### 1.1 先看传统字符串路径处理

```python
import os

filename = os.path.join("data", "reports", "result.txt")
parent = os.path.dirname(filename)
name = os.path.basename(filename)
root, suffix = os.path.splitext(filename)
```

这些写法完全有效。它们把路径交给一组函数处理，返回字符串、布尔值或其他结果。

当同一路径需要反复拆分、拼接、查询和打开时，你会不断把它作为参数传入不同函数。`pathlib` 提供另一种组织方式：让路径成为对象，把与它相关的操作放在对象上。

```python
from pathlib import Path

path = Path("data") / "reports" / "result.txt"
parent = path.parent
name = path.name
suffix = path.suffix
```

`pathlib` 可以理解为 path library，也就是路径库。它从 Python 3.4 起进入标准库，不需要通过 pip 安装。[官方说明：pathlib 的定位和引入版本](https://docs.python.org/3.14/library/pathlib.html)

它的由来不是因为字符串不能表示路径，而是因为路径有自己的组成规则和常见操作，专门的对象接口更容易把这些规则组织清楚。[设计背景：PEP 428](https://peps.python.org/pep-0428/)

### 1.2 路径对象不等于文件内容，也不等于已打开的文件

```python
path = Path("data") / "report.txt"
```

这一步通常只构造描述位置的对象，不读取磁盘内容，不创建 `data` 目录，也不创建 `report.txt`。

后续操作才决定是否访问文件系统：

```python
print(path.name)       # 分析路径组成，不读取文件内容
print(path.exists())   # 查询文件系统状态
```

```python
path.parent.mkdir(parents=True, exist_ok=True)  # 实际创建目录
path.write_text("一份报告\n", encoding="utf-8")  # 实际写入文件
```

始终分清这三类事情：构造地址、查询地址对应的对象、修改文件系统。许多初学者的困惑都来自把它们当成同一步。

### 1.3 `Path` 是类，`path` 是你取的变量名

```python
from pathlib import Path

path = Path("notes.txt")
```

`pathlib` 是模块名称。`Path` 是模块提供的类，按惯例首字母大写。调用类会创建一个路径对象。

`path` 是当前代码中的普通变量，可以改成 `input_path`、`report_path` 或 `config_file`。它不是 Python 保留字。

```python
report_path = Path("output") / "report.txt"
```

有意义的变量名能让你区分输入、输出、根目录和临时目录。不要在长程序中把所有路径都命名为 `p`，然后靠记忆判断每个对象用途。

## 第二章：先写一个小程序，观察路径对象究竟做了什么

这一章我们只完成一件小事：准备一个练习目录，在里面写一份报告，再把报告读出来。

暂时不同时引入遍历、复制、链接和批量操作。

把每一步的效果看清楚，后面的接口就更容易连接起来。

### 2.1 先只构造一个路径

```python
from pathlib import Path

report_path = Path("pathlib_practice") / "output" / "report.txt"

print(report_path)
```

在 Linux 或 WSL 的 Python 中，通常显示：

```text
pathlib_practice/output/report.txt
```

在原生 Windows Python 中，通常显示：

```text
pathlib_practice\output\report.txt
```

两种显示都表达相应平台的目录层级。

这里的组成是：

```text
pathlib_practice
    └── output
        └── report.txt
```

#### 运行后，文件夹创建了吗？

还没有。

这几行只是构造并打印一个地址。

就像你在纸上写下一个房间位置，写出地址这件事本身不会把房间建起来。

如果运行前磁盘上没有 `pathlib_practice`，运行后它仍然可能不存在。

如果之前已经有这个目录，目录仍在，也不能据此认为刚才是 `Path(...)` 创建的。

理解代码效果时，要把运行前状态和运行后状态分开观察。

### 2.2 如果想知道文件应该放在哪个目录呢？

这个文件的直接父目录是：

```python
report_path.parent
```

完整看：

```python
from pathlib import Path

report_path = Path("pathlib_practice") / "output" / "report.txt"
output_dir = report_path.parent

print(output_dir)
```

Linux 或 WSL 中通常得到：

```text
pathlib_practice/output
```

`parent` 是属性，不加调用括号。

它根据路径组成取得父路径，不需要先打开 `report.txt`。

因此，即使报告还没创建，我们仍然可以计算“将来应该把它放在哪个目录”。

### 2.3 现在真正创建目录

```python
output_dir.mkdir(parents=True, exist_ok=True)
```

这一行才请求操作系统创建目录。

把两个参数拆开。

首先：

```python
parents=True
```

表示路径中需要的父目录缺失时，也允许把它们创建出来。

假设最开始这两级都不存在：

```text
pathlib_practice/output
```

程序就可以先建立 `pathlib_practice`，再建立 `output`。

然后：

```python
exist_ok=True
```

表示目标已经是目录时，允许继续。

这让你第二次运行练习时，不必为了“目录已经存在”专门删除它再重来。

#### 那是不是任何错误都不会报了？

不是。

如果 `output` 这个名字被一个普通文件占用，它就不能同时作为目录。

如果没有权限，或者文件系统只读，也可能失败。

参数表达的是允许哪一种已有状态，不是忽略一切问题。

### 2.4 目录有了，怎样写文件？

```python
written = report_path.write_text(
    "你好，pathlib！\n",
    encoding="utf-8",
)
```

`write_text()` 会打开目标，写入字符串，再关闭。

如果目标文件已经存在，旧内容会被覆盖。

因此，这里使用的是专门的练习报告，不应把路径改成你唯一一份重要资料后反复试验。

#### `written` 保存的是文件对象吗？

不是。

它保存这次写入的字符数量。

可以打印：

```python
print(written)
```

在这个例子中，它统计字符串中的码点数量，包括末尾换行，而不是 UTF-8 编码后的字节数量。

#### 为什么没有写 `close()`？

因为 `write_text()` 已经把“打开、写入、关闭”组织在这个高层方法中。

但这个便利方法不表示写入具有事务性，也不表示失败后会自动恢复旧内容。

如果你要追加、逐行输出，或控制更细的写入过程，可以使用 `report_path.open()`。

### 2.5 如果还想把内容读回来呢？

```python
text = report_path.read_text(encoding="utf-8")

print(repr(text))
```

通常看到：

```text
'你好，pathlib！\n'
```

`read_text()` 返回的是字符串。

`repr()` 让末尾换行显示为 `\n`，方便确认文件内容。

如果直接：

```python
print(text)
```

末尾换行会真正执行，后面 `print()` 默认还会再加一个换行，视觉上可能多出一条空白行。

这属于文本输出行为，和 `pathlib` 有没有正确读取文件是不同问题。

### 2.6 合成一个能运行的小程序

```python
from pathlib import Path

# 第一步：构造目标地址。
report_path = Path("pathlib_practice") / "output" / "report.txt"

# 第二步：取得并创建父目录。
output_dir = report_path.parent
output_dir.mkdir(parents=True, exist_ok=True)

# 第三步：写入一份练习文本。
written = report_path.write_text("你好，pathlib！\n", encoding="utf-8")

# 第四步：重新读取内容。
text = report_path.read_text(encoding="utf-8")

# 第五步：观察结果。
print("目标路径：", report_path)
print("文件名：", report_path.name)
print("父目录：", report_path.parent)
print("写入字符数：", written)
print("读取内容：", repr(text))
```

你可以先运行，再打开练习目录观察文件结构。

然后再次运行，确认目录已存在不会阻止程序继续，而报告会重新写成指定内容。

### 2.7 如果去掉创建目录的那一行，会发生什么？

这要看目录之前是否存在。

如果你已经运行过完整程序，目录仍在，那么删掉 `mkdir()` 后再次写入，可能依然成功。

这不意味着 `write_text()` 自动创建了目录。

它只是使用了上一次运行留下的目录。

要验证缺失父目录的行为，应换一个新的练习目录名称，例如：

```python
report_path = Path("another_new_practice") / "output" / "report.txt"
```

在父目录确实不存在时，直接 `write_text()` 通常会因无法找到相应父路径而失败。

这个实验很重要：学习文件系统操作时，之前运行留下的状态经常会影响下一次结果。

不要仅凭“这一遍成功了”就推断某个方法具备它并没有提供的功能。

### 2.8 如果只想知道文件名，不读内容呢？

使用：

```python
report_path.name
```

得到：

```text
report.txt
```

这里没有调用读取接口。

名字来自路径组成，内容来自磁盘文件，两者分别取得。

有了这个区别，看到下面两行就能知道它们在做什么：

```python
name = report_path.name
text = report_path.read_text(encoding="utf-8")
```

第一行分析地址。

第二行访问内容。

后面的很多方法，仍然是在这两类工作上继续展开。


## 第三章：`Path` 如何选择操作系统规则？

### 3.1 `Path` 使用运行 Python 的平台规则

```python
from pathlib import Path

path = Path("data", "report.txt")
print(type(path))
```

在常见 Linux、WSL 和 macOS Python 中，得到 `PosixPath`；在原生 Windows Python 中，得到 `WindowsPath`。

你通常不需要自己判断后再选择这两个类，使用 `Path` 即可。

注意，WSL 中运行的是 Linux 程序。电脑的桌面虽然是 Windows，WSL 的 Python 路径规则仍然按 Linux 处理。

### 3.2 `PurePath` 与具体路径类

`pathlib` 把纯路径计算与文件系统访问分开：

| 类 | 主要作用 | 能否实际读写文件 |
| --- | --- | --- |
| `Path` | 按当前平台处理路径并提供 I/O | 可以 |
| `PurePath` | 按当前平台做纯路径计算 | 不提供实际 I/O |
| `PurePosixPath` | 明确按 POSIX 规则分析字符串 | 不提供实际 I/O |
| `PureWindowsPath` | 明确按 Windows 规则分析字符串 | 不提供实际 I/O |

“Pure” 是纯粹的意思：这些类适合只分析地址，不真正访问对应磁盘。

### 3.3 在 Linux 分析 Windows 路径

```python
from pathlib import PureWindowsPath

path = PureWindowsPath(r"E:\notes\Python\lesson.md")

print(path.drive)
print(path.name)
print(path.parent)
```

输出中的关键信息是盘符 `E:`、文件名 `lesson.md` 和父路径。

这个对象能分析 Windows 路径，不代表 Linux 已经能够通过这个 Windows 地址访问磁盘。

如果文件在 WSL 的挂载盘中，实际访问通常要使用对应 Linux 路径：

```python
from pathlib import Path

actual = Path("/mnt/e/notes/Python/lesson.md")
```

路径语法分析、系统间路径映射和磁盘挂载是三件不同的事。

### 3.4 `Path` 不接受随意的字节路径

`Path` 接受字符串路径或符合相应路径协议的对象。通常不能直接传 `bytes` 路径：

```python
# 错误示例：pathlib 以字符串路径为主要接口。
Path(b"data.txt")
```

某些底层 `os` 接口可以处理字节路径，这不表示 `pathlib` 的构造规则与它们完全相同。日常应用优先使用 `str` 与 `Path`。

## 第四章：路径构造与 `/` 运算符

### 4.1 最常见的三种写法

```python
from pathlib import Path

first = Path("project/data/report.txt")
second = Path("project", "data", "report.txt")
third = Path("project") / "data" / "report.txt"
```

在当前平台规则下，这几种方式可表达相同的相对路径。

第三种特别常见，因为 `/` 容易让人看出路径层级。但只有参与运算的对象支持这种路径运算时，它才具有这个含义。

```python
"project" / "data"
```

两个普通字符串之间没有这种除法运算，会触发 `TypeError`。

### 4.2 `/` 为什么在这里不是除法？

Python 允许对象通过特殊方法定义运算符行为。`Path` 的相关实现让 `/` 表达路径连接。

这属于运算符重载：运算符的含义取决于参与运算的对象类型。数字的 `/` 做除法，路径对象的 `/` 做路径连接。

这不是把所有斜杠都重新定义，也不是字符串里出现 `/` 就会执行运算。

```python
path = Path("project") / "data"
text = "project/data"
```

前者执行对象运算，后者只是字符串字面量。

### 4.3 `joinpath()`：用方法表达连接

```python
base = Path("project")
path = base.joinpath("data", "report.txt")
```

它与相应 `/` 连接的意图相同。路径片段由列表提供时，可以解包：

```python
parts = ["data", "reports", "2026", "summary.txt"]
path = base.joinpath(*parts)
```

`*parts` 把列表的每个元素分别作为位置参数传入。它是函数调用的解包语法，不属于 `pathlib` 专有语法。

### 4.4 后面的绝对路径可能重置前面的部分

POSIX 规则示例：

```python
from pathlib import PurePosixPath

base = PurePosixPath("/home/alice/project")
print(base / "data.txt")
print(base / "/etc/config.txt")
```

输出：

```text
/home/alice/project/data.txt
/etc/config.txt
```

后者不是把 `/etc/config.txt` 安全限制到项目目录，而是使用了新的绝对路径。

Windows 还要考虑盘符、根路径和盘符相对路径，稍后单独展开。不要认为路径连接天然提供用户输入的目录限制。

### 4.5 `Path()`、`Path("")` 和 `Path(".")`

这些常见写法都表示当前目录这个相对位置：

```python
Path()
Path("")
Path(".")
```

它们不等于已经把当前工作目录的绝对地址固定下来。

```python
relative = Path(".")
absolute_now = Path.cwd()
```

如果之后进程改变当前工作目录，`relative` 参与后续 I/O 时会按新的基准解释；`absolute_now` 保存的是取得时的绝对路径表示。

这也是为什么大型程序不宜到处调用 `os.chdir()`。一个看起来没有被修改的相对路径对象，实际访问目标仍可能随着进程状态变化。

## 第五章：拆分路径——属性的返回值类型一定要分清

### 5.1 使用统一例子

下面使用纯 POSIX 路径，让结果不受你当前操作系统影响：

```python
from pathlib import PurePosixPath

path = PurePosixPath("/home/alice/data/report.final.tar.gz")
```

| 表达式 | 结果 | 类型或性质 |
| --- | --- | --- |
| `path.name` | `report.final.tar.gz` | 字符串 |
| `path.stem` | `report.final.tar` | 字符串 |
| `path.suffix` | `.gz` | 字符串 |
| `path.suffixes` | `['.final', '.tar', '.gz']` | 字符串列表 |
| `path.parent` | `/home/alice/data` | 路径对象 |
| `path.parts` | `('/', 'home', 'alice', 'data', 'report.final.tar.gz')` | 元组 |
| `path.anchor` | `/` | 字符串 |
| `path.root` | `/` | 字符串 |
| `path.drive` | 空字符串 | POSIX 下通常无盘符 |

这里 `suffixes` 也把 `.final` 当作后缀组成。这是根据点号拆分路径名称的规则，不是在识别某个真正的归档格式。

### 5.2 `.name` 是最后一个路径组成部分

```python
print(PurePosixPath("data/report.txt").name)
print(PurePosixPath("data/archive").name)
```

结果分别是 `report.txt` 和 `archive`。

第二个名字可能属于目录，也可能属于无扩展名文件。仅靠 `name` 不会查询真实类型。

`Path("data/")` 通常会规范掉末尾多余分隔符，因此其 `name` 与 `os.path.basename("data/")` 的空字符串结果不完全相同。迁移旧代码时要确认这些边界。

### 5.3 `.stem` 只去掉最后一个后缀

```python
p = PurePosixPath("backup.tar.gz")

print(p.stem)
print(p.suffix)
print(p.suffixes)
```

输出：

```text
backup.tar
.gz
['.tar', '.gz']
```

`stem` 不自动去掉所有点号后面的内容，也不自动认识 `.tar.gz` 是一个整体归档后缀。

如果业务明确处理 `.tar.gz`，可以按这个完整约定操作文件名：

```python
name = "backup.tar.gz"
base_name = name.removesuffix(".tar.gz")
```

这样比不加区分地删除所有 `suffixes` 更能保留像 `report.final` 这样的业务名称。

### 5.4 隐藏文件与结尾点号

```python
p = PurePosixPath(".gitignore")
print(p.stem)
print(repr(p.suffix))
```

通常得到 `.gitignore` 和空后缀。开头的点号不直接被当作一个普通扩展名分隔。

结尾只有点号的名字存在版本差异：Python 3.14 起，单个 `.` 可以被视为有效后缀。旧版本对 `name.` 等情况的结果或 `with_suffix(".")` 的接受程度不同。

因此，普通 `.txt`、`.csv`、`.py` 示例可以稳定学习；遇到特殊名字，应检查目标 Python 版本和真实文件系统限制。[版本说明：suffix](https://docs.python.org/3.14/library/pathlib.html#pathlib.PurePath.suffix)

### 5.5 `.parent` 与 `.parents`

```python
p = PurePosixPath("/home/alice/project/data/report.txt")

print(p.parent)
print(p.parents[0])
print(p.parents[1])
```

`parent` 是直接父路径。`parents` 是按由近到远组织的祖先路径序列，索引零也是直接父路径。

```text
/home/alice/project/data
/home/alice/project/data
/home/alice/project
```

可以迭代：

```python
for parent in p.parents:
    print(parent)
```

这些是路径结构的计算结果，不保证祖先目录真的存在。

### 5.6 根目录的父路径不会继续向上

```python
root = PurePosixPath("/")
print(root.parent)
```

仍然是 `/`。因此，自己写“不断 `parent` 直到没有父目录”的循环时，要考虑根路径是一个不再变化的终点。

相对路径的 `Path("report.txt").parent` 是 `Path(".")`，不会自动变成当前工作目录绝对地址。要取得明确的绝对父目录，应先按需求 `absolute()` 或 `resolve()`。

## 第六章：修改名字，和真正重命名，是两类操作

### 6.1 `with_name()`：生成新的最后一段名称

```python
from pathlib import Path

old = Path("reports") / "draft.txt"
new = old.with_name("final.txt")

print(old)
print(new)
```

旧对象仍描述 `reports/draft.txt`，新对象描述 `reports/final.txt`。

磁盘文件没有因为这个方法被重命名。`with_name()` 是路径构造方法，不执行文件系统修改。

新名字应是单独的名称，不能用它偷偷传入多级路径。需要改变父目录时，用新的父路径连接文件名。

### 6.2 `with_suffix()`：替换最后一个后缀

```python
path = Path("data/report.final.txt")

print(path.with_suffix(".md"))
print(path.with_suffix(""))
```

得到 `data/report.final.md` 和 `data/report.final`。

新后缀通常要以点号开头。`.with_suffix("md")` 并不是等价简写。

对 `backup.tar.gz` 使用 `.with_suffix(".zip")`，得到 `backup.tar.zip`。它只替换最后的 `.gz`，不会自动把整个归档文件转换成 zip。

### 6.3 `with_stem()`：保留最后后缀，替换主体

```python
path = Path("report.txt")
new = path.with_stem(path.stem + "_backup")
print(new)
```

输出 `report_backup.txt`。`with_stem()` 从 Python 3.9 起提供。

对于 `backup.tar.gz`，主体是 `backup.tar`。设计批量命名规则前要先确认“主体”究竟指最后一个后缀之前，还是整个业务后缀之前。

### 6.4 路径对象不可变，但目标文件系统会变化

```python
p = Path("draft.txt")
q = p.with_name("final.txt")
```

这里 `p` 本身不会被改成另一个地址。你要保存新路径，就将新对象绑定给变量。

不可变的是路径对象的表示，不是其对应文件的内容、大小、权限和存在状态。其他程序仍然可以在任何时候修改目标，`Path` 不是给文件拍了一份永久快照。

## 第七章：我现在究竟在哪个目录？怎样找到脚本旁边的文件？

前面已经学会构造和拆分路径。

接下来会遇到一个特别实际的问题：

同样写：

```python
Path("data") / "input.txt"
```

为什么在编辑器里运行可以找到文件，换到终端却说不存在？

为什么我明明运行的是 D 盘脚本，打印出来的目录却在 C 盘？

要把这些现象解释清楚，需要分开三个位置：

当前工作目录。

当前脚本所在目录。

当前用户的主目录。

它们可能碰巧相同，也可能完全不同。

### 7.1 `Path.cwd()`：程序目前以哪里解释相对路径？

先写：

```python
from pathlib import Path

print(Path.cwd())
```

`cwd` 来自 current working directory，也就是当前工作目录。

这里的 `cwd()` 是类方法，通过 `Path` 直接调用，返回表示当前目录的路径对象。

它回答的是：“当前进程使用的工作目录在哪里？”

#### 先从最容易成功的启动方式看起

假设你的脚本是：

```text
D:\project\main.py
```

项目中还有：

```text
D:\project\data\input.txt
```

结构如下：

```text
D:\project\
├── main.py
└── data\
    └── input.txt
```

你在终端中进入：

```text
D:\project
```

然后运行：

```powershell
python main.py
```

如果没有其他代码修改工作目录，脚本中的：

```python
print(Path.cwd())
```

通常输出：

```text
D:\project
```

再构造：

```python
input_path = Path("data") / "input.txt"
```

读取时就会以这个工作目录为基准，寻找：

```text
D:\project\data\input.txt
```

所以这一次能正常找到项目里的数据。

#### 如果终端没有进入项目目录呢？

现在，脚本和数据都不移动。

但你的终端停留在：

```text
C:\Users\Alice
```

然后执行：

```powershell
python D:\project\main.py
```

这条命令指定要运行哪个脚本。

它通常不会自动把工作目录切换到脚本位置。

因此，脚本里的：

```python
print(Path.cwd())
```

通常仍然输出：

```text
C:\Users\Alice
```

而这条相对路径：

```python
Path("data") / "input.txt"
```

用于读取时，对应的通常是：

```text
C:\Users\Alice\data\input.txt
```

它和项目中的输入文件不在同一个位置。

这就解释了“文件明明在项目里，怎么换一种启动方式就找不到”。

脚本位置没变，工作目录变了，相对路径的基准也随之变了。

#### 能不能把两者同时打印出来看？

可以，而且这是很有用的排错习惯。

在普通 `.py` 脚本中：

```python
from pathlib import Path

print("当前工作目录：", Path.cwd())
print("当前脚本位置：", Path(__file__).resolve())
```

如果按照刚才第二种方式启动，可能看到：

```text
当前工作目录： C:\Users\Alice
当前脚本位置： D:\project\main.py
```

这样就能一眼看出：程序在哪个位置运行，与代码文件保存在哪里，是两个问题。

### 7.2 如果想获取脚本所在目录呢？

常见写法是：

```python
from pathlib import Path

script_dir = Path(__file__).resolve().parent

print(script_dir)
```

这行代码很常见，但初学时不要把它当成不可拆分的固定写法。

我们分四步来看。

#### 第一步：`__file__` 提供模块文件位置

在普通脚本中：

```python
print(__file__)
```

可以观察当前模块的文件位置。

它可能受启动方式、解释器版本和加载方式影响，因此不应单凭显示形式就推断一定是某种固定样子。

我们需要的是：从当前模块位置出发，构造明确的路径。

#### 第二步：`Path(__file__)` 把位置交给路径对象处理

```python
script_path = Path(__file__)
```

现在 `script_path` 是路径对象，可以继续使用 `resolve()`、`parent`、`name` 等成员。

请注意，这一步并没有打开脚本读取源码。

它只是把脚本位置表示为一个适合处理的对象。

#### 第三步：`.resolve()` 解析成绝对位置

```python
script_path = Path(__file__).resolve()
```

在普通项目例子中，得到的地址对应：

```text
D:\project\main.py
```

`resolve()` 还会解析符号链接。

如果你通过一个链接位置运行代码，解析后的目录可能是实际目标文件所在位置。要以链接位置为基准还是以真实目标为基准，应按项目需求决定。

#### 第四步：`.parent` 取得父目录

```python
script_dir = script_path.parent
```

从：

```text
D:\project\main.py
```

取得：

```text
D:\project
```

这就是“脚本所在目录”这个需求对应的结果。

熟悉后把这四步合并：

```python
script_dir = Path(__file__).resolve().parent
```

看到这条链时，你仍然应该能读出每一层做什么。

### 7.3 如果想读取脚本旁边 `data` 目录中的文件呢？

沿着已经取得的脚本目录继续拼接：

```python
from pathlib import Path

script_dir = Path(__file__).resolve().parent
input_path = script_dir / "data" / "input.txt"

with input_path.open("r", encoding="utf-8") as source:
    text = source.read()

print(text)
```

这次 `input_path` 从一开始就以脚本目录为明确基准。

它不再要求用户先把终端切到项目目录。

#### 为什么 `script_dir / "data" / "input.txt"` 后面不加斜杠？

因为每一项提供一个路径组成，`pathlib` 负责按照当前平台连接。

不必自己写成：

```python
str(script_dir) + "/data/input.txt"
```

后者在一些环境里也能工作，但会把原本已经有的路径对象能力退回字符串拼接，还容易产生遗漏或重复分隔符。

#### 为什么 `input_path.open()` 不再传路径参数？

因为这个方法属于已经描述目标地址的路径对象。

对照看：

```python
open(input_path, "r", encoding="utf-8")
```

和：

```python
input_path.open("r", encoding="utf-8")
```

前者把路径作为内置函数参数。

后者由对象本身提供路径，参数从模式开始。

文件的读取、编码和关闭规则仍然相同。

#### 能不能直接 `read_text()`？

对于规模可控的小文本，可以：

```python
text = input_path.read_text(encoding="utf-8")
```

它会内部打开、读取并关闭，返回字符串。

如果你要逐行处理大型日志，使用 `open()` 加循环更合适。是否采用简短写法，应根据数据大小和处理方式选择。

### 7.4 那是不是应该先 `chdir()` 到脚本目录？

你可能会想到：既然相对路径会受工作目录影响，那我先把工作目录改成脚本目录，不就好了？

在一个非常简单的独立脚本中，这样做有时可以工作。

但要理解它改的是什么。

```python
import os
from pathlib import Path

script_dir = Path(__file__).resolve().parent
os.chdir(script_dir)
```

它改变的是当前进程的工作目录。

并不只对下一次文件读取生效。

程序里其他模块使用的相对路径、后续启动的子进程，以及同进程其他线程的普通相对文件操作，都可能受影响。

假设用户还提供了一个相对于启动目录的输入文件名，你中途切换工作目录后，这个文件名也可能改指其他位置。

所以，对于固定项目资源，直接构造明确路径通常更容易看清影响范围。

### 7.5 如果用户从命令行提供文件呢？

这个需求和“读取脚本自带资源”稍有不同。

假设工具放在：

```text
D:\tools\count_lines.py
```

用户当前在：

```text
C:\Users\Alice\Documents
```

这里有：

```text
C:\Users\Alice\Documents\notes.txt
```

用户执行：

```powershell
python D:\tools\count_lines.py notes.txt
```

用户通常希望处理当前目录中的笔记。

这时可以直接按命令行传入的相对路径构造对象：

```python
import sys
from pathlib import Path

input_path = Path(sys.argv[1])

with input_path.open(encoding="utf-8") as source:
    count = sum(1 for line in source)

print(count)
```

这个最小例子假设用户已经提供参数。完整命令行程序还应验证参数数量，或用 `argparse` 给出帮助信息。

重点在于：这里保留当前工作目录作为基准，是符合用户调用意图的选择。

你不应该无条件给所有相对路径都加上脚本目录。

### 7.6 `Path.home()`：如果想找当前用户的主目录呢？

使用：

```python
from pathlib import Path

home = Path.home()
print(home)
```

原生 Windows 上，可能得到：

```text
C:\Users\Alice
```

Linux 或 WSL 中，可能得到：

```text
/home/alice
```

实际结果由当前用户和系统环境决定。

这里的 home 是“用户主目录”，不是脚本目录，也不是工作目录。

#### 为什么需要它？

假设你想给个人工具保存配置。

如果直接写：

```python
config_path = Path("C:/Users/Alice/.myapp/config.json")
```

换一个用户名或换到 Linux 就不方便。

可以改成：

```python
config_path = Path.home() / ".myapp" / "config.json"
```

这样把用户主目录交给系统确定，再按工具自己的约定构造子路径。

#### 构造出来后，目录会自动存在吗？

不会。

要写配置之前，还需要：

```python
config_path.parent.mkdir(parents=True, exist_ok=True)
```

`home()` 帮你定位主目录。

`/` 帮你构造目标地址。

`mkdir()` 才负责创建缺失目录。

把三个动作分开理解，后面就不容易把路径对象当成自动文件管理器。

#### `Path.home() / "Downloads"` 一定是下载目录吗？

它描述的是主目录中名为 `Downloads` 的子位置。

很多系统确实使用这个常见默认位置。

但用户可以把下载目录重定位到另一个盘，系统也可能采用不同配置。

所以这是一种常见位置约定，不是查询所有操作系统“真实下载目录”的通用接口。

### 7.7 路径里的 `~` 会自动变成主目录吗？

看下面：

```python
path = Path("~/notes.txt")
print(path)
```

通常仍然看到：

```text
~/notes.txt
```

单纯构造 `Path` 不会自动模仿 shell 展开 `~`。

需要调用：

```python
expanded = path.expanduser()
print(expanded)
```

或者直接写：

```python
path = Path.home() / "notes.txt"
```

如果当前平台无法确定相应主目录，`expanduser()` 可能抛出 `RuntimeError`，不要假设任何形状的用户名称都一定能展开。

### 7.8 如果我运行在 WSL，应该用哪一种位置？

假设 Windows 中的项目位置是：

```text
E:\notes\Python
```

在常见 WSL 挂载布局下，Linux 中通常通过：

```text
/mnt/e/notes/Python
```

访问相应内容。

如果当前运行的是 WSL 的 Python，就应按照 Linux 视角构造路径：

```python
from pathlib import Path

project = Path("/mnt/e/notes/Python")
```

下面这种写法不会自动转换盘符：

```python
Path(r"E:\notes\Python")
```

对于 Linux 路径规则，它不等价于 `/mnt/e/notes/Python`。

如果只是想在 Linux 中分析一串 Windows 路径的文件名，可以使用：

```python
from pathlib import PureWindowsPath

windows_path = PureWindowsPath(r"E:\notes\Python\lesson.md")
print(windows_path.name)
```

输出：

```text
lesson.md
```

这一步是在分析 Windows 路径语法，没有把它转换为当前 Linux 文件系统能直接访问的地址。

### 7.9 到这里，用三个问题检查自己有没有混淆

我要知道相对路径目前从哪里开始解释？

看：

```python
Path.cwd()
```

我要知道普通模块文件所在目录？

看：

```python
Path(__file__).resolve().parent
```

我要知道当前用户主目录？

看：

```python
Path.home()
```

这三项分别描述不同位置。

以后看到路径问题，先明确自己需要的是哪一种，再选接口，通常比先到处拼一个绝对路径更清楚。


## 第八章：`absolute()` 与 `resolve()`，为什么不能混用？

### 8.1 `absolute()`：补成绝对地址表示

```python
p = Path("data/../report.txt")
print(p.absolute())
```

它根据当前工作目录构造绝对路径，但不负责完整规范化或解析符号链接，`..` 可能仍保留。

它与 `os.path.abspath()` 不能在所有细节上直接视为同一个操作，尤其路径简化规则不同。

### 8.2 `resolve()`：解析符号链接和 `..`

```python
p = Path("data/../report.txt")
resolved = p.resolve()
```

它会转成绝对路径，尝试解析符号链接，并处理 `..` 等实际路径含义。

默认 `strict=False`，会尽可能解析已有部分，再保留尚不存在的剩余部分。因此返回结果不意味着整条路径已经存在，更不会自动创建目录或文件。

### 8.3 `strict=True`：要求解析成功

```python
try:
    actual = Path("data/input.txt").resolve(strict=True)
except OSError as error:
    print("无法解析目标：", error)
```

缺失目标、不可访问路径等情况会导致失败。符号链接循环的异常类型在 Python 3.13 前后有变化：较老版本可能抛出 `RuntimeError`，新版统一到相应 `OSError` 处理规则。

### 8.4 为什么 `..` 不能总是直接从字符串中删除？

假设在 POSIX 文件系统中：

```text
/work/project/link -> /srv/releases/current
```

路径：

```text
/work/project/link/../config.txt
```

操作系统跟随链接后，`..` 的基准可能是 `/srv/releases/current` 的父目录。它不必然与简单删掉字符串里的 `link/..` 后得到的路径相同。

因此，路径词法运算和实际文件系统解析不能混为一谈。`Path` 构造时不会贸然消掉所有 `..`，是有语义原因的。

### 8.5 解析完成后，文件仍可能变化

`resolve(strict=True)` 成功，说明解析时刻取得了相应结果。下一次 `open()` 之前，文件或目录仍可能被其他进程改动。

对自己的静态项目目录，这种方式很好用。对不可信用户可以修改的共享目录，还要考虑竞争条件和更底层的操作系统约束，不能把一次解析当成永久锁定。

## 第九章：判断存在、文件类型与链接类型

### 9.1 `exists()`：查询目标是否可以确认存在

```python
path = Path("data/input.txt")

if path.exists():
    print("路径对应的目标存在")
```

它不专门检查普通文件。目录、其他可确认存在的文件系统对象也可能得到 `True`。

默认情况下它跟随符号链接。如果链接本身还在，但链接目标已被删除，`exists()` 可能返回 `False`，而 `is_symlink()` 仍然为 `True`。

### 9.2 `False` 不总能精确解释为“绝对不存在”

Python 3.14 起，`exists()`、`is_file()`、`is_dir()` 等查询方法对操作系统返回的各种 `OSError` 都改为返回 `False`。较早版本会压制部分错误，但对另一些错误抛异常。

因此，需要区分“不存在”和“权限不足”等原因时，应使用会保留错误的 `stat()`，并分别捕获异常。

```python
try:
    info = path.stat()
except FileNotFoundError:
    print("目标不存在")
except PermissionError:
    print("无法访问目标或某级父目录")
except OSError as error:
    print("其他系统错误：", error)
else:
    print("已取得状态：", info.st_size)
```

这种写法的重点是错误信息，而不是为了做一个多余的存在性预检查。[版本依据：文件类型与状态查询](https://docs.python.org/3.14/library/pathlib.html#querying-file-type-and-status)

### 9.3 `is_file()` 与 `is_dir()`

```python
if path.is_file():
    print("普通文件")
elif path.is_dir():
    print("目录")
else:
    print("无法确认属于以上两类")
```

文件名包含 `.txt`，不保证它是普通文件；一个叫 `report.txt` 的目录也是合法的可能情况。

这两个方法默认会跟随符号链接判断目标。因此，链接指向普通文件时，`is_file()` 和 `is_symlink()` 可能同时为真。

希望排除链接的常见兼容写法是：

```python
if path.is_file() and not path.is_symlink():
    print("当前观察到的是非符号链接的普通文件")
```

Python 3.13 起，`is_file()`、`is_dir()` 还支持 `follow_symlinks=False`。需要兼容 3.11 或 3.12 时，不要直接使用这个新参数。

### 9.4 `is_symlink()`：观察链接条目本身

```python
print(path.is_symlink())
```

即使链接目标缺失，它也可以返回 `True`。这与默认跟随目标的存在判断不同。

Python 3.12 起，`exists(follow_symlinks=False)` 可以用于不跟随链接的存在性检查。这个参数不是所有旧版本都有，版本要求应写在程序说明中。

### 9.5 其他类型查询

`is_socket()`、`is_fifo()`、`is_block_device()`、`is_char_device()` 用于识别特定系统对象。它们主要在系统编程和 Unix 类环境中有实际意义。

不是所有路径都指向可当普通文件读取的对象。例如 FIFO 可能等待另一端，设备文件可能具有特殊行为。来源不可信的路径输入，不宜仅凭扩展名就无条件按普通文件处理。

Windows 的目录联接可以通过较新版本的 `is_junction()` 识别。它与普通快捷方式 `.lnk` 文件也不是同一个概念。需要针对 Windows 链接做完整管理时，应理解具体链接类型和平台支持。

### 9.6 查询不是加锁

```python
if path.exists():
    text = path.read_text(encoding="utf-8")
```

检查和读取之间，其他进程可能删除目标。即使 `exists()` 返回 `True`，读取仍然可能失败。

如果真正的任务是读取，直接尝试读取并处理预期错误通常更贴近需求：

```python
try:
    text = path.read_text(encoding="utf-8")
except FileNotFoundError:
    print("读取时目标已经不存在")
```

这也避免了把存在性检查误当成后续操作成功的保证。

## 第十章：`stat()`、时间和文件大小

### 10.1 `stat()` 返回状态对象

```python
info = Path("data/input.txt").stat()

print(info.st_size)
print(info.st_mtime)
print(info.st_mode)
```

`stat` 来自文件状态查询相关接口名称。返回的 `os.stat_result` 包含大小、时间、类型和权限等信息。

常用属性：

| 属性 | 含义 | 注意事项 |
| --- | --- | --- |
| `st_size` | 大小，通常按字节 | 目录值不是递归内容总大小 |
| `st_mtime` | 内容最后修改时间戳 | 不等于业务创建时间 |
| `st_atime` | 最近访问时间戳 | 受挂载和系统更新策略影响 |
| `st_ctime` | 系统定义的 ctime | Unix 通常是元数据变更时间 |
| `st_mtime_ns` | 纳秒形式修改时间 | 表示精度不等于硬件真实分辨率 |
| `st_mode` | 类型与权限位 | 不是只包含三位权限数字 |
| `st_dev`、`st_ino` | 设备与文件身份信息 | 平台和文件系统差异存在 |

### 10.2 一次查询后使用多个字段

```python
path = Path("report.pdf")
info = path.stat()

size_mib = info.st_size / (1024 * 1024)
print(f"大小：{size_mib:.2f} MiB")
```

如果需要大小和修改时间，先取得一个状态对象，可以减少重复查询，也能让这几个字段来自同一次状态观察。

但这仍不是整个目录树的事务快照。扫描期间发生变化时，不同文件的状态可能来自不同时间。

### 10.3 把修改时间转换为可读日期

```python
from datetime import datetime, timezone

info = Path("report.pdf").stat()
modified = datetime.fromtimestamp(info.st_mtime, tz=timezone.utc)
print(modified.isoformat())
```

显式使用 UTC，避免读者把本地时间与 UTC 混在一起。需要显示某个地区时间时，再使用相应时区转换。

不要不加区分地把 `st_ctime` 命名为 `created_at`。Unix 的 ctime 通常不是文件创建时间；某些系统另有 birth time 等信息，但跨平台可用性不同。

### 10.4 `lstat()`：查看符号链接自身

```python
link_info = Path("latest").lstat()
```

默认 `stat()` 看链接目标，`lstat()` 看链接条目本身。这样就能理解为什么一个很短的链接和它指向的大文件会报告不同的大小。

`stat(follow_symlinks=False)` 也可以表达相应意图，但这个参数的可用版本与底层支持应按实际环境检查。

### 10.5 目录总大小为什么需要遍历？

目录有自己的文件系统表示，占用的空间或状态大小不等于目录下所有文件大小之和。

如果你要计算逻辑文件总量，需要遍历文件并累加。是否跟随符号链接、是否对硬链接去重、是否计算稀疏文件实际占用，都会影响结果。

因此，“目录大小”不是一个只有唯一答案的问题。先明确统计口径，再选择方法。

## 第十一章：创建目录与创建空文件

### 11.1 `mkdir()` 基本语法

```python
path.mkdir(mode=0o777, parents=False, exist_ok=False)
```

| 参数 | 作用 |
| --- | --- |
| `mode` | 请求的权限模式，实际受平台和 umask 等影响 |
| `parents` | 是否创建缺失的父目录 |
| `exist_ok` | 目标已经是目录时是否允许继续 |

最常用的初始化方式：

```python
output = Path("project") / "output" / "reports"
output.mkdir(parents=True, exist_ok=True)
```

### 11.2 `parents=True` 为什么不等于忽略全部错误？

如果 `project/output` 已经是一个普通文件，它就不能同时作为目录容纳 `reports`。这种情况应该报错，因为项目布局已经与程序需求冲突。

`exist_ok=True` 处理的是目标目录已存在这一种可接受状态。权限不足、只读文件系统、父路径类型错误等仍然是实际失败。

创建中间父目录的权限规则也不一定与末级 `mode` 完全相同。需要严格权限时，应结合系统规则检查最终结果，不要只盯着调用参数。

### 11.3 `touch()`：创建空文件或更新时间

```python
marker = Path("output") / "done.flag"
marker.parent.mkdir(parents=True, exist_ok=True)
marker.touch()
```

目标不存在时，通常创建空文件。目标已存在时，默认允许存在并更新时间，不会把已有内容清空。

这与 `write_text("")` 很不同：后者会覆盖并使文件内容变空。

### 11.4 `touch(exist_ok=False)`

```python
try:
    Path("new.flag").touch(exist_ok=False)
except FileExistsError:
    print("标记已存在")
```

如果需要创建时写入完整内容，使用 `open("x")` 往往更直接。`touch()` 不自动建立缺失父目录，也不意味着任务真的成功，它只执行文件系统操作。

用一个 `.flag` 表示业务完成时，要确保你在正确时机创建它。不能先创建标记，再执行还可能失败的工作，然后让其他程序把标记误认为可靠完成信号。

## 第十二章：文本读写和二进制读写

### 12.1 `read_text()`：适合小型完整文本

```python
text = Path("notes.txt").read_text(encoding="utf-8")
```

它内部打开、读取、关闭，返回字符串。使用方便，但会把整份文本放入内存。

如果你的工作需要逐行处理大日志，就用 `path.open()` 和循环，不必先 `read_text().splitlines()` 创建额外的大列表。

### 12.2 `write_text()`：覆盖写，不是追加

```python
path = Path("output/report.txt")
path.parent.mkdir(parents=True, exist_ok=True)
count = path.write_text("报告内容\n", encoding="utf-8")
```

它返回写入字符数，已有文件被覆盖。它不会因为操作只有一行，就自动具备原子发布或失败回滚能力。

需要追加时：

```python
with path.open("a", encoding="utf-8") as target:
    target.write("追加一条内容\n")
```

### 12.3 `read_bytes()` 与 `write_bytes()`

```python
payload = Path("input.bin").read_bytes()
Path("output.bin").write_bytes(payload)
```

它们不进行字符编解码。适合规模可控的原始字节处理，不适合无条件整份读取非常大的文件。

这个例子复制内容，但不会自动保留源文件权限、修改时间和所有其他元数据。普通复制工作使用 `shutil` 更贴近需求。

### 12.4 `Path.open()` 与内置 `open()`

```python
path = Path("notes.txt")

with path.open("r", encoding="utf-8") as source:
    for line in source:
        print(line, end="")
```

普通文件读写模式、编码、缓冲和关闭规则，与内置 `open(path, ...)` 一致。

区别主要在调用入口和参数形式。内置 `open()` 还有描述符、自定义 opener 等较底层参数；`Path.open()` 用于路径对象的常规打开，不要机械假设它支持内置函数的每一个扩展参数。

### 12.5 `newline` 的版本差异

`write_text(..., newline=...)` 从 Python 3.10 起支持；`read_text(..., newline=...)` 从 Python 3.13 起支持。

兼容 Python 3.11、3.12，又希望读取时控制换行，可以这样写：

```python
with Path("notes.txt").open(encoding="utf-8", newline="") as source:
    text = source.read()
```

不要从新版文档复制一个参数，再把旧版解释器的 `TypeError` 误认为安装损坏。

## 第十三章：`iterdir()`——查看当前目录的直接子项

### 13.1 返回路径对象的迭代器

```python
folder = Path("data")

for child in folder.iterdir():
    print(child)
```

如果目录中有 `a.txt` 和 `images`，你拿到的通常是带有 `data` 前缀的子路径对象，而不是 `os.listdir()` 那样单独的名称字符串。

```python
for child in folder.iterdir():
    print(child.name)
```

这样才只输出名字。

### 13.2 不递归，不承诺排序

`iterdir()` 只查看这一层，不会自动进入 `images`。结果顺序不保证按名称排列。

需要稳定显示时：

```python
for child in sorted(folder.iterdir(), key=lambda p: p.name.casefold()):
    print(child.name)
```

`casefold()` 是字符串的无大小写比较辅助操作，不是修改真实文件名。文件系统的实际大小写规则仍然可能不同。

如果存在大小写折叠后相同的名字，还可以加入原始名称作为次级排序键，以表达明确的稳定顺序。

### 13.3 只处理普通文件

```python
for child in folder.iterdir():
    if child.is_file() and not child.is_symlink():
        print(child.name, child.stat().st_size)
```

这里每个 `child` 已经是可继续调用方法的路径对象，不需要再手动连接 `folder / child.name`。

符号链接是否包含在结果中是你的业务策略。为了说明“只统计目录内实际普通文件”的口径，这个例子排除了链接。

### 13.4 扫描时目录发生变化怎么办？

扫描结果不是一个永久快照。如果扫描后文件被删除，后面的 `stat()` 可能失败。

```python
for child in folder.iterdir():
    try:
        if child.is_file() and not child.is_symlink():
            size = child.stat().st_size
            print(child.name, size)
    except OSError as error:
        print("当前条目处理失败：", child, error)
```

外层 `iterdir()` 自身也可能因为目录不存在或权限不足失败。是否继续、是否终止，取决于完整任务要求。不要把所有错误都静默跳过后，仍宣称“扫描结果完整”。

## 第十四章：`glob()` 与 `rglob()`——按路径模式查找

### 14.1 `glob("*.txt")`

```python
folder = Path("data")

for path in folder.glob("*.txt"):
    print(path)
```

`*` 表示匹配当前路径部分中零个或多个字符；`?` 匹配一个字符；`[abc]` 匹配集合中的一个字符。这些是 glob 通配模式，不是完整正则表达式。

`.txt` 后缀匹配不保证对应条目是普通文件。需要时继续调用 `is_file()`。

### 14.2 `rglob("*.txt")`

```python
for path in folder.rglob("*.txt"):
    if path.is_file():
        print(path)
```

`r` 可以按 recursive，即递归来理解。它会在目录树中查找匹配项，意图类似 `glob("**/*.txt")`。

`**` 是特殊路径模式，匹配零层或多层目录。因此根目录直接包含的匹配文件也可以被找到。

### 14.3 大小写与隐藏文件

`pathlib` 的 glob 默认遵循平台相应的大小写规则：常见 POSIX 场景区分大小写，Windows 场景通常不区分。Python 3.12 起可使用 `case_sensitive` 显式控制相关匹配。

如果希望兼容旧版本并忽略文件扩展名大小写，可以遍历后自行比较：

```python
for path in folder.rglob("*"):
    if path.is_file() and path.suffix.casefold() == ".txt":
        print(path)
```

`pathlib` 的模式匹配不会像某些 shell 默认行为那样把所有点号开头名字都自动排除。不要想当然地把 shell、`glob` 模块和 `Path.glob()` 的边界规则视为完全相同。

### 14.4 符号链接和递归参数

递归通配默认不会无条件沿所有目录符号链接继续扩展。Python 3.13 起增加 `recurse_symlinks` 参数，让相关行为可以更明确控制。

开启跟随时，要考虑链接可能回到上级目录，造成重复访问或循环。查找个人项目文件时通常保留默认策略即可。

### 14.5 glob 适合简洁搜索，未必适合严格完整性审计

部分扫描错误会被压制，具体版本对错误处理还有调整。如果你的任务要求知道每个无法访问的目录，就应使用支持错误回调的 `Path.walk()` 或自己组织扫描。

文件数量很大时，`sorted(folder.rglob(...))` 会为了排序把所有结果收集到内存。直接迭代节省结果保存空间，却不提供全局排序。根据报告需求选择。

## 第十五章：`Path.walk()`——带剪枝与错误回调的目录树遍历

### 15.1 版本与返回结构

`Path.walk()` 从 Python 3.12 起提供。Python 3.11 可以使用 `os.walk()`，再把当前目录字符串包装成 `Path`。

```python
for root, dirnames, filenames in Path("project").walk():
    print("当前目录：", root)
    print("子目录名称：", dirnames)
    print("非目录条目名称：", filenames)
```

每一轮返回三元组：当前目录是 `Path`，后两项是名称字符串列表。继续处理文件要用 `root / filename`。

### 15.2 为什么不能直接对 `filename` 调用 `.stat()`？

```python
for root, dirs, files in Path("project").walk():
    for filename in files:
        path = root / filename
        print(path)
```

`filename` 是字符串名称，不是完整的 `Path`。`path` 才是拼接后的对象。

这与 `iterdir()` 每次直接产出子路径对象的设计不同，需要记清返回值，而不是只看它们都能“遍历文件”。

### 15.3 跳过缓存与版本控制目录

```python
skip = {".git", ".venv", "__pycache__"}

for root, dirs, files in Path("project").walk(top_down=True):
    dirs[:] = [name for name in dirs if name not in skip]

    for filename in files:
        print(root / filename)
```

`dirs[:] = ...` 原地修改扫描器持有的列表，决定后面进入哪些子目录。

如果写成 `dirs = ...`，只是让当前变量指向新列表，原列表没有改变，通常起不到剪枝效果。

`top_down=False` 时，子目录已经先处理，修改这个列表不再能阻止已经发生的递归。

### 15.4 记录错误

```python
errors = []


def record_error(error):
    errors.append((error.filename, str(error)))


for root, dirs, files in Path("project").walk(on_error=record_error):
    print(root, len(files))

print("扫描失败目录数：", len(errors))
```

`on_error` 接收扫描期间的 `OSError` 对象。你可以记录后继续，也可以在回调中重新抛出错误终止扫描。

如果报告必须完整，那么遇到权限错误还继续运行时，应该把“结果不完整”明确记录在报告中。

### 15.5 与 `os.walk()` 的一个重要差别

默认不跟随链接时，`Path.walk()` 会把指向目录的符号链接放进 `filenames`，而 `os.walk()` 的分类不完全相同。

因此，不要看到变量名叫 `files` 就假设每项一定是普通文件。需要严格分类时，继续检查真实类型和链接策略。[官方说明：Path.walk](https://docs.python.org/3.14/library/pathlib.html#pathlib.Path.walk)

### 15.6 兼容 Python 3.11 的写法

```python
import os
from pathlib import Path

for root_text, dirs, files in os.walk("project", onerror=print):
    root = Path(root_text)
    dirs[:] = [name for name in dirs if name != "__pycache__"]
    for filename in files:
        print(root / filename)
```

注意参数拼写：`Path.walk()` 使用 `top_down`、`on_error`、`follow_symlinks`；`os.walk()` 使用 `topdown`、`onerror`、`followlinks`。概念相近不代表每个关键字都相同。

## 第十六章：真正重命名、替换和删除

### 16.1 `rename()` 才会改变磁盘条目

```python
source = Path("practice") / "draft.txt"
target = source.with_name("final.txt")

new_path = source.rename(target)
```

`with_name()` 只构造目标；`rename()` 才请求文件系统执行改名。

返回值是表示目标的新路径对象。原变量 `source` 仍描述旧地址，不会自动变成新地址。

```python
print(source)
print(new_path)
```

因此，后续需要操作新文件时，要使用返回值或原先构造的目标对象。

### 16.2 相对目标以当前工作目录为基准

这是 `rename()` 中非常容易踩的坑：

```python
source = Path("data") / "draft.txt"
source.rename("final.txt")
```

目标通常是当前工作目录下的 `final.txt`，不是自动放在 `data` 中。

想保留原父目录，就明确写：

```python
source.rename(source.with_name("final.txt"))
```

路径对象知道自己描述哪个位置，但方法不会因此替你重新定义目标参数中相对路径的基准。

### 16.3 目标存在时有平台差异

Unix 系统上，目标若为允许替换的普通文件，`rename()` 可能直接覆盖；Windows 通常在目标存在时抛 `FileExistsError`。

因此，不能把 `rename()` 当成跨平台的“绝不覆盖”保证。

如果你明确要替换目标，可以使用 `replace()`。如果你明确要求创建时目标不能存在，应选择排他创建等对应方案。先检查 `exists()` 再 `rename()` 仍然有竞争窗口。

### 16.4 `replace()`：表达替换目标的意图

```python
Path("result.tmp").replace("result.txt")
```

权限和文件系统条件允许时，已有目标文件会被替换。它不是先把内容复制过去再慢慢覆盖，而是执行相应路径替换操作。

同一文件系统上的成功替换可以利用文件系统提供的原子重命名语义。跨文件系统可能失败，替换也不等于数据在任意断电情形下都已经持久保存。

非空目录、不同对象类型等情况还有额外限制。不要把这个方法想成不检查条件的“强制覆盖任何东西”。

### 16.5 `unlink()`：删除文件或符号链接

```python
Path("practice/old.txt").unlink()
```

如果目标是符号链接，删除的是链接条目，而不是自动递归删除它指向的目标。

文件不存在默认抛异常。允许它已经不存在时，可以使用：

```python
Path("practice/old.txt").unlink(missing_ok=True)
```

`missing_ok=True` 从 Python 3.8 起提供，仅忽略“目标不存在”这一类情况，不忽略权限、目录类型错误等所有失败。

它通常不会把文件送入回收站，所以练习前应确认路径。对文件处理脚本来说，正确的目标构造比写出一个删除调用更重要。

### 16.6 `rmdir()`：只删除空目录

```python
Path("practice/empty_folder").rmdir()
```

目录非空会失败。需要递归删除目录树，传统通用方式是 `shutil.rmtree()`。

```python
import shutil

shutil.rmtree(Path("practice") / "generated")
```

这会实际删除树中内容。不要对未经确认的用户输入直接调用，也不要为了“清理缓存”把整个项目目录当成目标。

### 16.7 批量修改先生成计划

重命名一百个文件时，不能只看单个 `rename()` 的例子。你还需要发现目标冲突、循环改名、大小写差异和部分成功的情况。

比较实用的流程是：扫描来源，构造目标，检查明显冲突，展示计划，再执行已确认的计划，并记录每一项结果。

下面的综合案例会使用这种思路。这里的重点是：文件操作具有真实副作用，路径变换正确只是完成任务的一部分。

## 第十七章：复制与移动，旧版本和新版本分别怎么写？

### 17.1 兼容常见版本：搭配 `shutil`

```python
import shutil
from pathlib import Path

source = Path("data") / "report.txt"
backup_dir = Path("backup")
backup_dir.mkdir(parents=True, exist_ok=True)

copy_path = Path(shutil.copy2(source, backup_dir / source.name))
```

`shutil` 的常用函数接受路径协议，因此可以直接传 `Path`，不必每次都 `str(...)`。

`copy2()` 尽量复制内容和部分元数据，但不是对所有平台元数据的完整克隆，例如所有者、ACL 和替代数据流等可能无法完全保留。

### 17.2 内容复制、元数据复制、目录树复制要区分

| 操作 | 传统常用接口 | 含义 |
| --- | --- | --- |
| 复制单文件内容 | `shutil.copyfile(src, dst)` | 目标一般作为文件路径 |
| 内容加权限模式 | `shutil.copy(src, dst)` | 目标可以是目录 |
| 尽量保留更多元数据 | `shutil.copy2(src, dst)` | 不承诺全部元数据 |
| 复制目录树 | `shutil.copytree(src, dst)` | 递归复制 |
| 跨文件系统移动 | `shutil.move(src, dst)` | 可能退化为复制后删除 |

复制后删除不是单一步骤的原子移动。中途失败时，可能留下部分目标或同时存在两个副本，因此重要资料迁移要设计验证和恢复。

### 17.3 Python 3.14 新增的路径复制方法

Python 3.14 起，`Path` 提供 `copy()` 和 `copy_into()`：

```python
# Python 3.14+
source = Path("data/report.txt")
copy = source.copy(Path("backup/report.txt"), preserve_metadata=True)
```

`copy(target)` 把目标作为复制位置；`copy_into(target_dir)` 则明确把内容复制到已有目标目录中，并保留源名称。

```python
# Python 3.14+
Path("data/report.txt").copy_into(Path("backup"))
```

这些方法还能处理目录树，并可通过参数控制符号链接与元数据策略。不要假设旧版解释器已经具有这些成员，也不要默认认为复制会拒绝覆盖目标。[版本依据：复制与移动新方法](https://docs.python.org/3.14/library/pathlib.html#copying-moving-and-deleting)

### 17.4 Python 3.14 新增的移动方法

```python
# Python 3.14+
source = Path("data/report.txt")
new_path = source.move(Path("archive/report.txt"))
```

同一文件系统时可以利用路径替换；不同文件系统时可能采用复制并删除源的方式。

`move_into(target_dir)` 表达移动到已有目录内的意图。目标存在、目录非空、源目标指向同一对象等情况都有相应限制，应按具体方法文档理解。

旧版本仍然可以通过 `shutil.move()` 完成常见移动需求。新方法让调用方式更统一，没有消除底层文件系统的限制。

### 17.5 不要用读取全文再写全文代替所有复制

```python
# 小文件教学上能复制内容，但不是通用文件复制方案。
target.write_bytes(source.read_bytes())
```

这会在内存中保存整个源文件，并覆盖目标；没有自动保留源元数据；读和写之间的失败结果也要考虑。

学习时理解这两步很有帮助，实际处理大文件或目录树时，优先使用专门的复制工具。

## 第十八章：相对路径计算和路径归属判断

### 18.1 `relative_to()`：去掉指定路径前缀

```python
from pathlib import PurePosixPath

base = PurePosixPath("/home/alice/project")
target = PurePosixPath("/home/alice/project/data/report.txt")

print(target.relative_to(base))
```

输出：

```text
data/report.txt
```

它按路径组成计算，不访问文件系统。`target` 或 `base` 不存在也不影响这种词法操作。

### 18.2 默认不是任意两点之间的导航计算

```python
outside = PurePosixPath("/home/alice/images/cat.png")
outside.relative_to(base)
```

在默认行为下，这会抛出 `ValueError`，因为 `outside` 不在 `base` 的词法子路径范围内。

如果你希望得到 `../images/cat.png` 这样的导航路径，传统上可使用 `os.path.relpath()`。

Python 3.12 起，`relative_to(..., walk_up=True)` 可以计算包含 `..` 的相对路径，但仍然是纯路径规则，不解析符号链接；不同盘符或绝对相对混合等情况也可能失败。

### 18.3 `is_relative_to()`：先问词法结构是否包含

```python
if target.is_relative_to(base):
    print(target.relative_to(base))
```

这个方法从 Python 3.9 起提供。它按路径组成判断，而不是简单字符串前缀比较，所以 `/srv/uploads-other` 不会因为文本开头相似就被当作 `/srv/uploads` 的子路径。

但是，词法包含仍不等于解析后的真实文件系统包含。

### 18.4 `..` 与符号链接会影响真实归属

```python
base = Path("uploads")
candidate = base / ".." / "private.txt"
```

没有解析的 `candidate` 表示中仍可能包含 `uploads` 这个开头组成，但实际目标可能已经离开该目录。

对于自己的稳定目录树，常见检查思路是先解析再比较：

```python
base = Path("uploads").resolve()
candidate = (base / "subdir" / "image.png").resolve()

if not candidate.is_relative_to(base):
    raise ValueError("目标不在指定目录内")
```

这可以处理许多普通路径穿越和既有链接情况。但它不是面对恶意并发替换目录时的完整沙箱，检查之后文件系统还可能变化。

### 18.5 不要使用字符串 `startswith()` 证明目录归属

```python
str(Path("/srv/uploads-other/file.txt")).startswith("/srv/uploads")
```

字符串上可以得到真，但路径并不属于 `/srv/uploads`。

路径比较应使用路径组成和明确的解析策略。即使比较方法选对了，也要说明它检查的是哪一个时刻、哪一种含义。

## 第十九章：路径相等和文件相同，是两个问题

### 19.1 `==` 比较路径表示

```python
from pathlib import PurePosixPath

print(PurePosixPath("data/report.txt") == PurePosixPath("data/report.txt"))
```

这类比较不读取磁盘，不需要文件存在。

POSIX 和 Windows 纯路径的大小写比较规则不同。Windows 路径对象按对应规则进行比较，不代表它已经向实际文件系统确认了两个名字是否指向同一个对象。

### 19.2 相对地址与绝对地址可能指向同一文件却不相等

```python
relative = Path("data/report.txt")
absolute = relative.absolute()

print(relative == absolute)
```

即使此时二者用于 I/O 指向同一文件，路径表示仍可能不同。

两个不同的符号链接、一个硬链接与原名字，也可能最终指向同一个真实文件，却具有不同路径表示。

### 19.3 `samefile()` 查询文件身份

```python
first = Path("data/report.txt")
second = Path("backup/report_link.txt")

try:
    print(first.samefile(second))
except OSError as error:
    print("无法判断文件身份：", error)
```

它使用文件系统身份信息，要求相应路径可查询，可能抛异常。适合在复制、覆盖前发现“目标其实是同一个文件”的情况。

它仍然不是加锁。判断后另一进程修改文件系统，后续操作目标仍可能变化。

### 19.4 使用 Path 作为字典键或集合元素

路径对象可哈希，因此可以放进集合、作为字典键：

```python
seen = {Path("a.txt"), Path("b.txt")}
```

这里去重的是按对象路径规则比较的表示，不是按磁盘文件身份去重。需要对硬链接去重时，通常要借助状态中的设备和文件标识，且要说明平台支持与扫描口径。

## 第二十章：符号链接、硬链接与 `readlink()`

### 20.1 符号链接保存一个目标路径

可以把符号链接理解为文件系统中的一种特殊条目，里面引用另一个路径。解析时，系统会按规则继续寻找目标。

链接目标可以存在，也可以暂时不存在；可以是绝对路径，也可以是相对路径。链接失效不代表链接条目本身一定消失。

### 20.2 `symlink_to()` 的调用方向

```python
from pathlib import Path

link = Path("practice") / "latest.txt"
link.symlink_to("report.txt")
```

这里 `link` 是要创建的链接位置，参数是链接中引用的目标。

相对目标 `report.txt` 通常相对于链接所在目录解释，所以它指向 `practice/report.txt`，而不是无条件相对于执行代码时的工作目录。

创建目录链接时，在 Windows 等平台可能需要传递 `target_is_directory=True`，还可能受开发者模式或权限限制。这个例子不是保证每台机器都能创建链接。

### 20.3 `readlink()` 返回链接记录的目标

```python
target = link.readlink()
print(target)
```

如果链接存的是相对路径，返回的也可能是相对路径。它不会自动把结果变成完全解析后的最终绝对目标。

想尽可能解析最终目标，使用 `resolve()`；想观察链接本身记录什么，使用 `readlink()`。两者用途不同。

### 20.4 `hardlink_to()` 创建另一个文件名

```python
# 需要系统和文件系统支持，并且目标普通文件已经存在。
new_name = Path("practice/report-copy-name.txt")
new_name.hardlink_to(Path("practice/report.txt"))
```

硬链接通常让两个名字引用同一个文件对象，不是复制了一份独立内容。

修改任意名字所访问的内容，另一个名字通常也看到同样变化。删除一个名字，另一个名字可能仍然可用。

硬链接往往受同文件系统等限制，普通用户通常不能随意对目录创建硬链接。备份时要确认你要的是独立副本还是同一个文件的另一个名字。

### 20.5 递归扫描为什么要明确链接策略？

如果目录 A 中的链接指回其父目录，递归跟随就可能一次又一次回到同一位置。

此外，链接可以把扫描范围引向项目目录之外。统计代码行数、打包资料或清理临时文件时，是否允许这种行为必须先决定。

个人项目常采用“不递归跟随目录链接，必要时单独记录链接”的策略。它使范围更容易理解，但不能替代对恶意并发修改目录的防护。

## 第二十一章：Windows 路径规则专门解释

### 21.1 盘符、根和锚点

```python
from pathlib import PureWindowsPath

p = PureWindowsPath(r"C:\Users\Alice\notes.txt")

print(p.drive)
print(p.root)
print(p.anchor)
print(p.parts)
```

`drive` 是 `C:`，`root` 是根分隔部分，`anchor` 组合盘符和根。`parts` 按 Windows 规则返回路径组成。

这些概念在 POSIX 路径中往往看起来重合，但 Windows 有盘符与不同根形式，因此不能只用“有没有 `/` 开头”来理解所有绝对路径。

### 21.2 `C:notes.txt` 不等于 `C:\notes.txt`

```python
relative_drive = PureWindowsPath("C:notes.txt")
absolute_drive = PureWindowsPath("C:/notes.txt")

print(relative_drive.is_absolute())
print(absolute_drive.is_absolute())
```

第一个是盘符相对路径，不是完整的根目录绝对路径。第二个包含盘符和根，才是常见完整绝对路径。

在真实 Windows 环境中，盘符相对路径的解释还涉及相应盘符的当前目录状态。为了减少歧义，应用配置和教学例子通常优先使用清楚的完整路径或普通相对路径。

### 21.3 只有根分隔符时不会自动更换盘符

```python
base = PureWindowsPath("C:/Users/Alice")
print(base / "/Windows")
```

结果使用 `C:` 盘下的根位置，而不是丢掉盘符变成 POSIX 路径。

如果后面提供另一个明确盘符路径，前面的路径组成可能被重置。跨平台拼接用户输入时，这些规则尤其值得注意。

### 21.4 UNC 网络共享路径

```python
p = PureWindowsPath(r"\\server\share\folder\report.txt")
print(p.drive)
print(p.anchor)
```

UNC 形式用服务器与共享名表达网络位置。它不需要普通 `C:` 这样的本地盘符，却可以构成 Windows 的绝对路径。

能够解析 UNC 字符串，不表示网络共享已经连接，也不表示当前账号有权限访问。实际 I/O 仍依赖网络和系统认证状态。

### 21.5 `as_posix()` 不会把 Windows 路径映射到 Linux

```python
p = PureWindowsPath(r"E:\notes\Python\lesson.md")
print(p.as_posix())
```

输出形式类似 `E:/notes/Python/lesson.md`。

这只是把路径的显示分隔符变成正斜杠，不会变成 `/mnt/e/notes/Python/lesson.md`。显示格式转换与系统路径映射必须分别处理。

### 21.6 文件 URI 是另一种表示

```python
p = Path("notes.txt").absolute()
uri = p.as_uri()
```

`as_uri()` 把绝对路径表示成 `file:` URI，适合需要文件 URI 的接口。它不是 HTTP 地址，也不自动把本地文件发布到网络。

Python 3.13 起提供 `Path.from_uri()`，但平台、authority 和 UNC 解析细节还有版本差异。普通磁盘操作继续使用 `Path`；只有对接明确要求 URI 的接口时才转换。

## 第二十二章：完整实践——建立一个可重复运行的项目目录

学完创建目录的单独调用后，我们来写一个真正能复用的初始化程序。

### 22.1 目标结构

```text
pathlib_practice/
└── mini_project/
    ├── data/
    │   ├── raw/
    │   └── processed/
    ├── logs/
    ├── output/
    └── README.md
```

要求：重复运行时不因目录已存在而失败；已有 README 不被清空；初始化根目录可以由调用者指定。

### 22.2 完整代码

```python
from pathlib import Path


def initialize_project(root):
    root = Path(root)

    directories = [
        root / "data" / "raw",
        root / "data" / "processed",
        root / "logs",
        root / "output",
    ]

    for directory in directories:
        directory.mkdir(parents=True, exist_ok=True)

    readme = root / "README.md"

    try:
        with readme.open("x", encoding="utf-8", newline="\n") as target:
            target.write("# Mini Project\n\n")
            target.write("这是一个用于学习路径与文件操作的项目。\n")
    except FileExistsError:
        print("保留已有说明文件：", readme)

    return root.absolute()


project = initialize_project(Path("pathlib_practice") / "mini_project")
print("项目目录：", project)
```

### 22.3 为什么目录和 README 使用不同策略？

目录已存在通常符合初始化目标，所以用 `exist_ok=True`。

README 可能已经由用户写了内容，不能每次初始化都 `write_text()` 覆盖。这里用 `x` 表示“只有尚不存在时才创建”，把不覆盖的意图交给实际打开步骤。

捕获 `FileExistsError` 后只说明这个名字已经存在，不代表它一定是我们期望的普通 README 文件。更严格的项目初始化器还可以检查类型，发现同名目录时报告布局冲突。

### 22.4 为什么返回根目录绝对形式？

调用者后续打印或使用结果时，绝对形式更容易看清实际生成位置。

这里采用 `absolute()`，表达以当前工作目录为基准固定位置，而没有额外声明必须解析所有链接。若项目约定需要真实目标位置，可以选择 `resolve()`，但应知道其语义差别。

### 22.5 如何验证程序行为？

先运行一次，观察目录是否创建。再向 README 手动加入一句话，第二次运行，确认已有内容仍然存在。

然后在新的练习根目录中，提前创建一个普通文件 `data`，再次初始化。程序应报告路径类型冲突，而不是静默宣称初始化成功。

这个实验告诉你：可重复运行不是“所有异常都忽略”，而是明确区分允许重复出现的状态和真正阻止任务的状态。

## 第二十三章：完整实践——生成目录文件清单

### 23.1 我们要什么结果？

扫描指定目录树，记录普通文件的相对路径、字节大小和 UTC 修改时间。跳过 `.git`、`.venv` 和 Python 缓存目录，跳过符号链接，并记录扫描错误。

输出采用 CSV，因为文件名可能包含逗号，用手工字符串拼接容易破坏字段边界。

### 23.2 实现，要求 Python 3.12+

```python
import csv
from datetime import datetime, timezone
from pathlib import Path


def create_inventory(root, output_path):
    root = Path(root).resolve(strict=True)
    output_path = Path(output_path).absolute()

    if not root.is_dir():
        raise NotADirectoryError(root)

    skipped_dirs = {".git", ".venv", "__pycache__"}
    errors = []
    rows = []

    def handle_scan_error(error):
        errors.append((str(error.filename), str(error)))

    for current, dirs, filenames in root.walk(on_error=handle_scan_error):
        dirs[:] = [name for name in dirs if name not in skipped_dirs]

        for filename in filenames:
            path = current / filename

            try:
                if path.is_symlink() or not path.is_file():
                    continue

                if path.resolve() == output_path.resolve():
                    continue

                info = path.stat()
                modified = datetime.fromtimestamp(
                    info.st_mtime,
                    tz=timezone.utc,
                ).isoformat()

                rows.append((
                    path.relative_to(root).as_posix(),
                    info.st_size,
                    modified,
                ))
            except OSError as error:
                errors.append((str(path), str(error)))

    rows.sort(key=lambda row: row[0])
    output_path.parent.mkdir(parents=True, exist_ok=True)

    with output_path.open("w", encoding="utf-8", newline="") as target:
        writer = csv.writer(target)
        writer.writerow(["relative_path", "size_bytes", "modified_utc"])
        writer.writerows(rows)

    return len(rows), errors


count, errors = create_inventory("pathlib_practice", "reports/inventory.csv")
print("记录文件数：", count)
print("错误数量：", len(errors))
for path, message in errors:
    print("未能完整处理：", path, message)
```

### 23.3 为什么使用相对路径？

报告可能要复制给别人。`/home/alice/...` 这样的完整地址属于本机环境，而相对项目根目录的路径更容易迁移和比较。

`path.relative_to(root)` 根据扫描根计算相对表示。扫描对象原本就是从 `root` 向下构造，因此普通稳定目录树中具有清楚的前缀关系。

`as_posix()` 在这里用于让报告统一使用 `/`，便于显示和版本比较。它不会把这些字符串自动变成另一台机器上的有效绝对地址。

### 23.4 为什么输出文件应尽量放在扫描目录外？

如果报告也放在输入目录内，下一次扫描就可能把上一份报告统计进来。这个示例会跳过当前输出路径，但别名、其他历史报告和更复杂的包含关系仍然需要你定义策略。

最清楚的布局通常是让扫描输入与生成输出分开。这样报告的含义不会因为运行次数变化而悄悄改变。

### 23.5 为什么先收集再排序？

需要全局按相对路径排序，就必须保存结果或使用外部排序。这里为教学清晰，使用列表。

对数百万文件，可以边扫描边写，代价是放弃全局排序；或者将中间记录写入数据库、分块排序后合并。不能同时声称“不保存全部结果”和“随意进行全局排序”而不解释额外机制。

### 23.6 错误数量为零，也不是永恒精确快照

文件可能在扫描过程中变化。两个不同时间读取到的状态不能自动形成一个瞬间的全局视图。

这个程序生成的是扫描期间成功观察到的清单。需要文件系统快照或严格审计一致性时，应使用相应平台机制或业务锁定流程。

## 第二十四章：完整实践——按扩展名统计数量和大小

### 24.1 先定义统计口径

我们统计非符号链接的普通文件。硬链接的不同名称分别计数，因为本例关注目录条目，不尝试计算实际物理空间去重。

扩展名统一为小写；没有后缀的文件归入 `[无扩展名]`。`.tar.gz` 在本例中按最后后缀 `.gz` 统计，复合格式可以在后续改进中单独识别。

### 24.2 完整代码

```python
from collections import Counter, defaultdict
from pathlib import Path


def summarize_extensions(root):
    root = Path(root)
    counts = Counter()
    sizes = defaultdict(int)
    errors = []

    for path in root.rglob("*"):
        try:
            if path.is_symlink() or not path.is_file():
                continue

            info = path.stat()
            suffix = path.suffix.casefold() or "[无扩展名]"

            counts[suffix] += 1
            sizes[suffix] += info.st_size
        except OSError as error:
            errors.append((path, error))

    return counts, sizes, errors


counts, sizes, errors = summarize_extensions("pathlib_practice")

for suffix, count in counts.most_common():
    print(f"{suffix:16} {count:6} 个 {sizes[suffix]:12} 字节")

print("未处理成功的条目数：", len(errors))
```

### 24.3 逐句理解组合方式

`Counter()` 是专门计数的容器，访问缺失计数时能自然从零开始，因此适合 `counts[suffix] += 1`。

`defaultdict(int)` 在缺失键时调用 `int()` 得到零，适合累计字节数量。这里两个容器分别表达“数量”和“大小”，比一个混合含义的字典更容易阅读。

`path.suffix.casefold() or "[无扩展名]"` 先取得最后后缀，再做无大小写归一；空后缀在布尔判断中为假，于是使用分类标签。

### 24.4 这个简洁版本有什么限制？

`rglob()` 的扫描错误不都通过上面循环体里的 `try` 暴露，版本也有相应错误压制行为。列表 `errors` 只保证记录循环体中实际抛出的错误，不能证明扫描目录完全无遗漏。

如果统计必须列出每一个扫描失败目录，应改用 `walk(on_error=...)`。这也是我们同时讲 `rglob()` 和 `walk()` 的原因：它们接口简洁程度和可控制性不同。

### 24.5 一个值得自己完成的改进

把 `.tar.gz`、`.tar.bz2`、`.tar.xz` 识别为完整业务后缀，同时保留像 `report.final.txt` 的正常 `.txt` 分类。

可以按明确的复合后缀列表优先匹配，再回退到 `suffix`。不要直接拼接全部 `suffixes`，因为文件名中的版本号或说明词也会被当成后缀组成。

## 第二十五章：完整实践——批量重命名，先预览再执行

### 25.1 需求与边界

把某个练习目录中第一层的 `.txt` 文件，改成带 `archived_` 前缀的名字。已带前缀的文件跳过。任何明显的目标冲突都先报告，不自动覆盖。

示例只面向你控制的本地静态练习目录。它不提供面对恶意并发修改者的跨平台排他重命名保证。

### 25.2 生成计划

```python
from pathlib import Path


def make_rename_plan(folder):
    folder = Path(folder)
    plan = []
    destinations = set()

    for source in sorted(folder.iterdir(), key=lambda path: path.name):
        if source.is_symlink() or not source.is_file():
            continue
        if source.suffix.casefold() != ".txt":
            continue
        if source.name.startswith("archived_"):
            continue

        destination = source.with_name("archived_" + source.name)

        if destination in destinations:
            raise ValueError(f"多个输入产生同一目标：{destination}")

        if destination.exists() or destination.is_symlink():
            raise FileExistsError(destination)

        destinations.add(destination)
        plan.append((source, destination))

    return plan
```

### 25.3 预览与执行

```python
def execute_rename_plan(plan, dry_run=True):
    for source, destination in plan:
        if dry_run:
            print("计划重命名：", source, "->", destination)
        else:
            result = source.rename(destination)
            print("已重命名为：", result)


plan = make_rename_plan("pathlib_practice/rename_demo")
execute_rename_plan(plan, dry_run=True)
```

先在练习目录检查计划，明确要执行时再传 `dry_run=False`。这个参数是程序自己设计的，没有全 Python 通用的自动预览机制。

### 25.4 为什么也检查 `is_symlink()`？

默认 `exists()` 对断开目标的链接可能返回 `False`。如果目标名字上存在一个失效链接，依旧应把它看成已被占用，而不是随意覆盖。

旧版本兼容情况下，`destination.exists() or destination.is_symlink()` 是个人脚本里常见的检查组合。需要更严格错误辨别时，应进一步使用 `lstat()` 并处理异常。

### 25.5 为什么检查后仍要承认覆盖风险？

目标检查发生在重命名之前。两者之间，另一个程序可能创建目标。Unix 的 `rename()` 又可能覆盖允许替换的普通文件。

因此，预检查提高了本地脚本的可审阅性和错误发现能力，却不能把两步变成一个不可分割的“不覆盖事务”。真实共享目录需要更明确的平台能力、协调或锁定策略。

### 25.6 批量操作中途失败会怎样？

如果前五个文件已成功改名，第六个因权限失败，前五个不会自动改回原名。

你可以记录执行结果和逆向计划，但回滚本身也可能失败。更复杂的改名工具应把计划、执行和恢复记录作为明确功能，而不是在一个 `except` 中假设所有事情都能撤回。

### 25.7 大小写改名的特殊情况

在常见不区分大小写的文件系统上，把 `Report.txt` 改成 `report.txt` 可能需要额外考虑。批量生成目标时，两个不同源名称也可能在目标系统的比较规则下冲突。

教程的普通路径集合去重不能代替对目标文件系统规则的完整理解。跨平台迁移尤其应提前检查保留名称、大小写冲突和末尾点空格等限制。

## 第二十六章：完整实践——整理图片副本，保留原始输入

### 26.1 为什么先做复制版整理？

学习批量文件管理时，先生成分类副本比较容易检查结果。原文件仍在，便于比较，也便于修正分类规则。

这不是说复制没有风险：目标覆盖、磁盘容量不足和中途失败仍然可能发生。因此，输出应放在独立练习目录，采用明确不覆盖的创建策略。

### 26.2 完整实现

```python
import shutil
from pathlib import Path


def copy_images(source_dir, target_dir):
    source_dir = Path(source_dir).resolve(strict=True)
    target_dir = Path(target_dir).absolute()
    accepted = {".jpg", ".jpeg", ".png", ".gif", ".webp"}

    if not source_dir.is_dir():
        raise NotADirectoryError(source_dir)

    target_dir.mkdir(parents=True, exist_ok=True)
    copied = []
    skipped = []

    for source in sorted(source_dir.iterdir(), key=lambda path: path.name):
        if source.is_symlink() or not source.is_file():
            continue

        suffix = source.suffix.casefold()
        if suffix not in accepted:
            continue

        category = suffix.removeprefix(".")
        destination_dir = target_dir / category
        destination_dir.mkdir(parents=True, exist_ok=True)
        destination = destination_dir / source.name

        try:
            with source.open("rb") as input_file:
                with destination.open("xb") as output_file:
                    shutil.copyfileobj(input_file, output_file, length=1024 * 1024)
        except FileExistsError:
            skipped.append(destination)
        else:
            copied.append(destination)

    return copied, skipped


copied, skipped = copy_images(
    "pathlib_practice/source_images",
    "pathlib_practice/sorted_images",
)
print("复制成功：", len(copied))
print("因目标已存在而跳过：", len(skipped))
```

### 26.3 按扩展名分类不验证图片真实格式

一个名为 `cat.jpg` 的文件，可能根本不是 JPEG。这个脚本处理的是命名分类，不执行图像解码验证。

如果任务要求检查真实图片格式或尺寸，应使用图像库进行内容解析，并考虑损坏文件和资源限制。路径库本身不会根据扩展名验证每个字节。

### 26.4 为什么使用 `xb`？

目标创建时要求名字尚不存在。相比预先 `exists()` 再 `wb`，排他创建更直接表达“不覆盖已有目标”的要求。

但若复制过程中磁盘写满，已经创建的目标可能只是一部分内容。要实现“成功后才出现最终名字”，可以先写目标同目录临时文件，再用适合目标政策的发布机制。简单 `replace()` 会覆盖已有目标，不能未经说明就用于严格不覆盖发布。

### 26.5 这个脚本复制什么，不复制什么？

它通过流复制文件内容，不自动复制全部源元数据。目标文件的权限、时间等可能与源不同。

如果你希望尽量保留源元数据，`copy2()` 是常见选择，但又要重新明确覆盖策略。选择工具时，应把“内容正确”“元数据保留”“目标不覆盖”“发布原子性”分别考虑。

## 第二十七章：完整实践——查找最近修改的 Python 源文件

### 27.1 明确“最近”的时间含义

我们查找修改时间戳在当前时刻前七天以内的 `.py` 普通文件，排除符号链接，按修改时间从新到旧输出。

这里的“七天”是七乘二十四小时的经过时长，不是按某个地区日历从七个日期前的零点开始。

### 27.2 实现

```python
import time
from datetime import datetime, timezone
from pathlib import Path


def recently_modified_python(root, days=7):
    if days < 0:
        raise ValueError("days 不能为负数")

    cutoff = time.time() - days * 24 * 60 * 60
    results = []
    errors = []

    for path in Path(root).rglob("*.py"):
        try:
            if path.is_symlink() or not path.is_file():
                continue
            info = path.stat()
        except OSError as error:
            errors.append((path, error))
            continue

        if info.st_mtime >= cutoff:
            results.append((info.st_mtime, path))

    results.sort(key=lambda item: (-item[0], str(item[1])))
    return results, errors


results, errors = recently_modified_python("pathlib_practice", days=7)

for timestamp, path in results:
    when = datetime.fromtimestamp(timestamp, tz=timezone.utc)
    print(when.isoformat(), path)

print("条目访问错误：", len(errors))
```

### 27.3 为什么不用 `perf_counter()` 算文件日期？

`perf_counter()` 适合测量经过时长，起点不用于对应文件系统的 Unix 时间戳。

文件的 `st_mtime` 与 `time.time()` 属于可对应的墙上时钟时间戳体系，所以这里使用后者生成截止点。

测量扫描耗时则应使用 `perf_counter()`，避免系统时间调整直接影响耗时差值。这两个时间问题需要不同工具。

### 27.4 文件时间不等于真实业务历史

文件修改时间可以被复制工具保留，也可以被主动修改。它不一定等于作者最后编辑内容的业务时间，更不能单独充当可信审计证明。

这个脚本适合个人项目的快速观察。版本历史应参考版本控制系统，审计需求应建立更完整的记录机制。

## 第二十八章：完整实践——只接受受控根目录下的文件

这个例子用于理解路径归属检查的思路，应用场景是你控制的稳定本地目录。对外提供文件下载接口时，还需要结合权限、文件类型和并发变化等因素。

### 28.1 常见错误写法

```python
base = Path("public")
user_input = "../../private.txt"
path = base / user_input
```

路径连接不会自动拒绝 `..`，也不会自动拒绝用户提供新的绝对路径。

### 28.2 在稳定目录中的基本限制

```python
from pathlib import Path


def locate_existing_file(base, relative_name):
    base = Path(base).resolve(strict=True)
    requested = Path(relative_name)

    if requested.is_absolute():
        raise ValueError("这里只接受相对路径")

    candidate = (base / requested).resolve(strict=True)

    if not candidate.is_relative_to(base):
        raise ValueError("目标超出允许的目录")

    if not candidate.is_file():
        raise ValueError("目标不是普通文件")

    return candidate
```

### 28.3 为什么先拒绝绝对路径，还要解析后检查？

拒绝绝对路径只约束了一种表达形式，相对路径仍可以通过 `..` 离开根目录，或通过符号链接指向外部。

解析后比较能发现许多这类普通情况。两个检查分别表达输入格式约定和解析后的归属约定。

### 28.4 为什么不直接把返回路径看成永久授权？

检查后，目标可以被替换。对不可信用户可修改的目录，攻击者可能在检查和打开之间改变链接或目录条目。

因此，这个函数的返回值表示“检查时符合这些条件的路径”，不是不可伪造的文件能力句柄。真正的安全文件服务要结合操作系统支持、资源所有权与访问控制进行设计。

### 28.5 平台语法必须一致

代码使用当前平台 `Path` 规则解释输入。在 Linux 上，反斜杠通常不是路径分隔符；在 Windows 上，它参与路径语法。

如果协议允许用户上传另一平台的路径字符串，应先明确协议使用什么格式，再用对应纯路径规则分析。不要先按一种平台验证，再按另一种平台执行 I/O。

## 第二十九章：容易混淆的语法和返回值集中辨析

### 29.1 `Path` 与 `str` 可以转换，但不必到处转换

```python
path = Path("data/report.txt")
text_path = str(path)
```

`str(path)` 取得当前平台风格的字符串表示，适合显示或传给只接受字符串的旧接口。

现代 Python 的许多文件系统接口接受 `os.PathLike`，可以直接传 `Path`。例如内置 `open()`、多数 `os` 路径操作和 `shutil` 常用函数。

另一方面，`sys.path` 文档要求使用字符串条目，因此向模块搜索路径添加位置时，通常仍应明确 `str(directory)`。不要因为两个地方都接收“路径”，就假设它们拥有完全相同的协议。

### 29.2 `as_posix()` 与 `as_uri()` 不同

`as_posix()` 主要改变路径字符串的分隔符显示形式；`as_uri()` 构造文件 URI，要求绝对路径，并执行对应 URI 表示规则。

需要保存项目相对路径清单，常用前者；需要把本地绝对位置交给明确要求文件 URI 的接口，才使用后者。

两者都不会自动上传文件，不会让另一台电脑凭这个地址就访问你的磁盘。

### 29.3 名称后缀与真实格式不同

`path.suffix` 是名字分析。重命名 `.txt` 为 `.json` 只改名字，不会自动产生合法 JSON；重命名 `.jpg` 为 `.png` 不执行图片转码。

你应该根据目标格式重新生成内容，并在需要时同时选择新名字。路径处理解决“地址怎么写”，格式处理解决“字节怎么组织”。

### 29.4 `match()` 与 `glob()` 不同

```python
from pathlib import PurePosixPath

path = PurePosixPath("src/tools/main.py")
print(path.match("*.py"))
```

`match()` 对已有路径表示进行模式判断，不扫描磁盘。相对模式的匹配规则也不一定等价于把模式从路径开头完整匹配。

`glob()` 则从某个目录出发查找文件系统条目。一个返回布尔值，一个产生匹配路径，不能只因都使用通配符就混为同一个操作。

较新版本还有 `full_match()`，用于完整路径模式匹配，其 `**` 规则与旧 `match()` 有区别。普通初学阶段先掌握 `glob()`、`rglob()` 和后缀筛选，遇到纯路径模式判断再读对应版本说明。

### 29.5 `Path.info` 是较新的缓存信息接口

Python 3.14 起提供 `Path.info`。它可以利用某些目录扫描已有信息减少类型查询开销，但信息具有缓存性质。

需要获取当前状态时，直接使用 `path.is_file()`、`path.is_dir()` 或 `stat()` 更容易表达重新查询的意图。

不要把一个缓存结果当成长期不变的事实，也不要把它与 `os.DirEntry` 的行为简单完全等同。普通脚本不必为了追求“最新接口”而主动改用缓存信息。

### 29.6 `chmod()` 修改权限，不修改文本内容

```python
# 主要用于支持 Unix 权限语义的平台。
Path("practice/script.sh").chmod(0o755)
```

这个八进制模式与所有者、组和其他用户的权限有关，受到平台和权限模型限制。

它不会让无效 shell 文本变成正确脚本，也不会修复错误的换行或解释器路径。文件“可执行”和文件“内容能正确执行”仍是不同条件。

Windows 中的相关权限处理较有限，不等价于 Unix 完整权限模型。跨平台授权管理应使用适合平台的机制。

### 29.7 路径对象可排序，不代表排序符合人类数字习惯

`chapter10.txt` 和 `chapter2.txt` 的字符串顺序，可能与你预期的自然数字顺序不同。

如果文件名由你生成，可以统一补零，例如 `chapter02.txt`；如果必须处理外部名称，就要明确自然排序规则，并选择相应实现。

同样，`sorted()` 会收集结果，不再是纯粹一边扫描一边处理。对巨型目录，应考虑内存与排序需求之间的取舍。

## 第三十章：版本差异表，以及学习时应该怎样核对

| 功能或行为 | 相关版本 | 学习建议 |
| --- | --- | --- |
| `pathlib` 进入标准库 | 3.4 | 现代 Python 可直接导入 |
| `unlink(missing_ok=...)` | 3.8+ | 兼容更老版本时自行捕获不存在异常 |
| `is_relative_to()`、`with_stem()` | 3.9+ | 本文基础环境可使用 |
| `write_text(newline=...)` | 3.10+ | 输出稳定 LF 时很方便 |
| `Path.walk()` | 3.12+ | 3.11 使用 `os.walk()` |
| glob 的 `case_sensitive` | 3.12+ | 旧版可后置筛选后缀 |
| `relative_to(walk_up=True)` | 3.12+ | 默认行为仍要求词法子路径 |
| `exists(follow_symlinks=...)` | 3.12+ | 旧版结合链接判断或 lstat |
| `is_file/is_dir(follow_symlinks=...)` | 3.13+ | 不要误用于 3.12 |
| `read_text(newline=...)` | 3.13+ | 旧版改用 `open()` |
| glob 的 `recurse_symlinks` | 3.13+ | 默认递归与链接规则需单独理解 |
| `Path.from_uri()`、`full_match()` | 3.13+ | 对接对应需求时再学习 |
| `Path.copy/move` 及 into 变体 | 3.14+ | 旧版搭配 shutil |
| 单个点号作为有效后缀 | 3.14 行为调整 | 特殊文件名注意版本 |
| 类型查询统一压制 OSError | 3.14 行为调整 | 需要原因用 stat 保留异常 |
| `Path.info` | 3.14+ | 注意缓存和新鲜度 |

版本表帮助你知道某个写法能否在目标解释器上运行，不意味着所有旧接口都应该升级替换。项目已经稳定使用 `shutil.copy2()`，并不会因为出现 `Path.copy()` 就自动变得错误。

查看当前解释器版本可用 `sys.version_info`，不要靠字符串比较版本大小。环境里同时安装多个 Python 时，还应确认正在运行的 `sys.executable`。

## 第三十一章：带参考答案的动手练习

### 31.1 练习一：生成路径，不访问文件系统

要求：得到 `project/data/report.csv`，打印文件名、父目录、主体和后缀，不创建任何目录。

```python
from pathlib import Path

path = Path("project") / "data" / "report.csv"

print(path.name)
print(path.parent)
print(path.stem)
print(path.suffix)
```

这里的方法和属性只分析路径组成。即使电脑上没有 `project`，仍可得到相应结果。把这一步与 `mkdir()`、`read_text()` 的真实 I/O 区分，是第一项练习的重点。

### 31.2 练习二：修改目标名字，保留原父目录

要求：把 `data/report.txt` 的目标路径改成 `data/report_backup.txt`，先只生成计划。

```python
path = Path("data/report.txt")
backup = path.with_stem(path.stem + "_backup")

print(path, "->", backup)
```

这段代码不会移动真实文件。若你下一步决定重命名，应调用 `path.rename(backup)`，并提前确认覆盖策略与真实目录状态。

### 31.3 练习三：只列出第一层文件

```python
folder = Path("pathlib_practice")

for child in sorted(folder.iterdir(), key=lambda path: path.name):
    if child.is_file() and not child.is_symlink():
        print(child.name)
```

没有递归进入子目录。把 `iterdir()` 换成 `rglob("*")` 会改变扫描范围，所以它不是纯粹换一种拼写。

### 31.4 练习四：递归查找并显示项目相对路径

```python
root = Path("pathlib_practice").resolve(strict=True)

for path in sorted(root.rglob("*.py")):
    if path.is_file() and not path.is_symlink():
        print(path.relative_to(root).as_posix())
```

这里先把根目录解析为明确绝对位置，再从它构造后续路径，避免把不同基准的路径直接比较。

`sorted()` 适合结果规模有限的练习目录。如果结果极大，应考虑是否真的需要全局排序。

### 31.5 练习五：区分失效链接和缺失名字

要求：在支持符号链接的平台创建一个指向缺失目标的链接，观察 `exists()` 和 `is_symlink()`。

```python
from tempfile import TemporaryDirectory
from pathlib import Path

with TemporaryDirectory() as temp_dir:
    root = Path(temp_dir)
    link = root / "link.txt"
    link.symlink_to("missing.txt")

    print(link.exists())
    print(link.is_symlink())
```

在支持并允许创建链接的环境中，通常分别得到 `False` 和 `True`。如果平台权限不允许创建链接，失败应在创建步骤被报告，不能把它误解释为路径库不认识文件。

### 31.6 练习六：证明路径拼接不提供目录边界

```python
from pathlib import PurePosixPath

base = PurePosixPath("/srv/uploads")

print(base / "image.png")
print(base / "/etc/config")
print(base / ".." / "private.txt")
```

观察后两项为何不能直接视为安全上传目标。第二项使用新的绝对位置；第三项包含上级目录引用。`PurePosixPath` 不执行真实解析，正适合用来观察词法规则。

### 31.7 练习七：记录目录扫描错误

使用 `Path.walk(on_error=...)` 扫描你自己的练习目录，把错误保存到列表。不要通过修改系统目录权限来制造练习环境；可以传入一个不存在的根目录，或在你有明确控制权的临时目录中设置情形。

```python
# Python 3.12+
errors = []

for root, dirs, files in Path("a_directory_that_does_not_exist").walk(
    on_error=errors.append,
):
    print(root, files)

for error in errors:
    print(type(error).__name__, error.filename)
```

这一练习观察错误回调，并不要求每种平台错误字符串完全相同。跨平台测试应优先比较错误类型和相关行为，而不是硬编码整段系统报错文本。

### 31.8 练习八：判断两个名字是否指向同一文件

在支持硬链接的平台，创建临时普通文件和硬链接，然后比较路径相等与 `samefile()`。

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as temp_dir:
    root = Path(temp_dir)
    first = root / "first.txt"
    second = root / "second.txt"

    first.write_text("同一个内容对象\n", encoding="utf-8")
    second.hardlink_to(first)

    print(first == second)
    print(first.samefile(second))
```

通常分别为 `False` 和 `True`。前者比较路径表示，后者查询文件身份。这个区别对避免自我覆盖和理解硬链接尤其重要。

## 第三十二章：常用方法速查表

| 类别 | 接口 | 返回或效果 | 访问文件系统吗 |
| --- | --- | --- | --- |
| 构造 | `Path("a") / "b"` | 新路径对象 | 通常不访问 |
| 拼接 | `p.joinpath(...)` | 新路径对象 | 不访问 |
| 文件名 | `p.name` | 字符串 | 不访问 |
| 主体 | `p.stem` | 去最后后缀的名称 | 不访问 |
| 后缀 | `p.suffix` | 字符串 | 不访问 |
| 多后缀 | `p.suffixes` | 列表 | 不访问 |
| 父路径 | `p.parent`、`p.parents` | 路径或祖先序列 | 不访问 |
| 改名计划 | `p.with_name(...)` | 新路径对象 | 不访问 |
| 后缀计划 | `p.with_suffix(...)` | 新路径对象 | 不访问 |
| 工作目录 | `Path.cwd()` | 当前工作目录路径 | 查询进程环境 |
| 用户目录 | `Path.home()` | 主目录路径 | 查询相应系统信息 |
| 补绝对形式 | `p.absolute()` | 新路径对象 | 需要当前目录信息 |
| 解析实际路径 | `p.resolve()` | 新路径对象 | 会解析文件系统 |
| 词法相对路径 | `p.relative_to(base)` | 新路径对象 | 不访问 |
| 词法包含 | `p.is_relative_to(base)` | 布尔值 | 不访问 |
| 存在性 | `p.exists()` | 布尔值 | 查询 |
| 类型 | `p.is_file/is_dir/is_symlink()` | 布尔值 | 查询 |
| 身份 | `p.samefile(other)` | 布尔值 | 查询 |
| 状态 | `p.stat()`、`p.lstat()` | 状态对象 | 查询 |
| 创建目录 | `p.mkdir(...)` | 创建请求 | 修改 |
| 空文件或更新时间 | `p.touch(...)` | 创建或时间更新 | 修改 |
| 文本读取 | `p.read_text(...)` | `str` | 读取 |
| 文本写入 | `p.write_text(...)` | 字符数量 | 修改并可能覆盖 |
| 字节读取 | `p.read_bytes()` | `bytes` | 读取 |
| 字节写入 | `p.write_bytes(...)` | 字节数量 | 修改并可能覆盖 |
| 打开流 | `p.open(...)` | 文件对象 | 按模式读写 |
| 第一层扫描 | `p.iterdir()` | 子路径迭代器 | 扫描 |
| 模式扫描 | `p.glob(...)`、`p.rglob(...)` | 匹配路径迭代器 | 扫描 |
| 目录树扫描 | `p.walk(...)` | 三元组迭代器 | 扫描，3.12+ |
| 真正改名 | `p.rename(target)` | 目标路径对象 | 修改 |
| 替换 | `p.replace(target)` | 目标路径对象 | 修改 |
| 删除文件或链接 | `p.unlink(...)` | 删除请求 | 修改 |
| 删除空目录 | `p.rmdir()` | 删除请求 | 修改 |
| 复制、移动 | `p.copy/move(...)` | 目标路径对象 | 修改，3.14+ |

学习顺序可以从“构造和拆分”开始，再学“读写和创建”，最后学“扫描、批量操作与链接”。遇到复杂问题时，先问自己是在计算地址，还是查询当前状态，还是修改真实文件系统；再看返回值是什么，是否需要进一步拼接或调用。

配套阅读：[Python-open文件处理.md](Python-open文件处理.md)。路径对象只是让常见文件系统操作更容易组织，打开模式、编码、缓冲和关闭规则仍然需要结合文件 I/O 一起理解。

官方参考：[pathlib 接口文档](https://docs.python.org/3.14/library/pathlib.html)、[os 路径协议](https://docs.python.org/3.14/library/os.html#os.PathLike)、[shutil 文件复制](https://docs.python.org/3.14/library/shutil.html)。本篇采用独立教学例子解释常见接口，平台与版本特殊行为应以目标环境对应文档为准。
