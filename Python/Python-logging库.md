# Python `logging` 库超详细教程：从一条运行记录，到多模块、文件轮转与可验证的日志流程

## 零基础先导：为什么 `print()` 不能解决所有排错？

如果只有一个五行脚本，`print("做到这里")` 很好用。但程序一旦定时运行、处理几百个文件、部署在服务器上，你可能要回答：**什么时间、哪个模块、哪一级别、发生了什么？**

`logging` 是 Python 标准库，负责**记录程序运行事件**；它不是“屏幕打印的复杂版本”，而是可按等级筛选、按不同方式输出、保留上下文的记录系统。

```python
import logging

logging.basicConfig(level=logging.INFO)       # 最简单的根记录器配置
logging.debug("进入循环")                      # 默认 INFO 阈值下不会显示
logging.info("开始处理文件：%s", "a.txt")      # 正常运行中的重要信息
logging.warning("文件缺少可选字段")            # 值得留意但未必失败
```

- `logging.basicConfig(...)` 配置**基本输出行为**，在已有日志处理器的环境里重复调用可能不会按你预想覆盖配置。
- `INFO` 是记录等级的阈值，低于它的 `DEBUG` 通常被过滤；更高级别一般包含 `WARNING`、`ERROR`、`CRITICAL`。
- `"文件：%s", "a.txt"` 是延迟格式化风格，避免不必要的消息格式化开销；这不是 SQL 占位符。
- 记录消息和抛异常不同：`logging.error(...)` 不会自动终止程序，`raise` 才会抛出异常。

后文的 Logger/Handler/Formatter 可以类比为：**谁记录（Logger）→ 送到哪里（Handler）→ 写成什么样（Formatter）**。先熟练使用一个记录器，再进入复杂的多模块配置；切勿只为了演示就在库模块导入时配置全局日志。

---

你的脚本开始时只有十几行。

你用 `print("开始")`、`print("读取成功")` 看看程序运行到哪里，很方便。

后来脚本需要批量处理几百个文件。

你想知道是哪一个文件失败，失败前程序走到哪一步，程序到底处理了多少项。

你又希望终端只显示重要消息，文件里留下更详细的信息。

再后来，代码拆成多个模块，每个模块都需要记录情况，但你不希望每个文件都各自打开一份日志文件。

这些需求就是 `logging` 要解决的问题。

这篇从最小示例开始，逐步解释记录器、级别、处理器、格式器、传播、过滤器与配置。

每一个名字都放回实际流程里理解，而不是先背一张类名表。

主要代码按 Python 3.11 及以上使用习惯展开。

Python 3.12 的过滤器返回新记录等变化会在相关地方注明。

命令示例中的 `python` 表示你当前选用的解释器。Windows 或已激活虚拟环境中通常可以这样调用；如果 WSL 中只有 `python3`，就使用 `python3 logging_demo.py`。先确认环境，再运行，不必为了教程里的命令名称另装一套 Python。

同一小节的短示例可以沿用本节对象；完整实践包含所需导入与输入。

练习修改的是临时目录或你自己建立的练习目录，先不要把实验代码直接接到重要程序的运行日志上。

## 第一章：为什么有了 `print`，还需要 `logging`？

### 1.1 `print` 解决的是把内容写出来

```python
print("开始读取")
print("读取完成")
```

这两行能告诉你程序运行到了哪里。

但随着程序变大，你会继续问：

这条消息是调试信息，还是严重错误？

是哪一个模块发出的？

发生在什么时间？

终端要不要显示？

要不要同时写入文件？

想关闭所有调试消息，是否要修改几十处 `print`？

这些问题不适合靠每一处输出都单独实现一遍。

### 1.2 日志记录的是事件，不只是文字

一条日志可以包含消息文本，也可以包含级别、记录器名称、时间、文件名、行号、异常信息与业务字段。

`logging` 把这些内容组织成一条记录，再交给配置决定如何输出。

可以先按这条路线理解：

```text
业务代码说：发生了一件事
    ↓
记录器决定是否接收这个级别
    ↓
形成一条 LogRecord
    ↓
处理器决定输出到哪里、接收哪些级别
    ↓
格式器决定显示成什么样子
```

后面还会加入传播与过滤规则，但先理解这条主线就够了。

### 1.3 `logging` 是标准库吗？

是。

正常 Python 安装已经包含它：

```python
import logging
```

不需要为了使用标准库再执行 `pip install logging`。

如果项目文件也叫 `logging.py`，反而可能遮挡标准库导入。

所以练习文件可以叫 `logging_demo.py`，不要直接叫 `logging.py`。

### 1.4 最小可运行例子

把下面内容保存到一个新脚本，单独运行：

```python
import logging

logging.basicConfig(level=logging.INFO)

logging.debug("准备打印很详细的变量")
logging.info("开始处理文件")
logging.warning("有一项配置未填写，使用默认值")
logging.error("某个文件处理失败")
```

在没有其他日志配置影响的新进程中，通常显示：

```text
INFO:root:开始处理文件
WARNING:root:有一项配置未填写，使用默认值
ERROR:root:某个文件处理失败
```

调试信息没有显示，因为入口阈值设为 INFO。

这不是 `debug()` 没有被调用，而是这类消息在当前策略下被过滤掉。

### 1.5 为什么消息里有一个 `root`？

直接调用 `logging.info(...)`，使用的是根记录器。

root 可以理解为整个记录器层次的根。

小脚本直接使用它很方便。

代码拆成模块后，更常见的是每个模块拿一个有名字的记录器，再由程序入口统一安排输出。

两种写法使用同一套系统，并不是两个互不相干的日志库。

## 第二章：级别——不是颜色标签，而是消息的重要程度

### 2.1 五个最常见级别

从较详细到较严重，日常主要使用：

```text
DEBUG       10
INFO        20
WARNING     30
ERROR       40
CRITICAL    50
```

数字用于阈值比较。

默认常见配置下，低于阈值的消息不会继续输出。

设置 INFO，一般允许 INFO、WARNING、ERROR、CRITICAL，而不允许 DEBUG。

设置 ERROR，不代表只看 ERROR，也会允许更高的 CRITICAL。

### 2.2 具体什么时候用哪个？

你正在批量导入学习记录。

打印每一行解析出的细节，适合 DEBUG。

打印“开始导入”“本次成功导入一百条”，适合 INFO。

某行有可恢复问题，程序按约定跳过，可能适合 WARNING。

一个任务确实没完成，适合 ERROR。

程序的重要功能已经无法继续，例如关键基础资源不可用，才可能考虑 CRITICAL。

级别表达事件影响，不是取决于消息长度。

不要因为想让一行文字显眼，就把普通进度全部写成 ERROR。

### 2.3 发出 `error` 会让程序退出吗？

不会自动退出。

```python
import logging

logging.basicConfig(level=logging.INFO)
logging.error("这里只是记录错误")
print("程序仍然可以继续执行")
```

记录错误与抛异常、返回失败状态、结束程序，是不同动作。

你仍然需要根据业务决定控制流。

如果必须终止，应在相应位置抛异常或返回退出码，而不是希望日志级别替你完成。

### 2.4 `warn` 和 `warning` 选哪个？

使用 `warning()`。

旧的 `warn()` 名称已被弃用，不适合作为新笔记里的常规写法。

```python
logging.warning("没有找到可选配置")
```

### 2.5 `NOTSET` 是不是最低输出级别？

它的数值是零，但放在记录器上时要结合继承理解。

非根记录器级别是 NOTSET，通常表示向上寻找有效级别，而不是无条件记录一切。

根记录器有自己的级别，默认通常为 WARNING。

处理器上的 NOTSET 则通常表示处理器不再额外按正数阈值排除记录。

相同名称放在不同对象上，含义需要放回处理流程看。

后面讲层级时会用完整例子验证。

## 第三章：`basicConfig`——它配置谁，为什么第二次似乎没作用？

### 3.1 最常见的起步配置

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(levelname)s | %(name)s | %(message)s",
)

logging.info("配置完成")
```

输出：

```text
INFO | root | 配置完成
```

`level` 设置根记录器级别。

`format` 决定处理器使用的默认显示格式。

没有指定文件时，通常建立面向标准错误流的处理器。

### 3.2 默认写到标准错误流是什么意思？

终端里标准输出和标准错误经常显示在同一个窗口，所以肉眼不一定看得出区别。

但命令行重定向时，它们是不同通道。

你的工具可以把正式数据写到 stdout，把运行诊断写到 stderr。

例如脚本输出 JSON 结果时，日志不要随意混进同一份标准输出数据中。

`logging.StreamHandler()` 默认使用 `sys.stderr`，需要标准输出时可以明确指定。

这个默认值在 [官方 StreamHandler 说明](https://docs.python.org/3.12/library/logging.handlers.html#logging.StreamHandler) 中有明确规定。

### 3.3 为什么第一次调用 `warning` 也可能影响后续配置？

模块级的 `logging.warning()` 等便捷函数，在根记录器还没有处理器时，可能自动调用基本配置。

于是你先写：

```python
import logging

logging.warning("先出现一条消息")
logging.basicConfig(level=logging.DEBUG)
logging.debug("为什么没显示？")
```

第二次 `basicConfig` 可能没有重新配置，因为根记录器已经有处理器。

这不是级别比较失灵，而是配置并没有按你以为的方式替换。

### 3.4 如何明确重新配置？

现代 Python 支持 `force=True`：

```python
logging.basicConfig(
    level=logging.DEBUG,
    format="%(levelname)s:%(message)s",
    force=True,
)
```

它会移除并关闭根记录器已有的处理器，再按新参数配置。

这个动作只应由有权管理整个应用输出的入口执行。

你写一个供别人导入的小工具模块，就不应该在导入时擅自强制重置宿主程序的根日志配置。

在交互环境和笔记本里，已有环境也可能提前配置日志，因此独立脚本的结果通常更容易预测。

### 3.5 `basicConfig` 能不能同时指定 `filename` 和 `stream`？

这些目标配置不能任意混用。

`filename` 与 `stream` 是相互排斥的常见选项。

传入已有 `handlers` 时，也不能再随意同时传 `filename` 或 `stream`。

如果需要终端和文件双目标，最清楚的方式是明确建立两个处理器。

后面会完整演示。

### 3.6 配置代码放哪里比较好？

通常放在应用入口，且尽量在启动其他需要记录日志的工作之前完成。

```python
import logging


def main():
    logging.basicConfig(level=logging.INFO)
    logging.info("开始执行")


if __name__ == "__main__":
    main()
```

业务模块只负责取记录器和发事件。

这样，别人复用你的函数时，可以决定输出策略，而不会被模块导入副作用抢先控制。

## 第四章：`getLogger`——给不同模块一个稳定名字

### 4.1 为什么不一直使用根记录器？

如果所有消息名字都叫 root，你很难区分来自读取模块、解析模块还是输出模块。

有名字以后，可以只提高某个模块的详细程度，也能在输出里看见来源。

```python
import logging

logger = logging.getLogger("study.importer")
logger.info("准备导入学习记录")
```

这里还需要程序整体配置允许 INFO，并安排处理器，才能正常看见输出。

取记录器本身不会自动为每个名字创建一个文件。

### 4.2 为什么经常写 `getLogger(__name__)`？

`__name__` 通常对应当前模块的导入名称。

在包里运行时，可能是：

```text
study.reader
study.parser
study.report
```

这些名字自然形成层级。

```python
logger = logging.getLogger(__name__)
```

这是模块内常见的稳定写法。

如果脚本直接运行，它的名称可能是 `__main__`。

所以输出名字取决于模块如何加载，不一定等于磁盘文件的绝对路径。

### 4.3 多次取相同名字，会得到不同对象吗？

同一解释器进程内，相同名字通常返回同一个记录器对象。

```python
import logging

first = logging.getLogger("study.reader")
second = logging.getLogger("study.reader")
print(first is second)
```

输出：

```text
True
```

因此，某个地方修改这个记录器的级别，会影响通过同一名字取得的对象。

它不是每次创建一个互不关联的临时记录器。

### 4.4 能不能给每个用户创建一个记录器？

一般不建议把用户编号、请求编号这类无限增长数据塞进记录器名称。

记录器会在日志管理结构中被保存，数量不断增长可能不合适。

名称更适合表达稳定模块或组件。

用户编号、任务编号应作为每条记录的字段传递。

后面会讲 `extra` 和 `LoggerAdapter`。

## 第五章：处理器与格式器——一个决定去哪，一个决定长什么样

### 5.1 先建立一套不依赖根配置的练习

我们创建一个指定名字的记录器。

为了让输出稳定，先检查这个练习记录器有没有旧处理器，再使用一个内存文本流。

这样不需要写磁盘，就能看到完整结果。

```python
import io
import logging

buffer = io.StringIO()
logger = logging.getLogger("lesson.basic")
logger.setLevel(logging.DEBUG)
logger.propagate = False

for old_handler in list(logger.handlers):
    logger.removeHandler(old_handler)
    old_handler.close()

handler = logging.StreamHandler(buffer)
handler.setLevel(logging.INFO)
handler.setFormatter(logging.Formatter("%(levelname)s|%(name)s|%(message)s"))
logger.addHandler(handler)

logger.debug("调试细节")
logger.info("开始学习")
logger.error("演示错误")

print(buffer.getvalue(), end="")
logger.removeHandler(handler)
handler.close()
```

输出：

```text
INFO|lesson.basic|开始学习
ERROR|lesson.basic|演示错误
```

### 5.2 为什么记录器允许 DEBUG，处理器仍然不显示？

因为两个对象都有门槛。

记录器先决定这条记录是否进入流程。

处理器再决定是否把已经进入流程的记录输出到自己的目标。

记录器 DEBUG、处理器 INFO，DEBUG 记录可以形成，但不通过这个处理器。

记录器 INFO、处理器 DEBUG，则 DEBUG 在入口已经被拒绝，后面再宽松也拿不到它。

可以把它理解成先过入口，再过具体出口。

### 5.3 `addHandler` 是复制处理器吗？

它把处理器关联到记录器，不是为你复制一份全新对象。

多个位置使用同一个处理器对象时，会共享它的目标、格式和过滤器等状态。

同一个处理器在同一个记录器上重复添加，库会避免简单重复关联；但你每次初始化都新建一个处理器，再添加进去，就会累积多个不同对象。

这是重复日志的常见来源。

### 5.4 为什么示例最后移除并关闭？

处理器可能持有打开的文件或其他资源。

练习完以后明确移除，避免这个名字的记录器在同一进程后续练习中继续输出。

关闭与移除不是同一动作。

只从列表移除，不等于一定释放了文件句柄。

只关闭，不等于你已经清楚地解除所有关联。

在真实应用里，长期处理器通常由入口创建，并在应用生命周期结束时关闭。

## 第六章：格式字符串——每个百分号后面的名称从哪里来？

### 6.1 最常见的显示字段

```python
formatter = logging.Formatter(
    "%(asctime)s | %(levelname)s | %(name)s | %(message)s"
)
```

`asctime` 是格式化时间。

`levelname` 是级别名称。

`name` 是记录器名称。

`message` 是把消息模板与参数结合后得到的文本。

这些名称来自日志记录及格式过程，不是 Python 自动读取你程序里所有同名变量。

### 6.2 为什么 `message` 不等于传入的模板？

```python
logger.info("成功处理 %s 个文件", 12)
```

模板是 `成功处理 %s 个文件`。

参数是十二。

最终消息才是 `成功处理 12 个文件`。

一条记录可以先保存模板与参数，等真正格式化时再形成文本。

这也是日志调用常用 `%s` 参数写法的原因。

### 6.3 `Formatter(style="{")` 会改变日志调用语法吗？

不会自动改变消息参数合并方式。

```python
formatter = logging.Formatter(
    "{levelname}|{name}|{message}",
    style="{",
)
```

这里改变的是外层记录格式。

日常记录消息仍然可以写：

```python
logger.info("处理文件：%s", "scores.csv")
```

不要因此误写：

```python
# 反面示例：默认日志消息合并不是 str.format。
logger.info("处理文件：{}", "scores.csv")
```

格式器的 `style` 与消息模板的参数约定，是两个层次。

### 6.4 为什么不直接使用 f-string？

f-string 可以使用，但它在调用前就会构造文本。

```python
logger.debug(f"总数：{count}")
```

如果只是一个整数，差别通常很小。

如果字符串构造包含昂贵表达式，即使 DEBUG 被关闭，这个表达式仍可能先执行。

常见参数写法能延迟对象到文本的格式化：

```python
logger.debug("总数：%s", count)
```

但也要注意：函数参数表达式本身仍然先求值。

```python
logger.debug("报告：%s", build_expensive_report())
```

即便消息被拒绝，`build_expensive_report()` 也已经调用了。

需要避免这项工作，可以先判断：

```python
if logger.isEnabledFor(logging.DEBUG):
    logger.debug("报告：%s", build_expensive_report())
```

这里的判断回答入口是否启用，不会保证每个处理器最终一定输出。

### 6.5 怎么显示文件名、行号和函数名？

```python
formatter = logging.Formatter(
    "%(filename)s:%(lineno)d %(funcName)s | %(message)s"
)
```

这对定位代码很有帮助。

不过，如果日志通过公共辅助函数转发，默认看到的可能是辅助函数的位置。

这时要考虑 `stacklevel`，后面会用小例子解释。

### 6.6 时间格式怎么控制？

```python
formatter = logging.Formatter(
    "%(asctime)s | %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
```

`datefmt` 控制时间文本格式。

默认转换通常使用本地时间。

如果你明确需要 UTC，可以为格式器安排对应转换函数，例如 `time.gmtime`，再把时区含义写进输出。

不要把没有时区标记的本地时间文本直接宣称是 UTC。

日志时间适合描述事件发生时刻；测量耗时通常应使用适合经过时间的单调时钟，而不是拿墙上时间相减就假设不会跳变。

## 第七章：传播——为什么一条消息打印了两遍？

### 7.1 先看名字形成的关系

```text
root
└── study
    └── study.reader
```

这个层级由记录器名称中的点表达，不要求磁盘上真的存在同名文件夹。

`study.reader` 发出记录后，可以让记录继续到父级记录器的处理器。

这就是传播。

### 7.2 一条消息出现两次的典型情形

子记录器自己装了一个终端处理器。

父记录器也装了一个终端处理器。

子记录器保持 `propagate=True`。

于是同一条记录先由子级处理器输出一次，再由父级处理器输出一次。

```python
import io
import logging

output = io.StringIO()
parent = logging.getLogger("lesson.propagation")
child = logging.getLogger("lesson.propagation.reader")

for item in (parent, child):
    for old in list(item.handlers):
        item.removeHandler(old)
        old.close()
    item.setLevel(logging.DEBUG)

parent.propagate = False
child.propagate = True
first = logging.StreamHandler(output)
second = logging.StreamHandler(output)
first.setFormatter(logging.Formatter("父处理器:%(message)s"))
second.setFormatter(logging.Formatter("子处理器:%(message)s"))
parent.addHandler(first)
child.addHandler(second)

child.info("读取完成")
print(output.getvalue(), end="")

for owner, handler in ((parent, first), (child, second)):
    owner.removeHandler(handler)
    handler.close()
```

输出：

```text
子处理器:读取完成
父处理器:读取完成
```

不是业务调用了两次，而是同一记录有两个输出路径。

### 7.3 怎么选择正确处理方式？

如果希望父级统一管理输出，通常不要再给每个子记录器重复挂同样目标的处理器。

如果某个子组件明确自行管理输出，可以把它设为 `propagate=False`，阻止继续向上。

不能一看到重复就把所有模块传播全部关闭，否则某些没有自己处理器的模块可能完全没输出。

先画出实际关联，再决定在哪里保留出口。

### 7.4 父记录器的级别会再检查一次吗？

这是最容易误解的细节之一。

记录从子级通过入口检查后，传播时会交给祖先的处理器。

祖先记录器自身的级别与记录器过滤器，不会按照“重新调用父级日志方法”的方式再次应用。

处理器级别仍然会检查。

因此，如果子记录器明确设成 DEBUG，根记录器是 ERROR，而根处理器允许 DEBUG，子级 DEBUG 仍可能输出。

不要把根记录器级别想成整个树上所有已生成记录都必须通过的最后总闸。

### 7.5 用一个最小实验验证

```python
import io
import logging

buffer = io.StringIO()
parent = logging.getLogger("lesson.levels")
child = logging.getLogger("lesson.levels.child")
parent.setLevel(logging.ERROR)
parent.propagate = False
child.setLevel(logging.DEBUG)
child.propagate = True

handler = logging.StreamHandler(buffer)
handler.setLevel(logging.DEBUG)
parent.addHandler(handler)
try:
    child.debug("子记录器明确允许的调试事件")
    print(buffer.getvalue(), end="")
finally:
    parent.removeHandler(handler)
    handler.close()
```

在这两个练习名字没有额外旧处理器干扰时，仍然输出这条 DEBUG 消息。

这项行为在 [官方传播规则说明](https://docs.python.org/3.12/library/logging.html#logging.Logger.propagate) 中明确给出。

理解它以后，你就知道“关闭所有 DEBUG”往往需要把处理器阈值也安排好。

### 7.6 子记录器 NOTSET，父级 ERROR，又会怎样？

这时子记录器没有自己明确的正数级别，会向上寻找有效级别。

它可能继承父级 ERROR，于是在生成 DEBUG 记录之前就被拒绝。

和上一节不同的地方，是子级入口使用的有效级别改变了。

可以用下面方法观察：

```python
child.setLevel(logging.NOTSET)
print(child.getEffectiveLevel())
```

在本节父级配置下得到四十。

所以要区分“入口级别向上继承”与“已创建记录向上传播”两个过程。

## 第八章：写文件——日志文件放在哪里，什么时候打开？

### 8.1 最简单的文件配置

在你自己的练习目录运行：

```python
import logging

logging.basicConfig(
    filename="app.log",
    level=logging.INFO,
    encoding="utf-8",
    format="%(levelname)s|%(message)s",
)
logging.info("程序启动")
```

相对路径 `app.log` 通常根据当前工作目录解释。

它不会因为这行代码写在 `main.py` 中，就自动保存到 `main.py` 所在目录。

### 8.2 如果想把日志放在脚本旁边呢？

假设脚本位于：

```text
D:\project\main.py
```

你当前终端位于：

```text
C:\Users\Alice
```

运行：

```bash
python D:\project\main.py
```

如果没有其他代码修改工作目录，`filename="app.log"` 通常会对应当前工作目录下的文件。

如果确实要放在脚本旁边，应明确计算：

```python
from pathlib import Path

base_directory = Path(__file__).resolve().parent
log_path = base_directory / "logs" / "app.log"
log_path.parent.mkdir(parents=True, exist_ok=True)
```

然后把 `log_path` 传给文件处理器或基本配置。

交互环境可能没有 `__file__`，这时要明确提供基准目录，而不是假装所有运行方式都与脚本相同。

### 8.3 `FileHandler` 会帮我创建父目录吗？

一般不会替你递归创建日志目录。

```python
from pathlib import Path
import logging

path = Path("practice_logs") / "app.log"
path.parent.mkdir(parents=True, exist_ok=True)
handler = logging.FileHandler(path, encoding="utf-8")
```

创建目录和打开日志文件，是两个动作。

目录权限不足、目标是目录、磁盘不可写，都可能让配置或输出失败。

不要把这些错误都误认为格式字符串写错。

### 8.4 默认是追加还是覆盖？

`FileHandler` 默认模式是追加 `a`。

同一文件的旧内容通常会保留，新消息接在后面。

使用 `mode="w"` 则涉及重新打开时截断文件，不适合在不了解后果时随意添加。

```python
handler = logging.FileHandler(path, mode="a", encoding="utf-8")
```

脚本每次启动都想得到独立文件，可以考虑文件名包含明确运行编号，而不是靠覆盖清空历史来解决。

### 8.5 `delay=True` 有什么用？

```python
handler = logging.FileHandler(path, encoding="utf-8", delay=True)
```

它把实际打开文件延迟到需要输出时。

如果始终没有符合条件的记录，文件可能不需要真正打开。

但它不会解决目录不存在或权限不足，只是相应问题可能延迟到第一次输出时才出现。

延迟打开与容错是不同概念。

### 8.6 日志显示到文件了，是否表示已经安全落盘？

处理器会执行相应写入与刷新，但不能把普通日志调用视为每条记录都已经经过严格持久化保证。

操作系统还可能有缓存，突然断电与进程正常结束也不同。

如果业务需要事务级审计或必须持久化的关键数据，应使用符合要求的数据存储协议，而不是仅把一行日志当作唯一事实来源。

普通运行日志的价值主要是诊断与观察。

## 第九章：同时输出到终端和文件，而且详细程度不同

### 9.1 实际需求

终端希望看到 INFO 及以上，避免太吵。

文件希望保留 DEBUG 及以上，方便以后查细节。

记录器必须允许最详细的一类进入，然后让两个处理器分别筛选。

如果记录器先设为 INFO，文件处理器设成 DEBUG 也无法找回已经被入口拒绝的消息。

### 9.2 完整配置函数

```python
import logging
import sys
from pathlib import Path


def configure_study_logging(log_path, *, console_stream=None):
    path = Path(log_path)
    path.parent.mkdir(parents=True, exist_ok=True)
    logger = logging.getLogger("study_app")
    logger.setLevel(logging.DEBUG)
    logger.propagate = False

    for old in list(logger.handlers):
        logger.removeHandler(old)
        old.close()

    formatter = logging.Formatter("%(levelname)s|%(name)s|%(message)s")
    console = logging.StreamHandler(sys.stderr if console_stream is None else console_stream)
    console.setLevel(logging.INFO)
    console.setFormatter(formatter)

    file_handler = logging.FileHandler(path, encoding="utf-8")
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)

    logger.addHandler(console)
    logger.addHandler(file_handler)
    return logger


def close_study_logging(logger):
    for handler in list(logger.handlers):
        logger.removeHandler(handler)
        handler.close()
```

这个函数管理的是 `study_app` 这条应用记录器分支。

它不强制清空整个根记录器。

但它会替换自己负责的记录器上原有处理器，所以应该由这个应用的入口调用，而不是任意业务函数调用。

### 9.3 用临时目录验证输出区别

```python
import io
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "run.log"
    console_output = io.StringIO()
    logger = configure_study_logging(path, console_stream=console_output)
    try:
        logger.debug("读取了配置细节")
        logger.info("开始处理")
        logger.warning("跳过一条无效记录")
    finally:
        close_study_logging(logger)

    print("终端内容：")
    print(console_output.getvalue(), end="")
    print("文件内容：")
    print(path.read_text(encoding="utf-8"), end="")
```

主要输出：

```text
终端内容：
INFO|study_app|开始处理
WARNING|study_app|跳过一条无效记录
文件内容：
DEBUG|study_app|读取了配置细节
INFO|study_app|开始处理
WARNING|study_app|跳过一条无效记录
```

### 9.4 为什么把 `console_stream` 做成可传入？

真实运行默认可以写 stderr。

验证时传入 `StringIO`，就能在内存里检查确切内容。

你不需要靠盯着终端猜测消息有没有出现。

这是一种很常见的可验证设计：把外部目标作为参数，核心流程保持不变。

### 9.5 每调用一次业务函数，都重新配置好吗？

通常不好。

配置会建立处理器、打开文件、修改共享状态。

业务函数每调用一次就配置，不仅有开销，也容易出现重复关联和文件生命周期混乱。

先配置一次，再反复发记录，比较符合这套系统的设计。

同一个进程需要重新加载配置时，也应由明确的配置流程处理，而不是每个功能各自抢着重置。

## 第十章：异常信息——`error`、`exception`、`exc_info` 有什么区别？

### 10.1 只有文字，可能还不够

```python
try:
    number = int("abc")
except ValueError as error:
    logger.error("转换失败：%s", error)
```

这会记录你写的文字和异常字符串。

但它不会仅因为参数是异常对象，就自动把完整调用栈一起输出。

如果需要知道哪一行经过哪些调用导致异常，应包含异常信息。

### 10.2 在异常处理中使用 `logger.exception`

```python
try:
    number = int("abc")
except ValueError:
    logger.exception("转换数字失败")
```

它按 ERROR 级别记录，并附带当前异常信息。

主要显示形态是：

```text
ERROR|记录器名称|转换数字失败
Traceback (most recent call last):
  ...
ValueError: invalid literal for int() with base 10: 'abc'
```

行号与文件路径取决于你实际保存和运行的位置，所以这里用省略表示变化部分。

### 10.3 `exc_info=True` 是同一种用途吗？

在已有异常上下文中，可以写：

```python
try:
    number = int("abc")
except ValueError:
    logger.error("转换数字失败", exc_info=True)
```

它允许你在不同级别调用中附带异常信息。

`exception()` 是非常常见的方便入口，但最好在真正处理异常的位置使用。

### 10.4 没有正在处理的异常，也调用 `exception` 呢？

这通常不会得到你想要的业务调用栈。

可能出现 `NoneType: None` 这类无有效异常内容。

如果你想记录当前程序堆栈，而不是刚抛出的异常，相关选项是 `stack_info=True`。

两者的信息来源不同。

### 10.5 `stack_info` 和 `exc_info` 分别回答什么？

`exc_info` 说明异常传播所携带的追踪信息。

`stack_info` 可以描述当前日志调用位置的堆栈。

它们可以同时存在，但不要为了显得详细就对每条 INFO 都打印完整堆栈。

输出体积、阅读成本和定位需求要一起考虑。

### 10.6 记录以后，异常还会自动重新抛出吗？

不会。

```python
try:
    number = int("abc")
except ValueError:
    logger.exception("转换失败")
    raise
```

这里重新抛出是因为你明确写了 `raise`。

如果不写，控制流按你的异常处理逻辑继续。

日志调用不是异常传播协议。

### 10.7 每一层都记录再抛，会不会太多？

可能。

一个异常穿过三层，每层都打印同一份追踪，就可能产生三大段重复信息。

常见做法是在能够提供明确业务上下文、并决定如何处理的边界记录一次。

底层可以补充异常类型与消息，必要时使用异常链，而不是每一层都无条件输出完整栈。

例如底层说明“第七行日期无效”，任务边界说明“本次导入失败”，两者职责不同。

## 第十一章：`stacklevel`——辅助函数转发日志，怎么保留真正调用位置？

### 11.1 一个公共提示函数

```python

def report_progress(logger, message):
    logger.info(message)
```

如果格式里显示 `funcName` 和 `lineno`，常常定位到 `report_progress` 内部。

你真正想看的，也许是哪个业务函数调用了它。

### 11.2 使用 `stacklevel`

```python

def report_progress(logger, message):
    logger.info(message, stacklevel=2)
```

这个参数让日志来源定位时向外多看相应层级。

一层普通辅助函数常用二。

如果还有多层包装，应根据实际调用关系验证，而不是永远填一个大数字。

### 11.3 它会改变异常栈吗？

它主要影响记录里识别调用位置的文件名、行号、函数名等。

不会把异常真实经过的调用栈重写成另一条路径。

也不会让辅助函数在运行时真的消失。

这是记录来源归属的调整，不是程序执行流程的改变。

## 第十二章：业务字段——任务编号应该放进哪一部分？

### 12.1 每条消息都手工拼一遍，很容易不统一

你正在导入一个文件夹中的学习数据。

同一轮运行有一个 `run_id`。

每个文件又有自己的文件名。

如果每次都写：

```python
logger.info("[run-202] 文件 a.csv 读取成功")
```

下一处可能写成完全不同形式。

更好的选择之一，是把这些数据作为记录字段，让格式器统一展示。

### 12.2 使用 `extra`

```python
logger.info(
    "文件读取成功",
    extra={"run_id": "run-202", "source_file": "a.csv"},
)
```

格式器可以使用：

```python
formatter = logging.Formatter(
    "%(levelname)s|%(run_id)s|%(source_file)s|%(message)s"
)
```

最终可能是：

```text
INFO|run-202|a.csv|文件读取成功
```

`extra` 中的键进入记录属性空间，因此要避免与已有记录属性冲突。

例如不要随意使用 `name`、`message`、`levelname` 等保留用途名称。

### 12.3 有一条记录忘记传字段，会怎样？

如果格式器要求 `run_id`，记录却没有这个属性，格式化可能失败。

它不会自动从你某个局部变量里寻找同名数据。

因此，要么确保所有经过这个处理器的记录都有所需字段，要么提供默认值。

Python 3.10 起，`Formatter` 支持 `defaults`：

```python
formatter = logging.Formatter(
    "%(levelname)s|%(run_id)s|%(message)s",
    defaults={"run_id": "-"},
)
```

本文的 3.11 基线可以使用。

这项默认值服务于格式化，不表示原始记录对象已经永久带上同名业务属性。

### 12.4 用内存流看缺省与显式字段

```python
import io
import logging

buffer = io.StringIO()
logger = logging.getLogger("lesson.extra")
logger.setLevel(logging.INFO)
logger.propagate = False
handler = logging.StreamHandler(buffer)
handler.setFormatter(logging.Formatter(
    "%(levelname)s|%(run_id)s|%(message)s",
    defaults={"run_id": "-"},
))
logger.addHandler(handler)
try:
    logger.info("普通事件")
    logger.info("任务事件", extra={"run_id": "run-202"})
    print(buffer.getvalue(), end="")
finally:
    logger.removeHandler(handler)
    handler.close()
```

输出：

```text
INFO|-|普通事件
INFO|run-202|任务事件
```

### 12.5 字段是给机器看的，还是给人看的？

都可以。

文本格式可以把字段拼成容易阅读的一行。

结构化格式可以把字段写成 JSON 对象，便于按任务编号查询。

关键是字段名称与含义稳定。

不要今天把 `count` 当成功数，明天又当总输入数，却不改变事件含义。

日志字段本身也是一个小型数据协议。

## 第十三章：`LoggerAdapter`——同一个任务的固定上下文，能不能少写几遍？

### 13.1 先看重复参数

```python
logger.info("开始", extra={"run_id": "run-202"})
logger.info("正在读取", extra={"run_id": "run-202"})
logger.info("完成", extra={"run_id": "run-202"})
```

如果这些事件都属于同一个任务，可以用一个适配器保存这份固定上下文。

### 13.2 基本写法

```python
import logging

base_logger = logging.getLogger("study.jobs")
job_logger = logging.LoggerAdapter(base_logger, {"run_id": "run-202"})
job_logger.info("开始处理")
```

适配器仍然把工作交给原记录器。

它不是重新创建一条完全独立的层级，也不会自动开新文件。

级别和处理器最终仍由对应记录器系统决定。

### 13.3 每次调用再传 `extra`，会自动与适配器合并吗？

在 Python 3.11、3.12 的默认适配器行为里，不要假设它会按你期待合并。

默认处理会使用适配器自己的上下文，调用时提供的 `extra` 可能被覆盖。

Python 3.13 增加了相关的 `merge_extra` 配置能力；兼容本文基线时，不应直接把新版参数复制过来。[版本依据：LoggerAdapter](https://docs.python.org/3.14/library/logging.html#logging.LoggerAdapter)

需要稳定的自定义合并，可以写范围明确的适配器。

### 13.4 一个明确规定覆盖方向的适配器

```python
import logging


class MergingAdapter(logging.LoggerAdapter):
    def process(self, msg, kwargs):
        supplied = kwargs.get("extra") or {}
        merged = dict(self.extra)
        merged.update(supplied)
        kwargs["extra"] = merged
        return msg, kwargs
```

这里规定调用时的同名字段覆盖固定上下文。

如果你想禁止覆盖 `run_id`，应改成相反顺序，或显式检查冲突。

无论哪种选择，都应该在接口说明里写清楚。

### 13.5 适配器保存的是上下文快照吗？

如果你把可变字典传进去，再在其他地方修改它，后续记录可能看到新内容。

如果希望固定一份简单上下文，可以在创建时传入新字典。

但即使浅复制字典，内部嵌套可变对象也不因此自动独立。

通常日志上下文字段使用简单字符串、整数和布尔值，更容易理解与序列化。

### 13.6 异步任务之间怎样避免互相覆盖上下文？

不要使用一个全局可变字典，让多个并发任务轮流把 `run_id` 改成自己的值。

它们可能在记录时读到别人的编号。

可以每个任务建立独立适配器，或者使用 `contextvars` 搭配过滤器传递当前上下文。

后者涉及上下文传播规则，不应该简单描述成“线程局部变量换个名字”。

这里先使用独立适配器，已经能覆盖许多普通脚本与批处理需求。

## 第十四章：过滤器——不仅可以按级别筛选

### 14.1 想让某个目标只接收一种事件

级别回答严重程度，但你也许只想把统计事件送到一个输出目标。

可以让记录带 `event` 字段，再用过滤器判断。

过滤器收到记录，返回是否允许继续。

```python
import logging


class EventFilter(logging.Filter):
    def __init__(self, accepted_event):
        super().__init__()
        self.accepted_event = accepted_event

    def filter(self, record):
        return getattr(record, "event", None) == self.accepted_event
```

把它加到某个处理器：

```python
handler.addFilter(EventFilter("summary"))
```

这个处理器就只接收 `event="summary"` 的记录。

其他处理器不一定受影响。

### 14.2 挂在记录器上和挂在处理器上有什么区别？

记录器过滤器影响在该记录器入口处理的记录。

处理器过滤器影响经过这个出口的记录。

尤其要记住前面传播规则：祖先记录器过滤器不会自动重新处理所有子记录器传播来的记录。

如果你希望某个文件目标统一筛选本分支所有事件，把过滤器放在这个文件处理器上更符合目标。

### 14.3 `logging.Filter("study.reader")` 是什么？

它可以按记录器名称层级匹配。

例如匹配 `study.reader` 及其相关子级名称。

这不等于任意字符串包含匹配。

`study.reader2` 与 `study.reader.child` 的层级关系不同，不能仅靠前几个字母相同就混为一谈。

如果需要复杂业务规则，可以自己写 `filter()` 或可调用过滤器。

### 14.4 过滤器也能补字段吗？

在 3.11 常见做法中，可以修改记录并返回真。

```python
class DefaultRunFilter(logging.Filter):
    def filter(self, record):
        if not hasattr(record, "run_id"):
            record.run_id = "-"
        return True
```

但同一条记录可能交给多个处理器。

你在其中一个过滤器里修改原记录，其他出口也可能看到修改后的对象。

所以这种操作不是自动隔离的。

### 14.5 Python 3.12 的返回新记录解决什么？

Python 3.12 起，过滤器可以返回一个新的 `LogRecord`，用它替换相应流程中的记录。

放在处理器上的过滤器可以据此为该出口修改内容，而不直接改动其他处理器正在使用的原记录。

这是明确的新版能力，3.11 不能直接把“返回一个对象”理解成同样的替换语义。

[官方过滤器版本说明](https://docs.python.org/3.12/library/logging.html#logging.Filter.filter) 说明了这一变化。

### 14.6 不要把脱敏只理解成替换消息里的一个词

如果日志包含用户输入，信息可能出现在消息、参数、异常文本、额外字段或对象的字符串表示中。

只对最终消息做一次简单替换，并不表示所有输出都已经符合你的字段政策。

日常最有效的做法通常是从源头决定应该记录什么。

例如记录文件数量、状态和任务编号，而不是默认把完整内容与所有凭据都打印出来。

这不是要求每个练习构建复杂过滤系统，而是让日志信息与诊断目标对应。

## 第十五章：日志文件持续变大，怎样轮转？

### 15.1 轮转是什么意思？

程序长期运行，单一文件会不断追加。

轮转是在满足一定条件后，把当前文件切换成历史文件，再继续写新的当前文件。

它解决文件规模与保留份数问题，不等于自动备份整个应用状态。

标准库常用大小轮转和时间轮转。

### 15.2 按大小轮转：`RotatingFileHandler`

```python
import logging
from logging.handlers import RotatingFileHandler

handler = RotatingFileHandler(
    "app.log",
    maxBytes=1_000_000,
    backupCount=3,
    encoding="utf-8",
)
handler.setFormatter(logging.Formatter("%(levelname)s|%(message)s"))
```

当前文件是 `app.log`。

旧文件通常依次使用 `.1`、`.2` 等后缀。

`backupCount=3` 表示保留相应数量的备份文件，不是总共只允许三条日志。

### 15.3 `maxBytes=0` 是只允许零字节吗？

不是。

对这个处理器，零用于表示不按正数大小阈值进行这类轮转。

要实际使用大小轮转，通常需要非零 `maxBytes` 和合适的 `backupCount`。

另外，一条记录可能很大，不能把这个参数当作严格的每个文件绝不超过某个字节数的承诺。

编码、估算与单条输出都会影响实际边界。

### 15.4 可以用特别小的阈值练习吗？

可以。

在临时目录建立处理器，连续写入带序号的固定长度消息，再关闭处理器，检查生成了哪些文件。

不要把生产日志阈值也设成几十字节，只为了看见轮转效果。

练习参数用于观察机制，实际参数要根据输出量、保留时间和磁盘预算选择。

### 15.5 按时间轮转：`TimedRotatingFileHandler`

```python
from logging.handlers import TimedRotatingFileHandler

handler = TimedRotatingFileHandler(
    "app.log",
    when="midnight",
    interval=1,
    backupCount=7,
    encoding="utf-8",
    utc=True,
)
```

这里按 UTC 的相应午夜边界安排轮转，并保留指定数量历史文件。

不要把 `utc=True` 自动等同于格式器也输出 UTC；轮转时间规则与消息显示时间仍是两个配置点。

如果希望输出时间也明确 UTC，需要同时配置格式器转换。

### 15.6 到了时间，就一定立刻生成新文件吗？

时间轮转通常在后续有记录需要输出时检查并执行。

不是内部自动启动一个永远准点醒来的定时线程。

如果夜里没有消息，文件可能在下一次日志发生时才切换。

这解释了为什么文件名时间与第一次实际写入时间可能不完全相同。

### 15.7 Windows 和 Linux 外部轮转处理一样吗？

并不完全相同。

`WatchedFileHandler` 主要面向 Unix/Linux 上外部工具移动或替换日志文件的情形。

它会检查文件变化并安排重新打开。

Windows 的文件锁定与相关文件属性机制不同，不应把这个处理器当作通用跨平台轮转答案。

选择时可以直接对照 [官方处理器文档](https://docs.python.org/3.12/library/logging.handlers.html)，不要只看类名里有 watched 就以为能监控所有变化。

### 15.8 多个进程都用同一个轮转文件安全吗？

标准文件处理器没有提供多个独立进程共同轮转同一个文件的通用协调保证。

两个进程可能同时判断需要轮转、重命名或继续写入。

更清楚的结构是由一个专门的接收端拥有文件处理器，其他进程通过队列或其他通道发送记录。

线程与进程不是同一个并发范围。

同一进程里一个处理器的锁，不能自动成为多个进程共享的文件轮转协议。

## 第十六章：日志记录是不是天然不会影响程序？

### 16.1 输出也需要时间

写终端、写磁盘、格式化大对象，都可能花时间。

在普通同步处理器中，这些工作会发生在发出日志的执行路径上。

所以每处理一条数据就写大量 DEBUG，即使最终能帮助定位，也可能明显改变程序速度。

合理级别、适量字段与必要统计，比无差别打印所有数据更有用。

### 16.2 线程安全覆盖什么？

标准 logging 设计支持普通多线程使用，并为共享状态与处理器 I/O 使用相应锁。

但它不会保证两个不同线程的业务事件严格按你想象的先后输出。

线程调度与实际事件发生存在交错。

也不会让你自定义处理器里的任意外部状态自动安全。

你仍然要按共享状态实际使用方式设计。

### 16.3 `raiseExceptions=False` 是不是吞掉所有问题？

它主要影响 logging 在处理内部记录错误时如何报告，而不是一个覆盖所有异常路径的总开关。

配置参数错误、某些自定义代码错误、业务函数错误，不会因此都神奇消失。

也不能用它证明日志文件已经成功写入。

开发时应能看见配置与格式错误，运行中则应根据应用要求决定如何处理输出失败。

把这个全局值改成假，不等于建立了可靠的日志交付保证。

### 16.4 `logging.disable` 与记录器级别有什么区别？

它提供全局层面的级别禁用阈值。

```python
logging.disable(logging.CRITICAL)
```

会禁用到该级别范围的记录，影响范围比一个组件的 `setLevel` 更大。

恢复常规处理可以使用：

```python
logging.disable(logging.NOTSET)
```

不要在库函数里临时禁用全局日志却忘记恢复，否则其他模块也会受到影响。

如果只是想安静某个噪声组件，通常调整该组件的级别或处理器规则更合适。

## 第十七章：队列日志——把较慢的输出交给另一个线程

### 17.1 为什么要把“生成记录”和“写目标”分开？

你的工作线程正在处理文件。

每次记录都直接写慢速目标，会让它等待输出。

`QueueHandler` 可以把记录交给队列。

`QueueListener` 在另一个线程从队列取记录，再交给真正的文件或终端处理器。

可以先按这条流程理解：

```text
业务线程
    ↓
QueueHandler
    ↓
队列
    ↓
QueueListener 线程
    ↓
实际输出处理器
```

这只是把工作转移，不代表格式化、队列操作与内存占用全部消失。

### 17.2 一个完整的线程内队列示例

```python
import io
import logging
from logging.handlers import QueueHandler, QueueListener
from queue import Queue


def run_queue_demo():
    output = io.StringIO()
    queue = Queue()
    target = logging.StreamHandler(output)
    target.setLevel(logging.INFO)
    target.setFormatter(logging.Formatter("%(levelname)s|%(message)s"))
    listener = QueueListener(queue, target, respect_handler_level=True)

    logger = logging.getLogger("lesson.queue")
    logger.setLevel(logging.DEBUG)
    logger.propagate = False
    forwarding = QueueHandler(queue)
    logger.addHandler(forwarding)
    listener.start()
    try:
        logger.debug("不需要送到最终目标的细节")
        logger.info("已进入队列")
        logger.error("一项任务失败")
    finally:
        logger.removeHandler(forwarding)
        listener.stop()
        forwarding.close()
        target.close()
    print(output.getvalue(), end="")
    return output.getvalue()


if __name__ == "__main__":
    run_queue_demo()
```

正常输出：

```text
INFO|已进入队列
ERROR|一项任务失败
```

### 17.3 为什么要先停止生产，再停止监听器？

监听器需要知道不再有新的业务记录持续加入。

本例先把队列处理器从记录器移除，再请求监听器停止。

已有消息按队列与监听器停止协议处理。

如果业务线程仍然不断写入，停止时就需要更完整地协调所有生产者。

不能在程序最后随手调用 `stop()`，却让其他线程继续向已经没有消费者的队列追加消息。

### 17.4 `respect_handler_level=True` 为什么明确写出来？

它让监听器按目标处理器的级别筛选。

这样目标 INFO 就会拒绝队列中的 DEBUG。

不要默认认为监听器与普通记录器分发流程的每个细节都完全相同。

明确配置能够让例子表达预期，也便于检查。

### 17.5 队列是无限的，就不会丢日志了吗？

没有有限容量限制的队列可以持续占用内存。

如果生产速度长期超过消费速度，积压会不断增长。

使用有限容量队列，又要决定满了怎么办：丢弃、阻塞、降级，还是让调用者知道失败。

标准 `QueueHandler` 默认使用非阻塞入队方式，队列满时会进入相应错误处理路径。

这不是自动保证所有消息永久保存的通道。

### 17.6 队列里的记录保留原样吗？

标准 `QueueHandler.prepare()` 会准备可交给队列的记录，处理消息与异常信息等内容，以便减少不可序列化对象问题。

它不是简单把所有原始参数与异常对象完整原样留给接收端。

如果你希望接收端独立决定特殊异常格式，需要理解并按需定制准备过程。

但定制后也要确认跨进程传输时对象可序列化。

不要既要求保留任意活跃对象，又假设它们可以毫无条件跨进程发送。

### 17.7 多进程也用同样代码吗？

整体思路类似，但队列类型、启动方式、接收端生命周期和进程初始化都需要明确设计。

普通 `queue.Queue` 是线程内队列，不是多个独立进程之间的通用传输工具。

跨进程可以考虑 `multiprocessing.Queue` 与一个明确拥有文件处理器的接收端。

同时要避免把 multiprocessing 自身产生的调试记录又送回同一队列，造成递归或死锁等问题。

这是 [官方 QueueHandler 文档](https://docs.python.org/3.12/library/logging.handlers.html#logging.handlers.QueueHandler) 专门提醒的边界。

本文先把单进程线程队列实践完整做对，再理解它如何扩展，避免用一段未经生命周期设计的大代码宣称解决所有并发日志。

## 第十八章：`dictConfig`——配置变多以后，怎样集中描述？

### 18.1 什么时候需要配置字典？

只有一个终端目标时，普通代码配置已经很清楚。

当你有多个处理器、多个格式、特定组件级别和过滤规则，配置字典可以把这些关系集中表达。

它不是必须从外部文件读取。

你可以先在 Python 里写一个字典，确认理解结构，再决定是否由 JSON 等文件提供。

### 18.2 一个最小可运行配置

```python
import logging
import logging.config


def configure_from_dictionary():
    logging.config.dictConfig({
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "plain": {
                "format": "%(levelname)s|%(name)s|%(message)s",
            },
        },
        "handlers": {
            "console": {
                "class": "logging.StreamHandler",
                "level": "INFO",
                "formatter": "plain",
                "stream": "ext://sys.stderr",
            },
        },
        "loggers": {
            "study.configured": {
                "level": "DEBUG",
                "handlers": ["console"],
                "propagate": False,
            },
        },
    })
    return logging.getLogger("study.configured")


if __name__ == "__main__":
    logger = configure_from_dictionary()
    logger.info("使用配置字典")
```

主要输出：

```text
INFO|study.configured|使用配置字典
```

### 18.3 `version: 1` 是 Python 版本吗？

不是。

它是配置字典方案的版本字段。

不要因为解释器是 Python 3.12，就把它改成十二。

### 18.4 `plain`、`console` 是内置名称吗？

不是固定保留的魔法名字。

它们是你给配置对象取的标识。

处理器中的 `formatter: "plain"`，引用上面定义的格式器。

记录器中的 `handlers: ["console"]`，引用处理器。

所以可以把它理解成一张对象关系说明，而不是一堆彼此无关的参数。

如果引用拼错，配置阶段可能报错，应该先修正配置，不要继续假设输出仍然符合预期。

### 18.5 `ext://sys.stderr` 表示什么？

它让配置系统解析已有外部对象。

这里指向标准错误流。

普通字符串 `"sys.stderr"` 与带 `ext://` 的引用并不完全相同。

配置系统还有可调用对象工厂等能力，因此配置文件应来自你信任的应用配置来源。

不要把任意外部用户上传的字典直接交给能够导入和构造对象的配置系统。

### 18.6 `disable_existing_loggers` 为什么明确写假？

配置系统有禁用已有非根记录器的相关默认行为。

你已经导入第三方库，再加载配置时，可能意外让原先记录器不再发出消息。

明确写假，通常更符合“保留已有库记录器，由统一目标接收”的应用预期。

但这仍然是你的配置选择，不是所有系统都必须使用同一个值。

如果你希望严格只启用少量组件，应明确列出相关设计并验证。

### 18.7 配置字典能在运行时只修改级别吗？

它支持增量配置的有限能力，但不是任意热更新对象图的万能接口。

过滤器、格式器和现有处理器关系，不能简单假设都能按同一规则增量改变。

日常如果只是切换详细程度，直接取得目标记录器或处理器并设置级别，往往更直观。

需要完整重载时，应协调输出资源与工作线程，避免一边写一边随意拆目标。

配置字典的确切结构可以在 [官方 dictConfig 方案说明](https://docs.python.org/3.12/library/logging.config.html#configuration-dictionary-schema) 中对照。

## 第十九章：库作者与脚本作者，应该做的事情不一样

### 19.1 你写的是供自己启动的应用

应用知道日志要写哪里、用户要多详细、程序如何退出。

因此由应用入口统一配置是合理的。

### 19.2 你写的是会被别人导入的库

库不知道宿主是否使用终端、文件、队列或其他目标。

所以通常只做：

```python
import logging

logger = logging.getLogger(__name__)
```

需要时发记录，不在导入时调用 `basicConfig(force=True)`。

否则宿主刚配置好的输出可能被你重置。

### 19.3 `NullHandler` 有什么用？

库可以在自己的顶层记录器加一个空处理器。

```python
import logging

logging.getLogger("study_library").addHandler(logging.NullHandler())
```

它本身不输出。

它帮助库在没有应用配置时保持合适的默认行为，同时不阻止正常向上交给宿主处理器。

不要因此再把传播全部关闭，否则宿主可能接不到库的消息。

### 19.4 没有处理器时，为什么有些警告仍然能看到？

系统有 `lastResort` 兜底处理器，默认会处理相应 WARNING 及以上记录。

再加上模块级便捷函数可能自动触发基本配置，观察结果容易与“没配置就绝对没有任何输出”的直觉不同。

所以排查时要分清：使用的是根便捷函数，还是有名字记录器；有没有已有处理器；有没有兜底路径。

这比看到一条输出就断言一定有人偷偷调用了某个配置函数更准确。

### 19.5 第三方库太吵，可以改它的源码吗？

通常先调整它使用的记录器级别。

```python
logging.getLogger("some_library").setLevel(logging.WARNING)
```

但名称要依据该库实际使用的记录器，而不是猜测安装包名称一定相同。

如果子记录器自己明确设置了级别，或挂了独立处理器，也要看实际层级。

不要为了压住一个组件而把所有错误输出全局关闭。

## 第二十章：结构化日志——一行 JSON 有什么意义？

### 20.1 先说明目标

纯文本非常适合人直接看。

如果你希望按事件类型、文件名、成功数量查询，JSON 行可以保留字段结构。

每条记录一行 JSON，常叫 JSON Lines 的使用方式。

它不是标准库额外提供的专用数据库，也不保证自动索引。

我们只写一个小格式器，把明确允许的字段导出。

### 20.2 完整格式器

```python
import json
import logging
from datetime import datetime, timezone


class JsonEventFormatter(logging.Formatter):
    def format(self, record):
        item = {
            "time": datetime.fromtimestamp(record.created, timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        for key in ("event", "run_id", "source_file", "success_count"):
            if hasattr(record, key):
                item[key] = getattr(record, key)
        if record.exc_info:
            item["exception"] = self.formatException(record.exc_info)
        return json.dumps(item, ensure_ascii=False)
```

### 20.3 为什么没有把 `record.__dict__` 整个导出？

里面可能有内部字段、参数对象、异常信息和不适合直接 JSON 序列化的值。

即使可以转换，也不代表每个字段都值得长期记录。

明确挑选字段，更容易保持结构稳定。

业务扩展时，可以逐项决定新字段的类型与含义。

### 20.4 消息里有换行，会破坏一行一条吗？

JSON 编码会把字符串中的换行表示成转义序列。

处理器再在整段 JSON 后面写一个记录终止换行。

因此，在正确使用 JSON 编码而不是手工拼字符串时，可以保持物理上一条记录一行的基本结构。

不要自己写：

```text
'{"message":"' + user_text + '"}'
```

引号、反斜杠与换行都会带来问题。

### 20.5 自定义字段含有 Path，会自动转换吗？

这个格式器只对明确的字段取值，然后交给 `json.dumps`。

如果字段值不是支持的 JSON 类型，可能失败。

最直接的规则是发记录时就把路径转换成字符串，把计数保存为整数。

也可以在格式器里按字段做明确转换，但不要把所有对象一律 `str()` 后就认为数据语义仍然一样。

结构化日志要保留的是稳定可解释的数据，不只是“能输出”。

## 第二十一章：完整实践一——批量读取成绩文件，记录进度、跳过原因与最终汇总

### 21.1 我们实际要处理什么？

一个目录里有多个 UTF-8 文本文件。

每个非空行表示一个整数成绩，允许范围为零到一百。

单行无效时，记录警告并跳过这一行。

某个文件无法解码或读取时，记录文件级失败，继续处理其他文件。

最终返回成功文件数、失败文件数、有效成绩数与成绩总和。

这个程序不修改输入文件。

日志帮助我们解释发生了什么，返回结果仍然明确表达计算结果。

### 21.2 为什么把日志记录器作为参数？

这样，处理逻辑不负责决定输出到终端还是文件。

真实运行可以传应用记录器。

验证时可以传带内存处理器的记录器。

业务逻辑不需要为了检查输出再重写一套。

### 21.3 完整程序

```python
import io
import logging
from pathlib import Path
from tempfile import TemporaryDirectory


def collect_scores(paths, logger):
    result = {"ok_files": 0, "failed_files": 0, "score_count": 0, "score_sum": 0}
    for raw_path in paths:
        path = Path(raw_path)
        source = path.name
        logger.info("开始读取", extra={"source_file": source})
        try:
            text = path.read_text(encoding="utf-8")
        except (OSError, UnicodeError):
            result["failed_files"] += 1
            logger.exception("文件读取失败", extra={"source_file": source})
            continue

        file_count = 0
        for line_number, line in enumerate(text.splitlines(), start=1):
            stripped = line.strip()
            if not stripped:
                continue
            try:
                score = int(stripped)
            except ValueError:
                logger.warning(
                    "第 %d 行不是整数，已跳过", line_number,
                    extra={"source_file": source},
                )
                continue
            if not 0 <= score <= 100:
                logger.warning(
                    "第 %d 行成绩超出范围，已跳过", line_number,
                    extra={"source_file": source},
                )
                continue
            result["score_count"] += 1
            result["score_sum"] += score
            file_count += 1
            logger.debug(
                "第 %d 行有效", line_number,
                extra={"source_file": source},
            )
        result["ok_files"] += 1
        logger.info("读取完成，有效成绩 %d 项", file_count, extra={"source_file": source})

    logger.info(
        "汇总：成功文件 %d，失败文件 %d，有效成绩 %d",
        result["ok_files"], result["failed_files"], result["score_count"],
        extra={"source_file": "-", "event": "summary"},
    )
    return result


def run_score_import_demo():
    output = io.StringIO()
    logger = logging.getLogger("lesson.score_import")
    logger.setLevel(logging.INFO)
    logger.propagate = False
    handler = logging.StreamHandler(output)
    handler.setFormatter(logging.Formatter("%(levelname)s|%(source_file)s|%(message)s"))
    logger.addHandler(handler)
    try:
        with TemporaryDirectory() as directory:
            root = Path(directory)
            first = root / "first.txt"
            second = root / "second.txt"
            first.write_text("80\nabc\n101\n90\n", encoding="utf-8")
            second.write_text("100\n\n", encoding="utf-8")
            result = collect_scores([first, second], logger)
        print(result)
        print(output.getvalue(), end="")
        return result, output.getvalue()
    finally:
        logger.removeHandler(handler)
        handler.close()


if __name__ == "__main__":
    run_score_import_demo()
```

主要输出：

```text
{'ok_files': 2, 'failed_files': 0, 'score_count': 3, 'score_sum': 270}
INFO|first.txt|开始读取
WARNING|first.txt|第 2 行不是整数，已跳过
WARNING|first.txt|第 3 行成绩超出范围，已跳过
INFO|first.txt|读取完成，有效成绩 2 项
INFO|second.txt|开始读取
INFO|second.txt|读取完成，有效成绩 1 项
INFO|-|汇总：成功文件 2，失败文件 0，有效成绩 3
```

### 21.4 为什么没有记录整行原始文本？

定位问题需要知道文件和行号，未必需要把完整输入重复保存进日志。

对于这个练习，写“不是整数”已经能说明跳过原因。

如果实际需要记录原值，也应考虑内容长度、是否包含不必要信息，以及文本中换行会不会干扰阅读。

日志应该围绕诊断目标选信息，而不是默认复制输入数据。

### 21.5 `ok_files` 的含义是什么？

它表示文件成功读取并完成了行处理流程。

即使其中有无效行，文件仍然计入成功文件。

如果你的业务要求“任何一行无效就整文件失败”，需要改变状态模型和控制流。

这里特别解释计数定义，是因为相同字段名在不同项目里可能代表不同意思。

不要让日志里写“成功两个文件”，读者却误以为每一行都合法。

### 21.6 文件读取失败路径怎么练习？

把一个不存在的路径加入输入列表。

程序应增加 `failed_files`，记录异常，并继续处理后面的文件。

你也可以创建含非法 UTF-8 字节的文件，验证解码失败路径。

两种失败都在本例明确捕获的范围内。

本例没有捕获所有 `Exception`，因此程序自身的逻辑错误不会一律被伪装成普通输入文件失败。

### 21.7 为什么每行有效只写 DEBUG？

处理很多行时，逐行消息会很密集。

日常使用只看文件开始、完成和警告，已经能理解主要进展。

需要定位具体解析顺序时，再把相关入口与处理器允许到 DEBUG。

这就是按级别设计事件，而不是把每一行都用 INFO 刷满终端。

### 21.8 如果要失败以后整个程序返回非零退出码呢？

让入口根据返回统计决定。

例如存在失败文件，就返回一；否则返回零。

不要让 `logger.error()` 隐含承担退出职责。

计算结果、诊断信息与退出状态可以相互关联，但仍然是不同输出。

## 第二十二章：完整实践二——可重复初始化，但不会每次多打印一份

### 22.1 重复配置为什么常出现在开发环境？

你在交互窗口多次运行一个单元。

测试连续创建应用对象。

程序重新加载某段配置。

如果每次都新建处理器并挂上去，第二次可能输出两份，第三次三份。

我们写一个明确由自己拥有的组件配置函数。

它只管理指定名称的记录器，替换自己这套练习输出。

### 22.2 完整程序

```python
import io
import logging


def configure_owned_logger(stream):
    logger = logging.getLogger("lesson.reconfigure")
    logger.disabled = False
    logger.setLevel(logging.INFO)
    logger.propagate = False
    for old in list(logger.handlers):
        logger.removeHandler(old)
        old.close()
    handler = logging.StreamHandler(stream)
    handler.setFormatter(logging.Formatter("%(levelname)s|%(message)s"))
    logger.addHandler(handler)
    return logger


def run_reconfiguration_demo():
    output = io.StringIO()
    logger = configure_owned_logger(output)
    logger.info("第一次配置")
    logger = configure_owned_logger(output)
    logger.info("第二次配置")
    try:
        print(output.getvalue(), end="")
        return output.getvalue(), len(logger.handlers)
    finally:
        for handler in list(logger.handlers):
            logger.removeHandler(handler)
            handler.close()


if __name__ == "__main__":
    run_reconfiguration_demo()
```

输出：

```text
INFO|第一次配置
INFO|第二次配置
```

### 22.3 为什么不直接 `logger.handlers.clear()`？

它会改变关联列表，但没有表达对每个旧处理器执行关闭。

文件处理器可能仍然持有资源。

明确逐个移除和关闭，更容易让人看懂资源归属。

当然，如果某个处理器本来由别处拥有、还被其他记录器共享，就不能随意关闭它。

这个实践之所以可以全部替换，是因为函数明确拥有这条练习记录器的全部处理器配置。

### 22.4 `hasHandlers()` 能判断自己有没有处理器吗？

它可能沿祖先链检查，因此不是只看当前 `logger.handlers`。

如果你的目的是判断当前记录器直接关联了多少个处理器，应看直接关联列表。

如果目的是知道记录有没有可能向上找到出口，`hasHandlers()` 更接近这个问题。

同样的判断名字，要放在你真正想问的问题里使用。

### 22.5 为什么不每次生成随机记录器名称？

那会让旧记录器留在管理结构里，名称数量不断增长。

而且无法稳定地按组件调整级别。

修复重复初始化应处理配置生命周期，而不是通过不停换名字掩盖旧状态。

## 第二十三章：完整实践三——观察文件轮转，而不依赖猜测文件大小

### 23.1 用有序事件观察保留策略

我们连续写二十条带序号的消息。

轮转阈值故意设得很小，只保留两个历史文件。

结束后，读取当前文件与历史文件，看看最新事件在哪里。

这里不承诺每个文件严格包含相同条数，因为轮转条件与格式化长度有关。

我们检查真正重要的性质：文件数量受保留策略限制，最新事件仍然可以从保留文件中找到。

### 23.2 完整程序

```python
import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path
from tempfile import TemporaryDirectory


def run_rotation_demo():
    with TemporaryDirectory() as directory:
        path = Path(directory) / "events.log"
        logger = logging.getLogger("lesson.rotation")
        logger.setLevel(logging.INFO)
        logger.propagate = False
        handler = RotatingFileHandler(
            path, maxBytes=100, backupCount=2, encoding="utf-8"
        )
        handler.setFormatter(logging.Formatter("%(message)s"))
        logger.addHandler(handler)
        try:
            for index in range(20):
                logger.info("event-%02d payload-xxxxxxxx", index)
        finally:
            logger.removeHandler(handler)
            handler.close()

        files = sorted(Path(directory).glob("events.log*"))
        contents = {item.name: item.read_text(encoding="utf-8") for item in files}
        print([item.name for item in files])
        print("最新事件保留：", "event-19" in contents["events.log"])
        return contents


if __name__ == "__main__":
    run_rotation_demo()
```

在本例参数与输入下，主要观察结果是：

```text
['events.log', 'events.log.1', 'events.log.2']
最新事件保留： True
```

### 23.3 旧事件为什么不一定还在？

因为只保留两个历史文件。

轮转多次后，超出保留范围的旧内容会被淘汰。

所以轮转不是无限历史保存。

如果你需要保留一整个月，不能只凭 `backupCount=2` 就认为实现了时间保留要求。

大小轮转的历史时间跨度取决于日志产生速度。

### 23.4 能不能给历史文件加压缩？

处理器提供相应的命名与轮转定制能力，可以配合压缩。

但压缩耗时、文件命名与清理规则会一起变化。

没有实际需求时，先使用默认规则通常更容易维护。

一旦自定义名称，应确认历史文件排序与删除逻辑仍能识别需要保留的文件。

### 23.5 为什么先关闭再读取内容？

这样可以让验证不依赖仍打开的文件流状态，也更符合 Windows 对打开文件的限制。

我们还在临时目录退出前完成所有读取。

目录离开上下文后会被清理，不会在用户项目里留下二十条演示事件。

这和真实应用的长期日志目录是两种生命周期。

## 第二十四章：完整实践四——用 JSON 记录任务事件，并重新读回来验证

### 24.1 不只看输出像不像 JSON

一个字符串看起来像大括号包起来，不表示它一定是合法 JSON。

最直接的验证，是逐行交给 `json.loads()`。

然后检查字段类型与值。

本例沿用第二十章的 `JsonEventFormatter`。

### 24.2 完整调用过程

```python
import io
import json
import logging


def run_json_log_demo():
    output = io.StringIO()
    logger = logging.getLogger("lesson.json_events")
    logger.setLevel(logging.INFO)
    logger.propagate = False
    handler = logging.StreamHandler(output)
    handler.setFormatter(JsonEventFormatter())
    logger.addHandler(handler)
    try:
        logger.info(
            "处理完成\n下一步生成报告",
            extra={"event": "summary", "run_id": "demo-1", "success_count": 3},
        )
    finally:
        logger.removeHandler(handler)
        handler.close()

    lines = output.getvalue().splitlines()
    records = [json.loads(line) for line in lines]
    print("物理记录行数：", len(lines))
    print(records[0]["event"], records[0]["success_count"])
    print("消息保留换行：", "\n" in records[0]["message"])
    return records, output.getvalue()


if __name__ == "__main__":
    run_json_log_demo()
```

这里需要先定义第二十章的格式器；如果单独保存脚本，把那个类放在本程序导入以后即可。

主要输出：

```text
物理记录行数： 1
summary 3
消息保留换行： True
```

### 24.3 为什么展示验证结果，而不写死完整时间？

记录时间来自真正发生事件的时刻。

它不应该为了让例子看上去整齐，就假装每次都是同一个固定时间。

我们验证的是时间字段可以被格式化，其他业务字段能正确往返。

如果要严格验证时间文本，也可以直接构造带固定 `created` 的 `LogRecord`，然后测试格式器。

这属于下一层更细的单元验证，而不是把实时输出当作常量。

### 24.4 JSON 里保存异常，会不会又变成很多物理行？

格式器先把异常追踪构造成字符串，再由 `json.dumps` 转义字符串中的换行。

因此仍能保持一条物理 JSON 行。

读取后，异常字段中可以恢复成包含换行的文本。

如果终端展示工具主动展开它，看起来可能是多行，但这与磁盘文件中的原始行分隔是两个层面。

## 第二十五章：完整实践五——同一条事件，一个目标保留详情，一个目标隐藏字段

### 25.1 为什么强调“同一条记录”？

假设一个事件同时写向两个出口。

本地诊断目标希望看见文件名。

另一个摘要目标只需要知道发生了读取事件，不需要具体文件名。

如果你直接在第一个处理器的过滤器里修改 `record.source_file`，第二个处理器可能也看见修改后的内容。

因为它们原本共享同一条记录对象。

我们用 Python 3.12 的新记录替换能力，让某个出口拥有独立修改。

### 25.2 Python 3.12 及以上的完整示例

```python
# Python 3.12+
import copy
import io
import logging


class HideSourceForThisHandler(logging.Filter):
    def filter(self, record):
        replacement = copy.copy(record)
        replacement.source_file = "<摘要省略>"
        return replacement


def run_filter_copy_demo():
    detailed = io.StringIO()
    summary = io.StringIO()
    logger = logging.getLogger("lesson.filter_copy")
    logger.setLevel(logging.INFO)
    logger.propagate = False

    detail_handler = logging.StreamHandler(detailed)
    summary_handler = logging.StreamHandler(summary)
    formatter = logging.Formatter("%(source_file)s|%(message)s")
    detail_handler.setFormatter(formatter)
    summary_handler.setFormatter(formatter)
    summary_handler.addFilter(HideSourceForThisHandler())
    logger.addHandler(summary_handler)
    logger.addHandler(detail_handler)
    try:
        logger.info("读取完成", extra={"source_file": "scores.csv"})
    finally:
        for handler in (summary_handler, detail_handler):
            logger.removeHandler(handler)
            handler.close()
    print("摘要：", summary.getvalue().strip())
    print("详情：", detailed.getvalue().strip())
    return summary.getvalue(), detailed.getvalue()


if __name__ == "__main__":
    run_filter_copy_demo()
```

输出：

```text
摘要： <摘要省略>|读取完成
详情： scores.csv|读取完成
```

### 25.3 为什么把摘要处理器放在前面？

为了让实验更有辨别力。

如果摘要处理器错误地原地修改记录，后面的详情处理器就会受到影响。

现在它返回副本，所以详情仍然拿到原字段。

我们没有依靠“先输出详情，后修改原记录”来掩盖共享问题。

### 25.4 浅复制够不够？

本例只替换顶层的字符串字段，所以浅复制足够表达目标。

如果你在副本里原地修改某个嵌套字典或列表，底层对象仍可能共享。

这时需要按实际字段模型复制相应数据。

不要因为使用了 `copy.copy`，就宣称所有嵌套对象完全隔离。

### 25.5 Python 3.11 应该怎么办？

不要直接复制这段并假设返回记录会触发同样的替换行为。

可以在自定义处理器中复制后再格式化，或者让不同格式器只选择自己需要显示的字段，而不修改原记录。

对于“摘要不显示文件名”这种简单需求，使用不含 `source_file` 的摘要格式器就足够了。

这里使用复制是为了理解版本能力与对象共享，而不是为了把一个简单输出需求复杂化。

### 25.6 隐藏显示与删除原始数据一样吗？

不一样。

摘要出口不展示某字段，不表示原记录从未包含该字段，也不表示其他出口不会保留。

如果根本不应该记录某类内容，应在事件设计时就不传入。

这个例子只表达两个输出视角，不承担完整的信息治理模型。

## 第二十六章：排查日志问题时，沿真实路径一步一步问

### 26.1 我调用了 `info`，为什么什么都没有？

先看记录器有效级别。

```python
print(logger.getEffectiveLevel())
print(logger.isEnabledFor(logging.INFO))
```

如果入口本身拒绝 INFO，后面自然看不到。

再看 `logger.disabled` 是否为真，以及有没有全局 `logging.disable` 的影响。

入口允许以后，再检查当前记录器与祖先是否有处理器，传播是否提前停止。

最后检查处理器级别、过滤器和输出目标。

这条顺序比反复添加 `basicConfig` 更能定位问题。

### 26.2 终端有消息，文件没有，说明什么？

至少说明某条输出路径接收了事件。

但文件目标可能有不同级别、过滤器、路径、编码或写入错误。

先确认你正在查看的是实际配置的文件。

相对路径按工作目录解释，是最常见原因之一。

再确认文件处理器真的挂到会被经过的记录器路径上。

一个终端消息成功，不足以证明所有处理器都成功输出。

### 26.3 文件存在但为空，为什么？

文件可能在处理器创建时就打开了。

但没有任何记录通过这个处理器的筛选。

也可能程序尚未走到真正业务日志位置。

所以“文件已建立”只能说明打开或创建发生过，不代表已经记录过一条有效事件。

使用 `delay=True` 后，文件建立时间又会不同。

这些都是生命周期问题，不一定是日志库坏了。

### 26.4 同一行出现两份，先检查什么？

先检查当前记录器直接关联了几个处理器。

再检查祖先有没有类似目标，当前传播是否开启。

还要看应用是否重复执行了配置函数，每次新建一个处理器。

如果同一业务函数本身真的调用两次，也会产生两条独立记录。

可以暂时记录事件编号或观察调用路径，区分重复执行与重复分发。

不要只凭文本相同，就直接认定是哪一种原因。

### 26.5 中文乱码，是 logging 专属问题吗？

很多时候是编码链路问题。

写文件时明确 UTF-8，读取时也按 UTF-8。

终端显示则还涉及终端编码与字体等环境因素。

如果文件内容读取正确，但终端显示异常，应分开检查两个目标。

`encoding="utf-8"` 配给文件处理器，不会自动改变所有终端流的编码设置。

### 26.6 为什么错误日志里显示 `--- Logging error ---`？

这可能是日志系统在报告格式化或输出过程自己的错误。

例如消息模板占位符与参数不匹配，格式器需要不存在的额外字段，或目标写入失败。

请先看这段诊断里真正的异常类型和提示。

它不一定来自你要记录的业务异常。

把日志内部错误与业务错误分开，才能修对位置。

### 26.7 `%d` 和 `%s` 怎么选？

`%s` 更通用，会使用对象的字符串表示。

`%d` 表达整数格式意图，但传入不合适对象可能导致格式化失败。

如果字段已经严格验证为整数，使用 `%d` 很清楚。

如果你只是想记录一个任意状态描述，`%s` 更符合用途。

这里不是要求所有消息都用同一个占位符，而是让格式与数据契约一致。

### 26.8 消息参数里有字典，为什么有时行为不同？

日志支持相应的映射式消息格式。

```python
logger.info("用户 %(name)s 完成 %(count)d 项", {"name": "Alice", "count": 3})
```

这里的映射用于消息模板替换。

它与 `extra` 把字段加入记录，是两种不同用途。

前者决定 `message` 文本，后者提供格式器或其他处理流程可以使用的记录属性。

不要把一个字典参数的出现都笼统理解成业务字段注入。

### 26.9 为什么拿到处理器后，不直接调用 `emit`？

正常业务应通过记录器发事件。

处理器的公开处理流程还包括过滤、锁与错误处理等相关组织。

直接跳到 `emit`，可能绕过你原本期待的步骤。

只有在实现和测试自定义处理器时，才需要更仔细地讨论这些底层方法。

日常记录不要绕过系统主流程，只为少写几个字。

### 26.10 `logging.shutdown()` 可以随便调用吗？

它会刷新并关闭系统管理的处理器，通常也在解释器正常退出阶段安排。

如果程序还有后台线程继续记录，你提前关闭输出就可能造成生命周期冲突。

一个库更不应该随便替整个宿主调用它。

应用应先停止生产任务，再停止队列监听等组件，最后安排输出资源收尾。

正常关闭与进程被强制终止不是同一保证。

### 26.11 进度条、结果数据和日志能放在一起吗？

技术上可以，但可能互相干扰。

终端进度条经常使用回车更新同一行，普通日志会插入新的行。

正式机器可读结果又可能要求纯 JSON 或 CSV。

先确定每个通道的用途：结果写哪里、诊断写哪里、交互进度写哪里。

必要时使用对应工具提供的日志协调能力，而不是让所有输出任意竞争同一个文本流。

### 26.12 有了日志，就可以不处理异常吗？

不可以这样理解。

日志帮助你观察，异常处理决定程序行为。

你仍然要决定是否重试、跳过、回滚、返回失败或终止。

只写一行 ERROR 然后继续返回“成功”，会让调用者得出错误结论。

同样，所有错误都直接终止也未必符合批处理需求。

日志应该准确描述程序真正采取的处理方式。

## 第二十七章：练习题与参考思路——把配置判断变成可以解释的行为

### 27.1 练习一：只显示 INFO 及以上

建立一个独立脚本，发出 DEBUG、INFO、WARNING 三条消息。

要求只显示后两条，并且不在每处调用外面手写 `if`。

参考答案：

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s|%(message)s")
logging.debug("细节")
logging.info("正常开始")
logging.warning("有可恢复问题")
```

请在新进程里运行，避免交互环境已有处理器干扰。

这道题验证的是级别阈值，不是异常控制流。

### 27.2 练习二：记录错误以后，程序是否停止？

运行：

```python
import logging

logging.error("模拟错误")
print("后面的代码")
```

解释为什么第二行仍然执行。

参考答案：日志调用发出事件，不会仅因为 ERROR 级别就抛出业务异常或结束进程。

如果要求退出，应该由入口明确返回或抛出相应控制信号。

### 27.3 练习三：记录器 DEBUG，处理器 WARNING

分别发 DEBUG、INFO、WARNING、ERROR。

预测这个处理器会收到什么。

参考答案：WARNING 与 ERROR 可以通过处理器阈值。

前两条即使通过记录器入口，也被这个出口拒绝。

如果还有其他更宽松处理器，它们仍可能输出前两条。

题目问的是某个处理器，不是整个应用是否有任何输出。

### 27.4 练习四：记录器 WARNING，处理器 DEBUG

仍然发出四种级别。

参考答案：DEBUG 与 INFO 在记录器入口已被拒绝，处理器宽松不能把它们恢复。

这与上一题最终看到的消息可能相同，但拒绝发生在不同位置。

理解流程比只背最后两行结果更重要。

### 27.5 练习五：父 ERROR，子 DEBUG，父处理器 DEBUG

子级传播开启，没有额外过滤器。

子级发 DEBUG，是否可能输出？

参考答案：可以。

子级明确 DEBUG，所以入口允许。

传播直接交给父级处理器，不重新检查父记录器自身 ERROR 级别。

处理器 DEBUG 允许输出。

如果子级改成 NOTSET，继承到 ERROR，结果就不同。

### 27.6 练习六：修复三次初始化产生三份输出

某配置函数每次都新建 StreamHandler 并添加到同一名字记录器。

要求修复，而且解释旧文件处理器如何释放。

参考思路：先明确配置函数是否拥有该记录器全部处理器。

如果拥有，逐个移除并关闭，再建立一份新配置。

如果不拥有，只能操作自己创建的那部分，不能顺手关闭别人共享的目标。

不要用每次换一个随机记录器名字作为主要解决方案。

### 27.7 练习七：辅助函数显示调用者行号

写 `notice(logger, message)`，内部调用 `logger.info`。

要求格式显示业务调用位置。

参考答案：

```python

def notice(logger, message):
    logger.info(message, stacklevel=2)
```

在单层辅助函数模型下，二让来源向外定位一层。

如果还叠了其他包装，应根据实际流程调整并验证。

### 27.8 练习八：只记录摘要事件

要求一个处理器只允许 `event="summary"`。

参考答案可以使用第十四章的 `EventFilter`。

请验证三种输入：正确事件、其他事件、完全没有 `event` 字段。

只有第一种通过。

过滤器应使用 `getattr(record, "event", None)` 等方式处理缺失，不要因为普通记录没有业务字段就意外抛错。

### 27.9 练习九：格式器要求 `run_id`，第三方记录没有怎么办？

给格式器提供默认值，或在明确目标上安排补字段过滤器。

本文基线可以使用：

```python
formatter = logging.Formatter(
    "%(run_id)s|%(message)s", defaults={"run_id": "-"}
)
```

如果需要结构化输出，也要考虑默认字段的类型，而不只是在文本中显示一个横杠。

### 27.10 练习十：为什么 DEBUG 关闭了，昂贵函数仍然执行？

分析：

```python
logger.debug("详情：%s", build_expensive_report())
```

参考答案：参数表达式在调用日志方法前求值。

`%s` 延迟的是对象到文本的格式化，不是任意函数调用。

可先用 `isEnabledFor` 判断，再构造报告。

但如果 DEBUG 开启，后续处理器过滤仍可能拒绝，所以它是入口判断，不是最终输出承诺。

### 27.11 练习十一：分别验证读取失败和解析失败

给成绩导入程序提供一个不存在文件，以及一个含 `abc` 的合法 UTF-8 文件。

参考答案：前者增加文件失败数，记录异常；后者文件读取成功，单行警告后继续。

解释为什么两者不能都计成“文件打不开”。

程序阶段不同，恢复策略也不同。

### 27.12 练习十二：JSON 消息里包含换行

记录文本 `第一行\n第二行`，用 JsonEventFormatter 输出，再逐物理行解析。

参考答案：应该只有一条物理 JSON 记录，解析后消息字符串仍包含真正换行。

如果你手工拼接 JSON，可能破坏这项性质。

因此验证必须使用 JSON 解析器，而不是只观察外形。

### 27.13 练习十三：队列监听器何时停止？

要求解释先后关系：停止业务生产、移除队列处理器、停止监听器、关闭目标处理器。

参考思路：先确保不会继续产生需要接收的消息，再让消费者处理停止协议，最后释放输出目标。

具体应用可能还需要等待工作线程或子进程结束。

不能让生产者仍在不断写，消费者却已经退出。

### 27.14 练习十四：日志轮转等于备份吗？

不是完整意义的业务备份。

它通常会淘汰超出保留范围的旧文件。

也不保存程序数据库、配置与输入文件的一致性快照。

如果需要事故恢复，应该有独立的备份与恢复方案。

运行日志只是其中可能有价值的一类诊断材料。

### 27.15 练习十五：为什么库不应强制 `basicConfig(force=True)`？

因为库不知道宿主的目标与策略。

强制配置可能关闭宿主处理器、改变级别与格式。

参考做法是获取自己的命名记录器，必要时加 NullHandler，由应用入口决定输出。

这道题让你区分“我写的是可导入组件”与“我拥有整个程序启动过程”。

### 27.16 练习十六：手写一个内存记录处理器

目标不是写文本，而是把记录对象放入列表，便于验证级别与业务字段。

参考答案：

```python
import logging


class CollectingHandler(logging.Handler):
    def __init__(self):
        super().__init__()
        self.records = []

    def emit(self, record):
        self.records.append(record)
```

通过正常记录器调用后，检查 `record.levelno`、`record.getMessage()` 与自定义属性。

注意保存的是记录对象引用；如果后面有人修改同一对象，列表中也会看到变化。

需要冻结数据时，可以在 emit 中选取并复制明确字段。

### 27.17 练习十七：捕获完整异常信息并继续传播

写一个函数，把 `int("bad")` 的失败记录下来，然后让调用者仍然收到 ValueError。

参考答案：

```python

def parse_with_context(text, logger):
    try:
        return int(text)
    except ValueError:
        logger.exception("整数解析失败")
        raise
```

这里裸 `raise` 重新抛出当前异常。

不要写成记录以后 `return None`，却对外宣称保持原本异常协议。

### 27.18 练习十八：有名字但没有输出目标

创建一个记录器，设 DEBUG，再设 `propagate=False`，不挂任何处理器。

发 INFO 和 WARNING，结果一定都是静默吗？

参考答案：不能忘记 lastResort 兜底处理器的存在。

默认兜底可处理 WARNING 及以上，因此 WARNING 可能出现，而 INFO 不会。

如果你明确需要某组件在未配置时不输出，可以考虑库的 NullHandler 模式。

这道题提醒你：排查必须包括兜底路径，而不只看 `logger.handlers`。

## 第二十八章：完整实践六——两个异步任务，怎样各自保留自己的任务编号？

### 28.1 为什么全局变量不适合这个场景？

假设任务 A 把全局 `current_run_id` 改成 A，然后等待文件或网络。

任务 B 在这段时间把它改成 B。

任务 A 恢复以后再写日志，就可能错误地带上 B 的编号。

问题来自共享可变状态，不是日志格式器不知道怎么打印。

`contextvars` 提供按当前执行上下文保存值的能力，很适合搭配过滤器获取当前任务字段。

我们让每个任务设置自己的编号，结束时恢复原状态。

### 28.2 完整程序

```python
import asyncio
import io
import logging
from contextvars import ContextVar


current_run = ContextVar("current_run", default="-")


class CurrentRunFilter(logging.Filter):
    def filter(self, record):
        record.run_id = current_run.get()
        return True


async def logged_job(run_id, logger):
    token = current_run.set(run_id)
    try:
        logger.info("开始")
        await asyncio.sleep(0)
        logger.info("结束")
    finally:
        current_run.reset(token)


async def gather_logged_jobs(logger):
    await asyncio.gather(logged_job("A", logger), logged_job("B", logger))


def run_context_logging_demo():
    output = io.StringIO()
    logger = logging.getLogger("lesson.context")
    logger.setLevel(logging.INFO)
    logger.propagate = False
    handler = logging.StreamHandler(output)
    handler.addFilter(CurrentRunFilter())
    handler.setFormatter(logging.Formatter("%(run_id)s|%(message)s"))
    logger.addHandler(handler)
    try:
        asyncio.run(gather_logged_jobs(logger))
        logger.info("主流程")
    finally:
        logger.removeHandler(handler)
        handler.close()
    print(output.getvalue(), end="")
    return output.getvalue()


if __name__ == "__main__":
    run_context_logging_demo()
```

这个程序的一种正常输出是：

```text
A|开始
B|开始
A|结束
B|结束
-|主流程
```

并发输出的全局顺序不应该在更复杂程序里被当作固定承诺。

真正要验证的是，每项任务的两条事件都使用自己的编号，而且主流程恢复默认值。

### 28.3 为什么保存 `token`？

设置上下文变量会返回一个可用于恢复的令牌。

在 `finally` 中恢复，可以让普通异常或取消等离开路径也执行清理。

如果你只在正常结束后恢复，一旦中途失败，当前任务后续逻辑可能继续带着旧上下文。

这与资源管理中的成对进入和退出有相似之处。

### 28.4 过滤器放在队列接收端，还能读到生产任务编号吗？

不应该这样假设。

接收线程有它自己的当前上下文。

如果到了接收端才调用 `current_run.get()`，得到的可能是默认值，而不是原任务上下文。

因此，应在生产端记录形成并送入队列之前，把所需业务字段放进记录。

日志记录跨执行边界传输时，应携带已经确定的上下文字段。

这比让接收端猜测“原来是谁发的”更清楚。

### 28.5 新线程、新进程会自动得到同一上下文吗？

上下文传播要看具体执行 API 与版本行为。

不要把异步 Task 中常见的上下文复制规则，直接套用到任意线程或新进程。

如果你明确把任务编号作为工作参数传入，并在工作边界设置上下文，通常更容易理解。

本文实践限定为一个事件循环内的异步任务，不宣称覆盖所有跨线程执行工具。

## 第二十九章：时间、告警与缓冲——把常见搭配放回具体需求

### 29.1 如果想让日志时间明确使用 UTC 呢？

可以配置格式器的转换函数。

```python
import logging
import time

formatter = logging.Formatter(
    "%(asctime)sZ|%(levelname)s|%(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)
formatter.converter = time.gmtime
```

这里的 Z 与 gmtime 配合，表达 UTC。

如果仍然使用本地时间，却只在末尾机械加 Z，会产生误导。

显示格式与实际时间基准必须一致。

### 29.2 要计算一次任务用了多久，直接看两行时间差吗？

可以作为粗略观察，但程序内部最好使用适合经过时间的计时器。

```python
from time import perf_counter

started = perf_counter()
# 在这里执行具体工作。
elapsed = perf_counter() - started
logger.info("任务耗时 %.3f 秒", elapsed)
```

`perf_counter` 返回值适合做差，不适合直接解释成某年某月某日。

日志创建时间则适合描述事件时刻。

一个回答“什么时候”，一个回答“经历多久”，用途不同。

### 29.3 Python 的 warnings 能进入日志吗？

可以使用：

```python
import logging
import warnings

logging.captureWarnings(True)
warnings.warn("这个旧选项以后将移除", UserWarning)
logging.captureWarnings(False)
```

被捕获的 warning 会交给相应的 `py.warnings` 记录器路径。

不过 warnings 自己的筛选规则仍然存在，不是每次调用都一定产生一个新告警。

同一个警告可能按规则只显示一次。

如果练习要观察，需要理解 warnings 的过滤设置，并在结束后恢复自己改变的状态。

### 29.4 warning 级别与 Python Warning 是同一个机制吗？

名称相似，但不是同一个入口。

`logger.warning()` 发出 logging 事件。

`warnings.warn()` 使用 Python 告警系统，有不同类别与过滤规则。

`captureWarnings` 是把两套机制连接起来的一种方式。

不要因为都叫 warning，就以为修改一个日志级别能改变所有 Python 告警的产生政策。

### 29.5 `MemoryHandler` 可以在出错时把前面记录一起输出吗？

它可以先把记录保存在内存缓冲中，在容量达到或遇到指定级别时交给目标处理器。

```python
import logging
from logging.handlers import MemoryHandler

buffering = MemoryHandler(
    capacity=100,
    flushLevel=logging.ERROR,
    target=handler,
)
```

这里沿用你已经建立的目标 `handler`。

它可以帮助保留错误之前的上下文，但要理解默认关闭时也可能刷新缓冲。

如果你希望成功结束时完全丢弃，不能只看到“ERROR 时 flush”就假设其他路径永远不输出。

应明确检查 `flushOnClose` 等行为，并设计正常结束与异常结束如何处理。

### 29.6 缓冲一定比直接写快吗？

减少目标调用可能有帮助，但缓冲本身也占内存，最终刷新仍然需要时间。

如果目标处理很慢，刷新时可能出现集中等待。

如果进程突然终止，尚未交付的数据还可能丢失。

所以缓冲、队列、轮转解决的是不同维度问题，不能因为名字都属于 handlers 就当作可以随意互换。

### 29.7 网络处理器是不是开箱即用的远程日志平台？

标准库提供 SocketHandler、HTTPHandler 等接口，但它们只是相应传输方式的构件。

你还需要考虑接收端、认证、格式、网络故障、积压和部署方式。

某些标准示例会使用对象序列化协议，接收端不能把不可信输入直接当作安全数据执行解析。

没有实际远程需求时，先把本地文件、终端和队列流程理解清楚更有价值。

本篇不把一段向任意地址发送数据的示例混进普通入门实践中。

### 29.8 自定义日志级别要不要一开始就做？

大多数日常脚本，用现有五种级别已经足够。

新增级别涉及数字、名称、调用入口和第三方工具兼容。

如果只是想区分导入事件与汇总事件，更适合使用业务字段，而不是不断创造新严重程度。

级别用于事件影响，事件类型用于描述发生了什么。

这两个维度分开，后续查询与配置更容易理解。

## 第三十章：完整实践七——用记录对象验证业务事件，而不是只比较一大段文字

### 30.1 为什么要直接收集记录？

文本格式可能随时间、文件路径和行号变化。

如果你的验证目标是“失败时发出 ERROR，带上对应文件名”，直接检查记录字段更准确。

这样，不会因为你把分隔符从竖线换成空格，就让业务行为验证全部失效。

### 30.2 完整程序

```python
import logging


class SnapshotHandler(logging.Handler):
    def __init__(self):
        super().__init__()
        self.events = []

    def emit(self, record):
        self.events.append({
            "level": record.levelno,
            "message": record.getMessage(),
            "source_file": getattr(record, "source_file", None),
            "has_exception": record.exc_info is not None,
        })


def parse_score_text(text, logger, source_file):
    try:
        score = int(text)
    except ValueError:
        logger.exception("成绩不是整数", extra={"source_file": source_file})
        raise
    if not 0 <= score <= 100:
        logger.warning("成绩超出范围", extra={"source_file": source_file})
        return None
    logger.info("成绩有效", extra={"source_file": source_file})
    return score


def run_record_validation_demo():
    logger = logging.getLogger("lesson.record_validation")
    logger.setLevel(logging.DEBUG)
    logger.propagate = False
    handler = SnapshotHandler()
    logger.addHandler(handler)
    try:
        assert parse_score_text("90", logger, "a.txt") == 90
        assert parse_score_text("200", logger, "b.txt") is None
        try:
            parse_score_text("bad", logger, "c.txt")
        except ValueError:
            pass
        assert [event["level"] for event in handler.events] == [
            logging.INFO, logging.WARNING, logging.ERROR,
        ]
        assert handler.events[-1]["source_file"] == "c.txt"
        assert handler.events[-1]["has_exception"] is True
        print("三种事件的级别、上下文与异常信息均符合预期")
        return handler.events
    finally:
        logger.removeHandler(handler)
        handler.close()


if __name__ == "__main__":
    run_record_validation_demo()
```

### 30.3 为什么把异常对象转换成布尔标记？

这个验证只关心是否带有异常信息。

保存完整异常追踪可能延长相关对象的引用生命周期。

我们选取需要的字段建立快照，避免为了验证一个标志就长期持有整个记录与追踪。

如果要检查异常类型，也可以明确提取类型名称。

验证数据越贴近目标，通常越容易理解。

### 30.4 `assert` 能当正式输入检查吗？

这里用于练习验证。

正式业务输入检查应使用明确的异常或返回协议，因为 Python 优化模式可能移除断言。

不要把 `assert 0 <= score <= 100` 当成必须始终执行的数据校验。

本例在业务函数里用普通判断，在验证函数里用断言，职责是分开的。

### 30.5 如果项目已经使用 pytest 呢？

可以利用它的日志捕获能力检查记录与消息。

但仍应保留同样的思路：先明确要验证级别、字段、异常还是最终文本。

如果每个测试都只比较一大段带时间与路径的输出，维护成本会很高。

选择结构化断言，是为了让失败指向真正改变的行为。

## 第三十一章：把一个多模块项目的日志职责安排清楚

### 31.1 一个小项目结构

```text
study_project/
├── main.py
└── study/
    ├── __init__.py
    ├── reader.py
    └── report.py
```

`reader.py` 负责读取。

`report.py` 负责汇总。

`main.py` 负责接收命令行参数、建立配置、调用流程并决定退出状态。

日志配置放在入口，两个模块各自用 `getLogger(__name__)`。

### 31.2 `reader.py` 应该写什么？

```python
import logging
from pathlib import Path

logger = logging.getLogger(__name__)


def read_lines(path):
    path = Path(path)
    logger.debug("准备读取 %s", path)
    text = path.read_text(encoding="utf-8")
    lines = text.splitlines()
    logger.info("读取完成，共 %d 行", len(lines))
    return lines
```

它不决定日志文件名，也不强制修改根配置。

读取失败时可以让异常交给任务边界处理。

如果要补充异常上下文，可以抛出清楚的业务异常，但不一定每一层都打印完整追踪。

### 31.3 `main.py` 应该决定什么？

```python
import logging
from study.reader import read_lines


def main():
    logging.basicConfig(
        level=logging.INFO,
        format="%(levelname)s|%(name)s|%(message)s",
    )
    logger = logging.getLogger(__name__)
    try:
        lines = read_lines("scores.txt")
    except (OSError, UnicodeError):
        logger.exception("读取任务失败")
        return 1
    logger.info("任务完成，接下来处理 %d 行", len(lines))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

这段多模块示例需要按前述结构保存，且准备输入文件。

入口把失败转换成明确退出状态，记录一次任务级错误。

库函数仍然可以被其他入口复用。

### 31.4 为什么导入了模块以后再配置也可以？

模块导入时只是取得命名记录器，没有立即强制输出或配置。

真正发记录时，会使用当前配置状态。

因此，先导入函数，再在 `main()` 配置，并不矛盾。

但如果模块导入阶段就执行了大量业务并发日志，配置顺序就更难控制。

所以避免导入时启动真正任务，也有助于日志初始化清晰。

### 31.5 怎么从命令行打开详细模式？

由命令行参数决定入口级别，例如有 `--verbose` 时使用 DEBUG。

如果同时有文件和终端两个目标，应明确这个参数调整哪一个。

可以只把终端从 INFO 降到 DEBUG，文件一直保留 DEBUG。

也可以只调整某个组件，减少第三方库的细节噪声。

“详细模式”是应用政策，不是 logging 自动定义好的唯一行为。

### 31.6 最后核对哪些具体问题？

配置由谁执行，是否只执行预期次数？

每个模块的记录器名称是否稳定？

记录器入口允许的最低级别，是否能覆盖最详细目标的需要？

处理器目标、编码、格式和保留政策是否明确？

业务错误是否与返回结果、异常传播、退出码一致？

正常关闭前，是否已经停止仍会发记录的后台工作？

这些问题都能对应到前面真实的代码路径，而不是一份与程序无关的口号。

## 第三十二章：读完之后，用这一张表快速找回工具

| 你想解决的问题 | 常用入口 | 需要记住的边界 |
| --- | --- | --- |
| 小脚本快速配置 | `basicConfig` | 已有根处理器时通常不重复配置，`force` 有全局影响 |
| 模块有清楚来源名称 | `getLogger(__name__)` | 名字稳定，不为每个请求无限创建记录器 |
| 控制事件入口 | `Logger.setLevel` | NOTSET 可能继承，入口拒绝后出口无法恢复 |
| 控制单个输出目标 | Handler 及其级别 | 同一记录可经过多个出口，注意传播重复 |
| 决定一行长什么样 | `Formatter` | 外层 style 不改变默认消息参数约定 |
| 输出完整异常追踪 | `exception` / `exc_info` | 在异常上下文使用，不自动重新抛出 |
| 调整辅助函数来源位置 | `stacklevel` | 调整记录归属，不重写异常执行路径 |
| 加任务编号等字段 | `extra` / `LoggerAdapter` | 避免字段冲突与缺失，合并规则要明确 |
| 按业务事件筛选 | Filter | 记录器与处理器位置不同，3.12 可返回新记录 |
| 防止文件无限增长 | 轮转文件处理器 | 保留有限历史，不提供多进程共同轮转保证 |
| 把慢目标交给另一个线程 | `QueueHandler` / `QueueListener` | 容量、停止顺序与交付保证需要设计 |
| 集中描述配置关系 | `dictConfig` | 注意已有记录器策略与配置可信来源 |
| 给库提供安静默认行为 | `NullHandler` | 不应擅自接管宿主根配置 |
| 验证真实记录内容 | 自定义收集处理器 | 选择所需字段快照，区分文本与业务行为 |

完整参数可以在 [Python 3.11 logging 参考](https://docs.python.org/3.11/library/logging.html)、[处理器参考](https://docs.python.org/3.12/library/logging.handlers.html) 与 [官方 Logging Cookbook](https://docs.python.org/3.12/howto/logging-cookbook.html) 中继续对照。

查资料时，先带着具体问题，例如“这条记录在哪个门槛被拒绝”“这个处理器由谁关闭”，再定位对应接口，会比从头背所有类名更容易形成实际判断。
