# Python `sys` 模块超详细教程：从基本原理、语法，到常用方法、组合应用和避坑指南

这一篇我们系统地讲清楚 Python 标准库里的 `sys` 模块。

我不会简单地把 `sys.argv`、`sys.path`、`sys.exit()` 等功能罗列出来，而是尽量做到：

每一个重要功能都解释：

- 是什么：它到底表示什么，英文名称是什么意思？
- 为什么：Python 为什么需要它？解决了什么问题？
- 怎么写：完整语法、参数、返回值、属性。
- 怎么用：提供可以运行的代码，解释每一行。
- 怎么搭配：如何与 `os`、`pathlib`、`subprocess`、`argparse` 等配合。
- 注意什么：实际开发中常见的报错、误区、跨平台问题。
- 什么时候用：哪些场景适合使用，哪些场景不应该使用。

本教程主要依据 Python 3.15 官方文档，同时考虑 Python 3.11～3.14 的常见使用情况。Python 3.15.2 已于 2026 年 10 月发布，因此我也会区分一些较新的功能，避免把新版本的写法误当成所有版本都支持的写法。

# 第一部分：先彻底理解 `sys` 模块

## 一、`sys` 到底是什么？

### 1. `sys` 的英文是什么意思？

`sys` 是英文 system 的缩写。

但是，这里非常容易产生一个误解：

`sys` 并不等于操作系统操作库。

它的正式定位是：

> 为 Python 解释器提供系统相关的参数、状态信息和控制功能。

什么意思？

假设你有一个 Python 文件：

```python
print("Hello Python")
```

你在终端执行：

```bash
python main.py
```

表面上，是你的电脑在运行 Python 程序。

但实际上，中间存在一个负责执行代码的程序，叫作：

**Python 解释器（Python Interpreter）。**

`sys` 能够帮助你了解和控制很多与解释器有关的事情。

例如：

```python
import sys

print(sys.version)      # 当前 Python 版本
print(sys.executable)   # 当前 Python 解释器的位置
print(sys.argv)         # 当前程序收到的命令行参数
print(sys.path)         # Python 查找模块的路径
```

甚至可以要求程序终止：

```python
sys.exit()
```

因此，可以这样理解：

`sys` 就是 Python 程序与 Python 解释器之间的重要接口。

当然，这是一种帮助理解的简化说法，不代表它能控制解释器的全部内部行为。

## 二、为什么 Python 要设计 `sys`？

我们用几个实际问题解释。

### 场景 1：怎么知道当前使用哪个 Python？

你电脑里可能同时存在：

```text
Python 3.11
Python 3.12
Python 3.13
Python 3.15
```

还有一个虚拟环境：

```text
my_project/.venv/
```

现在，你想知道程序究竟由哪个 Python 运行。

使用：

```python
import sys

print(sys.executable)
```

就能得到当前解释器可执行文件的路径。

### 场景 2：怎么让用户从终端传递参数？

你希望程序支持：

```bash
python hello.py Alice
```

程序读取 `Alice`，然后输出：

```text
你好，Alice
```

那么可以使用：

```python
sys.argv
```

### 场景 3：为什么明明安装了模块，仍然无法导入？

比如：

```python
import requests
```

却报错：

```text
ModuleNotFoundError: No module named 'requests'
```

这时候，可以通过：

```python
sys.path
sys.executable
sys.prefix
```

检查模块搜索路径、解释器和虚拟环境。

### 场景 4：发现参数错误，想立即结束程序

例如用户必须提供年龄，但没有提供：

```python
sys.exit("错误：没有提供年龄")
```

以上功能虽然都和运行环境有关，但它们彼此不同。

`sys` 把这些与解释器密切相关的接口集中在同一个模块里，方便程序使用。

## 三、`sys` 与 `os` 到底有什么区别？

这是非常重要的基础概念。

虽然两个模块都带有系统相关功能，但侧重点不同。

| 比较             | `sys`                    | `os`                       |
| ---------------- | ------------------------ | -------------------------- |
| 主要对象         | Python 解释器、运行时    | 操作系统提供的功能         |
| 查看 Python 版本 | `sys.version`            | 不是主要用途               |
| 获取命令行参数   | `sys.argv`               | 不是主要用途               |
| 查看模块搜索路径 | `sys.path`               | 不是主要用途               |
| 程序退出         | `sys.exit()`             | `os._exit()`（特殊、底层） |
| 获取当前工作目录 | 不是主要用途             | `os.getcwd()`              |
| 创建文件夹       | 不是主要用途             | `os.mkdir()`               |
| 查看环境变量     | `sys.flags` 等解释器配置 | `os.environ`               |
| 文件及目录操作   | 不是主要用途             | `os`、`pathlib`            |

一句话：

`sys` 更关注 Python 自己是怎么运行的；`os` 更关注 Python 怎样与操作系统交互。

例如：

```python
import os
import sys

print(os.getcwd())       # 当前工作目录
print(sys.executable)    # Python 解释器位置
```

注意：

当前工作目录与解释器所在目录不是一回事。

假设：

```text
当前工作目录：
D:\Projects\demo

Python 解释器：
C:\Python315\python.exe
```

这完全正常。

## 四、怎么导入 `sys`？

最常见的写法：

```python
import sys
```

之后通过：

```python
sys.名称
```

访问具体成员。

例如：

```python
import sys

print(sys.version)
```

也可以只导入某个名称：

```python
from sys import version

print(version)
```

或者：

```python
from sys import argv, exit

print(argv)

exit()
```

但我更推荐初学者采用：

```python
import sys
```

原因是更直观。

看到：

```python
sys.exit()
```

你马上知道调用的是 `sys` 模块里的 `exit()`。

而看到：

```python
exit()
```

则没有那么明确。

另外，不建议：

```python
from sys import *
```

因为会把大量名称引入当前命名空间，容易发生名称冲突，也降低代码可读性。

## 五、为什么 `sys.version` 不加括号，但 `sys.exit()` 要加？

这里要先搞清楚两个概念。

### 第一类：属性（Attribute）

例如：

```python
sys.version
sys.argv
sys.path
sys.platform
sys.executable
```

这些是模块提供的数据。

可以直接读取。

```python
print(sys.platform)
```

不需要写：

```python
sys.platform()
```

因为它并不是函数。

### 第二类：函数（Function）

例如：

```python
sys.exit()
sys.getsizeof()
sys.getrecursionlimit()
sys.setrecursionlimit()
```

它们是可以调用的函数。

所以需要使用括号：

```python
sys.getrecursionlimit()
```

如果你只是写：

```python
sys.getrecursionlimit
```

得到的是函数对象，而不是函数的执行结果。

例如：

```python
import sys

a = sys.getrecursionlimit
b = sys.getrecursionlimit()

print(a)
print(b)
```

`a` 是函数本身，`b` 才是当前递归限制数值。

后面讲解时，一定要注意这个差别。

# 第二部分：`sys.argv`——获取命令行参数

这是 `sys` 最常用、最值得重点掌握的功能之一。

## 一、`argv` 是什么意思？

`argv` 来自：

argument vector

这里：

- `argument`：参数
- `vector`：按顺序排列的一组元素

在 Python 中，`sys.argv` 是一个列表（`list`），保存程序接收到的命令行参数。

所谓命令行参数，就是执行程序时，在程序名字后面输入的那些内容。

例如：

```bash
python hello.py Alice 18
```

这里：

```text
python   → 启动解释器的命令
hello.py → 要运行的脚本
Alice    → 参数 1
18       → 参数 2
```

Python 会将传给脚本的参数组织成一个列表。

## 二、最基础的使用方法

假设创建：

```text
hello.py
```

代码：

```python
import sys

print(sys.argv)
```

在终端执行：

```bash
python hello.py Alice 18
```

可能得到：

```text
['hello.py', 'Alice', '18']
```

为什么是这样？

因为 `sys.argv` 的基本结构是：

```text
[
    "脚本名称",
    "第一个参数",
    "第二个参数",
    ...
]
```

请注意，`argv[0]` 通常是脚本名称，但它是否包含完整路径取决于启动方式和平台。

### 逐个理解列表索引

```python
sys.argv[0]
```

通常得到：

```text
hello.py
```

```python
sys.argv[1]
```

得到：

```text
Alice
```

```python
sys.argv[2]
```

得到：

```text
18
```

对应关系如下：

| 表达式          | 含义               | 示例结果                      |
| --------------- | ------------------ | ----------------------------- |
| `sys.argv`      | 完整参数列表       | `['hello.py', 'Alice', '18']` |
| `sys.argv[0]`   | 脚本名称或启动标识 | `'hello.py'`                  |
| `sys.argv[1]`   | 第一个用户参数     | `'Alice'`                     |
| `sys.argv[2]`   | 第二个用户参数     | `'18'`                        |
| `len(sys.argv)` | 列表长度           | `3`                           |
| `sys.argv[1:]`  | 所有用户参数       | `['Alice', '18']`             |

这里还有一个很重要的概念：

`sys.argv` 本身是列表，因此可以使用列表的很多操作。

例如：

```python
sys.argv[1:]
```

是切片。

```python
len(sys.argv)
```

是计算长度。

```python
for arg in sys.argv:
    print(arg)
```

是遍历。

这些都不是 `sys` 独有的语法，而是 Python 列表本身的特性。

## 三、重要注意：命令行参数默认是字符串

假设你执行：

```bash
python hello.py 10 20
```

代码：

```python
import sys

a = sys.argv[1]
b = sys.argv[2]

print(a + b)
```

你认为结果是多少？

不是：

```text
30
```

而是：

```text
1020
```

为什么？

因为：

```python
type(sys.argv[1])
```

得到：

```text
<class 'str'>
```

`sys.argv` 中的元素是字符串，而不是自动转换好的整数、浮点数。

所以：

```python
"10" + "20"
```

执行字符串拼接：

```text
"1020"
```

如果你希望它做加法，需要主动转换：

```python
import sys

a = int(sys.argv[1])
b = int(sys.argv[2])

print(a + b)
```

执行：

```bash
python hello.py 10 20
```

输出：

```text
30
```

当然，如果用户输入的是：

```bash
python hello.py abc 20
```

那么：

```python
int("abc")
```

会触发 `ValueError`。

因此，一个比较完整的程序还应该考虑输入校验。

## 四、常见问题：没有传参数会怎么样？

看下面的代码：

```python
import sys

name = sys.argv[1]

print(name)
```

如果执行：

```bash
python hello.py
```

因为没有提供额外参数，列表可能只有：

```text
['hello.py']
```

这时访问：

```python
sys.argv[1]
```

就会报错：

```text
IndexError: list index out of range
```

原因很简单：

列表只有一个元素，合法索引只有 `0`。

### 正确写法：先检查长度

```python
import sys

if len(sys.argv) < 2:
    print("错误：请提供名字")
    sys.exit(1)

name = sys.argv[1]

print(f"你好，{name}")
```

逐行看：

```python
if len(sys.argv) < 2:
```

意思是：

如果参数列表的长度小于 2，说明用户没有提供第一个脚本参数。

```python
print("错误：请提供名字")
```

输出错误提示。

```python
sys.exit(1)
```

终止程序，并表示程序未正常完成。

如果参数存在：

```python
name = sys.argv[1]
```

才安全地读取。

关于 `sys.exit(1)` 为什么传 `1`，后面会详细讲解。

## 五、进阶：`sys.argv` 与 `for` 循环搭配

假设执行：

```bash
python hello.py apple banana orange
```

代码：

```python
import sys

for item in sys.argv[1:]:
    print(item)
```

输出：

```text
apple
banana
orange
```

解释：

```python
sys.argv[1:]
```

从索引 `1` 开始切片，一直到列表结尾。

相当于排除了脚本名称。

这样无论用户传递 3 个、5 个还是 100 个参数，都可以遍历。

再举一个求和的例子：

```python
import sys

total = 0

for number in sys.argv[1:]:
    total += int(number)

print("总和：", total)
```

执行：

```bash
python sum.py 10 20 30
```

输出：

```text
总和： 60
```

这里的：

```python
total += int(number)
```

可以展开为：

```python
total = total + int(number)
```

## 六、终端参数包含空格怎么办？

假设你想传递一个完整名字：

```text
Alice Smith
```

如果写：

```bash
python hello.py Alice Smith
```

通常会被终端解析为两个参数：

```text
['hello.py', 'Alice', 'Smith']
```

因此应该使用引号：

```bash
python hello.py "Alice Smith"
```

结果：

```text
['hello.py', 'Alice Smith']
```

注意：

引号通常由命令行环境负责处理，不会成为传递给 Python 的字符串内容。

所以 `sys.argv[1]` 的值是：

```text
"Alice Smith"
```

而不是包含两侧引号的字符串。

## 七、`sys.argv` 与 `argparse` 的关系

如果你想编写真正的命令行程序，可能希望用户这样调用：

```bash
python app.py --name Alice --age 18
```

此时单纯使用 `sys.argv` 当然也能解析，但你得自己处理很多问题：

比如识别参数名称、检查参数值、显示帮助信息、处理参数顺序等。

Python 因此提供了更适合完整命令行工具的 `argparse` 模块。

例如：

```python
import argparse

parser = argparse.ArgumentParser()

parser.add_argument("--name", required=True)
parser.add_argument("--age", type=int, required=True)

args = parser.parse_args()

print(args.name)
print(args.age)
```

执行：

```bash
python app.py --name Alice --age 18
```

输出：

```text
Alice
18
```

`argparse` 默认会读取当前程序的命令行参数，正常情况下你不需要再手动处理 `sys.argv`。

如何选择？

| 场景                      | 推荐                  |
| ------------------------- | --------------------- |
| 学习命令行参数原理        | `sys.argv`            |
| 简单的一两个位置参数      | `sys.argv`            |
| 需要参数名称、类型检查    | `argparse`            |
| 需要 `-h` / `--help`      | `argparse`            |
| 多命令、多选项的 CLI 工具 | `argparse` 等专门工具 |

## 八、`sys.argv` 还有哪些特殊情况？

它的第一个元素并不总是普通文件名。

例如：

```bash
python -c "import sys; print(sys.argv)"
```

输出：

```text
['-c']
```

因为 `-c` 表示直接执行传入的 Python 代码。

而：

```bash
python -m http.server
```

是让 Python 按模块名称执行代码。

此时 `sys.argv[0]` 通常是被执行模块对应文件的路径，而不是字面上的 `'http.server'`。

另外，Python 3.10 起还提供了：

```python
sys.orig_argv
```

它保存传给 Python 解释器的原始命令行参数，包括已被解释器消耗的选项。

例如，运行：

```bash
python -B demo.py hello
```

在通常情况下：

```python
sys.argv
```

类似：

```text
['demo.py', 'hello']
```

而：

```python
sys.orig_argv
```

类似：

```text
['python', '-B', 'demo.py', 'hello']
```

实际的解释器路径和参数形式可能不同。

两者最重要的区别是：

`sys.argv` 关注脚本收到的参数；`sys.orig_argv` 还包含解释器本身收到的选项。 

Python 3.15.2 documentation

# 第三部分：`sys.exit()`——主动终止 Python 程序

## 一、`exit` 是什么意思？

英文 `exit` 就是：

退出、离开。

`sys.exit()` 的主要作用是：

> 请求退出当前 Python 程序，并向外部执行环境传递退出状态。

基本语法：

```python
sys.exit([arg])
```

这里的：

```text
[arg]
```

在文档语法中代表可选参数，不表示你真的要输入方括号。

因此，常用写法有：

```python
sys.exit()
sys.exit(0)
sys.exit(1)
sys.exit("发生错误")
```

注意，在实际代码里，调用形式是：

```python
sys.exit(1)
```

而不是：

```python
sys.exit([1])
```

后者传进去的是一个列表，含义完全不同。

## 二、为什么退出还要有一个状态码？

因为 Python 程序不仅会被人运行，也可能被其他程序、终端脚本、自动化工具调用。

这些外部程序需要判断：

你的程序到底执行成功了，还是失败了？

这时候就需要退出状态码（Exit Status）。

最基本的约定：

| 写法                   | 退出状态 | 通常代表               |
| ---------------------- | -------- | ---------------------- |
| `sys.exit()`           | `0`      | 正常结束               |
| `sys.exit(0)`          | `0`      | 正常结束               |
| `sys.exit(1)`          | `1`      | 执行失败               |
| `sys.exit(2)`          | `2`      | 经常用于命令行参数错误 |
| `sys.exit("错误信息")` | `1`      | 输出错误信息并失败退出 |

注意：

非零状态通常代表失败，但不同程序可以根据自己的约定定义具体含义。

`1`、`2` 并不是所有 Python 程序都必须遵守的固定错误分类。

## 三、实际例子：年龄校验

```python
import sys

age = int(input("请输入年龄："))

if age < 0:
    print("错误：年龄不能小于 0")
    sys.exit(1)

print("年龄合法")
print("程序继续执行")
```

如果输入：

```text
-5
```

输出：

```text
错误：年龄不能小于 0
```

后面的：

```python
print("年龄合法")
```

不会执行。

因为程序已经请求退出。

如果输入：

```text
20
```

程序正常继续。

## 四、为什么 `sys.exit()` 可以直接传字符串？

来看：

```python
import sys

sys.exit("错误：找不到配置文件")
```

效果类似：

```text
错误：找不到配置文件
```

这条信息会被输出到标准错误流（`stderr`），并以失败状态退出。

它是一个很方便的简写形式。

例如：

```python
import sys
from pathlib import Path

config = Path("config.json")

if not config.exists():
    sys.exit("错误：config.json 不存在")

print("找到配置文件")
```

这比先打印错误、再写退出状态更加简洁。

但如果是更大型的命令行程序，我更推荐：

```python
print("错误：配置文件不存在", file=sys.stderr)
sys.exit(1)
```

因为它把：

“错误消息输出到哪里”

和：

“以什么状态退出”

分开表达，更适合结构化处理。

## 五、`sys.exit()` 的底层原理：实际上是抛出异常

这一点非常重要！

很多初学者认为：

```python
sys.exit()
```

会立即强行杀死整个进程。

严格来说并不是。

它的核心行为是抛出一个：

```text
SystemExit
```

异常。

例如：

```python
import sys

try:
    print("开始")
    sys.exit(0)
finally:
    print("执行清理工作")
```

输出：

```text
开始
执行清理工作
```

为什么？

因为 Python 在处理退出时，仍然会执行适用的 `finally` 清理代码。

再来看：

```python
import sys

try:
    sys.exit(1)
except SystemExit:
    print("捕获到了退出异常")

print("程序仍然继续")
```

输出：

```text
捕获到了退出异常
程序仍然继续
```

原因是：

`sys.exit()` 抛出的 `SystemExit` 被捕获了，所以它没有继续传播到解释器顶层。

这说明：

`sys.exit()` 是正常的解释器退出机制，不是无条件终止进程的系统调用。

另外，`SystemExit` 继承自 `BaseException`，而不是普通的 `Exception` 分支。

所以：

```python
try:
    sys.exit()
except Exception:
    print("捕获异常")
```

通常不会捕获 `SystemExit`。

它的设计目的之一，就是避免一般性的异常处理代码意外阻止程序退出。

## 六、`sys.exit()` 和 `os._exit()` 区别是什么？

这也是常见考点。

```python
sys.exit()
```

与：

```python
os._exit()
```

并不等价。

| 项目             | `sys.exit()`             | `os._exit()`         |
| ---------------- | ------------------------ | -------------------- |
| 所属模块         | `sys`                    | `os`                 |
| 退出机制         | 抛出 `SystemExit`        | 直接进行底层进程退出 |
| 执行 `finally`   | 正常异常展开时会执行     | 不会                 |
| 常规 Python 清理 | 会进行正常解释器关闭流程 | 不进行               |
| 普通程序         | 推荐                     | 不推荐               |
| 特殊进程管理     | 一般不需要               | 特定场景会使用       |

例如：

```python
import os

print("开始", flush=True)

os._exit(1)

print("结束")
```

这里后面的代码不会执行。

`os._exit()` 通常只用于特殊的低层进程场景，例如某些 `fork()` 后的子进程处理。

正常写 Python 程序，优先使用 `sys.exit()`。

## 七、`sys.exit()` 最推荐的搭配：`main()` 函数

实际开发中，建议将程序逻辑放进 `main()`：

```python
import sys


def main():
    if len(sys.argv) != 2:
        print("用法：python app.py <名字>", file=sys.stderr)
        return 2

    name = sys.argv[1]

    print(f"你好，{name}")

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

运行：

```bash
python app.py Alice
```

输出：

```text
你好，Alice
```

运行：

```bash
python app.py
```

输出：

```text
用法：python app.py <名字>
```

这里：

```python
return 0
```

并不是直接结束整个解释器。

它只是从 `main()` 返回整数 `0`。

然后：

```python
sys.exit(main())
```

相当于把 `main()` 的返回值传给 `sys.exit()`，让它成为退出状态。

这种写法的优势是程序结构清楚，`main()` 可以独立测试，错误状态也方便其他程序检查。

# 第四部分：`sys.path`——Python 到底在哪里查找模块？

这一部分非常重要。

当你遇到：

```text
ModuleNotFoundError
```

或者模块导入路径不正确时，`sys.path` 往往是排查问题的关键。

## 一、`sys.path` 是什么？

`sys.path` 是一个列表，包含 Python 搜索模块时会使用的路径。

语法：

```python
sys.path
```

它不是函数，所以不需要括号。

例如：

```python
import sys

print(sys.path)
```

可能输出：

```text
[
    'D:\\Projects\\demo',
    'C:\\Python315\\python315.zip',
    'C:\\Python315\\DLLs',
    'C:\\Python315\\Lib',
    'C:\\Python315\\Lib\\site-packages'
]
```

这里只是示例。

实际结果取决于：

操作系统、Python 安装方式、虚拟环境、启动命令、环境变量等因素。

## 二、为什么 Python 要有一个模块搜索路径？

假设你写：

```python
import random
```

Python 必须找到 `random` 模块，才能导入它。

再比如：

```python
import requests
```

它也需要找到对应模块或包。

但电脑上可能存在几万个 Python 文件。

Python 总不能把整个硬盘都搜索一遍吧？

所以需要一套模块查找机制。

可以简单理解为：

这是帮助理解的简化流程。真实导入机制还包括元路径查找器、内置模块、冻结模块及缓存等，不完全由 sys.path 决定。

特别注意：

`sys.path` 主要控制基于路径的模块查找，并不是 Python 整个导入机制的全部。

## 三、怎么查看模块搜索路径？

```python
import sys

for index, path in enumerate(sys.path):
    print(index, path)
```

这里的：

```python
enumerate(sys.path)
```

会依次提供：

```text
(索引, 路径)
```

例如：

```text
0 D:\Projects\demo
1 C:\Python315\python315.zip
2 C:\Python315\DLLs
3 C:\Python315\Lib
4 C:\Python315\Lib\site-packages
```

列表顺序很重要。

对于常规路径导入，查找机制会按照相关搜索规则依次尝试路径。

如果不同目录里出现同名模块，搜索顺序可能影响最终导入哪个模块。

## 四、`sys.path` 的第一个元素为什么特别重要？

在常见启动方式下，Python 会把一个与启动位置相关的路径放到搜索列表前面。

例如：

```bash
python D:\Projects\demo\main.py
```

通常会将脚本所在目录放在较前的位置。

而：

```bash
python -c "import sys; print(sys.path)"
```

以及普通交互式 Python 会话中，开头通常是：

```text
''
```

空字符串在这里代表当前工作目录。

另外：

```bash
python -m package.module
```

通常会把启动时的当前工作目录放到搜索路径前面。

这些细节会受到 `-P`、`-I`、环境配置等选项影响。

Python 3.15.2 documentation

这就解释了为什么：

有时候你在某个目录里运行程序，模块能成功导入；

换个执行方式或者运行位置，却突然报：

```text
ModuleNotFoundError
```

因为导入搜索环境可能变化了。

## 五、`sys.path.append()` 是怎么用的？

注意：

```python
sys.path
```

是列表。

所以：

```python
sys.path.append(...)
```

使用的实际上是列表的 `append()` 方法，不是 `sys` 自己提供的独立函数。

假设目录：

```text
project/
│
├── main.py
│
└── extra/
    └── mymath.py
```

`mymath.py`：

```python
def add(a, b):
    return a + b
```

现在，你想在 `main.py` 里使用它。

一种临时方法是：

```python
import sys
from pathlib import Path

root = Path(__file__).resolve().parent
extra_dir = root / "extra"

sys.path.append(str(extra_dir))

import mymath

print(mymath.add(10, 20))
```

输出：

```text
30
```

### 逐行解释

```python
from pathlib import Path
```

导入处理文件路径的 `Path` 类。

```python
Path(__file__)
```

得到当前脚本对应的文件路径对象。

```python
.resolve()
```

将路径解析成绝对路径，并按具体情况解析符号链接。

```python
.parent
```

获取脚本所在的父目录。

```python
root / "extra"
```

使用 `pathlib` 的 `/` 运算符拼接路径。

这里的 `/` 不是数学除法。

```python
str(extra_dir)
```

将路径对象转换成字符串。

```python
sys.path.append(...)
```

把路径追加到模块搜索路径列表中。

最后：

```python
import mymath
```

就能通过新加入的目录找到模块。

## 六、`append()` 和 `insert()` 有什么区别？

都是列表方法。

### `append()`

```python
sys.path.append("/some/path")
```

追加到列表末尾。

### `insert()`

```python
sys.path.insert(0, "/some/path")
```

插入到索引 `0` 的位置。

区别：

```python
a = ["A", "B", "C"]
```

执行：

```python
a.append("X")
```

结果：

```text
["A", "B", "C", "X"]
```

执行：

```python
a.insert(0, "X")
```

结果：

```text
["X", "A", "B", "C"]
```

由于路径查找顺序很重要：

```python
sys.path.insert(0, path)
```

会提高这个路径的搜索优先级。

但这可能产生副作用。

例如，新加入的目录包含一个与第三方库同名的文件，Python 可能导入了你不想导入的那个版本。

因此不要随意把来源不可信的目录放到最前面。

## 七、为什么不建议大型项目到处修改 `sys.path`？

虽然：

```python
sys.path.append(...)
```

能够解决不少临时导入问题，但它容易引起：

模块名称冲突、依赖于启动顺序、代码迁移后失效等问题。

对于正规项目，往往更推荐组织包结构。

例如：

```text
project/
│
├── main.py
│
└── tools/
    ├── __init__.py
    └── math_utils.py
```

`math_utils.py`：

```python
def add(a, b):
    return a + b
```

`main.py`：

```python
from tools.math_utils import add

print(add(10, 20))
```

如果从合理的项目根目录启动，通常不需要手动修改 `sys.path`。

大型项目还可以使用 `pyproject.toml` 管理打包与安装，使用：

```bash
python -m pip install -e .
```

进行可编辑安装。

更稳健的原则是：

优先正确组织项目、包和执行方式，而不是把 `sys.path.append()` 当成所有导入报错的通用修复方法。

## 八、`sys.path` 和 `os.getcwd()`、`__file__` 的区别

这三个概念经常被混淆。

```python
import os
import sys
from pathlib import Path

print("工作目录：", os.getcwd())
print("脚本目录：", Path(__file__).resolve().parent)
print("模块搜索路径：", sys.path)
```

| 内容                              | 含义                      |
| --------------------------------- | ------------------------- |
| `os.getcwd()`                     | 进程当前工作目录          |
| `Path(__file__).resolve().parent` | 当前文件所在目录          |
| `sys.path`                        | Python 的模块搜索路径列表 |

例如：

你位于：

```text
D:\Projects
```

运行：

```bash
python D:\Other\demo\main.py
```

此时工作目录仍可能是：

```text
D:\Projects
```

但脚本目录是：

```text
D:\Other\demo
```

不能把它们混为一谈。

另外，`__file__` 通常存在于普通 Python 脚本中，但在某些交互式环境、Notebook 或特殊执行环境里可能不存在。

# 第五部分：`sys.modules`——Python 已加载模块的缓存

理解完 `sys.path`，接着必须了解 `sys.modules`。

两者名字看起来接近，但含义完全不同。

## 一、`sys.modules` 是什么？

`sys.modules` 是一个字典。

它记录当前解释器中已经加载或正在加载的模块，键是模块名称，值通常是模块对象。

```python
import sys

print(type(sys.modules))
```

得到：

```text
<class 'dict'>
```

## 二、为什么 Python 需要它？

想象你写：

```python
import math
```

过一会儿又写：

```python
import math
```

如果每次 `import` 都完整重新加载模块，就会产生不必要的开销。

而且，有些模块初始化时会执行代码，重复执行可能带来严重问题。

因此 Python 使用模块缓存。

例如：

```python
import sys
import math

print("math" in sys.modules)
```

输出：

```text
True
```

再看：

```python
print(sys.modules["math"])
```

会得到一个模块对象的表示。

例如：

```text
<module 'math' from '...'>
```

具体路径取决于 Python 安装。

## 三、它和 `sys.path` 有什么不同？

| 对比     | `sys.path`             | `sys.modules`      |
| -------- | ---------------------- | ------------------ |
| 类型     | `list`                 | `dict`             |
| 作用     | 搜索模块的位置         | 保存加载的模块对象 |
| 包含内容 | 路径字符串             | 模块名和模块对象   |
| 常见操作 | `append()`、`insert()` | `get()`、键查询    |
| 典型用途 | 排查模块找不到         | 检查模块是否已加载 |

可以形象地理解为：

`sys.path` 像地图，告诉 Python 去哪里寻找模块。

`sys.modules` 像已经登记的模块记录，告诉 Python 哪些模块已经进入当前解释器。

## 四、查看模块是否已经导入

```python
import sys

if "json" in sys.modules:
    print("json 已经加载")
else:
    print("json 还没有加载")
```

注意：

你并不应该假设只有自己明确 `import` 过的模块才会出现在字典里。

因为某些模块可能在解释器启动时、其他库的导入过程中就已经加载。

## 五、怎么获取缓存里的模块对象？

```python
import sys
import math

module = sys.modules.get("math")

print(module)
```

这里：

```python
sys.modules.get("math")
```

使用字典的 `get()` 方法。

如果没有对应键，默认返回：

```text
None
```

## 六、为什么不建议直接乱删 `sys.modules`？

例如：

```python
del sys.modules["math"]
```

这并不意味着所有代码里引用的 `math` 模块对象都消失了。

已经存在的模块引用仍然可以继续存在。

未来再次导入时，还可能得到一个新的模块对象，导致同一个程序里出现不同的模块对象引用。

如果你的目标是开发时重新执行模块代码，可以了解：

```python
import importlib

importlib.reload(module)
```

不过 `reload()` 本身也有一些限制，例如不会自动刷新其他地方通过 `from module import name` 保存的旧引用。

因此，不要把手动操作 `sys.modules` 当作平常的模块管理方式。

还有一个细节：

如果要遍历整个 `sys.modules`，可以先复制键：

```python
import sys

for name in tuple(sys.modules):
    print(name)
```

因为这个字典可能在程序运行中发生变化，直接遍历它可能遇到字典大小变化的问题。

Python 3.15.2 documentation

# 第六部分：`sys.stdin`、`sys.stdout`、`sys.stderr`——标准输入、标准输出、标准错误

这是 `sys` 里非常核心的一组功能。

也是理解命令行、日志、文件重定向、数据管道的基础。

## 一、先理解为什么需要“三个标准流”

你平时写：

```python
name = input("请输入姓名：")

print(name)
```

看起来只是：

读取输入，再打印输出。

但底层还涉及数据通过什么渠道进入程序、离开程序。

很多操作系统和命令行环境都有三个标准流概念。

| 名称     | 英文            | 主要用途 |
| -------- | --------------- | -------- |
| `stdin`  | Standard Input  | 标准输入 |
| `stdout` | Standard Output | 标准输出 |
| `stderr` | Standard Error  | 标准错误 |

注意英文缩写：

```text
std → standard（标准）
in  → input（输入）
out → output（输出）
err → error（错误）
```

### 直观理解

为什么要把正常输出与错误输出分开？

因为有时候你需要将程序的正常结果交给下一个程序处理，但不希望错误提示也混进数据。

例如：

一个程序专门输出 CSV 数据。

如果错误信息也进入同一输出流，就可能破坏 CSV 格式。

所以：

正常结果写 `stdout`，错误诊断写 `stderr`，是命令行程序非常重要的设计习惯。

## 二、`sys.stdout`——标准输出

最常见的写法：

```python
import sys

sys.stdout.write("Hello Python")
```

它的效果类似：

```python
print("Hello Python", end="")
```

但是两者有重要区别。

### 区别 1：`write()` 默认不会添加换行

```python
import sys

sys.stdout.write("Hello")
sys.stdout.write("World")
```

可能输出：

```text
HelloWorld
```

而：

```python
print("Hello")
print("World")
```

输出：

```text
Hello
World
```

如果你希望 `write()` 换行，需要自己添加：

```python
sys.stdout.write("Hello\n")
sys.stdout.write("World\n")
```

这里：

```text
\n
```

是换行符。

### 区别 2：`write()` 接收的是字符串

例如：

```python
sys.stdout.write(123)
```

会抛出 `TypeError`。

因为文本输出流的 `write()` 方法需要字符串。

应该写成：

```python
sys.stdout.write(str(123))
```

但：

```python
print(123)
```

是可以的。

因为 `print()` 会把对象转换成文本表示再输出。

### 区别 3：返回值不同

```python
n = sys.stdout.write("Hello")
```

`write()` 返回写入的字符数。

所以通常：

```text
n == 5
```

而：

```python
result = print("Hello")
```

通常：

```text
result is None
```

### `print()` 和 `sys.stdout.write()` 怎样选？

一般日常打印，使用：

```python
print()
```

更方便。

如果你需要更精确地控制输出内容，或者处理流式文本，可以使用：

```python
sys.stdout.write()
```

## 三、`sys.stdout.flush()`——刷新输出缓冲区

### 1. 为什么会有缓冲？

为了提高效率，输出数据有时候并不会在每一次 `write()` 后立刻交付到最终目的地。

它可能先进入缓冲区。

你可以把缓冲区想象为一个临时存放数据的地方。

例如：

```python
import sys
import time

sys.stdout.write("程序正在执行...")

time.sleep(3)

sys.stdout.write("完成\n")
```

根据运行环境和缓冲方式，第一段文字可能不会立即出现。

### 2. 解决方法：`flush()`

```python
import sys
import time

sys.stdout.write("程序正在执行...")
sys.stdout.flush()

time.sleep(3)

sys.stdout.write("完成\n")
```

这里：

```python
sys.stdout.flush()
```

要求输出流把缓冲的内容刷新到底层流。

但需要注意：

刷新 Python 这一层的缓冲，不保证所有终端、操作系统、网络设备都已经把内容最终展示或持久化。

### 3. 更简单的写法

```python
print("程序正在执行...", end="", flush=True)
```

这相当于结合了：

不自动换行和立即刷新。

### 4. 实际例子：进度显示

```python
import sys
import time

for i in range(1, 6):
    sys.stdout.write(f"\r当前进度：{i}/5")
    sys.stdout.flush()
    time.sleep(1)

print("\n任务完成")
```

这里的：

```text
\r
```

是回车符。

它会把输出位置移回当前行的开头。

在支持此行为的终端里，就能实现同一行更新进度的效果。

注意：

`'\r'` 不等于 `'\n'`。

`\r` 是回到行首，而 `\n` 是换行。

## 四、`sys.stderr`——标准错误

语法：

```python
sys.stderr.write("错误信息")
```

例如：

```python
import sys

sys.stderr.write("错误：文件不存在\n")
```

更常见、更推荐的写法是：

```python
print("错误：文件不存在", file=sys.stderr)
```

为什么能这样写？

因为 `print()` 有一个参数：

```python
file=
```

它指定输出到哪个文件对象或文本输出流。

默认情况下：

```python
print("Hello")
```

相当于：

```python
print("Hello", file=sys.stdout)
```

正常输出到标准输出。

而：

```python
print("Error", file=sys.stderr)
```

表示输出到标准错误。

### 实际使用

```python
import sys

print("开始处理数据")

try:
    value = int("abc")
except ValueError as e:
    print(f"数据处理失败：{e}", file=sys.stderr)
    sys.exit(1)
```

其中：

```python
except ValueError as e:
```

把捕获到的异常对象保存为 `e`。

然后：

```python
file=sys.stderr
```

让错误信息走错误通道。

最后以非零状态退出。

## 五、为什么 `stderr` 比普通 `print()` 更适合错误提示？

来看终端重定向。

假设：

```python
import sys

print("正常结果")
print("错误提示", file=sys.stderr)
```

文件叫：

```text
app.py
```

在终端执行：

```bash
python app.py > output.txt
```

对于常见命令行环境：

```text
output.txt
```

通常只保存：

```text
正常结果
```

而：

```text
错误提示
```

仍然输出到标准错误的目的地，通常是终端。

这是因为：

```text
>
```

通常重定向标准输出。

如果想单独保存错误：

```bash
python app.py 2> error.txt
```

这里的：

```text
2
```

通常对应标准错误流。

常见文件描述符约定是：

| 数字 | 含义            |
| ---- | --------------- |
| `0`  | 标准输入 stdin  |
| `1`  | 标准输出 stdout |
| `2`  | 标准错误 stderr |

如果你使用 Bash 或兼容的 shell：

```bash
python app.py > output.txt 2> error.txt
```

就能将正常结果与错误信息保存到两个不同文件。

注意：不同 shell 对重定向、编码以及两个流的合并处理可能有差异。

## 六、`sys.stdin`——标准输入

平时你最熟悉的：

```python
input()
```

本质上就与标准输入有关。

而 `sys.stdin` 可以让你直接读取标准输入流。

### 1. `readline()`——读取一行

```python
import sys

print("请输入一行文字：")

line = sys.stdin.readline()

print("你输入的是：", line)
```

如果输入：

```text
Hello
```

那么 `line` 通常是：

```text
"Hello\n"
```

注意，`readline()` 通常会保留行尾的换行符。

这和 `input()` 有明显区别：

```python
name = input()
```

通常得到的是不包含行尾换行符的字符串。

如果使用：

```python
line = sys.stdin.readline().strip()
```

可以去掉首尾空白字符。

但 `.strip()` 也会删除用户实际输入的首尾空格。

如果你只想去掉换行符，可以：

```python
line = sys.stdin.readline().rstrip("\r\n")
```

这样不会删除普通的首尾空格。

### 2. `read()`——读取所有剩余内容

```python
import sys

data = sys.stdin.read()

print("读取到的内容：")
print(data)
```

这个方法会尝试读取标准输入中剩余的全部内容，通常一直读到 EOF（输入结束）。

如果你在交互式终端运行，程序可能一直等待输入结束。

常见的 EOF 操作：

在 Linux/macOS 终端中通常是 `Ctrl+D`。

在 Windows 控制台中通常是 `Ctrl+Z` 后按 Enter。

具体行为取决于终端环境。

### 3. `for` 循环逐行读取

```python
import sys

for line in sys.stdin:
    print(line.rstrip("\r\n"))
```

这段代码会逐行读取输入，一直到 EOF。

它适合处理大量文本。

为什么？

因为：

```python
sys.stdin.read()
```

可能一次性把所有剩余文本读入内存。

而：

```python
for line in sys.stdin:
```

可以按行处理，通常不会要求一次性保存全部输入数据。

### 4. 标准输入和管道配合

假设：

```python
# count.py

import sys

count = 0

for line in sys.stdin:
    count += 1

print("总行数：", count)
```

可以使用支持相应命令的终端：

```bash
python count.py < data.txt
```

这里：

```text
<
```

表示把文件作为程序的标准输入来源。

程序不需要知道文件名是什么。

它只负责读取标准输入。

这正是标准流的优势。

## 七、标准流的其他常用方法

| 表达式                   | 作用                         |
| ------------------------ | ---------------------------- |
| `sys.stdin.readline()`   | 读取一行                     |
| `sys.stdin.read()`       | 读取剩余全部内容             |
| `sys.stdin.read(10)`     | 最多读取 10 个字符           |
| `sys.stdout.write(text)` | 写入文本                     |
| `sys.stderr.write(text)` | 写入错误文本                 |
| `sys.stdout.flush()`     | 刷新输出缓冲                 |
| `sys.stdout.isatty()`    | 检查输出是否连接终端类设备   |
| `sys.stdout.encoding`    | 查看输出流使用的字符编码     |
| `sys.stdout.buffer`      | 在支持时访问底层二进制缓冲流 |

例如：

```python
import sys

print(sys.stdout.isatty())
```

在普通终端中可能是：

```text
True
```

如果输出重定向到文件：

```bash
python app.py > output.txt
```

可能是：

```text
False
```

不过不能保证所有终端、IDE、Notebook 都有相同结果。

### 特别注意：`stdout` 是文本流

所以：

```python
sys.stdout.write(b"Hello")
```

通常会报类型错误。

因为：

```python
b"Hello"
```

是 `bytes` 类型。

如果你确实需要写二进制数据，并且输出对象支持 `.buffer`：

```python
sys.stdout.buffer.write(b"Hello")
```

但这种方式不适用于所有环境。

例如，一些测试工具可能用 `io.StringIO` 替换 `sys.stdout`，而它不提供 `.buffer`。

因此，编写通用库时不要无条件假设：

```python
sys.stdout.buffer
```

始终存在。

Python 3.15.2 documentation

## 八、`sys.stdout` 和 `logging` 怎么选？

对于一般脚本：

```python
print("程序开始")
```

没有问题。

但是大型项目会需要：

时间戳、日志级别、写入文件、日志轮转、不同模块统一管理日志。

这时应该优先考虑：

```python
import logging

logging.basicConfig(level=logging.INFO)

logging.info("程序开始")
logging.warning("发现异常情况")
logging.error("处理失败")
```

`logging` 默认的常规控制台处理器通常使用 `stderr`，但它的输出位置和日志格式都可以配置。

因此：

普通结果用 `print()` 或 `stdout`，诊断信息用 `stderr`，大型项目的日志使用 `logging`。

# 第七部分：`sys.version` 和 `sys.version_info`——查看 Python 版本

这两个名字特别相似，但适用场景不同。

## 一、`sys.version` 是什么？

语法：

```python
sys.version
```

它是一个字符串，用于描述 Python 解释器版本以及一些构建信息。

例如：

```python
import sys

print(sys.version)
```

可能出现类似：

```text
3.15.2 (main, Oct 25 2026, ...) [GCC ...]
```

这里只是示意，并不是所有电脑的输出都一样。

因为它可能包含：

版本号、构建日期、编译器、构建标识等信息。

注意：

`sys.version` 是字符串，不是数字。

因此：

```python
type(sys.version)
```

得到：

```text
<class 'str'>
```

## 二、为什么不能简单地通过字符串比较版本？

假设：

```python
"3.9" > "3.10"
```

结果是：

```text
True
```

你可能觉得奇怪。

Python 3.9 明明低于 Python 3.10。

为什么比较反了？

因为字符串比较是按字符逐个比较的，而不是把它们理解为软件版本号。

所以这样写：

```python
if sys.version >= "3.10":
    ...
```

不可靠。

这就是 `sys.version_info` 存在的重要原因之一。

## 三、`sys.version_info` 是什么？

语法：

```python
sys.version_info
```

它是一个可以通过索引和名称访问的版本信息对象。

假设当前版本是：

```text
Python 3.15.2
```

那么：

```python
import sys

print(sys.version_info)
```

会得到类似：

```text
sys.version_info(
    major=3,
    minor=15,
    micro=2,
    releaselevel='final',
    serial=0
)
```

不同版本中内容会变化。

这里的字段：

| 属性           | 含义       | 示例      |
| -------------- | ---------- | --------- |
| `major`        | 主版本号   | `3`       |
| `minor`        | 次版本号   | `15`      |
| `micro`        | 微版本号   | `2`       |
| `releaselevel` | 发布阶段   | `'final'` |
| `serial`       | 预发布序号 | `0`       |

例如：

```python
print(sys.version_info.major)
print(sys.version_info.minor)
print(sys.version_info.micro)
```

分别获取对应的版本组成部分。

## 四、最常见用法：判断 Python 版本

比如，你的程序要求 Python 3.11 或更高版本。

可以写：

```python
import sys

if sys.version_info < (3, 11):
    sys.exit("错误：需要 Python 3.11 或更高版本")

print("Python 版本满足要求")
```

### 为什么是 `(3, 11)`？

因为：

```python
(3, 11)
```

是一个元组。

Python 可以按顺序比较元组：

先比较第一个元素。

如果相同，再比较第二个元素。

例如：

```python
(3, 9) < (3, 11)
```

结果：

```text
True
```

这样就避免了字符串比较的问题。

注意：

比较时：

```python
sys.version_info >= (3, 11)
```

并不要求右边的元组必须有五个元素。

可以直接用两个元素判断主版本与次版本。

## 五、`sys.version` 与 `sys.version_info` 如何选择？

| 使用目的                 | 推荐                     |
| ------------------------ | ------------------------ |
| 打印详细解释器版本信息   | `sys.version`            |
| 判断能不能使用某个新特性 | `sys.version_info`       |
| 获取主版本号             | `sys.version_info.major` |
| 获取次版本号             | `sys.version_info.minor` |
| 做版本大小比较           | `sys.version_info`       |

记住：打印版本可以用 `version`，判断版本优先用 `version_info`。

# 第八部分：`sys.executable`——找到当前真正运行的 Python

## 一、`sys.executable` 是什么意思？

`executable` 的意思是：

可执行的、可执行文件。

所以：

```python
sys.executable
```

代表当前 Python 解释器可执行文件的位置。

例如：

```python
import sys

print(sys.executable)
```

Windows 下可能得到：

```text
C:\Python315\python.exe
```

Linux/macOS 下可能得到：

```text
/usr/local/bin/python3
```

虚拟环境中可能得到：

```text
D:\Projects\demo\.venv\Scripts\python.exe
```

这些都只是示例。

某些特殊运行环境可能返回空字符串或 `None`，不能保证它总是普通可执行文件的完整路径。

## 二、为什么这个属性特别有用？

假设你的电脑安装了多个 Python。

终端输入：

```bash
python --version
```

看到：

```text
Python 3.15.2
```

但在 VS Code 或 PyCharm 里运行同一个程序，可能使用的是另一个解释器。

而：

```python
print(sys.executable)
```

可以告诉你：

当前这次程序执行究竟使用哪个 Python 可执行文件。

这在排查虚拟环境与依赖问题时非常有价值。

## 三、与 `subprocess` 配合：让当前 Python 启动另一个 Python 程序

假设：

```text
project/
├── main.py
└── child.py
```

`child.py`：

```python
print("我是子程序")
```

`main.py`：

```python
import sys
import subprocess
from pathlib import Path

child = Path(__file__).resolve().parent / "child.py"

subprocess.run(
    [sys.executable, str(child)],
    check=True
)
```

运行：

```bash
python main.py
```

输出：

```text
我是子程序
```

### 逐个解释

```python
sys.executable
```

取得当前 Python 可执行文件路径。

```python
str(child)
```

取得子程序文件路径字符串。

```python
[sys.executable, str(child)]
```

告诉 `subprocess`：

使用当前 Python 解释器来运行这个脚本。

```python
check=True
```

代表如果子进程返回非零状态，就让 `subprocess.run()` 抛出 `CalledProcessError`。

### 为什么不直接这样写？

```python
subprocess.run(["python", "child.py"])
```

因为这里的：

```python
"python"
```

依赖操作系统和执行环境去解析对应的可执行文件。

找到的 Python 不一定和当前程序使用的 Python 一样。

尤其在虚拟环境、多 Python 版本共存的场景下，更容易出问题。

因此：

想让子进程使用当前同一个解释器时，推荐使用 `sys.executable`。

# 第九部分：`sys.prefix` 和 `sys.base_prefix`——识别虚拟环境

这是 `sys.executable` 之后非常值得理解的一组属性。

## 一、为什么有虚拟环境？

假设有两个项目：

```text
项目 A：需要 requests 2.x
项目 B：需要 requests 3.x
```

如果所有项目都往同一个 Python 环境里安装依赖，就容易出现版本冲突。

因此 Python 提供了虚拟环境机制。

例如：

```bash
python -m venv .venv
```

可以创建一个虚拟环境。

它能够让项目使用相对独立的包安装环境。

## 二、`sys.prefix` 是什么？

```python
sys.prefix
```

表示当前 Python 环境使用的安装前缀路径。

在普通 Python 环境下，它通常指向基础 Python 安装。

如果使用了标准的 Python 虚拟环境，它通常指向虚拟环境目录。

例如：

```python
import sys

print(sys.prefix)
```

可能得到：

```text
D:\Projects\demo\.venv
```

## 三、`sys.base_prefix` 是什么？

```python
sys.base_prefix
```

通常指向基础 Python 安装的前缀，而不会因为进入标准虚拟环境就变为虚拟环境目录。

例如：

```python
print(sys.base_prefix)
```

可能得到：

```text
C:\Python315
```

因此，两个路径不同。

## 四、如何检查是否正在虚拟环境中？

常见写法：

```python
import sys

in_venv = sys.prefix != sys.base_prefix

print(in_venv)
```

输出可能是：

```text
True
```

这通常表示程序正在 `venv` 之类的虚拟环境中运行。

也可以写成：

```python
import sys

if sys.prefix != sys.base_prefix:
    print("当前正在虚拟环境中")
else:
    print("当前未检测到标准虚拟环境")
```

这里需要说准确：

这种判断适用于常见的 `venv` 环境，但不保证能够识别所有 Python 环境管理机制。

例如 Conda 使用的是另一套环境管理方式，不能简单地把上面的判断当成通用 Conda 检测器。

此外，Python 3.14 起，虚拟环境的 `sys.prefix` 初始化行为有所调整，具体来说，它更早地在路径初始化阶段设置，而不再依赖 `site` 完成这一步。

Python 3.15.2 documentation

# 第十部分：`sys.platform`——识别运行平台

## 一、`platform` 是什么？

英文：

`platform` = 平台。

语法：

```python
sys.platform
```

用于获取 Python 当前运行的平台标识。

例如：

```python
import sys

print(sys.platform)
```

典型结果：

| 操作系统或平台      | 常见值      |
| ------------------- | ----------- |
| Windows             | `'win32'`   |
| Linux               | `'linux'`   |
| macOS               | `'darwin'`  |
| Android（较新版本） | `'android'` |
| iOS                 | `'ios'`     |
| FreeBSD             | `'freebsd'` |

这些是 Python 平台标识，不一定与系统日常展示的名称一致。

Python 3.15.2 documentation

## 二、为什么 Windows 64 位仍然叫 `'win32'`？

这是非常经典的误区。

很多人看到：

```python
sys.platform == "win32"
```

就认为自己的 Python 是 32 位。

实际上不是。

`'win32'` 是 Python 使用的 Windows 平台标识。

它不表示 Python 解释器一定是 32 位。

即使使用 64 位 Windows 和 64 位 Python，也通常会得到：

```text
'win32'
```

因此：

不能通过 `sys.platform == "win32"` 判断 CPU 或 Python 是不是 32 位。

## 三、跨平台分支判断

例如：

```python
import sys

if sys.platform == "win32":
    print("当前是 Windows")
elif sys.platform == "darwin":
    print("当前是 macOS")
elif sys.platform.startswith("linux"):
    print("当前是 Linux")
else:
    print("其他平台")
```

这里的：

```python
startswith()
```

是字符串方法，表示判断字符串是否以某个内容开头。

## 四、`sys.platform` 与 `os.name` 区别

```python
import os
import sys

print(sys.platform)
print(os.name)
```

在普通 Windows Python 中，可能是：

```text
win32
nt
```

在普通 Linux Python 中，可能是：

```text
linux
posix
```

为什么不同？

因为：

```python
sys.platform
```

相对更具体地标识运行平台。

而：

```python
os.name
```

主要用更粗略的分类表示操作系统接口类型。

如果你希望得到详细的操作系统名称、版本、架构，应该了解：

```python
import platform

print(platform.system())
print(platform.release())
print(platform.machine())
```

这里的 `platform` 是另一个标准库模块，不是 `sys.platform` 属性。

# 第十一部分：`sys.maxsize`、`sys.byteorder`——整数范围与字节顺序

## 一、`sys.maxsize` 是什么？

语法：

```python
sys.maxsize
```

它表示当前 Python 实现使用的 `Py_ssize_t` 类型可表示的最大正整数。

在 CPython 中，`Py_ssize_t` 常用于表示某些对象的大小、索引及长度等。

例如：

```python
import sys

print(sys.maxsize)
```

典型 64 位 CPython 中，可能是：

```text
9223372036854775807
```

即：

\[ 2^{63}-1 \]

典型 32 位 CPython 中可能是：

```text
2147483647
```

即：

\[ 2^{31}-1 \]

### 重点：它不是 Python 整数的最大值！

很多其他语言的整数类型存在固定的数值范围。

但 Python 的普通 `int` 可以根据需要使用更多内存，表示非常大的整数。

例如：

```python
x = 10 ** 100

print(x)
```

这是一个有 101 位十进制数字的整数。

它远远超过了 64 位有符号整数的最大值。

但 Python 仍然可以正常表示。

所以：

```python
sys.maxsize
```

不是 Python `int` 的最大值。

它更直接地与解释器内部的大小、索引限制有关。

此外，即使某个平台的 `sys.maxsize` 很大，也不意味着电脑真的有足够内存创建那么大的列表。

## 二、能否通过 `sys.maxsize` 判断 Python 是 64 位？

常见的粗略判断方式：

```python
import sys

if sys.maxsize > 2**32:
    print("通常是 64 位 Python")
else:
    print("通常是 32 位 Python")
```

对常见 CPython 构建来说，这个判断通常有用。

但它不是一个描述操作系统架构的通用 API。

从 Python 3.15 开始，还增加了：

```python
sys.abi_info.pointer_bits
```

用于获取当前 Python 解释器 ABI 所使用的指针位数。

例如：

```python
import sys

if sys.version_info >= (3, 15):
    print(sys.abi_info.pointer_bits)
```

可能得到：

```text
64
```

这个新属性更加直接地反映解释器指针大小。

## 三、`sys.byteorder` 是什么？

`byteorder` 的意思是：

字节顺序。

语法：

```python
sys.byteorder
```

可能返回：

```text
'little'
```

或者：

```text
'big'
```

分别对应：

小端序（Little-endian）和大端序（Big-endian）。

### 为什么需要字节顺序？

假设一个整数：

```text
0x12345678
```

占用四个字节：

```text
12 34 56 78
```

在大端序中，按从低到高的内存地址排列：

```text
12 34 56 78
```

在小端序中：

```text
78 56 34 12
```

整数 0x12345678 的四字节表示

图中从左到右代表低地址到高地址。

大端序 Big-endian

```text
12
34
56
78
```

小端序 Little-endian

```text
78
56
34
12
```

Python 例子：

```python
import sys

number = 0x12345678

result = number.to_bytes(
    4,
    byteorder=sys.byteorder
)

print(result)
```

在小端序机器上可能得到：

```text
b'xV4\x12'
```

这是对应字节的 `bytes` 表示形式。

### 应用场景

字节序主要用于：

网络协议、二进制文件、硬件通信、底层数据格式等。

但是，如果某个文件格式明确要求大端序，就应该写：

```python
number.to_bytes(4, byteorder="big")
```

而不是简单地使用当前机器的：

```python
sys.byteorder
```

因为协议要求的字节顺序，不一定等于运行机器的字节顺序。

# 第十二部分：`sys.getsizeof()`——查看对象占用多少内存

这个函数很实用，但也极其容易被误解。

## 一、语法

```python
sys.getsizeof(object[, default])
```

实际调用例如：

```python
sys.getsizeof(123)
```

或者：

```python
sys.getsizeof("Hello")
```

参数：

`object` 表示要检查的对象。

`default` 是可选参数，当对象不提供可用的大小信息时，可以返回这个默认值。

返回值通常是：

对象直接占用的内存大小，单位为字节（byte）。

## 二、基础例子

```python
import sys

a = 100
b = "Hello"
c = [1, 2, 3]

print(sys.getsizeof(a))
print(sys.getsizeof(b))
print(sys.getsizeof(c))
```

结果取决于 Python 版本、实现和架构。

因此，不应该认为每个对象的大小都是永远固定的数字。

## 三、重要陷阱：它不等于对象所有关联内容的总内存！

假设：

```python
import sys

a = []
b = [1, 2, 3]
```

你分别查看：

```python
print(sys.getsizeof(a))
print(sys.getsizeof(b))
```

得到的大小可能不同。

但这里的关键是：

```python
sys.getsizeof(b)
```

通常只计算列表对象自身及其直接负责的存储开销。

它并不会自动把：

```python
1
2
3
```

这些被列表引用的对象的全部内存大小都递归加进去。

### 为什么？

因为 Python 的列表保存的是对象引用。

可以简化理解为：

```text
列表对象
  │
  ├── 引用 → 整数对象 1
  ├── 引用 → 整数对象 2
  └── 引用 → 整数对象 3
```

`getsizeof()` 主要关注传入对象直接归属的内存。

并不会默认递归统计整个对象图。

官方文档也特别指出了这一点。

Python 3.15.2 documentation

## 四、进一步：比较字符串与列表

```python
import sys

s = "a" * 1000
lst = [s]

print("字符串大小：", sys.getsizeof(s))
print("列表大小：", sys.getsizeof(lst))
```

你会发现：

列表大小通常远小于这个长字符串的大小。

因为列表内部只是引用了 `s` 这个对象。

更重要的是：

```python
lst = [s, s, s]
```

三次引用的是同一个字符串对象，而不是自动创建三份独立字符串内容。

所以即使你想计算整个结构的占用，也要考虑共享引用的问题，否则容易重复统计。

## 五、`sys.getsizeof()` 和其他工具如何选择？

| 工具              | 适合什么                 |
| ----------------- | ------------------------ |
| `sys.getsizeof()` | 查看单个对象直接内存大小 |
| `tracemalloc`     | 跟踪 Python 内存分配情况 |
| `gc`              | 分析垃圾回收及对象引用   |
| 内存分析工具      | 分析进程整体内存占用     |

例如：

```python
import tracemalloc

tracemalloc.start()

data = [i for i in range(100000)]

current, peak = tracemalloc.get_traced_memory()

print("当前跟踪内存：", current)
print("峰值跟踪内存：", peak)
```

不过 `tracemalloc` 跟踪的是被它管理或追踪到的内存分配，不等于操作系统报告的整个进程物理内存。

结论：`getsizeof()` 非常适合观察对象，但不适合直接充当完整的进程内存检测工具。

# 第十三部分：`sys.getrecursionlimit()` 和 `sys.setrecursionlimit()`——递归深度限制

## 一、先理解什么是递归

递归（Recursion）就是函数直接或间接调用自己。

例如：

```python
def countdown(n):
    if n == 0:
        print("结束")
        return

    print(n)
    countdown(n - 1)


countdown(5)
```

输出：

```text
5
4
3
2
1
结束
```

这段代码有一个明确的终止条件：

```python
if n == 0:
```

所以最终可以退出。

但如果没有终止条件：

```python
def f():
    f()

f()
```

函数会不断调用自己。

这会不断增加调用栈的深度。

为了避免无限递归造成严重问题，Python 设置了递归深度限制。

## 二、`sys.getrecursionlimit()`——查看递归限制

语法：

```python
sys.getrecursionlimit()
```

例如：

```python
import sys

print(sys.getrecursionlimit())
```

在很多常见 CPython 环境中，结果是：

```text
1000
```

但是这个值可以被修改，也可能受运行环境影响。

所以不能认为它一定是 `1000`。

另外，递归限制是解释器用于控制调用栈深度的机制。

它不意味着“一个递归函数恰好能再调用自己 1000 次”。

因为当前程序已经存在一定的调用层级，而且实际检测还有一些实现细节。

## 三、`sys.setrecursionlimit()`——设置递归限制

语法：

```python
sys.setrecursionlimit(limit)
```

例如：

```python
import sys

sys.setrecursionlimit(2000)

print(sys.getrecursionlimit())
```

通常输出：

```text
2000
```

它的意思是：

将当前解释器的递归限制设置为 `2000`。

注意：

这个函数正常情况下返回 `None`，不是返回设置后的限制值。

## 四、是不是设置得越大越好？

当然不是！

很多初学者看到：

```text
RecursionError: maximum recursion depth exceeded
```

第一反应就是：

```python
sys.setrecursionlimit(1000000)
```

这可能非常危险。

因为解释器递归限制的设计目的之一，就是保护底层调用栈。

如果把它设置得特别高，在部分平台或 Python 实现中可能导致栈溢出、进程崩溃等问题。

官方文档也明确提醒，高限制值必须谨慎使用。

Python 3.15.2 documentation

因此，正确排查顺序应该是：

先检查递归是否存在死循环。

再检查算法能否改为迭代。

最后才考虑是否需要适度提高递归限制。

## 五、递归和迭代对比例子

递归版本：

```python
def factorial(n):
    if n == 0:
        return 1

    return n * factorial(n - 1)
```

迭代版本：

```python
def factorial(n):
    result = 1

    for i in range(1, n + 1):
        result *= i

    return result
```

两者都可以计算非负整数的阶乘。

对于很大的 `n`，迭代版本不会因为函数层层调用而消耗不断增长的递归调用栈。

所以在可以方便地改成迭代的场景下，迭代常常更合适。

# 第十四部分：`sys.exception()`、`sys.exc_info()`——获取正在处理的异常

这一组功能与 Python 的异常处理密切相关。

先看平时最常见的写法：

```python
try:
    x = 10 / 0
except ZeroDivisionError as e:
    print(e)
```

我们都知道：

```python
10 / 0
```

会触发：

```text
ZeroDivisionError
```

但如果你想获取当前正在处理的异常对象，除了 `except ... as e`，`sys` 也提供对应的方法。

## 一、`sys.exception()`——获取当前异常对象

Python 3.11 开始提供：

```python
sys.exception()
```

例如：

```python
import sys

try:
    x = 10 / 0
except ZeroDivisionError:
    error = sys.exception()

    print(error)
    print(type(error))
```

输出类似：

```text
division by zero
<class 'ZeroDivisionError'>
```

解释：

```python
sys.exception()
```

会返回当前异常处理器正在处理的异常实例。

如果当前没有正在执行的异常处理器，则返回：

```text
None
```

例如：

```python
import sys

print(sys.exception())
```

通常输出：

```text
None
```

### 它和 `except ... as e` 有什么区别？

对于普通情况：

```python
try:
    ...
except ValueError as e:
    print(e)
```

往往已经够用了。

没必要为了获取异常对象，专门调用：

```python
sys.exception()
```

`sys.exception()` 主要适用于异常处理框架、调试工具，或者需要在不显式命名异常变量的情况下获取当前异常的场景。

## 二、`sys.exc_info()`——获取异常信息三元组

语法：

```python
sys.exc_info()
```

返回：

```text
(异常类型, 异常实例, traceback对象)
```

例如：

```python
import sys

try:
    10 / 0
except ZeroDivisionError:
    info = sys.exc_info()

    print(info[0])
    print(info[1])
    print(info[2])
```

输出类似：

```text
<class 'ZeroDivisionError'>
division by zero
<traceback object at ...>
```

这里：

```python
info[0]
```

是异常类型。

```python
info[1]
```

是异常对象。

```python
info[2]
```

是 traceback 对象，记录异常的调用栈信息。

如果没有正在处理的异常，通常得到：

```text
(None, None, None)
```

## 三、`traceback` 到底是什么意思？

英文拆解：

`trace` = 追踪

`back` = 向后、回溯

在 Python 异常处理中，traceback 用于描述异常发生时的调用路径。

例如：

```python
def a():
    b()

def b():
    c()

def c():
    1 / 0

a()
```

错误信息会包含类似：

```text
Traceback (most recent call last):
  ...
  File "...", line ..., in a
  File "...", line ..., in b
  File "...", line ..., in c
ZeroDivisionError: division by zero
```

它告诉你：

程序调用了哪个函数，哪个函数又调用了哪个函数，最后在哪里出现错误。

这对于定位复杂程序中的错误非常有价值。

## 四、与 `traceback` 模块搭配

如果你需要在捕获异常之后打印完整的错误调用信息：

```python
import sys
import traceback

try:
    result = 10 / 0
except ZeroDivisionError:
    traceback.print_exc(file=sys.stderr)
```

这里：

```python
traceback.print_exc()
```

是 `traceback` 模块提供的函数。

它用于打印当前正在处理的异常。

```python
file=sys.stderr
```

表示把异常信息输出到标准错误流。

### 什么时候使用？

小程序：

```python
except ValueError as e:
    print(e)
```

通常够用。

调试复杂程序：

```python
except Exception:
    traceback.print_exc()
```

会保留更多错误定位信息。

生产环境中的大型项目则通常优先使用：

```python
logging.exception("处理失败")
```

它能够把当前异常的 traceback 记录到配置好的日志系统中。

## 五、`sys.excepthook`——处理未捕获的异常

这比前面两个方法更高级。

语法上，解释器通过：

```python
sys.excepthook(exc_type, exc_value, exc_traceback)
```

处理某些未捕获异常。

正常情况下，当主线程中的程序出现未捕获异常时，解释器会打印错误信息并退出。

而 `sys.excepthook` 允许你自定义这种处理行为。

例如：

```python
import sys
import traceback


def custom_hook(exc_type, exc_value, exc_tb):
    print("程序发生了未处理的异常！", file=sys.stderr)

    traceback.print_exception(
        exc_type,
        exc_value,
        exc_tb,
        file=sys.stderr
    )


sys.excepthook = custom_hook

x = 1 / 0
```

执行后，未捕获异常就会经过自定义函数。

### 三个参数的意思

| 参数        | 含义           |
| ----------- | -------------- |
| `exc_type`  | 异常类型       |
| `exc_value` | 异常实例       |
| `exc_tb`    | 异常 traceback |

注意，这个钩子不会把程序里所有异常都拦截下来。

已经被 `try...except` 正常处理的异常，不会因为这个钩子而自动再次执行。

而 `SystemExit` 也不是通过普通未捕获异常机制来打印错误的。

另外，`threading.Thread` 中未捕获的异常由 `threading.excepthook()` 等相关机制处理，不能认为 `sys.excepthook` 能直接统一处理所有线程的异常。

## 六、`sys.tracebacklimit`——控制未捕获异常的 traceback 长度

例如：

```python
import sys

sys.tracebacklimit = 2
```

当发生未处理异常时，解释器默认的 traceback 输出会限制显示的层数。

如果设置：

```python
sys.tracebacklimit = 0
```

通常会隐藏调用栈 traceback，只保留异常类型和值。

但要注意：

这并不是忽略异常，也不会让程序自动恢复运行。

它只是影响默认的 traceback 显示。

一般不建议为了让错误输出看起来简短，就长期隐藏 traceback，因为这会减少诊断信息。

## 七、`sys.last_exc`——交互式调试时的上一个未处理异常

Python 3.12 起提供：

```python
sys.last_exc
```

当解释器打印了某些未处理异常后，这个属性会保存该异常实例。

它不是永远存在的。

例如：

```python
getattr(sys, "last_exc", None)
```

可以安全地尝试获取。

它的典型用途是交互式会话中进行事后调试（post-mortem debugging）。

普通应用程序不应该依赖这个属性作为主要异常处理方式。

Python 3.15.2 documentation+1

# 第十五部分：字符编码相关功能——`sys.getdefaultencoding()` 等

这一部分对处理中文、文件名和跨平台程序尤其有帮助。

## 一、先理解什么是字符编码

电脑底层处理的是字节。

但我们平时使用的是：

```text
Hello
你好
Python
```

这样的文本。

编码负责把文本转换为字节。

解码负责把字节转换回文本。

例如：

```python
text = "你好"

data = text.encode("utf-8")

print(data)
```

输出：

```text
b'\xe4\xbd\xa0\xe5\xa5\xbd'
```

这里的字节序列就是 UTF-8 编码后的结果。

再解码：

```python
print(data.decode("utf-8"))
```

输出：

```text
你好
```

## 二、`sys.getdefaultencoding()`

语法：

```python
sys.getdefaultencoding()
```

例如：

```python
import sys

print(sys.getdefaultencoding())
```

返回：

```text
utf-8
```

它表示 Python 默认字符串编码，常用于不显式指定编码的 `str.encode()` 等操作。

但这里有一个特别重要的误区：

`sys.getdefaultencoding()` 不等于所有文件、终端和操作系统使用的编码。

例如：

```python
sys.getdefaultencoding()
```

和：

```python
sys.stdout.encoding
```

并不是同一个概念。

Python 3.15 起，UTF-8 也成为了普通文本文件操作等场景中的默认编码；而较早版本中的某些文件打开行为会依赖区域设置。

即便如此，终端流、重定向、外部文件格式仍然可能有独立的编码配置。

Python.org

因此，为了让文件处理更可预测，我仍然推荐显式写：

```python
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

## 三、`sys.stdout.encoding`——当前标准输出的编码

```python
import sys

print(sys.stdout.encoding)
```

可能返回：

```text
utf-8
```

但不要保证所有电脑都一定如此。

因为输出编码可能受到：

终端、操作系统、重定向环境、Python 启动配置等因素影响。

比如你想了解当前三个标准流的编码：

```python
import sys

print("stdin:", sys.stdin.encoding)
print("stdout:", sys.stdout.encoding)
print("stderr:", sys.stderr.encoding)
```

这些值可能相同，也可能不同。

## 四、`sys.getfilesystemencoding()`——文件系统编码

语法：

```python
sys.getfilesystemencoding()
```

它主要与：

文件路径名称在 Python 字符串与底层字节表示之间如何转换有关。

例如：

```python
import sys

print(sys.getfilesystemencoding())
```

可能得到：

```text
utf-8
```

### 重要区别

文件名：

```text
报告.txt
```

和文件内容：

```text
这是实验报告
```

是两回事。

`sys.getfilesystemencoding()` 主要关心文件路径的表示，不是告诉你所有文本文件的内容编码。

所以不要因为它返回：

```text
utf-8
```

就认为电脑上的所有 `.txt`、`.csv` 文件都一定是 UTF-8。

## 五、`sys.getfilesystemencodeerrors()`

语法：

```python
sys.getfilesystemencodeerrors()
```

它返回文件系统编码转换时使用的错误处理策略名称。

例如，某些 Unix 环境中可能得到：

```text
surrogateescape
```

这是为了处理某些无法正常解码的文件名字节。

通常情况下，你不需要自己手动组合这些配置。

更推荐使用 `os` 提供的方法：

```python
import os

path = "报告.txt"

data = os.fsencode(path)

print(data)

restored = os.fsdecode(data)

print(restored)
```

这里：

```python
os.fsencode()
```

使用适合文件系统路径的编码与错误处理规则。

```python
os.fsdecode()
```

则执行对应的解码。

这就是 `sys` 与 `os` 在字符编码方面的一种典型搭配。

# 第十六部分：`sys.flags`——查看解释器启动选项

## 一、`sys.flags` 是什么？

Python 启动时，可以使用不同的选项。

例如：

```bash
python -O app.py
```

或者：

```bash
python -B app.py
```

这些选项会改变解释器的一些行为。

`sys.flags` 就用于查看相关启动标志的状态。

语法：

```python
sys.flags
```

例如：

```python
import sys

print(sys.flags)
```

它返回一个带有名称字段的元组式对象。

你可以通过属性查看指定标志：

```python
print(sys.flags.optimize)
```

## 二、几个非常常用的标志

| 属性                            | 常见对应选项 | 含义                         |
| ------------------------------- | ------------ | ---------------------------- |
| `sys.flags.optimize`            | `-O`、`-OO`  | 优化级别                     |
| `sys.flags.dont_write_bytecode` | `-B`         | 是否禁用字节码缓存写入       |
| `sys.flags.isolated`            | `-I`         | 是否开启隔离模式             |
| `sys.flags.ignore_environment`  | `-E`         | 是否忽略相关 Python 环境变量 |
| `sys.flags.no_site`             | `-S`         | 是否跳过自动导入 `site`      |
| `sys.flags.safe_path`           | `-P`         | 是否避免添加潜在不安全路径   |
| `sys.flags.dev_mode`            | `-X dev`     | 是否启用 Python 开发模式     |

不同 Python 版本可能增加新的 `flags` 字段。

Python 3.15.2 documentation

## 三、例子：查看优化级别

创建：

```python
# test.py

import sys

print(sys.flags.optimize)
```

普通运行：

```bash
python test.py
```

通常输出：

```text
0
```

使用：

```bash
python -O test.py
```

通常输出：

```text
1
```

使用：

```bash
python -OO test.py
```

通常输出：

```text
2
```

特别注意：

`-O` 并不意味着 Python 会自动把你的程序变成“极速运行模式”。

它会改变某些代码的执行行为，例如移除 `assert` 语句。

`-OO` 还会移除文档字符串等。

因此，不应该使用 `assert` 作为必须执行的用户输入校验。

错误示范：

```python
assert age >= 0, "年龄不能为负数"
```

如果程序使用 `-O` 运行，这条断言可能根本不会执行。

真正的业务校验应该使用：

```python
if age < 0:
    raise ValueError("年龄不能为负数")
```

## 四、`sys.dont_write_bytecode`——控制 `.pyc` 缓存文件

你可能见过这样的目录：

```text
__pycache__/
```

里面可能包含：

```text
mymodule.cpython-315.pyc
```

这些 `.pyc` 文件用于缓存 Python 字节码。

相关属性：

```python
sys.dont_write_bytecode
```

例如：

```python
import sys

print(sys.dont_write_bytecode)
```

通常为布尔值。

如果设置：

```python
sys.dont_write_bytecode = True
```

Python 将不再尝试为随后导入的源模块写入相应的 `.pyc` 缓存文件。

你也可以在启动时使用：

```bash
python -B app.py
```

效果更明确，因为它从解释器启动阶段就生效。

注意：

这不会自动删除已有的 `__pycache__`。

也不意味着 Python 不再编译或执行字节码。

它主要控制字节码缓存文件的写入行为。

Python 3.15.2 documentation

## 五、`sys.pycache_prefix`——改变字节码缓存位置

对于一些项目，你可能不希望 `.pyc` 缓存散落在源码目录里。

可以了解：

```python
sys.pycache_prefix
```

它可以配置独立的字节码缓存目录树。

例如启动时使用：

```bash
python -X pycache_prefix=cache app.py
```

具体效果还会受到路径和运行配置影响。

这个功能对于特殊构建环境和开发工具有用，但普通学习阶段不需要频繁修改。

# 第十七部分：`sys.intern()`——字符串驻留机制

这部分属于相对进阶的内容。

如果你正在学习 Python 对象、字符串优化或者 `is` 与 `==` 的区别，就很值得理解。

## 一、什么是字符串驻留？

英文：

`intern` = 驻留、内部共享。

Python 中的字符串是对象。

假设：

```python
a = "hello"
b = "hello"
```

Python 可能对部分字符串进行共享和优化。

但不能因此认为：

只要两个字符串内容一样，就一定是同一个对象。

这涉及两个概念：

```python
==
```

与：

```python
is
```

### `==`

比较对象的值是否相等。

```python
a == b
```

### `is`

判断两个变量是否引用同一个对象。

```python
a is b
```

这是完全不同的比较。

## 二、`sys.intern()` 的作用

语法：

```python
sys.intern(string)
```

返回驻留后的字符串。

对于相同内容的字符串，驻留机制可以让它们共享驻留字符串对象。

例如：

```python
import sys

a = "".join(["hello", "_", "world"])
b = "".join(["hello", "_", "world"])

a = sys.intern(a)
b = sys.intern(b)

print(a == b)
print(a is b)
```

通常输出：

```text
True
True
```

这里两次 `join()` 创建内容相同的字符串，然后通过 `sys.intern()` 将它们驻留。

注意：

我们应该保存 `intern()` 返回的对象：

```python
a = sys.intern(a)
```

而不是仅仅：

```python
sys.intern(a)
```

然后继续假设原来的变量已经被替换。

## 三、为什么字符串驻留可能提高效率？

假设程序有大量重复的字符串键：

```text
username
password
email
username
password
email
...
```

如果这些字符串被驻留，部分相等性比较可以利用对象身份比较来加速。

但要知道：

Python 自身已经会自动驻留某些字符串。

对普通程序来说：

不要为了所谓的性能优化，就对所有字符串无差别调用 `sys.intern()`。

它未必带来明显收益，还会增加代码复杂度。

并且：

```python
a is b
```

仍然不应该成为普通字符串内容比较的主要方法。

比较文本内容，应当用：

```python
a == b
```

而不是：

```python
a is b
```

# 第十八部分：`sys.getrefcount()`——查看对象引用计数

## 一、为什么会有引用计数？

在 CPython 中，对象内存管理的一项核心机制是引用计数。

我们举例：

```python
a = []
```

创建一个列表，并通过变量 `a` 引用它。

再写：

```python
b = a
```

此时：

```python
a is b
```

为：

```text
True
```

因为 `a` 和 `b` 指向同一个列表对象。

这个对象的引用情况发生了变化。

## 二、`sys.getrefcount()` 的语法

```python
sys.getrefcount(object)
```

例如：

```python
import sys

a = []

print(sys.getrefcount(a))

b = a

print(sys.getrefcount(a))
```

在常见的 CPython 构建中，第二次结果可能比第一次大。

但不应该死记具体数值。

### 为什么它经常比你想的多 1？

因为调用：

```python
sys.getrefcount(a)
```

时，把 `a` 作为参数传给函数，调用过程本身通常也会产生一个临时引用。

此外，CPython 还存在内部引用、特殊对象、不可销毁对象（immortal objects）等实现机制。

因此，返回值不能简单视为“程序里有几个变量指向这个对象”。

Python 官方文档也明确提醒，尤其对某些特殊对象，其引用计数不一定准确反映普通意义上的引用数量。

Python 3.15.2 documentation

## 三、它与 `sys.getsizeof()` 有什么不同？

| 方法                   | 检查什么                     |
| ---------------------- | ---------------------------- |
| `sys.getsizeof(obj)`   | 对象直接内存大小             |
| `sys.getrefcount(obj)` | CPython 中的引用计数相关信息 |

一个关注大小。

一个关注引用。

它们完全不同。

而且 `sys.getrefcount()` 与 CPython 实现细节密切相关，不保证其他 Python 实现都提供相同接口或行为。

所以：

学习对象内存机制时可以使用，普通业务代码不要依赖它进行关键逻辑判断。

# 第十九部分：其他值得掌握的 `sys` 功能

前面讲的是核心内容。

下面把一些平时不是天天使用，但在调试、性能、模块管理或者跨平台开发中经常遇到的功能补充完整。

## 一、`sys.implementation`——当前 Python 的实现信息

很多人不知道：

Python 既是一门编程语言，也有不同的解释器实现。

例如：

CPython、PyPy 等。

最常见的官方 Python 实现是 CPython。

你可以查看：

```python
import sys

print(sys.implementation.name)
```

普通 CPython 中输出：

```text
cpython
```

也可以查看实现版本：

```python
print(sys.implementation.version)
```

为什么需要这个？

因为某些底层行为属于 CPython 的实现细节，其他 Python 解释器不一定采用相同机制。

例如：

引用计数、部分调试接口、底层内存结构。

如果写的是通用 Python 程序，应当优先使用语言和标准库保证的行为，而不是过度依赖某一种解释器的内部细节。

## 二、`sys.builtin_module_names`——查看内置模块名称

语法：

```python
sys.builtin_module_names
```

它是一个元组，包含编译进当前解释器的内置模块名称。

例如：

```python
import sys

print("sys" in sys.builtin_module_names)
```

在通常的 CPython 环境中会得到：

```text
True
```

但是它并不包含所有标准库模块。

例如：

```python
import json
```

`json` 属于标准库，但不意味着它一定是编译进解释器的内置模块。

## 三、`sys.stdlib_module_names`——查看标准库模块名称

Python 3.10 起提供：

```python
sys.stdlib_module_names
```

它是一个 `frozenset`，包含标准库模块名称。

例如：

```python
import sys

print("json" in sys.stdlib_module_names)
print("os" in sys.stdlib_module_names)
print("sys" in sys.stdlib_module_names)
```

通常都会得到：

```text
True
True
True
```

注意：

它记录标准库模块集合，并不代表这些模块都已经导入。

也不保证列表里的每个模块都能在所有操作系统上成功使用。

### 三个相关属性区别

| 属性                       | 代表什么                   |
| -------------------------- | -------------------------- |
| `sys.modules`              | 当前已加载的模块映射       |
| `sys.builtin_module_names` | 编译进解释器的内置模块名称 |
| `sys.stdlib_module_names`  | Python 标准库模块名称      |

这一组很容易在学习 Python 模块机制时混淆。

## 四、`sys.float_info`——浮点数相关信息

```python
import sys

print(sys.float_info)
```

会返回 Python 浮点数实现相关的信息。

例如：

```python
print(sys.float_info.max)
```

在常见的 IEEE 754 双精度浮点环境里，结果大约是：

```text
1.7976931348623157e+308
```

代表可表示的有限浮点数最大值。

再比如：

```python
print(sys.float_info.epsilon)
```

常见结果约为：

```text
2.220446049250313e-16
```

这是与浮点数精度有关的一个重要量。

它表示从 `1.0` 开始，下一个大于 `1.0` 的可表示浮点数与 `1.0` 之间的差值。

但要注意：

它不是任何浮点数计算都统一适用的绝对误差上限。

浮点数误差与数值大小、运算方式等因素有关。

## 五、`sys.int_info`——整数内部表示信息

例如：

```python
import sys

print(sys.int_info)
```

它提供整数底层表示相关的参数。

其中：

```python
sys.int_info.bits_per_digit
```

表示 Python 内部大整数表示中，每个“数字单元”承载的位数。

在一些常见 CPython 构建中，这个值可能是 `30`。

但这里的“digit”并不是十进制的一个数字字符。

它是解释器内部的大整数表示单位。

## 六、`sys.get_int_max_str_digits()`——整数与字符串转换的位数限制

Python 3.11 起提供：

```python
sys.get_int_max_str_digits()
```

用于查看整数与某些字符串形式转换时的最大数字位数限制。

例如：

```python
import sys

print(sys.get_int_max_str_digits())
```

在常见配置下，可能得到：

```text
4300
```

但可以通过启动配置等方式调整。

### 为什么 Python 要有这个限制？

因为非常庞大的十进制整数字符串转换可能消耗明显的 CPU 资源。

如果恶意输入包含极其巨大的数字，转换过程可能带来资源消耗问题。

所以 Python 提供了这项保护机制。

设置方法：

```python
sys.set_int_max_str_digits(5000)
```

注意：

它影响的是特定类型的整数和字符串转换，不代表 Python 整数计算本身最多只能有 5000 位。

而且：

```python
sys.set_int_max_str_digits(0)
```

可以禁用该限制，但这会取消相关保护，普通程序不建议这么做。

这与 `sys.maxsize` 是两个完全不同的概念。

## 七、`sys.getswitchinterval()` 和 `sys.setswitchinterval()`

这两个功能与解释器线程切换有关。

查看：

```python
import sys

print(sys.getswitchinterval())
```

某些常见配置中，可能返回：

```text
0.005
```

单位是秒。

也就是：

```text
5 毫秒
```

可以设置：

```python
sys.setswitchinterval(0.01)
```

表示把目标线程切换间隔调整为约 0.01 秒。

但这里特别需要强调：

这不是让操作系统强制每隔 0.01 秒切换一次线程。

实际线程调度受解释器构建、GIL 状态、操作系统调度和代码执行情况影响。

普通 Python 程序一般不需要修改这个值。

如果你遇到多线程性能问题，不应该首先想到调整它，而应该分析线程模型、任务类型和 GIL 等因素。

## 八、`sys._getframe()`——获取调用栈帧对象

注意方法名前面的下划线：

```python
sys._getframe()
```

这类接口往往更接近解释器实现细节。

例如：

```python
import sys


def hello():
    frame = sys._getframe()

    print(frame.f_code.co_name)


hello()
```

通常输出：

```text
hello
```

解释：

```python
sys._getframe()
```

取得当前调用栈帧对象。

```python
frame.f_code
```

取得对应代码对象。

```python
.co_name
```

取得代码对象的名称。

如果传入：

```python
sys._getframe(1)
```

表示尝试获取调用栈中上一层的帧对象。

如果深度超出实际栈范围，会抛出异常。

它适用于：

调试器、测试工具、调用栈分析、某些日志框架。

但普通代码不建议依赖它来决定业务逻辑。

因为这属于实现相关的功能，未必能在所有 Python 解释器中得到保证。

## 九、`sys.settrace()`——追踪 Python 代码执行

这个功能主要供调试器使用。

语法：

```python
sys.settrace(trace_func)
```

其中 `trace_func` 是追踪回调函数。

例如：

```python
import sys


def tracer(frame, event, arg):
    if event == "call":
        print("调用函数：", frame.f_code.co_name)

    return tracer


def hello():
    print("Hello")


sys.settrace(tracer)

try:
    hello()
finally:
    sys.settrace(None)
```

通常会包含类似：

```text
调用函数： hello
Hello
```

这里：

```python
event
```

表示追踪事件的种类，例如函数调用、返回等。

```python
sys.settrace(None)
```

用于关闭当前线程的追踪功能。

注意：

启用追踪可能显著影响程序性能。

它也涉及线程和事件处理的复杂机制。

所以日常调试优先使用 IDE 调试器，而不是自己手写完整的 `sys.settrace()` 框架。

与之相关的还有：

```python
sys.gettrace()
sys.setprofile()
sys.getprofile()
```

分别用于获取追踪或性能分析回调，以及安装相关回调。

## 十、`sys.monitoring`——较新的监控接口

Python 3.12 起引入了：

```python
sys.monitoring
```

这是为了支持更高效的执行事件监控接口。

它主要服务于：

调试器、代码覆盖率工具、性能分析器等。

它和 `sys.settrace()` 的用途存在交集，但接口和监控机制并不一样。

如果你只是学习普通 Python 编程，知道它的存在即可。

没有必要在每一个程序里使用它。

## 十一、`sys.audit()` 与 `sys.addaudithook()`——审计事件

Python 还提供审计钩子机制。

例如：

```python
sys.audit(event, *args)
```

可以触发一个审计事件。

而：

```python
sys.addaudithook(hook)
```

可以注册用于观察某些审计事件的回调。

举个简化例子：

```python
import sys


def hook(event, args):
    if event == "open":
        print("检测到文件打开事件", file=sys.stderr)


sys.addaudithook(hook)

with open("demo.txt", "w", encoding="utf-8") as f:
    f.write("Hello")
```

程序可以观察到相关的审计事件。

但是，审计钩子不是一个完整的安全沙箱。

特别是不能简单地认为：

“我添加了 `sys.addaudithook()`，所以不可信 Python 代码就被安全隔离了。”

Python 官方文档明确说明，这种机制不能替代真正的安全隔离。

Python 3.15.2 documentation

## 十二、`sys._is_gil_enabled()`——检查 GIL 状态

在支持这个接口的 CPython 版本中，可以写：

```python
import sys

if hasattr(sys, "_is_gil_enabled"):
    print(sys._is_gil_enabled())
```

返回布尔值，表示当前 GIL 是否启用。

这与近年来 CPython 的自由线程构建（free-threaded build）有关。

需要区分：

解释器是否支持自由线程，与当前 GIL 是否启用，是两个不同层面的信息。

从 Python 3.15 起还可以了解：

```python
sys.abi_info.free_threaded
```

它表示当前解释器构建是否具有自由线程支持。

这些都属于较新的、与解释器实现紧密相关的功能，不应假设旧 Python 或其他实现都支持。

## 十三、Python 3.15 新增的懒加载导入相关接口

Python 3.15 引入了显式懒加载导入等新特性，并在 `sys` 中提供相关配置和观察接口。

例如：

```python
sys.get_lazy_imports()
sys.set_lazy_imports(mode)
sys.lazy_modules
```

其中：

```python
sys.get_lazy_imports()
```

可以查看当前懒加载导入模式。

```python
sys.set_lazy_imports("normal")
```

表示使用普通模式，仅让明确标记为懒加载的导入采用相应行为。

```python
sys.set_lazy_imports("all")
```

则允许更多顶层导入采用潜在懒加载方式。

但这是影响整个解释器导入行为的高级功能。

它可能改变模块初始化与副作用发生的时机，因此不建议普通项目随便全局开启。

初学者应该先彻底掌握普通 `import`、包结构和 `sys.path`，再去研究懒加载导入。

# 第二十部分：`sys` 与其他标准库的常见搭配

理解 `sys` 最有效的方法，不是孤立地背方法，而是学会和其他模块组合。

## 一、`sys` + `os`：检查程序运行环境

```python
import os
import sys

print("Python 版本：", sys.version)
print("解释器路径：", sys.executable)
print("操作系统平台：", sys.platform)
print("当前工作目录：", os.getcwd())
print("命令行参数：", sys.argv)
```

这是一种很实用的环境诊断组合。

例如，当程序在自己电脑上能够运行，但在朋友的电脑上报错时，可以先打印这些信息。

你就能检查双方的运行环境有什么差异。

## 二、`sys` + `pathlib`：稳定地定位项目文件

假设项目：

```text
project/
├── main.py
└── data/
    └── config.txt
```

有些初学者会写：

```python
open("data/config.txt")
```

但这个相对路径依赖进程当前工作目录。

如果从不同地方启动脚本，就可能找不到文件。

更稳健的写法：

```python
from pathlib import Path

base_dir = Path(__file__).resolve().parent
file_path = base_dir / "data" / "config.txt"

with file_path.open("r", encoding="utf-8") as f:
    print(f.read())
```

如果还希望输出解释器信息：

```python
import sys

print("运行解释器：", sys.executable)
print("配置文件位置：", file_path)
```

这就是 `sys` 与 `pathlib` 非常典型的配合方式。

## 三、`sys` + `subprocess`：使用当前解释器执行命令

例如，你希望检查当前解释器的 pip 版本：

```python
import subprocess
import sys

subprocess.run(
    [sys.executable, "-m", "pip", "--version"],
    check=True
)
```

这里每个部分的含义：

```python
sys.executable
```

当前 Python 可执行文件。

```python
"-m"
```

告诉 Python 按模块方式执行。

```python
"pip"
```

运行 `pip` 模块。

```python
"--version"
```

传给 pip 的参数。

这在多 Python 版本共存时特别有用。

如果你遇到：

```text
ModuleNotFoundError: No module named 'requests'
```

可以先执行：

```python
import subprocess
import sys

print("当前解释器：", sys.executable)

subprocess.run(
    [sys.executable, "-m", "pip", "show", "requests"]
)
```

它可以帮助判断：

当前解释器关联的环境里是否安装了 `requests`。

不过，这依赖该环境中的 pip 可用。

## 四、`sys` + `json`：输出可被其他程序读取的数据

如果你要编写命令行工具，并让它输出 JSON：

```python
import json
import sys

data = {
    "name": "Alice",
    "age": 18,
    "status": "success"
}

json.dump(
    data,
    sys.stdout,
    ensure_ascii=False
)

sys.stdout.write("\n")
```

这里：

```python
json.dump(...)
```

直接把 JSON 内容写入指定文件对象。

```python
sys.stdout
```

表示输出到标准输出。

```python
ensure_ascii=False
```

允许中文等非 ASCII 字符在输出中保持原样，而不强制转成 Unicode 转义形式。

为什么这种搭配很有用？

因为你可以执行：

```bash
python app.py > result.json
```

将 JSON 数据保存到文件。

而且，如果发生错误，应该把错误提示输出到：

```python
sys.stderr
```

这样正常 JSON 不会混入诊断文本。

# 第二十一部分：完整综合案例——制作一个真正可用的命令行计算工具

前面讲了大量 API。

现在用一个实际小项目把最重要的知识串联起来。

## 一、项目目标

创建一个程序：

```text
calculator.py
```

让用户能够在终端输入：

```bash
python calculator.py 10 20 30
```

得到：

```text
数字个数：3
总和：60
平均值：20
```

如果用户没有提供参数，程序应该给出提示。

如果用户输入非法数据，也应该清楚地报告错误。

## 二、完整代码

```python
import sys


def main():
    # 1. 获取用户输入的全部命令行参数
    args = sys.argv[1:]

    # 2. 检查是否提供参数
    if not args:
        print(
            "用法：python calculator.py 数字1 数字2 ...",
            file=sys.stderr
        )
        return 2

    # 3. 将字符串参数转换为浮点数
    try:
        numbers = [float(arg) for arg in args]

    except ValueError:
        print(
            "错误：所有参数都必须是数字",
            file=sys.stderr
        )
        return 2

    # 4. 执行计算
    total = sum(numbers)
    average = total / len(numbers)

    # 5. 输出结果
    print(f"数字个数：{len(numbers)}")
    print(f"总和：{total:g}")
    print(f"平均值：{average:g}")

    # 6. 返回成功状态
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## 三、逐步拆解

### 第一步：获取命令行参数

```python
args = sys.argv[1:]
```

假设执行：

```bash
python calculator.py 10 20 30
```

那么：

```python
sys.argv
```

类似：

```text
['calculator.py', '10', '20', '30']
```

而：

```python
sys.argv[1:]
```

得到：

```text
['10', '20', '30']
```

排除了脚本名称。

### 第二步：检查参数是否为空

```python
if not args:
```

因为空列表：

```python
[]
```

在布尔判断中为 `False`。

所以：

```python
not []
```

得到 `True`。

这就可以判断用户是否没有提供数据。

### 第三步：转换数据类型

```python
numbers = [float(arg) for arg in args]
```

这里使用列表推导式。

相当于：

```python
numbers = []

for arg in args:
    numbers.append(float(arg))
```

输入：

```text
['10', '20', '30']
```

转换为：

```text
[10.0, 20.0, 30.0]
```

如果某个参数不能转换为浮点数：

```python
float("abc")
```

就会触发 `ValueError`。

因此：

```python
except ValueError:
```

可以捕获这个错误。

### 第四步：计算结果

```python
total = sum(numbers)
```

计算总和。

```python
average = total / len(numbers)
```

计算平均数。

因为前面已经验证列表非空，所以此处不会因为元素数量为零而发生除零错误。

### 第五步：格式化输出

```python
print(f"总和：{total:g}")
```

这里的：

```text
:g
```

是 Python 格式化语法中的通用数字格式。

例如：

```text
60.0
```

可能显示为：

```text
60
```

使输出更简洁。

但它也有默认精度规则，不适合所有需要严格保留小数位数的场景。

### 第六步：向操作系统反馈结果

```python
return 0
```

代表成功。

```python
return 2
```

代表参数错误。

最后：

```python
sys.exit(main())
```

把这些状态转换为进程退出状态。

## 四、交互式测试

运行效果模拟器

交互练习

正常输入非法输入无参数

自己试着输入一组参数（空格分隔）

终端命令

```bash
$ python calculator.py 10 20 30
```

程序输出

```text
数字个数：3
总和：60
平均值：20
```

退出状态码

```text
0
```

这里模拟程序逻辑，不会真实启动 Python。输入按空格拆分；实际命令行还支持引号及 shell 特定的参数解析规则。

这个小程序同时运用了：

`sys.argv`、`sys.stderr`、`sys.exit()`，以及列表切片、异常处理、函数返回值、格式化输出。

把这个例子弄懂，你就已经能够编写不少实用的 Python 命令行小工具了。

# 第二十二部分：10 个常见误区与重要注意事项

这部分建议重点阅读，因为不少错误不在于不知道 API，而在于对它的行为理解有偏差。

| 常见误区                                          | 正确理解                               |
| ------------------------------------------------- | -------------------------------------- |
| `sys.argv[0]` 一定是完整脚本路径                  | 不一定，取决于启动方式与平台           |
| `sys.argv` 中的数字自动是 `int`                   | 不是，命令行参数是字符串               |
| `sys.path` 包含全部导入机制                       | 不是，它主要与路径搜索有关             |
| 修改 `sys.path` 就能解决所有导入错误              | 不一定，可能是解释器、包结构或环境问题 |
| `sys.exit()` 会强行杀掉整个进程                   | 它通过 `SystemExit` 发起退出           |
| `sys.platform == "win32"` 说明是 32 位 Python     | 错误，它是 Windows 平台标识            |
| `sys.maxsize` 是 Python 整数最大值                | 错误，普通 `int` 可以更大              |
| `sys.getsizeof()` 是整个对象结构的内存总量        | 不是，主要统计对象直接归属的内存       |
| `sys.stdout.write()` 自动换行                     | 不会，需自己加入 `\n`                  |
| `sys.getdefaultencoding()` 就是任意文本文件的编码 | 不是，文件内容编码需要单独确定         |

还有几个容易遗漏的重要细节。

### 1. `sys.exit()` 在子线程中的行为不同

例如：

```python
import sys
import threading


def worker():
    print("子线程开始")
    sys.exit(1)


t = threading.Thread(target=worker)
t.start()
t.join()

print("主线程继续运行")
```

这里的 `sys.exit()` 只会使该线程因 `SystemExit` 退出，并不会因为它写在子线程里，就自动结束整个 Python 进程。

因此，不能把 `sys.exit()` 当成跨线程的进程终止开关。

### 2. `sys.stdout` 等对象可能被替换

比如在 Notebook、测试框架或其他嵌入环境中：

```python
sys.stdout
```

可能不是普通终端输出流。

所以：

```python
sys.stdout.fileno()
```

或者：

```python
sys.stdout.buffer
```

不一定适用于所有环境。

某些没有连接控制台的 Windows GUI Python 进程中，标准流甚至可能为 `None`。

### 3. 不要随便覆盖 `sys` 的关键属性

例如：

```python
sys.path = []
```

可能导致原本可以导入的模块无法找到。

再比如：

```python
sys.stdout = None
```

可能使原本依赖标准输出的代码出现错误。

这些属性虽然有些允许修改，但并不意味着可以没有后果地修改。

### 4. 修改 `sys.path` 也可能产生安全问题

如果你把一个不可信目录放到导入路径前面，里面正好有与目标库同名的文件，程序可能错误地加载那个文件。

因此，不要为了解决导入问题随便添加不可信的目录。

### 5. 区分 Python 语言特性与 CPython 实现细节

例如：

```python
sys.version_info
```

属于通用性很强的版本信息接口。

但：

```python
sys._getframe()
sys.getrefcount()
```

与实现细节关系更紧密。

如果代码需要兼容 PyPy 或其他 Python 实现，应当特别检查兼容性。

# 第二十三部分：`sys` 常用方法与属性速查表

下面把前面讲解的内容整理到一张表里，方便之后复习。

| 功能               | 语法                          | 主要用途                 |
| ------------------ | ----------------------------- | ------------------------ |
| 命令行参数         | `sys.argv`                    | 读取脚本参数             |
| 原始启动参数       | `sys.orig_argv`               | 获取含解释器选项的参数   |
| 退出程序           | `sys.exit(code)`              | 请求程序退出             |
| 模块搜索路径       | `sys.path`                    | 查看或修改搜索路径       |
| 模块缓存           | `sys.modules`                 | 查看已加载模块           |
| Python 版本        | `sys.version`                 | 查看详细版本字符串       |
| 版本信息           | `sys.version_info`            | 判断 Python 版本         |
| 解释器位置         | `sys.executable`              | 获取当前解释器路径       |
| 操作系统标识       | `sys.platform`                | 判断平台                 |
| 环境前缀           | `sys.prefix`                  | 获取当前环境前缀         |
| 基础环境前缀       | `sys.base_prefix`             | 获取基础 Python 前缀     |
| 标准输入           | `sys.stdin`                   | 读取输入流               |
| 标准输出           | `sys.stdout`                  | 输出正常数据             |
| 标准错误           | `sys.stderr`                  | 输出错误信息             |
| 刷新标准输出       | `sys.stdout.flush()`          | 刷新缓冲                 |
| 最大大小整数       | `sys.maxsize`                 | 获取 `Py_ssize_t` 最大值 |
| 字节顺序           | `sys.byteorder`               | 获取大小端信息           |
| 对象大小           | `sys.getsizeof(obj)`          | 查看直接内存大小         |
| 递归限制           | `sys.getrecursionlimit()`     | 查看递归深度限制         |
| 修改递归限制       | `sys.setrecursionlimit(n)`    | 设置递归限制             |
| 当前异常           | `sys.exception()`             | 获取正在处理的异常       |
| 异常信息           | `sys.exc_info()`              | 获取异常三元组           |
| 异常处理钩子       | `sys.excepthook`              | 自定义未处理异常报告     |
| 默认字符串编码     | `sys.getdefaultencoding()`    | 获取默认字符串编码       |
| 文件系统编码       | `sys.getfilesystemencoding()` | 查看路径编码             |
| 启动选项           | `sys.flags`                   | 查看解释器启动状态       |
| 禁止字节码缓存写入 | `sys.dont_write_bytecode`     | 控制 `.pyc` 写入         |
| 字符串驻留         | `sys.intern(s)`               | 驻留字符串               |
| 引用计数           | `sys.getrefcount(obj)`        | CPython 引用分析         |
| 解释器实现         | `sys.implementation`          | 查看 CPython 等实现信息  |
| 内置模块名         | `sys.builtin_module_names`    | 查看编译进解释器的模块   |
| 标准库模块名       | `sys.stdlib_module_names`     | 查看标准库模块名称       |
| 浮点数信息         | `sys.float_info`              | 获取浮点表示参数         |
| 整数内部信息       | `sys.int_info`                | 获取整数内部表示参数     |
| 调用栈帧           | `sys._getframe()`             | 调试、栈分析             |
| 追踪执行           | `sys.settrace()`              | 调试代码执行             |
| 审计事件           | `sys.audit()`                 | 触发审计事件             |

其他较少使用、但有时会在工具开发中见到的接口，包括 `sys.getallocatedblocks()`（内部内存块统计）、`sys.hash_info`（哈希实现参数）、`sys.maxunicode`（最大 Unicode 码点）、`sys.thread_info`（线程实现信息）、`sys.is_finalizing()`（解释器是否正在结束）、`sys.getwindowsversion()`（Windows 专用）以及 `sys._xoptions`（CPython 的 `-X` 选项信息）。

这些不需要一开始全部背下来，遇到对应的调试或平台需求再深入即可。

# 第二十四部分：建议你按什么顺序掌握 `sys`？

虽然我们已经介绍了很多内容，但它们的重要程度和日常使用频率并不一样。

我建议按下面的顺序学习。

### 学习进度清单

**已掌握：0 / 20 项（0%）**

- [ ] **第一阶段：基础必会**
  - [ ] `sys.argv`
  - [ ] `sys.exit()`
  - [ ] `sys.path`
  - [ ] `sys.version_info`
  - [ ] `sys.executable`

- [ ] **第二阶段：实战常用**
  - [ ] `sys.stdin` / `sys.stdout` / `sys.stderr`
  - [ ] `sys.platform`
  - [ ] `sys.modules`
  - [ ] `sys.prefix` / `sys.base_prefix`
  - [ ] `sys.getsizeof()`

- [ ] **第三阶段：进阶理解**
  - [ ] `sys.getrecursionlimit()`
  - [ ] `sys.setrecursionlimit()`
  - [ ] `sys.exception()` / `sys.exc_info()`
  - [ ] `sys.flags`
  - [ ] `sys.byteorder`

- [ ] **第四阶段：工具与底层**
  - [ ] `sys.intern()` / `sys.getrefcount()`
  - [ ] `sys._getframe()` / `sys.settrace()`
  - [ ] `sys.excepthook`
  - [ ] `sys.monitoring`
  - [ ] `sys.audit()`

**重置进度**（原交互按钮，Markdown 中保留文字）

如果你现在只想真正熟练掌握最有实际价值的内容，我建议优先练习下面三个程序。

练习一：命令行参数

写一个程序：

```bash
python greet.py Alice 18
```

能够输出：

```text
你好，Alice
你的年龄是 18 岁
```

要求使用：

```python
sys.argv
```

并且能正确处理缺少参数的情况。

练习二：模块导入排错

创建两个模块文件，尝试从不同目录启动程序，观察：

```python
sys.path
```

的变化。

再结合：

```python
os.getcwd()
```

和：

```python
__file__
```

理解工作目录、文件位置、模块搜索路径之间的关系。

练习三：标准输入输出

写一个程序，从标准输入读取多行数字，计算总和。

然后尝试：

```bash
python sum.py < numbers.txt > result.txt
```

并让输入错误时的信息通过：

```python
sys.stderr
```

输出。

完成这三个练习，你就会真正体会到 `sys` 在命令行程序中的实际作用。

## 最后总结

学习 `sys` 最重要的不是死记所有方法，而是建立下面这套思维：

需要知道 Python 是怎么启动、怎么运行的，就考虑 `sys.argv`、`sys.version_info`、`sys.executable` 和 `sys.flags`。

需要理解 Python 怎么寻找模块，就考虑 `sys.path`、`sys.modules`。

需要让程序与终端交互，就考虑 `sys.stdin`、`sys.stdout`、`sys.stderr` 和 `sys.exit()`。

需要分析解释器运行细节，就考虑 `sys.getsizeof()`、`sys.getrecursionlimit()`、`sys.exception()` 等。

而在涉及文件目录、环境变量、进程管理时，再与 `os`、`pathlib`、`subprocess` 等标准库配合。

这样，`sys` 就不再是一堆需要背诵的零散名称，而是一套你知道什么时候该拿出来使用的工具。

完整、权威的接口参考可以直接查阅 [Python 官方文档：sys — System-specific parameters and functions](https://docs.python.org/3/library/sys.html?utm_source=chatgpt.com)。