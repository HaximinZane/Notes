# Python `with` 与上下文管理超详细教程：从文件关闭，到资源归属、异常退出与 `contextlib`

## 零基础先导：`with` 到底在替你保证什么？

你可能最早见到：

```python
with open("notes.txt", encoding="utf-8") as f:
    text = f.read()
```

不要把 `with` 误解为“读文件专属语法”，它是一种**上下文管理语法**。主要用途是：进入一段代码之前取得/准备资源，**无论正常结束还是发生异常，都执行约定的退出操作**。

先不使用 `with`，你可能写：

```python
f = open("notes.txt", encoding="utf-8")
try:
    text = f.read()     # 若读取出错，也要清理资源
finally:
    f.close()           # 总会尝试关闭已打开的文件
```

使用 `with` 就是让支持上下文管理协议的对象来承担这类退出约定，而不是要求每个调用者手写 `try/finally`。

### 逐字分析四个部分

```python
with open("notes.txt", "r", encoding="utf-8") as file:
    content = file.read()
```

- `open(...)` **首先求值**，它返回文件对象；如果打开阶段本身失败，`with` 还没有成功进入。
- `with` 调用该对象提供的上下文进入机制（通常涉及 `__enter__`）。
- `as file` 将进入方法返回的值绑定给变量 `file`，这个值**不一定总是上下文管理对象本身**。
- 代码块执行完，调用约定的退出机制（通常涉及 `__exit__`），即使代码块中发生异常也一样会调用；**能否压制异常由退出方法的行为决定**。

重要：`with` 不能保证“程序断电或强制杀死后照常执行退出操作”，也不意味着数据库、锁、文件在所有情况下都自动处理为“业务成功”。学 `contextlib.contextmanager` 前，先把普通 `with open`、`try/finally` 对应关系看清楚。

---

你最早见到 `with`，很可能是在读文件时。

```python
with open("data.txt", encoding="utf-8") as file:
    text = file.read()
```

你知道这样写会在结束后关闭文件。

但继续学习时，很容易产生新的疑问。

`with` 是不是 `open` 的专用语法？

`as file` 得到的，是否一定是括号前面那个对象？

如果里面执行 `return`，后面的清理还会执行吗？

为什么有些对象能放进 `with`，有些只有 `close()` 却不行？

我能不能自己写一种“进入时准备，离开时恢复”的工具？

如果一次需要打开数量不固定的文件，又该怎么办？

这篇围绕这些问题展开。

`open` 文档主要讲文件本身的读写、编码、位置与保存；本篇把 `with` 单独当作语言机制，讲它如何协调进入、使用、退出和异常。

文件只是最熟悉的起点，后面还会遇到临时目录、锁、输出重定向、资源集合与异步清理。

主要示例适用于 Python 3.11 及以上。

Python 3.12 的 `suppress` 异常组支持等变化会在具体位置注明。

命令里的 `python` 指你当前选用的解释器；如果 WSL 只有 `python3`，就用 `python3 demo.py`，虚拟环境中则使用该环境的解释器。

完整实践在临时目录里创建文件，普通短例子如果使用 `data.txt`，请先在你自己的练习目录准备它。

## 第一章：先从最熟悉的文件说起

### 1.1 `open` 做什么，`with` 又做什么？

```python
file = open("data.txt", encoding="utf-8")
```

这一步请求打开文件，并返回文件对象。

你可以通过它读取数据，最后再调用：

```python
file.close()
```

`with` 不负责决定文件的编码，也不负责替你选择读模式或写模式。

它负责使用对象提供的上下文管理协议，安排进入与退出过程。

对普通文件对象来说，退出过程会关闭文件。

所以这两个部分可以先这样区分：

```text
open(...)：建立一次文件访问
with：围绕一个代码块安排成对的进入与退出
```

### 1.2 不用 `with`，为什么最后一行 `close` 还不够？

```python
file = open("data.txt", encoding="utf-8")
text = file.read()
process(text)
file.close()
```

如果 `process()` 抛出异常，控制流可能在到达 `close()` 之前离开。

你写了关闭，不等于所有路径都会执行到关闭。

问题不是语句是否存在，而是控制流是否经过它。

### 1.3 用 `try` 和 `finally` 可以处理吗？

可以。

```python
file = open("data.txt", encoding="utf-8")
try:
    text = file.read()
    process(text)
finally:
    file.close()
```

只要文件已经成功打开，正常返回或普通异常离开 `try` 时，都会执行 `finally`。

但相似的成对结构会反复出现。

打开与关闭。

加锁与释放锁。

临时修改状态与恢复状态。

`with` 把这些对象共同遵守的结构组织成统一语法。

### 1.4 所以 `with` 只是为了少写两行吗？

减少重复是一个好处，但更重要的是把清理政策交给资源对应的对象。

调用者不必每次重新知道如何关闭、恢复或提交。

只要知道这个对象支持哪种进入退出约定，就能围绕代码块使用。

当然，约定具体做什么仍然需要查清楚。

一个文件的退出是关闭，一个数据库连接的退出可能是事务处理，不能看到相同语法就以为行为完全相同。

## 第二章：逐字理解 `with expression as target`

### 2.1 基本形式

```text
with 上下文表达式 as 目标变量:
    代码块
```

先计算上下文表达式，得到上下文管理器。

再执行进入协议。

把进入协议的返回值绑定给 `as` 后面的目标。

执行缩进代码块。

离开时执行退出协议。

后面会把这几步对应到 `__enter__` 与 `__exit__`。

### 2.2 `as` 后面的变量必须叫 `f` 吗？

不需要。

```python
with open("data.txt", encoding="utf-8") as input_file:
    text = input_file.read()
```

名称应该帮助你理解对象用途。

`f` 是常见简写，不是关键字。

`as` 本身是语法的一部分，后面则是普通绑定目标。

### 2.3 一定需要 `as` 吗？

不需要。

如果你只需要进入与退出产生的效果，不需要使用进入返回值，可以省略。

例如锁：

```python
from threading import Lock

lock = Lock()
with lock:
    print("在持有锁的范围内执行")
```

这里使用的是上下文管理效果，不需要给进入返回值另取名字。

### 2.4 `with` 会创建新的局部作用域吗？

普通 `with` 代码块不会像函数定义那样创建一个新的局部作用域。

```python
with open("data.txt", encoding="utf-8") as file:
    text = file.read()

print(text)
print(file.closed)
```

在相应作用域里，变量通常仍然存在。

文件关闭了，不代表变量名被自动删除。

资源状态与名称作用域，是两件事。

### 2.5 离开以后，文件对象还能看属性吗？

通常可以看 `closed`、`name` 等状态属性。

但再执行需要打开资源的操作，比如读取，通常会报错。

```python
print(file.closed)
# 预期 ValueError：文件已经关闭。
file.read()
```

对象还存在，不等于它关联的外部资源仍可使用。

这种区分会贯穿本篇。

## 第三章：自己写一个最小上下文管理器

### 3.1 两个特殊方法

普通同步上下文管理器核心方法是：

```text
__enter__(self)
__exit__(self, exc_type, exc_value, traceback)
```

双下划线方法把普通对象接入 Python 语言协议。

你已经学过魔法方法，可以把它理解为同一设计思路的另一个例子。

`with` 在指定时机调用这些协议方法，而不是要求每个调用者手动调用它们。

### 3.2 先只记录执行顺序

```python
class TracedContext:
    def __enter__(self):
        print("进入")
        return "进入返回的内容"

    def __exit__(self, exc_type, exc_value, traceback):
        print("退出")
        return False


with TracedContext() as value:
    print("代码块：", value)

print("后续代码")
```

输出：

```text
进入
代码块： 进入返回的内容
退出
后续代码
```

### 3.3 `value` 是管理器对象本身吗？

在这个例子里不是。

管理器是 `TracedContext()` 创建的实例。

`value` 是 `__enter__()` 返回的字符串。

这是理解 `as` 最关键的一点：绑定的是进入返回值，不保证是管理器自身。

文件对象的 `__enter__()` 常常返回自身，所以初学时很容易把两个角色混为一谈。

### 3.4 如果 `__enter__` 不写 `return` 呢？

就像普通函数一样，默认返回 `None`。

```python
class NoValueContext:
    def __enter__(self):
        print("准备完成")

    def __exit__(self, exc_type, exc_value, traceback):
        print("已经收尾")


with NoValueContext() as value:
    print(value)
```

输出中的业务值是 `None`。

这不代表 `with` 没有工作，而是进入方法没有提供其他返回值。

### 3.5 `__exit__` 的返回值交给 `as` 吗？

不是。

它的返回值主要用于决定是否抑制正在传播的异常。

正常退出时，不能通过 `__exit__` 返回值给 `as` 重新赋值。

如果需要从代码块得到业务结果，应通过普通变量、对象状态或函数返回值设计，不要把退出返回值误当成第二个 `as`。

## 第四章：异常发生以后，退出方法会收到什么？

### 4.1 正常退出时三个参数通常都是 `None`

```python
class InspectExit:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        print("异常类型：", exc_type)
        print("异常对象：", exc_value)
        print("有追踪对象：", traceback is not None)
        return False


with InspectExit():
    print("正常执行")
```

后面的异常类型与异常对象是 `None`，追踪标记是假。

### 4.2 发生异常时呢？

```python
try:
    with InspectExit():
        int("bad")
except ValueError:
    print("外层接到了 ValueError")
```

退出方法会获得异常类型、异常实例和追踪对象。

因为返回假，异常继续向外传播，最终被外层捕获。

退出方法不需要为了继续传播原异常，再主动写一遍 `raise exc_value`。

按协议返回假，正常结束退出方法即可。

### 4.3 为什么建议明确写 `return False`？

不写返回值，得到 `None`，同样是假值，也不会抑制异常。

明确写假，有时更能让初学者看出政策。

在简单清理代码中省略返回也很常见。

重点是不要无意返回真值，误把尚未处理的异常吞掉。

### 4.4 返回真会怎样？

```python
class IgnoreValueError:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        return exc_type is not None and issubclass(exc_type, ValueError)


with IgnoreValueError():
    int("bad")
    print("这行不会执行")

print("异常被明确抑制后，继续到这里")
```

异常发生的位置之后，代码块剩余语句不会恢复执行。

抑制的意思是退出以后不再向外传播，不是把失败语句变成功，也不是跳过那一行后回到代码块里面继续。

### 4.5 可以返回一个非布尔真值吗？

协议会按真值判断。

但为了让政策清楚，自己写退出方法时通常直接返回布尔值或 `None`。

不要随手返回一份非空结果列表，结果意外被当成“异常已处理”。

业务结果与异常抑制信号应分开。

### 4.6 什么时候不应该抑制？

如果你只是记录错误或释放资源，通常应让原异常继续传播。

只有当你真的定义了“这种失败可以忽略，后续状态仍然合理”的政策，才考虑抑制。

例如清理一个可选临时文件，文件本来不存在可以接受。

但解析配置失败以后返回空配置继续运行，不一定是合理恢复，不能只为了不报错就抑制所有异常。

## 第五章：`return`、`break`、`continue` 与清理顺序

### 5.1 函数在代码块里返回，也会退出吗？

```python
class AnnounceExit:
    def __enter__(self):
        print("进入")
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        print("退出，异常类型：", exc_type)
        return False


def calculate():
    with AnnounceExit():
        return 42


print(calculate())
```

输出：

```text
进入
退出，异常类型： None
42
```

`return` 不会跳过正常安排的退出过程。

它也不是普通异常，所以退出参数里异常类型为 `None`。

### 5.2 循环里的 `break` 呢？

```python
for number in range(3):
    with AnnounceExit():
        print(number)
        break
```

先退出上下文，再离开循环。

`continue` 也是先完成本次上下文退出，再进入下一次循环。

因此，`with` 的边界由控制流离开代码块决定，不是只在执行到最后一行时才生效。

### 5.3 `__exit__` 返回真，能阻止 `return` 吗？

不能把这个返回值理解为通用控制流取消按钮。

它用于异常抑制，不是拦截所有 `return`、`break`、`continue`。

当然，如果退出方法自己抛出新异常，会影响原本的返回路径。

这又属于清理失败的问题，后面会单独说明。

### 5.4 如果是 `sys.exit` 呢？

`sys.exit()` 通过抛出 `SystemExit` 表达退出请求。

正常栈展开会经过相应的退出过程。

但不能把这扩大成“任何方式结束进程都保证清理”。

例如强制终止、进程崩溃、断电或 `os._exit()`，不应期待所有 Python 清理代码都会正常运行。

上下文管理提供的是语言执行过程中的结构化收尾，不是对一切外部终止方式的绝对保证。

### 5.5 `KeyboardInterrupt` 会不会经过退出？

在正常异常传播的情况下，会。

退出方法可能收到这类异常。

但不要为图方便抑制所有 `BaseException`，否则用户想中断、程序想退出等控制信号也可能被你吞掉。

通常资源清理可以放在 `finally`，异常政策则应尽量具体。

## 第六章：进入阶段失败了，谁来清理？

### 6.1 `with` 不是还没开始就已经承诺一切

先看这段：

```python
with open("missing-file.txt", encoding="utf-8") as file:
    text = file.read()
```

如果 `open()` 本身失败，连文件对象都没有成功取得。

不会有一个成功进入的文件上下文等待退出。

这与“代码块里读取时失败，随后关闭已经打开的文件”不同。

把失败发生的阶段区分清楚，才能知道谁拥有清理责任。

### 6.2 如果 `__enter__` 执行到一半失败？

一个自定义管理器可能先取得资源 A，再取得资源 B。

如果 B 获取失败，而进入方法没有成功返回，`with` 不会为这次失败的进入自动调用该管理器的 `__exit__`。

因此，进入方法需要自己处理已经获得的一部分资源。

```python
class PartialEntry:
    def __enter__(self):
        print("取得第一项资源")
        try:
            raise RuntimeError("第二项资源准备失败")
        except BaseException:
            print("归还已经取得的第一项资源")
            raise

    def __exit__(self, exc_type, exc_value, traceback):
        print("这个退出不会因失败的进入而自动执行")


try:
    with PartialEntry():
        print("不会进入代码块")
except RuntimeError:
    print("调用者看见进入失败")
```

输出顺序说明清理由进入方法自己的失败路径完成。

这里捕获 BaseException 的目的，是完成清理后立刻重新抛出，没有把退出或中断信号当作成功。

日常业务错误处理不应因此无条件抑制所有 BaseException。

### 6.3 `as` 赋值失败，算不算已经进入？

如果 `__enter__()` 已经成功返回，但把返回值绑定给目标时失败，退出协议仍然会执行。

例如目标需要解包两个元素，进入返回值却只有一个。

```python
class OneValue:
    def __enter__(self):
        print("已经进入")
        return [1]

    def __exit__(self, exc_type, exc_value, traceback):
        print("执行退出：", exc_type.__name__ if exc_type else None)
        return False


try:
    with OneValue() as (first, second):
        print("不会运行")
except ValueError:
    print("绑定目标时解包失败")
```

这个例子能帮助你精确区分：进入方法抛错，与进入成功后绑定目标抛错，不是同一阶段。

这项顺序也可以对照 [官方 with 语句执行规则](https://docs.python.org/3.11/reference/compound_stmts.html#the-with-statement)。

### 6.4 为什么资源获取通常放进 `__enter__`？

这样，创建管理器对象与真正取得资源可以分开。

你可以先保存一个配置对象，直到进入代码块才实际打开文件或建立连接。

但这不是所有管理器必须遵守的唯一组织方式。

`open()` 返回时文件已经打开，文件对象随后再进入 `with`。

因此，判断资源何时获得，必须看具体 API，而不是只看它后来能不能用 `with`。

### 6.5 创建了管理器却从未进入，会有问题吗？

如果创建过程已经取得资源，可能需要另外关闭。

如果只是保存配置、资源等到进入才取得，通常没有同样的问题。

比如 `manager = open(...)` 与 `manager = 某个延迟获取的自定义对象`，生命周期不一样。

这也是建议把资源创建与 `with` 紧密写在一起的原因之一：减少“取得以后却忘记进入或关闭”的空档。

## 第七章：多个上下文管理器，为什么按照相反顺序退出？

### 7.1 最常见的文件复制写法

```python
with open("input.txt", encoding="utf-8") as source, open("output.txt", "w", encoding="utf-8") as target:
    target.write(source.read())
```

先进入 source，再进入 target。

离开时先退出 target，再退出 source。

这与嵌套结构相对应：

```python
with open("input.txt", encoding="utf-8") as source:
    with open("output.txt", "w", encoding="utf-8") as target:
        target.write(source.read())
```

### 7.2 为什么后进入先退出？

后面的资源可能依赖前面的资源。

先释放依赖者，再释放被依赖者，通常更符合成对管理的结构。

比如先建立工作目录，再在里面打开文件；离开时应先关闭文件，再删除临时目录。

这就是常说的后进先出。

但实际依赖关系仍然由你排列进入顺序，不是 `with` 自动分析所有对象之间的关系。

### 7.3 用事件列表观察顺序

```python
class NamedContext:
    def __init__(self, name, events):
        self.name = name
        self.events = events

    def __enter__(self):
        self.events.append(f"进入 {self.name}")
        return self.name

    def __exit__(self, exc_type, exc_value, traceback):
        self.events.append(f"退出 {self.name}")
        return False


events = []
with NamedContext("A", events) as first, NamedContext("B", events) as second:
    events.append(f"使用 {first} 和 {second}")
print(events)
```

输出：

```text
['进入 A', '进入 B', '使用 A 和 B', '退出 B', '退出 A']
```

### 7.4 第二个进入失败，第一个会退出吗？

会，前提是第一个已经成功进入。

这等价于在外层代码块里尝试进入内层，内层失败时外层正常获得异常并退出。

失败的第二个自身，仍然需要处理自己进入到一半取得的资源。

这两项责任可以同时成立：已成功进入的外层由协议收尾，尚未成功进入的内层负责自己的部分失败。

### 7.5 可以分行写得更清楚吗？

Python 3.11 基线支持带括号的多项形式：

```python
with (
    open("input.txt", encoding="utf-8") as source,
    open("output.txt", "w", encoding="utf-8") as target,
):
    target.write(source.read())
```

这不改变进入与退出顺序，只改善长参数情况下的排版。

如果代码块表达不同职责，保留嵌套也可能更容易阅读。

### 7.6 两个路径相同，会不会因为用了 `with` 就自动保护原文件？

不会。

如果先以读模式打开，再以写模式打开同一个文件，写模式可能截断原内容。

`with` 只安排退出，不替你识别危险的文件处理策略。

安全保存需要单独设计临时文件、替换与失败处理，后面会给一个范围明确的实践。

## 第八章：清理阶段自己失败，原来的异常怎么办？

### 8.1 清理代码也是普通代码

```python
class BrokenCleanup:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        raise RuntimeError("清理失败")
```

如果代码块本来正常完成，退出抛错会使整个 `with` 失败。

如果代码块已经有异常，新异常会影响最终向外传播的异常，同时通常能通过异常上下文看到原来的错误关系。

不能因为它叫清理，就以为它永远不会失败。

### 8.2 `return` 会被退出异常影响吗？

会。

```python

def return_with_broken_cleanup():
    with BrokenCleanup():
        return 100
```

虽然代码已经准备返回一百，但退出阶段抛出的 RuntimeError 会阻止这次正常返回。

所以“`finally` 或退出一定执行”与“执行以后一定成功返回原结果”不是一回事。

### 8.3 应该把清理异常全忽略吗？

不能一概而论。

关闭失败可能意味着数据没有完整写出，忽略会误报成功。

删除可选临时文件失败，也可能只是需要记录并稍后清理。

你要根据资源与业务要求决定哪个错误更应暴露，是否保留两个错误，以及如何通知调用者。

不要写一个通用管理器，把任何清理失败都默默吞掉，却对外承诺可靠保存。

### 8.4 `__exit__` 里再调用日志，也可能影响结果吗？

如果你自己提供的记录函数抛错，就可能影响退出流程。

普通 logging 在某些内部输出错误上有自己的处理，但不应假设任何自定义 sink 都永远不失败。

资源管理器应该让必要清理尽量执行，而把可选诊断与核心释放职责安排清楚。

复杂清理可以使用 `ExitStack`，不过它也不能把不正确的退出函数自动变正确。

## 第九章：`contextlib.contextmanager`——不用写类，也能定义上下文

### 9.1 先看重复结构

很多管理器只需要三段：

准备。

交给代码块使用。

离开后清理。

`contextlib` 提供 `@contextmanager`，让一个只产生一次值的生成器函数表达这三段。

### 9.2 最小示例

```python
from contextlib import contextmanager


@contextmanager
def labeled_context(label):
    print(f"准备 {label}")
    try:
        yield f"资源:{label}"
    finally:
        print(f"清理 {label}")


with labeled_context("demo") as value:
    print("正在使用：", value)
```

输出：

```text
准备 demo
正在使用： 资源:demo
清理 demo
```

### 9.3 `yield` 在这里交给谁？

不是让你写 `for` 循环消费一串业务数据。

装饰器把生成器接入上下文协议。

进入时运行到 `yield`。

产生的值交给 `as`。

代码块结束后，生成器在对应位置继续执行，以完成退出逻辑。

因此，`yield` 是控制权交接点。

### 9.4 为什么要求恰好产生一次？

零次没有提供正常进入值与交接点。

多次则无法符合“一次进入，一次代码块，一次退出”的协议。

如果函数根本没有走到 yield，可能报告生成器没有产生值。

如果退出时又产生第二个值，也会报告生成器没有按预期结束。

这与普通生成器可以产生任意多项不同。

`@contextmanager` 使用的是生成器能力的一种受约束协议。

### 9.5 为什么清理必须放在 `finally`？

如果你写：

```python
from contextlib import contextmanager


@contextmanager
def fragile_context():
    print("准备")
    yield "资源"
    print("只在正常继续时清理")
```

代码块抛异常时，异常会在生成器暂停的 yield 位置重新抛入。

于是 yield 后面的普通语句可能不会执行。

用 `try/finally` 包围 yield，才让相应退出路径完成清理。

这与普通函数里最后一行 close 可能被异常跳过的根源相同。

### 9.6 生成器里捕获异常但不再抛，会怎样？

会让上下文管理器认为异常已经处理，可能抑制异常。

```python
from contextlib import contextmanager


@contextmanager
def logged_but_suppressed(events):
    try:
        yield
    except ValueError:
        events.append("看见 ValueError")
```

如果你的目标只是记录而不是恢复，应在记录后重新抛出：

```python
@contextmanager
def logged_and_propagated(events):
    try:
        yield
    except ValueError:
        events.append("看见 ValueError")
        raise
```

这条区别在 [官方 contextmanager 说明](https://docs.python.org/3.12/library/contextlib.html#contextlib.contextmanager) 中特别重要。

### 9.7 为什么没有使用 `return False`？

类式 `__exit__` 通过返回值表达抑制政策。

生成器式管理器通过 yield 处异常是否继续传播等行为，与装饰器适配层共同完成协议。

不要把两套表面写法机械混在一起。

在生成器异常处理中写 `return False`，并不等于直接编写了一个 `__exit__`。

通常要继续传播，就直接 `raise`。

### 9.8 调用函数时，准备代码就开始了吗？

对这个生成器式工厂，通常创建的是管理器对象。

生成器函数体真正开始运行是在进入阶段。

```python
manager = labeled_context("later")
print("还没有进入")
with manager:
    print("现在进入了")
```

先输出“还没有进入”，再出现准备消息。

这与 `open()` 调用时已经打开文件的行为不同。

同样支持 `with`，资源获取时机也可以不同。

## 第十章：单次使用、可重复使用、可重入，分别是什么意思？

### 10.1 一次性管理器

生成器式上下文管理器的单个实例通常只能使用一次。

```python
manager = labeled_context("once")
with manager:
    pass

# 反面示例：不要重复进入同一个生成器式管理器实例。
# with manager:
#     pass
```

想使用第二次，应再次调用工厂：

```python
with labeled_context("first"):
    pass
with labeled_context("second"):
    pass
```

工厂可以反复调用，不等于每次返回的那个实例可以反复进入。

### 10.2 可重复使用但不可重入

有些管理器可以先进入退出，再进入退出。

但不能在同一次尚未退出时，再嵌套进入自身。

普通 `threading.Lock` 是很直观的例子：可以多次先后获取释放，但同一线程持有时再次无超时获取，可能阻塞自己。

不要为了验证这个边界，写一段会永久挂起的练习。

理解模型以后，可以使用非阻塞获取观察是否可用。

### 10.3 可重入

可重入意味着同一个管理器在尚未退出的情况下，再次进入也有定义良好的行为。

`threading.RLock` 可以在同一线程里嵌套获取，并需要对应次数释放。

`redirect_stdout` 和 `suppress` 也有各自可重入特性，但它们的共享状态与并发适用性还要另外判断。

“可重入”不等于“适合任意多线程并发使用”。

### 10.4 为什么文件对象通常不能关闭以后再进入？

文件关闭后，底层访问已经结束。

同一个对象不是自动带有“重新 open 原路径”的承诺。

需要新的打开操作，就再次调用 `open()`。

这与工厂和实例的区分相似。

### 10.5 自己写管理器要不要记录状态？

如果它不允许重复或嵌套进入，最好明确拒绝，而不是产生难理解的资源泄漏。

如果允许重入，就需要保存足够的状态层级，不能只用一个字段覆盖旧状态。

例如临时修改某个属性，嵌套两次时要分别恢复到上一次值，而不是最终都恢复成最后一次保存值。

后面的状态恢复实践会把这点具体展开。

## 第十一章：`closing`——对象有 `close`，却不能直接放进 `with` 怎么办？

### 11.1 有关闭方法，不代表实现了上下文协议

某些旧接口返回一个对象。

它提供 `read()` 和 `close()`，却没有 `__enter__`、`__exit__`。

直接写 `with object:` 会失败。

我们可以用 `contextlib.closing` 包装，使离开时调用 `.close()`。

### 11.2 一个不依赖外部网络的例子

```python
from contextlib import closing


class LegacyReader:
    def __init__(self, text):
        self.text = text
        self.closed = False

    def read(self):
        if self.closed:
            raise ValueError("读取器已关闭")
        return self.text

    def close(self):
        self.closed = True


reader = LegacyReader("hello")
with closing(reader) as current:
    print(current.read())
print(reader.closed)
```

输出：

```text
hello
True
```

### 11.3 `closing` 会自动猜测释放方法吗？

不会。

它按约定调用 `close()`，不是在 `shutdown`、`disconnect`、`release` 等方法里自动挑一个。

如果资源使用其他清理接口，可以自己写 `contextmanager`，或使用 ExitStack 的回调注册。

### 11.4 文件对象已经支持 `with`，还要套 `closing` 吗？

通常没有必要。

直接使用它原本的上下文协议更清楚，也保留具体对象定义的退出语义。

尤其对于数据库对象等，原协议可能不只是调用 close。

不要把 `closing` 当作“任何管理器外面都要再套一层”的固定模板。

### 11.5 包装前对象已经存在，谁负责失败前的部分？

`closing(resource)` 接收的是已经取得的对象。

取得对象的表达式如果自己失败，包装器没有得到资源，也无法关闭。

这再次对应获取阶段与使用阶段的责任边界。

## 第十二章：`nullcontext`——有时需要管理，有时只是借用，怎样少写重复？

### 12.1 一个函数接受路径，也接受已打开文件

调用者传路径时，函数自己打开，自己关闭。

调用者传现成文件对象时，函数只是借用，不应该顺手关闭调用者还要继续使用的对象。

这就是资源所有权问题。

我们可以为借用场景使用不做特殊进入退出的 `nullcontext`。

### 12.2 完整示例

```python
from contextlib import nullcontext
from os import PathLike


def read_first_line(source):
    if isinstance(source, (str, bytes, PathLike)):
        manager = open(source, encoding="utf-8")
    else:
        manager = nullcontext(source)
    with manager as stream:
        return stream.readline().rstrip("\r\n")
```

路径分支建立真实文件上下文。

借用分支把原对象作为进入返回值，但退出时不替调用者关闭。

### 12.3 怎样验证借用对象仍然打开？

```python
import io

stream = io.StringIO("第一行\n第二行\n")
print(read_first_line(stream))
print(stream.closed)
print(stream.readline().rstrip("\r\n"))
stream.close()
```

输出：

```text
第一行
False
第二行
```

读取位置仍然推进了，借用并不等于完全没有状态变化。

本例只承诺不关闭，不承诺恢复读取位置。

这种说明非常重要：资源归属与对象状态是两个维度。

### 12.4 `nullcontext()` 没有参数，`as` 得到什么？

默认是 `None`。

你也可以传一个对象，让进入时返回它。

```python
from contextlib import nullcontext

with nullcontext(42) as value:
    print(value)
```

输出四十二。

### 12.5 可选锁也可以这样处理吗？

可以在规则明确时使用。

```python
from contextlib import nullcontext


def update_values(values, new_value, lock=None):
    manager = nullcontext() if lock is None else lock
    with manager:
        values.append(new_value)
```

如果调用者没传锁，本函数就没有同步保证。

`nullcontext` 让代码块形式统一，不会凭空提供互斥。

### 12.6 用 `hasattr(source, "read")` 判断就万能了吗？

不是。

对象可能提供不同读取协议，文本流和二进制流也有不同返回类型。

本例明确期待 `readline()` 返回文本。

真实接口应该说明接受哪些输入，而不是靠几个属性检查声称兼容所有文件类对象。

## 第十三章：`suppress`——只忽略你真的允许忽略的异常

### 13.1 一个合理的清理需求

你想删除一个可选练习文件。

文件存在就删，不存在也视为已经达到目标。

其他问题，例如权限不足，仍然需要暴露。

```python
from contextlib import suppress
from pathlib import Path

with suppress(FileNotFoundError):
    Path("optional.tmp").unlink()
```

这相当于一个范围很窄的异常处理政策。

### 13.2 它会继续执行代码块剩下部分吗？

不会。

```python
from contextlib import suppress

events = []
with suppress(ValueError):
    events.append("开始")
    int("bad")
    events.append("异常以后仍在块内")
events.append("代码块之后")
print(events)
```

输出：

```text
['开始', '代码块之后']
```

这与类式管理器返回真时的行为一致。

### 13.3 为什么不写 `suppress(Exception)`？

范围太大时，很容易把你没有预料到的程序错误一起隐藏。

例如本来只想忽略文件不存在，却把路径类型错误、解析错误或其他逻辑问题也吞掉。

允许忽略哪类失败，就写哪类异常，并把代码范围缩小到对应动作。

一整个函数都放进宽泛 suppress，通常很难知道最终有没有真正完成任务。

### 13.4 文件检查后再删，是否更直观？

```python
path = Path("optional.tmp")
if path.exists():
    path.unlink()
```

这在简单脚本里可以理解，但检查与删除之间文件仍可能变化。

如果目标政策就是“删除时不存在可以接受”，直接捕获对应删除异常，能更贴近实际动作。

这不是要求所有程序都避免预检查，而是不要把预检查当作后续操作绝不会失败的保证。

### 13.5 Python 3.12 的异常组支持是什么？

Python 3.11 引入异常组。

Python 3.12 起，`suppress` 可以从异常组中移除匹配的异常，剩余部分继续以相应异常组传播。

因此，它不是遇到异常组就无条件忽略整组，也不是递归乱删所有错误。

如果要兼容 3.11，不应假设这项 3.12 行为已经存在。

[官方 suppress 版本说明](https://docs.python.org/3.12/library/contextlib.html#contextlib.suppress) 给出了这一差异。

### 13.6 一个明确标版本的例子

```python
# Python 3.12+
from contextlib import suppress

try:
    with suppress(ValueError):
        raise ExceptionGroup("批量失败", [ValueError("可忽略"), TypeError("需要处理")])
except ExceptionGroup as group:
    print([type(error).__name__ for error in group.exceptions])
```

输出保留下来的 TypeError。

这个例子只是展示筛选结果；复杂异常组还涉及嵌套关系与 `except*`，不需要为使用普通 suppress 先掌握所有细节。

## 第十四章：`redirect_stdout` 与 `redirect_stderr`——临时捕获输出

### 14.1 有个函数只会 `print`，我想得到它的文本

如果你能修改函数，通常直接让它返回文本或接受输出流更清楚。

但有时你需要临时适配现成函数。

可以把标准输出重定向到内存流。

```python
from contextlib import redirect_stdout
from io import StringIO


def show_report():
    print("学习报告")
    print("完成 3 项")


output = StringIO()
with redirect_stdout(output):
    show_report()
print(repr(output.getvalue()))
```

输出：

```text
'学习报告\n完成 3 项\n'
```

### 14.2 它修改的是这个函数的局部输出吗？

不是只给某个函数增加一个私有缓冲。

它临时替换当前进程中的 `sys.stdout` 引用。

因此，同一期间其他使用这个全局流的代码也可能受到影响。

这就是它通常不适合作为通用库或并发服务内部随意使用的原因。

### 14.3 会捕获子进程输出吗？

不会仅因为 Python 的 `sys.stdout` 引用被替换，就自动改变已经运行的子进程输出通道。

子进程输出应通过 `subprocess` 的管道与相关参数处理。

Python 层对象重定向与操作系统文件描述符重定向，也不是同一个机制。

### 14.4 能不能捕获 logging？

取决于处理器使用哪个流，以及什么时候保存了流对象。

一个早已建立的 StreamHandler 通常保存了当时的流引用。

后来改 `sys.stderr` 或 `sys.stdout`，不等于处理器自动改成新对象。

如果你要验证日志，更清楚的方法是让处理器直接使用传入的 StringIO，而不是依靠全局重定向碰巧捕获。

### 14.5 `redirect_stderr` 怎么用？

语法类似：

```python
from contextlib import redirect_stderr
from io import StringIO
import sys

output = StringIO()
with redirect_stderr(output):
    print("一条诊断", file=sys.stderr)
print(output.getvalue(), end="")
```

它只处理相应全局流引用，不会吞掉异常。

异常如果继续向外传播，仍然需要调用者处理。

### 14.6 重定向嵌套时会恢复到哪里？

可重入设计会保存对应层的旧流。

内层退出恢复到外层流，外层退出再恢复到最初流。

但这种嵌套能力不意味着不同线程交错修改同一全局流就安全。

重入与并发隔离，仍然是不同问题。

## 第十五章：`chdir`——临时切换工作目录，为什么要特别谨慎？

### 15.1 一个临时目录中的相对路径任务

有些旧函数总用相对路径，短期内你想让它在指定目录执行。

Python 3.11 起，`contextlib.chdir` 可以进入时切换目录，离开时恢复。

```python
from contextlib import chdir
from pathlib import Path
from tempfile import TemporaryDirectory

original = Path.cwd()
with TemporaryDirectory() as directory:
    with chdir(directory):
        Path("example.txt").write_text("hello", encoding="utf-8")
        print(Path("example.txt").exists())
    print(Path.cwd() == original)
```

输出两个真。

### 15.2 为什么说这不是并发隔离工具？

当前工作目录是进程范围状态。

一个线程切换，其他线程的相对路径解释也可能受影响。

异步代码在里面 `await`，让别的任务运行，也会出现类似干扰。

官方说明不建议在不适合全局状态变化的线程或异步场景使用。

### 15.3 更常见的替代方案是什么？

把基准路径作为参数，使用明确路径构造。

```python

def write_example(directory):
    path = Path(directory) / "example.txt"
    path.write_text("hello", encoding="utf-8")
    return path
```

这不需要临时改变整个进程的目录。

上下文工具让全局状态变化成对恢复，但不会把全局状态变成局部状态。

### 15.4 在 `chdir` 里面创建一个生成器，再出去消费可以吗？

要看生成器什么时候真正执行相对路径操作。

如果它在离开以后才开始运行，当前目录已经恢复。

因此，创建生成器的地方，不一定是读取文件真正发生的地方。

惰性执行与资源边界交错时，应特别看执行时机。

这与文件读取生成器不能随意逃出文件上下文，是同一类判断问题。

## 第十六章：`ExitStack`——资源数量运行时才知道，怎么统一退出？

### 16.1 固定两个文件，可以直接写两个 `with`

但如果文件列表来自用户选择，可能是一个，也可能二十个。

你不可能提前写二十层固定嵌套。

`ExitStack` 提供一个动态退出栈，让你在运行时逐个进入上下文，并在结束时逆序退出。

### 16.2 打开一组文件

```python
from contextlib import ExitStack


def read_first_lines(paths):
    with ExitStack() as stack:
        files = [stack.enter_context(open(path, encoding="utf-8")) for path in paths]
        return [file.readline().rstrip("\r\n") for file in files]
```

每成功进入一个文件，就登记对应退出。

如果后面某个文件打开失败，前面已经成功登记的文件会由退出栈收尾。

### 16.3 `enter_context` 返回什么？

返回被进入管理器的 `__enter__()` 结果。

就像普通 `as` 一样，不保证与管理器对象相同。

所以可以直接把文件对象放进列表，或接收自定义进入返回值。

### 16.4 先建立所有对象，再统一进入，会不会一样？

不一定。

如果创建对象时就获取资源，而你先用列表推导建立十个对象，第六个创建失败，前五个可能还没登记到退出栈。

因此，获取与登记的顺序应尽量紧密。

下面这种写法把成功打开的文件立即交给栈，是为了减少未登记资源的窗口：

```python
with ExitStack() as stack:
    files = []
    for path in paths:
        file = stack.enter_context(open(path, encoding="utf-8"))
        files.append(file)
```

当然，如果 `__enter__` 自身在部分取得资源后失败，仍然遵守前面讲过的进入责任。

ExitStack 不会替一个失败进入的管理器自动调用尚未登记的退出。

### 16.5 退出栈会自动在垃圾回收时执行吗？

不要依靠这个假设。

应使用 `with ExitStack()`，或在明确生命周期中调用 `close()`。

栈对象被丢弃，不等于已经安排所有回调按你期待执行。

结构化清理最重要的就是有明确结束点，而不是寄希望于对象什么时候被回收。

## 第十七章：`ExitStack` 的三种登记方式，为什么不能混用含义？

### 17.1 `enter_context`：进入并登记退出

```python
resource = stack.enter_context(manager)
```

它会执行进入协议，再把相应退出登记进栈。

返回的是进入结果。

这是动态管理标准上下文对象时最常用的方法。

### 17.2 `callback`：登记一个普通清理函数

有些资源的释放是普通函数，不接受异常类型等三个参数。

```python
from contextlib import ExitStack

events = []
with ExitStack() as stack:
    stack.callback(events.append, "先登记的清理")
    stack.callback(events.append, "后登记的清理")
    events.append("执行工作")
print(events)
```

输出：

```text
['执行工作', '后登记的清理', '先登记的清理']
```

`callback` 接收函数和需要绑定的参数。

它登记，不会马上执行函数。

### 17.3 普通回调能抑制代码块异常吗？

不能通过普通回调返回真来抑制。

这类回调不接收完整退出异常参数，其返回值也不按 `__exit__` 的方式处理。

如果你需要根据异常决定抑制，应使用符合退出协议的函数或管理器。

不要因为两个方法都把东西放进栈，就以为参数和返回值语义一样。

### 17.4 `push`：登记一个退出协议回调，但不自动进入

`push()` 可以接受符合退出协议的对象或退出函数。

它登记相应退出，却不替你调用 `__enter__`。

这在资源已经取得，或者你只想保护进入过程中的后半段时有用。

但初学者更容易误用，所以日常先掌握 `enter_context` 和 `callback` 就能解决很多任务。

### 17.5 用事件说明 `push` 的异常参数

```python
from contextlib import ExitStack

events = []


def record_exit(exc_type, exc_value, traceback):
    events.append(None if exc_type is None else exc_type.__name__)
    return False


try:
    with ExitStack() as stack:
        stack.push(record_exit)
        raise ValueError("演示")
except ValueError:
    print("错误继续传播")
print(events)
```

可以看见回调收到 ValueError 类型，返回假以后异常继续传播。

### 17.6 内层抑制以后，外层回调看到什么？

退出栈按嵌套上下文的含义依次展开。

如果较晚登记的退出回调抑制了异常，较早登记的外层回调会根据更新后的异常状态继续执行。

如果某个退出回调抛出新异常，外层回调也会看到相应变化。

所以它不是简单“把最初异常复制给所有回调”的广播列表。

这项语义让动态栈能够模拟真正嵌套的管理器。

### 17.7 `close()` 是按有异常退出吗？

它按无异常参数执行已登记退出。

所以如果你自己手动调用 `close()`，不要期待它自动知道外面某个已经被你捕获的异常。

异常上下文应通过正确的 `with` 展开或明确协议传递。

普通关闭入口与异常退出入口有区别。

### 17.8 `pop_all()` 做什么？

它把当前登记的退出责任转移到一个新的 ExitStack 对象。

转移本身不执行清理。

原来的栈随后不再拥有这些回调，新栈需要在合适时机关闭。

可以把它理解为资源归属转移，而不是取消资源释放这件事本身。

### 17.9 为什么不用“成功就永远不清理”来解释？

因为资源仍然需要有明确拥有者。

如果你 `pop_all()` 后把返回的新栈丢掉，可能造成资源长期未关闭。

正确实践应把新栈保存并交给调用者，或放进另一个有明确生命周期的对象。

后面会写一套返回资源集合的完整例子。

## 第十八章：`async with`——退出过程也需要等待时怎么办？

### 18.1 同步 `with` 不会自动等待

如果资源的获取与释放需要异步 I/O，例如异步连接池租借与归还，就需要异步上下文协议。

核心方法是：

```text
__aenter__(self)
__aexit__(self, exc_type, exc_value, traceback)
```

通常定义为 `async def`，并在 `async with` 中等待相应过程。

### 18.2 一个不依赖网络的最小类

```python
import asyncio


class AsyncTrackedResource:
    def __init__(self, events):
        self.events = events

    async def __aenter__(self):
        self.events.append("开始获取")
        await asyncio.sleep(0)
        self.events.append("获取完成")
        return "可用资源"

    async def __aexit__(self, exc_type, exc_value, traceback):
        self.events.append("开始归还")
        await asyncio.sleep(0)
        self.events.append("归还完成")
        return False


async def use_async_resource():
    events = []
    async with AsyncTrackedResource(events) as resource:
        events.append(resource)
    return events


if __name__ == "__main__":
    print(asyncio.run(use_async_resource()))
```

输出顺序：

```text
['开始获取', '获取完成', '可用资源', '开始归还', '归还完成']
```

### 18.3 为什么不能直接放在普通顶层？

`async with` 需要在允许异步执行的函数环境里使用。

普通脚本可以通过 `asyncio.run()` 启动一个异步入口。

如果环境已经运行事件循环，例如某些交互环境，不应随手再嵌套 `asyncio.run()`；应使用该环境提供的等待方式。

### 18.4 `async with open(...)` 可以吗？

内置普通文件对象主要提供同步上下文协议。

不能因为你处在 `async def` 里，就把所有 `with` 机械改为 `async with`。

异步函数里可以使用同步 `with open(...)`，但同步文件 I/O 仍可能阻塞事件循环。

语法选择与 I/O 是否阻塞，需要分别判断。

### 18.5 取消时也会退出吗？

异步任务取消通常通过异常传播进入清理路径。

但异步清理本身还可能需要等待，进一步取消、关闭循环或强制结束进程，都可能影响它完成。

因此，不应承诺“用了 async with，任何取消下都绝对完整归还”。

重要资源需要库自身明确的取消与清理设计，必要时使用适当的超时或保护策略。

不要在入门示例里无条件包 `shield` 就宣称解决所有取消问题，因为后台清理任务的归属与等待仍然需要处理。

### 18.6 同步管理器与异步管理器可以互相自动转换吗？

不能仅凭名称相似就自动转换。

同步退出不会等待一个协程完成。

异步栈可以登记同步与异步清理，但你必须使用正确方法告诉它该怎样执行。

后面会讲 `AsyncExitStack`。

## 第十九章：`asynccontextmanager` 与 `aclosing`

### 19.1 异步生成器也能表达成对过程

```python
import asyncio
from contextlib import asynccontextmanager


@asynccontextmanager
async def async_labeled(events):
    events.append("异步准备")
    await asyncio.sleep(0)
    try:
        yield "资源"
    finally:
        await asyncio.sleep(0)
        events.append("异步清理")


async def run_async_labeled():
    events = []
    async with async_labeled(events) as resource:
        events.append(resource)
    return events


if __name__ == "__main__":
    print(asyncio.run(run_async_labeled()))
```

与同步版本一样，只产生一次值。

代码块异常会在交接位置进入生成器，清理一般用 `finally`。

### 19.2 `async def` 里写了 `yield`，是什么？

它成为异步生成器函数。

这与普通协程函数不同。

`@asynccontextmanager` 使用这种生成器提供进入与退出两段异步执行。

不要因为所有定义都写 `async def`，就把协程对象、异步迭代器和异步上下文管理器当成同一种东西。

### 19.3 提前停止异步迭代，生成器的清理何时执行？

如果你只是 `break`，不应把清理时机完全交给后续回收。

`aclosing` 可以在退出代码块时等待对象的 `aclose()`。

```python
import asyncio
from contextlib import aclosing


async def tracked_numbers(events):
    try:
        for number in range(3):
            yield number
    finally:
        await asyncio.sleep(0)
        events.append("生成器已关闭")


async def use_first_number():
    events = []
    async with aclosing(tracked_numbers(events)) as numbers:
        async for number in numbers:
            events.append(number)
            break
    events.append("离开管理范围")
    return events


if __name__ == "__main__":
    print(asyncio.run(use_first_number()))
```

输出：

```text
[0, '生成器已关闭', '离开管理范围']
```

### 19.4 为什么强调同一上下文中的确定清理？

异步生成器的最终清理可能依赖当前任务中的上下文变量或其他资源。

如果清理拖到很晚，外围资源可能已经退出。

把 `aclose` 放在明确范围里等待，能让顺序更清楚。

它不是让生成器自动完整消费到最后，而是明确关闭，执行相应终结逻辑。

### 19.5 `aclosing` 只能管理异步生成器吗？

它围绕对象的 `aclose()` 组织清理。

异步生成器是典型用途，但关键是对象提供符合要求的异步关闭方法。

如果对象使用其他释放协议，可以写自己的异步上下文管理器。

`closing` 与 `aclosing` 的区别是清理过程是否需要等待，不是一个名字更高级就可以替代所有另一个名字的用途。

## 第二十章：`AsyncExitStack`——动态混合同步与异步清理

### 20.1 为什么普通 ExitStack 不够？

普通 ExitStack 的退出过程是同步的。

你给它一个需要 await 的清理函数，它不会因为返回协程对象就自动等待。

`AsyncExitStack` 在异步退出时可以等待登记的异步清理，同时也能管理同步退出。

### 20.2 最小完整示例

```python
import asyncio
from contextlib import AsyncExitStack, asynccontextmanager, contextmanager


@contextmanager
def sync_part(events):
    events.append("进入同步部分")
    try:
        yield "sync"
    finally:
        events.append("退出同步部分")


@asynccontextmanager
async def async_part(events):
    events.append("进入异步部分")
    try:
        yield "async"
    finally:
        await asyncio.sleep(0)
        events.append("退出异步部分")


async def run_mixed_stack():
    events = []
    async with AsyncExitStack() as stack:
        first = stack.enter_context(sync_part(events))
        second = await stack.enter_async_context(async_part(events))
        events.append((first, second))
    return events


if __name__ == "__main__":
    print(asyncio.run(run_mixed_stack()))
```

退出仍然按登记的相反顺序。

异步部分先退出，然后同步部分退出。

### 20.3 为什么一个要 await，另一个不要？

`enter_context` 执行同步进入协议，直接返回结果。

`enter_async_context` 需要等待异步进入，所以调用处使用 `await`。

这不是随意的命名差异，而是执行协议不同。

### 20.4 普通清理回调与异步回调分别怎么登记？

同步普通函数可以用 `callback()`。

需要等待的普通异步清理函数可以用 `push_async_callback()`。

接收完整异常参数的异步退出函数可以用对应的 `push_async_exit()`。

与同步版本一样，普通回调的返回值不能用来随意抑制异常。

### 20.5 手动结束用 `close` 吗？

异步退出栈使用可等待的 `aclose()`。

日常更推荐 `async with AsyncExitStack()`，让生命周期写在语法结构里。

如果把栈转交给调用者，就要让调用者知道自己需要等待关闭，而不是仅把对象丢进某个变量就算完成。

## 第二十一章：`ContextDecorator`——同一段范围管理，还能用装饰器形式吗？

### 21.1 有时想管理整个函数

如果一个函数的全部主体都需要同样的进入退出效果，可以使用支持装饰器形式的上下文管理器。

`ContextDecorator` 提供这种组织能力。

```python
from contextlib import ContextDecorator


class AnnouncedCall(ContextDecorator):
    def __enter__(self):
        print("开始一次调用")
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        print("结束一次调用")
        return False


@AnnouncedCall()
def greet(name):
    return f"你好，{name}"


print(greet("小林"))
```

### 21.2 会把进入返回值自动传给函数吗？

不会像 `with ... as value` 那样给函数增加一个业务参数。

它围绕函数调用管理范围。

如果函数需要资源对象，显式 `with` 通常更容易把资源拿到并传给后续逻辑。

不要为了写成装饰器，就把本来清楚的资源变量藏起来。

### 21.3 被装饰函数会调用多次，管理器要满足什么？

管理逻辑必须适合每次调用进入退出。

如果单个管理器只能使用一次，装饰器形式就需要在每次调用时创建新实例。

`@contextmanager` 产生的工具支持相应装饰器用法，会为调用安排新生成器实例。

自己继承 ContextDecorator 时，仍然要考虑状态与重复使用，不能简单认为继承以后所有一次性对象自动可复用。

### 21.4 异步函数也可以用同步 ContextDecorator 吗？

不能假设它会自动等待整个协程执行。

同步包装器可能只围绕创建协程对象那一步。

异步范围需要相应的 `AsyncContextDecorator` 或支持异步装饰器形式的异步上下文工厂。

与 functools 中同步装饰器不能自动具备 await 语义一样，执行模型需要明确。

## 第二十二章：完整实践一——自己封装文件管理，但不要重复制造复杂的 `open`

### 22.1 为什么还要写这个练习？

真实程序读普通文件，直接 `with open(...)` 就很好。

这个实践不是推荐你把标准接口全部重新包装。

它的目标是把类式管理器的获取、进入返回值、退出清理、单次使用与失败路径放在同一个可运行例子里观察。

我们只支持 UTF-8 文本读取。

调用者在代码块里拿到真实文件对象。

管理器额外保存事件，方便验证。

### 22.2 完整程序

```python
from pathlib import Path
from tempfile import TemporaryDirectory


class ManagedTextInput:
    def __init__(self, path, events):
        self.path = Path(path)
        self.events = events
        self.stream = None
        self.used = False

    def __enter__(self):
        if self.used:
            raise RuntimeError("这个管理器实例只能使用一次")
        self.used = True
        self.events.append("尝试打开")
        self.stream = self.path.open(encoding="utf-8")
        self.events.append("打开成功")
        return self.stream

    def __exit__(self, exc_type, exc_value, traceback):
        try:
            if self.stream is not None:
                self.stream.close()
        finally:
            self.events.append(("已执行关闭", None if exc_type is None else exc_type.__name__))
        return False


def run_managed_text_demo():
    events = []
    with TemporaryDirectory() as directory:
        path = Path(directory) / "input.txt"
        path.write_text("第一行\n第二行\n", encoding="utf-8")
        manager = ManagedTextInput(path, events)
        with manager as stream:
            print(stream.readline().rstrip("\n"))
            print("返回的是文件对象：", stream is manager.stream)
        print("已经关闭：", manager.stream.closed)
        print(events)
        return events, manager.stream.closed


if __name__ == "__main__":
    run_managed_text_demo()
```

主要输出：

```text
第一行
返回的是文件对象： True
已经关闭： True
['尝试打开', '打开成功', ('已执行关闭', None)]
```

### 22.3 为什么 `used` 在打开前就设真？

这个例子选择单次尝试政策：不论打开是否成功，同一个管理器实例不再重复进入。

如果你希望失败后允许重试，可以另外设计状态机。

两种政策都可以，但不能让行为取决于偶然哪一行抛异常。

使用者需要知道自己应当重用实例，还是重新创建。

### 22.4 打开失败，会有关闭事件吗？

不会因为 with 自动调用这次未成功进入的 `__exit__`。

但本例只有一个 open 操作，失败时没有成功保存一个打开流需要关闭。

如果你在打开前还取得别的资源，就必须在进入失败路径处理那些资源。

所以这个例子的简单性来自资源模型很窄，不是 `with` 对所有部分失败都自动兜底。

### 22.5 代码块里故意抛错，会怎样？

```python
try:
    with manager_factory() as stream:
        raise ValueError("处理内容时失败")
except ValueError:
    print("调用者收到失败")
```

这里 `manager_factory` 代表你重新建立一个同类管理器的工厂，需要按自己的输入实现。

真实验证时可以直接创建新的 `ManagedTextInput(path, events)`。

关闭会执行，退出记录带 ValueError，异常继续传播。

不要把本节这一段示意当作已经定义了 `manager_factory` 的独立脚本。

### 22.6 事件里的“已执行关闭”是不是保证磁盘永远不会丢数据？

不是。

这里管理的是读取流，而且事件表示关闭路径被执行。

资源释放成功、数据持久化与程序崩溃恢复是不同层面。

取一个准确的事件名很重要，不能用一条清理事件宣称完成了自己没有实现的保证。

## 第二十三章：完整实践二——动态打开多份文件，第三份失败时前两份怎么办？

### 23.1 这次要验证失败，不只验证正常读取

有三条路径。

前两份文件存在，第三份不存在。

要求在尝试第三份失败后，前两份打开的文件全部关闭。

为了观察状态，我们把已经取得的流暂存在列表里。

函数本身仍然让 FileNotFoundError 向调用者传播。

### 23.2 完整程序

```python
from contextlib import ExitStack
from pathlib import Path
from tempfile import TemporaryDirectory


def open_and_read_first_lines(paths, observed):
    with ExitStack() as stack:
        streams = []
        for path in paths:
            stream = stack.enter_context(Path(path).open(encoding="utf-8"))
            streams.append(stream)
            observed.append(stream)
        return [stream.readline().rstrip("\r\n") for stream in streams]


def run_partial_open_demo():
    observed = []
    with TemporaryDirectory() as directory:
        root = Path(directory)
        first = root / "first.txt"
        second = root / "second.txt"
        missing = root / "missing.txt"
        first.write_text("A\n", encoding="utf-8")
        second.write_text("B\n", encoding="utf-8")
        try:
            open_and_read_first_lines([first, second, missing], observed)
        except FileNotFoundError:
            print("第三份文件未找到")
        print("已经打开过的流数量：", len(observed))
        print("它们都关闭了：", all(stream.closed for stream in observed))
        return len(observed), all(stream.closed for stream in observed)


if __name__ == "__main__":
    run_partial_open_demo()
```

输出：

```text
第三份文件未找到
已经打开过的流数量： 2
它们都关闭了： True
```

### 23.3 为什么不在异常处理里手工遍历关闭？

可以手写，但很容易把正常路径、读取失败路径与打开失败路径分别写一套。

退出栈在每次成功进入后就登记责任，后面无论哪个步骤失败，都按同一结构退出。

这正是它比简单“最后遍历列表”更有价值的地方。

### 23.4 文件很多时，同时打开合理吗？

不一定。

操作系统允许的打开文件数量有限，打开流也有资源成本。

如果每个文件可以独立处理，逐个 `with open`，读完立即关闭通常更合适。

ExitStack 解决动态成组生命周期，不是鼓励一次打开所有文件。

需求确实需要同时访问多份资源时，再使用这个结构。

### 23.5 空路径列表呢？

没有进入任何文件，返回空列表。

栈退出时也没有文件需要关闭。

这是一种自然且明确的空输入政策。

如果业务至少需要一份文件，应在入口显式拒绝空输入，不要让后面某处索引错误偶然承担验证职责。

## 第二十四章：完整实践三——资源需要返回给调用者，怎样转移关闭责任？

### 24.1 为什么直接从 `with` 返回流列表不行？

```python
from contextlib import ExitStack


def wrong_open_group(paths):
    with ExitStack() as stack:
        return [stack.enter_context(open(path, encoding="utf-8")) for path in paths]
```

函数返回之前会退出栈，列表里的流已经关闭。

你虽然拿到对象，却拿不到可继续读取的打开资源。

这个错误与从 `with open` 中返回文件对象相同。

### 24.2 给调用者一个明确的管理器

最直接的设计是把整个打开组也做成上下文工厂。

```python
from contextlib import ExitStack, contextmanager


@contextmanager
def opened_text_group(paths):
    with ExitStack() as stack:
        streams = [stack.enter_context(open(path, encoding="utf-8")) for path in paths]
        yield streams
```

调用者这样使用：

```python
with opened_text_group(paths) as streams:
    first_lines = [stream.readline() for stream in streams]
```

资源在调用者代码块期间保持打开，退出后统一关闭。

这是日常最清楚的接口之一。

### 24.3 如果确实需要普通函数返回已打开资源呢？

可以把流和拥有关闭责任的栈一起包装成一个对象。

成功获取所有资源后，使用 `pop_all()` 把责任转移给返回对象。

对象再提供明确的 close 与上下文协议。

### 24.4 完整资源集合类

```python
from contextlib import ExitStack


class OpenTextGroup:
    def __init__(self, streams, owner):
        self.streams = tuple(streams)
        self._owner = owner
        self.closed = False

    @classmethod
    def open(cls, paths):
        with ExitStack() as temporary_owner:
            streams = [
                temporary_owner.enter_context(open(path, encoding="utf-8"))
                for path in paths
            ]
            owner = temporary_owner.pop_all()
        return cls(streams, owner)

    def close(self):
        if not self.closed:
            self._owner.close()
            self.closed = True

    def __enter__(self):
        if self.closed:
            raise ValueError("资源组已经关闭")
        return self.streams

    def __exit__(self, exc_type, exc_value, traceback):
        self.close()
        return False
```

### 24.5 每一步的拥有者是谁？

获取过程中，`temporary_owner` 拥有已成功打开流的退出责任。

任何一次获取失败，它都会关闭前面资源。

全部成功以后，`pop_all()` 返回新的 owner，原临时栈不再负责这些回调。

返回对象保存 owner，成为新的明确拥有者。

调用者需要使用 `with group` 或主动 `group.close()`。

这不是让清理责任消失，而是把责任从局部临时栈转到可以跨函数返回的对象上。

### 24.6 为什么 close 里不直接全部 `stream.close()`？

保留栈可以维护原来登记的退出顺序与未来扩展的清理动作。

不过这个类的 close 调用按无异常关闭退出栈，不负责把用户代码块异常传给内部文件退出。

对普通文件关闭这已经符合本例用途，因为文件退出不靠业务异常决定提交或回滚。

如果内部管理的是事务型资源，需要把异常信息正确传给 owner 的退出协议，不能机械套用这个类。

这条边界说明为什么“资源集合类”不是适配所有管理器的万能实现。

### 24.7 返回对象构造失败怎么办？

这个简化类的构造只保存几个引用。

更复杂的真实设计如果在责任转移后还可能执行会失败的初始化，就要保护新 owner，确保失败时关闭。

原则是：任何时刻都应该有一个明确负责收尾的地方。

不要在两个拥有者之间留下“都以为对方负责”的空隙。

## 第二十五章：完整实践四——临时修改对象属性，结束后恢复原样

### 25.1 为什么不是修改全局工作目录？

这次我们管理的是一个普通设置对象上的属性。

临时把详细模式打开，执行一段操作，再恢复原值。

如果属性原来不存在，退出后也应该恢复成不存在。

这比“总是设回 False”更准确，因为原来可能就是 True，或者根本没有这个属性。

### 25.2 完整实现

```python
from contextlib import contextmanager
from types import SimpleNamespace


_MISSING = object()


@contextmanager
def temporary_attribute(target, name, value):
    if not isinstance(name, str) or not name:
        raise ValueError("属性名必须是非空字符串")
    previous = getattr(target, name, _MISSING)
    setattr(target, name, value)
    try:
        yield target
    finally:
        if previous is _MISSING:
            delattr(target, name)
        else:
            setattr(target, name, previous)


def run_attribute_demo():
    settings = SimpleNamespace(verbose=False)
    print(settings.verbose)
    with temporary_attribute(settings, "verbose", True):
        print(settings.verbose)
        with temporary_attribute(settings, "verbose", "内层"):
            print(settings.verbose)
        print(settings.verbose)
    print(settings.verbose)
    with temporary_attribute(settings, "temporary_flag", 123):
        print(settings.temporary_flag)
    print(hasattr(settings, "temporary_flag"))
    return settings


if __name__ == "__main__":
    run_attribute_demo()
```

输出：

```text
False
True
内层
True
False
123
False
```

### 25.3 为什么需要专门的 `_MISSING`？

如果用 `None` 表示不存在，那么原属性恰好等于 None 时就无法区分。

一个独立的 object 实例作为哨兵，只用身份判断，不会与普通业务值混淆。

这与缓存里区分“没有条目”和“结果是 None”的问题有相似之处。

### 25.4 为什么嵌套以后能逐层恢复？

每次调用工厂，都会建立独立的生成器状态。

外层保存 False。

内层进入时看到外层的 True，所以保存 True。

内层退出恢复 True，外层退出恢复 False。

这里不是一个共享字段被反复覆盖，而是每层各自保存进入前状态。

### 25.5 可以用于任意属性吗？

不要这样承诺。

属性可能是描述符，读取、赋值或删除可能执行任意逻辑，也可能失败。

本例针对普通、可读写删除的实例属性使用。

如果目标是 property、只读对象或有特殊副作用的属性，需要专门分析恢复规则。

此外，如果代码块自己删除了这个临时属性，退出时再次删除也可能失败。

调用协议应该要求使用者不要破坏管理器负责恢复的那部分状态，或者进一步设计允许变化的政策。

### 25.6 多线程共享同一个设置对象呢？

临时恢复不会自动提供隔离。

其他线程在这段时间仍然可能看见临时值，或者同时修改它。

如果这是并发任务各自的上下文，优先考虑显式参数或 contextvars 等适合的模型。

“最后恢复了”不等于“期间别人看不到”。

### 25.7 如果代码块抛异常呢？

`finally` 会尝试恢复，然后原异常继续传播。

如果恢复动作自己失败，又会出现清理异常的问题。

所以这个工具适合普通设置对象，不应被描述为对任意对象永远可靠的回滚机制。

理解适用范围，才能把小工具用得放心。

## 第二十六章：完整实践五——写完才替换目标文件，`with` 在里面负责哪一段？

### 26.1 先说清楚要避免什么

你想生成一份报告，替换原来的报告文件。

如果直接 `open(target, "w")`，旧文件可能在开始时就被截断。

计算或写入中途失败，原报告也已经不完整了。

一种常见方案是在同一目录先写临时文件。

全部写入并关闭成功后，再替换目标。

这能让“生成新内容”与“发布为目标文件”分开。

### 26.2 不要先许诺所有可靠性

下面的实践只处理普通本地文件的临时写入与替换流程。

它不保留目标文件全部权限、所有权或扩展属性。

它不协调多个并发写入者，最后替换者可能覆盖前一个结果。

它也不把普通关闭与替换等同于断电后的完整持久化协议。

这些限制直接来自操作系统与文件系统行为，不是多加一个 with 就能自动消除。

### 26.3 完整管理器

```python
import os
from contextlib import contextmanager, suppress
from pathlib import Path
from tempfile import NamedTemporaryFile, TemporaryDirectory


@contextmanager
def replacement_text_writer(target, *, encoding="utf-8"):
    target = Path(target)
    temporary_path = None
    try:
        with NamedTemporaryFile(
            mode="w",
            encoding=encoding,
            dir=target.parent,
            prefix=f".{target.name}.",
            suffix=".tmp",
            delete=False,
        ) as stream:
            temporary_path = Path(stream.name)
            yield stream
            stream.flush()
        os.replace(temporary_path, target)
        temporary_path = None
    finally:
        if temporary_path is not None:
            with suppress(FileNotFoundError):
                temporary_path.unlink()


def run_replacement_demo():
    with TemporaryDirectory() as directory:
        target = Path(directory) / "report.txt"
        target.write_text("旧报告", encoding="utf-8")

        try:
            with replacement_text_writer(target) as stream:
                stream.write("写到一半的新报告")
                raise ValueError("生成失败")
        except ValueError:
            print("失败后内容：", target.read_text(encoding="utf-8"))

        with replacement_text_writer(target) as stream:
            stream.write("完整的新报告")
        print("成功后内容：", target.read_text(encoding="utf-8"))
        print("临时文件剩余数量：", len(list(Path(directory).glob("*.tmp"))))
        return target.read_text(encoding="utf-8")


if __name__ == "__main__":
    run_replacement_demo()
```

输出：

```text
失败后内容： 旧报告
成功后内容： 完整的新报告
临时文件剩余数量： 0
```

### 26.4 为什么临时文件放在目标目录？

这样通常可以让临时文件与目标处于同一文件系统，符合常见原子替换操作的要求。

如果临时文件在另一个磁盘，替换可能无法按同样方式完成。

但目标目录需要已经存在，并允许创建文件。

这个函数没有悄悄创建任意父目录，因为目录建立政策应由调用者明确决定。

### 26.5 为什么先关闭，再调用 `os.replace`？

关闭可以完成 Python 流的收尾，也减少跨平台打开文件状态带来的限制。

在 Windows 上，打开文件与删除替换之间的限制尤其容易影响示例。

用完临时流后再替换，流程更清楚。

### 26.6 代码块失败时，为什么不会执行替换？

异常从 yield 位置进入生成器。

没有被这里捕获为成功，于是跳过 yield 后的正常提交路径，进入 finally 清理。

临时文件关闭由内部文件上下文处理，临时路径删除由外层 finally 处理。

两个资源层次各自有明确责任。

### 26.7 为什么替换成功后把 `temporary_path` 设为 `None`？

表示临时路径已不再由清理逻辑负责。

成功替换后，原临时名称通常已经不存在。

虽然 suppress 可以忽略不存在，明确置空仍然更能说明状态转换。

这类似资源归属转移：内容已经成为目标文件，不能再按临时资源处理。

### 26.8 删除临时文件也失败了怎么办？

本例只忽略“文件已不存在”。

其他删除失败会暴露，可能影响最终传播的异常。

如果应用希望保留原生成错误、另外报告清理失败，需要更明确的错误组合或记录策略。

这项政策不能由一个通用小例子替所有应用决定。

### 26.9 `flush` 后是否还需要 `fsync`？

`flush()` 把 Python 层缓冲交给底层。

`os.fsync()` 针对文件描述符请求更强的同步，仍需结合平台与文件系统理解。

若要讨论崩溃一致性，目录项的同步与替换顺序也可能相关。

本例没有把这套流程实现成完整的持久化协议，因此不宣称突然断电后一定得到某个版本。

先把普通失败时不破坏旧报告的行为做清楚，再根据实际可靠性要求增加必要机制。

### 26.10 `with` 自动回滚了所有事情吗？

没有。

它只是执行我们写好的退出逻辑。

如果代码块还发送了消息、修改了别的文件，这些副作用不会因为报告生成失败自动撤销。

“看上去像事务”的成对结构，也只能回滚你明确设计并实现的那部分状态。

## 第二十七章：完整实践六——限时取得锁，没拿到就不能误释放

### 27.1 普通 `with lock` 适合等待直到取得

但有些脚本希望最多等待一小段时间。

如果拿不到锁，就报告超时，不进入受保护代码块。

获取和释放必须配对，而且只有成功获取后才释放。

我们用 contextmanager 封装这个窄需求。

### 27.2 完整程序

```python
from contextlib import contextmanager
from threading import Lock


@contextmanager
def acquired_lock(lock, *, timeout):
    if not lock.acquire(timeout=timeout):
        raise TimeoutError("规定时间内未取得锁")
    try:
        yield lock
    finally:
        lock.release()


def run_timed_lock_demo():
    lock = Lock()
    with acquired_lock(lock, timeout=0):
        print("第一次取得锁")
        try:
            with acquired_lock(lock, timeout=0):
                print("不会进入这里")
        except TimeoutError:
            print("第二次获取失败，没有释放外层的锁")
        print("外层仍持有锁：", lock.locked())
    print("退出后已释放：", not lock.locked())
    return not lock.locked()


if __name__ == "__main__":
    run_timed_lock_demo()
```

主要输出：

```text
第一次取得锁
第二次获取失败，没有释放外层的锁
外层仍持有锁： True
退出后已释放： True
```

### 27.3 为什么 `try` 放在成功获取后面？

失败获取没有取得释放责任。

如果不加判断就在 finally 里 release，可能错误释放其他范围持有的普通锁，或触发异常。

这里成功获取以后才进入 try/finally，配对关系清楚。

### 27.4 为什么例子用 `timeout=0`？

为了立即得到结果，不让练习等待很久。

真实应用可以选择合适的有限超时，但要遵守具体锁 API 对参数的要求。

这个函数把 timeout 交给锁接口验证，没有声称接受所有自定义锁类型。

如果你想支持不同锁实现，应明确它们是否具有同样 acquire 签名与超时语义。

### 27.5 换成 `RLock` 会怎样？

同一线程再次取得可重入锁，可能成功。

因此第二层不一定超时。

这是锁类型的语义变化，不是上下文工厂失效。

同样的管理器围绕不同资源使用时，资源自身协议仍然决定具体行为。

### 27.6 退出以后可以证明程序没有数据竞争吗？

不能仅凭锁已释放就这样推断。

必须让所有访问同一共享状态的路径遵守一致同步规则。

如果某些代码绕开锁直接修改，局部 with 写得再正确，也无法保护完整状态。

上下文管理让配对更可靠，但不会替你找出所有共享访问。

## 第二十八章：完整实践七——SQLite 的 `with` 为什么不等于关闭连接？

### 28.1 同样语法，不同退出合同

这是学习上下文管理最值得亲自验证的例子之一。

普通文件的 with 退出会关闭。

标准库 SQLite 连接的上下文协议主要管理事务提交或回滚，不自动关闭连接。

因此，不能背成“with 后面任何资源都会 close”。

必须查具体对象对退出行为的定义。

### 28.2 用内存数据库验证

```python
import sqlite3
from contextlib import closing


def run_sqlite_context_demo():
    with closing(sqlite3.connect(":memory:", isolation_level="DEFERRED")) as connection:
        connection.execute("CREATE TABLE scores (value INTEGER NOT NULL)")
        connection.commit()

        with connection:
            connection.execute("INSERT INTO scores VALUES (?)", (80,))

        print("事务块后仍可查询：", connection.execute("SELECT COUNT(*) FROM scores").fetchone()[0])

        try:
            with connection:
                connection.execute("INSERT INTO scores VALUES (?)", (90,))
                raise ValueError("本次事务取消")
        except ValueError:
            pass

        values = [row[0] for row in connection.execute("SELECT value FROM scores")]
        print("回滚以后：", values)

    try:
        connection.execute("SELECT 1")
    except sqlite3.ProgrammingError:
        print("外层 closing 退出后，连接已关闭")
    return values


if __name__ == "__main__":
    run_sqlite_context_demo()
```

输出：

```text
事务块后仍可查询： 1
回滚以后： [80]
外层 closing 退出后，连接已关闭
```

### 28.3 为什么用了两种上下文？

外层 `closing` 拥有最终关闭连接的责任。

内层连接上下文拥有这一段事务的提交或回滚政策。

退出内层以后，连接仍然可以用于下一段事务。

退出外层以后，连接才彻底关闭。

这正好展示资源生命周期与事务范围可以不同。

### 28.4 进入连接上下文就一定立即开始事务吗？

不要把它理解成无条件执行 BEGIN 的通用包装。

实际事务状态与 sqlite3 的事务控制模式、执行语句及版本配置有关。

本例采用常见的 legacy 事务控制场景，明确设置 `isolation_level="DEFERRED"`，并通过插入语句产生对应事务。

Python 3.12 引入 `autocommit` 相关控制选项，使用新版事务政策时应按官方文档核对。

本篇的重点是上下文合同，不把不同模式混成一句“with 自动开关数据库”。

### 28.5 如果异常被内层 suppress 吞掉，事务会怎样？

连接退出时是否看见异常，会影响提交回滚路径。

如果你在事务块里面把本应导致失败的异常抑制掉，外层可能按正常退出处理。

所以异常抑制顺序会改变事务含义。

这是非常实际的后果：不是“错误已经打印过了”就代表事务必然回滚。

要么让失败传播到负责回滚的边界，要么明确执行符合业务要求的回滚政策。

### 28.6 这个知识能迁移到其他库吗？

可以迁移的是阅读思路：先确定上下文管理器的进入和退出合同。

不能迁移的是具体行为。

某个 HTTP 响应上下文可能释放连接，数据库会话可能回滚与关闭，临时目录可能删除内容。

不同库要分别核对，不能从文件对象的直觉推断全部。

## 第二十九章：完整实践八——计时范围，既记录成功，也保留失败

### 29.1 为什么计时也属于上下文管理？

它不一定持有需要关闭的外部资源。

但它仍然有成对行为：进入时取开始时间，退出时取结束时间。

因此，上下文管理不只适用于“有 close 的对象”。

它可以表达围绕代码块的前后动作，只要合同清楚。

### 29.2 使用可控制的时钟

```python
from contextlib import contextmanager
from time import perf_counter


@contextmanager
def measured_block(label, records, *, clock=perf_counter):
    started = clock()
    succeeded = False
    try:
        yield
        succeeded = True
    finally:
        records.append({
            "label": label,
            "elapsed": clock() - started,
            "succeeded": succeeded,
        })


def run_measured_block_demo():
    moments = iter((1.0, 1.5, 3.0, 3.25))
    records = []
    clock = lambda: next(moments)
    with measured_block("成功步骤", records, clock=clock):
        result = 2 + 3
    try:
        with measured_block("失败步骤", records, clock=clock):
            raise ValueError("失败")
    except ValueError:
        print("调用者仍然收到异常")
    print(result)
    print(records)
    return records


if __name__ == "__main__":
    run_measured_block_demo()
```

结果中的耗时分别是零点五和零点二五，成功标记分别为真和假。

这些数字来自注入时钟，是行为验证，不是真实机器跑分。

### 29.3 如果代码块 `return`，成功标记是什么？

对于这个生成器式管理器，普通返回离开会按正常退出继续 yield 后面的代码，所以标记为真。

如果你想把“提前返回”单独分类，就需要不同的业务信号，因为退出异常参数本身不会告诉你每一种普通控制流原因。

上下文协议能区分有无异常，但不是自动提供完整业务执行报告。

### 29.4 如果记录列表追加失败呢？

普通列表在正常内存条件下通常不会有额外业务错误，但极端资源问题或你换成自定义接收器时，仍可能失败。

这与日志计时装饰器中的记录接收器问题相同。

必要清理和可选观察动作应明确安排，不能为了加一条统计反而无意遮住真正业务错误。

### 29.5 装饰器计时与上下文计时怎么选？

计时整个函数，装饰器很方便。

只计时函数中间某一段，with 更直接。

如果需要拿到进入返回的临时资源，with 也更自然。

选择由你希望管理的范围决定，不需要在两种语法之间争论哪个永远更高级。

## 第三十章：练习题与参考答案——从“会写 with”到“会判断责任”

### 30.1 练习一：`as` 得到哪个对象？

定义一个管理器，进入返回整数十，管理器实例本身保存在 `manager`。

预测 `value is manager`。

参考答案是假。

```python
class NumberContext:
    def __enter__(self):
        return 10

    def __exit__(self, exc_type, exc_value, traceback):
        return False


manager = NumberContext()
with manager as value:
    print(value)
    print(value is manager)
```

`as` 绑定的是进入返回值，不是固定绑定管理器本身。

### 30.2 练习二：省略 `as` 是否仍然进入和退出？

使用第一题管理器，不接收 value，增加事件列表观察。

参考答案：仍然会执行进入和退出。

`as` 只是接收进入值的可选部分，不是触发上下文管理的必要开关。

### 30.3 练习三：变量是否离开就不存在？

在代码块里设置 `answer = 42`，离开后打印。

参考答案：普通 with 不创建新的函数作用域，因此在相应外层作用域里仍然能看到这个绑定。

但如果整个 with 位于函数内部，函数局部变量的规则仍然适用。

不能从“with 不创建作用域”推出函数内部变量会自动变成全局变量。

### 30.4 练习四：进入失败是否调用退出？

```python
class FailingEnter:
    def __enter__(self):
        print("进入方法开始")
        raise ValueError("进入失败")

    def __exit__(self, exc_type, exc_value, traceback):
        print("退出方法")
```

参考答案：这次失败的进入不会自动调用该对象的退出方法。

如果进入阶段已经取得部分资源，必须在进入方法自己的失败路径处理。

注意与“进入成功后 as 解包失败”区分，后者会执行退出。

### 30.5 练习五：抑制以后从哪里继续？

在 `with suppress(ValueError)` 内写开始事件、失败转换、结束事件。

外面再写后续事件。

参考答案：开始与后续出现，内部结束不会出现。

抑制不是恢复到失败语句下一行继续执行。

这是异常控制流的跳转，不是错误自动修复。

### 30.6 练习六：只记录错误，为什么反而吞掉了？

```python
from contextlib import contextmanager


@contextmanager
def mistaken_logger(events):
    try:
        yield
    except ValueError:
        events.append("记录错误")
```

请修复。

参考答案是在记录后加 `raise`，让异常继续传播。

如果还需要总是释放资源，再在正确范围使用 finally。

不要在生成器里写 `return False`，误以为它与类式退出的返回假完全等价。

### 30.7 练习七：生成器产生两次，为什么不行？

```python
from contextlib import contextmanager


@contextmanager
def twice():
    yield 1
    yield 2
```

参考答案：contextmanager 的协议要求恰好一次交接。

第二次 yield 会让退出无法正常完成，并报告对应错误。

如果你想提供多项数据，可以一次 yield 一个列表或迭代器，而不是多次交接控制权。

但 yield 迭代器后，还要保证消费发生在资源有效范围内。

### 30.8 练习八：一组动态文件，第三个打开失败

要求验证前两个流关闭，而不是只看程序打印了“失败”。

参考答案沿用第二十三章的 observed 列表，直接检查两个流的 `closed`。

这能证明资源状态，而不是仅证明异常被捕获。

还可以增加读取阶段失败，让退出栈同样关闭全部已进入资源。

### 30.9 练习九：`callback` 返回真能抑制异常吗？

不能。

普通 callback 的返回值不按退出协议处理。

如果你需要异常参数与抑制能力，应使用符合 `__exit__` 约定的退出函数或上下文管理器。

判断标准是方法协议，不是“它都在同一个栈里”。

### 30.10 练习十：清理顺序预测

依次登记 A、B、C 三个回调。

参考答案是 C、B、A。

如果在登记 B 后抛异常，C 根本没有登记，只有 B、A 收尾。

如果 B 的登记表达式在取得资源后、正式登记前失败，还需要分析是否出现未登记资源。

这就是为什么获取与登记应紧密安排。

### 30.11 练习十一：`pop_all` 后谁负责关闭？

参考答案是返回的新栈，或者保存它的资源拥有者。

原栈不再执行这些转移出去的回调。

`pop_all` 不会立即关闭，也不表示永远无需关闭。

请画出从临时 owner 到返回对象的引用关系，再运行资源集合实践。

### 30.12 练习十二：借来的文件不能关闭

编写一个读取第一行的函数，输入可以是路径或文本流。

路径由函数打开关闭，流由调用者管理。

参考答案使用 `nullcontext(stream)` 统一代码块。

再验证读取位置已经向后推进，说明“不拥有关闭责任”不等于“不改变任何状态”。

如果业务还要求恢复位置，需要明确流是否可 seek，并另外保存恢复位置。

### 30.13 练习十三：临时属性原来是 None

用第二十五章工具设置一个原值为 None 的属性。

退出以后应该保留 None，还是删除？

参考答案是保留 None。

专门的缺失哨兵用来区分“不存在”和“存在但值为 None”。

如果你把 None 当作缺失标记，就会错误删除原属性。

### 30.14 练习十四：临时属性原来不存在

退出以后应恢复成不存在。

参考答案检查 `hasattr` 为假。

如果只是设回 None，目标对象的结构已经改变，不符合完整恢复到进入前状态的政策。

这道题让你关注状态模型，而不只关注某个显示值。

### 30.15 练习十五：输出重定向能隔离其他线程吗？

不能把它当作线程私有输出。

它改变全局流引用，其他线程使用同一引用时可能被影响。

如果你控制函数接口，更适合传入输出流。

如果要捕获日志，给处理器直接传入内存流比全局重定向更准确。

### 30.16 练习十六：退出以后临时目录还在吗？

使用 TemporaryDirectory 创建目录，块内创建文件，块外检查路径。

参考答案：正常清理成功时，目录应已移除。

注意路径字符串仍然存在，但对应文件系统对象不存在。

再次区分名称或路径对象的生命周期，与外部资源生命周期。

### 30.17 练习十七：同步文件能直接 `async with` 吗？

普通内置文件对象不因此自动提供异步上下文协议。

异步函数里可以写同步 `with open`，但 I/O 阻塞性质仍然存在。

需要异步文件方案时，应选择明确的执行方式，而不是只改一个关键字。

### 30.18 练习十八：提前停止异步生成器

使用 aclosing，消费第一个值就 break。

要求生成器清理事件出现在“离开管理范围”之前。

参考答案见第十九章。

验证的是清理顺序确定，不是生成器剩余数据被自动消费。

### 30.19 练习十九：SQLite 块结束，连接为什么还能用？

因为连接上下文主要处理事务，并不自动关闭连接。

外层 closing 才承担关闭责任。

请在事务退出后执行查询，再在 closing 退出后执行查询并捕获 ProgrammingError。

两次操作验证的是两种不同生命周期边界。

### 30.20 练习二十：同一文件替换失败，旧内容是否保留？

在 replacement_text_writer 的块内写一半后故意抛 ValueError。

参考答案是目标仍然保留旧内容，临时文件被清理。

再写一次正常流程，目标变成新内容。

这项验证只覆盖我们实现的生成失败路径，不代表所有断电、权限、并发与文件系统故障都已验证。

### 30.21 练习二十一：`return` 与清理异常

定义一个管理器退出时抛 RuntimeError，在代码块里 return 100。

参考答案：调用者最终收到 RuntimeError，正常返回被退出失败打断。

请解释为什么“退出一定会执行”并不等于“函数一定返回原值”。

### 30.22 练习二十二：装饰器形式会传入进入结果吗？

不会自动给被装饰函数添加一个资源参数。

如果函数需要这个资源对象，显式 with as 通常更清楚。

装饰器形式适合围绕整个调用建立范围效果，而不是所有资源传递需求。

### 30.23 练习二十三：同一个生成器式管理器实例能用两次吗？

通常不能。

应每次重新调用工厂，取得新的实例。

工厂可重复调用，与实例可重复进入，要分开表达。

如果你把一个创建好的管理器存在默认参数里，随后每次函数调用都重用它，也可能触发这个问题。

### 30.24 练习二十四：异常被里面 suppress 掉，外层还看见吗？

如果确实匹配并被抑制，外层退出会按后续更新后的异常状态执行。

因此事务、统计或其他异常敏感管理器的行为可能改变。

请用事件管理器嵌套 suppress，观察外层收到的 `exc_type`。

这个题不是教你隐藏失败，而是让你知道异常政策会沿嵌套结构影响外层。

## 第三十一章：真实写代码时，你还可能继续问这些问题

### 31.1 上下文管理器是不是必须管理外部资源？

不是。

计时范围、临时状态、日志上下文都可以使用。

共同点是存在一段明确范围，以及围绕它的进入退出政策。

但不要为了套用模式，把一行普通计算也包装成管理器。

如果没有成对职责，普通函数更直观。

### 31.2 能不能把任何对象补上两个方法就算支持？

从协议层面，需要类型提供正确的进入退出实现。

但“方法存在”只是表面条件。

它们还必须按约定取得、返回、清理并处理异常。

如果你的 `__exit__` 只是 `pass`，资源不会因为方法名正确就自动释放。

协议提供挂接点，不替你实现资源行为。

### 31.3 可以在实例上临时赋 `__enter__` 吗？

Python 特殊方法查找通常涉及类型层级的专门规则，不应把它等同于任意普通实例属性查找。

因此，最清楚的方式是在类上定义相应特殊方法，或使用 contextlib 的正式适配工具。

不要把临时给一个实例塞两个同名 lambda 当作通用上下文扩展方案。

如果你需要动态适配，写一个小包装类或工厂更可读。

### 31.4 `__exit__` 的 traceback 参数会占用资源吗？

追踪对象可能引用栈帧，而栈帧又引用局部变量。

如果你长期保存完整异常信息，可能延长很多对象的生命周期。

一般退出方法用完就结束，不需要把追踪对象永久放进全局列表。

如果只是为了诊断，可以提取明确的文本或字段，结合日志策略处理。

这与缓存持有对象引用的现象有相似根源。

### 31.5 上下文管理可以取代垃圾回收吗？

它们解决不同问题。

垃圾回收管理对象内存等生命周期。

上下文管理让外部资源与状态恢复有明确的代码范围。

你不应依赖某个 Python 实现恰好立即回收文件对象，来替代显式关闭。

特别是循环引用、调试器保留引用或不同实现，都可能改变回收时机。

### 31.6 把 `with` 放进生成器函数里，有什么容易忽略？

生成器只有在推进时才执行。

如果它在 with 内 yield，暂停期间上下文可能仍然保持进入状态。

调用者若不继续消费，也不明确关闭，资源可能迟迟不释放。

所以“我在生成器内部写了 with”不等于每次拿到一项后立即关闭。

需要看 yield 与资源范围的关系。

### 31.7 给一个具体的文件生成器例子

```python

def iter_nonempty_lines(path):
    with open(path, encoding="utf-8") as stream:
        for line in stream:
            stripped = line.strip()
            if stripped:
                yield stripped
```

完整消费时，with 最终正常退出。

如果只取第一项就把生成器长期保存在变量里，文件可能仍然打开。

如果调用者拥有“只用一部分”的场景，可以使用 closing 管理生成器：

```python
from contextlib import closing

with closing(iter_nonempty_lines("data.txt")) as lines:
    first = next(lines, None)
```

退出时调用生成器的 close，促使相应 finally 与文件退出运行。

这与异步生成器的 aclosing 在思路上对应。

### 31.8 从函数里返回生成器表达式，文件已经关了怎么办？

看这个反面例子：

```python

def broken_lines(path):
    with open(path, encoding="utf-8") as stream:
        return (line.strip() for line in stream)
```

函数返回前退出 with，随后才消费生成器表达式时，流已经关闭。

如果数据不大，可以在块内收集列表再返回。

如果要惰性读取，可以使用前一节把 with 放进生成器函数的写法，并明确关闭责任。

两种方案的内存与生命周期不同。

### 31.9 一个管理器可以多次 yield 不同资源供调用者逐个处理吗？

不是在同一次 contextmanager 协议里多次 yield。

你可以一次 yield 一个资源集合，或一次 yield 一个迭代器。

也可以让调用者循环创建多个独立管理器，每次管理一个资源。

关键是区分“给代码块的一次控制权交接”与“业务数据迭代”。

把它们混在一起，会造成生成器没有停止或资源范围错误。

### 31.10 管理器内部用 `except Exception`，能保证中断时清理吗？

不能仅凭这个 except 保证。

`SystemExit`、`KeyboardInterrupt` 等不属于普通 Exception 分支。

必要清理一般使用 finally，让正常异常展开经过它。

如果你为了记录或转换普通业务失败使用 except Exception，也不要把它当作所有退出路径的代替。

清理职责与错误分类应分开。

### 31.11 `__enter__` 返回 self，是不是最标准写法？

很常见，但不是必须。

如果调用者需要管理器自身的方法，返回 self 合理。

如果管理器只是拥有一个文件，调用者需要文件对象，返回文件更直接。

如果只需要范围效果，可以返回 None。

选择应该符合使用者需要，不是为了统一形式一律返回 self。

### 31.12 `with` 会自动撤销列表修改吗？

不会。

```python
from contextlib import nullcontext

values = []
with nullcontext():
    values.append(1)
print(values)
```

列表仍然含一。

是否恢复由管理器具体实现决定。

空管理器没有回滚政策，文件管理器也不会自动撤销程序里其他对象的修改。

### 31.13 想实现一个“失败就恢复列表”的管理器，可以吗？

可以针对明确模型实现浅快照，但需要说明嵌套对象和外部副作用边界。

```python
from contextlib import contextmanager


@contextmanager
def restore_list_on_error(values):
    before = list(values)
    try:
        yield values
    except BaseException:
        values[:] = before
        raise
```

这能恢复列表的元素引用与顺序。

如果元素本身是字典，代码块修改了字典内部，浅快照不会自动恢复字典内容。

如果恢复赋值本身失败，还要处理清理错误。

所以这个小工具不能被称为任意对象事务系统。

### 31.14 为什么在这段用 BaseException？

目的是在异常离开时尝试恢复，然后立即重新抛出原信号。

它没有把取消、中断或退出当作成功吞掉。

但如果你的恢复政策只针对某些业务异常，也可以更具体。

关键是理解捕获范围与重新传播之间的关系，不能只机械记“永远不能捕获 BaseException”或“清理就捕获所有”。

### 31.15 资源归属能不能靠命名约定表达？

可以辅助，但最好还体现在接口结构里。

例如 `open_group()` 返回需要关闭的对象，`with opened_group()` 明确范围。

接受 `stream` 参数的函数说明只是借用，不关闭。

返回普通字符串或列表，则调用者不需要继续管理打开资源。

名字、文档和实际行为一致，才能减少误用。

### 31.16 第三方客户端提供 `with`，是否一定每次都新建连接？

不一定。

有的进入建立连接，有的借用已有会话，有的从池中取得资源。

退出可能关闭，也可能归还池，甚至只改变状态。

因此，不能从语法推断成本或网络行为。

具体库的文档、源码与实际验证才决定它的合同。

### 31.17 一个对象既支持迭代，又支持上下文，该怎么搭配？

这两个协议可以独立存在。

文件对象既能 with，也能 for。

```python
with open("data.txt", encoding="utf-8") as stream:
    for line in stream:
        print(line.rstrip("\n"))
```

with 管生命周期，for 管逐项取得数据。

提前 break 会离开 for，随后如果也离开 with，文件就关闭。

如果 break 后仍在同一个 with 内做其他工作，文件仍然保持打开。

范围由缩进结构和控制流决定。

### 31.18 `ExitStack` 可以让所有回调都完全成功吗？

它能按结构展开回调，并处理相应异常状态变化。

但某个删除、关闭或网络释放本身失败，工具不能保证外部动作一定成功。

不要把“调用过清理函数”与“资源确实已释放且所有数据安全”完全等同。

必要时需要检查资源 API 的返回与错误，设计重试或记录。

### 31.19 代码块里关闭了资源，退出时又关闭会怎样？

有些 close 是幂等的，重复调用可以接受。

有些释放动作重复执行会失败或破坏状态。

管理器与调用者应该明确谁负责关闭，避免双方同时承担而没有约定。

不要因为文件 close 通常可重复，就推断所有 release、commit 或删除动作都一样。

### 31.20 上下文管理器怎样写得更容易测试？

把时钟、输出流、事件接收器等外部依赖作为参数。

把资源状态暴露为可以合理观察的属性，而不是只能靠打印猜测。

验证正常路径，也验证进入失败、块内失败、提前返回和清理失败。

不需要为每一行都写一个镜像测试，重点检查真正可能改变资源与异常合同的边界。

本篇用事件列表、closed 属性、临时目录和固定时钟，都是为了让这些结论能被具体检查。
