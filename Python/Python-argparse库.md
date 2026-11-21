# Python `argparse` 库超详细教程：从命令行参数，到能独立使用的小工具

## 零基础先导：命令行参数到底从哪里来？

### 先区分“启动程序时的参数”和“程序运行后的输入”

你在终端执行 `python hello.py Alice`，操作系统启动 Python 解释器，并把命令行上的字符串传递给它。Python 再通过 `sys.argv` 把这些字符串提供给脚本；**这里并没有自动调用 `input()` 或 `cin`**。

```python
# hello.py
import sys

print(sys.argv)       # 例如 ['hello.py', 'Alice']
print(sys.argv[1])    # Alice；没有第二项时会 IndexError
```

真正麻烦的不是读取一个字符串，而是用户可能传 `--count 3`、漏传必需参数、输入 `--help`、把数字写成英文、调换选项顺序。你当然可以自己写几十个 `if` 和字符串判断，但 `argparse` 就是为这类常见“解析命令行接口”提供统一规则的标准库。

### 从三行开始，不要先背一长张参数表

```python
import argparse                         # 标准库，无需 pip 安装
parser = argparse.ArgumentParser()       # 创建“命令行规则表”
parser.add_argument("name")              # 规定启动时需要一个位置参数
args = parser.parse_args()               # 真正读取并检查本次命令行
print("你好，", args.name)                 # 从结果对象取出 name
```

保存为 `hello.py`，运行 `python hello.py Alice`。`add_argument` 是**声明规则**，不是让程序当场等待键盘输入；`parse_args()` 才是解析 `sys.argv` 的动作。`args.name` 是解析结果的属性；少了位置参数时，它会提示用法并退出，不需要自己写 `sys.argv[1]` 的下标判断。

把 `add_argument("--count", type=int, default=1)` 想成“可选的、默认一次、接收整数的次数参数”。`type=int` 是转换与基本验证，不代表数值一定大于零；业务限制仍需自己检查。后文讲 `store_true`、`nargs`、互斥组时，也始终在回答：**这个程序希望用户怎样启动？输入不符合规则怎么办？**

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

假设你已经写好了一个程序，可以读取文件、统计行数，再输出结果。

现在你不想每次都打开源代码修改文件名，而是希望这样运行：

```bash
python count_lines.py notes.txt
```

以后还想加上：

```bash
python count_lines.py notes.txt --encoding utf-8 --skip-empty
```

这些跟在脚本名后面的内容，怎样进入 Python？

哪些参数必须提供，哪些可以省略？

用户输入错了，怎样显示清楚的提示？

这就是 `argparse` 经常解决的问题。

本篇以 Python 3.11 及以上的常用接口为基础，新版本功能会明确说明。

示例从一两个参数逐步增加，不要求你一开始就看懂一个复杂命令行工具。

## 第一章：为什么已经有 `sys.argv`，还需要 `argparse`？

### 1.1 先回顾参数从哪里来

创建 `greet.py`：

```python
import sys

print(sys.argv)
```

在终端中执行：

```bash
python greet.py Alice 20
```

通常输出：

```text
['greet.py', 'Alice', '20']
```

`sys.argv` 保存脚本收到的参数字符串。

其中索引零通常是脚本名称或相应启动标识，后面才是用户提供的参数。

我们可以自己取值：

```python
name = sys.argv[1]
age = int(sys.argv[2])
```

对于非常简单的练习，这样已经能工作。

### 1.2 如果用户没有输入年龄呢？

执行：

```bash
python greet.py Alice
```

程序直接访问 `sys.argv[2]`，就会因为列表里没有这个位置而报错。

于是，你要先检查参数数量。

### 1.3 如果年龄写成 `twenty` 呢？

```bash
python greet.py Alice twenty
```

`int("twenty")` 会抛出 `ValueError`。

于是，你还需要解释哪一个参数错了，以及它应该是什么类型。

### 1.4 如果用户想知道程序怎么用呢？

他可能执行：

```bash
python greet.py --help
```

你需要自己识别 `--help`，再打印说明。

接着你又想支持：

```bash
python greet.py --name Alice --age 20
```

这时不能只靠固定索引了，因为参数顺序可能交换。

还要处理选项后没有值、重复选项、不认识的选项、默认值，以及多个互斥开关。

这些都是命令行程序反复遇到的工作。

`argparse` 把这套常见规则组织成一个标准库接口：你声明程序接受什么，解析器按规则解释用户输入。[官方定位与基本流程](https://docs.python.org/3.14/library/argparse.html)

### 1.5 名称怎么理解？需要安装吗？

`arg` 来自 argument，表示参数。

`parse` 表示解析，也就是按照规则分析输入。

`argparse` 可以理解为“参数解析”。

它属于标准库，直接导入：

```python
import argparse
```

不需要安装一个同名第三方包。

它也不会自动完成文件读取、网络请求或业务计算。

解析参数之后，真正的工作仍然由你的程序执行。

## 第二章：第一个完整例子，先只接收一个名字

### 2.1 创建程序

把下面内容保存为 `greet.py`：

```python
import argparse

parser = argparse.ArgumentParser(description="向指定的人打招呼")
parser.add_argument("name", help="要问候的名字")

args = parser.parse_args()

print(f"你好，{args.name}！")
```

运行：

```bash
python greet.py Alice
```

输出：

```text
你好，Alice！
```

先不要急着复制更多选项，我们逐行拆开。

### 2.2 `ArgumentParser` 是什么？

```python
parser = argparse.ArgumentParser(description="向指定的人打招呼")
```

`ArgumentParser` 是类，用来创建一个参数解析器对象。

变量 `parser` 保存这个对象。

你后面会通过它登记参数规则，再让它执行解析。

`description` 是程序用途说明，主要显示在帮助信息中。

它不会自动改变程序功能。

### 2.3 `add_argument("name")` 做了什么？

```python
parser.add_argument("name", help="要问候的名字")
```

这一步向解析器登记规则：程序需要一个叫 `name` 的位置参数。

所谓位置参数，是根据它在命令行中的位置解释的参数。

例如：

```bash
python greet.py Alice
```

`Alice` 就被放进 `name`。

这行登记规则本身不读取当前命令行，也不会立刻向用户询问输入。

真正解析发生在下一步。

### 2.4 `parse_args()` 从哪里取参数？

```python
args = parser.parse_args()
```

普通脚本中不传实参时，它通常读取 `sys.argv[1:]`。

也就是跳过脚本名称，只解析用户参数。

解析器检查输入是否符合前面登记的规则，然后返回一个 `Namespace` 对象。

### 2.5 `args.name` 为什么可以用点号取值？

`Namespace` 是保存解析结果的对象。

这里登记了 `name`，所以结果对象具有相应属性。

```python
print(args)
```

通常显示：

```text
Namespace(name='Alice')
```

再写：

```python
args.name
```

取得的就是字符串：

```text
Alice
```

`args` 不是关键字，`name` 也不是内置字段。

它们分别来自你选择的变量名和参数定义。

### 2.6 如果用户没提供名字呢？

运行：

```bash
python greet.py
```

解析器通常在终端显示类似：

```text
usage: greet.py [-h] name
greet.py: error: the following arguments are required: name
```

程序以参数错误状态结束，后面的问候不会执行。

实际帮助措辞、颜色和排版可能随 Python 版本变化。

重点是：缺少必填参数被识别，错误指出缺少哪一项。

### 2.7 `--help` 是不是还要自己写？

默认已经有。

```bash
python greet.py --help
```

通常显示使用方式、程序描述、位置参数，以及 `-h`、`--help` 说明。

显示帮助后，解析器通常以成功状态退出，不再执行正常业务。

这与输入错误时通常使用退出状态二不同。

## 第三章：位置参数和带名字的选项，到底区别在哪里？

### 3.1 再加一个年龄

```python
import argparse

parser = argparse.ArgumentParser(description="显示问候信息")
parser.add_argument("name")
parser.add_argument("age", type=int)
args = parser.parse_args()

print(f"{args.name}，你今年 {args.age} 岁。")
```

运行：

```bash
python greet.py Alice 20
```

输出：

```text
Alice，你今年 20 岁。
```

第一个用户参数对应姓名，第二个对应年龄。

### 3.2 我交换顺序行不行？

执行：

```bash
python greet.py 20 Alice
```

解析器仍然按位置解释。

`20` 被当作姓名，`Alice` 被当作年龄。

因为年龄要求整数，转换失败。

位置参数本身不携带字段名字，因此顺序是协议的一部分。

### 3.3 如果希望选项顺序更灵活呢？

可以使用带前缀的选项：

```python
import argparse

parser = argparse.ArgumentParser(description="显示问候信息")
parser.add_argument("--name", required=True)
parser.add_argument("--age", type=int, required=True)
args = parser.parse_args()

print(args.name)
print(args.age)
```

下面两条都可以表达相同输入：

```bash
python greet.py --name Alice --age 20
```

```bash
python greet.py --age 20 --name Alice
```

选项名称帮助解析器识别它对应哪一项。

### 3.4 叫“可选参数”，为什么还能 `required=True`？

很多文档把带 `-` 或 `--` 的条目称为 optional argument。

这个分类主要描述它的选项形式，并不意味着你永远不能要求它出现。

`required=True` 可以要求某个选项必须提供。

不过，设计命令行时也应考虑用户习惯：最核心且总是必需的输入文件，经常适合做位置参数；编码、显示格式等经常适合做有默认值的选项。

### 3.5 短选项和长选项能指向同一个结果吗？

```python
parser.add_argument("-n", "--name", required=True)
```

这样可以使用：

```bash
python greet.py -n Alice
```

或者：

```bash
python greet.py --name Alice
```

它们是同一项的不同入口，不会自动产生两个独立字段。

长选项通常更容易记住，短选项适合频繁手动输入。

## 第四章：`type=int`、`choices` 和范围验证

### 4.1 为什么命令行里写了数字，仍然需要类型转换？

终端传给脚本的普通参数首先是字符串。

```bash
python app.py --count 10
```

这里的 `10` 并不会因为外观像数字就自动成为整数。

定义：

```python
parser.add_argument("--count", type=int, default=1)
```

表示解析到相应参数时，调用 `int` 转换它。

请注意写的是函数对象 `int`，不是 `int()`。

### 4.2 `type=int()` 为什么不对？

`int()` 会在登记规则时先执行，得到整数零。

于是你传给 `type` 的是一个数值，不是可调用的转换器。

我们需要的是“以后收到文本，使用哪个函数转换”，因此传函数本身。

同样的道理适用于：

```python
parser.add_argument("--ratio", type=float)
```

### 4.3 如果值不能转换，解析器怎么处理？

假设参数定义为整数：

```python
parser.add_argument("--count", type=int)
```

用户输入：

```bash
python app.py --count many
```

解析器会把转换失败报告为参数错误，并指出 `--count` 的值不合法。

调用方通常不必自己围着每个 `int()` 再写同样的异常处理。

### 4.4 能转成整数，不代表数值符合要求

`-100` 可以成功转换成整数。

但如果它代表下载次数，就可能不符合业务需求。

可以自定义转换器：

```python
import argparse


def positive_int(text):
    try:
        value = int(text)
    except ValueError as error:
        raise argparse.ArgumentTypeError("必须是整数") from error

    if value <= 0:
        raise argparse.ArgumentTypeError("必须大于零")

    return value


parser = argparse.ArgumentParser()
parser.add_argument("--count", type=positive_int, default=1)
args = parser.parse_args()
print(args.count)
```

这个转换器先转换类型，再检查范围，最后返回业务需要的值。

`ArgumentTypeError` 让解析器把错误组织成正常的命令行提示。

### 4.5 只有几个合法值，可以用 `choices`

```python
parser.add_argument("--format", choices=["text", "json"], default="text")
```

用户可以选择 `text` 或 `json`。

写成 `yaml` 就会被拒绝，并得到可用值提示。

帮助信息里也通常会展示这些选择。

### 4.6 类型转换和 choices，谁先发生？

先转换，再检查选择。

```python
parser.add_argument("--level", type=int, choices=[1, 2, 3])
```

`choices` 中应该放转换后的整数。

如果写成字符串列表 `["1", "2", "3"]`，输入先变成整数，再与字符串比较，就不符合原先预期。

### 4.7 默认值也会被检查 choices 吗？

不要依赖 `choices` 自动把所有程序内置默认值都验证一遍。

默认值的处理与用户实际输入的转换路径不完全相同。

你应该给出本来就合法的默认值，并在测试中覆盖无参数调用。

例如选项只允许 `text` 与 `json`，就不要把默认值写成 `csv`，再以为解析器一定替你发现配置错误。

## 第五章：`default`、`required`、`dest` 和 `metavar`

### 5.1 如果用户没提供选项，我想使用默认值呢？

```python
parser.add_argument("--encoding", default="utf-8")
```

没有 `--encoding` 时，结果使用 `utf-8`。

用户明确提供值时，使用用户输入。

`default` 是“缺少这项输入时使用什么”，不会覆盖用户实际提供的空字符串或其他合法值。

### 5.2 默认值没有设置时是什么？

常见的普通存值选项默认是 `None`。

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--output")
print(parser.parse_args([]))
```

通常得到：

```text
Namespace(output=None)
```

它和输出路径为空字符串不是同一个状态。

### 5.3 字符串默认值的转换要留意

```python
parser.add_argument("--count", type=int, default="10")
```

字符串默认值会按相应规则经过类型转换。

而非字符串默认值通常直接使用，不应该假设解析器会对任意对象执行同样转换。

为减少意外，整数选项通常直接提供整数默认值：

```python
parser.add_argument("--count", type=int, default=10)
```

程序内部生成的配置仍然要由你保证正确。

### 5.4 带连字符的名字，结果属性怎么访问？

```python
parser.add_argument("--page-size", type=int, default=20)
```

通常访问：

```python
args.page_size
```

长选项中的连字符被转换成下划线，适合 Python 属性名称。

不能写 `args.page-size`，那会被 Python 当成减法相关表达式解析。

### 5.5 如果想自己指定结果属性名呢？

用 `dest`：

```python
parser.add_argument("-o", "--output-file", dest="output_path")
```

结果访问：

```python
args.output_path
```

`dest` 改的是存储属性名，不是用户输入的选项拼写。

### 5.6 `metavar` 又改什么？

```python
parser.add_argument("--output", metavar="PATH")
```

帮助里通常显示类似：

```text
--output PATH
```

`PATH` 是给人看的占位符，不会真的变成参数值，也不会把结果属性改名为 `PATH`。

所以：

`dest` 影响结果怎么取。

`metavar` 影响帮助怎么展示。

`help` 解释这个参数是什么。

三个选项各有用途。

## 第六章：布尔开关，为什么不能直接写 `type=bool`？

### 6.1 我想让用户加上 `--verbose` 就显示详情

可以定义：

```python
parser.add_argument("--verbose", action="store_true", help="显示详细信息")
```

不写这个开关时，通常得到 `False`。

写了开关，得到 `True`。

完整观察：

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--verbose", action="store_true")

print(parser.parse_args([]))
print(parser.parse_args(["--verbose"]))
```

输出：

```text
Namespace(verbose=False)
Namespace(verbose=True)
```

这里传入列表，是为了直接在代码中演示不同输入，后面会详细说明。

### 6.2 `action` 是什么意思？

普通参数默认把一个值存下来。

`action` 决定解析器遇到这个参数时执行什么存储动作。

`store_true` 不需要再接一个普通值，它只表示“出现即为真”。

所以：

```bash
python app.py --verbose
```

已经足够。

通常不应该继续写：

```bash
python app.py --verbose true
```

后面的 `true` 可能变成不认识的额外输入。

### 6.3 那 `store_false` 呢？

它表达相反方向：默认通常为真，出现时存为假。

```python
parser.add_argument("--no-cache", dest="cache", action="store_false")
```

未提供时，`args.cache` 通常为 `True`。

提供 `--no-cache` 后为 `False`。

选项名字应该帮助用户理解动作，避免一个叫 `--enable-feature` 的开关却使用 `store_false` 产生困惑。

### 6.4 如果想同时支持开启和关闭形式呢？

Python 3.9 起提供 `BooleanOptionalAction`：

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument(
    "--cache",
    action=argparse.BooleanOptionalAction,
    default=True,
    help="是否使用缓存",
)

print(parser.parse_args([]))
print(parser.parse_args(["--cache"]))
print(parser.parse_args(["--no-cache"]))
```

通常得到：

```text
Namespace(cache=True)
Namespace(cache=True)
Namespace(cache=False)
```

一个定义提供正反两种明确入口，比要求用户猜测 `false` 应该如何拼写更方便。

### 6.5 为什么 `type=bool` 容易出错？

你可能写：

```python
parser.add_argument("--debug", type=bool)
```

然后运行：

```bash
python app.py --debug false
```

希望结果为假。

但 `bool("false")` 在 Python 中是 `True`，因为它是非空字符串。

这里调用的是 Python 对字符串的真假转换，并不是英文单词含义解释器。

所以，`type=bool` 很容易把用户想关闭功能的输入解释为开启。

### 6.6 如果接口确实要求显式输入 true 或 false 呢？

你可以写一个只接受约定词的转换器：

```python
import argparse


def parse_bool(text):
    normalized = text.casefold()
    if normalized in {"true", "yes", "1"}:
        return True
    if normalized in {"false", "no", "0"}:
        return False
    raise argparse.ArgumentTypeError("请输入 true/false、yes/no 或 1/0")


parser = argparse.ArgumentParser()
parser.add_argument("--debug", type=parse_bool, default=False)
```

这套可接受词是你制定的约定，不是所有命令行工具统一使用的规则。

如果不需要显式值，优先使用布尔开关，通常更自然。

## 第七章：一个参数接收几个值？理解 `nargs`

### 7.1 普通选项默认接收一个值

```python
parser.add_argument("--output")
```

典型调用：

```bash
python app.py --output report.txt
```

`args.output` 是字符串。

如果你希望一个参数接收多个值，就需要定义数量规则。

`nargs` 可以理解为 number of arguments，也就是接收的参数数量。

### 7.2 固定两个值

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--size", nargs=2, type=int, metavar=("WIDTH", "HEIGHT"))
args = parser.parse_args(["--size", "800", "600"])
print(args.size)
```

输出：

```text
[800, 600]
```

这里结果是列表，即使只定义一个叫 `size` 的选项。

两个输入分别转换为整数。

缺少其中一项就不符合数量要求。

### 7.3 `nargs=1` 和默认有区别吗？

有。

```python
import argparse

first = argparse.ArgumentParser()
first.add_argument("--name")

second = argparse.ArgumentParser()
second.add_argument("--name", nargs=1)

print(first.parse_args(["--name", "Alice"]).name)
print(second.parse_args(["--name", "Alice"]).name)
```

通常输出：

```text
Alice
['Alice']
```

默认单值规则返回一个值，显式固定数量规则返回对应列表。

不要只因为数量都是一，就假设结果类型完全相同。

### 7.4 零个或多个：`*`

```python
parser.add_argument("files", nargs="*")
```

它允许没有文件，也允许提供多个。

```bash
python app.py
```

```bash
python app.py a.txt b.txt c.txt
```

没有值时通常得到空列表。

这适合“没有提供输入时也有合理行为”的程序。

如果程序没有文件就无法工作，应该考虑下一种规则。

### 7.5 至少一个：`+`

```python
parser.add_argument("files", nargs="+")
```

它要求至少一个值，得到列表。

```bash
python app.py a.txt b.txt
```

合法。

```bash
python app.py
```

缺少必需输入，报告参数错误。

这里的 `+` 属于 `nargs` 的规则字符串，和文件 `open()` 模式中的加号没有同一套含义。

### 7.6 零个或一个：`?`

位置参数可以这样定义：

```python
parser.add_argument("filename", nargs="?", default="notes.txt")
```

用户提供一个文件名就使用它，没有提供就采用默认文件名。

对于带名字的选项，`?` 还可以结合 `const` 区分三种情况。

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--color", nargs="?", const="auto", default="never")

print(parser.parse_args([]).color)
print(parser.parse_args(["--color"]).color)
print(parser.parse_args(["--color", "always"]).color)
```

输出：

```text
never
auto
always
```

没有选项使用 `default`。

出现选项但没有值使用 `const`。

出现选项且提供值使用实际值。

### 7.7 多个可变长度参数会不会含糊？

会增加理解和解析难度。

假设你定义两个位置参数都接收 `*`，用户给三个文件名，究竟哪些归第一组，哪些归第二组，就很难从命令文本直观看出来。

即使解析器有自己的分配规则，用户也可能猜错。

可以改成明确选项、子命令，或让其中一组固定数量。

命令行设计应该帮助人表达意图，不宜只追求把每一种 `nargs` 都塞进同一个工具。

## 第八章：重复选项、累计列表和详细级别

### 8.1 我想多次写 `--tag`，收集所有标签

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--tag", action="append", default=[])
args = parser.parse_args(["--tag", "python", "--tag", "backend"])
print(args.tag)
```

输出：

```text
['python', 'backend']
```

`append` 表示每出现一次，追加这一次的值。

这与普通存值选项不同：普通选项重复时，通常后面的值覆盖前面的值。

### 8.2 `append` 搭配 `nargs` 会得到什么？

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--point", action="append", nargs=2, type=int)
args = parser.parse_args(["--point", "1", "2", "--point", "3", "4"])
print(args.point)
```

得到：

```text
[[1, 2], [3, 4]]
```

每次两个值先形成一个列表，`append` 再把这次列表追加到外层。

所以它适合收集一组组二维坐标，而不是自动摊平成四个数字。

### 8.3 如果我想摊平收集多个值呢？

Python 3.8 起有 `extend`：

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--tag", action="extend", nargs="+", default=[])
args = parser.parse_args(["--tag", "python", "backend", "--tag", "testing"])
print(args.tag)
```

结果：

```text
['python', 'backend', 'testing']
```

`extend` 把这次可迭代结果中的各元素扩展进去。

通常应配合明确的多值规则。

如果把它用于普通单字符串，可能按字符串字符扩展，得到与期待不同的结果。

### 8.4 默认列表会不会和函数可变默认参数一样？

解析器处理 `append` 的默认值与一个普通函数直接反复修改同一默认列表，不完全是同一机制。

但你仍然应该理解默认列表的语义：如果默认列表已有内容，用户提供的项可能追加在默认内容之后，而不是替换整个列表。

```python
parser.add_argument("--tag", action="append", default=["base"])
```

输入 `--tag python`，通常得到包含 `base` 和 `python` 的列表。

如果你的规则是“用户一旦提供就替换默认集合”，可以先用 `default=None`，解析后再做明确处理。

### 8.5 `-v`、`-vv`、`-vvv` 怎样实现？

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("-v", "--verbose", action="count", default=0)

print(parser.parse_args([]).verbose)
print(parser.parse_args(["-v"]).verbose)
print(parser.parse_args(["-vvv"]).verbose)
```

通常输出：

```text
0
1
3
```

`count` 累计出现次数。

你可以在业务代码里把零映射到 WARNING，一映射到 INFO，两次及以上映射到 DEBUG。

这个映射是程序自己决定的，`argparse` 不会自动配置日志。

### 8.6 固定写入某个值：`store_const`

```python
parser.add_argument("--json", dest="format", action="store_const", const="json")
```

选项出现时，把固定字符串 `json` 放入 `format`。

它适合几个不同开关映射到同一个配置字段，但复杂时也可能让帮助不够清楚。

若用户直接选择格式，`--format {text,json}` 往往更容易理解。

## 第九章：帮助信息，怎样让别人一看就会用？

### 9.1 `description` 和 `epilog`

```python
parser = argparse.ArgumentParser(
    description="统计 UTF-8 文本文件的行数",
    epilog="例子：python count_lines.py notes.txt --skip-empty",
)
```

`description` 解释工具做什么。

`epilog` 可以放调用例子、额外说明或文档位置。

不要在帮助里只写“一个强大的工具”，用户更需要知道它接受什么输入、产生什么结果。

### 9.2 `help` 应该说明值的含义

```python
parser.add_argument(
    "--max-lines",
    type=int,
    default=100,
    help="最多处理的行数；必须大于零",
)
```

比只写“最大行数参数”更具体。

如果有单位，也要写明，比如秒、字节或 MiB。

### 9.3 默认值能自动显示吗？

可以使用：

```python
parser = argparse.ArgumentParser(
    formatter_class=argparse.ArgumentDefaultsHelpFormatter,
)
```

它会为适用参数在帮助中补充默认值信息。

也可以在某个帮助文本里使用：

```python
help="最多处理多少行，默认 %(default)s"
```

这里是 `argparse` 帮助文本的格式占位规则，不是 Python f-string。

### 9.4 帮助文本里真的要写百分号呢？

帮助会进行相应百分号格式处理，字面百分号通常要写成 `%%`。

```python
parser.add_argument("--ratio", type=float, help="阈值百分比，例如 80%%")
```

否则某些百分号片段可能被误认为格式占位，导致生成帮助时出错。

### 9.5 我已经手动换行，为什么帮助又重新折行？

默认帮助格式器会按终端宽度组织文字。

如果确实需要保留多行说明，可以考虑 `RawDescriptionHelpFormatter` 或 `RawTextHelpFormatter`。

```python
parser = argparse.ArgumentParser(
    description="第一行说明\n第二行说明",
    formatter_class=argparse.RawDescriptionHelpFormatter,
)
```

这仍不保证所有终端宽度下都具有完全一样的视觉效果。

### 9.6 `prog` 是运行程序的实际位置吗？

```python
parser = argparse.ArgumentParser(prog="linecount")
```

它主要改变使用说明里显示的程序名。

不会移动文件，也不会创建一个可以在系统任何位置运行的 `linecount` 命令。

真正安装命令入口需要打包配置或其他系统安排。

### 9.7 怎样显示工具版本？

```python
parser.add_argument("--version", action="version", version="linecount 1.0")
```

用户执行 `--version` 时显示版本并退出。

版本字符串需要你提供，不会自动从项目所有文件里猜出来。

## 第十章：命令行有空格、负数和连字符时怎么办？

### 10.1 文件路径包含空格

假设 Windows 文件位于：

```text
D:\My Notes\learning.txt
```

终端调用应把它作为一个参数传入：

```powershell
python count_lines.py "D:\My Notes\learning.txt"
```

引号主要由命令行环境解释。

`argparse` 收到的通常是一个完整路径字符串，不包含用于分组的那对外层引号。

不要在程序内部又给路径加一对真正的双引号字符，否则文件名就变了。

### 10.2 我在 `parse_args([...])` 列表里要不要加外层引号？

```python
args = parser.parse_args(["D:/My Notes/learning.txt"])
```

这已经是一个列表元素，空格不会再把它分开。

不要写成一个内部还包含引号字符的字符串。

shell 字符串解析和 Python 列表传参属于两个层次。

### 10.3 负数会不会被当成选项？

许多常见情况，解析器能识别需要的负数值。

但如果你定义了类似 `-1` 的选项，或者值的形式与另一个选项冲突，就可能产生歧义。

```python
parser.add_argument("--offset", type=int)
```

典型调用：

```bash
python app.py --offset -3
```

普通场景可以解析，但复杂前缀值应当主动测试，不宜假设所有以减号开头的文本都自动识别为值。

### 10.4 如果文件名真的以 `-` 开头呢？

常见做法是使用选项终止标记：

```bash
python count_lines.py -- -notes.txt
```

`--` 告诉解析器：后面的内容按位置数据处理，不继续当普通选项解释。

它不属于文件名本身。

也可以在许多场景写成 `./-notes.txt`，让路径不以选项前缀开头。

### 10.5 长选项可以用等号吗？

常见写法：

```bash
python app.py --encoding=utf-8
```

与对应的分开写法具有相同意图：

```bash
python app.py --encoding utf-8
```

对某些以连字符开头的值，等号形式有助于明确它属于哪个选项。

具体是否接受和如何解释，还要结合参数数量规则和目标 Python 版本。

### 10.6 能不能把整条命令字符串交给 `parse_args()`？

不能这样使用：

```python
# 错误方向：parse_args 需要参数项序列，不是整条 shell 命令。
parser.parse_args("--count 3")
```

字符串本身可以被逐字符迭代，结果不会等同于你预期的两个参数。

正确写：

```python
parser.parse_args(["--count", "3"])
```

如果输入确实是 shell 风格字符串，POSIX 规则下可以了解 `shlex.split()`，但它不是 Windows 所有命令行解析规则的通用替代。

## 第十一章：直接传参数列表，为什么对学习和测试特别有用？

### 11.1 `parse_args()` 不传值时读取当前命令行

```python
args = parser.parse_args()
```

适合正常脚本入口。

如果你想测试五种输入，不必每次真的启动五个终端进程。

可以显式传列表：

```python
args = parser.parse_args(["--count", "3"])
```

它只解析这个列表，不再读取本次进程的其他普通命令行参数。

### 11.2 列表里要不要包含脚本名？

通常不要。

```python
parser.parse_args(["--count", "3"])
```

对应用户参数部分。

如果你把 `app.py` 也放进去，它可能被当作额外位置参数，而不是自动识别为脚本名称。

这与 `sys.argv` 整体列表包含启动标识的习惯需要区分。

### 11.3 空列表与 `None` 不一样

```python
parser.parse_args([])
```

表示明确测试“用户没有提供任何参数”。

```python
parser.parse_args(None)
```

与不提供 `args` 的常规行为一样，使用当前命令行输入。

如果测试中写 `None` 想表示空输入，可能意外读到测试运行器自己的参数。

### 11.4 把解析器创建放进函数

```python
import argparse


def build_parser():
    parser = argparse.ArgumentParser(description="重复输出一句问候")
    parser.add_argument("name")
    parser.add_argument("--count", type=int, default=1)
    return parser


def parse_options(argv=None):
    return build_parser().parse_args(argv)


if __name__ == "__main__":
    options = parse_options()
    for _ in range(options.count):
        print(f"你好，{options.name}！")
```

正常运行时，`parse_options()` 读取终端。

测试时，可以调用：

```python
options = parse_options(["Alice", "--count", "2"])
```

这样解析逻辑可以独立观察，不必真的执行后面的重复输出。

### 11.5 为什么函数默认参数是 `argv=None`？

`None` 用来表达“使用当前命令行”。

调用者显式传空列表，则表达“没有任何用户参数”。

这比把两种情况都塞进一个空字符串更清楚。

也避免把一个可变列表作为需要长期共享的默认输入。

## 第十二章：文件路径参数，怎样搭配 `pathlib`？

### 12.1 `type=Path` 会自动打开文件吗？

不会。

```python
import argparse
from pathlib import Path

parser = argparse.ArgumentParser()
parser.add_argument("input", type=Path)
args = parser.parse_args(["data/notes.txt"])

print(type(args.input).__name__)
print(args.input.name)
```

在 Linux 上类型名称通常是 `PosixPath`，原生 Windows 上通常是 `WindowsPath`。

它只是把文本参数变成路径对象，便于后续使用 `parent`、`suffix`、`open()` 等能力。

文件可以尚不存在，类型转换本身也不验证内容编码。

### 12.2 相对路径是相对于脚本目录吗？

假设脚本在：

```text
D:\tools\count_lines.py
```

终端当前位于：

```text
C:\Users\Alice\Documents
```

这里有一份：

```text
C:\Users\Alice\Documents\notes.txt
```

用户运行：

```powershell
python D:\tools\count_lines.py notes.txt
```

`type=Path` 得到相对路径 `notes.txt`。

后续普通文件读取，仍通常以当前工作目录为基准，寻找 Documents 中的文件。

这往往正符合命令行工具使用者的意图。

`argparse` 不会自动把它改成脚本旁边的 `D:\tools\notes.txt`。

### 12.3 如果想显示本次实际访问位置呢？

```python
print(args.input.absolute())
```

可以帮助排错。

是否使用 `resolve()`，要根据是否希望解析符号链接决定。

仅仅为了显示路径，不必未经说明地改变路径语义。

### 12.4 能不能在 `type` 里检查文件存在？

可以，但要控制职责。

```python
import argparse
from pathlib import Path


def existing_file(text):
    path = Path(text).expanduser()
    if not path.is_file():
        raise argparse.ArgumentTypeError("请输入一个可以确认存在的普通文件路径")
    return path
```

它适合简单的输入约束。

但存在性检查并不能保证稍后打开成功，文件可能变化，权限也可能不同。

因此，业务读取仍需要处理实际 I/O 错误。

### 12.5 输出文件为什么通常不检查“已经存在”？

因为输出往往还没生成。

你真正要检查的是：是否允许覆盖、父目录如何创建，以及失败时怎么处理。

这些规则依赖其他选项和业务阶段，通常放在解析之后更清楚。

例如：

```python
parser.add_argument("--output", type=Path)
parser.add_argument("--overwrite", action="store_true")
```

解析后再决定用 `x` 还是 `w` 打开。

### 12.6 `argparse.FileType` 会做什么？

它会在参数转换阶段尝试打开文件，并返回文件对象。

这看起来很方便，但也意味着解析尚未完全成功，I/O 副作用就可能发生。

特别是以写模式打开输出，后面另一个参数出错时，已有文件可能已经被截断。

Python 3.14 文档已把 `FileType` 标为弃用，建议保留路径，到需要时自己打开。

本篇主要采用 `type=Path`，再在业务范围中使用 `with` 管理文件。[官方说明：FileType](https://docs.python.org/3.14/library/argparse.html#argparse.FileType)

### 12.7 想让 `-` 表示标准输入或标准输出呢？

这是一种常见命令行约定，但需要你的程序实现。

`Path("-")` 本身只表示名叫减号的路径，不会自动变成 `sys.stdin`。

可以先把参数保留成字符串，解析后判断：

```python
if options.input == "-":
    source = sys.stdin
else:
    source = open(options.input, encoding=options.encoding)
```

这里还要处理谁负责关闭流：你自己打开的文件应关闭，借用的标准输入通常不由这个函数关闭。

后面的完整实践会用上下文管理把两种情况接起来。

## 第十三章：参数之间有关系，怎样验证？

### 13.1 单个字段合法，组合可能不合法

```bash
python export.py --start 20 --end 10
```

两个值都能转成整数，但起点大于终点不符合业务需求。

`type=int` 只能处理各自转换，不能自动猜测字段关系。

解析后检查：

```python
options = parser.parse_args()

if options.start > options.end:
    parser.error("--start 不能大于 --end")
```

### 13.2 `parser.error()` 与 `raise ValueError()` 有什么不同？

`parser.error()` 会按解析器的方式打印用法和参数错误，通常以状态二退出。

`ValueError` 是普通 Python 异常，若未处理，通常显示 traceback。

对于用户可修正的参数组合错误，前者往往更友好。

对于程序内部逻辑错误，不应一概包装成用户参数错误来隐藏问题。

### 13.3 如果只有启用某项功能时，另一个参数才必填呢？

例如：启用上传时，必须提供服务器地址。

```python
parser.add_argument("--upload", action="store_true")
parser.add_argument("--server")

options = parser.parse_args()

if options.upload and not options.server:
    parser.error("使用 --upload 时，必须提供 --server")
```

不要直接把 `--server` 设为始终 `required=True`，否则没有启用上传时也会被要求提供。

规则应该对应真正条件。

### 13.4 两个选项不能同时出现呢？

使用互斥组：

```python
import argparse

parser = argparse.ArgumentParser()
group = parser.add_mutually_exclusive_group()
group.add_argument("--quiet", action="store_true")
group.add_argument("--verbose", action="store_true")
```

现在用户同时提供两个，就会被拒绝。

没有提供任何一个，仍然允许。

### 13.5 如果必须二选一呢？

```python
group = parser.add_mutually_exclusive_group(required=True)
group.add_argument("--text", action="store_true")
group.add_argument("--json", action="store_true")
```

这表达“恰好选一项”。

但如果只是选择输出格式，一个 `--format` 加 `choices` 往往更简洁。

互斥组最适合真正不同的输入方式或模式，而不是为了展示 API 使工具难用。

### 13.6 互斥组是任意条件表达式吗？

不是。

它擅长处理选项是否出现这一类互斥关系，不是完整业务规则引擎。

“至少两项”“A 出现时 B 可以出现但 C 不能出现”等复杂条件，通常在解析后用清楚的普通 Python 条件判断。

不要为了把所有验证都塞进解析器配置，写出比业务代码更难读的结构。

## 第十四章：把参数和业务分开，`main(argv=None)` 怎样写？

### 14.1 先看程序结构

```python
import argparse
import sys
from pathlib import Path


def build_parser():
    parser = argparse.ArgumentParser(description="统计一份文本的行数")
    parser.add_argument("input", type=Path)
    parser.add_argument("--encoding", default="utf-8")
    parser.add_argument("--skip-empty", action="store_true")
    return parser


def count_lines(path, encoding, skip_empty):
    count = 0
    with path.open(encoding=encoding) as source:
        for line in source:
            if skip_empty and not line.strip():
                continue
            count += 1
    return count


def main(argv=None):
    parser = build_parser()
    options = parser.parse_args(argv)

    try:
        count = count_lines(options.input, options.encoding, options.skip_empty)
    except (OSError, UnicodeError, LookupError) as error:
        print(f"读取失败：{error}", file=sys.stderr)
        return 1

    print(count)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 14.2 为什么解析函数不直接读文件？

`build_parser()` 只声明接口，便于查看工具接受什么。

`count_lines()` 只负责业务计算，可以被其他 Python 程序直接调用。

`main()` 把参数解析、业务调用、用户输出和退出状态连接起来。

这种分工让你测试业务时不用每次构造命令行，也能在 CLI 中复用同一套逻辑。

### 14.3 为什么捕获 `LookupError`？

用户可能写一个不存在的编码名称。

这与文件内容不能按合法编码解码不同，可能在查找编码器时触发 `LookupError`。

作为用户输入的一部分，这里给出读取失败提示，并返回失败状态。

不要把所有程序异常都放进一个宽泛捕获里，否则真正的代码 bug 也可能伪装成输入文件问题。

### 14.4 `main()` 的返回值就是自动退出码吗？

普通函数返回整数，不会自动让进程使用它作为退出状态。

最后的：

```python
sys.exit(main())
```

把返回值交给解释器退出机制。

零通常表示成功，非零表示失败。

参数解析器自己处理的帮助和参数错误，通常会通过 `SystemExit` 在 `parse_args()` 阶段退出。

### 14.5 怎样在代码里测试它？

```python
status = main(["notes.txt", "--skip-empty"])
```

它使用你给的参数列表，而不是当前进程命令行。

这里仍会真正读取指定文件，并打印结果。

如果只想测试解析结果，可以单独调用 `build_parser().parse_args([...])`。

如果只想测试行数计算，直接调用 `count_lines()`。

你可以选择与问题相匹配的测试层次。

## 第十五章：子命令，为什么很多工具写成 `tool add`、`tool list`？

### 15.1 一个工具有几个不同动作

假设你想做一个文本工具：

```bash
python texttool.py count notes.txt
```

统计行数。

```bash
python texttool.py preview notes.txt --lines 5
```

查看前五行。

`count` 和 `preview` 是两个子命令，各自有自己的参数。

这比一大串 `--count-mode`、`--preview-mode` 和互斥字段更容易理解。

### 15.2 创建子解析器

```python
import argparse
from pathlib import Path

parser = argparse.ArgumentParser(prog="texttool")
subparsers = parser.add_subparsers(dest="command", required=True)

count_parser = subparsers.add_parser("count", help="统计文本行数")
count_parser.add_argument("input", type=Path)
count_parser.add_argument("--skip-empty", action="store_true")

preview_parser = subparsers.add_parser("preview", help="预览文本开头")
preview_parser.add_argument("input", type=Path)
preview_parser.add_argument("--lines", type=int, default=5)
```

`add_subparsers()` 创建子命令集合。

每个 `add_parser()` 再创建相应子命令解析器。

`dest="command"` 保存用户选择的子命令名称。

`required=True` 要求必须选择一个动作。

### 15.3 解析后结果是什么？

```python
print(parser.parse_args(["count", "notes.txt", "--skip-empty"]))
```

结果会包含 `command='count'`、输入路径和跳过空行的布尔值。

不会凭空带上 `preview` 专属的 `lines` 字段。

所以业务代码不要不加判断地访问所有子命令的所有属性。

### 15.4 子命令帮助怎么查看？

总帮助：

```bash
python texttool.py --help
```

子命令帮助：

```bash
python texttool.py preview --help
```

后者重点展示 `preview` 接受的输入和 `--lines`。

参数属于哪个解析器，也会影响它放在子命令前面还是后面。

### 15.5 为什么全局选项写在后面有时不认识？

如果 `--verbose` 只在最外层解析器定义，常见自然位置是：

```bash
python texttool.py --verbose count notes.txt
```

你想支持子命令后同样写法，需要明确设计共享参数或在相应子解析器中定义。

不要认为父解析器登记的所有选项都会无条件在任何层级、任何位置被识别。

### 15.6 用 `set_defaults()` 绑定处理函数

```python
def handle_count(options):
    print("执行统计：", options.input)
    return 0


def handle_preview(options):
    print("执行预览：", options.input, options.lines)
    return 0


count_parser.set_defaults(handler=handle_count)
preview_parser.set_defaults(handler=handle_preview)

options = parser.parse_args(["preview", "notes.txt", "--lines", "3"])
status = options.handler(options)
```

这里把函数对象存进结果属性，没有在配置时就调用函数。

选择哪个子命令，就带上对应处理函数。

主入口不必写很长的 `if/elif`，但工具只有两个动作时，直接判断也完全可以。

### 15.7 `set_defaults()` 是自动声明命令行参数吗？

不是。

它能给结果命名空间放入默认属性，但不会自动让用户可以输入 `--handler`。

`handler` 在这个例子中属于内部调度信息。

如果要把结果输出成 JSON，应主动选择适合公开的字段，不要直接序列化这个函数对象。

## 第十六章：配置文件、环境变量和命令行，谁覆盖谁？

### 16.1 三种来源可能同时存在

一个工具可能有内置默认值：每页二十条。

配置文件写每页五十条。

环境变量又设置为三十条。

本次命令行传入 `--page-size 10`。

最终应该取多少？

`argparse` 不会自动为你的整个应用决定优先级。

你应先制定政策，例如：

```text
内置默认值 < 配置文件 < 环境变量 < 显式命令行
```

右侧覆盖左侧。

也可以采用其他合理政策，但要保持一致，并在帮助中说明。

### 16.2 为什么有时要区分“用户没写”和“用户明确给了默认值”？

假设配置文件中 `debug=True`。

命令行没有写任何调试选项，你希望保留配置。

但解析器若直接给 `debug=False`，你可能不知道这是用户主动关闭，还是解析器自己的默认。

可以先采用三态：

```python
parser.add_argument(
    "--debug",
    action=argparse.BooleanOptionalAction,
    default=None,
)
```

`None` 表示没有通过命令行明确选择。

`True` 表示显式开启。

`False` 表示显式关闭。

再根据 `is not None` 判断是否覆盖配置。

不要用 `if options.debug:` 判断是否覆盖，否则明确关闭的 `False` 会被漏掉。

### 16.3 `argparse.SUPPRESS`：没提供时不建立属性

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--page-size", type=int, default=argparse.SUPPRESS)

print(vars(parser.parse_args([])))
print(vars(parser.parse_args(["--page-size", "10"])))
```

输出：

```text
{}
{'page_size': 10}
```

这适合把“本次明确给出的覆盖项”直接合并到已有配置。

但是，不存在的属性不能直接 `options.page_size` 读取，应该先检查或通过字典处理。

### 16.4 环境变量不是自动整数或布尔值

```python
import os

raw = os.getenv("APP_PAGE_SIZE")
```

取得的通常是字符串或 `None`。

你仍需要转换与验证。

尤其不能用 `bool("false")` 解释关闭标记，这与前面 `type=bool` 的问题相同。

### 16.5 一个清楚的合并方法

```python
import argparse
import os


def parse_page_size(text):
    try:
        value = int(text)
    except ValueError as error:
        raise ValueError("page_size 必须是整数") from error
    if not 1 <= value <= 100:
        raise ValueError("page_size 必须在 1 到 100 之间")
    return value


def build_options(argv=None, file_config=None):
    config = {"page_size": 20, "debug": False}

    if file_config is not None:
        config.update(file_config)

    env_value = os.getenv("APP_PAGE_SIZE")
    if env_value is not None:
        config["page_size"] = parse_page_size(env_value)

    parser = argparse.ArgumentParser()
    parser.add_argument("--page-size", type=int, default=argparse.SUPPRESS)
    parser.add_argument(
        "--debug",
        action=argparse.BooleanOptionalAction,
        default=argparse.SUPPRESS,
    )
    explicit = vars(parser.parse_args(argv))
    config.update(explicit)

    if type(config["page_size"]) is not int or not 1 <= config["page_size"] <= 100:
        parser.error("最终 page_size 必须是 1 到 100 之间的整数")
    if type(config["debug"]) is not bool:
        parser.error("最终 debug 必须是布尔值")

    return config
```

### 16.6 为什么合并后还验证一遍？

命令行转换器只约束经过它的输入。

配置文件、环境变量、内置默认和调用者传入对象可能走不同路径。

合并后的数据才是最终业务使用的配置，应该在这个入口获得统一保证。

示例没有允许任意配置字段，也没有完整实现配置文件加载；真实工具还应检查允许键，防止拼写错误被静默保留。

### 16.7 配置文件路径本身也是参数，怎样先知道它？

这时可以分两阶段。

第一阶段只识别 `--config`。

读取配置后，把结果用于完整解析器的默认值或合并政策。

```python
import argparse

pre_parser = argparse.ArgumentParser(add_help=False, allow_abbrev=False)
pre_parser.add_argument("--config")
early, remaining = pre_parser.parse_known_args(["--config", "settings.toml", "--count", "3"])

print(early.config)
print(remaining)
```

输出：

```text
settings.toml
['--count', '3']
```

这只是演示预解析。

第二阶段应采用明确的完整参数协议，并决定是重新解析原始参数，还是解析剩余参数且补入配置字段，不能无意把某些值丢掉。

## 第十七章：`parse_known_args()`，为什么会剩下一组参数？

### 17.1 普通解析通常拒绝不认识的参数

如果你的工具只接受 `--count`，用户却提供 `--unknown`，`parse_args()` 通常报告错误。

这能发现拼写错误，适合一般 CLI。

但包装工具有时只负责一部分选项，剩余参数要传给另一个程序。

### 17.2 已知与未知分开返回

```python
import argparse

parser = argparse.ArgumentParser(allow_abbrev=False)
parser.add_argument("--count", type=int, default=1)

known, unknown = parser.parse_known_args(["--count", "3", "--worker-mode", "fast"])

print(known)
print(unknown)
```

输出：

```text
Namespace(count=3)
['--worker-mode', 'fast']
```

返回的是二元组，所以通常用两个变量解包。

### 17.3 不认识的参数可以直接忽略吗？

只有在你明确决定这样做时才行。

普通工具若无声忽略，用户把 `--encoding` 拼错，程序可能仍运行并使用默认编码，造成很难理解的结果。

`parse_known_args()` 适合明确转发或分层解析，不宜当成“让任何输入都不报错”的通用开关。

### 17.4 为什么这里关闭缩写？

默认长选项可能接受没有歧义的缩写。

假设你定义 `--count`，输入 `--cou` 可能被解析器当作这个已知选项。

对转发工具，你原本可能想把 `--cou` 留给下游。

使用 `allow_abbrev=False` 可以减少这种意外消费。

### 17.5 长选项缩写为什么可能让脚本以后坏掉？

今天只有 `--output`，用户写 `--out` 能工作。

以后新增 `--outline`，同一个缩写可能变得有歧义。

因此，交互式临时输入可以理解这种便利，正式脚本和文档最好使用完整长选项。

你也可以一开始关闭缩写，获得更明确的接口。

### 17.6 `REMAINDER` 能做什么？

```python
parser.add_argument("command", nargs=argparse.REMAINDER)
```

它用于把剩余参数收集为一组，常见于包装命令。

不过，可选项位置和边界仍需要设计。

如果程序将这些参数传给 `subprocess`，应继续传参数列表，不能再用空格拼成一条 shell 字符串。

解析阶段保留了边界，执行阶段也应该保留边界。

## 第十八章：错误、退出和嵌入式使用

### 18.1 `parse_args()` 为什么会让程序退出？

普通命令行程序遇到参数错误，通常不能继续做业务工作。

解析器会输出错误提示，再通过 `SystemExit` 结束当前入口。

这与调用 `sys.exit()` 的机制有关，不是无条件强制杀进程的系统调用。

### 18.2 测试时怎样观察退出状态？

```python
import argparse
from contextlib import redirect_stderr
from io import StringIO

parser = argparse.ArgumentParser(prog="demo")
parser.add_argument("--count", type=int)

buffer = StringIO()
with redirect_stderr(buffer):
    try:
        parser.parse_args(["--count", "many"])
    except SystemExit as error:
        status = error.code

print(status)
print("--count" in buffer.getvalue())
```

通常输出：

```text
2
True
```

这里只是把预期退出捕获下来做观察。

正常 CLI 入口通常让解析器按约定退出即可。

### 18.3 `except Exception` 为什么没有捕获它？

`SystemExit` 直接属于 `BaseException` 分支，不是普通 `Exception` 子类。

这样一般业务异常捕获不容易无意阻止用户请求退出。

需要专门测试退出时，明确捕获 `SystemExit`。

不要为了省事把所有代码都包进 `except BaseException`，否则中断等控制信号也可能被吞掉。

### 18.4 `exit_on_error=False` 是万能“不退出”开关吗？

Python 3.9 起可以设置：

```python
parser = argparse.ArgumentParser(exit_on_error=False)
```

它让部分解析错误以 `ArgumentError` 形式交给调用者处理。

但帮助动作、显式 `parser.error()`，以及不同 Python 版本里某些错误路径，仍可能触发退出。

所以，如果你要把解析器嵌入长期运行程序，不应仅设置这个参数就假设所有路径都不会退出。

应对目标版本和需要支持的错误类型做实际测试。

### 18.5 `parser.exit()`、`print_help()` 和 `format_help()`

`parser.exit(status, message)` 用于输出可选信息并退出。

`parser.print_help()` 把帮助打印到相应流，通常不主动像帮助 action 一样完成整个退出流程。

`parser.format_help()` 返回帮助字符串，适合测试或放到其他展示位置。

```python
help_text = parser.format_help()
```

不要把返回字符串与直接打印的接口混在一起。

### 18.6 GUI 或 Notebook 里为什么解析到奇怪参数？

因为不传列表时，解析器读取当前进程命令行。

Notebook 内核、测试运行器或宿主应用可能有自己的启动参数。

如果你只是演示这个工具的输入，明确传 `[]` 或测试参数列表。

这与库有没有安装正确无关，是输入来源不同。

## 第十九章：参数分组与共享定义

### 19.1 选项多了，帮助怎样分类？

```python
import argparse

parser = argparse.ArgumentParser()
input_group = parser.add_argument_group("输入选项")
input_group.add_argument("--encoding", default="utf-8")
input_group.add_argument("--skip-empty", action="store_true")

output_group = parser.add_argument_group("输出选项")
output_group.add_argument("--output")
output_group.add_argument("--format", choices=["text", "json"], default="text")
```

分组主要帮助阅读帮助信息。

它不会自动建立独立的嵌套结果对象，也不会让同组参数自动具有业务依赖。

结果仍然通常放在同一个 `Namespace` 中。

### 19.2 共享一组子命令选项

可以使用父解析器：

```python
common = argparse.ArgumentParser(add_help=False)
common.add_argument("--encoding", default="utf-8")

parser = argparse.ArgumentParser()
commands = parser.add_subparsers(dest="command", required=True)
count = commands.add_parser("count", parents=[common])
preview = commands.add_parser("preview", parents=[common])
```

`add_help=False` 避免父子帮助选项重复冲突。

通常应先完整配置父解析器，再创建使用它的子解析器。

后续修改父解析器，不应被当成对子解析器的自动实时广播。

### 19.3 共享选项的默认值可能互相影响预期

如果同一个 `dest` 在不同解析层重复定义，默认值与显式值的覆盖顺序可能让结果意外。

比如父层已经解析出 `verbose=True`，子层同名选项默认 `False` 又可能覆盖它。

这类接口应写具体调用测试，而不是只看帮助里能显示就认为所有顺序都正确。

有时把全局选项只放在一个明确位置，会比双向兼容简单很多。

### 19.4 `conflict_handler="resolve"` 适合随便开启吗？

它能在某些定义冲突时替换旧选项对应关系。

但如果你只是无意重复定义同一个参数，自动消解可能掩盖程序错误。

普通工具保留默认冲突报错，往往更容易及早发现问题。

只有在你确实设计了可覆盖的参数组合时，才研究相应替换规则。

## 第二十章：参数文件与更复杂的解析方式

### 20.1 参数很多，能不能写进一个文件？

可以配置 `fromfile_prefix_chars`：

```python
parser = argparse.ArgumentParser(fromfile_prefix_chars="@")
```

调用时使用类似：

```bash
python app.py @options.txt
```

解析器把对应文件内容作为参数来源之一。

默认处理通常是一行一个参数项，不等于把整个文件当成 shell 脚本执行。

### 20.2 文件里的空格会自动按 shell 分词吗？

不要这样假设。

例如你想表达 `--name` 和 `Alice Smith` 两项，默认按行规则需要分别放成合适参数项，而不是照搬一行终端命令。

可以通过重写 `convert_arg_line_to_args()` 定义自己的规则，但要清楚说明注释、引号和空白怎样处理。

### 20.3 什么时候配置文件更合适？

如果内容具有嵌套结构、长期保存、类型丰富和版本迁移需求，JSON 或 TOML 通常比参数文件更适合。

参数文件主要复用一组调用参数，不是所有配置需求的统一格式。

不要因为解析器支持 `@file`，就把复杂应用配置都塞进一长串命令行选项。

### 20.4 选项和多值位置参数交错，为什么有时解析不了？

普通 `parse_args()` 对可变长度位置参数与选项交错存在规则限制。

Python 3.7 起的 `parse_intermixed_args()` 支持部分更灵活的交错方式。

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--label")
parser.add_argument("numbers", nargs="*", type=int)

options = parser.parse_intermixed_args(["1", "--label", "demo", "2", "3"])
print(options.numbers)
```

得到：

```text
[1, 2, 3]
```

### 20.5 为什么不全部换成这个方法？

它不支持所有 `argparse` 功能，例如子解析器，以及混合位置项和选项的某些互斥组。

因此，这是针对特定调用风格的接口，不是普通解析器的无条件升级版。[官方限制](https://docs.python.org/3.14/library/argparse.html#intermixed-parsing)

### 20.6 Python 3.14 的提示和颜色

Python 3.14 增加 `suggest_on_error` 与 `color` 等选项。

`suggest_on_error` 可以对字符串 choices 和子命令名的某些拼写错误给出建议，并不是替你修正所有参数错误。

`color` 控制相应彩色输出，终端和环境也会影响表现。

如果要兼容旧版，可以显式按 `sys.version_info` 决定是否传这些新关键字，不要把旧版的未知参数错误当成安装故障。

### 20.7 自定义 action 值得初学时就写吗？

多数常见需求，已有的 `store_true`、`append`、`count`、类型转换和解析后验证已经够用。

自定义 `Action` 适合真正需要改变解析动作的场景，但会引入更多接口约定。

先把普通 Python 业务规则写清楚，再决定是否值得放到解析器内部，通常更容易维护。

## 第二十一章：完整实践一——把文本统计写成真正能用的 CLI

### 21.1 先把工具行为说清楚

我们希望支持：

```bash
python textstats.py notes.txt
```

统计一份文件。

```bash
python textstats.py notes.txt --skip-empty --format json
```

跳过全空白行，输出 JSON。

```bash
python textstats.py notes.txt --output reports/stats.json --format json
```

保存到新文件，默认不覆盖已有结果。

还希望支持 `-` 表示标准输入，方便终端管道使用。

### 21.2 输入和统计口径

行数按文件迭代得到的行计算，最后一行即使没有换行，也是一行。

`--skip-empty` 开启时，全为空白的行不计入统计。

字符数统计保留行内容中的码点数量，包括文本流读取后的换行；它不是原始文件字节数。

`--max-lines` 限制最多读取的物理行数量，空白行也消耗这个上限。

这些规则提前写清楚，用户才能理解不同选项为什么产生不同结果。

### 21.3 完整代码

保存为 `textstats.py`：

```python
import argparse
import json
import sys
from contextlib import nullcontext
from itertools import islice
from pathlib import Path


def positive_int(text):
    try:
        value = int(text)
    except ValueError as error:
        raise argparse.ArgumentTypeError("必须是整数") from error
    if value <= 0:
        raise argparse.ArgumentTypeError("必须大于零")
    return value


def build_parser():
    parser = argparse.ArgumentParser(
        description="统计文本行数和字符数；输入 - 时读取标准输入",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        allow_abbrev=False,
    )
    parser.add_argument("input", help="输入路径，或 -")
    parser.add_argument("--encoding", default="utf-8", help="输入文件编码")
    parser.add_argument("--skip-empty", action="store_true", help="不统计全空白行")
    parser.add_argument("--max-lines", type=positive_int, help="最多读取多少物理行")
    parser.add_argument("--format", choices=["text", "json"], default="text")
    parser.add_argument("-o", "--output", type=Path, help="输出路径，省略时写标准输出")
    parser.add_argument("--overwrite", action="store_true", help="允许覆盖已有输出")
    return parser


def inspect_stream(source, skip_empty=False, max_lines=None):
    read_lines = 0
    counted_lines = 0
    characters = 0
    lines = source if max_lines is None else islice(source, max_lines)

    for line in lines:
        read_lines += 1
        if skip_empty and not line.strip():
            continue
        counted_lines += 1
        characters += len(line)

    return {
        "read_lines": read_lines,
        "counted_lines": counted_lines,
        "characters": characters,
    }


def open_input(name, encoding):
    if name == "-":
        return nullcontext(sys.stdin)
    return Path(name).open("r", encoding=encoding)


def format_result(result, output_format):
    if output_format == "json":
        return json.dumps(result, ensure_ascii=False, indent=2, allow_nan=False) + "\n"
    return (
        f"读取行数：{result['read_lines']}\n"
        f"计入行数：{result['counted_lines']}\n"
        f"字符数量：{result['characters']}\n"
    )


def check_output(parser, options):
    if options.output is None or options.input == "-":
        return

    source = Path(options.input)
    target = options.output

    try:
        same_path = source.resolve() == target.resolve()
        same_file = target.exists() and source.samefile(target)
    except OSError as error:
        parser.error(f"无法检查输入输出位置：{error}")

    if same_path or same_file:
        parser.error("输入输出不能指向同一个文件")


def main(argv=None):
    parser = build_parser()
    options = parser.parse_args(argv)
    check_output(parser, options)

    try:
        with open_input(options.input, options.encoding) as source:
            result = inspect_stream(
                source,
                skip_empty=options.skip_empty,
                max_lines=options.max_lines,
            )

        text = format_result(result, options.format)

        if options.output is None:
            sys.stdout.write(text)
        else:
            options.output.parent.mkdir(parents=True, exist_ok=True)
            mode = "w" if options.overwrite else "x"
            with options.output.open(mode, encoding="utf-8", newline="\n") as target:
                target.write(text)
    except (OSError, UnicodeError, LookupError) as error:
        print(f"处理失败：{error}", file=sys.stderr)
        return 1

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 21.4 先准备一份完全确定的输入

```python
from pathlib import Path

Path("notes.txt").write_text("apple\n\nbanana", encoding="utf-8", newline="\n")
```

这里有三行：第一行 apple，第二行空白，第三行 banana。

最后一行没有换行。

运行：

```bash
python textstats.py notes.txt
```

输出：

```text
读取行数：3
计入行数：3
字符数量：13
```

`apple\n` 六个码点，空白行一个换行，`banana` 六个码点，加起来十三。

### 21.5 开启跳过空白后呢？

```bash
python textstats.py notes.txt --skip-empty
```

输出：

```text
读取行数：3
计入行数：2
字符数量：12
```

读取了三行，但只有两行计入结果。

这也是为什么报告分别保存 `read_lines` 和 `counted_lines`，避免“行数”这个词在不同选项下含糊。

### 21.6 如果最多读取前两行呢？

```bash
python textstats.py notes.txt --max-lines 2 --skip-empty
```

输出：

```text
读取行数：2
计入行数：1
字符数量：6
```

上限限制的是从输入取得的物理行数量。

它不会为了凑够两条非空行自动继续读到第三行。

如果你需要另一种规则，应明确设计“最多处理多少条有效记录”，不要悄悄改变现有选项含义。

### 21.7 为什么标准输入使用 `nullcontext`？

从文件名打开的流归我们管理，退出 `with` 时应关闭。

`sys.stdin` 是借用的标准流，通常不应该由这个小函数顺手关闭。

`nullcontext(sys.stdin)` 在进入时交出这个流，退出时不替你关闭它。

这样外层仍然可以写统一的 `with open_input(...) as source`，同时保留正确所有权。

### 21.8 输出 JSON 时，提示信息放哪里？

正常标准输出只写结果。

错误诊断写 `sys.stderr`。

于是：

```bash
python textstats.py notes.txt --format json > result.json
```

在支持相应重定向的终端里，结果文件不会混入“开始处理”“程序运行成功”等调试文字。

终端重定向的字节编码还可能受 shell 和 Python 标准输出配置影响；程序主动写文件时则明确使用 UTF-8。

### 21.9 为什么输入处理结束后才打开输出？

如果输入编码错误，或者读取失败，应该先报告，避免过早创建甚至截断结果文件。

这里先取得统计结果并格式化，再进入输出阶段。

输出阶段自身仍可能失败，已有输出若允许覆盖，也可能留下部分内容。

重要报告需要完整发布时，可以继续使用临时文件替换方案。

### 21.10 为什么还检查输入输出相同？

如果用户把 `--output` 指回输入文件，即使程序先完成读取，也会把原始输入改成统计报告。

这未必符合用户意图。

因此，本例明确禁止输入输出指向同一文件，包括某些不同名字的硬链接情况。

检查仍然不是并发锁，不能把它推广成不可信共享目录的完整安全保证。

## 第二十二章：完整实践二——两个子命令共享业务基础

### 22.1 期望的调用

```bash
python texttool.py count notes.txt --skip-empty
```

```bash
python texttool.py preview notes.txt --lines 2
```

我们让每个子命令只接收自己需要的选项。

预览的行数不应强迫统计命令也提供。

### 22.2 完整代码

```python
import argparse
import sys
from itertools import islice
from pathlib import Path


def positive_int(text):
    try:
        value = int(text)
    except ValueError as error:
        raise argparse.ArgumentTypeError("必须是整数") from error
    if value <= 0:
        raise argparse.ArgumentTypeError("必须大于零")
    return value


def handle_count(options):
    count = 0
    with options.input.open(encoding=options.encoding) as source:
        for line in source:
            if options.skip_empty and not line.strip():
                continue
            count += 1
    print(count)
    return 0


def handle_preview(options):
    with options.input.open(encoding=options.encoding) as source:
        for line in islice(source, options.lines):
            print(line, end="")
    return 0


def build_parser():
    parser = argparse.ArgumentParser(description="一个小型文本工具", allow_abbrev=False)
    commands = parser.add_subparsers(dest="command", required=True)

    count = commands.add_parser("count", help="统计行数", allow_abbrev=False)
    count.add_argument("input", type=Path)
    count.add_argument("--encoding", default="utf-8")
    count.add_argument("--skip-empty", action="store_true")
    count.set_defaults(handler=handle_count)

    preview = commands.add_parser("preview", help="查看开头几行", allow_abbrev=False)
    preview.add_argument("input", type=Path)
    preview.add_argument("--encoding", default="utf-8")
    preview.add_argument("--lines", type=positive_int, default=5)
    preview.set_defaults(handler=handle_preview)

    return parser


def main(argv=None):
    options = build_parser().parse_args(argv)
    try:
        return options.handler(options)
    except (OSError, UnicodeError, LookupError) as error:
        print(f"文本处理失败：{error}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

### 22.3 为什么两个子命令都定义 `--encoding`？

为了让完整例子每个子解析器的规则都看得见。

只有少量共享项时，适度重复配置比过早抽象更容易阅读。

共享项增加后，可以使用父解析器或一个添加公共选项的小函数。

但应先确认两个子命令的编码选项真的具有同样含义，而不是只因为名字相同就共享。

### 22.4 解析完成后，怎么知道调用谁？

每个子解析器通过 `set_defaults(handler=...)` 放入相应函数。

所以：

```python
options.handler(options)
```

调用的是用户选择动作对应的函数。

这里 `options.handler` 取得函数对象，后面的括号才真正调用。

如果写成配置时 `handler=handle_count()`，函数会提前执行，而且缺少参数，这不是我们想要的时机。

### 22.5 预览最后一行没有换行，会怎样？

`print(line, end="")` 尽量保持读到的行表现，不额外补换行。

如果输入最后一行没有换行，终端提示符可能紧接着显示。

你也可以决定在工具输出末尾补一个换行，但那属于展示政策，应明确说明，不要把它当成读取失败。

## 第二十三章：完整实践三——多文件校验报告，正常结果与错误分开

### 23.1 任务

接收多个 JSON 文件，逐份检查是否能解析。

支持 `--format text` 或 `--format json`。

有任意一份失败，最终退出状态为一；参数本身不合法则由解析器按参数错误处理。

这体现三个不同结果：调用方式错误、业务检查失败、全部成功。

### 23.2 完整代码

```python
import argparse
import json
import sys
from pathlib import Path


def inspect_file(path):
    try:
        with path.open(encoding="utf-8") as source:
            value = json.load(source)
    except (OSError, UnicodeError, ValueError) as error:
        return {"path": str(path), "ok": False, "error": str(error)}
    else:
        return {"path": str(path), "ok": True, "top_level_type": type(value).__name__}


def build_parser():
    parser = argparse.ArgumentParser(
        description="逐份检查 JSON 文件语法，不执行业务模式校验",
        allow_abbrev=False,
    )
    parser.add_argument("files", nargs="+", type=Path)
    parser.add_argument("--format", choices=["text", "json"], default="text")
    return parser


def main(argv=None):
    options = build_parser().parse_args(argv)
    results = [inspect_file(path) for path in options.files]
    success = all(item["ok"] for item in results)

    if options.format == "json":
        print(json.dumps({"all_valid": success, "results": results}, ensure_ascii=False, indent=2))
    else:
        for item in results:
            if item["ok"]:
                print(f"通过：{item['path']}，顶层为 {item['top_level_type']}")
            else:
                print(f"失败：{item['path']}：{item['error']}", file=sys.stderr)

    return 0 if success else 1


if __name__ == "__main__":
    sys.exit(main())
```

### 23.3 为什么文件列表使用 `nargs="+"`？

没有输入文件时，这个工具没有默认扫描目标。

我们主动要求至少一份，避免空集合的 `all()` 为真，让程序在什么都没检查时看起来“全部通过”。

这里参数数量规则帮助表达了业务入口要求。

### 23.4 为什么 JSON 模式把错误也写进正常结果？

因为这次输出本来就是一份“校验报告”，失败记录属于报告数据。

它与程序自身突然崩溃的诊断不同。

文本模式方便人看，错误记录写 stderr；JSON 模式为了形成一个完整机器可读报告，把每条检查结果统一写进结构中。

无论采用哪种政策，都应让调用者知道，不能一边输出半份 JSON，一边把错误文本混入同一流。

### 23.5 语法通过是不是代表数据能直接用于业务？

不是。

工具描述已经写明只检查 JSON 语法。

顶层是数组还是对象、字段是否存在、数值范围是否正确，需要另加模式校验。

把工具能力准确写在帮助里，也是命令行接口设计的一部分。

## 第二十四章：几个经常在真正使用时才冒出来的问题

### 24.1 参数名字里有中文，能不能解析？

Python 字符串可以包含中文，解析器也可以处理相应名称或值。

但公开工具的选项通常使用稳定、易输入的 ASCII 名称，帮助说明再使用中文。

这有利于跨终端、脚本和团队环境使用。

参数值中的中文则很常见，例如文件名、搜索词和描述内容。

真正到达程序前的编码与终端处理，还受操作系统和启动环境影响，不能仅靠 `argparse` 保证任意外部字节都正确。

### 24.2 我传入的是数字列表，能不能直接用 `[1,2,3]`？

如果定义：

```python
parser.add_argument("numbers", nargs="+", type=int)
```

常见调用是：

```bash
python sum.py 1 2 3
```

不是自动解释 Python 列表表达式。

`[1,2,3]` 通常只是一个字符串参数，不能直接被 `int()` 转换。

如果你确实想支持逗号列表，可以写受控转换器：

```python
import argparse


def comma_ints(text):
    parts = text.split(",")
    if not parts or any(part.strip() == "" for part in parts):
        raise argparse.ArgumentTypeError("请输入逗号分隔的整数，例如 1,2,3")
    try:
        return [int(part) for part in parts]
    except ValueError as error:
        raise argparse.ArgumentTypeError("列表中每一项都必须是整数") from error
```

然后定义 `--numbers` 使用这个转换器。

不要用 `eval()` 执行用户输入来获得列表，数据解析不需要任意 Python 代码执行能力。

### 24.3 输入同一个普通选项两次，为什么只剩最后一个？

普通 `store` 动作就是把值存到同一属性，后面的赋值覆盖前面的赋值。

```bash
python app.py --encoding gbk --encoding utf-8
```

通常最后使用 UTF-8。

如果你要拒绝重复选项，需要额外设计；如果你要累计，使用 `append` 等合适动作。

不要把普通选项的覆盖行为误当成解析器随机丢数据。

### 24.4 `vars(args)` 是复制结果吗？

`vars(args)` 通常返回对象的属性字典，而不是递归深拷贝。

```python
import argparse

args = argparse.Namespace(tags=["python"])
values = vars(args)
values["tags"].append("backend")
print(args.tags)
```

可以看到原对象里的列表也变了。

如果只想生成独立输出模型，最好主动选择需要的字段和结构，不要把它当成任意对象安全复制工具。

### 24.5 `Namespace` 可以自己创建吗？

可以：

```python
import argparse

options = argparse.Namespace(encoding="utf-8", skip_empty=False)
```

它只是一个便于存属性的对象，不自动证明数据经过了解析器验证。

业务函数如果对输入有重要约束，应在合适边界验证，或采用更清楚的内部数据模型。

### 24.6 我想只检查参数，不运行任务，可以吗？

可以把解析器构造和业务调用分开。

测试时调用 `build_parser().parse_args([...])`，只观察结果。

正式运行时再由 `main()` 调业务函数。

如果任务包含文件删除、网络提交或其他副作用，这种分层尤其有用：你能先验证调用方式，而不会因为导入模块就执行真实任务。

### 24.7 为什么导入模块时不应自动 `parse_args()`？

假设测试程序导入 `textstats` 想使用 `inspect_stream()`。

模块顶层如果立即解析当前命令行，就可能把测试运行器参数当成自己的输入，并提前退出。

把入口放在：

```python
if __name__ == "__main__":
    sys.exit(main())
```

能让“作为脚本运行”和“作为模块导入使用”具有清楚区别。

### 24.8 `--help` 为什么没有跑我后面的打印？

帮助 action 通常显示帮助后直接成功退出。

这是正常行为，用户请求的是了解用法，并没有要求同时执行任务。

如果你需要取得帮助字符串用于界面展示，使用 `format_help()`，不要通过真的解析 `--help` 再想办法阻止它退出。

## 第二十五章：怎样给命令行工具安排合适的测试？

### 25.1 先测参数规则

```python
import argparse


def make_parser():
    parser = argparse.ArgumentParser(prog="demo", allow_abbrev=False)
    parser.add_argument("input")
    parser.add_argument("--count", type=int, default=1)
    parser.add_argument("--verbose", action="store_true")
    return parser


parser = make_parser()
options = parser.parse_args(["notes.txt", "--count", "3", "--verbose"])

assert options.input == "notes.txt"
assert options.count == 3
assert options.verbose is True
```

这里检查的是参数转换结果，不需要真实文件。

测试文件内容时，再另外准备临时文件。

### 25.2 再测预期错误

```python
import argparse
from contextlib import redirect_stderr
from io import StringIO

parser = argparse.ArgumentParser(prog="demo")
parser.add_argument("--count", type=int)

with redirect_stderr(StringIO()) as errors:
    try:
        parser.parse_args(["--count", "many"])
    except SystemExit as error:
        assert error.code == 2
    else:
        raise AssertionError("非法整数没有被拒绝")

assert "--count" in errors.getvalue()
```

不要仅检查“抛了任何异常”，那可能是程序写错导致的 `NameError`。

预期参数错误应验证对应退出行为，并检查有助于用户定位的提示。

### 25.3 为什么不精确比较整段帮助文本？

不同 Python 版本可能调整标题、措辞、换行、颜色和程序名生成。

如果你的需求只是“帮助里要出现输出选项和用途”，可以检查关键片段。

只有产品明确要求固定整段帮助输出时，才需要维护更严格的快照测试，并接受版本变化带来的更新成本。

### 25.4 用 `StringIO` 测业务函数

```python
from io import StringIO

source = StringIO("apple\n\nbanana")
result = inspect_stream(source, skip_empty=True)

assert result == {
    "read_lines": 3,
    "counted_lines": 2,
    "characters": 12,
}
```

这里使用前面 `textstats.py` 中的 `inspect_stream()`。

输入是内存文本流，不必每次创建磁盘文件。

这能直接验证统计口径，避免把解析器、文件系统和算法错误混到同一个测试里。

### 25.5 哪些地方值得做真实文件测试？

路径基准、输出不覆盖、输入输出同一文件检查、父目录创建和文件编码，都与真实文件系统有关。

这些能力需要临时目录中的实际文件验证，单靠假对象不能证明它们按期望工作。

测试结束后删除临时目录，避免影响用户真实资料。

### 25.6 最后测进程退出状态

当工具需要被 shell、CI 或其他程序调用时，退出码也是接口的一部分。

可以用 `subprocess.run()` 启动脚本，检查 `returncode`、`stdout` 和 `stderr`。

这个层次不必把所有单字段组合都重新测试一遍，重点覆盖正常成功、参数错误、输入失败和帮助等入口行为。

### 25.7 不要让测试自己依赖偶然环境

传入明确参数列表，使用临时目录，控制输入编码，并避免依赖当前真实家目录是否恰好有 `notes.txt`。

如果测试必须改变工作目录，应恢复原状态，避免影响后续测试。

真实命令调用的引号规则需要在目标 shell 中验证，Python 列表参数测试不会自动覆盖所有 shell 分词差异。

## 第二十六章：练习一——从两个位置参数开始

### 26.1 题目

写 `sum_two.py`，接受两个整数，输出它们的和。

要求缺少输入、输入不是整数时，给出解析器正常错误提示。

### 26.2 先自己预测调用

```bash
python sum_two.py 10 20
```

应输出三十。

```bash
python sum_two.py 10
```

应报告缺少第二个输入。

```bash
python sum_two.py ten 20
```

应报告整数转换错误。

### 26.3 参考答案

```python
import argparse

parser = argparse.ArgumentParser(description="计算两个整数之和")
parser.add_argument("first", type=int)
parser.add_argument("second", type=int)
options = parser.parse_args()

print(options.first + options.second)
```

这里不需要自己索引 `sys.argv`，也不需要再为这两个转换各写一份相同错误格式。

但如果业务要求整数在特定范围内，仍要增加范围规则。

## 第二十七章：练习二——布尔开关和默认值

### 27.1 题目

写一个程序接受 `--name`，默认使用 `World`。

如果提供 `--upper`，把问候文本转换为大写。

要求 `--upper` 不再需要跟随 true 或 false。

### 27.2 参考答案

```python
import argparse

parser = argparse.ArgumentParser(description="生成问候文本")
parser.add_argument("--name", default="World")
parser.add_argument("--upper", action="store_true")
options = parser.parse_args()

message = f"Hello, {options.name}!"
if options.upper:
    message = message.upper()

print(message)
```

### 27.3 应该验证哪些调用？

无参数时输出 `Hello, World!`。

只提供 `--name Alice` 时改变姓名。

提供 `--name Alice --upper` 时输出大写问候。

额外写 `--upper false` 通常应被视为多余输入，帮助你确认这个选项确实是无值开关。

## 第二十八章：练习三——多个数字与操作选择

### 28.1 题目

接受一个或多个浮点数，通过 `--operation` 选择 `sum` 或 `max`，默认求和。

输入为空时拒绝，不允许没有元素时再调用 `max()`。

### 28.2 参考答案

```python
import argparse
import math


def finite_float(text):
    try:
        value = float(text)
    except ValueError as error:
        raise argparse.ArgumentTypeError("必须是数字") from error
    if not math.isfinite(value):
        raise argparse.ArgumentTypeError("必须是有限数字")
    return value


parser = argparse.ArgumentParser(description="计算一组有限数字")
parser.add_argument("numbers", nargs="+", type=finite_float)
parser.add_argument("--operation", choices=["sum", "max"], default="sum")
options = parser.parse_args()

if options.operation == "sum":
    result = sum(options.numbers)
else:
    result = max(options.numbers)

print(result)
```

### 28.3 为什么没有只写 `type=float`？

因为 `float()` 还能接受 NaN、Infinity 等非有限值。

如果题目约定只处理有限数字，就需要额外检查。

这再次说明：通用类型转换和业务合法性是两层要求。

极大数值累计仍可能溢出为无穷，实际计算结果也可以按需求继续检查；输入逐个有限并不自动保证任意后续计算有限。

## 第二十九章：练习四——输入方式二选一

### 29.1 题目

用户可以直接提供 `--text`，也可以提供 `--file`，必须选择一种且不能同时出现。

读取文件时使用 UTF-8，输出文本长度。

### 29.2 参考答案

```python
import argparse
import sys
from pathlib import Path

parser = argparse.ArgumentParser(description="统计一段文本的长度")
group = parser.add_mutually_exclusive_group(required=True)
group.add_argument("--text")
group.add_argument("--file", type=Path)
options = parser.parse_args()

try:
    if options.text is not None:
        text = options.text
    else:
        text = options.file.read_text(encoding="utf-8")
except (OSError, UnicodeError) as error:
    print(f"读取失败：{error}", file=sys.stderr)
    sys.exit(1)

print(len(text))
```

### 29.3 为什么检查 `is not None`？

用户可能明确传一个空字符串作为文本。

如果你用 `if options.text:`，空字符串为假，就会误走文件分支。

参数有没有提供，与内容在布尔判断中是真是假，需要分别判断。

## 第三十章：练习五——带预览模式的输出生成

### 30.1 题目

接收输入路径和输出路径。

默认只显示准备执行的转换计划；只有明确传 `--apply` 才写结果。

不得把预览解释为已经完成写入。

### 30.2 参考答案

```python
import argparse
import sys
from pathlib import Path

parser = argparse.ArgumentParser(description="把小型 UTF-8 文本转换为大写")
parser.add_argument("input", type=Path)
parser.add_argument("output", type=Path)
parser.add_argument("--apply", action="store_true", help="实际写入新结果")
options = parser.parse_args()

if options.input.resolve() == options.output.resolve():
    parser.error("输入输出不能使用同一路径")

if not options.apply:
    print(f"计划：读取 {options.input}，生成大写文本到 {options.output}")
    sys.exit(0)

try:
    text = options.input.read_text(encoding="utf-8")
    converted = text.upper()
    options.output.parent.mkdir(parents=True, exist_ok=True)
    with options.output.open("x", encoding="utf-8", newline="\n") as target:
        target.write(converted)
except (OSError, UnicodeError) as error:
    print(f"转换失败：{error}", file=sys.stderr)
    sys.exit(1)

print("已生成：", options.output)
```

### 30.3 这个练习还有哪些边界？

它整体读取小文本，不适合任意巨型文件。

预览模式只展示计划，不证明输入一定能读取或目标一定能创建。

实际输出使用 `x`，已有名字会拒绝；中途写入失败仍可能留下部分目标文件。

这几条应随工具说明一起交代，不能仅凭出现 `--apply` 就把它称为完整可靠的文件事务系统。

## 第三十一章：练习六——给已有程序增加可测试入口

### 31.1 题目

把直接在模块顶层解析并执行的脚本，改成 `build_parser()`、业务函数和 `main(argv=None)`。

要求导入模块时不解析测试运行器的参数，不执行真实任务。

### 31.2 参考结构

```python
import argparse
import sys


def build_parser():
    parser = argparse.ArgumentParser(description="重复一段文字")
    parser.add_argument("text")
    parser.add_argument("--times", type=int, default=1)
    return parser


def repeat_text(text, times):
    if times < 0:
        raise ValueError("times 不能为负数")
    return [text for _ in range(times)]


def main(argv=None):
    parser = build_parser()
    options = parser.parse_args(argv)
    if options.times < 0:
        parser.error("--times 不能为负数")

    for line in repeat_text(options.text, options.times):
        print(line)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 31.3 为什么业务函数里也检查范围？

因为它可能被其他 Python 代码直接调用，绕过 CLI。

CLI 的检查提供友好的参数错误；业务入口的检查保护函数自身约定。

简单练习可以共享一个验证函数，避免规则分叉，但不要以为所有调用者都会经过解析器。

## 第三十二章：回到自己的脚本，怎样逐步加入参数？

先找出当前写死在代码中的可变输入，例如文件路径、输出目录、数量上限和编码。

再问：哪些是每次都必须明确给出的核心输入，哪些有合理默认值？

核心输入适合考虑位置参数，调节行为的项适合考虑选项。

接着写出你希望用户执行的三五条实际命令。

如果你自己看这几条命令都很难判断含义，先调整接口，再编写解析器配置。

然后逐项登记规则，运行正常输入、缺失输入、错误类型和帮助命令。

最后把解析结果交给业务函数，约定正常结果、诊断信息和退出状态。

不要一开始就把一个二十项配置表全部变成开关。

一个只需要文件名和编码的小工具，可以先做好这两项，再随着真实需求增加功能。

## 第三十三章：常用接口速查

| 需求 | 写法 | 结果或注意事项 |
| --- | --- | --- |
| 创建解析器 | `argparse.ArgumentParser(...)` | 保存程序级规则 |
| 位置参数 | `add_argument("input")` | 按位置识别 |
| 长短选项 | `add_argument("-o", "--output")` | 同一项的两个入口 |
| 类型转换 | `type=int`、`type=Path` | 传转换器对象 |
| 限定值 | `choices=[...]` | 在转换后检查输入选择 |
| 默认值 | `default=...` | 缺少输入时使用 |
| 无值布尔开关 | `action="store_true"` | 出现即为真 |
| 正反布尔开关 | `BooleanOptionalAction` | 支持开启和关闭形式 |
| 至少一个值 | `nargs="+"` | 通常返回列表 |
| 零个或多个 | `nargs="*"` | 允许空列表 |
| 零个或一个 | `nargs="?"` | 可结合 default 与 const |
| 固定数量 | `nargs=2` | 对应长度列表 |
| 重复收集 | `action="append"` | 每次增加一项 |
| 多值摊平 | `action="extend"` | 注意字符串迭代行为 |
| 次数累加 | `action="count"` | 常用于 `-vv` |
| 显示占位名 | `metavar="PATH"` | 改帮助，不改属性 |
| 指定结果属性 | `dest="output_path"` | 改内部取值名 |
| 解析当前命令行 | `parse_args()` | 通常读取 argv 的用户部分 |
| 解析测试输入 | `parse_args([...])` | 列表里不包含脚本名 |
| 已知与剩余 | `parse_known_args()` | 不能默默吞掉未知输入 |
| 参数组合错误 | `parser.error(message)` | 通常提示并以状态二退出 |
| 帮助字符串 | `format_help()` | 返回字符串 |
| 互斥选择 | `add_mutually_exclusive_group()` | 可要求必须选一项 |
| 子命令 | `add_subparsers()` | 各命令独立规则 |
| 内部处理函数 | `set_defaults(handler=...)` | 传函数对象，不提前调用 |
| 无值不建属性 | `default=SUPPRESS` | 便于明确覆盖配置 |

你真正需要熟练的，是把一条终端命令逐项对应到结果对象，并知道每个字段的类型、默认来源和验证阶段。

当工具出错时，可以按顺序检查：shell 有没有正确分成参数项，解析器有没有登记对应规则，类型转换有没有完成，业务组合是否有效，最后才是文件或网络操作是否成功。

配套阅读：[Python-sys库.md](Python-sys库.md)、[Python-pathlib库.md](Python-pathlib库.md)、[Python-open文件处理.md](Python-open文件处理.md)。

官方参考：[argparse 接口](https://docs.python.org/3.14/library/argparse.html)、[argparse 入门教程](https://docs.python.org/3.14/howto/argparse.html)。本篇实际例子采用独立教学数据，帮助文本的完整排版与颜色应以目标 Python 和终端环境为准。
