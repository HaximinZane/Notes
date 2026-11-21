# Python `subprocess` 库超详细教程：从启动另一个程序，到输入输出、超时、管道与可靠调用

## 零基础先导：Python 为什么还要启动“另一个程序”？

`subprocess` 直译是“子进程”：你的 Python 脚本是一个运行中的进程，它可以向操作系统请求启动另一个程序，并等待其结束、拿到退出码和输出。

假设你已经写好了 `converter.exe` 或另一个 Python 脚本。你不需要把这个程序的源代码复制进当前脚本，而可以**调用它**。这不同于 `import`：`import` 主要加载模块到当前 Python 解释器，`subprocess` 通常启动一个新的进程。

### 第一次调用只做一件事

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "print('子程序你好')"],  # 要启动的程序与参数
    capture_output=True,                           # 收集 stdout/stderr
    text=True,                                     # 将输出解码为字符串
    check=True,                                    # 非零退出码时抛异常
)
print(result.stdout.strip())                        # 子程序你好
```

`sys.executable` 取得**当前正在运行的 Python 解释器路径**，这样不必猜测电脑上哪个 `python` 命令对应当前环境。`-c` 是 Python 解释器的命令行选项，表示运行后面的代码字符串。`result` 是 `CompletedProcess` 对象，不是子程序本身。

**最容易混淆**：默认 `shell=False`，列表里的每一项就是一个参数；字符串里的管道符 `|` 或重定向符 `>` **不会自动按 shell 语法执行**。打开 `shell=True` 是另一套调用方式，若把未经验证的用户输入拼进 shell 字符串，可能造成命令注入；普通场景优先用参数列表。

进阶内容里出现 `Popen`，通常是因为你要边运行边交互、不想立刻等结束，或要处理复杂输入输出。只运行一个命令并等待结果时，优先学好 `subprocess.run`。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

假设你已经写好了一个 Python 程序，现在想让它再执行一个外部工具。

例如：调用另一个 Python 脚本，运行图片转换器，检查工具版本，或者让几个小程序协作处理数据。

你可能首先想到：

```python
import os
os.system("python child.py")
```

但接下来会产生很多问题。

我怎么拿到子程序打印的内容？

如果它报错，错误信息在哪里？

路径中有空格怎么办？

等待太久能不能停止？

我想把一段文本送进去，再取得处理结果，怎么连接？

这一篇从最小例子开始，逐步把这些问题讲清楚。

常规示例面向 Python 3.11 及以上，优先使用当前解释器运行可控的小程序，减少对 Windows、Linux 命令名称差异的依赖。

## 第一章：进程、子进程和 shell，先分清三个概念

### 1.1 什么是进程？

可以先把进程理解为一个正在运行的程序实例。

你保存了一份 `main.py`，它是磁盘上的文件。

你执行它，操作系统为运行过程建立资源和状态，这次运行就处于某个进程中。

同一份程序可以同时启动两次，形成两个不同进程。

### 1.2 子进程是什么意思？

如果当前 Python 程序再启动一个外部程序，新启动的程序通常被称为它的子进程。

```text
main.py 所在的 Python 进程
    │ 启动
    ▼
child.py 所在的另一个 Python 进程
```

父进程和子进程拥有各自的运行状态。

子进程里的普通变量赋值不会自动修改父进程同名变量。

子进程改变自己的工作目录，也不会自动让父进程一起切换目录。

需要交换信息，就通过输入输出、文件、管道或其他明确通信方式连接。

### 1.3 shell 又是什么？

Bash、Windows 命令提示符和 PowerShell 都属于命令环境，但语法与行为不同。

它们可以解释引号、变量展开、重定向和管道等命令语言规则。

例如 Bash 中：

```bash
python app.py > result.txt
```

`>` 通常由 shell 解释为输出重定向。

Python 脚本本身不必收到一个普通的 `>` 参数。

而直接启动一个程序时，不一定需要经过 shell。

这一区别会影响参数边界、可移植性和安全性。

### 1.4 `subprocess` 名称和由来

`sub` 有下一级、子级的含义，`process` 是进程。

模块提供启动子进程、连接标准流、等待结果和取得返回状态的能力。

它把以前分散在 `os.system()`、部分旧式进程接口中的常见需求，组织成更一致的接口。[官方定位](https://docs.python.org/3.14/library/subprocess.html)

它是标准库，直接：

```python
import subprocess
```

不需要额外安装。

## 第二章：第一个 `run()`，只让另一个 Python 打印一句话

### 2.1 最小可运行例子

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "print('hello from child')"],
    check=True,
)

print("父进程继续执行")
print(result.returncode)
```

通常看到：

```text
hello from child
父进程继续执行
0
```

先理解三件事。

`subprocess.run()` 启动指定程序。

它通常等待这次子进程结束。

结束后返回一个 `CompletedProcess` 结果对象。

### 2.2 列表第一项为什么是 `sys.executable`？

`sys.executable` 通常给出当前 Python 解释器可执行文件的位置。

假设你当前使用虚拟环境：

```text
D:\project\.venv\Scripts\python.exe
```

我们希望子进程也使用这一套解释器，而不是系统 PATH 中另一个 `python`。

因此明确使用它。

某些嵌入或特殊运行环境里这个值可能不可用；普通脚本环境通常可以使用，但不能把特殊情况当成绝对不可能。

### 2.3 `-c` 是 subprocess 的参数吗？

这里不是。

`-c` 是 Python 解释器自己的选项，表示执行紧随其后的一段代码字符串。

所以列表表达的是：

```text
程序：当前 Python
参数一：-c
参数二：print('hello from child')
```

`subprocess` 负责启动程序和传参数。

参数对目标程序意味着什么，由目标程序决定。

### 2.4 为什么外部输出直接出现在终端？

因为我们没有要求捕获标准输出和标准错误。

子进程通常继承父进程相应流的去向。

如果父程序在终端中运行，子程序的输出也通常会显示在那里。

这不意味着 `result` 变量保存了输出文本。

### 2.5 `result.returncode` 为什么是零？

零通常表示子程序成功结束。

它是外部程序传回的状态，不是打印字符数，也不是子进程 ID。

不同工具可以定义不同非零状态含义，应参考目标程序约定。

## 第三章：参数列表，为什么比拼一条长字符串更清楚？

### 3.1 每一项保留一个参数边界

准备一个子脚本 `show_args.py`：

```python
import sys

for index, value in enumerate(sys.argv):
    print(index, repr(value))
```

父程序调用：

```python
import subprocess
import sys

subprocess.run(
    [sys.executable, "show_args.py", "Alice Smith", "data/input.txt"],
    check=True,
)
```

子脚本收到的 `Alice Smith` 是一个完整参数。

你不需要为了其中的空格，给列表元素再加一对真正的引号字符。

### 3.2 与终端命令中的引号有什么区别？

终端里你可能写：

```bash
python show_args.py "Alice Smith"
```

引号帮助命令环境把含空格内容分成一个参数。

在 Python 列表里，元素边界已经明确。

所以：

```python
"Alice Smith"
```

这个字符串值就是所需内容。

不要变成：

```python
'"Alice Smith"'
```

后者在字符串内部包含双引号字符，目标程序可能把它们也视为数据。

### 3.3 Windows 不是最终还要转换命令行字符串吗？

Windows 底层进程创建接口与 POSIX 的参数数组机制不同，Python 会根据相应规则把序列转换为命令行表示。

目标程序如何再解释命令行，也可能有自己的规则。

因此，参数列表是常见正确起点，却不是对所有具有特殊自定义解析器的程序都能作绝对保证。

遇到非常特殊的 Windows 工具，应核对它的参数规则并做实际测试。

### 3.4 直接给一个带空格的字符串，默认会自动分词吗？

不要这样假设。

```python
# 不推荐：默认 shell=False 时，这不是通用的 shell 分词接口。
subprocess.run("python show_args.py Alice")
```

在 POSIX 常见场景里，它可能把整段当成可执行文件名，而不是帮你拆成三个词。

不同平台的底层行为还有差异。

因此，普通直接执行采用参数列表：

```python
subprocess.run([sys.executable, "show_args.py", "Alice"])
```

### 3.5 能不能用 `command.split()` 得到列表？

简单空格分割会破坏带引号或空格的参数。

例如：

```python
command = 'python show_args.py "Alice Smith"'
print(command.split())
```

它不会像你希望的 shell 语法分析那样自动保留完整名字。

POSIX shell 风格字符串可了解 `shlex.split()`，但它不是所有 Windows shell 与目标程序解析规则的通用替代。

最清楚的方式通常是从一开始就用结构化列表保存程序和参数。

## 第四章：我想把输出保存进变量，怎么做？

### 4.1 使用 `capture_output=True`

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "print('hello')"],
    capture_output=True,
    text=True,
    check=True,
)

print(repr(result.stdout))
print(repr(result.stderr))
```

通常得到：

```text
'hello\n'
''
```

子进程输出被捕获到结果对象，通常不会再自动显示到原终端。

### 4.2 `stdout` 和 `stderr` 为什么分开？

标准输出通常用于正常结果。

标准错误通常用于诊断信息。

一个工具可以正常输出 JSON，同时把进度和错误写到标准错误，避免破坏 JSON 结构。

我们也应该在父进程里保留这种区别。

### 4.3 做一个两路输出实验

```python
import subprocess
import sys

code = "import sys; print('DATA'); print('warning message', file=sys.stderr)"

result = subprocess.run(
    [sys.executable, "-c", code],
    capture_output=True,
    text=True,
    check=True,
)

print("正常输出：", repr(result.stdout))
print("诊断输出：", repr(result.stderr))
```

通常得到：

```text
正常输出： 'DATA\n'
诊断输出： 'warning message\n'
```

代码字符串中分号只是这个受控 Python 片段的写法，实际项目里较长逻辑更适合保存为独立脚本。

### 4.4 `text=True` 是什么作用？

它让相应管道按文本处理。

没有开启文本模式时，捕获结果通常是 `bytes`。

```python
result = subprocess.run(
    [sys.executable, "-c", "print('hello')"],
    capture_output=True,
    check=True,
)
print(repr(result.stdout))
```

可能看到：

```text
b'hello\n'
```

字节模式适合原始二进制内容，文本模式适合约定编码的文字协议。

### 4.5 `.strip()` 可以随便加吗？

`print()` 默认产生末尾换行，所以很多示例使用：

```python
result.stdout.strip()
```

得到更整齐的单行。

但它会删除两端多种空白，可能改掉有意义的缩进或尾部空格。

如果只是移除一个末尾换行，可以根据协议使用更精确的 `removesuffix("\n")`。

要保留原始字节时，就不要先解码再清理。

## 第五章：编码，父进程和子进程需要分别约定

### 5.1 父进程写 `encoding="utf-8"`，子进程就自动输出 UTF-8 吗？

不会。

这个参数主要告诉父侧管道包装器如何编码输入、解码输出。

外部程序输出什么编码，仍取决于它自己的实现和配置。

两端必须匹配。

### 5.2 一个明确约定 UTF-8 的 Python 子进程

```python
import os
import subprocess
import sys

child_env = os.environ.copy()
child_env["PYTHONIOENCODING"] = "utf-8"

result = subprocess.run(
    [sys.executable, "-c", "print('你好，子进程')"],
    env=child_env,
    capture_output=True,
    text=True,
    encoding="utf-8",
    check=True,
)

print(result.stdout, end="")
```

`child_env` 基于当前环境复制，给这次子进程明确配置 Python 标准流编码。

`encoding="utf-8"` 则让父侧按同样编码解读。

这没有修改当前 Python 进程外部的全局环境配置。

### 5.3 如果目标是其他工具呢？

应查看工具自己的编码选项、输出协议或运行约定。

不是所有程序都认识 `PYTHONIOENCODING`，那是 Python 相关配置。

不要把父侧改成 UTF-8 当成给任意工具自动转码。

### 5.4 我不确定编码，能不能先取字节？

可以，保留 `text=False` 的默认二进制模式。

```python
result = subprocess.run(command, capture_output=True, check=True)
payload = result.stdout
```

之后按已知协议解码，或者直接保存原始字节。

如果只是临时诊断，明确选择替代策略可以帮助查看损坏文本；正式数据处理不应为了不报错随意 `errors="ignore"`，因为会丢失内容。

### 5.5 文本模式还会处理换行吗？

相应文本管道会采用通用换行处理，常见行结束形式可能被统一为 `\n`。

因此，你得到的字符串不一定保留线路上的原始换行字节。

如果要校验精确输出文件或二进制协议，使用字节模式更合适。

## 第六章：`check=True`，怎样判断外部工具是否成功？

### 6.1 收到一个结果对象，不代表命令成功

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "import sys; sys.exit(3)"],
    capture_output=True,
    text=True,
)

print(result.returncode)
print("父进程仍继续")
```

通常得到：

```text
3
父进程仍继续
```

默认 `check=False`，非零退出不会仅因状态非零自动抛出 `CalledProcessError`。

你可以自己根据返回码分支。

### 6.2 如果只要非零就当失败呢？

使用：

```python
check=True
```

```python
import subprocess
import sys

try:
    subprocess.run(
        [sys.executable, "-c", "import sys; sys.exit(3)"],
        capture_output=True,
        text=True,
        check=True,
    )
except subprocess.CalledProcessError as error:
    print("退出状态：", error.returncode)
```

通常输出三。

`check` 是父侧的失败处理政策，不会强迫子程序自己成功。

### 6.3 异常里还能看到输出吗？

如果原调用捕获了相应流，通常可以从异常取得：

```python
error.stdout
error.stderr
error.cmd
error.returncode
```

例如外部编译器失败，错误消息可能在 stderr。

只打印“命令失败”会丢失最有用的定位信息。

但输出可能包含路径、令牌或其他敏感信息，也不应无条件把完整命令和全部输出展示给所有人。

### 6.4 非零状态是否永远表示程序损坏？

不一定。

一些工具用不同状态表示“找到匹配”“没有匹配”“发生错误”等多种结果。

你应该按目标工具约定解释。

如果其中某个非零状态是正常业务结果，可以保留 `check=False`，再明确分支处理。

不要为了省一个判断就把所有非零都丢进同一错误类别。

### 6.5 已经拿到结果，还能再检查吗？

可以：

```python
result.check_returncode()
```

它会对非零状态抛出相应异常。

这适合先记录结果，再统一检查的流程。

成功时通常返回 `None`，不要把它赋回 `result` 然后期待仍然拥有结果对象。

### 6.6 返回码和进程 ID 不要混淆

`returncode` 描述结束结果。

`Popen.pid` 描述正在或曾经运行的子进程标识。

在 POSIX 上，负的返回码常表示被相应编号的信号终止。

Windows 的退出码与信号机制不同，不应把所有平台负数都按 Unix 信号表解释。

## 第七章：程序根本没启动，与启动后失败有什么区别？

### 7.1 可执行文件找不到

```python
import subprocess

try:
    subprocess.run(["a_program_that_does_not_exist_12345"], check=True)
except FileNotFoundError:
    print("没有找到要启动的程序")
```

这里连目标程序都没有成功启动。

不会因为设置 `check=True` 就统一变成 `CalledProcessError`。

### 7.2 脚本找不到，但 Python 找到了

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "a_script_that_does_not_exist.py"],
    capture_output=True,
    text=True,
)

print(result.returncode != 0)
print(bool(result.stderr))
```

这次 Python 解释器已经启动。

是它发现要执行的脚本不存在，再按自己的错误规则退出。

因此父侧拿到的是外部程序失败结果，而不是“解释器无法启动”的同一种问题。

### 7.3 权限、工作目录和系统资源也可能阻止启动

可执行权限不足、指定 `cwd` 不存在，或者系统资源不足，都可能在创建进程阶段失败。

这些通常属于 `OSError` 相关异常。

异常处理要分清：启动阶段、运行等待、退出状态，以及后续内容解析。

### 7.4 一个常见处理结构

```python
import subprocess

try:
    result = subprocess.run(
        command,
        capture_output=True,
        text=True,
        check=True,
        timeout=10,
    )
except FileNotFoundError:
    print("目标程序或工作目录不存在")
except subprocess.TimeoutExpired:
    print("等待超时")
except subprocess.CalledProcessError as error:
    print("程序启动后以失败状态退出：", error.returncode)
except OSError as error:
    print("启动或系统操作失败：", error)
```

这里的 `command` 应是前文已经构造的受控参数列表。

不要把一个未定义变量导致的 `NameError` 也当成外部工具错误。

## 第八章：`cwd`——只改变子进程的工作目录

### 8.1 为什么需要单独指定？

假设父程序从：

```text
C:\Users\Alice
```

启动。

子脚本在：

```text
D:\project\child.py
```

它想读取项目里的：

```text
D:\project\data.txt
```

如果子脚本直接 `open("data.txt")`，默认继承的工作目录可能让它去用户目录找文件。

### 8.2 用 `cwd` 明确子侧基准

```python
import subprocess
import sys
from pathlib import Path

project_dir = Path("D:/project")
child_script = project_dir / "child.py"

subprocess.run(
    [sys.executable, str(child_script)],
    cwd=project_dir,
    check=True,
)
```

这个 Windows 原生示例要求相应目录和脚本真实存在。

WSL 中应改用 Linux 可访问的项目路径，例如常见挂载布局中的 `/mnt/d/project`，不会自动翻译 `D:` 盘符。

### 8.3 父进程会跟着切换吗？

不会因为这个参数就自动切换。

`cwd` 是给新进程安排工作目录。

它比先在父程序全局 `os.chdir()` 再启动，更容易控制影响范围。

尤其父程序还有其他相对文件操作或多个线程时，应避免无必要地修改全进程基准。

### 8.4 做一个跨平台临时目录实验

```python
import subprocess
import sys
from pathlib import Path
from tempfile import TemporaryDirectory

before = Path.cwd()

with TemporaryDirectory() as temporary:
    target_dir = Path(temporary)
    result = subprocess.run(
        [sys.executable, "-c", "from pathlib import Path; print(Path.cwd())"],
        cwd=target_dir,
        capture_output=True,
        text=True,
        check=True,
    )
    print(Path(result.stdout.strip()).resolve() == target_dir.resolve())

print(Path.cwd() == before)
```

普通环境下通常两次都为真。

第一次确认子侧目录，第二次确认父侧目录未变。

### 8.5 `cwd` 会不会帮我创建目录？

不会。

如果目录不存在，启动通常失败。

需要创建时应先明确创建，再启动。

同样，子进程的输出路径是否相对于这个目录，取决于它收到的路径和自身实现。

## 第九章：`env`——给子进程一份环境，不改变父环境

### 9.1 默认继承与显式替换

不传 `env`，子进程通常继承父侧环境。

传入 `env` 映射，则使用你提供的环境。

它不是自动把字典中的几项补到原环境上。

所以常见写法是先复制：

```python
import os

child_env = os.environ.copy()
child_env["APP_MODE"] = "practice"
```

再传给启动调用。

### 9.2 为什么不能只传自己关心的一项？

如果写：

```python
env={"APP_MODE": "practice"}
```

你可能同时去掉 PATH、系统目录、语言区域和其他工具依赖的环境信息。

Windows 上一些程序还需要正常的系统环境字段。

如果你确实要构造最小环境，可以这样做，但必须知道目标工具需要什么，不能把它误认为局部更新。

### 9.3 环境变量值通常是字符串

```python
child_env["APP_PORT"] = "8000"
```

不要直接放整数。

子进程读取后再按自己的配置规则转换。

这与命令行参数首先是文本的思想相近：跨进程输入有明确表示约定。

### 9.4 修改子进程的变量会影响父进程吗？

观察：

```python
import os
import subprocess
import sys

child_env = os.environ.copy()
child_env["NOTES_DEMO_VALUE"] = "child-only"

result = subprocess.run(
    [sys.executable, "-c", "import os; print(os.environ['NOTES_DEMO_VALUE'])"],
    env=child_env,
    capture_output=True,
    text=True,
    check=True,
)

print(result.stdout.strip())
print(os.environ.get("NOTES_DEMO_VALUE"))
```

如果父环境原本没有这个名字，后面通常仍是 `None`。

`child_env` 是复制出来的映射，没有修改 `os.environ`。

### 9.5 网络代理需要在这里自动加吗？

不需要。

是否继承当前系统与环境、是否显式设置代理，应该按用户和项目约定决定。

Windows 使用 TUN 路由，并不意味着每个 WSL 子进程都应该再自动加一层 HTTP 或 SOCKS 代理。

真实排错应先检查当前状态，不能用教程中的示例替用户修改全局网络策略。

## 第十章：把文本送给子程序，`input` 怎么用？

### 10.1 一个把输入转成大写的子程序

```python
import subprocess
import sys

code = "import sys; data = sys.stdin.read(); sys.stdout.write(data.upper())"

result = subprocess.run(
    [sys.executable, "-c", code],
    input="hello\npython\n",
    capture_output=True,
    text=True,
    check=True,
)

print(repr(result.stdout))
```

输出通常是：

```text
'HELLO\nPYTHON\n'
```

父进程通过子进程标准输入交付文本。

子进程从 `sys.stdin` 读取，转换后写到 `sys.stdout`。

### 10.2 为什么子程序的 `read()` 会结束？

因为 `run(input=...)` 会管理输入管道，交付指定内容并结束相应输入，让子侧能读到 EOF。

如果你手动管理管道，却一直不关闭写入端，子程序可能持续等待后续内容。

这不是它没有收到前面文字，而是它还不知道输入已经结束。

### 10.3 文本与字节类型必须匹配

`text=True` 或设置相应编码时，`input` 应使用字符串。

二进制模式则传字节：

```python
result = subprocess.run(
    [sys.executable, "-c", "import sys; sys.stdout.buffer.write(sys.stdin.buffer.read())"],
    input=b"\x00\x01ABC",
    capture_output=True,
    check=True,
)

print(result.stdout == b"\x00\x01ABC")
```

这样可以传递零字节和任意原始内容，不经过文本编解码。

### 10.4 `input` 和 `stdin` 能同时指定吗？

常规 `run()` 调用中，`input` 会安排相应输入管道，不能再同时给一个冲突的 `stdin` 来源。

如果你已经有一个输入文件，传 `stdin=file_object`。

如果你有一段内存内容，传 `input=...`。

先明确数据来自哪里，再选接口。

## 第十一章：直接连接文件，避免把全部数据放进内存

### 11.1 把文件作为标准输入

```python
import subprocess
import sys
from pathlib import Path

input_path = Path("input.bin")
output_path = Path("output.bin")

with input_path.open("rb") as source:
    with output_path.open("xb") as target:
        subprocess.run(
            [sys.executable, "-c", "import sys; sys.stdout.buffer.write(sys.stdin.buffer.read())"],
            stdin=source,
            stdout=target,
            check=True,
        )
```

这段父侧不会先把输入内容整体 `read()` 到变量。

但示例子程序自己调用了无长度 `read()`，仍然整体读取，所以不能把这份子代码称为端到端恒定内存处理。

真正的大文件转换工具也应在子侧按块消费。

### 11.2 为什么文件用二进制模式？

子进程通过文件描述符连接底层内容。

二进制打开避免父侧文本层的编码和换行规则造成误解。

子程序输出什么字节，文件接收相应字节。

设置父侧 `text=True` 不会把一个直接重定向到磁盘描述符的外部输出自动转换成 UTF-8。

### 11.3 同时保存诊断日志

```python
with open("output.bin", "xb") as target:
    with open("child-errors.log", "wb") as errors:
        result = subprocess.run(command, stdout=target, stderr=errors, check=False)
```

这里 `command` 是已经准备好的目标程序参数列表。

正常数据与诊断分开保存。

执行失败后，输出文件可能已经存在且只有部分内容，`check=True` 也不会自动回滚。

### 11.4 `capture_output=True` 还能和 `stdout=target` 一起写吗？

不能同时指定冲突的捕获方式。

`capture_output=True` 相当于为 stdout 和 stderr 安排管道。

如果要一边捕获一边写文件，需要你自己读取并转发，或只对其中一个流单独设置 `PIPE`。

### 11.5 父程序先往同一个文件对象写了内容呢？

如果父侧写入还停留在缓冲里，就把描述符交给子进程，观察到的顺序可能不符合预期。

需要共享这类输出时，应明确刷新和位置安排。

更简单的设计通常是让一个写入者负责一个结果文件，减少多个层次共同操作同一偏移的混乱。

## 第十二章：合并标准输出和标准错误

### 12.1 什么情况下想合并？

执行一个构建工具时，你可能只想保存一份面向人的完整日志。

可以让 stderr 进入 stdout：

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "import sys; print('out', flush=True); print('err', file=sys.stderr)"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True,
    check=True,
)

print(repr(result.stdout))
print(result.stderr)
```

合并内容在 `stdout` 中，`stderr` 通常是 `None`，因为没有独立捕获这一路。

### 12.2 合并之后还能可靠区分来源吗？

不能直接从同一个字符串恢复原来每个片段属于哪一路。

如果你需要机器读取 stdout 中的 JSON，就不要把诊断合并进去。

日志方便性与结构化输出需求，应根据任务选择。

### 12.3 能保证两路消息严格按代码行顺序排列吗？

缓冲、调度和不同写入大小都可能影响观察结果。

子侧 stdout 连接管道时的缓冲行为，也可能与连接终端时不同。

上例对第一条输出使用 `flush=True`，是为了让简单实验更明确。

对于复杂并发程序，不能仅凭合并流就声称获得完整的全局事件顺序。

## 第十三章：我不想让子进程读取键盘，也不想显示输出

### 13.1 `DEVNULL` 的含义

`subprocess.DEVNULL` 表示系统的空设备。

用作标准输入，通常让读取立即遇到结束。

用作输出，丢弃相应内容。

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "import sys; print(repr(sys.stdin.read()))"],
    stdin=subprocess.DEVNULL,
    stdout=subprocess.DEVNULL,
    stderr=subprocess.DEVNULL,
    check=True,
)
```

这个例子会执行，但你看不到那条打印。

### 13.2 为什么自动化任务要注意标准输入？

如果外部工具以为自己可以询问确认，而父程序又没有处理交互，它可能一直等待键盘。

对于应当非交互运行的工具，优先使用工具自身明确的非交互选项，并考虑 `stdin=DEVNULL`。

丢弃输入并不保证所有程序都会正常继续，有些程序会因 EOF 退出，这是需要测试的行为。

### 13.3 所有输出都丢掉，失败怎么排查？

如果任务重要，至少保存必要诊断或在失败时显示捕获信息。

不要为了终端清爽，长期把唯一错误证据送入空设备。

可以正常时安静，失败时报告退出状态和适度裁剪的 stderr。

## 第十四章：超时，`run()` 和手动 `Popen` 要分开理解

### 14.1 `run(timeout=...)`

```python
import subprocess
import sys

try:
    subprocess.run(
        [sys.executable, "-c", "import time; time.sleep(2)"],
        timeout=0.2,
        check=True,
    )
except subprocess.TimeoutExpired:
    print("等待超过约定时间")
```

`run()` 在相应等待超时时会终止所管理的子进程并等待清理，再抛出超时异常。

进程创建的某些底层步骤在部分平台不能立即中断，所以这不是任意环境下精确到毫秒的完整总时限。

### 14.2 超时异常里有什么？

常用资料包括：

```python
error.cmd
error.timeout
error.stdout
error.stderr
```

是否有输出，取决于是否捕获、子侧是否已经输出并刷新等情况。

部分输出属性在超时路径中可能仍是字节，即使原调用开启文本模式，也不应无条件当字符串拼接。[官方异常说明](https://docs.python.org/3.14/library/subprocess.html#subprocess.TimeoutExpired)

### 14.3 超时意味着子程序完全没执行吗？

不是。

它可能已经创建文件、提交请求或写入一部分结果。

终止进程不会自动撤销这些外部副作用。

因此，重试一个超时任务之前，要明确任务是否可重复，以及如何识别前一次是否已完成。

### 14.4 如果子程序又启动了孙进程呢？

直接管理的子进程被停止，不代表所有后代进程都自动按你的业务需求一起停止。

它们可能仍持有管道或继续运行。

需要管理进程树时，要结合平台的进程组、会话、作业对象或其他机制，后面会说明边界。

不要把一行 `timeout=10` 解释成跨平台的完整作业树控制系统。

## 第十五章：`Popen`——如果我不想一直停在 run() 这一行呢？

### 15.1 先认识“启动”和“等待”的分离

假设外部程序需要两秒处理文件。

父程序不想立即等待结果，而是先记录任务编号、准备另一些输入，再回来查看状态。

这时可以直接使用 `Popen`。

```python
import subprocess
import sys

process = subprocess.Popen(
    [sys.executable, "-c", "import time; time.sleep(0.2)"],
    stdin=subprocess.DEVNULL,
)

print("父进程已经取得进程对象")
returncode = process.wait()
print(returncode)
```

正常输出：

```text
父进程已经取得进程对象
0
```

`Popen(...)` 完成创建过程，返回一个仍可能运行的进程对象。

它不会像 `run()` 那样天然把“等到结束，再返回结果”包成一次调用。

`process.wait()` 才明确等待子进程结束。

两个动作分开以后，你获得了控制机会，也承担了更多清理责任。

### 15.2 Popen 对象和 CompletedProcess 对象有何区别？

`run()` 返回的 `CompletedProcess` 描述一次已经完成的运行。

`Popen` 返回的对象代表由你继续管理的进程。

你可以读取它的管道、查询状态、等待、发送信号。

但它没有 `run(check=True)` 那样直接接受 `check` 的接口。

如果写 `Popen(..., check=True)`，不是“更严格”，而是参数用错。

你应等待以后自行判断返回码，或在不需要这些控制时直接使用 `run()`。

### 15.3 `pid` 是什么？

```python
import subprocess
import sys

process = subprocess.Popen(
    [sys.executable, "-c", "pass"],
    stdin=subprocess.DEVNULL,
)
try:
    print(isinstance(process.pid, int))
    print(process.wait())
finally:
    if process.poll() is None:
        process.kill()
        process.wait()
```

常见输出：

```text
True
0
```

这里避免把某个具体 PID 写成固定答案，因为它由操作系统分配。

PID 可以出现在诊断日志中，帮助关联系统进程。

它不应该被当成永久任务编号；进程退出以后，系统将来可能复用同一个数字。

通过别处保存的旧 PID 操作进程，要非常谨慎。

### 15.4 子进程还在运行时，`returncode` 是多少？

初始通常是 `None`，表示这个对象还没有记录到退出状态。

`poll()` 会检查状态并更新 `returncode`。

如果进程仍在运行，`poll()` 返回 `None`。

如果已经退出，它返回退出码。

这里 `0` 也是一个有效退出码，不能用 `if not process.poll()` 表示“仍在运行”。

因为 `None` 和 `0` 在布尔判断中都为假。

应明确写：

```python
if process.poll() is None:
    print("当前还在运行")
else:
    print("已经结束：", process.returncode)
```

这是依赖已经创建的 `process` 的状态检查片段。

由于调度时间不同，某次查询可能已结束，也可能尚未结束，不要把结果当成固定时序保证。

### 15.5 我能一直 while 轮询吗？

如果没有任何等待，循环会无意义地占用 CPU。

如果只想等待结束，直接 `wait()`。

如果必须周期更新界面或检查取消标记，才安排合理的轮询间隔，或者使用适合该应用的事件机制。

而且轮询状态不等于读取管道。

即使你每秒检查一次 `poll()`，没人消费的 stdout 仍可能填满，后面会说明。

### 15.6 父程序抛异常，子进程会自动消失吗？

不能依赖这个假设。

你需要给自己创建的进程安排生命周期。

下面只适用于自己直接创建、没有后代任务的简单子进程：

```python
import subprocess
import sys

process = subprocess.Popen(
    [sys.executable, "-c", "import time; time.sleep(2)"],
    stdin=subprocess.DEVNULL,
)
try:
    raise RuntimeError("模拟父侧处理失败")
except RuntimeError:
    print("父侧失败，准备清理子进程")
finally:
    if process.poll() is None:
        process.kill()
    process.wait()
```

输出：

```text
父侧失败，准备清理子进程
```

先停止仍在运行的直接子进程，再等待回收。

实际工具是否应该立即强制停止，取决于它能否正常结束；这里为了展示清理结构才使用 `kill()`。

终止不替你撤销文件写入，也不提供进程树整体清理。

## 第十六章：`communicate()`——输入、输出和等待为什么要一起安排？

### 16.1 先看一个完整的文本往返

让子程序读取标准输入，转成大写后输出。

```python
import subprocess
import sys

process = subprocess.Popen(
    [sys.executable, "-c", "import sys; sys.stdout.write(sys.stdin.read().upper())"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
    encoding="utf-8",
)

stdout, stderr = process.communicate("hello\n", timeout=5)
print(repr(stdout))
print(repr(stderr))
print(process.returncode)
```

输出：

```text
'HELLO\n'
''
0
```

这个例子使用 ASCII 文本，子侧默认编码差异不会影响这些字符。

真实中文协议仍应像前面那样给 Python 子程序设置明确编码，或使用 `-X utf8` 等已知策略。

### 16.2 communicate 实际帮我做了什么？

它向标准输入发送提供的数据。

输入发送结束后，安排关闭写端，让子进程能读到 EOF。

同时消费被捕获的标准输出和标准错误，等待进程结束。

最后返回 `(stdout_data, stderr_data)`。

你不用手动决定“先写全部输入还是先读取全部输出”。

这正是它比随意几次 `.write()`、`.read()` 更稳妥的地方。

### 16.3 EOF 为什么很重要？

子程序的：

```python
sys.stdin.read()
```

意思是读取直到输入结束。

假设父侧只写了 `hello`，但一直保持标准输入管道打开，子侧不知道后面是否还有内容。

它可能继续等。

父侧又等它输出，就形成双方等待。

发送数据、刷新缓冲、关闭输入写端，是三个相关但不同的动作。

`flush()` 只能让已缓冲数据向下送，不表示“我永远不会再发送”。

### 16.4 stdout 和 stderr 一定都是字符串吗？

取决于相应流是否设置为 PIPE，以及是否开启文本模式。

二进制模式通常返回 bytes。

没有捕获的那一路通常是 `None`。

如果设置 `stderr=STDOUT`，诊断合并进入 stdout，返回元组第二项通常为 `None`。

因此写通用包装时，先确定自己的流配置，别对所有结果无条件 `.strip()`。

### 16.5 Popen 的 communicate 不会自动检查非零状态

下面程序会正常完成通信，但退出码为 `4`：

```python
import subprocess
import sys

process = subprocess.Popen(
    [sys.executable, "-c", "import sys; print('bad input', file=sys.stderr); sys.exit(4)"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
)
stdout, stderr = process.communicate(timeout=5)

if process.returncode != 0:
    print("外部程序失败：", process.returncode)
    print(stderr.rstrip("\n"))
```

输出：

```text
外部程序失败： 4
bad input
```

通信成功只表示你得到了输出并观察到结束。

业务成功仍要按返回码和结果协议判断。

### 16.6 能用 communicate 做长时间的聊天吗？

它主要面向一次完整通信，会处理输入结束并等待进程完成。

不适合调用一次发一条问题，再调用一次等一个回答，循环保持会话。

长时间交互需要明确消息边界、刷新策略、并发读写和取消协议。

如果目标工具只提供交互式终端界面，还可能涉及伪终端，而普通 PIPE 不等于终端。

先看目标工具是否有非交互模式、JSON 模式或批处理接口，通常会省很多复杂性。

## 第十七章：管道为什么会死锁？请把“两边都在等”画出来

### 17.1 PIPE 不是无限大字符串

操作系统为管道提供有限缓冲。

子程序一直写，父程序一直不读，缓冲满后，子程序的写入可能阻塞。

这不是数据消失了，而是写入要等待读取释放空间。

实际容量依赖平台和配置，不要记住某个网上的数字就认为安全。

### 17.2 先 wait 再 read，为什么有风险？

看这段概念示范，不要拿无限输出程序实际运行：

```text
父进程：启动子进程，并把 stdout 接到 PIPE。
父进程：wait()，等待子进程退出。
子进程：不断写 stdout，直到管道缓冲满。
子进程：等待父进程读走数据后才能继续写。
```

父侧等结束。

子侧等读走。

如果子侧只有输出完成后才退出，两边就卡住了。

所以 `wait()` 本身不能替你排空管道。[官方 wait 警告](https://docs.python.org/3.14/library/subprocess.html#subprocess.Popen.wait)

### 17.3 我先 read stdout，再 read stderr，行不行？

仍有风险。

父侧等待 stdout 读到 EOF。

子侧可能正在向 stderr 写大量诊断，stderr 满了以后无法继续执行，也就无法关闭 stdout。

于是父侧读不到 EOF，子侧也前进不了。

分开两路并不意味着可以任意按先后整体读取。

### 17.4 小输出用什么？

使用 `run(capture_output=True)`，或者 `Popen.communicate()`。

它们安排两路通信，避免上述常见的读写顺序死锁。

但它们仍会把捕获内容保存在内存。

没有管道死锁，不等于没有内存风险。

### 17.5 大输出用什么？

如果只要保存，把 stdout、stderr 直接指向文件。

如果要边看边处理，要持续消费相应流；两路分开时，需要兼顾两路读取。

如果诊断也可以混进同一份人类日志，合并流后只有一路需要消费。

如果 stdout 是机器协议，诊断不能混进去，可以把 stderr 直接写日志文件，父侧只逐行读取 stdout。

这些方式各有取舍，要从实际输出需求选择。

### 17.6 给管道加大容量能根治吗？

不能。

`pipesize` 在支持的平台上可以影响管道大小，Python 文档目前主要描述 Linux 支持。

但有限缓冲加大以后仍然有限。

一个持续输出的子进程终究可能写满它。

解决方法是正确安排消费和生命周期，而不是把缓冲当无限空间。

### 17.7 我如何安全观察大量双路输出？

用一个有限输出的受控子程序，分别产生十万字节级别的数据，再用 `communicate()` 验证长度。

```python
import subprocess
import sys

code = "import sys; sys.stdout.buffer.write(b'A' * 100000); sys.stderr.buffer.write(b'B' * 100000)"
process = subprocess.Popen(
    [sys.executable, "-c", code],
    stdin=subprocess.DEVNULL,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)
stdout, stderr = process.communicate(timeout=5)
print(len(stdout))
print(len(stderr))
print(process.returncode)
```

输出：

```text
100000
100000
0
```

这个例子不需要知道管道容量，也没有故意运行会永远卡住的错误版本。

它验证的是所选通信方式能处理本次有界双路输出。

不能因此说任意无限输出都适合捕获到内存。

## 第十八章：手动管理超时，先停止，再排空，最后等待

### 18.1 Popen.communicate 超时，不会自动杀掉进程

这点与 `run(timeout=...)` 很容易混淆。

`communicate(timeout=...)` 超时会抛 `TimeoutExpired`，但它本身不会负责终止子进程。

你需要决定是否继续等待，或者结束它。

下面展示一个直接子进程的完整清理结构：

```python
import subprocess
import sys

process = subprocess.Popen(
    [sys.executable, "-u", "-c", "import time; print('started', flush=True); time.sleep(2)"],
    stdin=subprocess.DEVNULL,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
)

try:
    stdout, stderr = process.communicate(timeout=0.2)
except subprocess.TimeoutExpired:
    process.kill()
    stdout, stderr = process.communicate()
    print("已超时并完成直接子进程清理")
    print("started" in stdout)
```

正常资源条件下预期输出：

```text
已超时并完成直接子进程清理
True
```

极端繁忙环境中，子程序可能还没获得执行机会就超时，因此部分输出不应作为超时处理正确性的绝对前提。

### 18.2 为什么 kill 以后还要 communicate？

发出终止操作，不等于你已经回收进程并读完残留输出。

第二次 `communicate()` 会继续完成管道处理和等待。

不要只 `kill()` 然后抛出自己的异常，把进程状态和文件句柄留给不确定的后续清理。

### 18.3 第一次的部分输出要不要拼接第二次？

不要机械地拼接 `error.stdout + stdout`。

Python 文档说明，超时以后重试通信不会丢失已有输出。

最后成功完成的 `communicate()` 结果包含它保存和继续收集的数据；再拼接可能重复。

而超时异常的部分输出还可能是 bytes，与文本最终结果类型不同。

所以简单包装通常直接使用最后完成通信取得的结果。

### 18.4 `wait(timeout=...)` 超时呢？

同样不会自动终止进程。

如果你没有捕获管道，可按自己的停止政策终止并再等待。

如果有未消费管道，就优先处理通信，不能只顾等待状态。

`wait()` 可以重复调用，进程已经结束后仍可取得退出状态；但这不会让输出重新产生。

### 18.5 terminate 和 kill 有什么不同？

在 POSIX 上，`terminate()` 通常发送 SIGTERM，`kill()` 通常发送 SIGKILL。

程序可能处理 SIGTERM，安排清理；SIGKILL 不能被普通进程捕获并执行自己的清理逻辑。

在 Windows 上，`terminate()` 使用 TerminateProcess，`kill()` 是它的别名，不能照搬“一个温和，一个强硬”的 Unix 理解。

若工具要求温和关闭，最好优先使用它自己支持的退出命令或控制协议。

### 18.6 可以先请求退出，等一会，再强制停止吗？

可以安排有限的宽限期。

下面片段仍只管理直接子进程，不捕获无限输出，不保证终止所有后代：

```python
import subprocess


def stop_direct_child(process, grace=1.0):
    if process.poll() is not None:
        return process.wait()

    process.terminate()
    try:
        return process.wait(timeout=grace)
    except subprocess.TimeoutExpired:
        process.kill()
        return process.wait()
```

这个辅助函数适合输出已经重定向到文件、或者没有需要排空的管道的情况。

如果输出接 PIPE，应把停止和 `communicate()` 配合，而不是调用它后忘记读取。

多线程程序里还可能有别的线程同时管理同一进程，需要额外同步，不能让两个清理者任意竞争。

## 第十九章：边产生、边读取——为什么 for line 仍然可能等很久？

### 19.1 一个可观察的逐行输出例子

```python
import subprocess
import sys

code = "import time; [print(f'step {i}', flush=True) or time.sleep(0.05) for i in range(3)]"

process = subprocess.Popen(
    [sys.executable, "-u", "-c", code],
    stdin=subprocess.DEVNULL,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True,
    encoding="utf-8",
)

try:
    for line in process.stdout:
        print("收到：", line.rstrip("\n"))
    returncode = process.wait()
    if returncode:
        raise subprocess.CalledProcessError(returncode, process.args)
finally:
    if process.poll() is None:
        process.kill()
        process.wait()
    process.stdout.close()
```

输出：

```text
收到： step 0
收到： step 1
收到： step 2
```

这里 stderr 合并到同一条流，所以不存在另一条未消费的诊断管道。

### 19.2 每行都及时出现，靠的是哪一边？

父侧逐行读，只表示愿意消费完整行。

子侧还必须把数据真正写出来。

Python 子程序连接管道时，stdout 缓冲策略可能不同于终端。

`-u` 请求 Python 标准流无缓冲相关行为，`print(..., flush=True)` 也明确刷新。

两者在这个短例子中有重叠，是为了让初次观察更容易。

对非 Python 工具，`-u` 不一定存在，必须查它自己的选项。

### 19.3 Popen(bufsize=1) 能强迫外部工具及时打印吗？

不能。

`bufsize` 主要影响父侧创建的 Python 管道文件对象。

它不能远程改变另一个程序内部的 C stdio、日志系统或业务缓存。

行缓冲设置也与文本模式有关，不能把 `bufsize=1` 当成所有二进制流的万能实时开关。

先区分子程序还没写、操作系统管道有数据、父侧包装器还没返回完整行这几个位置。

### 19.4 如果子程序只输出进度百分比，不换行呢？

很多终端程序用 `\r` 重写同一行，或者输出几个字符就等下一次进度。

`for line` 可能要等到行结束或 EOF 才返回。

它不是“每写一个字符就回调一次”。

这种场景要按块读取或使用更合适的事件接口，并设计自己的消息切分。

如果能够控制子工具，输出一行一条 JSON 或一行一条日志，会大幅简化父侧处理。

### 19.5 读取过程中想严格设置总超时，能直接加一个 timeout 吗？

普通文件对象的 `readline()` 没有和 Popen 一样的通用 timeout 参数。

如果主线程阻塞在读行，主线程就没机会检查截止时间。

所以“在 for 循环里面比较时间”不能保证没有新行时也及时停止。

要实现有界等待，可以让专门线程消费管道，主线程通过队列带超时地收取消息；也可以采用 asyncio 等异步接口。

POSIX 的选择器能监视很多文件描述符，但 Windows 上普通管道的支持方式不同，不能拿一个 selectors 例子就宣称跨平台通用。

### 19.6 不需要实时计算时，写文件通常更简单

有些任务只是希望留下日志，不一定要让父程序每一秒知道日志内容。

这种情况下直接传文件对象，`run(timeout=...)`，最后判断状态，往往更可靠。

不要因为 Popen 更底层就认为所有任务都应该使用它。

只有确实要管理生命周期或实时协议时，再承担相应复杂度。

## 第二十章：两路输出都要保留来源，怎样读取而不堵住？

### 20.1 先选一个有限、容易理解的方案

如果 stdout 是逐行结果、stderr 是诊断，我们可以为每条管道各安排一个读取线程。

主线程只负责收队列消息和检查截止时间。

下面先展示基础版本，子程序只输出少量完整行，队列不设置容量。

它不是无限日志服务；无限生产、消费太慢会让无界队列增长。

```python
import queue
import subprocess
import sys
import threading
import time


def relay_lines(stream, label, events):
    try:
        for line in stream:
            events.put((label, line))
    finally:
        stream.close()
        events.put((label, None))


def read_demo_streams():
    code = (
        "import sys,time; "
        "print('result 1',flush=True); "
        "print('warning 1',file=sys.stderr,flush=True); "
        "time.sleep(0.1); print('result 2',flush=True)"
    )
    process = subprocess.Popen(
        [sys.executable, "-u", "-c", code],
        stdin=subprocess.DEVNULL,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        encoding="utf-8",
    )
    events = queue.Queue()
    threads = [
        threading.Thread(target=relay_lines, args=(process.stdout, "out", events)),
        threading.Thread(target=relay_lines, args=(process.stderr, "err", events)),
    ]
    for thread in threads:
        thread.start()

    ended = set()
    deadline = time.monotonic() + 5
    try:
        while len(ended) < 2:
            remaining = deadline - time.monotonic()
            if remaining <= 0:
                raise subprocess.TimeoutExpired(process.args, 5)
            try:
                label, line = events.get(timeout=min(remaining, 0.1))
            except queue.Empty:
                continue
            if line is None:
                ended.add(label)
            else:
                print(f"{label}: {line.rstrip()}")

        remaining = deadline - time.monotonic()
        if remaining <= 0:
            raise subprocess.TimeoutExpired(process.args, 5)
        returncode = process.wait(timeout=remaining)
        if returncode:
            raise subprocess.CalledProcessError(returncode, process.args)
    finally:
        if process.poll() is None:
            process.kill()
        process.wait()
        for thread in threads:
            thread.join()


read_demo_streams()
```

你会看到三条带来源的信息，例如：

```text
out: result 1
err: warning 1
out: result 2
```

两路相互交错的具体先后不保证一致。

### 20.2 为什么用 None 表示流结束？

这里真正的消息是字符串，而 `None` 是独立的结束标记。

空字符串与空白行要保留它们自己的含义，不能随便用来代表结束。

每条读取线程都在 `finally` 里发一个结束事件，让主线程知道这一条流不再有后续消息。

如果要把读线程发生的解码错误也明确交给主线程，应增加一个独立错误事件；本例只处理受控 UTF-8 小输出，不把它包装成面向任意输入的通用框架。

### 20.3 为什么 deadline 用 monotonic？

`time.monotonic()` 用于测量经过的时间，不受普通系统时钟校准那样的日历时间变化影响。

所有等待都根据同一个截止点计算剩余时间，才能接近“整个阶段最多等五秒”。

如果每收一条消息就重新给五秒，一个持续慢慢输出的子进程可能一直续命。

这与“最多五秒没有新消息”的空闲超时是两种不同政策。

### 20.4 为什么 finally 里 join 线程？

进程结束之后，读线程通常会从管道读到 EOF 并退出。

等待它们结束，才能确认父侧读资源也被清理。

但如果子程序启动了仍持有管道写端的后代，直接子进程结束不一定产生 EOF，线程可能继续等。

所以这个例子的适用条件明确限制为没有长期后代的受控子程序。

生产级作业管理要同时设计后代进程关闭和读取线程退出，不能在示例的最后加个 daemon=True 就宣称解决了资源生命周期。

### 20.5 队列满了怎么办？

如果改成有界队列，读取线程可能在 `put()` 上等待主线程。

主线程再等待线程结束而不消费队列，就可能制造新的等待环。

因此要先决定允许丢日志、背压、磁盘落地还是限流。

这些选择直接决定工具在负载过高时的行为。

对于一般个人脚本，标准输出结构化且有限、诊断直接落文件，通常比通用双流实时转发更容易维护。

## 第二十一章：`shell=True`——什么时候才真的需要命令解释器？

### 21.1 先看一个经常让人困惑的例子

你在终端里写：

```bash
some_tool *.txt
```

某些 shell 会先把通配符展开为一组文件名，再启动工具。

但是在 Python 中：

```python
subprocess.run(["some_tool", "*.txt"])
```

默认不会由 subprocess 替你做 shell 文件名展开。

目标程序收到的可能就是字面上的 `*.txt`，除非目标程序自己支持展开。

这不是参数列表失效，而是你没有经过提供这一能力的命令解释器。

### 21.2 想列出匹配文件，需要启动 shell 吗？

通常不需要。

Python 已经有 `Path.glob()`。

```python
from pathlib import Path

files = sorted(Path("practice-input").glob("*.txt"))
command = ["some_tool", *[str(path) for path in files]]
```

这是构造参数的片段，要求练习目录和目标工具实际存在。

同时要考虑空匹配时应该报错、跳过，还是允许工具零输入运行。

显式写出空匹配规则，比依赖不同 shell 的默认行为更容易控制。

### 21.3 `cd`、`dir`、`export` 都是普通可执行文件吗？

不是所有命令环境里的词都对应一个独立程序。

Windows cmd 的 `dir` 常作为内建命令工作。

Bash 的 `cd`、`export` 直接改变当前 shell 的状态。

所以直接 `subprocess.run(["cd", "somewhere"])` 不是跨平台的切换目录方法。

让子程序在某个目录运行，用 `cwd`。

给子程序设置变量，用 `env`。

让父 Python 自己切换目录才考虑 `os.chdir()`，但要理解它影响全进程。

### 21.4 子 shell 里 cd 成功，父 Python 会跟着改变吗？

不会。

它改变的是那次子 shell 的状态。

子 shell 退出后，父进程工作目录仍按父侧自己的状态存在。

如果你需要启动一个工具，先在子 shell 里 cd，再启动工具，可以在 shell 语言里串联；但多数情况下直接 `cwd=...` 更容易明确路径。

### 21.5 POSIX 的 shell=True 默认是 Bash 吗？

通常默认使用 `/bin/sh`，不保证是 Bash。

某些 Bash 专有语法在 `/bin/sh` 中会失败。

如果确实依赖 Bash，明确启动 Bash，并接受平台依赖。

不要把在个人交互终端能运行的一段命令，自动当成所有 Linux `/bin/sh` 都支持。

Windows 默认 shell 也不是自动变成 PowerShell。

### 21.6 我可以直接显式调用某个 shell 吗？

可以，例如 POSIX 平台的受控教学例子：

```python
import os
import subprocess

if os.name == "posix":
    result = subprocess.run(
        ["/bin/sh", "-c", "printf '%s\\n' hello"],
        capture_output=True,
        text=True,
        check=True,
    )
    print(result.stdout, end="")
```

在具有 `/bin/sh` 的通常 POSIX 环境输出：

```text
hello
```

这里虽然没有设置 `shell=True`，但你主动启动了 shell。

因此安全边界仍取决于给 `-c` 的脚本文本是否包含不可信内容。

“代码里没出现 shell=True”不等于绝对没有 shell 解释。

### 21.7 Windows 的批处理文件有什么特别之处？

`.bat`、`.cmd` 在某些 Windows 启动路径中可能由系统 shell 处理，即使 Python 调用使用默认 `shell=False`。

这会让其中的参数被按 shell 规则解释，不能简单套用“列表永远不会遇到 shell 元字符”的口号。

Python 官方安全说明专门讨论了这种边界；确实需要调用批处理且参数来自外部时，应按目标 Python 版本和官方说明验证转义与启动方式。

如果可以选择，优先调用真正的可执行程序或自己控制的 Python 脚本，避免把不可信输入带进复杂批处理链。[官方安全考虑](https://docs.python.org/3.14/library/subprocess.html#security-considerations)

## 第二十二章：参数安全不止 shell 注入，还要看工具自己的语义

### 22.1 一个分号只是普通参数时

下面子程序只打印收到的参数。

```python
import subprocess
import sys

value = "hello; echo SHOULD_NOT_RUN"
result = subprocess.run(
    [sys.executable, "-c", "import sys; print(repr(sys.argv[1]))", value],
    capture_output=True,
    text=True,
    check=True,
)
print(result.stdout, end="")
```

输出：

```text
'hello; echo SHOULD_NOT_RUN'
```

分号没有变成第二条 shell 命令。

它被目标 Python 当成一个字符串参数。

这里的 `-c` 程序代码固定，外部值通过后面的参数传入，而不是拼到代码字符串内部。

### 22.2 如果我把 value 拼进 Python -c 呢？

那么你可能制造 Python 代码注入。

shell 没参与，也不代表把外部输入当成另一门语言的代码是安全的。

类似情况还包括 SQL、正则表达式、模板和目标工具自己的表达式参数。

核心是：把数据送到解释器的“数据位置”，而不是“代码位置”。

对 Python 子脚本，普通值放在 argv、stdin 或受控数据文件里。

### 22.3 文件名以短横线开头，有什么问题？

某些工具可能把 `-something` 当作选项，而不是文件名。

这属于选项注入或参数语义问题，参数列表不会替你判断。

很多 POSIX 风格工具支持 `--` 表示选项结束，但不是所有程序都支持。

使用之前应查目标工具文档。

另一种方法是传入明确的完整路径，使普通文件路径不以选项前缀开头，但同样应结合目标程序规则。

### 22.4 找到的工具一定是你想运行的那个吗？

如果使用裸程序名，搜索位置可能受到 PATH、工作目录、平台规则等影响。

受控任务可以使用已知绝对路径，或通过 `shutil.which()` 找到并记录实际位置。

```python
import shutil

path = shutil.which("git")
if path is None:
    print("当前搜索路径中没有找到 git")
else:
    print(path)
```

输出因机器而异。

`which()` 帮助按环境查找，不保证该可执行文件来源可信，也不安装缺失工具。

不要把任意下载到当前目录的同名文件当成你原本的系统工具。

### 22.5 日志里可以直接打印完整 command 吗？

如果参数包含访问令牌或口令，完整日志就会暴露它们。

应尽量选择目标工具提供的安全输入方式，并对日志脱敏。

环境变量也不是天然秘密保险箱，仍取决于操作系统、监控和进程访问权限。

文档里的普通文件名示例不代表所有真实参数都适合公开。

### 22.6 `shlex.quote()` 可以跨平台保护所有字符串吗？

它面向 Unix shell 的单个词引用规则，不是 Windows cmd 或 PowerShell 的通用引用器。

如果根本不需要 shell，直接保留参数列表通常更清楚。

如果需要显示一条供人参考的 POSIX 命令，可以用适用的 shlex 工具帮助展示，但展示字符串不应再反过来成为唯一真实参数来源。

Python 的 `subprocess.list2cmdline()` 又针对 Windows 一类命令行规则，不等于任意 shell 安全转义。

参数构造、日志展示和 shell 脚本编码，应当各自有明确用途。

## 第二十三章：不经过 shell，把两个程序接成管道

### 23.1 一个程序产生内容，另一个程序转换内容

你在终端中可能见过：

```text
producer | consumer
```

竖线表示把前一个程序的标准输出连接到后一个程序的标准输入。

Python 可以直接连接文件描述符，不必先把整份中间结果保存在父进程内存里。

但你需要同时管理两个进程，而不是只看最后一个的状态。

### 23.2 一个有限、完整的例子

生产者输出两行文字。

消费者逐行转成大写。

```python
import subprocess
import sys

producer = subprocess.Popen(
    [sys.executable, "-c", "print('alpha'); print('beta')"],
    stdin=subprocess.DEVNULL,
    stdout=subprocess.PIPE,
    stderr=subprocess.DEVNULL,
)
consumer = None

try:
    consumer = subprocess.Popen(
        [sys.executable, "-c", "import sys; [sys.stdout.write(line.upper()) for line in sys.stdin]"],
        stdin=producer.stdout,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
    )
    producer.stdout.close()
    output, errors = consumer.communicate(timeout=5)
    producer_code = producer.wait(timeout=5)

    if producer_code != 0:
        raise subprocess.CalledProcessError(producer_code, producer.args)
    if consumer.returncode != 0:
        raise subprocess.CalledProcessError(
            consumer.returncode,
            consumer.args,
            output=output,
            stderr=errors,
        )
    print(output.decode("ascii"), end="")
finally:
    if consumer is not None:
        if consumer.poll() is None:
            consumer.kill()
        consumer.wait()
        if consumer.stdout is not None:
            consumer.stdout.close()
        if consumer.stderr is not None:
            consumer.stderr.close()
    if producer.poll() is None:
        producer.kill()
    producer.wait()
    producer.stdout.close()
```

输出：

```text
ALPHA
BETA
```

两个受控子程序都只处理 ASCII，因此这里明确用 ASCII 解码结果。

`stderr=DEVNULL` 只是为了让最小生产者示例少一条待管理管道；真实生产者的诊断建议写独立文件，不能无理由丢弃。

### 23.3 为什么父进程要关闭 producer.stdout？

消费者启动时已经继承了相应读取端。

父进程自己不打算读这条管道，应该关闭父侧多余的副本。

否则，当消费者提前退出，父侧还保留着读端，会影响生产者观察到读端关闭的时机。

在 POSIX 中，关闭多余读端有助于生产者按正常管道语义收到 SIGPIPE 或相关写入错误。

这不是把消费者的输入也关掉；它关闭的是父进程持有的那一份句柄。

同一个底层资源在不同进程里可以有各自持有的引用，这一点值得慢慢理解。[官方替代 shell 管道示例](https://docs.python.org/3.14/library/subprocess.html#replacing-shell-pipeline)

### 23.4 只检查 consumer.returncode 够吗？

不够。

生产者可能中途失败，只输出了部分数据。

消费者把这部分内容正常转换，最后成功退出。

如果你只检查消费者，就会把不完整数据当成完整任务。

因此要分别收集生产者和消费者状态，再按业务规则判断整条管道。

同样，真实工具可能把“下游主动提早停止”视为正常，而上游观察到断管道；这需要你明确约定，不是所有非零上游状态都能直接忽略。

### 23.5 为什么这个例子不用一份全局 timeout？

这里为了先看懂连接关系，分别给两个等待步骤设了五秒。

它不是严格“总共最多五秒”的接口。

如果要总截止时间，应先记录 `deadline = time.monotonic() + limit`，每次等待都使用剩余时间。

对于有后代进程或复杂清理的管道，还要把整组任务生命周期纳入设计。

不要把每个局部 timeout 简单相加以后仍叫“总超时”。

### 23.6 能不能先在父侧读 producer，再作为 input 传给 consumer？

可以，但那会保存全部中间内容，适合小数据。

直接连接管道则允许流动，适合生产和消费都按块或按行工作的情况。

是否真正节省内存，还取决于两个工具自己是否整体加载输入。

父侧没有 `read()` 不等于端到端恒定内存。

## 第二十四章：上下文管理、文件句柄和进程退出，三件事分别负责什么？

### 24.1 Popen 可以配合 with

```python
import subprocess
import sys

with subprocess.Popen(
    [sys.executable, "-c", "print('finished')"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
) as process:
    stdout, stderr = process.communicate(timeout=5)

print(stdout.rstrip("\n"))
print(process.returncode)
```

输出：

```text
finished
0
```

Popen 上下文退出时会关闭相应标准流，并等待进程结束。

这方便正常流程管理资源。

### 24.2 with 会自动限制进程运行时间吗？

不会。

如果块内抛异常，而子进程仍然很久不退出，上下文退出时的等待仍可能让父侧停住。

所以不能用“已经有 with”代替超时和停止政策。

如果块内通信超时，应在同一范围内明确终止、收集输出，然后再让上下文退出。

只在外面捕获超时，可能已经先卡在上下文的等待阶段。

### 24.3 哪些对象我应该主动关闭？

你自己打开的输入和输出文件，通常由文件 `with` 管理。

你手动创建的 Popen 管道，通信完成以后要安排关闭；`communicate()` 或 Popen 上下文会完成相应管理，具体以使用方式为准。

你直接传给 Popen 的外部文件对象，并不因此变成进程对象的永久私有资源。

最容易理解的结构是：外层文件上下文保持文件有效，里面启动并等待子程序，结束后离开外层关闭文件。

不要先离开文件 with，再指望后面读取对象还能正常使用。

### 24.4 进程退出以后，为什么还可能等不到 EOF？

EOF 与“所有写端都关闭”有关。

如果子程序启动了后代，后代继承 stdout 对应写端，那么直接子进程已经退出，后代仍可能持有写端。

父侧读操作就不一定立即结束。

所以 `poll()` 已经返回退出码，与管道已经没有写入者，不是同一件事。

这也解释了为什么处理进程树不能只看一条 Popen 记录。

### 24.5 close_fds 要不要改成 False？

普通任务通常保持默认值更容易避免不必要句柄泄露。

标准流重定向会按接口规则处理必要描述符，不代表你要允许所有打开的文件都被子程序继承。

POSIX 的 `pass_fds` 可以明确传递特定描述符，但属于更底层的协议设计。

Windows 句柄继承又有专门规则，应查官方平台说明。

不要为了“某个程序启动失败”就盲目关闭描述符保护，因为这可能让本不相关的资源进入子进程。

### 24.6 僵尸进程是什么意思？

在 POSIX 中，子进程结束后，父进程还需要取得其结束状态；未被回收的结束记录可能暂时保留。

`wait()`、完成的 `communicate()` 等操作帮助完成这一步。

它不是说子程序仍在执行任务，也不等同于占用大量内存的活动进程。

对初学者最实用的规则是：你创建的进程，要安排谁等待它结束，别只发一个信号就忘了。

## 第二十五章：进程组与平台差异——如何理解“整份任务停止”？

### 25.1 为什么这里不能给一行通用答案？

Windows 与 POSIX 的进程控制机制不同。

外部工具还可能主动创建新会话、脱离原来的组，或者把工作交给已有服务。

你直接创建的子进程，与业务上归属于同一任务的所有后续活动，未必完全重合。

因此先明确自己要管理什么，再选择平台机制。

### 25.2 POSIX 的 start_new_session

`Popen(..., start_new_session=True)` 在支持的 POSIX 场景中，让子程序启动为新的会话。

常见用途是把它与后续子任务安排到单独的进程组里，以便向组发信号。

下面只启动一个受控睡眠进程，用来演示组信号，不创建未知后代：

```python
import os
import signal
import subprocess
import sys

if os.name == "posix":
    process = subprocess.Popen(
        [sys.executable, "-c", "import time; time.sleep(5)"],
        stdin=subprocess.DEVNULL,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
        start_new_session=True,
    )
    try:
        os.killpg(process.pid, signal.SIGTERM)
        process.wait(timeout=2)
        print(process.returncode < 0)
    finally:
        if process.poll() is None:
            try:
                os.killpg(process.pid, signal.SIGKILL)
            except ProcessLookupError:
                pass
            process.wait()
```

在普通 POSIX 环境通常输出：

```text
True
```

由于新会话的组长是这次子进程，示例以它的 PID 作为新进程组标识。

只能对自己创建并明确归属的组做这种操作，不能把任意查到的 PID 直接当成进程组编号。

### 25.3 组信号保证所有后代都结束吗？

不保证。

后代可能脱离原来的组。

它也可能把工作交给其他长期服务。

SIGTERM 还可能被处理或忽略，宽限期和后续检查仍然重要。

如果要精确承诺整份作业结束，需要更完整的作业隔离与监控设计。

这个短例子只说明你可以控制一个新建组，不能当成任意进程树清理万能模板。

### 25.4 Windows 的 CREATE_NEW_PROCESS_GROUP 是同一套东西吗？

不是 POSIX session 的直接等价物。

Windows 控制台程序、控制事件、创建标志和句柄有自己的规则。

Python 提供 `CREATE_NEW_PROCESS_GROUP` 等常量，也允许在适用条件下发送 CTRL_BREAK_EVENT 等事件，但是否生效取决于控制台和目标程序。

复杂后代任务通常还需要了解 Windows Job Objects 等系统机制；标准库 Popen 并没有把所有这些包装成跨平台的一行“kill tree”。

本篇运行环境为 WSL/Linux，Windows 专有控制事件没有实际运行验证，因此这里只解释接口边界，不把 POSIX 实测结果套到 Windows。

### 25.5 可以用 preexec_fn 自己做所有准备吗？

普通任务不要优先这样做。

POSIX 的 `preexec_fn` 在 fork 后、exec 前执行，在线程程序中存在死锁风险。

常见目录、环境、会话、进程组设置，应优先使用已有明确参数。

Python 3.11 增加 `process_group` 参数用于适用的 POSIX 组设置；它仍不是任意平台的通用选项。

只有确实理解 fork 与线程边界时，才考虑更低层方式。[Popen 创建参数](https://docs.python.org/3.14/library/subprocess.html#popen-constructor)

## 第二十六章：完整实践一——把一个本地文本转换器做成可靠调用

### 26.1 先定义工具的责任

我们准备两个脚本。

子脚本读取指定 UTF-8 文本文件，把内容转成大写，写入指定输出路径。

父脚本负责准备工作目录、构造参数、捕获诊断、设置超时、检查退出状态和验证结果。

为了不覆盖已有数据，子工具使用 `x` 创建输出。

本节所有文件都在 `TemporaryDirectory` 中生成，练习结束自动清理。

### 26.2 子工具如何设计退出约定？

正常转换返回 `0`。

输入读取或输出写入等可预期文件问题，向 stderr 输出一条说明并返回 `2`。

参数解析由 argparse 负责，它也使用非零状态报告不合法调用。

stdout 不承担人类日志，避免以后需要机器结果时混在一起。

### 26.3 完整可运行代码

```python
from pathlib import Path
from tempfile import TemporaryDirectory
import os
import subprocess
import sys
import textwrap

CHILD_SOURCE = textwrap.dedent('''\
    import argparse
    from pathlib import Path
    import sys

    def main():
        parser = argparse.ArgumentParser()
        parser.add_argument("--input", type=Path, required=True)
        parser.add_argument("--output", type=Path, required=True)
        args = parser.parse_args()

        try:
            content = args.input.read_text(encoding="utf-8")
            with args.output.open("x", encoding="utf-8", newline="\\n") as target:
                target.write(content.upper())
        except (OSError, UnicodeError) as error:
            print(f"转换失败：{error}", file=sys.stderr)
            return 2
        return 0

    if __name__ == "__main__":
        raise SystemExit(main())
''')

with TemporaryDirectory(prefix="subprocess-convert-") as directory:
    root = Path(directory)
    child = root / "convert_text.py"
    source = root / "input with spaces.txt"
    target = root / "result with spaces.txt"
    child.write_text(CHILD_SOURCE, encoding="utf-8")
    source.write_text("hello Alice\n学习 Python\n", encoding="utf-8")

    child_env = os.environ.copy()
    child_env["PYTHONIOENCODING"] = "utf-8"
    command = [
        sys.executable,
        str(child),
        "--input", str(source),
        "--output", str(target),
    ]
    result = subprocess.run(
        command,
        cwd=root,
        env=child_env,
        stdin=subprocess.DEVNULL,
        capture_output=True,
        text=True,
        encoding="utf-8",
        timeout=5,
        check=True,
    )
    expected = "HELLO ALICE\n学习 PYTHON\n"
    assert target.read_text(encoding="utf-8") == expected
    print(result.returncode)
    print(target.read_text(encoding="utf-8"), end="")

    repeated = subprocess.run(
        command,
        cwd=root,
        env=child_env,
        stdin=subprocess.DEVNULL,
        capture_output=True,
        text=True,
        encoding="utf-8",
        timeout=5,
        check=False,
    )
    print("重复写入被拒绝：", repeated.returncode == 2)
    assert target.read_text(encoding="utf-8") == expected
```

输出：

```text
0
HELLO ALICE
学习 PYTHON
重复写入被拒绝： True
```

### 26.4 文件名里的空格为什么没出问题？

因为 `str(source)` 是一个列表元素，`str(target)` 是另一个列表元素。

它们的边界由参数序列表达。

我们没有手动给真实字符串加引号，也没有先拼成一整条 shell 命令。

子程序 argparse 收到的是完整路径。

### 26.5 为什么父侧还要检查文件内容？

退出 `0` 只表示子程序按照自己的约定认为成功。

子程序可能存在逻辑错误，例如忘记转大写、输出到错误路径、只写了一半就错误地返回成功。

练习中的断言检查关键预期，帮助发现这些问题。

真实工具的验证强度根据数据重要性决定：可以检查文件存在、格式能解析、条目数、校验和或业务约束。

### 26.6 子程序已经使用 x，是不是失败时绝对不留输出？

不是。

`x` 只保护“不覆盖已存在文件”。

第一次成功创建文件后，如果中途磁盘写入失败，可能留下部分结果。

要做到只发布完整成功结果，通常需要在同目录写临时文件，验证完成后再用明确的发布策略替换或创建最终文件。

是否允许覆盖旧结果，也必须由任务规则决定。

### 26.7 超时以后，应该自动重试吗？

当前子工具不是一个有全套幂等协议的任务服务。

如果超时发生在输出刚写完以后，文件可能已经存在。

再次执行会返回 `2`，但不能简单认为第一次什么也没做。

父侧应检查本次任务工作目录、输出状态和可重复规则。

独立任务目录、固定输入摘要与结果清单，都是以后扩展时可以采用的实际设计。

## 第二十七章：完整实践二——标准输入传 JSON，标准输出收 JSON

### 27.1 为什么这比解析随意日志更方便？

假设父程序给子程序一组笔记标题，希望得到条数与最长标题长度。

如果子程序打印：

```text
我开始工作了！
一共有 3 条笔记。
最长的是 8 个字符。
```

人能看懂，机器解析却容易依赖措辞。

我们给这次通信定义一份 JSON 协议。

输入是 `{"titles": [...]}`。

输出是 `{"count": ..., "max_length": ...}`。

诊断只写 stderr。

### 27.2 一个完全本地的协议实验

```python
import json
import subprocess
import sys
import textwrap

WORKER_CODE = textwrap.dedent('''\
    import json
    import sys

    def main():
        try:
            payload = json.load(sys.stdin)
            if not isinstance(payload, dict) or set(payload) != {"titles"}:
                raise ValueError("输入必须是只含 titles 的对象")
            titles = payload["titles"]
            if not isinstance(titles, list) or any(not isinstance(x, str) for x in titles):
                raise ValueError("titles 必须是字符串列表")
        except (ValueError, TypeError) as error:
            print(str(error), file=sys.stderr)
            return 2

        result = {
            "count": len(titles),
            "max_length": max((len(x) for x in titles), default=0),
        }
        json.dump(result, sys.stdout, ensure_ascii=True)
        sys.stdout.write("\\n")
        return 0

    if __name__ == "__main__":
        raise SystemExit(main())
''')

payload = {"titles": ["open", "pathlib", "学习笔记"]}
result = subprocess.run(
    [sys.executable, "-X", "utf8", "-c", WORKER_CODE],
    input=json.dumps(payload, ensure_ascii=True),
    capture_output=True,
    text=True,
    encoding="utf-8",
    timeout=5,
    check=True,
)
response = json.loads(result.stdout)
if not isinstance(response, dict) or set(response) != {"count", "max_length"}:
    raise ValueError("子程序返回结构不符合约定")
if type(response["count"]) is not int or type(response["max_length"]) is not int:
    raise ValueError("子程序返回计数类型不正确")
if response["count"] < 0 or response["max_length"] < 0:
    raise ValueError("子程序返回负计数")
print(response)
```

输出：

```text
{'count': 3, 'max_length': 7}
```

`pathlib` 长度是 `7`，比这里其他标题更长。

### 27.3 为什么使用 ensure_ascii=True？

它让 JSON 字符串中的非 ASCII 字符转义，整个传输内容可以由 ASCII 字符表达。

JSON 解码后仍恢复原来的 Unicode 字符串，不会把中文标题变成另一份业务内容。

同时子 Python 使用 `-X utf8`，父侧明确 UTF-8，使两端约定清楚。

这不是 JSON 必须采用 ASCII，而是本例选择的一种容易跨环境验证的表示。

### 27.4 返回码为零，为什么还要 json.loads？

退出成功不等于输出一定符合协议。

如果子程序不小心先 `print("debug")`，stdout 就可能不是一份纯 JSON 文档。

`json.loads()` 会让这种协议破坏显现出来。

解析完成还要验证字段与类型，否则 `{"count": "three"}` 在 JSON 语法上完全合法，却不符合业务契约。

### 27.5 为什么 type(value) is int，而不是 isinstance？

Python 中 bool 是 int 的子类。

如果协议的计数必须是整数且不能是 `true` 或 `false`，就需要明确排除布尔值。

这里使用 `type(...) is int` 表达这个要求。

不是所有地方都应该排除子类，但协议边界应按照你真实接受的值来校验。

### 27.6 如果输入是一百万条标题呢？

这个实践使用 `json.dumps()`、`input=` 与 `capture_output=True`，会在内存中持有相关内容。

适合较小的请求响应，不是超大数据的流式协议。

大批记录可以考虑 JSON Lines、文件交换或数据库任务队列；也要限制单行长度、总条数、输出大小和处理时长。

改为 Popen 不会自动消除内存占用，协议本身与读取方式都要调整。

## 第二十八章：完整实践三——逐行处理任务，父侧实时收到进度

### 28.1 先把“结果”和“进度”分开

父程序有三个学习项目，希望子程序依次处理。

每处理一个项目，子程序向 stderr 打印一条进度。

所有工作完成以后，stdout 只输出一个 JSON 结果。

父侧不一定要实时解析进度，可以先让 stderr 继承终端，stdout 捕获到变量。

这样只捕获一路，也避免为了漂亮的实时输出引入不必要线程。

### 28.2 最小可运行协议

```python
import json
import subprocess
import sys
import textwrap

code = textwrap.dedent('''\
    import json
    import sys

    names = json.load(sys.stdin)
    results = []
    for index, name in enumerate(names, 1):
        print(f"progress {index}/{len(names)}", file=sys.stderr, flush=True)
        results.append({"name": name, "length": len(name)})
    json.dump(results, sys.stdout, ensure_ascii=True)
    sys.stdout.write("\\n")
''')

result = subprocess.run(
    [sys.executable, "-X", "utf8", "-u", "-c", code],
    input=json.dumps(["open", "pathlib", "sqlite3"]),
    stdout=subprocess.PIPE,
    stderr=None,
    text=True,
    encoding="utf-8",
    timeout=5,
    check=True,
)
records = json.loads(result.stdout)
print([item["length"] for item in records])
```

终端会收到诊断，例如：

```text
progress 1/3
progress 2/3
progress 3/3
[4, 7, 7]
```

诊断流与父侧终端输出的具体显示顺序仍与环境有关。

这里子程序受控，因此略去重复的数据校验；面向任意调用者应沿用前一章的协议检查。

### 28.3 我想保存进度，但不在终端显示呢？

把 stderr 传给一个本次任务的新日志文件。

stdout 仍保持 PIPE 接 JSON。

这是比“stdout 和 stderr 全部 PIPE，再写两线程读取”更简单的方案。

### 28.4 如果我要显示任务百分比？

可以定义逐行 JSON 事件，让父侧按行解析。

例如：

```json
{"event":"progress","done":1,"total":3}
{"event":"progress","done":2,"total":3}
{"event":"finished","count":3}
```

每行是一份 JSON，不是把整个流作为一份多行 JSON 文档解析。

要决定错误事件、最后完成事件是否必须出现，以及进程退出码与事件发生冲突时以什么为准。

如果子程序返回 `0` 却没有 finished，父侧应按协议失败处理，而不是只看“收到一些进度”。

### 28.5 完成事件可以代替进程退出吗？

不能直接替代。

子程序可能先输出 finished，随后在清理文件时失败。

父侧仍需要等待进程结束并检查返回码。

同样，进程正常退出但没有完整结果，也不自动证明任务成功。

机器协议的完成状态与操作系统进程的退出状态，应共同满足约定。

## 第二十九章：完整实践四——大文件按块转换，诊断直接写日志

### 29.1 这次真正避免两侧整体读入

前面文件重定向例子里，父侧没有整体读取，但示例子程序使用无长度 `read()`。

现在改成固定大小的块，父侧文件直接连接到子进程，子侧也逐块读写。

只做字节复制，避免文本解码跨块边界问题。

```python
from pathlib import Path
from tempfile import TemporaryDirectory
import hashlib
import subprocess
import sys
import textwrap

COPY_CODE = textwrap.dedent('''\
    import sys
    while True:
        chunk = sys.stdin.buffer.read(65536)
        if not chunk:
            break
        sys.stdout.buffer.write(chunk)
''')


def file_sha256(path):
    digest = hashlib.sha256()
    with open(path, "rb") as source:
        for chunk in iter(lambda: source.read(65536), b""):
            digest.update(chunk)
    return digest.hexdigest()


with TemporaryDirectory(prefix="subprocess-copy-") as directory:
    root = Path(directory)
    source_path = root / "input.bin"
    target_path = root / "output.bin"
    log_path = root / "errors.log"

    with source_path.open("xb") as source:
        for number in range(64):
            source.write(bytes([number]) * 65536)

    with source_path.open("rb") as source:
        with target_path.open("xb") as target:
            with log_path.open("xb") as errors:
                subprocess.run(
                    [sys.executable, "-c", COPY_CODE],
                    stdin=source,
                    stdout=target,
                    stderr=errors,
                    timeout=5,
                    check=True,
                )

    print(target_path.stat().st_size)
    print(file_sha256(source_path) == file_sha256(target_path))
```

输出：

```text
4194304
True
```

### 29.2 65536 这个数字有什么魔法？

没有通用最优值。

它是一个便于示范的 64 KiB 块大小。

较小块增加循环和调用次数，较大块增加单次缓冲占用；实际性能取决于工具与存储。

这里重要的是每次读取有明确上限，不是必须记住这个数字。

### 29.3 为什么校验也要按块？

如果最后为了比较文件，直接把两份都 `read_bytes()`，你又在验证阶段整体载入了它们。

按块更新哈希，既验证完整内容，也不需要把整份文件放到内存。

哈希相等是本练习验证复制一致性的方式，不替代所有业务格式和真实性检查。

### 29.4 复制中途失败会怎样？

目标文件可能已经存在且不完整。

`CalledProcessError` 不会把文件删除或恢复成之前状态。

本篇临时目录会在离开时清理，所以不会留下长期练习文件。

真实任务需要自己决定失败残留的清理、保留诊断和成功发布规则。

### 29.5 为什么没把 stderr 捕获到内存？

日志直接写磁盘，父侧不必保存全部诊断。

如果工具可能产生大量错误，文件也可能增长，需要任务级大小限制或日志轮转策略。

从内存搬到磁盘只是改变资源使用位置，不等于资源无限。


## 第三十章：把调用封装成函数，错误信息应该在哪一层整理？

### 30.1 不要让每个调用者重复检查同一套协议

假设你已经确认某个外部程序的标准输出必须是 JSON 对象，非零返回都是失败。

可以写一个小函数，集中处理这份约定。

函数不应该叫“运行任何工具并保证成功”，而应描述它接受的协议。

下面这个函数只适用于有限大小的 JSON 输出。

```python
import json
import subprocess


class ToolResultError(Exception):
    """外部工具没有返回当前调用约定要求的结果。"""


def run_json_object(command, *, timeout=5, cwd=None, env=None):
    try:
        result = subprocess.run(
            command,
            cwd=cwd,
            env=env,
            stdin=subprocess.DEVNULL,
            capture_output=True,
            text=True,
            encoding="utf-8",
            errors="strict",
            timeout=timeout,
            check=True,
        )
    except subprocess.TimeoutExpired as error:
        raise ToolResultError("外部工具超过等待期限") from error
    except subprocess.CalledProcessError as error:
        detail = (error.stderr or "").strip()
        if len(detail) > 500:
            detail = detail[:500] + "……"
        raise ToolResultError(
            f"外部工具退出状态为 {error.returncode}：{detail}"
        ) from error
    except UnicodeError as error:
        raise ToolResultError("外部工具输出不符合 UTF-8 约定") from error
    except OSError as error:
        raise ToolResultError("外部工具无法启动或访问运行资源") from error

    try:
        value = json.loads(result.stdout)
    except json.JSONDecodeError as error:
        raise ToolResultError("外部工具没有输出合法 JSON") from error
    if not isinstance(value, dict):
        raise ToolResultError("外部工具输出必须是 JSON 对象")
    return value
```

### 30.2 为什么使用 raise ... from error？

对上层调用者，`ToolResultError` 表示这次外部结果不可用。

底层异常仍作为显式原因保留，调试时能看到究竟是找不到程序、超时还是 JSON 格式错。

如果只 `raise ToolResultError("失败")` 并到处隐藏回溯，你可能丢失定位线索。

面向普通用户的显示层可以只打印友好说明，开发日志保留异常链。

### 30.3 这个函数真的限制输出大小了吗？

没有。

把错误消息裁到五百字符，只是限制展示文本长度。

`capture_output=True` 在此之前已经把完整输出收集到内存。

所以函数文档必须说明只用于你控制输出规模的工具。

如果目标可能无限输出，需要流式限额、磁盘重定向和终止策略，不能把字符串切片宣传成内存保护。

### 30.4 同一个函数，试四种不同故障

依赖上面的函数定义：

```python
import sys

cases = [
    ("正常", "print('{\"count\": 2}')"),
    ("退出失败", "import sys; print('invalid', file=sys.stderr); sys.exit(3)"),
    ("不是 JSON", "print('hello')"),
    ("JSON 类型不符", "print('[1, 2]')"),
]
for name, code in cases:
    try:
        value = run_json_object([sys.executable, "-c", code])
    except ToolResultError:
        print(name, "失败")
    else:
        print(name, value)
```

输出：

```text
正常 {'count': 2}
退出失败 失败
不是 JSON 失败
JSON 类型不符 失败
```

四种情况让你看到：进程结束、退出状态、文本编码和业务协议分别承担不同检查。

### 30.5 我应该把所有 subprocess 调用统一套这个函数吗？

不应该。

图像工具输出二进制，搜索工具可能把某个非零状态当成正常未匹配，构建工具需要完整日志。

这些协议不同。

好的封装统一的是确实相同的约定，不是把所有差异藏起来。

如果某个工具的成功状态、输入格式和输出类型与这里不同，给它单独设计清楚的函数更合适。

### 30.6 多个任务要不要一口气全部启动？

如果有一千个文件，不要未经判断就创建一千个进程。

每个进程消耗内存、描述符、CPU 和磁盘带宽。

先做串行批处理，明确每个任务的状态、输出和失败规则，再按机器与任务特征引入有限并发。

子任务内部也可能自己开线程，一千个外部工具并行可能远远超过你想象的资源需求。

`subprocess` 提供启动能力，不替你制定调度策略。

## 第三十一章：练习与参考答案，把看起来相似的情况分开

### 31.1 练习：参数有空格、中文和分号

要求子程序准确收到下面三项：

```text
Alice Smith
学习笔记.txt
one;two
```

不能把它们拼成一段 Python 代码，也不使用 shell。

参考答案：

```python
import json
import subprocess
import sys

expected = ["Alice Smith", "学习笔记.txt", "one;two"]
result = subprocess.run(
    [
        sys.executable, "-X", "utf8", "-c",
        "import json,sys; print(json.dumps(sys.argv[1:], ensure_ascii=True))",
        *expected,
    ],
    capture_output=True,
    text=True,
    encoding="utf-8",
    timeout=5,
    check=True,
)
print(json.loads(result.stdout) == expected)
```

输出：

```text
True
```

为什么不直接对三项使用 `repr()` 后拼到代码里？

因为这里完全不需要改变代码结构。

把值放在 argv 这个数据通道中，代码和数据的边界更容易保持。

### 31.2 练习：stdout 没捕获和捕获后没内容，区别是什么？

让子程序什么都不输出。

一次不捕获，一次捕获，比较 stdout 属性。

```python
import subprocess
import sys

first = subprocess.run([sys.executable, "-c", "pass"], check=True)
second = subprocess.run(
    [sys.executable, "-c", "pass"],
    capture_output=True,
    text=True,
    check=True,
)
print(first.stdout is None)
print(repr(second.stdout))
```

输出：

```text
True
''
```

`None` 表示结果对象没有保存这一路捕获内容。

空字符串表示确实捕获了文本流，而本次没有内容。

写日志或协议判断时，不要把这两种状态无条件混为一谈。

### 31.3 练习：找不到程序，与程序返回二，怎样分别捕获？

要求输出两个不同分类。

```python
import subprocess
import sys

commands = [
    ["missing_program_for_notes_75123"],
    [sys.executable, "-c", "raise SystemExit(2)"],
]
for command in commands:
    try:
        subprocess.run(command, check=True, timeout=5)
    except FileNotFoundError:
        print("启动失败")
    except subprocess.CalledProcessError:
        print("运行后失败")
```

在该虚构名称不存在的环境输出：

```text
启动失败
运行后失败
```

这种分类能让用户知道应该检查安装路径，还是检查传给程序的数据。

### 31.4 练习：父目录不变，子目录明确

创建临时目录，让子程序在里面写 `marker.txt`。

父程序退出临时目录之前验证文件位置与自身 cwd。

```python
from pathlib import Path
from tempfile import TemporaryDirectory
import subprocess
import sys

before = Path.cwd()
with TemporaryDirectory(prefix="child-cwd-") as directory:
    root = Path(directory)
    subprocess.run(
        [sys.executable, "-c", "from pathlib import Path; Path('marker.txt').write_text('ok', encoding='utf-8')"],
        cwd=root,
        check=True,
        timeout=5,
    )
    print((root / "marker.txt").read_text(encoding="utf-8"))
    print(Path.cwd() == before)
```

输出：

```text
ok
True
```

`cwd` 只给子侧设置基准，没有让父侧也切换。

如果子程序接收一个相对输出路径，它会按自己的 cwd 解释；给绝对路径时，则按绝对位置处理。

### 31.5 练习：二进制内容不能随便用 text=True

子程序输出三个字节：`0`、`255`、`10`。

要求父侧准确取得三个字节，不解码、不去空白。

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "import sys; sys.stdout.buffer.write(bytes([0,255,10]))"],
    capture_output=True,
    timeout=5,
    check=True,
)
print(list(result.stdout))
```

输出：

```text
[0, 255, 10]
```

其中 `255` 不是单独有效的 UTF-8 文本字节。

如果随意启用 UTF-8 解码，可能触发解码失败。

即使某些编码能解出来，随后 `.strip()` 也可能删除本来需要保留的末尾字节表示。

字节协议应该按字节处理。

### 31.6 练习：超时以后仍然要能确认已等待清理

使用 Popen 创建一个不启动后代的睡眠程序。

通信超时后终止，再完成通信。

```python
import subprocess
import sys

process = subprocess.Popen(
    [sys.executable, "-c", "import time; time.sleep(2)"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)
try:
    process.communicate(timeout=0.1)
except subprocess.TimeoutExpired:
    process.kill()
    process.communicate()
print(process.returncode is not None)
```

输出：

```text
True
```

这个断言只确认当前 Popen 已记录结束状态，不把它扩大成“外部文件已恢复”或“所有后代已退出”。

验证的结论要与证据范围对应。

### 31.7 练习：空输入也要让子程序读到结束

使用 `run(input="")` 向子程序发送一个空文本输入。

让它输出输入长度。

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "import sys; print(len(sys.stdin.read()))"],
    input="",
    capture_output=True,
    text=True,
    timeout=5,
    check=True,
)
print(result.stdout.strip())
```

输出：

```text
0
```

空字符串不是“没有安排标准输入”。

这里 `input=` 创建相应输入管道并发送结束，使 `read()` 能返回。

如果完全不指定标准输入，子程序可能继承父侧输入来源，交互终端里就可能一直等你键盘输入。

### 31.8 练习：JSON 语法正确，业务类型却错误

让外部程序输出：

```json
{"count": true}
```

父侧要求 count 是非负整数且不能是 bool。

参考答案：

```python
import json
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "print('{\"count\": true}')"],
    capture_output=True,
    text=True,
    check=True,
    timeout=5,
)
value = json.loads(result.stdout)
valid = (
    isinstance(value, dict)
    and set(value) == {"count"}
    and type(value["count"]) is int
    and value["count"] >= 0
)
print(valid)
```

输出：

```text
False
```

这里进程成功、文本可读、JSON 可解析，都不等于业务契约通过。

### 31.9 练习：返回码检查顺序会不会丢掉诊断？

子程序先向 stderr 输出一条说明，再以 `5` 退出。

要求使用 `check=True`，但捕获后仍能取得那条说明。

```python
import subprocess
import sys

try:
    subprocess.run(
        [sys.executable, "-c", "import sys; print('missing title',file=sys.stderr); sys.exit(5)"],
        capture_output=True,
        text=True,
        check=True,
        timeout=5,
    )
except subprocess.CalledProcessError as error:
    print(error.returncode)
    print(error.stderr.strip())
```

输出：

```text
5
missing title
```

捕获输出并检查返回码可以同时使用。

异常不意味着所有结果信息都消失了。

### 31.10 练习：只验证字数，不验证内容，会漏掉什么？

假设一个工具应该把 `alpha` 转成 `ALPHA`。

错误实现却输出 `XXXXX`，长度也是 `5`。

如果你只检查长度、文件存在或返回码为零，就会漏掉逻辑错误。

对本篇的转换实践，至少断言精确预期内容。

对较大文件复制，按块哈希可以检查字节一致。

对结构化统计，检查字段、类型、条数与已知样本结果。

验证应当直接对应任务想得到的结果，不是找几个容易成功的指标代替。

## 第三十二章：那些实际写脚本时还会问的问题

### 32.1 已经会 Python 文件处理，为什么还要启动别的程序？

当功能在标准库中已经很直接，例如复制文件、列目录、读文本，直接调用库通常更容易控制错误和路径。

当你确实要用外部工具的成熟能力，例如某种编解码器、编译器或独立转换工具，再通过 subprocess 连接它。

一个任务是否需要新进程，取决于功能边界和隔离需求，不是为了把终端命令全部搬进字符串。

### 32.2 子进程是不是 Python 线程？

不是。

进程有独立地址空间和运行环境，通常需要序列化、标准流或其他机制交换数据。

线程共享同一进程内的很多状态，但也有同步与线程安全问题。

调用一个已有外部命令，用 subprocess。

并行执行自己的 Python 函数时，应另外比较 concurrent.futures、multiprocessing 等工具，不要为了传一个普通对象就强行设计一套命令行协议。

### 32.3 子脚本能直接 import 父脚本的变量吗？

它可以导入模块，但得到的是子进程里重新加载形成的模块状态，不是父侧那一份内存对象。

父程序刚刚在内存中赋值的列表不会自动传过去。

需要的数据通过 argv、stdin、文件等明确传递。

如果模块导入时会直接启动程序，也会带来副作用，所以脚本入口应使用 `if __name__ == "__main__"` 保护。

### 32.4 设置 cwd，能自动让 Python 找到所有项目模块吗？

工作目录可能影响相对路径和某些模块搜索情境，但它不是完整的包安装策略。

执行脚本、`python -m package.module`、已安装包和 PYTHONPATH 有不同规则。

优先使用正确的虚拟环境解释器与明确的项目安装方式。

不要遇到导入失败就不断把任意目录塞进环境变量，直到当前机器碰巧能运行。

### 32.5 为什么同一条命令在终端成功，在 Python 中失败？

按顺序对比实际差异：解释器或工具路径、参数列表、cwd、环境变量、标准输入来源、是否通过 shell、编码和权限。

交互 shell 的别名、函数和启动脚本，也可能在直接执行时不存在。

先记录可公开的结构化信息，再复现，不要立即认定是 subprocess 本身有问题。

如果命令涉及秘密值，记录时要脱敏。

### 32.6 外部工具的进度突然变少，是 Python 吞日志了吗？

可能是工具检测到 stdout 不再连接终端，改变了缓冲或进度显示策略。

也可能它原本把进度写到 stderr，而你只捕获 stdout。

还可能每条进度只有回车没有换行，父侧逐行读取暂时不返回。

沿着“子侧产生 -> 子侧刷新 -> 管道 -> 父侧读取 -> 父侧显示”的顺序检查，更容易找到停在哪一步。

### 32.7 run(timeout=5) 能保证第五秒立即回到主界面吗？

不能给所有平台作这样的绝对承诺。

进程创建可能存在不可立即中断的阶段，清理也需要时间，后代持有句柄还可能增加复杂性。

它为常规受控子进程提供很有用的超时机制，但不是硬实时调度保证。

对严格交互应用，应把阻塞操作放在适当的工作线程或异步任务中，并设计取消和状态反馈。

### 32.8 我想把外部工具放后台，父程序退出也继续运行呢？

这是与本篇“父侧负责等待和清理”不同的生命周期需求。

需要明确日志去向、输入关闭、工作目录、任务标识、后续如何停止和获知结果。

POSIX 与 Windows 的脱离控制台和会话机制不同。

长期服务通常适合由操作系统服务管理器或专门任务系统管理，不能简单把 `wait()` 删掉就认为后台任务可靠了。

### 32.9 这些例子实际验证了哪些平台？

本篇关键例子在 Python 3.12.3 的 WSL/Linux 环境运行，使用当前解释器、临时目录和有限本地数据。

验证覆盖参数边界、编码与 JSON、退出错误、工作目录、环境隔离、双路输出、超时清理、流式读取、直接管道、临时文件转换与按块复制。

POSIX 进程组示例在当前平台验证。

Windows 的批处理、控制台事件和原生路径规则依据官方文档解释，没有冒称在当前 Linux 环境执行了 Windows API。

需要在 Windows 原生 Python 使用的平台专属功能，应在该目标环境再做有意义的小样本验证。

### 32.10 想核对准确接口，应该看哪份原始资料？

常见参数与返回结果查 [Python subprocess 文档](https://docs.python.org/3.14/library/subprocess.html)。

理解 run 与 Popen 的定位、早期设计背景，可以看 [PEP 324](https://peps.python.org/pep-0324/)。

手动通信与超时清理查 [Popen.communicate](https://docs.python.org/3.14/library/subprocess.html#subprocess.Popen.communicate)。

管道等待风险查 [Popen.wait](https://docs.python.org/3.14/library/subprocess.html#subprocess.Popen.wait)。

涉及 shell 和 Windows 批处理参数时，查 [Security Considerations](https://docs.python.org/3.14/library/subprocess.html#security-considerations)。

POSIX 组信号还应结合 [os.killpg](https://docs.python.org/3.14/library/os.html#os.killpg) 与 [signal 文档](https://docs.python.org/3.14/library/signal.html)理解。

当你写下一次外部调用时，可以沿着一次真实运行顺着问：到底启动哪个文件，参数分别是什么，它从哪里读输入，输出给谁，多久算超时，失败以后谁清理，最后用什么证据确认结果。

这些问题都能用代码明确回答时，一次 subprocess 调用才真正成为你可以维护的工具。
