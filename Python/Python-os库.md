# Python `os` 库超详细教程：从基础原理到实际应用

适合 Python 初学者到进阶学习者 · 覆盖常用函数、语法、参数、返回值、原理、组合用法及易错点

Python 的 `os` 是一个非常重要的标准库模块。只要你开始接触文件管理、自动化脚本、批量处理数据、项目开发或者服务器程序，基本都会遇到它。

不过，`os` 的函数比较多，而且有些函数的名字很相似，比如：

- `os.mkdir()` 和 `os.makedirs()` 有什么区别？
- `os.remove()`、`os.rmdir()` 和 `os.removedirs()` 分别删除什么？
- `os.path.join()` 为什么比直接使用字符串拼接更合适？
- `os.getcwd()`、`os.chdir()`、`os.path.abspath()` 到底是什么关系？
- `os.listdir()`、`os.scandir()`、`os.walk()` 为什么都能遍历文件，却需要三个不同的函数？
- `os.system()` 和 `subprocess.run()` 到底应该用谁？

这些问题的关键不在于记住函数名，而在于理解它们背后的设计逻辑。

我们从最基础的概念出发，逐步讲清楚。

## 第一章：`os` 究竟是什么？为什么需要它？

### 1.1 `os` 的含义与由来

`os` 是 Operating System（操作系统） 的缩写。

我们平时使用电脑时，经常会做下面这些事情：

- 创建一个文件夹。
- 查看某个目录下有哪些文件。
- 将一个文件重命名。
- 删除不需要的文件。
- 查询文件大小、修改时间。
- 获取当前程序运行的位置。
- 读取系统环境变量。
- 启动其他程序。

这些行为，最终都需要操作系统参与。

例如，在 Windows 中，你可以使用文件资源管理器新建文件夹；在 Linux 中，你可以通过 `mkdir` 命令创建目录。

但是 Python 代码不能依靠人去点击文件管理器。

因此，Python 提供了 `os` 模块，帮助程序通过统一的接口调用操作系统提供的功能。

理解 `os` 的工作位置

```text
你编写的 Python 程序
例如：创建文件夹、列出目录、获取文件属性
        │
        ▼
调用 Python 的 `os` 模块
将 Python 调用转换为相应的系统操作接口
        │
        ▼
请求操作系统服务
        │
        ├── Windows
        │   Windows 系统接口
        │
        └── Linux / macOS
            POSIX / Unix 系统接口
```

例如：

```python
import os

os.mkdir("test")
```

这段代码的意思是：请求操作系统在指定位置创建名为 `test` 的文件夹。

在 Windows、Linux 和 macOS 上，代码的基本写法一致，但 Python 内部调用的系统机制可能不同。

这就是 `os` 的一项重要设计目的：尽可能提供跨操作系统的统一接口。

不过要注意：并不是 `os` 的所有函数都能在所有操作系统上使用。部分底层函数只支持 Unix 或 Windows。

### 1.2 `os` 是标准库，不需要安装

直接导入即可：

```python
import os
```

不需要：

```bash
pip install os
```

因为它已经包含在正常安装的 Python 标准库中。

导入后使用：

```python
os.getcwd()
os.listdir()
os.mkdir("demo")
```

这里的语法可以拆成三部分：

```text
os.getcwd()
│    │    │
│    │    └── 圆括号：调用函数
│    └─────── 函数名称
└──────────── 模块名称
```

准确地说，`getcwd()` 是 `os` 模块提供的函数，而不是某个对象实例的方法。日常口语里经常把它们统称为“方法”，但学习 Python 时最好知道这个区别。

### 1.3 `os` 和 `os.path` 是什么关系？

这是理解整个库最重要的基础之一。

```python
import os

os.getcwd()            # 获取当前工作目录
os.mkdir("test")        # 创建目录

os.path.exists("test")  # 判断路径是否存在
os.path.join("a", "b")   # 拼接路径
```

注意：

```python
os.getcwd()
```

和：

```python
os.path.exists()
```

它们的调用层级不同。

`os` 负责许多与操作系统直接交互的操作，而 `os.path` 主要负责路径的表示、判断、拼接和处理。

可以这样记：

| 功能             | 常用位置             |
| ---------------- | -------------------- |
| 创建文件夹       | `os.mkdir()`         |
| 删除文件         | `os.remove()`        |
| 查看目录内容     | `os.listdir()`       |
| 获取当前工作目录 | `os.getcwd()`        |
| 拼接路径         | `os.path.join()`     |
| 判断文件是否存在 | `os.path.exists()`   |
| 获取文件扩展名   | `os.path.splitext()` |

有一个特别重要的区别：

操作路径字符串，不代表真的修改了文件系统。

例如：

```python
path = os.path.join("data", "test.txt")
```

只会得到一个路径，并不会创建 `data` 文件夹，也不会创建 `test.txt` 文件。

而：

```python
os.mkdir("data")
```

才真正请求操作系统创建目录。

## 第二章：先彻底理解路径，否则后面会非常混乱

在使用 `os` 之前，需要先理解三个概念：相对路径、绝对路径和当前工作目录。

### 2.1 什么是路径？

假设你的文件结构如下：

```text
project/
├── main.py
├── data/
│   ├── scores.txt
│   └── students.csv
└── output/
```

路径就是用来描述文件或者文件夹所在位置的信息。

例如：

```text
data/scores.txt
```

表示 `data` 文件夹中的 `scores.txt` 文件。

### 2.2 相对路径

相对路径是相对于某个基准目录而言的路径。

例如：

```python
"data/scores.txt"
```

这段字符串没有说明它从哪个磁盘、哪个根目录开始。

在大多数普通文件操作中，Python 会根据当前工作目录解释它。

假设工作目录是：

```text
/home/alice/project
```

那么：

```text
data/scores.txt
```

对应：

```text
/home/alice/project/data/scores.txt
```

相对路径可以包含一些特殊符号：

| 符号    | 含义       | 示例             |
| ------- | ---------- | ---------------- |
| `.`     | 当前目录   | `./data`         |
| `..`    | 上一级目录 | `../images`      |
| `../..` | 上两级目录 | `../../test.txt` |

例如：

```python
os.path.join("..", "images", "cat.png")
```

表示从相对基准目录向上一级，再进入 `images`。

这里的 `..` 不属于 Python 专属语法，而是来自操作系统常见的路径表示规则。

### 2.3 绝对路径

绝对路径是从系统认可的根位置开始描述的路径。

Windows 示例：

```text
C:\Users\Alice\Documents\project\data.txt
```

Linux 示例：

```text
/home/alice/project/data.txt
```

macOS 示例：

```text
/Users/alice/project/data.txt
```

绝对路径一般不会因为当前工作目录变化而指向不同位置。

### 2.4 Windows 中的反斜杠问题

Windows 路径经常写成：

```python
path = "C:\new\test.txt"
```

这可能出错。

为什么？

因为 Python 字符串中的反斜杠 `\` 是转义符号。

例如：

```python
"\n"   # 换行
"\t"   # 制表符
```

所以：

```python
"C:\new\test.txt"
```

其中的 `\n` 会被解释成换行，`\t` 会被解释成制表符。

正确写法通常有三种：

```python
# 写法一：使用原始字符串
path = r"C:\new\test.txt"

# 写法二：反斜杠转义
path = "C:\\new\\test.txt"

# 写法三：使用正斜杠
path = "C:/new/test.txt"
```

三种写法在 Windows 常见文件系统操作中都能正常使用。

其中 `r` 表示 raw string，即原始字符串。

但是：

```python
r"C:\new\"
```

仍然是错误语法，因为原始字符串不能以奇数个反斜杠直接结束。

更推荐的方式是使用接下来要介绍的 `os.path.join()` 或 `pathlib`，避免手动处理分隔符。

## 第三章：`os.path`——最核心的路径处理工具

这是整个 `os` 体系中日常使用频率最高的部分之一。

### 3.1 `os.path.join()`：拼接路径

**高频必学**

基本语法：

```python
os.path.join(path1, path2, ...)
```

它会根据当前操作系统的路径规则，把多个路径部分拼接起来。

例如：

```python
import os

path = os.path.join("project", "data", "test.txt")

print(path)
```

在 Windows 中通常输出：

```text
project\data\test.txt
```

在 Linux / macOS 中通常输出：

```text
project/data/test.txt
```

为什么不用字符串加号？

你当然可以写：

```python
path = "project/" + "data/" + "test.txt"
```

但当路径很多、程序需要跨平台运行时，手动处理路径分隔符很容易出现问题。

例如：

```python
folder = "project"
filename = "test.txt"

path = folder + filename

print(path)
```

结果：

```text
projecttest.txt
```

这显然不是我们想要的路径。

使用：

```python
path = os.path.join(folder, filename)
```

就可以避免这种拼接错误。

#### 非常重要的细节：后面的绝对路径会覆盖前面的路径

在 Linux / macOS 中：

```python
print(os.path.join("/home/alice", "data.txt"))
```

结果：

```text
/home/alice/data.txt
```

但是：

```python
print(os.path.join("/home/alice", "/etc/config"))
```

结果：

```text
/etc/config
```

为什么 `/home/alice` 不见了？

因为第二部分 `/etc/config` 是绝对路径。

对于当前这个 POSIX 路径规则，新的绝对路径会使前面已经拼接的内容失效。

Windows 也有类似行为，但还涉及盘符和盘符相对路径等特殊情况。

因此，千万不要认为 `join()` 一定是在前一个路径后面简单地追加内容。

尤其不能把未经验证的用户输入直接拼进安全敏感的路径中。

### 3.2 `os.path.abspath()`：转换成绝对路径

语法：

```python
os.path.abspath(path)
```

作用是把给定路径转换成规范化的绝对路径形式。

假设当前工作目录是：

```text
/home/alice/project
```

执行：

```python
import os

print(os.path.abspath("data.txt"))
```

结果：

```text
/home/alice/project/data.txt
```

再看：

```python
print(os.path.abspath("../test.txt"))
```

结果：

```text
/home/alice/test.txt
```

这里发生了什么？

第一步：根据当前工作目录定位。

第二步：处理 `..` 等路径片段。

第三步：返回绝对路径。

要注意：

`abspath()` 不要求目标文件真实存在。

例如：

```python
print(os.path.abspath("abc123.txt"))
```

即使 `abc123.txt` 根本不存在，它仍然可以返回对应的绝对路径字符串。

原因是它主要负责路径计算，不负责判断文件是否存在。

这也意味着：

```python
os.path.abspath("abc123.txt")
```

不会帮你创建文件。

### 3.3 `os.path.basename()`：提取文件名

语法：

```python
os.path.basename(path)
```

示例：

```python
path = "/home/alice/data/report.pdf"

print(os.path.basename(path))
```

输出：

```text
report.pdf
```

它会获取路径的最后一个组成部分。

再看：

```python
print(os.path.basename("/home/alice/data"))
```

结果：

```text
data
```

这里的 `data` 可能是文件，也可能是目录。

`basename()` 不关心它的真实类型，只处理路径。

还有一个容易出错的细节：

```python
print(os.path.basename("/home/alice/data/"))
```

结果：

```text
""
```

因为路径末尾存在分隔符，最后一个路径组成部分为空。

如果希望先去掉末尾多余的分隔符，可以写：

```python
path = "/home/alice/data/"

print(os.path.basename(os.path.normpath(path)))
```

输出：

```text
data
```

### 3.4 `os.path.dirname()`：获取父目录部分

语法：

```python
os.path.dirname(path)
```

示例：

```python
path = "/home/alice/data/report.pdf"

print(os.path.dirname(path))
```

输出：

```text
/home/alice/data
```

它和 `basename()` 通常配合使用。

```python
path = "/home/alice/data/report.pdf"

folder = os.path.dirname(path)
filename = os.path.basename(path)

print(folder)
print(filename)
```

结果：

```text
/home/alice/data
report.pdf
```

但是要注意：

```python
os.path.dirname("report.pdf")
```

返回的是：

```text
""
```

而不是当前工作目录。

因为 `dirname()` 只是拆分传入的路径字符串，并没有自动补齐绝对路径。

如果你想得到完整父目录，可以组合：

```python
folder = os.path.dirname(os.path.abspath("report.pdf"))
```

这才会结合当前工作目录计算出绝对父目录。

注意：同样 `os.path.dirname()` **不会检查这个路径是文件还是文件夹**，它只是按字符串规则去掉最后一个路径组成部分

### 3.5 `os.path.split()`：一次性拆分路径

语法：

```python
os.path.split(path)
```

返回值是一个元组：

```text
(目录部分, 最后一个路径部分)
```

例如：

```python
path = "/home/alice/data/report.pdf"

result = os.path.split(path)

print(result)
```

结果：

```text
('/home/alice/data', 'report.pdf')
```

可以直接进行元组解包：

```python
folder, filename = os.path.split(path)

print(folder)
print(filename)
```

等价于分别调用：

```python
folder = os.path.dirname(path)
filename = os.path.basename(path)
```

因此，当你需要同时获得目录和文件名时，`split()` 很方便。

### 3.6 `os.path.splitext()`：分离文件名和扩展名

**高频必学**

语法：

```python
os.path.splitext(path)
```

它返回：

```text
(去掉最后一个扩展名的路径, 最后一个扩展名)
```

例如：

```python
name = "report.pdf"

print(os.path.splitext(name))
```

输出：

```text
('report', '.pdf')
```

实际应用：

```python
filename = "photo.jpg"

name, ext = os.path.splitext(filename)

print(name)
print(ext)
```

输出：

```text
photo
.jpg
```

#### 容易混淆的地方

```python
os.path.splitext("backup.tar.gz")
```

结果：

```text
('backup.tar', '.gz')
```

而不是：

```text
('backup', '.tar.gz')
```

因为它只分离最后一个扩展名。

对于隐藏文件：

```python
os.path.splitext(".gitignore")
```

结果：

```text
('.gitignore', '')
```

因为开头的点通常被认为属于文件名，而不是扩展名。

#### 常见组合：修改文件扩展名

```python
filename = "report.txt"

name, ext = os.path.splitext(filename)

new_filename = name + ".md"

print(new_filename)
```

输出：

```text
report.md
```

但这里仅仅是生成了一个新的文件名字符串，并没有真正修改文件。

如果你要重命名实际文件，还需要使用 `os.rename()` 等函数。

### 3.7 `os.path.exists()`：判断路径是否存在

**高频必学**

语法：

```python
os.path.exists(path)
```

返回布尔值：

```text
True
False
```

示例：

```python
import os

path = "data.txt"

if os.path.exists(path):
    print("文件或目录存在")
else:
    print("路径不存在")
```

这里有一个重要细节：

`exists()` 判断的是路径是否对应一个可确认存在的文件系统对象，而不是专门判断普通文件。

例如：

```python
os.path.exists("data")
```

当 `data` 是一个真实存在的文件夹时，同样会返回 `True`。

另外，`exists()` 对指向不存在目标的符号链接通常会返回 `False`；某些访问权限问题也可能造成 `False`。这意味着不能把所有 `False` 都简单理解成“磁盘上绝对没有这个名字”。

### 3.8 `isfile()`、`isdir()`、`islink()`：进一步判断类型

语法：

```python
os.path.isfile(path)
os.path.isdir(path)
os.path.islink(path)
```

各自含义：

| 函数       | 判断内容             |
| ---------- | -------------------- |
| `isfile()` | 是否是存在的普通文件 |
| `isdir()`  | 是否是存在的目录     |
| `islink()` | 是否是符号链接       |

例如：

```python
import os

path = "data"

if os.path.isfile(path):
    print("这是一个普通文件")
elif os.path.isdir(path):
    print("这是一个文件夹")
else:
    print("不是可确认存在的普通文件或文件夹")
```

注意两个问题。

第一，文件名包含 `.txt`，不代表它就一定是普通文件：

```python
os.path.isfile("test.txt")
```

Python 不会仅仅根据 `.txt` 判断文件类型。

第二，符号链接可能同时满足：

```python
os.path.islink(path) == True
os.path.isfile(path) == True
```

例如，某个符号链接指向一个真实存在的普通文件。

这不是判断出错，而是因为 `isfile()` 默认会跟随符号链接判断其目标。

如果你希望知道符号链接本身是否存在，包括已经失效的符号链接，可以了解：

```python
os.path.lexists(path)
```

### 3.9 `os.path.getsize()`：获取文件大小

语法：

```python
os.path.getsize(path)
```

返回大小，单位是字节（byte）。

```python
import os

size = os.path.getsize("report.pdf")

print(size)
```

假设输出：

```text
2048
```

意味着文件大小为 2048 字节。

如果需要换算：

```python
size_kib = size / 1024

print(size_kib)
```

输出：

```text
2.0
```

这里严格来说是 2 KiB，而不是十进制定义的 2 kB。

常用换算：

| 单位  | 大小      |
| ----- | --------- |
| 1 KiB | 1024 字节 |
| 1 MiB | 1024 KiB  |
| 1 GiB | 1024 MiB  |

实际应用：

```python
size = os.path.getsize("report.pdf")

if size > 10 * 1024 * 1024:
    print("文件大于10 MiB")
```

注意：对目录调用 `getsize()` 并不能可靠地得到目录中所有文件的大小之和。计算目录总大小通常要递归遍历。

### 3.10 `os.path.getmtime()`：获取最后修改时间

语法：

```python
os.path.getmtime(path)
```

它返回一个时间戳，通常表示自 Unix 纪元（1970 年 1 月 1 日 UTC）以来的秒数。

例如：

```python
import os

timestamp = os.path.getmtime("report.pdf")

print(timestamp)
```

可能得到类似：

```text
1790000000.0
```

这种形式不方便阅读。

因此一般会和 `datetime` 搭配：

```python
import os
from datetime import datetime

timestamp = os.path.getmtime("report.pdf")

modified = datetime.fromtimestamp(timestamp)

print(modified)
```

会将时间戳转换成电脑本地时区下的日期时间。

类似的方法还有：

```python
os.path.getatime(path)  # 最后访问时间
os.path.getctime(path)  # 系统定义的 ctime
```

尤其要注意 `getctime()`：

在 Windows 中，它通常代表创建时间；在 Unix 系统中，它通常代表文件元数据最后变更时间，而不是创建时间。

不要不加区分地把 `ctime` 翻译成创建时间。

### 3.11 `os.path.isabs()`：判断是否为绝对路径

语法：

```python
os.path.isabs(path)
```

在 Linux / macOS 上：

```python
print(os.path.isabs("/home/alice/data.txt"))
print(os.path.isabs("data.txt"))
```

输出：

```text
True
False
```

它不检查路径是否存在。

Windows 的绝对路径规则还涉及盘符、根分隔符、UNC 网络路径等情况，不能直接套用 Unix 的判断方式。

### 3.12 `os.path.relpath()`：计算相对路径

语法：

```python
os.path.relpath(path, start=".")
```

含义：计算从 `start` 目录出发，到 `path` 所对应位置的相对路径。

例如，在 Linux 上：

```python
target = "/home/alice/project/data/report.txt"
start = "/home/alice/project"

print(os.path.relpath(target, start))
```

输出：

```text
data/report.txt
```

再看：

```python
target = "/home/alice/images/cat.jpg"
start = "/home/alice/project"

print(os.path.relpath(target, start))
```

结果：

```text
../images/cat.jpg
```

它非常适合：

- 将绝对文件路径转换成项目相对路径。
- 生成文件清单。
- 把文件路径显示成更简短的形式。

特别注意：`relpath()` 只是计算相对路径，不要求真实文件存在；Windows 中跨不同盘符计算相对路径会引发 `ValueError`。

### 3.13 `normpath()`、`realpath()` 与 `expanduser()`

这三个方法不一定最常用，但非常值得理解。

① `normpath()`：规范化路径字符串

```python
import os

print(os.path.normpath("a/b/../c"))
```

在 Linux / macOS 中：

```text
a/c
```

这里的 `..` 表示上一级，因此路径被简化。

但它只是词法意义上的规范化，不负责真正追踪符号链接。

② `realpath()`：解析实际路径

```python
os.path.realpath(path)
```

它会尝试解析路径中的符号链接，并得到规范化后的实际路径。

例如：

```text
/project/latest  ->  /project/releases/v2
```

假设 `latest` 是指向 `releases/v2` 的符号链接：

```python
os.path.realpath("/project/latest/data.txt")
```

可能得到：

```text
/project/releases/v2/data.txt
```

区别如下：

| 函数         | 核心用途                       |
| ------------ | ------------------------------ |
| `abspath()`  | 转换为绝对路径                 |
| `normpath()` | 清理多余的路径片段             |
| `realpath()` | 尝试解析符号链接，获得实际路径 |

`realpath()` 默认也不保证路径存在；可以通过 `strict=True` 要求解析过程中遇到错误时抛出异常。

③ `expanduser()`：展开用户主目录

```python
os.path.expanduser("~")
```

会根据当前系统获取用户主目录。

例如，在 Linux 中可能得到：

```text
/home/alice
```

因此：

```python
config_path = os.path.join(
    os.path.expanduser("~"),
    "config.json"
)
```

比直接把 `/home/alice` 写死在程序中更灵活。

为什么需要 `expanduser()`？

因为 Python 的普通文件操作不会自动把 `~` 展开成用户主目录。

例如：

```python
open("~/notes.txt")
```

通常并不会自动去你的用户主目录中寻找 `notes.txt`。

你需要先展开路径，或者使用 `pathlib.Path(...).expanduser()`。

对应的环境变量展开函数是：

```python
os.path.expandvars("$HOME/data")
```

它会根据环境变量替换 `$HOME`；在 Windows 中还支持 `%变量名%` 形式。

### 3.14 `commonpath()` 与 `commonprefix()`：看似相同，实际不同

例如：

```python
paths = [
    "/home/alice/data",
    "/home/alice/docs"
]

print(os.path.commonpath(paths))
```

结果：

```text
/home/alice
```

`commonpath()` 按路径组成部分计算公共路径。

但是：

```python
os.path.commonprefix([
    "/usr/lib",
    "/usr/local/lib"
])
```

结果：

```text
/usr/l
```

因为它仅仅逐字符比较字符串前缀。

`/usr/l` 显然不是真正的公共目录。

所以涉及路径归属、公共目录等问题时，优先考虑 `commonpath()`，而不是 `commonprefix()`。

注意：即使使用 `commonpath()`，如果涉及用户输入和符号链接，也需要进一步考虑实际路径解析与文件系统竞争条件，不能把简单字符串比较视为完整的安全防护。

## 第四章：管理当前工作目录

### 4.1 `os.getcwd()`：获取当前工作目录

**高频必学**

语法：

```python
os.getcwd()
```

`getcwd` 来自：

`get` + `current` + `working` + `directory`

即获取当前工作目录。

例如：

```python
import os

cwd = os.getcwd()

print(cwd)
```

在 Windows 上可能输出：

```text
C:\Users\Alice\project
```

这里最重要的是理解：

当前工作目录不一定等于 Python 脚本所在的目录。

例如：

你的脚本位于：

```text
D:\project\main.py
```

但是你在终端中当前处于：

```text
C:\Users\Alice
```

然后执行：

```bash
python D:\project\main.py
```

如果没有其他程序修改工作目录，那么脚本内部的：

```python
print(os.getcwd())
```

通常输出：

```text
C:\Users\Alice
```

而不是：

```text
D:\project
```

因为当前工作目录一般由启动进程的环境决定，并不自动使用脚本所在位置。

#### 如果想获取脚本所在目录呢？

在正常运行的 `.py` 文件中，可以使用：

```python
import os

script_dir = os.path.dirname(
    os.path.abspath(__file__)
)

print(script_dir)
```

逐层拆解：

```python
__file__
```

表示当前模块文件的路径信息。

```python
os.path.abspath(__file__)
```

将其转换成绝对路径。

```python
os.path.dirname(...)
```

取出该文件所在目录。

注意，普通交互式解释器或某些 Notebook 环境中可能没有定义 `__file__`。

这两种路径的区别是学习文件操作时最容易忽略的问题之一。

### 4.2 `os.chdir()`：切换当前工作目录

语法：

```python
os.chdir(path)
```

`chdir` 是 change directory 的缩写。

例如：

```python
import os

print(os.getcwd())

os.chdir("data")

print(os.getcwd())
```

假设最初工作目录为：

```text
/home/alice/project
```

且其中确实存在 `data` 目录。

执行后可能输出：

```text
/home/alice/project
/home/alice/project/data
```

为什么需要它？

因为一旦切换工作目录，后面的相对路径会按照新工作目录解析。

例如：

```python
os.chdir("data")

with open("scores.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

此时 `scores.txt` 是从新的工作目录查找的。

但要注意：`chdir()` 改变的是当前进程的工作目录，会影响该进程中其他使用相对路径的代码，并不是只对下一行生效。

因此在大型程序、异步程序或多线程程序中，不建议频繁改变工作目录。

很多时候，更好的做法是直接使用绝对路径：

```python
path = os.path.join(
    "/home/alice/project",
    "data",
    "scores.txt"
)

with open(path, "r", encoding="utf-8") as f:
    content = f.read()
```

这样代码不依赖于工作目录被修改。

## 第五章：创建、查看与遍历文件夹

从这一章开始，我们不再只是处理路径字符串，而是真正操作文件系统。

### 5.1 `os.mkdir()`：创建单层目录

语法：

```python
os.mkdir(path, mode=0o777)
```

参数解释：

| 参数   | 作用                         | 默认值   |
| ------ | ---------------------------- | -------- |
| `path` | 要创建的目录路径             | 必须提供 |
| `mode` | 创建目录时请求设置的权限模式 | `0o777`  |

这里的 `0o777` 是八进制整数，主要涉及 Unix 文件权限。实际生效的权限还受到 `umask` 等因素影响；Windows 的权限处理与 Unix 不完全相同。

最简单的例子：

```python
import os

os.mkdir("data")
```

如果当前工作目录下不存在 `data` 文件夹，就会创建它。

创建成功时，一般返回：

```text
None
```

#### 问题一：如果文件夹已经存在呢？

```python
os.mkdir("data")
os.mkdir("data")
```

第二次执行通常会报错：

```text
FileExistsError
```

因为它尝试创建一个已经存在的目录。

#### 问题二：如果父目录不存在呢？

```python
os.mkdir("project/data/results")
```

假设 `project`、`data` 都不存在，那么通常会报：

```text
FileNotFoundError
```

为什么？

因为 `mkdir()` 的核心职责是创建指定的单个目录，而不是负责把整条目录层级都创建出来。

这就引出了下面的函数。

### 5.2 `os.makedirs()`：递归创建多层目录

实际开发中非常常用，尤其适合初始化项目目录。

语法：

```python
os.makedirs(name, mode=0o777, exist_ok=False)
```

三个参数：

| 参数       | 含义                             |
| ---------- | -------------------------------- |
| `name`     | 目标目录路径                     |
| `mode`     | 创建末级目录时请求使用的权限模式 |
| `exist_ok` | 目标目录已经存在时是否允许继续   |

例如：

```python
import os

os.makedirs("project/data/results")
```

即使 `project` 和 `data` 都不存在，也会依次创建所需要的目录。

最终结构：

```text
project/
└── data/
    └── results/
```

#### 最重要的参数：`exist_ok=True`

```python
os.makedirs(
    "project/data/results",
    exist_ok=True
)
```

这里：

```python
exist_ok=True
```

表示：如果目标目录已经存在且确实是目录，就不因为它已经存在而报错。

所以这段代码可以重复运行。

但它并不是忽略所有错误。例如目标路径上存在一个同名普通文件、权限不足，仍然可能抛出异常。

为什么推荐这种写法？

考虑一个程序，每次运行都要保证输出目录存在：

```python
output_dir = "results"

os.makedirs(output_dir, exist_ok=True)
```

不需要先写：

```python
if not os.path.exists(output_dir):
    os.mkdir(output_dir)
```

而且后一种“先检查再创建”的写法，存在检查之后目录状态发生变化的可能性。

前一种更简洁，通常也更合适。

#### `mkdir()` 和 `makedirs()` 总结

| 对比                 | `os.mkdir()` | `os.makedirs()` |
| -------------------- | ------------ | --------------- |
| 创建单层目录         | 支持         | 支持            |
| 自动创建缺失的父目录 | 不支持       | 支持            |
| 内置 `exist_ok`      | 无           | 有              |
| 常见使用场景         | 简单目录创建 | 项目目录初始化  |

另一个细节：`makedirs()` 的 `mode` 不会直接用于设置它新创建的所有中间父目录。现代 Python 版本中，它主要传递给末级目录，权限还会受到系统规则影响。

Python 3.14.10 documentation

### 5.3 `os.listdir()`：列出目录中的内容

语法：

```python
os.listdir(path=".")
```

这里的：

```python
path="."
```

表示默认查看当前工作目录。

例如：

```python
import os

result = os.listdir()

print(result)
```

假设当前目录有：

```text
main.py
data/
report.pdf
```

可能得到：

```text
['main.py', 'data', 'report.pdf']
```

返回值是一个列表，元素通常是字符串。

每个字符串代表对应目录中的一个条目名称。

注意：它返回的是名称，不是完整路径。

例如：

```python
files = os.listdir("data")
```

假设 `data` 内有：

```text
a.txt
b.txt
```

得到：

```text
['a.txt', 'b.txt']
```

而不是：

```text
['data/a.txt', 'data/b.txt']
```

因此，如果要进一步打开这些文件，需要拼接完整路径。

```python
import os

folder = "data"

for name in os.listdir(folder):
    full_path = os.path.join(folder, name)
    print(full_path)
```

结果，在 Linux / macOS 中：

```text
data/a.txt
data/b.txt
```

列表顺序由文件系统决定，不应该假设总是按名称排序。

如果希望排序：

```python
for name in sorted(os.listdir(folder)):
    print(name)
```

这样会按照 Python 默认字符串排序规则得到稳定的名称顺序。

#### 组合应用：只列出 `.txt` 文件

```python
import os

folder = "data"

for name in os.listdir(folder):
    full_path = os.path.join(folder, name)

    if os.path.isfile(full_path) and name.endswith(".txt"):
        print(name)
```

逐句解释：

```python
for name in os.listdir(folder):
```

遍历 `data` 目录中的每一个条目名称。

```python
full_path = os.path.join(folder, name)
```

组合出对应路径。

```python
os.path.isfile(full_path)
```

判断它是否是普通文件。

```python
name.endswith(".txt")
```

判断名称是否以 `.txt` 结尾。

```python
and
```

要求两个条件都满足。

最后：

```python
print(name)
```

输出匹配的文件名。

注意，`endswith(".txt")` 区分大小写，`A.TXT` 不会匹配。

如果需要忽略大小写：

```python
name.lower().endswith(".txt")
```

这个例子体现了 `os` 的一个经典使用模式：

列出目录 → 拼接完整路径 → 判断文件类型 → 处理目标文件。

### 5.4 `os.scandir()`：更高效地遍历目录

语法：

```python
os.scandir(path=".")
```

它同样能够获取目录中的内容。

不过，与 `listdir()` 不同，`scandir()` 返回的是迭代器，其中每个元素为 `os.DirEntry` 对象。

例如：

```python
import os

with os.scandir("data") as entries:
    for entry in entries:
        print(entry.name)
```

假设目录里有：

```text
a.txt
b.txt
images/
```

就会输出这些名称，顺序不保证。

#### `DirEntry` 为什么有用？

因为一个 `DirEntry` 对象不仅有名字，还提供了其他信息和方法。

| 属性或方法           | 含义             |
| -------------------- | ---------------- |
| `entry.name`         | 文件或目录名称   |
| `entry.path`         | 对应的路径       |
| `entry.is_file()`    | 是否为普通文件   |
| `entry.is_dir()`     | 是否为目录       |
| `entry.is_symlink()` | 是否为符号链接   |
| `entry.stat()`       | 获取文件状态信息 |

例如：

```python
import os

with os.scandir("data") as entries:
    for entry in entries:
        if entry.is_file():
            print("文件:", entry.name)

        elif entry.is_dir():
            print("文件夹:", entry.name)
```

与 `listdir()` 组合 `os.path.isfile()` 相比，`scandir()` 在某些文件系统上能够利用目录扫描时已有的类型信息，减少额外的系统调用。

因此，当你不仅需要文件名，还需要判断文件类型时，`scandir()` 通常更合适。

官方文档也明确说明了这一性能优势。

#### 为什么使用 `with`？

```python
with os.scandir("data") as entries:
```

`scandir()` 迭代器可能持有操作系统资源。

使用 `with` 可以保证离开代码块时正确关闭迭代器。

这和我们平时使用：

```python
with open("data.txt") as f:
```

的思路相似：让 Python 自动负责资源清理。

### 5.5 `os.walk()`：递归遍历整个目录树

这是一个非常值得重点学习的函数。

语法：

```python
os.walk(
    top,
    topdown=True,
    onerror=None,
    followlinks=False
)
```

参数说明：

| 参数          | 含义                           |
| ------------- | ------------------------------ |
| `top`         | 开始遍历的目录                 |
| `topdown`     | 是否优先访问父目录             |
| `onerror`     | 遍历发生错误时的处理回调       |
| `followlinks` | 是否递归进入符号链接指向的目录 |

最重要的是理解返回值。

`os.walk()` 在每次迭代时产生三个元素：

```text
(root, dirs, files)
```

其中：

- `root`：当前正在访问的目录路径。
- `dirs`：当前目录下的子目录名称列表。
- `files`：当前目录下的非目录条目名称列表。

这里的 `files` 不一定全部都是普通文件，也可能包含其他非目录类型。

假设有如下结构：

```text
project/
├── main.py
├── data/
│   ├── a.txt
│   └── b.txt
└── images/
    └── cat.png
```

执行：

```python
import os

for root, dirs, files in os.walk("project"):
    print("当前目录:", root)
    print("子目录:", dirs)
    print("文件:", files)
    print("------")
```

一种可能的输出（这里假设先访问 `data`）：

```text
当前目录: project
子目录: ['data', 'images']
文件: ['main.py']
------

当前目录: project/data
子目录: []
文件: ['a.txt', 'b.txt']
------

当前目录: project/images
子目录: []
文件: ['cat.png']
------
```

#### 为什么 `os.walk()` 比 `listdir()` 更适合递归？

因为 `listdir()` 只查看指定目录的直接子项。

它不会自动进入子目录。

而 `walk()` 会逐层访问目录树。

例如我们想找到整个项目中的 `.py` 文件：

```python
import os

for root, dirs, files in os.walk("project"):
    for name in files:
        if name.endswith(".py"):
            full_path = os.path.join(root, name)
            print(full_path)
```

无论 `.py` 文件在根目录还是某个子目录里，都有机会被找到。

#### `topdown=True` 有什么意义？

默认：

```python
os.walk("project", topdown=True)
```

父目录先于子目录被返回。

如果：

```python
os.walk("project", topdown=False)
```

则会先返回较深层的子目录，再返回父目录。

这在清理目录结构等任务中很有价值。

#### 一个很实用的技巧：跳过特定文件夹

假设你不想访问 Python 的缓存目录：

```python
import os

for root, dirs, files in os.walk("project"):
    dirs[:] = [
        d for d in dirs
        if d != "__pycache__"
    ]

    for name in files:
        print(os.path.join(root, name))
```

重点是：

```python
dirs[:]
```

这是对原列表进行切片赋值，也就是原地修改。

为什么不能简单地写：

```python
dirs = [...]
```

因为这只是把当前局部变量重新绑定到另一个列表。

`os.walk()` 在 `topdown=True` 的情况下，会读取原来的 `dirs` 列表来确定接下来进入哪些子目录。

因此：

```python
dirs[:] = [...]
```

能够影响之后的遍历，而：

```python
dirs = [...]
```

通常不能起到相同效果。

这是一个很典型的 Python 列表知识与 `os` 模块结合的应用。

#### `followlinks=True` 有什么风险？

```python
os.walk("project", followlinks=True)
```

会允许程序进入符号链接所指向的目录。

但符号链接可能构成循环，例如：

```text
A -> B
B -> A
```

此时可能导致重复遍历甚至无限递归。

所以如果没有明确需要，建议保持默认值：

```python
followlinks=False
```

还有一点容易忽略：`os.walk()` 默认会忽略某些目录扫描错误。如果需要记录权限错误等情况，应提供 `onerror` 回调。

Python 3.14.10 documentation

## 第六章：删除、重命名、移动文件和目录

这一章需要格外小心，因为有些操作会真实改变甚至永久删除磁盘内容。

### 6.1 `os.remove()`：删除文件

语法：

```python
os.remove(path)
```

例如：

```python
import os

os.remove("old.txt")
```

作用：删除路径指向的文件。

如果文件不存在：

```python
os.remove("missing.txt")
```

通常会得到：

```text
FileNotFoundError
```

如果传入的是普通目录，通常也会报错，因为 `remove()` 不负责删除目录。

#### 常见搭配：判断文件是否存在

```python
path = "old.txt"

if os.path.isfile(path):
    os.remove(path)
```

虽然看起来合理，但这里仍然有一个细节：检查之后，真正删除之前，文件可能被其他程序移走。

如果文件不存在是可以接受的情况，更推荐：

```python
try:
    os.remove("old.txt")

except FileNotFoundError:
    print("文件已经不存在")
```

不过这只处理了文件不存在的情况。权限不足等其他错误仍然会抛出，这通常是合理的。

重要提醒：`os.remove()` 通常不是把文件移动到回收站。不要用它直接清理未经确认的重要文件。

另外：

```python
os.unlink(path)
```

与 `os.remove(path)` 在 Python 中具有相同的删除语义。

### 6.2 `os.rmdir()`：删除空目录

语法：

```python
os.rmdir(path)
```

例如：

```python
os.mkdir("temp")

os.rmdir("temp")
```

因为刚刚创建的 `temp` 是空目录，所以能够删除。

但假设目录结构如下：

```text
temp/
└── data.txt
```

执行：

```python
os.rmdir("temp")
```

就会因为目录非空而失败。

这不是 Python 的缺陷，而是函数设计上的限制。

它只负责删除空目录。

### 6.3 `os.removedirs()`：沿路径删除空目录

语法：

```python
os.removedirs(name)
```

假设结构为：

```text
a/
└── b/
    └── c/
```

其中每一级目录都没有其他文件。

执行：

```python
os.removedirs("a/b/c")
```

程序会尝试：

第一步：删除 `c`。

第二步：如果 `b` 也为空，删除 `b`。

第三步：如果 `a` 也为空，删除 `a`。

但是：

```python
os.removedirs("a/b/c")
```

不会主动删除目录中的文件。

它也不会强制删除非空目录。

如果某个父目录仍有其他内容，向上的清理就会停止。

#### 那么怎么删除非空目录？

使用另一个标准库：

```python
import shutil

shutil.rmtree("temp")
```

`rmtree()` 的意思是递归删除目录树。

但要强调：这属于高风险操作。它会尝试删除该目录内的内容，不会默认提供回收站保护。

当目标路径来自用户输入、配置文件或者外部数据时，必须特别小心验证。

### 6.4 `os.rename()`：重命名文件或目录

语法：

```python
os.rename(src, dst)
```

参数：

```python
src  # 原路径
dst  # 新路径
```

例如：

```python
import os

os.rename("old.txt", "new.txt")
```

原来：

```text
old.txt
```

执行后：

```text
new.txt
```

它的本质不是仅仅修改字符串，而是真正请求文件系统把原路径对应的条目改名或移动。

#### 可以用它移动文件吗？

可以，在文件系统允许的情况下。

假设：

```text
project/
├── report.txt
└── backup/
```

执行：

```python
os.rename(
    "project/report.txt",
    "project/backup/report.txt"
)
```

结果：

```text
project/
└── backup/
    └── report.txt
```

所以 `rename()` 不仅能改名，还可以在适当条件下改变文件的位置。

但目标父目录通常需要已经存在。

#### 非常重要：目标文件已经存在时怎么办？

这里存在操作系统差异。

例如：

```python
os.rename("a.txt", "b.txt")
```

如果 `b.txt` 已经存在：

- Windows 通常抛出 `FileExistsError`。
- Unix 系统上，当二者都是普通文件且权限允许时，`b.txt` 可能被直接替换。

所以，不要把 `rename()` 理解成所有平台上都“自动拒绝覆盖”。

如果想跨平台明确表达替换目标文件的意图，可以使用：

```python
os.replace("a.txt", "b.txt")
```

如果想支持跨文件系统的移动，可以考虑：

```python
import shutil

shutil.move("a.txt", "backup/a.txt")
```

`os.rename()` 和 `os.replace()` 在跨文件系统移动时都可能失败。

Python 3.14.10 documentation+1

### 6.5 `os.replace()`：明确替换目标

语法：

```python
os.replace(src, dst)
```

例如：

```python
os.replace("new_data.txt", "data.txt")
```

如果 `data.txt` 已经存在且是普通文件，并且权限等条件允许，它会被替换。

这常用于程序更新配置文件或者输出文件。

#### 实战：先写临时文件，再替换正式文件

```python
import os

target = "config.txt"
temp = "config.txt.tmp"

with open(temp, "w", encoding="utf-8") as f:
    f.write("version=2\n")

os.replace(temp, target)
```

逐步理解：

```python
with open(temp, "w", encoding="utf-8") as f:
```

先把新数据写入临时文件。

```python
f.write("version=2\n")
```

写入新的内容。

```python
os.replace(temp, target)
```

最后把临时文件替换成正式文件。

为什么这么设计？

因为直接写入正式文件时，如果程序中途崩溃，可能留下只写入了一部分的内容。

先写临时文件，再替换，能减少这一风险。

在支持相应原子重命名语义的文件系统上，同一文件系统内的替换能够避免读者观察到部分写入的目标文件。

不过，原子替换不等于断电后数据一定持久保存。需要更强的数据可靠性时，还应考虑 `flush()`、`os.fsync()` 及目录元数据同步等问题。

### 6.6 `os` 并不负责所有文件操作

初学者容易误认为，既然 `os` 能删除、重命名文件，那么复制文件一定也是 `os.copy()`。

但并没有常规的：

```python
os.copy()
```

应该使用 `shutil`：

```python
import shutil

shutil.copy("a.txt", "b.txt")
```

更多常用对应关系：

| 任务                         | 推荐方法            |
| ---------------------------- | ------------------- |
| 删除普通文件                 | `os.remove()`       |
| 删除空目录                   | `os.rmdir()`        |
| 删除整棵目录树               | `shutil.rmtree()`   |
| 重命名                       | `os.rename()`       |
| 明确替换目标文件             | `os.replace()`      |
| 跨文件系统移动               | `shutil.move()`     |
| 复制文件内容                 | `shutil.copyfile()` |
| 复制文件并尽量保留部分元数据 | `shutil.copy2()`    |
| 复制整个目录树               | `shutil.copytree()` |

之所以这样分工，是因为 `os` 偏向提供操作系统基础接口，而 `shutil` 提供更高级、通常更方便的文件操作。

另外，`shutil.copy2()` 也不保证复制所有系统元数据，例如所有者、ACL 等信息在不同操作系统上可能无法完整保留。

Python 3.14.10 documentation

## 第七章：文件属性、权限和状态信息

前面的函数帮助我们找到文件、创建文件、删除文件。

这一章则讨论：

怎样知道一个文件到底有多大、最近什么时候修改、拥有哪些权限？

### 7.1 `os.stat()`：获取文件状态

语法：

```python
os.stat(path)
```

返回一个 `os.stat_result` 对象。

例如：

```python
import os

info = os.stat("report.txt")

print(info)
```

它包含多种文件系统信息。

最常用的属性如下：

| 属性       | 含义                    |
| ---------- | ----------------------- |
| `st_size`  | 文件大小，单位为字节    |
| `st_mtime` | 最后修改时间            |
| `st_atime` | 最后访问时间            |
| `st_ctime` | 系统定义的 ctime        |
| `st_mode`  | 文件类型和权限模式      |
| `st_ino`   | 文件的 inode 或对应标识 |
| `st_dev`   | 文件所在设备标识        |

例如：

```python
info = os.stat("report.txt")

print("大小:", info.st_size)
print("修改时间:", info.st_mtime)
```

注意 `st_ctime` 的跨平台含义与 `getctime()` 一样，不能一概视为创建时间。

#### 为什么既有 `stat()`，又有 `getsize()`？

因为：

```python
os.path.getsize(path)
```

只获取文件大小。

而：

```python
os.stat(path)
```

一次性提供多种状态信息。

如果你只需要文件大小，`getsize()` 简单直接。

如果需要很多属性，`stat()` 更方便。

例如：

```python
import os
from datetime import datetime

info = os.stat("report.txt")

size = info.st_size
modified = datetime.fromtimestamp(info.st_mtime)

print("文件大小:", size, "bytes")
print("修改时间:", modified)
```

### 7.2 `os.chmod()`：修改文件权限

语法：

```python
os.chmod(path, mode)
```

它主要在 Unix/Linux 权限体系下很有意义。

例如：

```python
os.chmod("script.sh", 0o755)
```

这里的：

```python
0o755
```

是什么意思？

先理解八进制。

Python 中：

```python
0o755
```

是一个八进制整数。

权限通常分为三组：

| 数字       | 对象       |
| ---------- | ---------- |
| 第一个 `7` | 文件所有者 |
| 第二个 `5` | 所属组     |
| 第三个 `5` | 其他用户   |

每一组的权限值：

| 权限   | 数值 |
| ------ | ---- |
| 读取 r | 4    |
| 写入 w | 2    |
| 执行 x | 1    |

因此：

```text
7 = 4 + 2 + 1 = rwx
5 = 4 + 1     = r-x
5 = 4 + 1     = r-x
```

最终：

```text
rwxr-xr-x
```

直观理解：

所有者可以读、写、执行；同组用户和其他用户可以读、执行。

另一个常见例子：

```python
os.chmod("private.txt", 0o600)
```

在典型 Unix 权限语义中，表示仅允许所有者读取和写入。

要注意，`chmod` 受操作系统支持、所有权、文件系统和权限模型影响。

在 Windows 中，Python 的 `os.chmod()` 主要对只读属性进行有限处理，并不等价于 Unix 的完整权限控制。

Python 3.14.10 documentation

### 7.3 `os.access()`：检查访问权限

语法：

```python
os.access(path, mode)
```

常用标志：

```python
os.F_OK  # 检查存在性
os.R_OK  # 检查读取权限
os.W_OK  # 检查写入权限
os.X_OK  # 检查执行权限
```

示例：

```python
import os

path = "report.txt"

print(os.access(path, os.F_OK))
print(os.access(path, os.R_OK))
print(os.access(path, os.W_OK))
```

也可以组合多个权限标志：

```python
os.access(path, os.R_OK | os.W_OK)
```

这里：

```python
|
```

是按位或运算符，用于组合权限标志。

但有一个特别重要的注意事项：

`os.access()` 返回 `True`，不保证后续操作一定成功。

因为检查与真正打开文件之间可能发生变化，而且访问判断还会受到系统权限机制影响。

所以实际业务中通常更推荐：

```python
try:
    with open("report.txt", "r", encoding="utf-8") as f:
        content = f.read()

except PermissionError:
    print("没有读取权限")

except FileNotFoundError:
    print("文件不存在")
```

而不是把 `os.access()` 的结果当成最终保证。

### 7.4 `os.utime()`：修改访问时间和修改时间

语法中常见的形式：

```python
os.utime(path, times=None, *, ns=None)
```

其中：

- `times` 可以是 `(访问时间, 修改时间)`。
- `ns` 可用于提供纳秒单位的时间值。
- 二者不能同时提供实际时间元组。

例如：

```python
import os
import time

now = time.time()

os.utime("report.txt", (now, now))
```

表示把访问时间和修改时间都设置为当前时间附近。

如果省略时间：

```python
os.utime("report.txt")
```

默认会请求将访问和修改时间更新为当前时间。

这类操作常见于自动化构建、文件同步、测试以及元数据管理。

但它不会修改文件内容，也不能凭空保证修改时间具有实际业务含义。

## 第八章：环境变量——`os.environ` 和 `os.getenv()`

环境变量是很多初学者第一次接触操作系统编程时比较难理解的概念。

### 8.1 什么是环境变量？

先看一个生活中的例子。

假设你写了一个程序，需要知道：

- 当前程序运行在哪种环境中。
- 某个工具安装在哪里。
- 数据库服务器的地址。
- 程序应该使用哪个配置。
- 某个访问令牌应该从哪里读取。

你可以把这些信息直接写在代码中：

```python
database_host = "localhost"
```

但如果程序要在不同电脑上运行，这种方式就不够灵活。

于是，操作系统提供了一种与进程运行环境相关的配置机制：环境变量。

例如：

```text
APP_MODE=development
```

意思是：

一个名为 `APP_MODE` 的环境变量，其值是 `development`。

Python 程序可以读取这个值，然后决定怎样运行。

### 8.2 `os.environ`：环境变量映射

最基本的写法：

```python
import os

print(os.environ)
```

你会看到一个包含许多环境变量的映射对象。

它的使用方式与字典非常相似。

例如：

```python
os.environ["APP_MODE"] = "development"

print(os.environ["APP_MODE"])
```

结果：

```text
development
```

逐句解释：

```python
os.environ
```

是 Python 提供的环境变量映射。

```python
os.environ["APP_MODE"]
```

表示访问键为 `APP_MODE` 的环境变量。

```python
= "development"
```

表示给这个环境变量赋值。

这里有一个重要规则：环境变量的名称和值通常都应该使用字符串。

错误示例：

```python
os.environ["PORT"] = 8080
```

会产生类型错误。

正确写法：

```python
os.environ["PORT"] = "8080"
```

后续如果需要整数：

```python
port = int(os.environ["PORT"])
```

这样才可以把字符串转换成数字。

### 8.3 `os.getenv()`：更安全地读取环境变量

语法：

```python
os.getenv(key, default=None)
```

参数：

| 参数      | 解释                     |
| --------- | ------------------------ |
| `key`     | 环境变量名称             |
| `default` | 变量不存在时返回的默认值 |

例如：

```python
import os

mode = os.getenv("APP_MODE")

print(mode)
```

如果没有这个变量，通常返回：

```text
None
```

可以提供默认值：

```python
mode = os.getenv("APP_MODE", "production")
```

意思是：

如果 `APP_MODE` 存在，就读取它。

如果不存在，就使用字符串 `"production"`。

#### `os.environ[]` 与 `os.getenv()` 的区别

```python
os.environ["APP_MODE"]
```

如果变量不存在，会抛出：

```text
KeyError
```

而：

```python
os.getenv("APP_MODE")
```

变量不存在时通常返回 `None`。

对比：

| 方法                  | 存在时     | 不存在时            |
| --------------------- | ---------- | ------------------- |
| `os.environ["X"]`     | 返回字符串 | `KeyError`          |
| `os.environ.get("X")` | 返回字符串 | `None` 或指定默认值 |
| `os.getenv("X")`      | 返回字符串 | `None` 或指定默认值 |

在一般情况下：

```python
os.environ.get("X")
```

与：

```python
os.getenv("X")
```

作用非常接近。

#### 实际应用：开发模式与正式模式

```python
import os

mode = os.getenv("APP_MODE", "production")

if mode == "development":
    print("启用调试功能")
else:
    print("使用常规运行模式")
```

通过更改环境变量，就能在不修改代码的情况下切换模式。

### 8.4 环境变量修改是永久的吗？

例如：

```python
import os

os.environ["TEST"] = "hello"
```

这会修改当前 Python 进程所使用的环境变量，并通常影响之后启动且继承该环境的子进程。

但它不会自动把这个变量永久写入系统配置，也不会修改已经存在的父进程的环境。

所以，下次重新启动程序时，`TEST` 不一定存在。

这点非常重要。

还有：

```python
del os.environ["TEST"]
```

可以从当前环境变量映射中删除该变量。

如果不确定变量是否存在：

```python
os.environ.pop("TEST", None)
```

不会因为键不存在而抛出 `KeyError`。

#### 一个容易忽略的细节

`os.environ` 维护着环境变量的映射，读取时不代表每次都重新向操作系统查询全部变量。

现代 Python 还提供了 `os.reload_environ()`（Python 3.14 新增）用于重新加载当前进程环境中的变化，不过该函数不是线程安全的，正常程序通常不需要使用。

优先通过 `os.environ` 修改环境变量，而不是直接调用 `os.putenv()`，因为后者不会同步更新 `os.environ` 映射。

Python 3.14.10 documentation

### 8.5 实际应用：使用环境变量保存配置

```python
import os

host = os.getenv("DB_HOST", "localhost")
port = int(os.getenv("DB_PORT", "5432"))
debug = os.getenv("DEBUG", "false").lower() == "true"

print("数据库地址:", host)
print("端口:", port)
print("调试模式:", debug)
```

这里有两处值得注意。

第一：

```python
int(os.getenv("DB_PORT", "5432"))
```

为什么要使用 `int()`？

因为环境变量值是字符串，而端口号在程序中经常需要整数类型。

第二：

```python
os.getenv("DEBUG", "false").lower() == "true"
```

不要直接写：

```python
bool(os.getenv("DEBUG", "false"))
```

因为：

```python
bool("false")
```

返回的仍然是：

```text
True
```

非空字符串在 Python 中的布尔值通常都是 `True`。

另外，环境变量能够帮助避免在代码中硬编码凭证，但并不意味着其中的秘密信息绝对安全。应避免将敏感环境变量直接打印到日志中。

## 第九章：系统信息、进程与命令执行

`os` 的另一大功能是与操作系统和正在运行的进程交互。

### 9.1 `os.name`：判断操作系统家族

```python
import os

print(os.name)
```

常见结果：

| 返回值    | 通常代表                   |
| --------- | -------------------------- |
| `"nt"`    | Windows                    |
| `"posix"` | Linux、macOS 等 POSIX 系统 |

例如：

```python
import os

if os.name == "nt":
    print("运行在 Windows 环境")
elif os.name == "posix":
    print("运行在 POSIX 环境")
```

注意：

```python
os.name == "posix"
```

不能直接说明当前系统是 Linux。

因为 macOS 也属于 POSIX 系统家族。

如果你需要更精确的操作系统名称，可以使用：

```python
import platform

print(platform.system())
```

常见输出包括：

```text
Windows
Linux
Darwin
```

这里的 `Darwin` 是 macOS 底层系统名称。

### 9.2 `os.sep`：系统路径分隔符

```python
import os

print(os.sep)
```

Windows 上通常是：

```text
\
```

Linux / macOS 上通常是：

```text
/
```

你也可以手动写：

```python
path = "data" + os.sep + "report.txt"
```

但这种写法通常没有必要。

因为：

```python
os.path.join("data", "report.txt")
```

更容易阅读，也能处理更多路径细节。

其他相关常量：

| 常量         | 含义                                 |
| ------------ | ------------------------------------ |
| `os.sep`     | 主要路径分隔符                       |
| `os.altsep`  | 可选路径分隔符，可能为 `None`        |
| `os.pathsep` | 环境变量 `PATH` 中分隔多个路径的字符 |
| `os.curdir`  | 当前目录标记，通常是 `.`             |
| `os.pardir`  | 父目录标记，通常是 `..`              |
| `os.linesep` | 系统惯用的文本行分隔符               |

这里需要特别区分：

```python
os.sep
```

与：

```python
os.pathsep
```

例如，Windows 中：

```python
os.pathsep == ";"
```

而 Linux 中通常：

```python
os.pathsep == ":"
```

这是因为环境变量 `PATH` 可能包含多个搜索目录。

例如 Linux 中：

```text
/usr/bin:/bin:/usr/local/bin
```

可以通过：

```python
import os

path_value = os.getenv("PATH", "")

directories = path_value.split(os.pathsep)

for directory in directories:
    print(directory)
```

得到各个目录。

另外，写普通文本文件时不必主动使用 `os.linesep`，通常直接在 Python 字符串中使用 `\n` 更合适。

### 9.3 `os.getpid()`：获取当前进程 ID

语法：

```python
os.getpid()
```

`pid` 是 Process ID，即进程标识符。

例如：

```python
import os

print(os.getpid())
```

可能得到：

```text
12345
```

这里的数字只是示例，实际由操作系统分配。

#### 进程是什么？

可以简单理解为正在运行的程序实例。

例如，你同时启动两个 Python 程序：

```bash
python program_a.py
python program_b.py
```

系统会为它们分配不同的进程标识符。

进程 ID 常用于：

- 日志定位。
- 排查程序运行问题。
- 进程管理。
- 监控系统状态。

也可以使用：

```python
os.getppid()
```

获取父进程 ID，具体支持程度取决于运行平台。

父进程通常是创建或启动当前进程的进程。

### 9.4 `os.cpu_count()`：获取逻辑 CPU 数量

语法：

```python
os.cpu_count()
```

例如：

```python
import os

count = os.cpu_count()

print(count)
```

可能得到：

```text
16
```

这里通常代表系统的逻辑处理器数量，不一定是物理 CPU 核心数量。

比如某些 CPU 具有 8 个物理核心、16 个逻辑处理器。

需要注意：

`os.cpu_count()` 有可能返回 `None`，而且它不一定代表当前进程实际能够使用的处理器数量。

例如，容器或进程 CPU 亲和性设置可能限制当前程序可用的处理器。

因此：

```python
workers = os.cpu_count() or 1
```

是常见的兜底写法。

如果你的程序涉及并行计算，还需要进一步考虑任务类型和实际 CPU 配额，而不是简单地把 `cpu_count()` 的结果作为最佳工作进程数。

### 9.5 `os.system()`：执行系统命令

语法：

```python
os.system(command)
```

例如：

```python
import os

os.system("echo hello")
```

它会让操作系统通过命令解释器执行给定的命令。

再例如：

```python
os.system("python --version")
```

通常会尝试运行相应的终端命令。

但这有几个问题。

第一，命令可能受到系统类型影响。

例如：

```python
os.system("dir")
```

是 Windows 命令提示符中常见的列目录命令。

而：

```python
os.system("ls")
```

是 Unix 类系统中常见的列目录命令。

第二，`os.system()` 的返回值不是命令输出的字符串。

```python
result = os.system("echo hello")

print(result)
```

`hello` 可能被直接输出到终端，而 `result` 是系统相关的状态值，不应该误以为它保存了 `"hello"`。

第三，把不可信字符串直接拼入系统命令，可能导致命令注入漏洞。

因此，现代 Python 中执行外部程序，通常更推荐 `subprocess` 模块。

### 9.6 `subprocess.run()`：推荐的命令执行方式

```python
import subprocess

result = subprocess.run(
    ["python", "--version"],
    capture_output=True,
    text=True
)

print(result.stdout)
```

这里使用了：

```python
capture_output=True
```

表示捕获标准输出和标准错误。

```python
text=True
```

表示以文本方式处理输出。

不过，更稳妥地调用当前 Python 解释器，可以这样写：

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "print(123)"],
    capture_output=True,
    text=True,
    check=True
)

print(result.stdout.strip())
```

输出：

```text
123
```

逐个理解：

| 部分                  | 含义                                 |
| --------------------- | ------------------------------------ |
| `sys.executable`      | 当前 Python 解释器的可执行路径       |
| `"-c"`                | 告诉 Python 执行一段命令行提供的代码 |
| `"print(123)"`        | 要执行的 Python 代码                 |
| `capture_output=True` | 捕获输出                             |
| `text=True`           | 把输出作为文本                       |
| `check=True`          | 子进程异常退出时抛出异常             |
| `.stdout`             | 标准输出内容                         |
| `.strip()`            | 去掉字符串两端的空白和换行           |

还可以设置超时：

```python
result = subprocess.run(
    [sys.executable, "-c", "print(123)"],
    capture_output=True,
    text=True,
    check=True,
    timeout=10
)
```

其中：

```python
timeout=10
```

要求执行过程遵守约 10 秒的超时限制，但进程创建等底层步骤在部分平台上不一定能立即被中断。

通常应优先传入参数列表，并保持默认的：

```python
shell=False
```

而不是将外部输入拼成一大段 shell 命令。

Python 官方文档也明确推荐使用 `subprocess.run()` 处理一般子进程调用。

Python 3.14.10 documentation

### 9.7 其他系统相关函数

这些不一定每天都用，但认识一下很有帮助。

| 函数                 | 用途                                   | 注意事项                   |
| -------------------- | -------------------------------------- | -------------------------- |
| `os.urandom(n)`      | 获取 `n` 个安全随机字节                | 返回 `bytes`               |
| `os.getlogin()`      | 获取登录终端关联的用户名               | 某些环境可能失败           |
| `os.getuid()`        | 获取 Unix 用户 ID                      | Windows 通常不支持         |
| `os.getgid()`        | 获取 Unix 用户组 ID                    | Windows 通常不支持         |
| `os.kill(pid, sig)`  | 向进程发送信号或执行平台相关的进程操作 | 平台差异大                 |
| `os._exit(code)`     | 直接终止进程                           | 不执行普通 Python 清理流程 |
| `os.startfile(path)` | 使用 Windows 文件关联方式打开文件      | 仅 Windows                 |

例如：

```python
import os

random_bytes = os.urandom(16)

print(random_bytes)
```

它返回 16 个随机字节。

如果你是为了生成密码、令牌等内容，通常更推荐高级接口：

```python
import secrets

token = secrets.token_hex(16)

print(token)
```

因为 `secrets` 更贴近安全令牌生成的常见需求。

对于获取用户名，通常也更推荐：

```python
import getpass

print(getpass.getuser())
```

而不是无条件依赖 `os.getlogin()`。

## 第十章：进阶知识——文件描述符和底层文件操作

这一章比前面的内容更接近操作系统原理。

如果你目前主要在学习基础 Python，可以先理解它们的区别，不必立即掌握全部细节。

### 10.1 为什么已经有 `open()`，还要有 `os.open()`？

平时读取文件，我们这样写：

```python
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

这里的 `open()` 是 Python 内置函数。

它返回文件对象。

而：

```python
os.open()
```

是更底层的接口。

它返回文件描述符，通常是一个整数。

简单理解：

- `open()` 提供方便的 Python 文件对象。
- `os.open()` 让你更直接地操作文件描述符以及底层打开标志。

### 10.2 `os.open()`：底层打开文件

语法：

```python
os.open(path, flags, mode=0o777)
```

其中 `flags` 控制打开方式。

常见标志：

| 标志          | 含义                              |
| ------------- | --------------------------------- |
| `os.O_RDONLY` | 只读                              |
| `os.O_WRONLY` | 只写                              |
| `os.O_RDWR`   | 读写                              |
| `os.O_CREAT`  | 文件不存在时创建                  |
| `os.O_EXCL`   | 与 `O_CREAT` 配合，要求目标不存在 |
| `os.O_TRUNC`  | 打开时截断文件内容                |
| `os.O_APPEND` | 追加写入                          |

例如：

```python
import os

fd = os.open(
    "data.txt",
    os.O_WRONLY | os.O_CREAT | os.O_TRUNC,
    0o666
)

try:
    os.write(fd, b"hello")
finally:
    os.close(fd)
```

执行后，`data.txt` 中会写入：

```text
hello
```

逐步解释：

```python
os.O_WRONLY
```

表示以只写方式打开。

```python
os.O_CREAT
```

表示文件不存在时创建。

```python
os.O_TRUNC
```

表示如果文件原来存在，打开时把原有内容截断。

```python
|
```

通过按位或组合多个标志。

```python
fd = os.open(...)
```

获取文件描述符。

```python
os.write(fd, b"hello")
```

使用文件描述符写入字节数据。

这里为什么是：

```python
b"hello"
```

而不是：

```python
"hello"
```

因为 `os.write()` 接受的是字节数据，而不是普通文本字符串。

你可以使用：

```python
"你好".encode("utf-8")
```

把字符串编码成字节。

如果手动使用 `os.open()`，记得关闭文件描述符：

```python
os.close(fd)
```

通常应配合 `try...finally`，防止中途异常导致资源未释放。

### 10.3 `os.read()`：读取底层字节数据

语法：

```python
os.read(fd, n)
```

其中：

- `fd` 是文件描述符。
- `n` 是本次最多读取的字节数。

例如：

```python
import os

fd = os.open("data.txt", os.O_RDONLY)

try:
    data = os.read(fd, 100)
    print(data.decode("utf-8"))
finally:
    os.close(fd)
```

这段代码会从当前文件偏移处最多读取 100 个字节。

注意：一次 `os.read()` 不保证读取到完整文件，也不保证一定读取到指定的字节数量。

如果需要处理整个文件，可以循环读取直到返回空字节串。

但对于普通文本文件，通常没必要这样写。

直接使用：

```python
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

更简单。

### 10.4 `os.lseek()`：移动文件读写位置

语法：

```python
os.lseek(fd, offset, whence)
```

三个参数：

| 参数     | 含义             |
| -------- | ---------------- |
| `fd`     | 文件描述符       |
| `offset` | 偏移量           |
| `whence` | 以哪里为基准移动 |

常用：

```python
os.SEEK_SET  # 文件开头
os.SEEK_CUR  # 当前读写位置
os.SEEK_END  # 文件结尾
```

例如：

```python
import os

fd = os.open("data.txt", os.O_RDONLY)

try:
    first = os.read(fd, 5)

    os.lseek(fd, 0, os.SEEK_SET)

    second = os.read(fd, 5)

    print(first)
    print(second)
finally:
    os.close(fd)
```

这里：

```python
os.lseek(fd, 0, os.SEEK_SET)
```

表示把文件偏移位置重新移动到开头。

所以，假设文件至少有 5 个字节，两次读取的内容会相同。

但是，并不是所有文件描述符都支持随机定位，例如某些管道就不支持。

### 10.5 `os.fsync()`：请求同步文件数据

语法：

```python
os.fsync(fd)
```

它用于请求操作系统将对应文件描述符的相关缓冲数据同步到存储设备。

例如：

```python
import os

with open("important.txt", "w", encoding="utf-8") as f:
    f.write("important data")

    f.flush()
    os.fsync(f.fileno())
```

这里有三个不同概念：

```python
f.write(...)
```

向文件对象写入数据。

```python
f.flush()
```

把 Python 层的缓冲数据刷新到底层文件接口。

```python
os.fsync(f.fileno())
```

请求同步底层文件数据和必要的元数据。

其中：

```python
f.fileno()
```

获取文件对象关联的文件描述符。

它通常用于对数据持久性有要求的程序。

但仍需注意：实际持久性还与文件系统、存储硬件及同步目录元数据等因素有关，`fsync()` 不是对所有故障场景的绝对保证。

## 第十一章：常用函数组合实战

真正有用的 Python 编程，不是一个一个孤立地调用函数，而是将它们组合起来完成工作。

下面给出三个由浅入深的例子。

### 实战一：自动创建项目文件夹

假设你有一个项目，每次初始化时都需要：

```text
my_project/
├── data/
│   ├── raw/
│   └── processed/
├── logs/
└── output/
```

可以这样写：

```python
import os

project = "my_project"

folders = [
    "data/raw",
    "data/processed",
    "logs",
    "output"
]

for folder in folders:
    full_path = os.path.join(project, folder)

    os.makedirs(full_path, exist_ok=True)

    print("目录已准备好:", full_path)
```

#### 分析这段程序

第一步：

```python
project = "my_project"
```

定义项目根目录。

第二步：

```python
folders = [...]
```

列出所有希望创建的子目录。

第三步：

```python
for folder in folders:
```

遍历这些目录名称。

第四步：

```python
full_path = os.path.join(project, folder)
```

拼接完整目标路径。

第五步：

```python
os.makedirs(full_path, exist_ok=True)
```

创建必要的目录，并允许目录已经存在。

这个例子非常适合应用在：

- 数据分析项目。
- 机器学习项目。
- 文件自动处理程序。
- 自动化实验。
- 日志系统。

### 实战二：批量统计文件类型、数量和大小

假设你希望统计一个目录树中 `.py`、`.txt`、`.csv` 等文件的数量，并计算总大小。

可以使用：

```python
import os

folder = "my_project"

counts = {}
total_size = 0

def handle_error(error):
    print("无法访问:", error.filename, error)

for root, dirs, files in os.walk(
    folder,
    onerror=handle_error
):
    for filename in files:
        full_path = os.path.join(root, filename)

        if os.path.islink(full_path):
            continue

        if not os.path.isfile(full_path):
            continue

        name, extension = os.path.splitext(filename)

        extension = extension.lower()

        if extension == "":
            extension = "[无扩展名]"

        counts[extension] = counts.get(extension, 0) + 1

        try:
            total_size += os.path.getsize(full_path)
        except OSError as error:
            print("无法获取大小:", full_path, error)

print("文件类型统计:")

for extension, count in sorted(counts.items()):
    print(extension, ":", count)

print("总大小:", total_size, "字节")
```

#### 关键代码详细解释

首先：

```python
counts = {}
```

创建一个空字典，用于保存扩展名与数量的对应关系。

例如最终可能是：

```text
{
    ".py": 12,
    ".txt": 8,
    ".csv": 5
}
```

然后：

```python
for root, dirs, files in os.walk(folder):
```

递归遍历目录。

接着：

```python
for filename in files:
```

遍历当前目录中的非目录条目名称。

为了避免直接跟随符号链接，我们写：

```python
if os.path.islink(full_path):
    continue
```

`continue` 表示跳过本次循环，继续处理下一个条目。

之后：

```python
if not os.path.isfile(full_path):
    continue
```

跳过不属于普通文件的其他条目。

然后：

```python
name, extension = os.path.splitext(filename)
```

获得文件扩展名。

再看这一句：

```python
counts[extension] = counts.get(extension, 0) + 1
```

它其实很好理解。

假设第一次遇到 `.txt`：

```python
counts.get(".txt", 0)
```

因为 `.txt` 不存在，返回默认值 `0`。

所以：

```python
counts[".txt"] = 0 + 1
```

变成：

```text
{".txt": 1}
```

第二次遇到 `.txt`：

```python
counts.get(".txt", 0)
```

得到 `1`。

于是：

```python
counts[".txt"] = 1 + 1
```

结果：

```text
{".txt": 2}
```

最后：

```python
total_size += os.path.getsize(full_path)
```

把文件大小累加起来。

这个例子整合了：

`os.walk()`、`os.path.join()`、`os.path.splitext()`、`os.path.islink()`、`os.path.isfile()`、`os.path.getsize()`。

它统计的是遍历过程中成功获取大小的普通文件，不包括被跳过的符号链接目标，也不包含目录条目本身占用的空间。文件系统在扫描期间发生变化时，统计也可能不完全一致。

### 实战三：安全地批量整理下载文件

假设你的下载目录非常混乱：

```text
Downloads/
├── photo.jpg
├── cat.png
├── notes.txt
├── report.pdf
└── budget.xlsx
```

你希望自动分类成：

```text
Downloads/
├── Images/
│   ├── photo.jpg
│   └── cat.png
├── Documents/
│   ├── notes.txt
│   ├── report.pdf
│   └── budget.xlsx
```

对于这种会改变真实文件位置的任务，建议先设置预览模式。

下面的代码默认只打印计划，不会移动任何文件。

```python
import os
import shutil

source = os.path.expanduser("~/Downloads")

dry_run = True

categories = {
    ".jpg": "Images",
    ".jpeg": "Images",
    ".png": "Images",
    ".gif": "Images",

    ".txt": "Documents",
    ".pdf": "Documents",
    ".docx": "Documents",
    ".xlsx": "Documents"
}

if not os.path.isdir(source):
    raise NotADirectoryError(source)

for filename in os.listdir(source):

    src = os.path.join(source, filename)

    if not os.path.isfile(src):
        continue

    if os.path.islink(src):
        continue

    _, extension = os.path.splitext(filename)

    extension = extension.lower()

    category = categories.get(extension)

    if category is None:
        continue

    target_folder = os.path.join(source, category)

    dst = os.path.join(target_folder, filename)

    if os.path.lexists(dst):
        print("跳过，目标已存在:", dst)
        continue

    if dry_run:
        print("计划移动:", src, "->", dst)
    else:
        os.makedirs(target_folder, exist_ok=True)

        try:
            shutil.move(src, dst)
            print("已移动:", filename)
        except OSError as error:
            print("移动失败:", filename, error)
```

#### 重点理解

首先：

```python
source = os.path.expanduser("~/Downloads")
```

获取当前用户主目录中的 `Downloads` 路径。

这比直接写：

```python
"C:/Users/Alice/Downloads"
```

更灵活。

但用户实际下载目录也可能被系统重新定位，所以这里采用的是常见默认位置，而不保证所有电脑都是如此。

然后：

```python
categories = {...}
```

建立扩展名与文件类型的对应关系。

例如：

```python
".pdf": "Documents"
```

代表 PDF 归入 `Documents`。

再通过：

```python
_, extension = os.path.splitext(filename)
```

提取扩展名。

这里的下划线：

```python
_
```

只是一个普通变量名，按照惯例表示“这个返回值我暂时不需要使用”。

然后：

```python
category = categories.get(extension)
```

查询文件属于哪个类别。

如果查询不到，返回 `None`，直接跳过。

最关键的两步：

```python
target_folder = os.path.join(source, category)

dst = os.path.join(target_folder, filename)
```

生成目标目录和目标文件路径。

最后：

```python
if dry_run:
    print(...)
else:
    ...
```

用于区分预览模式和实际执行模式。

当你确认计划合理以后，才把：

```python
dry_run = True
```

改成：

```python
dry_run = False
```

不过这个示例属于个人本地文件整理脚本，并不是具有并发安全保证的文件管理系统。`exists` 检查之后目标仍可能被其他程序创建，`shutil.move()` 在某些情况下也可能覆盖目标。因此，不应直接把它用于多人共享目录或其他高风险数据。

## 第十二章：`os` 和其他 Python 库应该怎么搭配？

这个问题很重要，因为现代 Python 文件处理通常不只使用 `os`。

### 12.1 `os` + `shutil`

适合：

文件复制、移动、目录树删除。

```python
import os
import shutil

source = "data.txt"
target_dir = "backup"

os.makedirs(target_dir, exist_ok=True)

target = os.path.join(target_dir, "data.txt")

shutil.copy2(source, target)
```

这里两个库分工很明确：

`os` 创建目录、拼接路径。

`shutil` 负责实际复制。

### 12.2 `os` + `datetime`

适合：

文件修改时间、按日期生成文件名。

```python
import os
from datetime import datetime

filename = "report.txt"

timestamp = os.path.getmtime(filename)

dt = datetime.fromtimestamp(timestamp)

print(dt.strftime("%Y-%m-%d %H:%M:%S"))
```

`strftime()` 可以根据格式字符串控制输出方式。

例如：

```text
2026-11-02 10:30:00
```

只是格式示例，实际时间取决于文件属性和本地时区。

### 12.3 `os` + `glob`

适合：

按照通配符寻找文件。

例如：

```python
import glob

files = glob.glob("data/*.csv")

print(files)
```

这表示搜索 `data` 目录中匹配 `*.csv` 模式的条目。

`*` 表示零个或多个匹配字符。

这里要注意，`glob` 模式匹配与 `os.listdir()` 有区别。

```python
os.listdir("data")
```

是列出目录条目。

而：

```python
glob.glob("data/*.csv")
```

是寻找满足指定路径模式的条目。

如果需要递归匹配：

```python
files = glob.glob(
    "data/**/*.csv",
    recursive=True
)
```

不过 `glob` 和 `pathlib` 对隐藏文件等细节的处理可能有所差异。

### 12.4 `os` + `pathlib`

这里要介绍现代 Python 中一个非常值得学习的模块：

`pathlib`。

它不是对 `os` 的简单复制，而是提供了更加面向对象的路径操作风格。

例如，传统写法：

```python
import os

path = os.path.join("data", "report.txt")

if os.path.exists(path):
    print(os.path.getsize(path))
```

`pathlib` 写法：

```python
from pathlib import Path

path = Path("data") / "report.txt"

if path.exists():
    print(path.stat().st_size)
```

注意：

```python
Path("data") / "report.txt"
```

这里的 `/` 不是数学除法，而是 `Path` 对象重载的路径拼接运算符。

对比：

| 操作       | `os` 写法                       | `pathlib` 写法                               |
| ---------- | ------------------------------- | -------------------------------------------- |
| 拼接路径   | `os.path.join(a, b)`            | `Path(a) / b`                                |
| 判断存在   | `os.path.exists(p)`             | `Path(p).exists()`                           |
| 判断文件   | `os.path.isfile(p)`             | `Path(p).is_file()`                          |
| 获取文件名 | `os.path.basename(p)`           | `Path(p).name`                               |
| 获取父目录 | `os.path.dirname(p)`            | `Path(p).parent`                             |
| 获取扩展名 | `os.path.splitext(p)[1]`        | `Path(p).suffix`                             |
| 创建目录   | `os.makedirs(p, exist_ok=True)` | `Path(p).mkdir(parents=True, exist_ok=True)` |

#### 到底该学哪个？

如果你正在系统学习 Python，我建议：

先理解 `os` 和 `os.path` 的基本工作方式，再学习 `pathlib`。

因为 `os` 能让你清楚理解操作系统、工作目录、文件路径和文件描述符之间的关系。

而 `pathlib` 在很多日常文件处理场景中更加直观。

它们并不是互相排斥的。

更何况，多数常见 `os` 路径操作可以直接接受 `Path` 对象，因为它们支持 Python 的 `os.PathLike` 协议。

Python 3.15.2 documentation

例如：

```python
import os
from pathlib import Path

path = Path("data") / "report.txt"

print(os.path.exists(path))
```

也是有效的。

## 第十三章：最容易踩的 12 个坑

这一章相当于前面知识的重点复习。

| 常见错误                                       | 为什么会错                   | 正确理解或处理                   |
| ---------------------------------------------- | ---------------------------- | -------------------------------- |
| 认为 `os.getcwd()` 是脚本所在目录              | 当前工作目录由进程状态决定   | 需要脚本目录时考虑 `__file__`    |
| 使用 `"C:\new\test"`                           | `\n`、`\t` 等被转义          | 使用原始字符串或路径拼接         |
| 使用 `os.mkdir("a/b/c")`                       | 父目录可能不存在             | 使用 `os.makedirs()`             |
| 使用 `os.path.join()` 后以为文件已经创建       | `join()` 只构造路径          | 创建目录或文件需要其他操作       |
| 把 `os.listdir()` 的名称直接当完整路径         | 它通常只返回子项名称         | 与目录路径进行 `join()`          |
| 认为 `os.path.exists()` 只检查文件             | 也能检查目录等对象           | 必要时使用 `isfile()`、`isdir()` |
| 把 `os.path.getctime()` 当成所有系统的创建时间 | Unix 与 Windows 语义不同     | 按操作系统理解 ctime             |
| 认为 `os.remove()` 可以删除整个文件夹          | 它主要删除非目录文件         | 空目录用 `rmdir()`               |
| 认为 `os.rename()` 永远不会覆盖                | Unix 与 Windows 存在差异     | 需要时使用明确的替换策略         |
| 认为 `os.walk()` 的文件名是完整路径            | 返回的名称列表不包含完整路径 | 使用 `os.path.join(root, name)`  |
| 认为 `os.environ` 中的数字自动成为整数         | 环境变量值通常是字符串       | 使用 `int()` 等转换              |
| 认为 `os.system()` 返回命令输出                | 返回的是系统相关的状态值     | 使用 `subprocess.run()` 捕获输出 |

再补充几个更深入的注意点。

### 13.1 `exists()` 后再操作，并不保证安全

例如：

```python
if os.path.exists(path):
    os.remove(path)
```

检查和删除之间存在时间间隔。

另一个程序可能在中间改变文件系统状态。

这类问题常被称为 TOCTOU（Time of Check to Time of Use），即检查时刻和使用时刻之间的竞争问题。

通常更合理的处理方式是直接尝试操作，并捕获预期异常：

```python
try:
    os.remove(path)
except FileNotFoundError:
    pass
```

但如果路径位于不可信用户可以修改的目录中，单纯捕获异常也不构成完整的安全保证。

### 13.2 `normpath()` 不等于安全路径验证

例如：

```python
import os

base = "/srv/uploads"

user_input = "../../secret.txt"

path = os.path.join(base, user_input)

print(os.path.normpath(path))
```

在 Linux 中可能得到：

```text
/secret.txt
```

这说明：

```python
os.path.join()
```

不会自动限制路径一定留在某个目录内部。

```python
os.path.normpath()
```

也只是路径规范化，不是安全过滤器。

如果程序需要处理不可信的上传路径，必须考虑路径穿越、符号链接以及文件系统竞争问题。

### 13.3 不要忽略文件编码

很多初学者认为：

```python
os.path.exists("中文.txt")
```

与：

```python
open("中文.txt")
```

使用的是相同层面的编码处理。

其实有区别。

文件名编码由 Python 的文件系统接口处理；文件内容的编码则由 `open()` 等文本 I/O 功能处理。

因此，操作文本文件通常建议：

```python
with open("中文.txt", "r", encoding="utf-8") as f:
    text = f.read()
```

`encoding="utf-8"` 指的是如何解码文件内容，不是在指定路径字符串本身的编码。

### 13.4 大型目录不要轻易一次性读取和存储所有结果

例如：

```python
all_files = []

for root, dirs, files in os.walk("very_large_folder"):
    for name in files:
        all_files.append(os.path.join(root, name))
```

如果目录树包含几百万个文件，列表可能占用大量内存。

很多时候，可以边遍历边处理：

```python
for root, dirs, files in os.walk("very_large_folder"):
    for name in files:
        path = os.path.join(root, name)

        # 立即处理当前文件
        print(path)
```

这体现了迭代处理的优点：不必先把全部结果存储下来。

## 第十四章：常用方法速查表

前面的内容已经比较多了，这里将常用知识放在一起，方便你复习。

### 交互式 `os` 函数速查

路径操作 / 检查与属性 / 文件与目录操作 / 环境、进程和底层操作

#### 路径操作

| 函数 / 属性                 | 用途                     |
| --------------------------- | ------------------------ |
| `os.path.join(a, b)`        | 拼接路径                 |
| `os.path.abspath(p)`        | 转为绝对路径             |
| `os.path.basename(p)`       | 获取最后一个路径组成部分 |
| `os.path.dirname(p)`        | 获取目录部分             |
| `os.path.split(p)`          | 拆分目录和文件名         |
| `os.path.splitext(p)`       | 拆分最后一个扩展名       |
| `os.path.relpath(p, start)` | 计算相对路径             |
| `os.path.normpath(p)`       | 规范化路径字符串         |
| `os.path.realpath(p)`       | 解析实际路径             |
| `os.path.expanduser(p)`     | 展开用户主目录           |

## 第十五章：通过练习真正掌握 `os`

理解一个函数的最好方式，是自己动手修改代码并观察结果。

下面安排五道循序渐进的练习。

1. 获取当前工作目录，并打印该目录中所有普通文件的名称。

   需要 `getcwd()`、`listdir()`、`join()` 和 `isfile()`。

   **查看参考答案**（原交互按钮）

2. 创建 `logs/2026/november` 目录，要求重复运行不报目录已存在错误。

   使用 `makedirs()` 的 `exist_ok` 参数。

   **查看参考答案**（原交互按钮）

3. 给定 `report.final.pdf`，分离最后一个扩展名，并生成 `report.final.txt` 这个新名字。

   用 `splitext()` 解包两个返回值。

   **查看参考答案**（原交互按钮）

4. 递归查找一个目录中所有 `.py` 文件，并打印完整绝对路径。

   使用 `walk()` 遍历；用 `join()` 和 `abspath()` 得到路径。

   **查看参考答案**（原交互按钮）

5. 编写代码读取 `APP_PORT` 环境变量；不存在时使用 `8000`，并把结果转换为整数。

   `getenv()` 返回的通常是字符串。

   **查看参考答案**（原交互按钮）

## 最后：应该按照什么顺序学习？

如果你希望真正掌握，而不是今天看完明天忘掉，我建议分四个阶段。

| 阶段                     | 应重点掌握的知识                                             | 能完成的任务             |
| ------------------------ | ------------------------------------------------------------ | ------------------------ |
| 第一阶段：路径基础       | `getcwd`、`join`、`abspath`、`basename`、`dirname`、`splitext` | 正确定位和处理文件路径   |
| 第二阶段：文件管理       | `exists`、`isfile`、`mkdir`、`makedirs`、`listdir`、`remove`、`rename` | 自动创建、整理、检查文件 |
| 第三阶段：目录遍历与属性 | `scandir`、`walk`、`stat`、`getsize`、`getmtime`             | 批量处理和统计文件       |
| 第四阶段：系统交互       | `environ`、`getenv`、`getpid`、`subprocess`、文件描述符      | 开发较完整的自动化程序   |

我尤其建议优先熟练掌握下面这七个：

```python
os.getcwd()
os.path.join()
os.path.exists()
os.makedirs()
os.listdir()
os.scandir()
os.walk()
```

因为它们贯穿了许多非常典型的文件操作任务。

最后再用一句话概括 `os` 的整体设计逻辑：

`os.path` 帮助你描述、计算和判断文件路径；`os` 帮助 Python 与操作系统交互，执行目录管理、文件属性查询、环境管理和底层操作。

只要始终区分“我现在是在计算一个路径字符串”，还是“我现在真的在修改文件系统”，再理解当前工作目录的作用，你就已经抓住了学习 `os` 最核心的思路。

官方参考资料：[Python `os` 文档](https://docs.python.org/3.14/library/os.html?utm_source=chatgpt.com)、[Python `os.path` 文档](https://docs.python.org/3.14/library/os.path.html?utm_source=chatgpt.com)。本教程以 Python 3 为基础，具体版本和操作系统可能存在少量 API 或行为差异。