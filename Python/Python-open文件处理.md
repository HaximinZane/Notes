# Python `open()` 文件处理超详细教程：从字节、编码和文件位置，到读写、追加与可靠保存

## 零基础先导：一个文件名、文件对象、文件内容，不是一件东西

假设你写下 `"notes.txt"`，这只是一段路径字符串；它不会自动打开文件。`open("notes.txt")` 才会让 Python 请求操作系统打开目标，并返回**文件对象**。从文件对象调用 `read()` 才会真正把内容读到内存。三个步骤一定要分开：

```python
path = "notes.txt"                         # ① 只是文件路径
with open(path, "r", encoding="utf-8") as f: # ② 打开文件，f 是文件对象
    content = f.read()                       # ③ 读取到字符串
print(content)                               # ④ with 已退出，内容仍在变量中
```

`with` 主要负责退出时释放资源；不要误会成“退出后字符串也会消失”。`"r"` 是读文本；`"rb"` 是读二进制；`"w"` 创建或**截断**后写入；`"a"` 通常表示追加。不同模式涉及覆盖风险，写之前先确认。

`encoding="utf-8"` 告诉 Python 如何把**文件里的字节**转换为**Python 字符串**。编码不匹配可能出现乱码或 `UnicodeDecodeError`，不是把所有内容强转成 UTF-8 就能修好。后文中的 `seek`、`tell`、`newline` 和缓冲都是在解释：文件对象如何记录当前位置，以及字符、字节、行之间如何转换。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

适合已经学过变量、字符串、循环和函数，正在学习文件操作的读者。

这一篇从一行最常见的代码开始：

```python
with open("notes.txt", "r", encoding="utf-8") as f:
    text = f.read()
```

看起来只有两行，却涉及路径、打开模式、字符编码、文件对象、上下文管理、读取位置和资源释放。把这些概念分清楚，文件操作就会从“照着写能跑”变成“知道每个参数为什么放在这里”。

本文的常规示例面向 Python 3.11 及以上版本；新增功能单独说明。示例中的文件都应放在专门的练习目录中。带有 `w` 的模式会在成功打开已有文件时清空原内容，这个动作发生得比 `write()` 更早。

## 第一章：文件、路径和文件对象，分别是什么？

### 1.1 磁盘上的文件，本质上保存字节

你看到一个文本文件，内容是：

```text
你好，Python
```

编辑器把底层字节按照某种字符编码解释成文字，再显示在屏幕上。磁盘并没有直接保存一个 Python `str` 对象。

理解下面这条关系非常重要：

```text
Python 字符串 str
    │ 编码 encode
    ▼
字节 bytes ──写入──> 文件

文件 ──读取──> 字节 bytes
                  │ 解码 decode
                  ▼
              Python 字符串 str
```

例如：

```python
text = "你好"
data = text.encode("utf-8")

print(len(text))
print(len(data))
print(data)
print(data.decode("utf-8"))
```

输出：

```text
2
6
b'\xe4\xbd\xa0\xe5\xa5\xbd'
你好
```

这里的两个汉字在 UTF-8 下占六个字节。`len(str)` 计算 Unicode 码点数量，`len(bytes)` 计算字节数量。对于带组合字符或复杂表情的文本，用户眼中的“一个字符”还可能对应多个码点，因此不要把它们统统当成同一种长度。

图片、压缩包、PDF 和可执行文件也保存字节，只是组织规则不同。文件扩展名帮助人和软件选择解释方式，却不会自动把任意字节变成那种格式。

### 1.2 路径是地址，文件对象是一次打开操作的结果

```python
filename = "notes.txt"
f = open(filename, "r", encoding="utf-8")
```

`filename` 是路径字符串，用来描述在哪里寻找文件。

`open(...)` 请求打开这个位置对应的文件，并建立可供 Python 使用的读写接口。

`f` 保存返回的文件对象。通过这个对象，可以读取内容、写入内容、查询位置或关闭资源。

因此：

```python
print(filename)
print(f)
```

前者显示地址，后者通常显示文件对象的描述。想得到文件内容，应调用：

```python
text = f.read()
```

同一个磁盘文件可以被打开多次，每次打开通常会得到独立的文件对象和读写位置。文件对象并不等于文件中的所有内容。

### 1.3 `open()` 为什么不需要导入？

`open()` 是内置函数，普通程序可以直接使用：

```python
open("notes.txt", "r", encoding="utf-8")
```

不需要 `import open`，也不需要安装名为 `open` 的包。

这里的 `open` 是“打开”。打开文件之后返回对象，再通过对象的方法读写，是为了让一次打开所涉及的状态有明确的归属：访问权限、编码、缓冲和当前位置都保存在相关 I/O 对象中。

Python 3 的流式 I/O 体系主要由 `io` 模块提供。它区分文本流、带缓冲的二进制流和原始流。初学阶段使用内置 `open()` 即可，不必自己组装这些底层对象。[官方说明：I/O 流的分类](https://docs.python.org/3.14/library/io.html#overview)

## 第二章：逐字理解 `with open(...) as f`

前面知道了文件对象的作用，现在来看最常见的完整写法。

```python
with open("notes.txt", "r", encoding="utf-8") as f:
    text = f.read()

print(text)
```

假设当前工作目录中已经有 UTF-8 文件 `notes.txt`。

先不要急着记住整行，我们从执行顺序看。

### 2.1 先执行的是哪一部分？

Python 先计算：

```python
open("notes.txt", "r", encoding="utf-8")
```

其中：

```python
"notes.txt"
```

是要访问的路径。

```python
"r"
```

表示只读文本模式。

```python
encoding="utf-8"
```

表示读取内容时，按照 UTF-8 进行解码。

如果路径不存在、权限不足等导致打开失败，程序就会在这一步抛异常，不能正常进入后面的缩进块。

打开成功后，得到一个文件对象。

接下来，`with` 会进入这个对象提供的上下文管理过程。

对于普通文件，进入时得到的就是这个文件对象本身。

### 2.2 `as f` 是什么意思？`f` 必须这样写吗？

```text
as f
```

在这里把进入上下文得到的对象绑定给变量 `f`。

于是，缩进块中可以通过：

```python
f.read()
```

使用这个文件对象。

`f` 只是普通变量名，不是关键字。

你完全可以写：

```python
with open("notes.txt", "r", encoding="utf-8") as source:
    text = source.read()
```

或者：

```python
with open("report.txt", "w", encoding="utf-8") as report_file:
    report_file.write("报告内容\n")
```

初学示例常用 `f`，因为短，容易把注意力放在读写本身。

实际程序中，`source`、`target`、`config_file` 等名称往往更能说明用途。

#### 那 `as` 是不是复制了一份文件？

没有。

它只是让一个变量引用进入上下文得到的对象。

磁盘上不会因此出现第二份文件，内存里也不会自动读入全文。

真正读取内容仍然需要执行 `read()`、`readline()` 或迭代等操作。

### 2.3 冒号和缩进表示什么？

看：

```python
with open("notes.txt", encoding="utf-8") as f:
    text = f.read()
    print("文件还在使用中")

print("已经离开 with 代码块")
```

冒号后面的缩进行属于 `with` 代码块。

这里读文件和第一条打印，都发生在文件保持打开的范围内。

最后一条打印没有这个缩进，表示已经离开该范围。

离开时，普通文件的上下文退出处理会尝试关闭文件。

所以缩进并不只是让代码好看，它直接决定哪些操作处在资源管理范围内。

### 2.4 为什么文件要关闭？

你可以把一次打开理解为：程序向操作系统取得了一个访问文件的资源。

这个资源不应在不用后一直占着。

此外，写入的内容可能还停留在 Python 的缓冲层中，需要在关闭等适当时机交给底层。

如果一个程序不断打开文件，却迟迟不关闭，最终可能遇到打开数量限制，也会让文件占用和错误定位变得更麻烦。

#### 我以前没写 `close()`，程序退出后好像也没事？

很多情况下，进程结束后操作系统会回收资源，某些 Python 实现也可能在对象不再引用时及时清理。

但这不适合当作正常程序的资源管理方式。

你无法只凭一个简单实验，确定对象会在所有解释器、所有异常路径和所有运行环境里及时关闭。

程序如果持续运行数小时，等待进程退出才释放也太晚了。

因此，应该在逻辑上不再需要文件时，就明确结束它的生命周期。

### 2.5 不用 `with`，能不能正常读文件？

可以。

最直接的写法是：

```python
f = open("notes.txt", "r", encoding="utf-8")
text = f.read()
f.close()
```

如果一切正常，这段代码可以完成读取和关闭。

但我们考虑另一种情况。

```python
f = open("notes.txt", "r", encoding="utf-8")
text = f.read()
number = int("not-a-number")
f.close()
```

`int("not-a-number")` 会抛出 `ValueError`。

正常顺序被打断，后面的 `f.close()` 就执行不到。

所以，真正需要保证的是：即使中间操作失败，也安排清理。

可以写成：

```python
f = open("notes.txt", "r", encoding="utf-8")

try:
    text = f.read()
    number = int("not-a-number")
finally:
    f.close()
```

`finally` 在正常执行或常规异常展开时都会被执行。

对于普通文件，`with` 正是把这种“在明确范围内使用，离开时清理”的安排写得更集中。

### 2.6 `with` 里面报错，文件会关闭吗？

在正常的异常传播过程中，会尝试执行退出处理并关闭。

我们可以观察：

```python
try:
    with open("notes.txt", encoding="utf-8") as f:
        print("块内是否关闭：", f.closed)
        raise RuntimeError("模拟处理中途失败")
except RuntimeError:
    print("捕获到了模拟异常")

print("块外是否关闭：", f.closed)
```

通常输出：

```text
块内是否关闭： False
捕获到了模拟异常
块外是否关闭： True
```

`closed` 是状态属性，所以读取时不加括号。

这个例子还说明：文件关闭，并不意味着异常自动被忽略。

异常仍然向外传播，由外层 `except` 处理。

#### 如果是 `return` 提前离开函数呢？

同样会安排退出。

```python
def read_notes(path):
    with open(path, encoding="utf-8") as f:
        return f.read()
```

执行 `return` 时，先计算 `f.read()`，取得字符串。

然后离开上下文，尝试关闭文件。

最后把字符串交给调用者。

所以，这种在 `with` 内返回已经读入的数据的写法很常见。

#### `break` 呢？

如果 `break` 导致离开文件所属的 `with` 范围，也会执行相应退出过程。

如果 `break` 只是退出 `with` 内部的一层循环，而后面仍在同一个 `with` 中，文件就仍然保持打开，直到真正离开上下文。

关键要看控制流是否已经离开 `with`，而不是看到某个提前退出关键字就一概而论。

### 2.7 离开 `with` 后，`f` 这个变量还在吗？

通常还在。

`with` 不像函数定义那样创建一套新的局部变量作用域。

```python
with open("notes.txt", encoding="utf-8") as f:
    text = f.read()

print(f.closed)
print(text)
```

第一条打印可以得到：

```text
True
```

第二条仍然可以打印已经读入的文本。

但如果你再执行：

```python
f.read()
```

会因为文件已经关闭而报错。

因此要把两件事分清楚：

变量 `f` 仍然引用一个对象。

这个对象所管理的文件资源已经关闭。

对象还在，不表示它仍然可以继续读写。

### 2.8 我把 `f` 返回出去，让别的函数继续读，可以吗？

如果你这样写：

```python
def open_notes(path):
    with open(path, encoding="utf-8") as f:
        return f
```

调用者拿到的通常是已经关闭的文件对象。

因为函数返回也会离开 `with`，触发关闭。

如果你只是需要内容，就在里面返回 `f.read()`。

如果你的接口确实要让调用者管理流，应该明确设计谁打开、谁关闭，而不是一边在内部用 `with` 关闭，一边要求外部继续读取。

后面讲文件类对象和生成器时，会继续说明这类资源所有权问题。

### 2.9 使用 `with`，是不是就绝对不会丢数据？

还不能这样理解。

`with` 帮助安排退出处理。

它不负责把失败的写入撤销，也不负责给多文件更新提供事务。

例如，以 `w` 打开已有文件后，原内容可能已经被截断。

后面出错，即使文件正确关闭，也不会自动恢复原内容。

机器断电、进程被强制终止等情况，也可能根本没有机会执行 Python 清理过程。

此外，关闭本身也可能因为底层 I/O 错误而失败。

所以，“使用了上下文管理”和“完整结果已经可靠发布”，需要分别理解。

我们后面会从缓冲、同步和临时文件替换继续讲这些区别。


## 第三章：`open()` 的参数到底控制什么？

### 3.1 完整调用形式

```python
open(
    file,
    mode="r",
    buffering=-1,
    encoding=None,
    errors=None,
    newline=None,
    closefd=True,
    opener=None,
)
```

这是一份参数结构展示。日常代码通常只需要前两个参数和 `encoding`，处理 CSV 时还经常需要 `newline`。

| 参数 | 作用 | 常见写法 |
| --- | --- | --- |
| `file` | 路径，或需要包装的文件描述符 | `"data.txt"`、`Path(...)` |
| `mode` | 读写方式、创建方式、文本或二进制 | `"r"`、`"w"`、`"ab"` |
| `buffering` | 缓冲策略 | 一般保留默认 |
| `encoding` | 文本内容的编码 | `"utf-8"` |
| `errors` | 编解码错误的处理方式 | 默认严格报错 |
| `newline` | 文本换行识别与转换规则 | `None`、`""`、`"\n"` |
| `closefd` | 关闭包装对象时是否关闭传入的描述符 | 主要用于底层接口 |
| `opener` | 自定义底层打开逻辑的可调用对象 | 普通业务很少需要 |

`file` 接受 `Path` 对象，是因为它支持路径协议；并不是所有任意对象都能直接传给 `open()`。

### 3.2 `open("data.txt")` 到底会去哪里找文件？

先看一句最普通的代码：

```python
with open("data.txt", "r", encoding="utf-8") as f:
    text = f.read()
```

这里的 `"data.txt"` 只有文件名，没有写它在哪个盘、哪个目录。

那么，Python 怎么知道应该打开哪一个 `data.txt`？

它通常根据**当前工作目录**来解释这个相对路径。

注意，“当前工作目录”是一项进程运行状态，不是“当前这份 Python 文件所在的目录”的另一种叫法。

我们用一个完整情境把这件事看清楚。

#### 情境一：你先进入项目目录，再启动脚本

假设你的项目放在：

```text
D:\project
```

项目结构是：

```text
D:\project\
├── main.py
└── data.txt
```

你在 Windows 终端中先进入：

```text
D:\project
```

然后执行：

```powershell
python main.py
```

假设没有其他代码修改工作目录，那么脚本里的：

```python
open("data.txt", "r", encoding="utf-8")
```

通常会去找：

```text
D:\project\data.txt
```

这一次正好找到了脚本旁边的文件。

于是，很容易产生一个印象：`open("data.txt")` 就是在打开“脚本旁边的 data.txt”。

但这次成功，是因为**工作目录恰好与脚本目录相同**。

我们换一种启动方式，再看一次。

#### 情境二：你停留在用户目录，用完整路径运行脚本

脚本的位置仍然是：

```text
D:\project\main.py
```

输入文件的位置也没有变：

```text
D:\project\data.txt
```

但是，你的终端当前处于：

```text
C:\Users\Alice
```

然后执行：

```powershell
python D:\project\main.py
```

这条命令告诉 Python：“去运行 D 盘项目中的 main.py。”

它没有自动附带“先把当前工作目录切换到 D:\project”这一步。

所以，如果启动环境和脚本都没有另行修改工作目录，那么：

```python
import os

print(os.getcwd())
```

通常输出：

```text
C:\Users\Alice
```

这时，脚本中的：

```python
open("data.txt", "r", encoding="utf-8")
```

通常寻找的是：

```text
C:\Users\Alice\data.txt
```

而项目里真正的输入文件是：

```text
D:\project\data.txt
```

两个位置不同。

如果用户目录下没有同名文件，就会触发 `FileNotFoundError`。

如果用户目录下恰好也有一个 `data.txt`，情况甚至更隐蔽：程序可能顺利运行，却读取了另一份内容。

因此，遇到“明明文件就在脚本旁边，为什么说不存在”，先检查程序实际找的是哪个位置。

#### 那么，能不能直接写完整路径？

可以。

例如，在原生 Windows Python 中：

```python
with open(r"D:\project\data.txt", "r", encoding="utf-8") as f:
    text = f.read()
```

这次提供了明确地址，普通情况下就不再依靠当前工作目录定位。

前面的 `r` 是原始字符串标记，让反斜杠更方便地按字面表达，避免 `\n`、`\t` 等转义影响路径。

但把自己的盘符和目录写死之后，项目复制到另一台电脑，地址可能不再成立。

所以，对于“资源固定放在脚本旁边”的需求，通常还可以进一步写得更灵活。

#### 如果想打开脚本所在目录里的文件呢？

假设项目结构是：

```text
D:\project\
├── main.py
└── data\
    └── input.txt
```

我们希望：无论从哪个终端目录启动，都读取这份项目内的 `input.txt`。

可以在普通 `.py` 脚本中这样写：

```python
from pathlib import Path

script_path = Path(__file__).resolve()
script_dir = script_path.parent
input_path = script_dir / "data" / "input.txt"

with open(input_path, "r", encoding="utf-8") as f:
    text = f.read()

print(text)
```

下面逐行理解，不要急着把它压成一行。

首先：

```python
__file__
```

它是当前模块文件的位置资料。

在这个例子里，我们使用的是 `main.py` 所在位置。

然后：

```python
Path(__file__)
```

把这个位置交给 `pathlib`，构造一个路径对象。

接着：

```text
.resolve()
```

取得绝对形式，并按照实际文件系统规则解析符号链接等路径组成。

在本例的普通布局下，`script_path` 对应：

```text
D:\project\main.py
```

下一行：

```python
script_dir = script_path.parent
```

取文件所在的父目录。

于是得到：

```text
D:\project
```

再看：

```python
input_path = script_dir / "data" / "input.txt"
```

从脚本目录出发，进入 `data`，再指向 `input.txt`。

最终得到：

```text
D:\project\data\input.txt
```

最后：

```text
with open(input_path, "r", encoding="utf-8") as f:
```

打开这条明确构造的路径。

即使终端仍然停在：

```text
C:\Users\Alice
```

我们也已经主动指定了项目文件的位置。

熟悉之后，前两步经常合写为：

```python
base_dir = Path(__file__).resolve().parent
```

但理解它的时候，最好仍能把这条链拆成“模块文件位置 → 路径对象 → 绝对解析 → 父目录”四步。

#### 那是不是以后所有文件都应该相对于脚本目录？

也需要看文件用途。

假设你写的是一个命令行工具，用户在终端输入：

```powershell
python D:\tools\count_lines.py .\notes.txt
```

用户通常想处理自己当前目录下的 `notes.txt`，而不是 `D:\tools` 下的同名文件。

这时，相对于当前工作目录解释用户传入的路径，反而符合直觉。

所以你需要先分清：

- 程序自带的资源，可以按脚本位置、包资源或约定的资源目录定位。
- 用户本次传入的相对文件名，通常按当前工作目录定位。
- 应用长期保存的数据，可以放在明确配置的数据目录。

重要的是先确定基准，再让代码始终遵守这个约定。

#### 如果在 Notebook 里运行，为什么 `__file__` 报错？

普通交互式环境和部分 Notebook 没有当前 `.py` 模块文件这个同样的语境，因此可能没有定义 `__file__`。

你不能因为上面的脚本写法在 Notebook 报 `NameError`，就认为 `pathlib` 或 Python 安装坏了。

这类环境通常需要明确指定项目根目录，或查看当前工作目录：

```python
from pathlib import Path

print(Path.cwd())
```

然后根据 Notebook 的保存位置和运行配置，选择清楚的数据路径。

#### `~`、环境变量和通配符，会不会自动展开？

再看三个很像终端命令的路径：

```python
"~/notes.txt"
"$HOME/notes.txt"
"data/*.txt"
```

直接传给 `open()`，通常不会得到 shell 自动展开的效果。

`open()` 接收的是一个文件位置，不负责把 `*.txt` 搜索成多个文件。

展开用户主目录可以写：

```python
from pathlib import Path

path = Path("~/notes.txt").expanduser()

with path.open(encoding="utf-8") as f:
    text = f.read()
```

查找多个文件则可以使用：

```python
from pathlib import Path

for path in Path("data").glob("*.txt"):
    if path.is_file():
        with path.open(encoding="utf-8") as f:
            print(path.name, len(f.read()))
```

先搜索得到各个路径，再分别打开，两个动作各自有明确职责。

### 3.3 如果路径对了，为什么仍然可能打开失败？

“地址写对”只是其中一个条件。

文件还可能没有创建，某级父目录可能不是目录，当前用户可能没有权限，或者另一个程序施加了文件共享限制。

因此，排查时可以先打印明确地址：

```python
from pathlib import Path

path = Path("data/input.txt")

print("工作目录：", Path.cwd())
print("本次目标：", path.absolute())
```

再处理实际打开步骤产生的异常。

不要因为打印出一条很完整的路径，就认为文件一定存在；也不要因为 `exists()` 此刻返回了真，就认为随后读取一定成功。

路径告诉你“打算访问哪里”，实际打开告诉你“这一次是否能访问”。

## 第四章：`r`、`w`、`a`、`x` 和 `+`，到底应该选哪个？

前面已经知道怎样找到文件。

现在还有一个问题：找到这个位置以后，你准备对它做什么？

是只读内容，还是重新写一份？

是保留旧内容继续添加，还是要求必须创建一个新文件？

`open()` 的 `mode` 参数，就是用来表达这些选择的。

### 4.1 我只是想读取文件，应该怎么写？

使用 `r`。

`r` 可以按 read，也就是“读取”来记。

```python
with open("notes.txt", "r", encoding="utf-8") as f:
    text = f.read()

print(text)
```

假设 `notes.txt` 已经存在，内容为：

```text
今天学习 Python 文件处理。
```

运行后，我们把其中的文字读入字符串，再打印出来。

文件原来的内容不会因为这次只读操作被清空。

#### 如果文件不存在呢？

例如，当前目录里没有：

```text
missing.txt
```

你执行：

```python
with open("missing.txt", "r", encoding="utf-8") as f:
    text = f.read()
```

通常在打开这一步触发：

```text
FileNotFoundError
```

原因是：你要求读取一个已有文件，但目标并不存在。

`r` 不会替你创建一份空文件，再假装读取成功。

这通常是有帮助的。否则路径拼错时，你可能得到空内容，却没有发现程序根本没读到真正输入。

#### `"r"` 能不能不写？

可以。

`open()` 默认就是只读文本模式。

下面两种写法在这里具有相同含义：

```python
open("notes.txt", "r", encoding="utf-8")
```

```python
open("notes.txt", encoding="utf-8")
```

初学时明确写 `"r"`，更容易看清你选择了什么模式。熟悉后按项目习惯省略即可。

#### 既然文件已经打开，能不能顺手写一行？

```python
with open("notes.txt", "r", encoding="utf-8") as f:
    f.write("再加一行\n")
```

通常会因为流不支持写入而报错。

打开文件时已经选择了只读能力，后续不能仅凭调用 `write()` 就自动获得写入权限。

### 4.2 我想重新保存一份结果，应该怎么写？

使用 `w`。

`w` 可以按 write，也就是“写入”来记。

```python
with open("result.txt", "w", encoding="utf-8") as f:
    f.write("这次计算的结果是 42。\n")
```

如果 `result.txt` 不存在，而且父目录存在、权限等条件允许，就会创建这个文件。

所以，刚开始写小脚本时，你会经常看到 `w`。

但这里有一个必须先讲明白的细节。

#### 如果文件原来有内容，旧内容去哪了？

假设原来的 `result.txt` 是：

```text
上一次的计算结果。
这段说明还需要保留。
```

现在执行：

```python
with open("result.txt", "w", encoding="utf-8") as f:
    f.write("新的结果。\n")
```

成功执行后，文件通常变成：

```text
新的结果。
```

原来的两行已经被截断，不会自动保留到后面。

`w` 适合“我要重新生成这份文件”的需求。

#### 是 `write()` 把它清空的吗？

这里尤其容易误解。

清空发生在**成功以 `w` 打开已有文件时**。

并不需要等到调用 `write()`。

例如：

```python
with open("result.txt", "w", encoding="utf-8") as f:
    pass
```

虽然代码块里什么都没写，已有文件仍然可能已经变成空文件。

再看：

```python
with open("result.txt", "w", encoding="utf-8") as f:
    raise RuntimeError("计算失败，还没来得及写结果")
```

异常发生后，`with` 会安排关闭文件，但不会帮你恢复被截断的旧内容。

所以，如果你要保存重要结果，应先完成数据计算和验证，再考虑打开正式输出文件。

需要更可靠的发布过程时，可以先写临时文件，成功后替换正式文件。后面会把这种方法展开。

#### 那我能不能一边读旧文件，一边用 `w` 写回同一个位置？

请看：

```python
# 反面例子：只在临时练习文件上理解，不要用于真实资料。
with open("result.txt", "r", encoding="utf-8") as source:
    with open("result.txt", "w", encoding="utf-8") as target:
        target.write(source.read())
```

外层先打开输入。

内层紧接着以 `w` 打开同一个文件。

这一步可能已经把底层文件截断。

然后外层才尝试 `read()`，旧内容当然不能再按照原先期待完整读取。

要修改小型文本，应先读到内存，完成修改，再写回或替换；较大文件则通常生成另一个输出文件。

### 4.3 我想保留旧内容，只在后面加一段呢？

使用 `a`。

`a` 来自 append，也就是“追加”。

```python
with open("events.txt", "a", encoding="utf-8") as f:
    f.write("完成一次学习任务\n")
```

假设原文件是：

```text
第一次任务完成
```

其中末尾已经有换行。

运行后得到：

```text
第一次任务完成
完成一次学习任务
```

再次运行，就会再追加一行。

#### 如果文件还不存在呢？

`a` 通常也会创建目标文件，前提仍然是父目录和权限等条件允许。

因此，简单的个人追加记录常用这种模式。

#### 它会自动帮我换行吗？

不会。

看这个例子：

```python
with open("append_demo.txt", "w", encoding="utf-8") as f:
    f.write("apple")

with open("append_demo.txt", "a", encoding="utf-8") as f:
    f.write("banana")
```

结果是：

```text
applebanana
```

因为我们两次都没有写入 `\n`。

追加模式只决定新数据写到结尾，不负责猜测你是不是想追加“一行”。

如果每条记录都应该占一行，就在记录格式中明确包含换行。

#### 我调用 `seek(0)`，能不能让追加模式改写开头？

不能把它当成这种用途。

追加模式用于追加写入。即使流允许改变读取位置，也不应该依赖它在文件中间执行覆盖写。

想做随机覆盖，应考虑 `r+` 或 `r+b`，并理解字节长度、缓冲和文件格式的限制。

#### 多个程序一起追加，一条记录就一定完整吗？

还不能直接这样保证。

操作系统的追加语义、Python 的编码与缓冲、单次底层写入大小，以及文件系统行为都可能影响结果。

一个高级 `write()` 调用，不必然等价于一条业务记录的完整事务。

个人单进程练习可以先用追加模式理解流程；多个写入者的日志系统应考虑统一写入、队列或合适的日志架构。

### 4.4 我只想创建新文件，已有同名文件就拒绝呢？

使用 `x`。

这里的含义是 exclusive creation，也就是排他创建。

```python
try:
    with open("new_report.txt", "x", encoding="utf-8") as f:
        f.write("这是新生成的报告。\n")
except FileExistsError:
    print("这个名字已经存在，请选择另一个输出文件名。")
```

第一次运行，目标不存在，就创建并写入。

第二次运行，同名位置已经存在，就报告 `FileExistsError`，而不是像 `w` 那样先清空它。

#### 我先 `exists()` 检查，再用 `w`，效果不是一样吗？

在没有其他程序参与的简单练习中，看起来可能一样。

但下面是两个分开的动作：

```python
if not path.exists():
    with path.open("w", encoding="utf-8") as f:
        f.write("内容")
```

第一步检查时，目标还不存在。

第二步打开之前，另一个程序可能创建了它。

随后你的 `w` 就可能清空那个新文件。

`x` 把“要求名字未被占用”的意图放在创建步骤中，更准确地表达需求。

当然，它仍不负责创建父目录，也不替代不可信目录中的完整安全设计。

### 4.5 `+` 是什么意思？是不是“往后加”？

看到加号，很容易联想到追加。

但文件模式中的 `+` 表示：**在基本模式上允许读和写**。

追加由基本模式 `a` 决定。

因此，你会看到：

```text
r+
w+
a+
x+
```

它们都允许读写，但原来的创建、截断或追加规则仍然保留。

#### `w+` 既能读又能写，为什么读不到旧内容？

因为 `w` 的截断规则仍然生效。

```python
with open("both.txt", "w", encoding="utf-8") as f:
    f.write("原来的内容")

with open("both.txt", "w+", encoding="utf-8") as f:
    print(repr(f.read()))
```

第二次打开时，原内容已经被截断，所以通常打印：

```text
''
```

“允许读取”与“保留旧内容”是两个不同选择。

#### 我刚刚 `write()` 了内容，接着 `read()`，为什么还是空？

看下面：

```python
with open("both.txt", "w+", encoding="utf-8") as f:
    f.write("hello")
    print(repr(f.read()))
```

写入以后，流位置已经到了写入内容后面。

下一次读取从这里继续，自然已经没有剩余内容。

把位置移回开头，再读：

```python
with open("both.txt", "w+", encoding="utf-8") as f:
    f.write("hello")
    f.seek(0)
    print(repr(f.read()))
```

通常输出：

```text
'hello'
```

这里的 `seek(0)` 既表达重新从头读取的意图，也处理相应流定位与缓冲状态。

#### `a+` 能读旧内容吗？

可以，但要注意初始位置在结尾。

```python
with open("events.txt", "a+", encoding="utf-8") as f:
    f.seek(0)
    old_text = f.read()
    f.write("新的事件\n")
```

先 `seek(0)`，读取旧内容；后面的写入仍然用于追加。

不要把 `a+` 理解成“自动先把旧文件读完，再让我修改任意位置”。

### 4.6 `r+` 会不会保留旧内容？

会保留，但它允许从当前位置覆盖。

我们用 ASCII 做实验，先避开多字节编码对定位的影响。

准备文件：

```python
with open("replace_demo.txt", "w", encoding="utf-8") as f:
    f.write("abcdef")
```

再执行：

```python
with open("replace_demo.txt", "r+", encoding="utf-8") as f:
    f.write("XY")
```

读取结果：

```python
with open("replace_demo.txt", encoding="utf-8") as f:
    print(f.read())
```

通常得到：

```text
XYcdef
```

前两个字节被覆盖，后面的 `cdef` 仍然存在。

#### 为什么不是只剩 `XY`？

因为 `r+` 没有要求截断整个文件。

这次写入只替换了当前位置对应的数据，没有自动把后面的旧内容删除。

如果你想让全文变成 `XY`，可以重新生成文件；需要低层控制时，也可以在明确位置使用 `truncate()`。

但普通文本修改通常优先选“读内容 → 修改字符串 → 生成新文件”，这样更容易理解变长内容和失败处理。

### 4.7 `b` 和 `t` 又是什么？

前面的 `r`、`w`、`a`、`x` 决定基本动作。

`b`、`t` 决定数据以什么形式在 Python 中交互。

`b` 是 binary，也就是二进制模式。

`t` 是 text，也就是文本模式，而且文本模式是默认选择。

例如：

```python
open("photo.jpg", "rb")
```

表示以二进制方式读取。

```python
open("notes.txt", "rt", encoding="utf-8")
```

表示以 UTF-8 文本方式读取，与这里的 `"r"` 写法等价。

`rb` 读取返回 `bytes`；文本 `r` 读取返回 `str`。

需要同时读写二进制内容时，可以使用：

```python
open("records.bin", "r+b")
```

这些标记按规则组合，不是把英文单词随意连起来。

例如，`"rw"` 不能表达读写；`"bt"` 也不能同时表达二进制和文本。

### 4.8 回头用一张表整理

先理解各个情境，再看表格，就比较容易看出每一项为什么不同。

| 模式 | 读 | 写 | 不存在时 | 已存在时 | 主要用途 |
| --- | --- | --- | --- | --- | --- |
| `r` | 是 | 否 | 报错 | 保留 | 读取输入 |
| `w` | 否 | 是 | 创建 | 打开时截断 | 重新生成输出 |
| `a` | 否 | 是 | 创建 | 保留，追加写 | 增加记录 |
| `x` | 否 | 是 | 创建 | 报错 | 只允许新建 |
| `r+` | 是 | 是 | 报错 | 保留，允许覆盖 | 受控读写 |
| `w+` | 是 | 是 | 创建 | 打开时截断 | 重新生成后还要读取 |
| `a+` | 是 | 是 | 创建 | 保留，追加写 | 读取已有内容并追加 |
| `x+` | 是 | 是 | 创建 | 报错 | 新建后允许读写 |

文本和二进制标记可以继续加在相应合法模式中，例如 `rb`、`wb`、`ab` 和 `r+b`。

你不必一开始背会所有组合。

先熟练回答四个问题：

文件已经存在吗？

旧内容需要保留吗？

我需要读、写，还是两者都需要？

我处理的是文字，还是希望保留原始字节？

模式选择往往就能顺着需求确定下来。

## 第五章：文本模式、二进制模式和编码

### 5.1 文本模式返回 `str`，二进制模式返回 `bytes`

```python
with open("notes.txt", "r", encoding="utf-8") as f:
    text = f.read()
    print(type(text))

with open("notes.txt", "rb") as f:
    data = f.read()
    print(type(data))
```

输出的类型分别是 `str` 和 `bytes`。

文本模式负责解码和可能的换行转换。二进制模式保留读取到的原始字节，不进行字符解码和文本换行转换。

读取图片、压缩包或计算原文件哈希时，使用二进制模式。读取确定编码的 Markdown、普通配置文本时，使用文本模式更方便。

### 5.2 写入类型必须匹配

```python
with open("hello.txt", "w", encoding="utf-8") as f:
    f.write("你好")

with open("hello.bin", "wb") as f:
    f.write("你好".encode("utf-8"))
```

文本流的 `write()` 接受 `str`；二进制流的 `write()` 接受字节类对象。

`f.write(123)` 不会自动变成字符串。需要 `f.write(str(123))`，或者使用能够执行文本转换的 `print(123, file=f)`。

二进制模式不能设置文本 `encoding`、`errors` 或 `newline` 参数。下面的写法会报错：

```python
# 错误示例：二进制读取不执行字符解码。
open("hello.bin", "rb", encoding="utf-8")
```

### 5.3 `encoding="utf-8"` 指的是内容，不能替你识别格式

```python
open("中文文件名.txt", encoding="utf-8")
```

这里设置的是文件内容的解码规则，不是指定文件名本身如何传给操作系统。

如果文件实际是 GBK，使用 UTF-8 可能抛出 `UnicodeDecodeError`，也可能在某些数据上得到错误解释。编码应该由文件格式、生成工具或双方约定来确定。

处理明确来自旧式中文 Windows 工具的 GBK 文件，可以这样读：

```python
with open("legacy.txt", encoding="gbk") as f:
    text = f.read()
```

编码转换是先按原编码读，再按新编码写：

```python
with open("legacy.txt", encoding="gbk") as source:
    text = source.read()

with open("converted.txt", "w", encoding="utf-8") as target:
    target.write(text)
```

更改扩展名不能完成编码转换。

### 5.4 为什么仍然建议明确写编码？

Python 3.11～3.14 的默认文本编码与区域设置、UTF-8 模式等有关。Python 3.15 按 PEP 686 默认启用 UTF-8 模式，但该模式仍可通过启动配置关闭。默认 UTF-8 也不会让既有 GBK 文件自动转码。

因此，对已约定 UTF-8 的文件直接写 `encoding="utf-8"`，能让读者看懂格式约定，也能减少跨环境差异。[版本依据：PEP 686](https://peps.python.org/pep-0686/)

### 5.5 `utf-8-sig` 和 BOM

BOM 可以理解为某些编码格式放在开头的标记。UTF-8 不需要依靠 BOM 判断字节序，但部分工具会在 UTF-8 文件开头写入它。

```python
with open("from_excel.csv", encoding="utf-8-sig", newline="") as f:
    first_line = f.readline()
```

`utf-8-sig` 读取时会处理开头的 UTF-8 BOM；文件没有 BOM 时也可以读取普通 UTF-8 内容。

用 `encoding="utf-8-sig"` 写文件会写出相应标记。只有对接工具需要时才选择它，不必把所有 UTF-8 文件都加上 BOM。

### 5.6 `errors`：出错时是报告，还是丢弃信息？

| 策略 | 含义 | 适用提醒 |
| --- | --- | --- |
| `strict` | 遇到无法编解码的内容就报错 | 默认，适合发现数据问题 |
| `replace` | 用替代内容表示错误 | 可读性优先的展示场景 |
| `ignore` | 跳过无法处理的内容 | 会静默丢数据 |
| `surrogateescape` | 通过特殊代理码点暂存部分非法字节 | 主要用于系统工具的可逆处理 |

读取损坏文本用于人工查看时，可以明确选择：

```python
with open("damaged.txt", encoding="utf-8", errors="replace") as f:
    text = f.read()
```

但订单、配置、程序源码等数据不宜为了“不报错”就改成 `ignore`。错误消失，内容却可能已经变了。

## 第六章：读取内容时，`read()`、`readline()` 和 `readlines()` 怎么选？

现在我们已经能找到文件，也知道应该用只读模式打开。

接下来才是真正取得内容。

这里最容易产生的误解是：以为每调用一次读取，都会重新把文件从头拿出来。

我们先不急着比较四种写法，做一个很小的实验。

### 6.1 第一次 `read()` 有内容，第二次为什么空了？

先准备练习文件：

```python
with open("reading_demo.txt", "w", encoding="utf-8") as f:
    f.write("apple\nbanana\n")
```

文件中有两行：

```text
apple
banana
```

然后执行：

```python
with open("reading_demo.txt", "r", encoding="utf-8") as f:
    first = f.read()
    second = f.read()

print(repr(first))
print(repr(second))
```

通常输出：

```text
'apple\nbanana\n'
''
```

第一次取得两行，第二次是空字符串。

文件没有被 `read()` 删除，也没有被清空。

变化的是**这个文件对象的当前读取位置**。

刚打开时，位置在开头。

第一次 `read()` 不指定长度，就从当前位置读到末尾。

读完以后，位置也到了末尾。

第二次再从这里读，已经没有剩余内容，就返回空字符串。

可以想象成书签：读书时书签往后移动，下一次继续从书签位置读，不会自动回第一页。

#### 如果我确实想再读一遍呢？

对于支持定位的普通文件，可以先回到开头：

```python
with open("reading_demo.txt", encoding="utf-8") as f:
    first = f.read()
    f.seek(0)
    second = f.read()

print(first == second)
```

在文件没有被其他程序修改等普通条件下，通常输出：

```text
True
```

`seek(0)` 表示回到流开头。

也可以关闭后重新打开，新的普通只读打开通常重新从开头读取。

#### 为什么用 `repr()` 打印？

普通 `print()` 会把字符串里的换行真正显示成换行。

`repr()` 会给出便于观察的表示，让你看清 `\n` 和空字符串。

例如：

```python
print("apple\n")
print(repr("apple\n"))
```

前者看起来像一行 apple 加换行；后者能让你明确看到这个字符串末尾包含一个换行字符。

学习文件读取时，`repr()` 很有用，因为许多差别恰好藏在看不见的空白里。

### 6.2 如果我只想读前面几个字符呢？

给 `read()` 提供大小：

```python
with open("reading_demo.txt", encoding="utf-8") as f:
    first = f.read(3)
    second = f.read(3)

print(repr(first))
print(repr(second))
```

通常得到：

```text
'app'
'le\n'
```

第一次读取前三个字符，第二次从当前位置继续取得三个字符。

第二次的第三个字符恰好是换行。

这里的 `3` 表示最多读取三个字符，不是三行。

#### 中文也是按三个字节算吗？

文本模式按字符处理，不能直接当作 UTF-8 字节数量。

```python
with open("chinese_demo.txt", "w", encoding="utf-8") as f:
    f.write("你好Python")

with open("chinese_demo.txt", "r", encoding="utf-8") as f:
    print(repr(f.read(2)))

with open("chinese_demo.txt", "rb") as f:
    print(f.read(2))
```

文本读取通常得到：

```text
'你好'
```

二进制读取只取得 UTF-8 内容的前两个字节，得到的是一个字节片段，不是两个完整汉字。

这也是为什么处理文字时，经常让文本流负责解码，而不自己拿任意字节片段逐块 `decode()`。

#### 文件只剩两个字符，我要求十个，会报错吗？

普通文件通常不会因为剩余内容不足而报错。

它返回实际能够读到的内容，数量可能比请求值少。

已经到末尾时，文本读取返回 `""`，二进制读取返回 `b""`。

不要把“本次没有读满”与“发生了异常”直接画等号。

### 6.3 如果我想一行一行读取呢？

使用 `readline()`。

名称可以拆成 read 和 line，也就是读取一行。

先准备一个有空白行、而且最后一行没有换行的文件：

```python
with open("lines.txt", "w", encoding="utf-8", newline="\n") as f:
    f.write("apple\n\nbanana")
```

再执行四次读取：

```python
with open("lines.txt", encoding="utf-8") as f:
    print(repr(f.readline()))
    print(repr(f.readline()))
    print(repr(f.readline()))
    print(repr(f.readline()))
```

通常输出：

```text
'apple\n'
'\n'
'banana'
''
```

我们分别解释。

第一次得到：

```text
'apple\n'
```

说明第一行文字和结束它的换行都被读到了。

第二次得到：

```text
'\n'
```

这是一条空白行。

它仍然有行结束符，所以不是空字符串。

第三次得到：

```text
'banana'
```

文件最后没有写换行，读取器不会凭空补一个。

第四次才得到：

```text
''
```

这表示已经没有剩余内容，也就是到达 EOF。

EOF 来自 End Of File，表示文件结束。

#### 空白行会不会让循环提前停止？

如果你检查的是：

```python
if line == "":
    break
```

空白行 `"\n"` 不等于空字符串，因此不会误结束。

但如果先 `.strip()`：

```python
line = f.readline().strip()
```

空白行和 EOF 都可能变成 `""`。

然后你再用空字符串决定是否结束，就把两种不同情况混在一起了。

所以，需要保留区别时，应先判断原始读取结果，再根据用途处理行内容。

#### 能不能给 `readline()` 也传数字？

可以，例如：

```python
f.readline(10)
```

它会限制本次最多读取的长度。

因此，某一行超过十个字符时，这次可能只拿到其中一部分。

`readline()` 有“按行读取”的默认行为，不代表加了长度限制后仍保证每次返回完整逻辑行。

### 6.4 如果我想直接得到每一行组成的列表呢？

使用 `readlines()`。

```python
with open("lines.txt", encoding="utf-8") as f:
    lines = f.readlines()

print(lines)
```

得到：

```text
['apple\n', '\n', 'banana']
```

外面是列表，每个元素对应读到的一行。

行尾换行仍按照实际内容和文本流的换行规则保留。

`readlines()` 从当前位置读取剩余行，不会因为名字多一个 `s` 就自动回到文件开头。

#### `readlines(5)` 是读五行吗？

不是。

这个位置参数是累计读取量的提示，不是精确行数。

如果你想读前五行，更清楚的写法是：

```python
from itertools import islice

with open("notes.txt", encoding="utf-8") as f:
    first_five_lines = list(islice(f, 5))
```

`islice(f, 5)` 最多从文件的逐行迭代中取得五个元素。

文件不足五行就自然结束。

#### 既然列表这么方便，为什么还要别的写法？

因为 `readlines()` 会把所有结果保存到列表里。

小笔记没什么问题。

几百万行日志就可能让内存开销明显增大。

如果你的目标只是每行检查一次，并不需要同时保存所有行，那么直接迭代更合适。

### 6.5 `for line in f` 为什么这么常见？

```python
with open("lines.txt", encoding="utf-8") as f:
    for line in f:
        print(repr(line))
```

文件对象支持迭代，每轮提供一行。

你不必先创建完整行列表，也不必自己反复写 `readline()` 和 EOF 判断。

对于按行组织的普通文本，这是一种很自然的写法。

#### 如果还想知道当前是第几行呢？

搭配 `enumerate()`：

```python
with open("lines.txt", encoding="utf-8") as f:
    for number, line in enumerate(f, start=1):
        print(number, repr(line))
```

通常得到：

```text
1 'apple\n'
2 '\n'
3 'banana'
```

`start=1` 让行号从一开始，符合人读文件时的习惯。

`enumerate()` 是 Python 的通用迭代工具，不是文件对象独有的方法。

#### 为什么我 `print(line)` 后，行与行之间多了空行？

文件里的 `line` 通常已经带着 `\n`。

`print()` 默认又加一个换行。

于是一次循环产生两个换行，看起来就多出空白。

如果希望按原样显示，可以写：

```python
with open("lines.txt", encoding="utf-8") as f:
    for line in f:
        print(line, end="")
```

`end=""` 表示这一次打印不要再额外追加换行。

#### 我直接 `.strip()` 一下不也能解决？

有时能，但要知道它还会删什么。

例如原来的文本是：

```python
line = "    这行有缩进，也有尾部空格  \n"
```

执行：

```python
line.strip()
```

前面的缩进和后面的普通空格也会被删除。

如果这是一段 Python 源代码、Markdown 代码缩进或需要保留空白的文本，内容就被改变了。

只想删除默认文本读取后的一个末尾换行，可以使用：

```python
line.removesuffix("\n")
```

它只去掉指定的一个末尾后缀，不动行首缩进和末尾普通空格。

如果你使用了保留原换行的读取配置，就还要根据实际 `\r\n`、`\r` 等形式处理，不能把默认统一换行的假设套到所有情况。

### 6.6 四种方式，应该怎么选择？

如果文件很小，而且处理需要完整文本，例如替换模板或解析一份小配置，先考虑 `read()`。

如果程序一次需要主动取一行，或者需要在每次读取后记录位置，考虑 `readline()`。

如果确实需要所有行组成的列表，而且文件大小适合内存，考虑 `readlines()`。

如果只是逐行检查、统计或转换，优先熟悉 `for line in f`。

最后再看一张表：

| 写法 | 返回或每轮得到什么 | 是否推进当前位置 | 是否一次保存全部剩余内容 |
| --- | --- | --- | --- |
| `f.read()` | 一个字符串 | 是 | 通常是 |
| `f.read(n)` | 最多 n 个字符的字符串 | 是 | 只取相应部分 |
| `f.readline()` | 一行字符串 | 是 | 否 |
| `f.readlines()` | 行字符串列表 | 是 | 是 |
| `for line in f` | 每轮一行 | 是 | 不要求保存全部行 |

注意，逐行读取仍可能遇到一条极长的行。

它减少的是保存所有行的需求，不代表任何输入下内存都固定不变。

## 第七章：`write()`、`writelines()` 和 `print(file=...)`

### 7.1 `write()` 写入一个字符串，不自动换行

```python
with open("write_demo.txt", "w", encoding="utf-8") as f:
    count = f.write("你好\n")
    f.write("Python")

print(count)
```

`count` 是 `3`：两个汉字和一个换行码点。它是文本流接受写入的字符数，不是 UTF-8 文件字节数，也不是“已经安全落盘的字节数”。

一般阻塞模式下的高层文件流会处理常规缓冲和写入；原始、非阻塞等低层流还可能出现部分写入等情况，不能把所有名为 `write` 的接口都视为完全相同。

### 7.2 `writelines()` 不会帮你加分隔符

```python
with open("joined.txt", "w", encoding="utf-8") as f:
    f.writelines(["apple", "banana", "orange"])
```

文件内容是：

```text
applebananaorange
```

它接受字符串的可迭代对象，依次写入每个字符串。名称里的 `lines` 不代表自动附加换行。

正确写多行可以使用生成器表达式：

```python
names = ["apple", "banana", "orange"]

with open("names.txt", "w", encoding="utf-8") as f:
    f.writelines(name + "\n" for name in names)
```

或者使用更直观的循环。`writelines()` 通常返回 `None`，不返回累计字符数。

### 7.3 `print(..., file=f)` 适合输出人类阅读的内容

```python
with open("summary.txt", "w", encoding="utf-8") as f:
    print("姓名", "分数", sep=",", file=f)
    print("小林", 95, sep=",", file=f)
```

`file=f` 改变输出目标。

`sep=","` 决定多个参数之间的分隔文本。

`end` 默认是 `"\n"`，所以每次调用会结束一行。

`print()` 会把数字等对象转成文本，写普通报告很方便。但上例只是简单输出，不是通用 CSV 序列化：字段里出现逗号、引号或换行时，应交给 `csv.writer()` 处理。

## 第八章：换行转换，为什么 Windows 和 Linux 的文件可能不同？

### 8.1 常见换行形式

| 形式 | 字符含义 | 常见来源 |
| --- | --- | --- |
| `\n` | LF | Linux、现代 macOS、许多数据格式 |
| `\r\n` | CRLF | Windows 工具常见文本 |
| `\r` | CR | 一些旧格式 |

Python 字符串里的 `"\n"` 是一个换行字符。写进文件时到底使用哪些字节，还取决于文本流的 `newline` 和编码规则。

### 8.2 读取时的 `newline`

| 参数 | 读取时的行为 |
| --- | --- |
| `None` | 识别 `\n`、`\r\n`、`\r`，返回时统一成 `\n` |
| `""` | 仍识别这几种行结束形式，但保留原始换行字符 |
| `"\n"` | 仅把指定的 `\n` 作为行结束依据，不执行通用转换 |

做一个实验：

```python
with open("mixed_newlines.bin", "wb") as f:
    f.write(b"A\r\nB\rC\n")

with open("mixed_newlines.bin", encoding="ascii", newline=None) as f:
    print(repr(f.read()))

with open("mixed_newlines.bin", encoding="ascii", newline="") as f:
    print(repr(f.read()))
```

输出：

```text
'A\nB\nC\n'
'A\r\nB\rC\n'
```

这说明文本读取可能改变返回文本中的换行表示。要逐字节复制文件，应使用二进制流。

### 8.3 写入时的 `newline`

写入时，`newline=None` 会把字符串中的 `\n` 转成系统默认行分隔形式；`newline=""` 或 `newline="\n"` 不进行这种替换。

需要稳定输出 LF 文本可以写：

```python
with open("portable.txt", "w", encoding="utf-8", newline="\n") as f:
    f.write("A\nB\n")
```

不要先手动把每个换行改成 `\r\n`，再让 Windows 默认文本转换重复处理。

CSV 常用：

```python
import csv

with open("table.csv", "w", encoding="utf-8", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["姓名", "分数"])
    writer.writerow(["小林", 95])
```

`newline=""` 让 CSV 模块管理自己的换行与引号规则。这个要求读写 CSV 时都应记住。[官方说明：CSV 文件的打开方式](https://docs.python.org/3.14/library/csv.html#csv.reader)

## 第九章：文件位置、`tell()`、`seek()` 与 `truncate()`

### 9.1 文件对象有当前位置

```python
with open("position.bin", "wb") as f:
    f.write(b"abcdef")

with open("position.bin", "rb") as f:
    print(f.tell())
    print(f.read(2))
    print(f.tell())
    print(f.read(2))
```

输出：

```text
0
b'ab'
2
b'cd'
```

`tell()` 返回当前流位置。对于这个二进制普通文件，它可按从开头算起的字节偏移理解。

### 9.2 二进制文件的 `seek(offset, whence)`

| `whence` | 常量 | 基准 |
| --- | --- | --- |
| `0` | `os.SEEK_SET` | 文件开头，默认 |
| `1` | `os.SEEK_CUR` | 当前位置 |
| `2` | `os.SEEK_END` | 文件末尾 |

```python
import os

with open("position.bin", "rb") as f:
    f.seek(2)
    print(f.read(2))

    f.seek(-2, os.SEEK_END)
    print(f.read())
```

输出是 `b"cd"` 和 `b"ef"`。这里只移动位置，不会插入或删除内容。

管道、套接字等流可能不支持随机定位，可以用 `seekable()` 查询能力，但真正操作仍可能失败。

### 9.3 文本 `tell()` 不是字符下标

文本流需要考虑多字节编码、解码状态和换行转换。因此其 `tell()` 返回的整数应当看成可用于恢复位置的标记，不能随便加一减一。

可靠的用法是保存后原样传回：

```python
with open("notes.txt", encoding="utf-8") as f:
    first = f.readline()
    marker = f.tell()
    second = f.readline()

    f.seek(marker)
    same_second = f.readline()

    print(second == same_second)
```

文本流常用的合法定位包括 `seek(0)`、`seek(tell返回的标记)`、`seek(0, 2)` 和 `seek(0, 1)`。不要直接套用二进制方式写 `seek(-10, 2)` 来读取最后十个字符。

另外，某些 `TextIOWrapper` 在使用 `for line in f` 的迭代期间会禁用位置报告，调用 `tell()` 可能报 `OSError`。需要记录行边界标记时，使用明确的 `readline()` 循环。[官方说明：文本流定位](https://docs.python.org/3.14/library/io.html#io.TextIOWrapper.seek)

### 9.4 `truncate()` 修改长度，不负责插入字符

```python
with open("truncate.bin", "wb") as f:
    f.write(b"abcdef")

with open("truncate.bin", "r+b") as f:
    f.write(b"XY")
    f.truncate()

with open("truncate.bin", "rb") as f:
    print(f.read())
```

输出：

```text
b'XY'
```

`truncate()` 不给大小时，以当前位置作为目标长度。指定大小时，对普通文件按字节长度理解。文本编码下不能把“希望保留三个汉字”直接写成 `truncate(3)`。

文件写操作本质上写入或覆盖字节，不会像文本编辑器一样自动把后面的内容全部后移。修改变长文本通常需要重新生成文件。

## 第十章：缓冲、`flush()` 和“已经保存”到底意味着什么？

### 10.1 为什么需要缓冲？

每写几个字符就请求操作系统处理一次，会增加开销。因此 I/O 系统常把小写入先累积，再按更合适的大小交给下一层。

```text
你的代码
    ▼
Python 文本编码和缓冲层
    ▼
操作系统缓存
    ▼
文件系统、存储设备
```

`f.write(...)` 返回，不代表每一层都已经完成持久化。

### 10.2 `flush()` 刷新 Python 流的缓冲

```python
with open("progress.txt", "w", encoding="utf-8") as f:
    f.write("任务已经开始\n")
    f.flush()
```

适合希望其他读取者更快看到输出等场景。它不会把当前位置重置到开头，也不会代替关闭文件。

如果要求把 Python 缓冲交出后进一步请求系统同步，可了解：

```python
import os

with open("important.txt", "w", encoding="utf-8") as f:
    f.write("重要结果\n")
    f.flush()
    os.fsync(f.fileno())
```

`fileno()` 返回底层文件描述符；`os.fsync()` 请求系统同步相关数据。持久性还涉及硬件、文件系统、目录元数据和错误处理，不要把这一段当成适用于所有故障的数据库事务实现。

### 10.3 `buffering` 的常见选项

| 值 | 意义 |
| --- | --- |
| `-1` | 使用默认策略，普通情况优先保留 |
| `0` | 无缓冲，仅适用于二进制模式 |
| `1` | 行缓冲，用于文本模式 |
| 大于 `1` 的整数 | 请求二进制缓冲层使用相应策略或大小 |

文本 I/O 还有自己的包装层，因此不能把 `buffering=4096` 简化成“每 4096 个汉字才写盘”。

行缓冲意味着遇到特定换行触发刷新等行为，不是“每条业务记录绝对不会丢失”。日常写文件，默认缓冲加 `with` 通常足够。

## 第十一章：常用属性与能力查询

```python
with open("notes.txt", encoding="utf-8") as f:
    print(f.name)
    print(f.mode)
    print(f.encoding)
    print(f.closed)
    print(f.readable())
    print(f.writable())
    print(f.seekable())
```

| 表达式 | 作用 |
| --- | --- |
| `f.name` | 打开时关联的名称或描述符信息 |
| `f.mode` | 打开模式信息 |
| `f.encoding` | 文本流的字符编码，二进制流没有同样的属性约定 |
| `f.closed` | 是否已经关闭 |
| `f.readable()` | 是否支持读取 |
| `f.writable()` | 是否支持写入 |
| `f.seekable()` | 是否支持随机定位 |
| `f.fileno()` | 底层描述符，内存流等对象可能不支持 |

属性不加调用括号，方法需要括号。`f.closed()` 是把布尔值当函数调用，会出错。

## 第十二章：异常处理——区分错误的层次

### 12.1 常见异常

| 异常 | 常见原因 |
| --- | --- |
| `FileNotFoundError` | 输入文件或父目录不存在 |
| `PermissionError` | 权限、访问控制或占用限制 |
| `IsADirectoryError` | 在部分平台上把目录当普通文件打开 |
| `NotADirectoryError` | 路径中某一段应为目录却是文件 |
| `UnicodeDecodeError` | 读取时内容不能按所选编码解码 |
| `UnicodeEncodeError` | 写入时字符不能按所选编码表示 |
| `io.UnsupportedOperation` | 在只读流写入，或在不支持定位的流上定位 |
| `ValueError` | 非法模式、已经关闭的文件等 |
| `OSError` | 包括多种系统 I/O 错误的基类 |

### 12.2 把打开、读取和关闭都放在需要保护的范围内

```python
try:
    with open("config.txt", encoding="utf-8") as f:
        text = f.read()
except FileNotFoundError:
    print("没有找到配置文件，请检查路径")
except UnicodeDecodeError:
    print("配置文件不能按 UTF-8 解码，请确认原编码")
except OSError as error:
    print("文件访问失败：", error)
```

`UnicodeDecodeError` 不是 `OSError` 子类，单独处理能更准确地反馈问题。

不要写 `except: pass` 把所有问题吞掉。磁盘写满、编码错误、路径拼错代表不同问题，静默继续可能让调用者误认为数据已经保存。

`exists()` 或 `access()` 检查只能描述当时观察到的状态，不能保证下一次打开成功。对于实际操作，直接尝试并处理预期异常通常更可靠。

## 第十三章：把“读一个文件”写成可以复用的函数

前面的小例子把文件名直接写在代码里，是为了便于观察。实际项目里，你往往需要把读取逻辑做成函数，让调用者提供路径，并决定如何处理结果。

这里要分清三层责任：调用者决定读取哪个位置；读取函数负责按约定取得内容；业务函数负责解释内容。把这三件事分开，路径错误、编码错误和数据格式错误就更容易定位。

### 13.1 最简单的读取函数

```python
from pathlib import Path


def read_utf8(path: str | Path) -> str:
    with open(path, "r", encoding="utf-8") as source:
        return source.read()


text = read_utf8("notes.txt")
print(text)
```

逐句看函数定义：

```text
def read_utf8(path: str | Path) -> str:
```

`def` 开始定义函数。`read_utf8` 是函数名，明确说明这个函数处理的是 UTF-8 文本，而不是任意编码的任意文件。

`path` 是参数名称。冒号后面的 `str | Path` 是类型注解，表示调用者通常传入字符串路径或 `Path` 对象。这个注解帮助人和静态检查工具理解约定，不会自动在运行时验证参数类型。

`-> str` 表示预期返回一个字符串。它不会自动把返回值转成字符串。

```text
with open(path, "r", encoding="utf-8") as source:
```

每次调用函数都打开自己的流。不要把打开的文件对象放成函数默认参数，例如 `def read(f=open(...))`，因为默认参数在定义函数时求值，可能造成文件长期打开、位置被多次调用共享等问题。

```python
return source.read()
```

先执行读取，得到字符串，然后退出上下文，最后把结果交给调用者。读入的字符串已经是独立的内存对象，关闭磁盘文件不会使它失效。

### 13.2 为什么这里没有捕获异常？

这不表示作者忘记处理错误。

假如文件不存在，读取函数可能不知道该怎么办：是提示用户选另一个文件、使用默认配置、重新下载，还是终止任务？这些决定往往只有调用者知道。

因此，底层函数保留异常，让上层在有足够上下文的位置处理，是一种常见而合理的设计。

```python
try:
    notes = read_utf8("notes.txt")
except FileNotFoundError:
    notes = ""
```

这里“缺少笔记时使用空内容”是调用者主动制定的规则。不要把它未经说明地写进所有读取函数，否则路径拼错也可能被伪装成一个合法空文件。

### 13.3 给读取设置明确的体积上限

从外部接收的文件不一定和你预想的一样小。如果程序只允许处理一份较小配置，可以在读取过程中实施限制。

文本字符数和文件字节数是不同指标。下面的函数限制解码后的字符数量：

```python
from pathlib import Path


def read_small_text(path: str | Path, max_chars: int = 100_000) -> str:
    if max_chars < 0:
        raise ValueError("max_chars 不能为负数")

    with open(path, encoding="utf-8") as source:
        text = source.read(max_chars + 1)

    if len(text) > max_chars:
        raise ValueError("文本超过允许的字符数量")

    return text
```

为什么读取 `max_chars + 1`？

如果只读取 `max_chars`，拿到的内容恰好达到上限时，你不知道原文件是刚好这么长，还是后面还有未读内容。多读一个字符，就能区分这两种情况。

为什么没有先调用 `stat().st_size` 然后直接 `read()`？

一方面，`st_size` 是字节数，不能与文本码点数量直接对应；另一方面，查询后文件可能继续增长。对于需要严格限制的输入，要让读取步骤本身遵守限制。

这个函数限制的是最终读取的文本长度，仍然不是完整的资源沙箱。特别复杂的编码、后续解析过程和特殊文件都可能有额外开销。普通应用应同时约定输入类型与格式。

### 13.4 返回列表还是返回迭代器？

小型文件可以一次读取所有行：

```python
def read_all_lines(path):
    with open(path, encoding="utf-8") as source:
        return list(source)
```

`list(source)` 会消费文件迭代器，把剩余各行保存下来。函数返回时文件已经关闭，列表仍能使用，但内存里已经存了全部行。

希望按需处理时，可以写生成器：

```python
def iter_nonempty_lines(path):
    with open(path, encoding="utf-8") as source:
        for number, line in enumerate(source, start=1):
            if line.strip():
                yield number, line.removesuffix("\n")
```

`yield` 暂时交出一条结果，下一次迭代再从暂停位置继续。文件的打开和关闭也随着生成器运行而发生。

使用时：

```python
for line_number, content in iter_nonempty_lines("notes.txt"):
    print(line_number, content)
```

如果生成器被完整消费，通常能正常退出 `with`。如果你提前停止而又长期保存生成器引用，文件可能仍然保持打开。需要明确提前结束时，可以显式关闭生成器：

```python
from contextlib import closing

with closing(iter_nonempty_lines("notes.txt")) as lines:
    for number, text in lines:
        print(number, text)
        if number >= 10:
            break
```

这里展示的是资源生命周期问题。很多程序直接把“打开”和“遍历”写在调用者的同一个 `with` 里，会更直观。不要仅仅为了把代码塞进函数，就让关闭时机变得难懂。

## 第十四章：三个完整组合案例

### 14.1 逐行统计非空文本行，并保存报告

准备 `input.txt`，每行一条笔记，允许空行。下面程序仅用 `strip()` 判断是否全为空白，不修改保存的原文本：

```python
from pathlib import Path

source_path = Path("input.txt")
target_path = Path("output") / "summary.txt"
target_path.parent.mkdir(parents=True, exist_ok=True)

total = 0
nonempty = 0

with source_path.open(encoding="utf-8") as source:
    for line in source:
        total += 1
        if line.strip():
            nonempty += 1

with target_path.open("w", encoding="utf-8", newline="\n") as target:
    print(f"总行数：{total}", file=target)
    print(f"非空行数：{nonempty}", file=target)
```

`target_path.parent` 获取输出文件的父目录；`mkdir(...)` 创建目录。`open(..., "w")` 能创建文件，但不会创建缺失的父目录，所以这两步分工明确。

`for line in source` 逐行消费输入。最后一行即使没有换行，也会被正常统计；空文件的两项计数都是零。

### 14.2 按块复制二进制内容

```python
from pathlib import Path

source_path = Path("input.bin")
target_path = Path("output.bin")

if source_path.resolve() == target_path.resolve():
    raise ValueError("输入输出不能是同一路径")
if target_path.exists() and source_path.samefile(target_path):
    raise ValueError("输入输出指向同一个文件")

with source_path.open("rb") as source:
    with target_path.open("xb") as target:
        while True:
            chunk = source.read(1024 * 1024)
            if not chunk:
                break
            target.write(chunk)
```

`1024 * 1024` 是一 MiB。循环每次读取最多这么多字节，直到返回 `b""`。

`xb` 要求新建目标，避免覆盖已经存在的名字。这个教学程序不构成并发文件管理系统；执行失败可能留下部分目标文件。需要普通复制时，`shutil.copyfile()` 或 `copy2()` 更方便。

不要分别对每个 UTF-8 字节块直接 `.decode()`：一个汉字的多字节序列可能刚好跨块。文本流会管理增量解码；自定义字节协议则需要增量解码器或完整的边界处理。

### 14.3 先写临时文件，再替换正式文本

直接 `w` 写正式文件，中途失败可能把原文件变成不完整结果。对于单个文本结果，可以先写同目录临时文件，成功关闭后再替换：

```python
import os
import tempfile
from pathlib import Path


def save_text(target, text):
    target = Path(target).absolute()
    target.parent.mkdir(parents=True, exist_ok=True)
    temp_path = None

    try:
        with tempfile.NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            newline="\n",
            dir=target.parent,
            prefix=target.name + ".",
            suffix=".tmp",
            delete=False,
        ) as temp:
            temp_path = Path(temp.name)
            temp.write(text)
            temp.flush()
            os.fsync(temp.fileno())

        temp_path.replace(target)
    finally:
        if temp_path is not None:
            temp_path.unlink(missing_ok=True)


save_text("output/config.txt", "mode=development\n")
```

逐步理解这套安排：

第一，临时文件放在目标同目录，有利于让最终替换位于同一文件系统。

第二，`NamedTemporaryFile` 生成并创建独立临时名字，减少固定 `.tmp` 名称造成的冲突。

第三，`delete=False` 保留关闭后的临时文件，使它可以在退出 `with` 后参与替换；先关闭也有利于兼容 Windows 的文件共享限制。

第四，`replace()` 执行替换，而不是先删除正式文件再改名。支持相应语义的文件系统上，成功的同文件系统替换可以让其他读者看到旧版本或新版本，避免只看到写到一半的目标。

第五，`finally` 清理未被替换走的临时文件。

这个函数仍有明确边界：它不保留原文件全部权限和元数据，不防止多个写入者相互覆盖，不同步父目录元数据，也不提供跨多个文件的事务。清理失败仍可能抛异常。对真实数据库应使用数据库自己的事务与备份接口。[临时文件接口依据](https://docs.python.org/3.14/library/tempfile.html#tempfile.NamedTemporaryFile)、[路径替换依据](https://docs.python.org/3.14/library/pathlib.html#pathlib.Path.replace)

## 第十五章：完整实践——读取成绩文件，验证数据，再输出报告

这一章把文件读取与真正的业务处理连接起来。目标不是再背一个读取函数，而是练习如何判断：文件访问成功之后，内容是否也符合程序要求。

### 15.1 输入格式先说清楚

创建 `scores.txt`，内容如下：

```text
# 每行格式：姓名 分数
小林 95
小周 88

小陈 100
```

我们约定：空白行跳过；去掉两端空白后以 `#` 开头的行作为注释；其他行必须恰好分成姓名和整数分数两项；姓名暂不允许包含空格；分数限定在零到一百之间。

这些规则属于这个练习的业务格式，不属于 `open()` 的规则。换成 CSV、JSON 或其他数据格式，应该由相应解析工具来解释。

### 15.2 完整程序

```python
from pathlib import Path


def load_scores(path):
    students = []

    with open(path, encoding="utf-8") as source:
        for line_number, line in enumerate(source, start=1):
            cleaned = line.strip()

            if not cleaned or cleaned.startswith("#"):
                continue

            parts = cleaned.split()

            if len(parts) != 2:
                raise ValueError(
                    f"第 {line_number} 行：应提供姓名和分数两项"
                )

            name, score_text = parts

            try:
                score = int(score_text)
            except ValueError as error:
                raise ValueError(
                    f"第 {line_number} 行：分数必须是整数"
                ) from error

            if not 0 <= score <= 100:
                raise ValueError(
                    f"第 {line_number} 行：分数必须在 0 到 100 之间"
                )

            students.append((name, score))

    return students


def write_report(path, students):
    path = Path(path)
    path.parent.mkdir(parents=True, exist_ok=True)

    count = len(students)
    total = sum(score for name, score in students)

    with path.open("w", encoding="utf-8", newline="\n") as target:
        print("成绩报告", file=target)
        print(f"人数：{count}", file=target)

        if count:
            print(f"平均分：{total / count:.2f}", file=target)
        else:
            print("平均分：暂无数据", file=target)

        for name, score in sorted(students, key=lambda item: (-item[1], item[0])):
            print(f"{name}\t{score}", file=target)


def main():
    students = load_scores("scores.txt")
    write_report("output/scores_report.txt", students)


if __name__ == "__main__":
    main()
```

### 15.3 为什么先完成输入验证，再打开输出文件？

`load_scores()` 先读完并验证，成功后才调用 `write_report()`。

如果第三行有非法分数，异常会在打开输出之前发生，已有报告不会因为输入错误就先被清空。这种执行顺序非常实用：会失败的准备工作尽量先做，真正修改结果文件的时间尽量后移。

它仍然不能防止输出写入期间发生的磁盘错误。对于重要报告，可以把最后的保存过程换成前面介绍的临时文件替换函数。

### 15.4 逐句理解几个容易忽略的细节

```python
cleaned = line.strip()
```

这个格式允许去掉姓名前后的无意义空白，所以这里使用 `strip()` 是有意的。它是否合适，取决于你已经明确制定了输入格式，而不是因为所有文件行都应该如此清理。

```text
if not cleaned or cleaned.startswith("#"):
```

前半部分判断空字符串。后半部分判断注释。`or` 表示满足其中一项就跳过当前行。

```python
parts = cleaned.split()
```

不传分隔符的 `split()` 会按连续空白分割。因此一个空格、多个空格或制表符都能分开姓名和分数。它不能支持带空格的姓名，这是我们提前写明的格式限制。

```python
raise ValueError(...) from error
```

新的错误提供“第几行出错”的业务信息，同时保留原始转换错误作为异常原因。排错时，你既能看到用户应该修正什么，也能追踪最初失败的位置。

```python
sorted(students, key=lambda item: (-item[1], item[0]))
```

`item[1]` 是分数，取负号让较高分数优先；分数相同时，再按姓名排序。这个排序规则是程序主动定义的，不是文件读取顺序天然保证的。

### 15.5 应当自己做的边界实验

依次修改练习文件，观察程序行为：

- 删除文件，确认报的是文件不存在，而不是分数错误。
- 将文件保存为 GBK，确认 UTF-8 解码可能失败。
- 写入 `小林 excellent`，确认错误包含对应行号。
- 写入 `小林 101`，确认整数转换成功后仍会进行范围检查。
- 只保留注释和空行，确认程序不会除以零。
- 去掉最后一行的换行，确认仍能读取这名学生。

这些实验分别验证路径层、编码层、格式层和业务层。把错误分层理解，比见到报错就换一个编码或加一个 `try` 更有帮助。

## 第十六章：完整实践——逐行筛选日志，控制输出和错误信息

日志可能很大，通常不适合把全部内容 `read()` 后再处理。我们用一个简化格式练习按行筛选。

### 16.1 假设的输入格式

```text
2026-11-02T09:00:00 INFO service started
2026-11-02T09:01:00 WARNING queue is growing
2026-11-02T09:02:00 ERROR task failed
```

这里约定每行前三部分是时间、级别和消息，消息本身可以包含空格。下面的程序筛选 `ERROR`，并保留原始行内容。

### 16.2 实现

```python
import sys
from pathlib import Path


def extract_errors(source_path, target_path):
    source_path = Path(source_path)
    target_path = Path(target_path)

    if source_path.resolve() == target_path.resolve():
        raise ValueError("输入和输出不能使用同一路径")

    target_path.parent.mkdir(parents=True, exist_ok=True)
    matched = 0
    malformed = 0

    with source_path.open(encoding="utf-8") as source:
        with target_path.open("x", encoding="utf-8", newline="\n") as target:
            for number, line in enumerate(source, start=1):
                parts = line.split(maxsplit=2)

                if len(parts) < 3:
                    malformed += 1
                    print(f"跳过第 {number} 行：格式不完整", file=sys.stderr)
                    continue

                timestamp, level, message = parts

                if level == "ERROR":
                    target.write(line)
                    matched += 1

    return matched, malformed


matched, malformed = extract_errors("service.log", "output/errors.log")
print(f"提取 {matched} 行，跳过 {malformed} 行格式错误数据")
```

### 16.3 为什么用 `maxsplit=2`？

`split(maxsplit=2)` 最多分割两次，得到最多三项。这样 `task failed because ...` 会保留在第三项中，而不会被拆成十几段。

这个练习没有解析时间，也没有识别多行 traceback。真实日志格式可能包含 JSON、多行异常或不同分隔符，应根据真实格式调整解析器，不要把这段程序当成通用日志分析引擎。

### 16.4 为什么输出用了 `x`？

日志筛选通常生成新的结果文件。采用排他创建，可以避免反复练习时无意覆盖上一份结果。如果目标已存在，应该由用户决定换名、删除旧结果，还是明确允许覆盖。

输入输出即使字符串不同，也可能通过符号链接或硬链接指向同一文件。这里的 `x` 会拒绝已经存在的目标名字，从而减少覆盖风险；要做更复杂的路径身份判断，可以结合 `samefile()`，但并发环境仍需要额外设计。

### 16.5 “逐行”并不等于完全不占内存

这个程序不会保存全部日志行，但每次仍然要存放当前行、分割结果和内部缓冲。

如果输入是一份没有换行的数 GiB 文本，一次行读取可能就非常大。需要处理不可信日志时，可以限制行长、文件总量和处理时限。对一般正常日志，逐行处理已经是很好的起点。

## 第十七章：完整实践——合并多个文本文件，并保证边界清晰

合并不是简单地把几个 `read()` 放在一起。你还需要决定顺序、文件间分隔、缺失文件如何处理，以及失败后留下什么结果。

### 17.1 先定义期望结果

假设有：

```text
chapters/
├── 01-intro.txt
├── 02-basic.txt
└── 03-practice.txt
```

希望按名称排序合并，并在每个文件之前写一行标题。输出写到单独目录，不在输入目录内参与下一次扫描。

### 17.2 按文件顺序流式合并

```python
from pathlib import Path


def merge_chapters(source_dir, output_path):
    source_dir = Path(source_dir)
    output_path = Path(output_path)

    sources = sorted(
        path for path in source_dir.glob("*.txt")
        if path.is_file() and not path.is_symlink()
    )

    if not sources:
        raise ValueError("输入目录中没有可合并的文本文件")

    output_resolved = output_path.resolve()
    if any(path.resolve() == output_resolved for path in sources):
        raise ValueError("输出路径不能属于输入文件集合")

    output_path.parent.mkdir(parents=True, exist_ok=True)

    with output_path.open("x", encoding="utf-8", newline="\n") as target:
        for path in sources:
            target.write(f"# 来源：{path.name}\n\n")
            last_char = ""

            with path.open(encoding="utf-8") as source:
                while True:
                    chunk = source.read(64 * 1024)
                    if not chunk:
                        break
                    target.write(chunk)
                    last_char = chunk[-1]

            if last_char != "\n":
                target.write("\n")
            target.write("\n")


merge_chapters("chapters", "output/book.txt")
```

### 17.3 每一个安排解决什么问题？

`sorted(...)` 定义稳定顺序。目录扫描返回顺序没有排序保证，不能把“今天看到的顺序”当成章节顺序。

文件名前补零，例如 `01`、`02`、`10`，是为了让普通字符串排序和预期章节序号保持一致。如果使用 `1`、`2`、`10`，默认排序通常会把 `10` 放在 `2` 前面。

`source_dir.glob("*.txt")` 只搜索当前目录这一层的匹配项。还要 `is_file()`，是因为匹配扩展名的条目不一定是普通文件。

`not path.is_symlink()` 是这个练习选择的扫描策略：不主动把符号链接指向的外部文件混入教材合并。它不是对恶意并发修改目录的完整防御。

`last_char` 记录最后一次读取的末尾字符。假如源文件最后没有换行，就补一个，避免下一份文件的标题紧接在上一段文字后面。

这里读的是文本块，`64 * 1024` 是最多读取的字符数量，不是精确的文件字节块大小。文本包装器管理 UTF-8 解码边界，所以不会因为一段汉字跨过底层字节缓冲而直接产生半个编码序列错误。

### 17.4 失败处理还可以如何加强？

当前程序如果合并第三个文件时遇到解码错误，会关闭流并抛异常，但已经创建的输出可能只包含前两个文件。

这并不违反 `with` 的行为：资源被清理了，已写入内容不会被自动撤回。

如果你需要“全部成功后才发布结果”，可以把外层输出改为同目录临时文件，所有输入处理成功后再替换正式文件。练习时最好主动制造第三个输入损坏的情形，观察两种保存策略的差别。

## 第十八章：二进制文件进一步实践——读取固定长度记录

文本文件依靠编码和分隔符解释内容；二进制格式通常依靠明确的字段长度、字节序和协议规则。

这里设计一个很小的练习格式：每条记录由一个四字节无符号整数编号和一个两字节无符号整数分数组成，统一使用大端序。一条记录共六个字节。

### 18.1 写入练习数据

```python
import struct

record_format = struct.Struct(">IH")

with open("records.bin", "wb") as target:
    target.write(record_format.pack(1001, 95))
    target.write(record_format.pack(1002, 88))
```

`struct` 是标准库中的二进制结构工具。这里的 `>` 指定大端序，`I` 对应四字节无符号整数，`H` 对应两字节无符号整数。

这套格式是我们定义的示例协议。换成别的设备、软件或文件格式，不能自行猜测它也使用相同字段和字节序。

### 18.2 读取并检查末尾是否完整

```python
import struct

record_format = struct.Struct(">IH")

with open("records.bin", "rb") as source:
    while True:
        data = source.read(record_format.size)

        if data == b"":
            break

        if len(data) != record_format.size:
            raise ValueError("文件末尾存在不完整记录")

        student_id, score = record_format.unpack(data)
        print(student_id, score)
```

`record_format.size` 是六。相比在多个位置手写数字六，从格式对象取得长度更不容易在修改字段后漏改。

先判断空结果，再判断短记录，是为了区分“正常到达记录边界的末尾”和“最后一条记录只保存了一部分”。

对于普通阻塞磁盘文件，按这种方式读取固定大小记录很常见。对于原始流、网络或其他可能短读的来源，还需要循环累积到期望长度，不能把一次短读都当成末尾损坏。

### 18.3 为什么不用 `readline()` 处理这种文件？

因为二进制记录中的某个字段字节可能刚好等于 `0x0A`，也就是 LF 的字节值。那不一定是分隔符，可能只是整数内容的一部分。

读文件的方法应该服从数据格式。只有格式明确规定按行组织，逐行读取才具有相应语义。文件对象不会根据扩展名自动知道“这一段是编号，那一段是图片像素”。

### 18.4 为什么修改一个字节不等于修改一个字符？

在二进制格式中，可以按协议的字节位置改字段。但 UTF-8 文本中，一个码点可能占一到四个字节；新文本的编码长度也可能不同。

例如，把一个三字节汉字直接覆盖成一个一字节 ASCII 字符，如果没有重新安排后面的字节，就可能留下原汉字的尾部字节，形成非法编码。

所以，随机定位适合固定格式和明确字节布局的需求。普通文本编辑优先按文本解析、修改内存数据，再生成新文件。

## 第十九章：`open()` 与 JSON、CSV、压缩文件应该怎么配合？

这一章只讲文件 I/O 与格式处理之间的交界，具体格式工具应在各自文档中继续学习。

### 19.1 `open()` 提供流，JSON 工具解释结构

```python
import json

with open("settings.json", encoding="utf-8") as source:
    settings = json.load(source)

print(settings["theme"])
```

`open()` 不知道 JSON 的花括号、数组和布尔值有什么含义。它提供可读取文本的流。

`json.load()` 从流中取得内容并解析为 Python 对象。文件能被打开，不代表它是合法 JSON；JSON 能被解析，也不代表它具有 `theme` 这个键；具有这个键，也不代表值符合业务要求。

这里至少存在三层检查：I/O、数据格式、业务结构。它们不能彼此代替。

写入时：

```python
import json

settings = {"theme": "dark", "language": "中文"}

with open("settings_out.json", "w", encoding="utf-8", newline="\n") as target:
    json.dump(settings, target, ensure_ascii=False, indent=2)
    target.write("\n")
```

`ensure_ascii=False` 让 JSON 文本中的中文保持可读形式；真正把字符串编码成 UTF-8 的，仍然是文本流的 `encoding`。

### 19.2 CSV 为什么强调 `newline=""`？

CSV 字段本身可以包含逗号、引号甚至换行。解析器必须区分“记录之间的换行”和“被引号包住的字段内部换行”。

因此，打开 CSV 时交出未经额外通用换行转换的流，能让 `csv` 模块按自己的规则处理这些边界。

```python
import csv

with open("students.csv", encoding="utf-8-sig", newline="") as source:
    reader = csv.DictReader(source)
    for row in reader:
        score = int(row["score"])
        print(row["name"], score)
```

`DictReader` 根据首行建立键与列的对应关系。读出的普通字段仍然是字符串，分数需要由你转换和验证。`open()` 不会因为文件扩展名是 `.csv` 就把数字字段变成整数。

### 19.3 压缩文件不能靠普通文本模式直接解压

如果 `notes.txt.gz` 是 gzip 压缩数据，那么：

```python
open("notes.txt.gz", encoding="utf-8")
```

不会执行解压。它会尝试把压缩后的原始字节直接当 UTF-8 文本读取，通常失败或得到无意义结果。

应选择对应格式接口：

```python
import gzip

with gzip.open("notes.txt.gz", "rt", encoding="utf-8") as source:
    text = source.read()
```

这里 `gzip.open()` 先处理 gzip 压缩层，`t` 再选择文本层。要写压缩文本，可以使用 `"wt"`。

这说明“看起来都有 `open` 这个名字”不等于“它们打开同一种底层数据”。先确定格式和容器，再选择相应库，是可靠文件处理的基础。

### 19.4 多层包装时，谁负责关闭？

如果你自己创建底层二进制流，再包装成解压流、文本流，就会出现多层对象与关闭传播的关系。不同包装器的所有权规则需要查各自文档，不能一概假设“关最外层一定关所有内层”。

初学阶段优先使用库提供的高层上下文管理形式，例如 `gzip.open(..., "rt")`。需要手动组合资源时，可以使用 `ExitStack` 逐个登记，明确每个对象的清理责任。

## 第二十章：进一步理解文件类对象

### 20.1 `StringIO`：让字符串表现得像文本文件

```python
from io import StringIO

with StringIO() as f:
    print("姓名,分数", file=f)
    print("小林,95", file=f)
    text = f.getvalue()

print(text)
```

`StringIO` 在内存中保存文本，不创建磁盘文件。它适合生成小型文本、测试接受文件对象的函数、让 `csv` 等工具读写内存数据。

`getvalue()` 取得完整内容，不受当前位置是否在末尾影响；`read()` 仍从当前位置读取。需要在关闭之前调用 `getvalue()`。

### 20.2 `BytesIO`：内存中的二进制流

```python
from io import BytesIO

with BytesIO() as f:
    f.write(b"ABC")
    f.seek(0)
    print(f.read(2))
```

输出 `b"AB"`。它与磁盘二进制文件有相似接口，却通常没有可供 `os.fsync()` 使用的真实文件描述符。

如果一个函数只需要 `.read()`，可以把这种依赖写成“接受可读取流”，而不必强迫调用者提供磁盘路径。反过来，编写通用流函数也不要无条件假设每个对象都有 `.name`、`.buffer` 或可用的 `.fileno()`。

### 20.3 动态打开多个文件：`ExitStack`

```python
from contextlib import ExitStack

paths = ["part1.txt", "part2.txt", "part3.txt"]

with ExitStack() as stack:
    files = [
        stack.enter_context(open(path, encoding="utf-8"))
        for path in paths
    ]
    first_lines = [f.readline() for f in files]
```

如果打开后面的文件时出错，已经由 `stack` 管理的前面几个文件也会被关闭。文件特别多时，仍应避免超过系统打开数量限制；能逐个处理时，逐个打开更简单。[官方说明：ExitStack](https://docs.python.org/3.14/library/contextlib.html#contextlib.ExitStack)

## 第二十一章：Windows、WSL 与 Linux 文件操作的实际差异

同一段 `open()` 语法在不同平台很相似，但路径解释、文件共享和大小写行为可能不同。

### 21.1 WSL 中的 Python 使用 Linux 路径规则

在 Windows 文件管理器中：

```text
E:\notes\Python\example.txt
```

在常见 WSL 挂载布局中，对应路径通常写为：

```text
/mnt/e/notes/Python/example.txt
```

在 WSL 的 Python 中：

```python
from pathlib import Path

path = Path("/mnt/e/notes/Python/example.txt")
```

这里运行的是 Linux Python。`Path(r"E:\notes\Python\example.txt")` 不会自动替你把盘符翻译成 `/mnt/e`。

WSL 挂载点可以被配置，因此真实环境应以当前挂载情况为准。教程中的映射说明常见布局，不表示任意机器都使用相同挂载目录。

### 21.2 Windows 中的原始字符串解决的是转义，不是路径有效性

```python
path = r"E:\notes\Python\example.txt"
```

前缀 `r` 让反斜杠更方便地按字面表达，不会自动创建盘符、验证文件存在、翻译到 WSL，也不会赋予读取权限。

路径里还有空格时，Python 字符串本身可以正常保存它：

```python
path = r"E:\My Notes\example.txt"
```

直接传给 `open(path)` 即可，不要再把双引号字符加进路径值。终端中为了避免参数被空格分开而加的引号，与 Python 字符串对象里真正保存什么，是两个层次。

### 21.3 已打开的文件能不能删除或重命名？

在 Unix 常见语义中，删除目录中的名字后，已经打开该文件的进程可能仍能通过原描述符访问内容，空间通常要等相关引用释放后才真正回收。

Windows 的文件共享标志和占用限制不同，有些情况下会拒绝删除或替换正在使用的文件。

因此，“我在 Linux 上边打开边替换没有报错”，不能推出这段程序在 Windows 上也一样。跨平台程序应缩短文件保持打开的时间，并在替换前关闭自己不再需要的流。

### 21.4 大小写与换行不要靠猜

某些文件系统区分 `Data.txt` 和 `data.txt`，某些常见 Windows 配置则通常不区分，但具体目录和文件系统设置仍可能不同。

文件内容的换行也不是由扩展名决定。Windows 上可以保存 LF，Linux 上也能读取 CRLF。决定你在 Python 中看到什么的是实际字节和 `newline` 配置。

遇到跨平台错误，建议先输出路径绝对形式、当前工作目录、打开模式和预期编码，再检查文件是否真在那个位置。不要为了修复一个相对路径问题，随意修改整个进程的工作目录。

## 第二十二章：进阶接口——文件描述符、`closefd` 与 `opener`

普通脚本主要使用路径打开文件。理解了高层接口后，再看少量底层能力，就能知道前面参数表中那些平时不常写的选项解决什么问题。

### 22.1 文件描述符是什么？

文件描述符通常是一个整数，是操作系统在进程中标识已打开资源的一种方式。

`open()` 返回 Python 文件对象；`os.open()` 返回底层文件描述符。两者名字相近，调用参数和返回结果却不同。

```python
import os

fd = os.open("descriptor.txt", os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)

try:
    with open(fd, "w", encoding="utf-8", closefd=False) as target:
        target.write("来自文件描述符包装的文本\n")
finally:
    os.close(fd)
```

`os.O_WRONLY | os.O_CREAT | os.O_TRUNC` 用按位或组合打开标志。这个组合表达只写、必要时创建、已有内容截断。

`0o600` 是请求的创建权限模式，在类 Unix 平台上大致表达仅所有者可读写，实际生效还受 `umask` 和文件系统规则影响。它不是 Windows 完整访问控制列表的通用替代。

`open(fd, ...)` 把描述符包装成更方便的文本流。`closefd=False` 表示关闭包装流时保留底层描述符，由外层 `finally` 最终负责关闭它。

这里必须保持责任一致：如果你让包装对象负责关闭，就不要又在外层无条件重复关闭同一个描述符。底层整数可能被系统重新分配，错误的重复关闭会造成难以排查的问题。

### 22.2 `closefd` 不是“不要关闭文件”的日常开关

这个参数主要用于传入整数描述符的情况。

如果 `file` 是普通路径，不能通过 `closefd=False` 让上下文关闭后仍保持文件可用。需要延长文件生命周期，就应把相关操作放在同一个合理的 `with` 范围内，或者明确把资源管理交给调用者。

不要为了绕过“文件已关闭”的错误去试 `closefd=False`。先判断是不是把文件对象带出了它所属的上下文。

### 22.3 `os.fdopen()` 与内置 `open(fd, ...)`

`os.fdopen()` 也是把已有描述符包装成文件对象的接口。它的用途与直接将描述符传给内置 `open()` 有重叠。

在你接触子进程、管道、临时文件的低层创建接口时，可能会得到描述符，再包装成 Python 流。这里最重要的不是背两种写法，而是确认：谁创建资源、谁拥有资源、谁关闭资源。

### 22.4 `opener` 让你定制底层打开动作

`opener` 接受一个可调用对象，调用形式是接收路径和标志，返回已打开的描述符。普通文本编码和缓冲仍由 `open()` 后续处理。

下面展示一个 Unix 风格的目录描述符相对打开示例：

```python
import os

base_fd = os.open("data", os.O_RDONLY)


def relative_opener(path, flags):
    return os.open(path, flags, dir_fd=base_fd)


try:
    with open("input.txt", "r", encoding="utf-8", opener=relative_opener) as source:
        text = source.read()
finally:
    os.close(base_fd)
```

执行前 `data/input.txt` 必须存在。这个示例依赖平台对目录描述符和相关参数的支持，不适用于所有操作系统。

`relative_opener` 中的 `dir_fd` 为相对路径提供目录基准。但这段代码没有自动拒绝绝对路径、`..` 或符号链接，因此不能仅靠它宣称“用户绝对无法访问目录之外”。真实的权限边界需要结合平台能力和威胁模型设计。

### 22.5 不要混用高层缓冲位置与底层描述符位置

如果一个描述符已经由文本流或二进制缓冲流管理，再绕开包装器调用 `os.read()`、`os.write()` 或 `os.lseek()`，可能让缓冲器记录的状态与底层位置不一致。

例如，文本流为了提高效率，可能提前从底层读入一批字节，但只向调用者交出一行。此时你直接从底层再读，会跳过仍停留在包装器缓冲区中的数据。

普通程序应选择一个一致的 I/O 层工作。需要底层操作时，先理解缓冲器的状态和所有权约定，而不是在几个名称相似的函数之间随意切换。

## 第二十三章：性能问题——什么时候逐行，什么时候分块，什么时候一次读取？

文件处理没有一种写法能在所有场景中同时最简单、最快、最省内存。判断方法应从数据结构和工作方式出发。

### 23.1 小型配置文件：整体读取更清楚

几十行的配置文件、少量模板、一个小型 JSON 文档，整体读取往往最直观。

```python
with open("template.txt", encoding="utf-8") as source:
    template = source.read()

result = template.replace("{{name}}", "小林")
```

这里整体字符串替换需要完整上下文，输入也受控，没有必要为了“流式”把它拆成复杂的块边界状态机。

但是，`replace()` 会生成新的字符串，可能在一段时间内同时保留原文和结果。对很大的文本，这种内存开销需要考虑。

### 23.2 按行组织的数据：逐行处理更自然

日志、简单记录、逐行 JSON 等格式通常能一行一行解释。

```python
count = 0

with open("events.txt", encoding="utf-8") as source:
    for line in source:
        if "ERROR" in line:
            count += 1
```

这里不需要保留匹配行，只累计数量。程序的数据规模主要随当前行长度变化，不随总行数保存一个持续增长的列表。

如果你改成 `matches = [line for line in source if ...]`，就重新把所有匹配结果留在内存中了。“用了迭代器”不等于后面的所有操作都不会累积数据。

### 23.3 与行无关的原始内容：按字节块处理

复制、哈希、上传原始文件时，按二进制块读取更合适。

```python
import hashlib

hasher = hashlib.sha256()

with open("download.bin", "rb") as source:
    while chunk := source.read(1024 * 1024):
        hasher.update(chunk)

print(hasher.hexdigest())
```

`:=` 是赋值表达式。它把读取结果赋给 `chunk`，再用这个结果判断循环是否继续。Python 3.8 及以上支持这种语法。

如果你暂时不熟悉赋值表达式，继续使用 `while True`、赋值、判断、`break` 的版本完全没有问题。学习重点是读取与退出条件，而不是追求短代码。

这里哈希的是原始字节，不受文本换行转换影响。SHA-256 用于摘要与完整性比较；如果要验证来源可信，还需要可信的摘要来源、签名或认证机制，单独算一个摘要不能证明作者身份。

### 23.4 不要把块读取和字符串边界处理混为一谈

假设你要把所有 `Python` 替换成 `PYTHON`。直接对每个独立文本块调用 `replace()`，可能漏掉跨块出现的单词：前一块末尾是 `Pyt`，后一块开头是 `hon`。

这不是文件读取错误，而是处理算法没有保存跨块上下文。

同理，正则匹配、分隔符识别和协议解析都可能跨块。流式读取只提供数据块，不会自动解决你在数据块上执行的算法边界。

如果文件较小，整体读取可能更清楚；如果必须流式处理，就要设计保留尾部、增量解析或采用专用流式解析工具。

### 23.5 `readinto()` 是什么时候用的？

某些二进制流支持 `readinto(buffer)`，把数据写入你提供的可变缓冲区，减少反复创建新字节对象的需求。

```python
buffer = bytearray(4096)

with open("download.bin", "rb") as source:
    while True:
        size = source.readinto(buffer)
        if size == 0:
            break

        view = memoryview(buffer)[:size]
        print("本轮有效字节数：", len(view))
```

`bytearray(4096)` 创建可修改的字节缓冲。`readinto()` 返回实际读入数量，最后一轮可能不足整个缓冲长度，所以只能处理前 `size` 个字节。

这里用 `memoryview` 展示不必每次复制有效片段的思路。保存这些视图到后面使用时要小心：下一次读取会覆盖同一个底层缓冲，旧视图看到的内容也会变化。普通脚本的 `read(chunk_size)` 更易读，先测量再决定是否需要这种优化。

### 23.6 性能测量要看完整工作

不要只计时 `open()` 一行，然后声称某种读取方法一定快几十倍。完整处理还包括读取、解码、解析、转换、写入和关闭。

操作系统文件缓存、磁盘类型、文件大小和第一次读取是否已经预热，都可能显著影响测量。

比较两个方案时，应使用相同输入、相同输出要求和相同环境，多次观察，并确保比较的程序确实做了同样的工作。对于教材示例，正确和清楚通常比没有测量依据的微小优化更重要。

## 第二十四章：修改已有文本时，如何选择保存策略？

文件更新有几种不同需求。把它们混为一谈，容易误用模式。

### 24.1 只是记一条新事件：追加

如果数据天然是不断增加的记录，例如简单任务历史，可以使用 `a`。

你需要事先决定记录分隔、编码、单条记录格式和并发写入规则。以后要修改一条旧记录时，追加日志可能需要写“更正事件”，而不是直接覆盖原位置。

### 24.2 重新生成一份报告：覆盖写或临时替换

报告往往是根据输入重新计算的最终结果。此时旧报告的内容不需要逐字保留，使用 `w` 重新写入很自然。

如果旧报告在生成失败时仍然应保持可用，则应使用临时文件替换。这里的区别在于失败时读者应看到什么，而不只是代码少写几行。

### 24.3 修改固定长度二进制字段：明确偏移的读写

已知格式规定偏移和字节长度时，可以使用 `r+b`、`seek()` 和按协议编码的数据进行覆盖。

这要求新字段长度符合格式，读取者能够接受更新过程，并且你已经考虑写入中断的情形。覆盖几个字节也不是天然具备事务性。

### 24.4 修改任意长度文本：重新生成

把一个单词换成另一个更长的单词，通常需要重新安排后续字节。最容易理解的方法是读取、修改字符串、写新文件。

小文件示例：

```python
from pathlib import Path

path = Path("draft.txt")

with path.open(encoding="utf-8") as source:
    old_text = source.read()

new_text = old_text.replace("TODO", "待处理")

save_text(path, new_text)
```

这里使用前面定义的 `save_text()`。如果结果与原文相同，还可以主动选择不保存，以免无意义地改变文件修改时间。

### 24.5 多个字段、多个文件、并发修改：考虑数据库

假设一个程序每次都要更新“余额文件”“订单文件”“库存文件”，并且三份必须同时成功或同时失败。单个 `with open()` 和单个文件的替换都不够表达这样的事务。

你需要理解锁、日志、恢复和事务隔离等更完整的机制。很多业务直接使用 SQLite 或其他数据库，会比自己拼装一套文件事务更可靠。

文件处理知识仍然有用：数据库导入导出、日志、备份和配置都要用到。但不要因为学会 `r+`，就把所有持久化需求都变成手动修改文本。

## 第二十五章：实践诊断——看到报错后按什么顺序检查？

### 25.1 “文件不存在”，先确认程序实际找哪里

```python
from pathlib import Path

path = Path("data/input.txt")

print("当前工作目录：", Path.cwd())
print("传入路径：", path)
print("绝对形式：", path.absolute())
```

`absolute()` 在这里帮助你看清基准，不会自动创建文件。不要只盯着编辑器左侧的项目目录：IDE 的运行配置可能使用另一工作目录。

还应检查文件名的实际拼写、大小写、扩展名是否被文件管理器隐藏，以及父目录是否存在。文件名末尾多一个空格等情况也可能让视觉判断失准，可以用 `repr(str(path))` 查看字符串的精确形式。

### 25.2 “解码错误”，先确认原始编码与字节位置

不要第一步就加 `errors="ignore"`。

先确认文件来自哪个程序，生成方是否说明编码，编辑器显示的当前编码是什么。必要时先 `rb` 读取少量开头字节，用 `repr()` 或十六进制观察 BOM 等线索。

```python
with open("input.txt", "rb") as source:
    prefix = source.read(32)

print(prefix)
print(prefix.hex(" "))
```

字节样本可以提供线索，却不能凭有限样本总是唯一识别编码。尤其纯 ASCII 内容同时兼容许多编码，不能因为前几行成功解码就认定整份文件一定使用 UTF-8。

### 25.3 “读到空内容”，区分空文件和位置已经在末尾

检查是不是之前已经调用过 `read()`，是不是使用 `a+` 后没有 `seek(0)`，是不是返回文件对象时外层上下文已经关闭。

也要检查是不是在读取前用 `w` 打开了同一路径。这个问题会真正改变文件内容，不能靠再 `seek(0)` 恢复原始数据。

### 25.4 “写入成功但内容没出现”，检查完整生命周期

检查写入目标是否与你查看的路径相同；程序是否仍在 `with` 里面，缓冲是否尚未刷新；是否中途发生异常；是否被另一个任务覆盖；是否因为当前工作目录变化写到了别处。

如果只是编辑器没有刷新，重新加载可以帮助观察。但不能把所有保存异常都归因于界面缓存。先查看真实路径和文件状态，才知道是否已经写到磁盘文件系统中。

### 25.5 “关闭后不能操作”，回到资源范围检查

```python
# 容易写错的结构：
with open("notes.txt", encoding="utf-8") as source:
    pass

# 这里 source 已经关闭。
```

把依赖文件的操作放在缩进块内，或在块内把需要的内容读成独立数据。不要为了让错误暂时消失就删除 `with`，却又没有补上可靠的关闭责任。

## 第二十六章：把几个容易混淆的问题再想清楚

### 26.1 已经调用 `open()`，为什么还要调用 `read()`？

因为打开和读取是两个动作。

`open()` 返回一个文件对象，你通过它选择读多少、从哪里读、怎样处理。

如果你要写入，也不必先把整个旧文件读出来。

这种分工让同一个接口可以服务读取、写入和逐步处理等需求。

### 26.2 我还没有写内容，为什么旧文件已经空了？

先检查模式是不是 `w`。

在普通文件上，`w` 成功打开时就会截断旧内容，不会等到第一次 `write()` 才这样做。

如果只是想读取，不要为了“万一以后要写”提前使用覆盖模式。

如果要重新生成结果，可以先在内存中完成验证，或者先生成临时结果，再按既定政策发布。

### 26.3 `+` 和 `a` 都有增加的感觉，它们有什么区别？

`+` 表示这次打开允许读和写。

`a` 表示追加模式，写入针对文件末尾。

所以 `r+` 不等于追加，`a+` 则把追加与允许读取组合在一起。

看到模式字符串时，先判断基本模式，再理解 `+`、`b` 等修饰，不要按中文直觉猜。

### 26.4 连续调用两次 `read()`，为什么第二次没有内容？

第一次无上限读取通常已经把当前位置推进到末尾。

第二次会继续从当前位置读取，而不是自动从头再来。

对支持定位的普通文件，可以 `seek(0)` 后重读。

也可以重新打开文件，从一次新的读取过程开始。

### 26.5 读到一行空字符串，就是读到空白行吗？

用 `readline()` 时，空字符串表示这次没有读到字符，通常是已经到达末尾。

单纯的空白行仍然包含换行符，一般会读成 `"\n"`。

如果你先对行调用 `strip()` 再判断，就主动把很多空白信息丢掉了。

所以需要区分空白行与输入结束时，应先检查原始读取结果。

### 26.6 我只想去掉换行，为什么缩进也没了？

因为 `strip()` 会去掉两端多种空白，并不只处理换行。

如果源数据的空格有意义，应按本章前面介绍的格式规则处理行尾。

例如，清理统一读取后的单个 `\n`，和清理任意空白，是两个不同需求。

### 26.7 `with` 已经退出，为什么结果还是可能不完整？

`with` 主要安排退出时的资源处理。

它不会把整个代码块自动变成事务。

中途已经成功写出的字节，不会因为后来抛异常就自动撤销。

同样，`flush()` 刷新流缓冲，也不等于已经解决断电、硬件故障和跨文件一致性。

选保存策略时，应先说明你希望避免哪一种失败，再选择相应机制。

### 26.8 后缀和编码，是不是都在描述文件格式？

文件后缀是一种命名约定，文本编码描述字符与字节之间怎样转换。

两者都不能代替内容验证。

一个叫 `photo.jpg` 的文件未必能解码为图片，一个叫 `config.json` 的文件也未必是合法 JSON。

打开路径、解释字节、解析格式和验证业务结构，需要分别做好。

## 第二十七章：带答案的巩固练习

### 27.1 练习一：准确区分空白行和输入结束

要求：逐行读取文件，空白行也要输出行号；只有到达 EOF 才结束；不得用 `if not line.strip(): break` 作为结束条件。

参考实现：

```python
with open("lines.txt", encoding="utf-8") as source:
    number = 0
    while True:
        line = source.readline()
        if line == "":
            break
        number += 1
        print(number, repr(line))
```

这里用精确的空字符串判断结束。对于一个只含换行的空白行，`line` 是 `"\n"`，不会误退出。

自己再验证：空文件、两行之间有空白行、最后一行没有换行。这三种输入能暴露很多“看起来差不多”的读取条件其实不等价。

### 27.2 练习二：保存用户输入，保留首尾空格

要求：将一个已经取得的字符串保存到文件，不要因为清理换行而删除用户输入中的缩进和尾部空格。

```python
user_text = "    需要保留缩进的内容  "

with open("user_text.txt", "w", encoding="utf-8", newline="\n") as target:
    target.write(user_text)
    target.write("\n")
```

这里没有调用 `strip()`，因为这些空白属于数据。是否需要增加末尾换行，是你制定的输出格式规则，而不是 `write()` 自动做的事情。

### 27.3 练习三：只允许生成新文件

要求：如果目标名字已存在，无论它原来有多少内容，都不要先清空它。

```python
try:
    with open("unique_result.txt", "x", encoding="utf-8") as target:
        target.write("新生成的结果\n")
except FileExistsError:
    print("目标已存在，未进行覆盖")
```

关键在于 `x`。如果换成先判断 `exists()` 再使用 `w`，两个操作之间仍可能出现竞争。这个练习不需要先打开旧文件来判断是否为空，因为需求是“名字已经存在就拒绝”。

### 27.4 练习四：读取前五行，而不是五个字符

```python
from itertools import islice

with open("notes.txt", encoding="utf-8") as source:
    for line in islice(source, 5):
        print(line, end="")
```

`islice(source, 5)` 从文件的逐行迭代中最多取得五个元素。文件不足五行就自然结束，不会因为缺少行而报错。

与之相比，`source.read(5)` 最多读取五个字符；`source.readlines(5)` 的参数是累计读取量提示。这三个表达式不能互相替代。

### 27.5 练习五：重读第二行

```python
with open("notes.txt", encoding="utf-8") as source:
    first = source.readline()
    second_start = source.tell()
    second = source.readline()

    source.seek(second_start)
    second_again = source.readline()

print(second == second_again)
```

不要把 `second_start` 理解为第一行汉字数量。它是流能识别的位置标记。只要文件内容和相关流状态没有被外部修改，原样恢复标记可以回到对应位置。

### 27.6 练习六：保证逐字节复制一致

要求：输入可能包含 CRLF、零字节和任意非文本字节；输出的每个字节都应保持一致。

```python
import shutil

with open("original.bin", "rb") as source:
    with open("copied.bin", "xb") as target:
        shutil.copyfileobj(source, target, length=1024 * 1024)
```

这里不用文本模式，因此不会触发编码或换行转换。`copyfileobj()` 从当前源流位置开始复制，所以如果前面已经读过文件头，需要先决定是否重新定位。

它复制的是内容流，不承诺保留源文件的所有元数据；目标关闭前也不能把复制完成理解成任何故障下都不会丢数据。

### 27.7 练习七：把读取问题与格式问题分开报告

```python
import json

try:
    with open("settings.json", encoding="utf-8") as source:
        settings = json.load(source)
except OSError as error:
    print("文件访问失败：", error)
except UnicodeDecodeError as error:
    print("文件编码不符合 UTF-8：", error)
except json.JSONDecodeError as error:
    print("文件不是合法 JSON：", error)
else:
    print("解析成功：", type(settings).__name__)
```

`else` 只有前面的 `try` 没有抛异常时才执行。这样的结构让你明确看出：读取失败、解码失败和格式解析失败分别由谁解释。

解析成功只说明结构符合 JSON 语法，不代表拿到的必然是字典。它也可以是列表、字符串或其他 JSON 顶层值，后续仍需要验证业务结构。

### 27.8 练习八：说明为什么这段代码会丢内容

```python
# 仅供分析，不要对真实重要文件执行。
with open("important.txt", "w", encoding="utf-8") as target:
    raise RuntimeError("还没来得及写入，就失败了")
```

成功进入 `with` 之前，`open(..., "w")` 已经完成截断。随后抛异常，文件对象会尝试关闭，但关闭不会恢复旧字节。

修正方向取决于需求：尽量先做验证；重要结果先写临时文件；需要多记录事务时使用合适的持久化系统。加一个 `except` 打印“失败”不会让旧文件自动回来。

## 第二十八章：把学习结果变成一份检查清单

下面的清单用于写代码前自查，尤其适合刚开始独立完成脚本时使用。

- [ ] 我知道相对路径是相对于哪个工作目录。
- [ ] 我确认输入输出没有无意指向同一个文件。
- [ ] 我选择了符合需求的模式，理解是否创建、截断或追加。
- [ ] 我知道流中传递的是 `str` 还是字节类对象。
- [ ] 我有文件内容编码的依据，而不是看到中文就随机试编码。
- [ ] 我知道是否需要保留原始换行或原始字节。
- [ ] 我没有把 `strip()` 当成没有副作用的万能清理。
- [ ] 我明确资源由谁打开、谁关闭，提前退出时也能清理。
- [ ] 我区分普通小文件、巨型文件和可能无限产生数据的流。
- [ ] 我知道失败时会留下旧文件、新文件还是部分文件。
- [ ] 我没有把 `flush()`、关闭和持久化事务当成同一件事。
- [ ] 我在临时练习目录验证了空输入、缺失路径和非法编码等边界。

这份清单并不要求每个两行脚本都写成复杂框架。它帮助你看见真正会影响结果的选择，再按任务的重要性决定需要做到哪一步。

## 第二十九章：常用搭配与返回值集中对照

| 需求 | 典型写法 | 结果或作用 | 需要记住的限制 |
| --- | --- | --- | --- |
| 整体读取 UTF-8 | `f.read()` | `str` | 从当前位置开始，结果占内存 |
| 读取部分文本 | `f.read(100)` | 最多一百个字符 | 不是一百行或固定字节数 |
| 读取一行 | `f.readline()` | 含可能换行的 `str` | EOF 返回空字符串 |
| 保存剩余所有行 | `f.readlines()` | `list[str]` | 会保留全部结果 |
| 逐行处理 | `for line in f` | 每轮一行 | 不一定支持同时 `tell()` |
| 写一段文本 | `f.write(text)` | 写入字符数 | 不自动添加换行 |
| 写多段文本 | `f.writelines(parts)` | 通常为 `None` | 不自动添加分隔符 |
| 格式化输出 | `print(value, file=f)` | 通常为 `None` | 默认添加换行并转换文本 |
| 查询位置 | `f.tell()` | 整数位置标记 | 文本流不能当字符下标 |
| 返回开头 | `f.seek(0)` | 新位置 | 需要流支持定位 |
| 修改长度 | `f.truncate(size)` | 目标大小信息 | 不按汉字数量裁切 |
| 刷新缓冲 | `f.flush()` | 通常为 `None` | 不等于完整故障持久性 |
| 关闭资源 | `f.close()` | 通常为 `None` | 后续不能继续读写 |
| 使用路径对象 | `Path(...).open(...)` | 文件对象 | 与内置 `open()` 的普通读写规则相同 |
| 处理内存文本 | `StringIO(...)` | 文本流 | 不提供真实磁盘文件的全部能力 |
| 处理内存字节 | `BytesIO(...)` | 二进制流 | 整个内容仍然保存在内存 |

学习时可以先只熟练掌握 `with`、模式选择、明确编码、逐行迭代和 `write()`。遇到需要随机访问、增量处理或可靠替换的需求，再返回相关章节，把选择与具体问题对应起来。

官方参考：[内置 open()](https://docs.python.org/3.14/library/functions.html#open)、[Python 文件读写教程](https://docs.python.org/3.14/tutorial/inputoutput.html#reading-and-writing-files)、[io 流接口](https://docs.python.org/3.14/library/io.html)。
