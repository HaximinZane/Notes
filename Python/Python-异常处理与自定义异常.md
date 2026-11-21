# Python 异常处理与自定义异常：从看懂报错，到安排失败、清理、重试和业务边界

## 零基础先导：为什么程序需要“异常”，而不是只有返回值？

程序运行时总有无法事先确保的情况：文件不存在、输入的年龄不是数字、网络突然断开、数据库写入失败。**异常（exception）** 是 Python 表达“这次操作没能正常完成”的一种机制，它能携带错误类型与信息，并沿着调用链传播，直到有代码处理它。

### 从一行简单转换开始

```python
age = int("20")       # 合法：字符串 "20" 转换为整数 20
print(age)
```

如果输入改为 `"二十"`，`int` 会抛 `ValueError`。这不是 Python 随便打印一条红色文字，而是**执行流程发生了变化**：当前语句没有正常返回结果，如果没人捕获，异常会继续向调用方传播，最终显示回溯并终止该未处理的执行流程。

```python
raw = "二十"

try:
    age = int(raw)                 # ① 尝试可能失败的操作
except ValueError as exc:         # ② 只处理字符串无法转成整数这一类失败
    print("年龄必须是整数：", exc)  # ③ 提示错误原因
else:
    print("转换成功：", age)         # ④ try 块未抛异常时才执行
finally:
    print("本次检查结束")           # ⑤ 正常或异常退出都会尝试执行
```

不要把 `except Exception: pass` 当万能办法：它可能把重要的程序错误吞掉，让后续数据变得不可信。也不是任何错误都应该重试；如文件路径不存在、JSON 格式错误，通常需要先修正输入或程序配置。

### 为什么会有 `raise` 与自定义异常？

当你的业务知道“分数不能为负”时，可以主动抛出更明确的错误：

```python
def set_score(score):
    if score < 0:
        raise ValueError("分数不能是负数") # 主动报告违反规则
    return score
```

`raise` 会中断当前正常流程，向上交给调用者；`return` 则是正常返回结果。调用者可以根据错误类型决定是提示用户、记录日志、清理资源还是结束操作。自定义异常适合**领域语义明确**的情况，不是为了给每一个函数都创造一个新异常类。

遇到异常先分三层：**哪里发生（traceback）→ 为什么发生（异常类型与信息）→ 谁负责处理（调用边界）**。后文讲 `raise ... from ...`、`finally`、`with`、异常组时，都应回到这条主线。

---

假设你正在写一个学习笔记整理脚本。

它要读取一个文件，把里面的天数转换成整数，再生成一份报告。

第一次运行成功了。

第二次，文件被挪走了。

第三次，别人把 `30` 写成了 `三十`。

第四次，报告已经写了一半，磁盘写入失败。

你可能会问：

“能不能让程序不要一报错就退出？”

当然可以处理一部分错误。

但接下来的问题更重要。

如果只是继续运行，后面的数据还可信吗？

如果失败时返回 `None`，调用者怎么知道是没有数据，还是根本没读成功？

如果打印一句‘出错了’，我以后还能找到真实原因吗？

这一篇就顺着这些实际问题，逐步讲清 `try`、`except`、`else`、`finally`、`raise`、异常链、自定义异常，以及失败以后的资源和数据状态。

普通例子采用 Python 3.11 及以上语法。

`ExceptionGroup`、`except*`、`add_note()` 会注明它们从 3.11 开始可用；涉及 3.14 的变化会单独说明。

示例只写新建临时目录里的练习文件，或者使用内存中的小数据。

## 第一章：先亲眼看一次异常，别急着把红字藏起来

### 1.1 一个正常转换

```python
text = "30"
days = int(text)
print(days + 1)
```

输出：

```text
31
```

`int(text)` 接受一个符合整数表示规则的字符串，把它转成整数。

成功时，下一行继续执行。

### 1.2 如果读到的是“三十”呢？

```python
text = "三十"
days = int(text)
print("转换之后")
```

这是故意失败的例子。

通常会看到包含以下最后一行的回溯：

```text
ValueError: invalid literal for int() with base 10: '三十'
```

后面的 `print("转换之后")` 不会按普通顺序执行。

这不是 `int()` 给 `days` 填入一个特殊失败数字。

它是在告诉调用者：这次操作无法按照当前约定正常完成。

### 1.3 异常的英文名字由什么组成？

`ValueError` 中的 value 是值，error 是错误。

通常表示参数的种类可能合适，但具体值不符合操作要求。

字符串可以作为 `int()` 的输入，可是这个字符串不是它接受的整数写法。

这与把一个列表交给不接受列表的接口所产生的 `TypeError` 又有所区别。

不过最终应以具体函数契约为准，不能仅凭英文直译预测所有函数抛什么异常。

### 1.4 为什么 Python 不直接返回一个失败值？

一个函数可能正常返回 `0`、空字符串、空列表，甚至 `None`。

如果再挑其中某个值代表失败，调用者就容易分不清正常结果和错误。

异常把正常返回路径与失败传播路径分开。

它还携带类型、消息和调用位置，允许上层根据能力决定是否恢复。

这不表示所有“没找到”都必须抛异常；设计接口时仍要看缺失是否属于正常业务情况。

### 1.5 exception 与 error 有什么关系？

exception 可以理解为偏离正常执行流程的一种情况。

Python 用异常对象表达和传播这种情况，许多具体类名以 `Error` 结尾。

也有不以 Error 结尾的异常，例如 `StopIteration`、`KeyboardInterrupt`。

所以不要以类名有没有 Error 判断它是不是异常。

具体继承关系才决定 `except` 能否捕获。

## 第二章：回溯不是一堵英文墙，它在给你一条调用路线

### 2.1 两层函数的例子

```python
def parse_days(text):
    return int(text)


def build_report(text):
    days = parse_days(text)
    return f"学习计划：{days} 天"


print(build_report("三十"))
```

这段会故意触发异常。

回溯通常按照调用路径列出模块入口、`build_report()`、`parse_days()`，最后给出 `ValueError` 和消息。

真实文件名、行号与 Python 版本的展示样式可能不同，不需要逐字背输出。

### 2.2 应该从哪里开始读？

先看最后一行：异常类型与消息。

再看最靠近失败位置的那一段，确认哪个表达式失败。

然后向上看调用者是怎样传入这个值的。

本例最底层失败的是 `int(text)`，但真正需要改动的位置可能在读文件、表单转换或调用参数处。

最后一行告诉你发生了什么，调用路径帮助你判断为什么会把这个值送到这里。

### 2.3 为什么不是永远只修改回溯最后一行？

假设 `parse_days()` 的职责就是把数字文本转换成天数。

它拒绝 `三十` 可能完全符合约定。

你真正需要做的是在输入边界向用户说明“请输入阿拉伯数字”，或者明确增加中文数字解析规则。

随便在底层 `except: return 0`，只会让坏输入悄悄变成零天计划。

错误消失了，业务含义却被改坏。

### 2.4 回溯中的箭头一定只有 Python 代码吗？

具体显示取决于解释器版本、执行环境与调用链。

有时底层 C 实现只通过最后的 Python 调用位置表现出来。

Python 3.11 的更细错误位置展示能帮助定位子表达式，但它不是所有环境都完全一样的格式协议。

不要靠解析终端回溯文本来实现程序逻辑。

需要结构化处理时，使用异常对象与 traceback 模块提供的接口。

### 2.5 我能把回溯复制给别人吗？

可以，但先检查里面是否包含口令、令牌、个人数据或不该公开的路径。

不要为了隐私把所有诊断都彻底删掉；可以保留类型、函数位置和脱敏后的输入。

良好的报错信息既能帮助人定位问题，也清楚知道哪些内容可以展示给谁。

## 第三章：语法错误、运行时异常和逻辑错误，不是一回事

### 3.1 代码根本没组成合法语法

下面是不完整语法示意，不可作为正常 Python 块运行：

```text
if ready
    print("开始")
```

`if` 行末缺少冒号，解释器解析源代码时会报 `SyntaxError`。

同一份文件里普通的运行期 try 不能保护它自己尚未成功解析的语法结构。

### 3.2 已经开始运行，某一步失败

```python
items = ["open", "pathlib"]
print(items[5])
```

这段语法正确，但索引超出范围，运行时触发 `IndexError`。

这种情况可以由恰当位置的 `try/except` 捕获。

是否应该捕获，又取决于这是正常可预期输入，还是程序自己的下标算错。

### 3.3 没报错，却算错了

```python
price = 100
quantity = 3
total = price + quantity
print(total)
```

输出：

```text
103
```

如果你想算总价，应当是乘法。

Python 不知道你的业务目标，所以不会自动抛异常。

异常处理不是正确性验证的替代品；逻辑错误仍需要明确的需求、样本和测试。

### 3.4 能捕获动态代码的 SyntaxError 吗？

可以，因为这里被解析的是运行中交给 `compile()` 的另一段源代码。

```python
source = "if True print('x')"
try:
    compile(source, "<practice>", "exec")
except SyntaxError:
    print("动态源码语法不合法")
```

输出：

```text
动态源码语法不合法
```

这不表示普通源文件自己的语法错误可以靠同一文件中一个 try 包起来解决。

也不应为了演示异常而把任意外部文本当代码执行。

这里仅编译一个固定错误字符串，没有执行外部输入。

## 第四章：`try/except` 的最小语法，逐行看它怎样改变路线

### 4.1 先保护真正可能失败的那一步

```python
text = "三十"

try:
    days = int(text)
except ValueError:
    print("请输入数字，例如 30。")

print("程序来到后续位置")
```

输出：

```text
请输入数字，例如 30。
程序来到后续位置
```

`try` 块里的转换失败。

Python 查找匹配的 `except ValueError`，进入它的块。

处理块正常结束以后，继续执行整个 try 语句之后的代码。

### 4.2 它会回到失败那一行重试吗？

不会自动回去。

异常发生后，try 块中失败点后面的普通语句会被跳过。

进入 except 不意味着补做失败语句。

```python
try:
    print("转换前")
    value = int("bad")
    print("转换后")
except ValueError:
    print("进入错误处理")
```

输出：

```text
转换前
进入错误处理
```

`转换后` 没执行。

如果需要重试，应明确使用循环，而且让用户有机会修改输入或让条件发生实际变化。

### 4.3 没发生异常，except 会执行吗？

不会。

```python
try:
    value = int("30")
except ValueError:
    print("转换失败")

print(value)
```

输出：

```text
30
```

try 不是“先做一次普通流程，再检查错误处理代码”。

它建立的是发生异常时的控制路径。

### 4.4 `except ValueError as error` 多了什么？

它把本次捕获的异常对象绑定到变量 `error`。

```python
try:
    int("bad")
except ValueError as error:
    print(type(error).__name__)
    print(isinstance(error, Exception))
```

输出：

```text
ValueError
True
```

可以检查类型、消息或某些具体属性。

`error` 不是一个布尔开关，而是这次失败的对象。

### 4.5 except 后面写的是类，还是字符串？

写异常类或异常类组成的元组。

不是写英文类名字符串。

```python
try:
    int("bad")
except (ValueError, TypeError) as error:
    print(type(error).__name__)
```

输出：

```text
ValueError
```

这个元组表示两类失败采用同一套处理。

不要写 `except ValueError or TypeError`，那是普通布尔表达式，不表示你想要的两个异常类别。

为了兼容 3.11，本篇多个类型始终保留括号元组写法；不要把较新语法样式不加说明地复制到旧解释器。

## 第五章：异常继承与捕获顺序，为什么 Exception 应该放后面？

### 5.1 捕获会匹配子类

`FileNotFoundError` 是 `OSError` 的子类。

所以捕获 `OSError` 可以捕获文件不存在，也可能捕获权限不足、磁盘相关等其他系统错误。

```python
print(issubclass(FileNotFoundError, OSError))
print(issubclass(ValueError, Exception))
```

输出：

```text
True
True
```

这就是父类捕获较大范围的原因。

### 5.2 按从上到下匹配，第一个合适的处理块生效

```python
try:
    raise FileNotFoundError("练习文件不存在")
except FileNotFoundError:
    print("具体处理：文件不存在")
except OSError:
    print("其他文件系统问题")
```

输出：

```text
具体处理：文件不存在
```

如果把 `except OSError` 放前面，它已经匹配这个子类，后面的具体分支就不会得到机会。

因此通常具体类型在前，宽泛类型在后。

### 5.3 为什么不能总写 except Exception 然后继续？

它会捕获很多普通程序错误。

包括你自己的变量拼错、属性用错、参数数量写错等。

如果把它们全部显示为“文件格式不正确”，用户会反复改文件，真正的程序缺陷却藏起来了。

宽泛捕获可以用于最外层任务记录和统一失败出口，但不意味着内层所有失败都应该被当成可恢复输入问题。

### 5.4 BaseException 与 Exception 区别在哪里？

`BaseException` 是更高层的基类。

`Exception` 覆盖大部分普通应用错误。

`KeyboardInterrupt`、`SystemExit` 等直接处在更高层分支，不属于普通 Exception 子类。

这让一般的 `except Exception` 不会随便吞掉用户中断和退出请求。

```python
print(issubclass(KeyboardInterrupt, BaseException))
print(issubclass(KeyboardInterrupt, Exception))
print(issubclass(SystemExit, Exception))
```

输出：

```text
True
False
False
```

### 5.5 裸 except 为什么特别宽？

`except:` 没有限定类型，会捕获更广范围，包括退出和中断。

如果你写：

```python
try:
    raise KeyboardInterrupt
except:
    print("用户想中断，却被吞掉了")
```

会输出：

```text
用户想中断，却被吞掉了
```

这个固定演示不实际等待键盘，但让你看清捕获范围。

普通业务代码应尽量避免裸 except。

资源清理需要覆盖中断时，更适合用 `finally`，或清理后立即重新抛出，而不是继续装作任务成功。

## 第六章：捕获范围要小，才知道你到底在处理哪一次失败

### 6.1 一个看似方便的“大 try”

假设你把读配置、解析数字、调用统计、写结果都放在同一个 try 里。

最后写 `except ValueError: print("天数不是数字")`。

如果统计函数内部也因为其他原因抛 ValueError，你就会给出错误提示。

异常类型相同，不代表业务原因相同。

### 6.2 把转换与后续计算分开

```python
def parse_day_count(text):
    try:
        days = int(text)
    except ValueError as error:
        raise ValueError("天数必须写成整数，例如 30") from error

    if days < 1 or days > 365:
        raise ValueError("天数必须在 1 到 365 之间")
    return days


print(parse_day_count("30"))
```

输出：

```text
30
```

这里只有 `int(text)` 的失败被翻译成整数格式提示。

范围检查单独表达，所以 `0`、`366` 会得到另一种更准确的说明。

如果你要求输入必须是字符串，还应先做类型检查，不要默默接受所有 `int()` 能转换的对象。

### 6.3 有没有必要捕获 TypeError？

取决于函数是否承诺接收任意外部对象。

如果上层已经保证 text 是字符串，TypeError 可能表示调用者写错。

如果这是一个直接接收 JSON 字段的公共入口，字段可能是 null、数组或对象，就应该在边界明确拒绝这些类型。

不要为了“全面”捕获所有可能异常，而不解释哪些是可预期输入，哪些是程序缺陷。

### 6.4 处理块自己出错，会回到同一个 except 吗？

不会这样自动循环。

```python
try:
    try:
        int("bad")
    except ValueError:
        raise RuntimeError("错误提示模块自己失败")
except RuntimeError as error:
    print(str(error))
```

输出：

```text
错误提示模块自己失败
```

内层处理块产生的新异常继续向外传播。

所以异常处理代码也应尽量简单，避免在失败路径里再依赖一串不可靠操作。

## 第七章：`else`——“成功以后才做”为什么不一定放进 try？

### 7.1 一份完整结构

```python
text = "30"
try:
    days = int(text)
except ValueError:
    print("天数格式错误")
else:
    print(f"已经得到 {days} 天")
```

输出：

```text
已经得到 30 天
```

`else` 在 try 块正常完成、没有向外发生异常时执行。

它把“受保护的操作”和“成功后的其他工作”分开。

### 7.2 如果 else 里面抛 ValueError，会被前面的 except 捕获吗？

不会。

```python
try:
    try:
        days = int("30")
    except ValueError:
        print("数字格式错误")
    else:
        raise ValueError("成功后报告生成失败")
except ValueError as error:
    print(str(error))
```

输出：

```text
成功后报告生成失败
```

这正是 else 的实际价值：不要把后续工作的同类异常误判成前面转换步骤的失败。

### 7.3 有 return 时，else 总会执行吗？

不会。

try 块通过 return、break、continue 离开时，不会因为“没抛异常”就继续执行对应 else。

“正常完成”还包含控制流程是否自然走到块尾。

初学时可以先按直观写法理解：else 是 try 顺利走完以后才进入的分支，不是任何离开方式都会经过的钩子。

## 第八章：`finally`——无论成功失败，都安排清理

### 8.1 先看顺序

```python
try:
    print("开始")
    int("bad")
except ValueError:
    print("处理输入错误")
finally:
    print("结束本次工作")
```

输出：

```text
开始
处理输入错误
结束本次工作
```

finally 面向离开这个结构时必须安排的工作。

它不等于成功分支。

### 8.2 没有 except，也可以有 finally

```python
def attempt():
    try:
        raise ValueError("仍要向外报告")
    finally:
        print("先清理")


try:
    attempt()
except ValueError:
    print("外层收到失败")
```

输出：

```text
先清理
外层收到失败
```

finally 执行以后，原来的异常继续传播。

这让底层可以负责资源，上层负责决定错误怎么展示。

### 8.3 return 会经过 finally 吗？

普通控制流程下会。

```python
def get_value():
    try:
        return 7
    finally:
        print("返回前执行清理")


print(get_value())
```

输出：

```text
返回前执行清理
7
```

所以你不需要在每个 return 前复制一段资源释放代码。

### 8.4 finally 里面也 return，会怎样？

它可能覆盖 try 里原本的返回值，甚至压住原本的异常。

```python
def dangerous_result():
    try:
        raise ValueError("原本应该暴露的失败")
    finally:
        return "看起来成功"


print(dangerous_result())
```

在允许运行此写法的版本中输出：

```text
看起来成功
```

这是故意展示反例，不是推荐模板。

Python 3.14 会对从 finally 块向外转移控制的某些 return、break、continue 写法发出 SyntaxWarning，帮助发现这类误用。

本篇实际运行使用 3.12，因此不能把未出现警告说成所有版本都不会提示。[try/finally 官方语义](https://docs.python.org/3.14/reference/compound_stmts.html#finally-clause)

### 8.5 finally 保证电脑断电也能执行吗？

不能。

它保证的是 Python 正常异常展开和相关控制流程中的清理安排。

操作系统强制杀死进程、断电、`os._exit()` 等路径可能不执行正常清理。

因此，重要结果还需要安全写入、事务、恢复和幂等设计。

不能把 finally 当作所有故障下的绝对保险。


## 第九章：`raise`——我自己的规则不满足时，怎样明确失败？

### 9.1 不需要等解释器替你发现

假设计划天数必须在 `1` 到 `365` 之间。

`int("0")` 能成功转换，却不符合你的业务规则。

你可以自己抛异常：

```python
def validate_days(days):
    if type(days) is not int:
        raise TypeError("days 必须是整数，不能是布尔值或其他类型")
    if not 1 <= days <= 365:
        raise ValueError("days 必须在 1 到 365 之间")
    return days


print(validate_days(30))
try:
    validate_days(0)
except ValueError as error:
    print(error)
```

输出：

```text
30
days 必须在 1 到 365 之间
```

### 9.2 `raise ValueError("...")` 里面是什么？

`ValueError` 是异常类。

`ValueError("...")` 创建一个异常实例。

`raise` 把它放到异常传播路径中。

括号里的字符串是这次失败的解释，不是 Python 负责理解的一段处理指令。

所以消息要写具体：哪个字段、接受什么、当前出了什么问题。

### 9.3 `raise ValueError` 可以吗？

可以直接抛出异常类，Python 会实例化它。

但没有具体消息时，使用者可能不知道为何失败。

公共接口一般更值得写成带上下文的实例。

并不是消息越长越好，而是要支持下一步处理。

“出错了”太少；把整个含秘密的配置字典塞进消息又太多。

### 9.4 捕获后想继续向外传，用什么？

使用不带参数的 `raise`。

```python
def parse_with_notice(text):
    try:
        return int(text)
    except ValueError:
        print("这一层只补充记录，仍然向外报告")
        raise


try:
    parse_with_notice("bad")
except ValueError:
    print("外层确实收到了异常")
```

输出：

```text
这一层只补充记录，仍然向外报告
外层确实收到了异常
```

这表示当前层没有恢复任务，只做了记录或清理。

### 9.5 `raise error` 和裸 raise 一样吗？

它们都能抛出这个对象，但重新抛出的回溯位置处理并不完全相同。

在 except 中想保留原有传播语义，通常优先裸 `raise`。

不要为了“显得明确”把每个重新抛出都写成 `raise error`，使回溯增加不必要的当前行。

如果需要换一个抽象层次的异常，则用 `raise NewError(...) from error`，后面会讲。

### 9.6 except 外面可以裸 raise 吗？

只有当前存在正在处理的异常时，裸 raise 才有明确对象。

没有活跃异常时会触发 `RuntimeError`。

不要把 `raise` 当成“让程序回到之前某个失败点”的命令。

它是在当前异常上下文中重新传播，不是时间回退。

### 9.7 assert 与 raise ValueError 哪个用于用户输入？

用户输入应使用明确的条件和异常。

`assert` 主要用于表达开发期内部不变量，优化模式可能移除断言语句。

```python
def require_positive(value):
    if value <= 0:
        raise ValueError("value 必须大于 0")
    return value
```

不能把必须一直生效的权限、金额、路径或输入验证只写成 assert。

本篇练习中用 assert 检查结果，是验证示例的方式，不是建议把所有产品验证写成断言。

## 第十章：异常对象里有什么？不仅仅是一段字符串

### 10.1 `str()`、`repr()` 与 `args`

```python
error = ValueError("天数不正确")
print(str(error))
print(repr(error))
print(error.args)
```

输出：

```text
天数不正确
ValueError('天数不正确')
('天数不正确',)
```

`str()` 通常面向可读消息。

`repr()` 展示对象形式，有助于调试。

`args` 是构造异常时保存的参数元组，但一些内建异常有专门构造规则，不应假设每一种异常都只是任意字符串的薄壳。

### 10.2 为什么有的异常还有 filename、errno？

例如 `OSError` 家族可以提供系统错误编号和相关文件名。

调用者不必从英文消息里硬拆路径。

```python
error = FileNotFoundError(2, "练习目标不存在", "practice.txt")
print(error.errno)
print(error.filename)
```

输出：

```text
2
practice.txt
```

这是手动创建的演示对象，真实错误编号和消息来自操作系统。

如果是实际文件错误，可以根据明确属性决定如何提示，而不是只看 `str(error)` 有没有某个英文词。

### 10.3 所有异常都有同样属性吗？

没有。

`JSONDecodeError` 有行列等位置属性。

`UnicodeDecodeError` 有编码、起止范围等信息。

自定义异常可以有业务字段。

因此捕获具体异常类的一个好处，是你知道该对象承诺提供什么信息。

写通用日志可以使用类型和消息；写具体恢复逻辑则应按具体接口文档使用结构化属性。

### 10.4 except 变量离开处理块以后还能用吗？

Python 会在 except 结束时清除绑定的异常目标，避免异常、回溯与局部变量形成长期引用环。

```python
try:
    int("bad")
except ValueError as error:
    message = str(error)

print(message.startswith("invalid literal"))
```

在常见 CPython 英文消息环境输出：

```text
True
```

这里保存的是消息字符串，没有在块外继续访问 `error`。

不要先在外层也用同名 error 保存重要数据，再以为 except 后那个绑定仍然原样存在。

### 10.5 我真的想保存异常对象呢？

可以显式保存到另一个变量或数据结构。

但要注意异常回溯可能间接引用栈帧及其中对象，让内存保留时间变长。

批处理报告很多时候只需要文件名、异常类型和脱敏消息，不必永久保存所有异常对象。

如果确实需要在后台任务结果中保留异常，应该设计释放时机，而不是把每次失败无限追加进全局列表。

## 第十一章：自定义异常，什么时候需要自己的名字？

### 11.1 内建类型能用，为什么还要新建类？

假设你的笔记工具有三类失败。

配置文件不符合工具规则。

某条笔记记录无效。

用户修改的版本已经过期。

它们都可以勉强使用 ValueError，但上层很难区分。

给这些业务概念明确异常类型，调用者就不必靠消息字符串判断。

### 11.2 最小定义

```python
class NotesError(Exception):
    """笔记工具向调用者公开的可预期错误基类。"""


class NotesConfigError(NotesError):
    """工具配置不符合约定。"""


class NoteValidationError(NotesError):
    """一条笔记的数据不合法。"""


class NoteConflictError(NotesError):
    """记录状态与调用者预期不一致。"""
```

继承 `Exception`，就能正常通过 raise 与 except 工作。

类体只有文档字符串也是合法定义，不一定非要写 pass。

### 11.3 为什么先有 NotesError？

命令行入口可以捕获 `NotesError`，统一显示所有可预期业务失败。

某个编辑界面又可以单独捕获 `NoteConflictError`，提示重新加载记录。

一套小层级同时支持总入口与具体分支。

```python
try:
    raise NoteConflictError("这条笔记已被其他操作更新")
except NoteConflictError:
    print("请重新加载后再修改")
except NotesError:
    print("笔记操作没有完成")
```

输出：

```text
请重新加载后再修改
```

前面的具体分支比后面的基类分支先匹配。

### 11.4 是不是每个 if 都要定义一个异常类？

不需要。

如果调用者对“标题为空”“标题太长”的处理方式完全相同，一个带字段信息的验证异常就足够。

只有调用者需要不同分支、重试政策或恢复行为时，区分类别才更有价值。

一百个只有名字区别、没有实际处理差异的异常类，会增加理解成本。

先根据业务边界设计，不要把所有英文名词都变成类。

### 11.5 为什么一般不继承 BaseException？

你定义的是普通业务错误，希望一般的 `except Exception` 能处理它。

直接继承 BaseException 会让它和退出、中断那类控制信号靠得更近，容易破坏调用者预期。

除非你正在设计非常特殊的解释器级机制，否则从 Exception 或合适的现有子类继承。[自定义异常官方建议](https://docs.python.org/3.14/tutorial/errors.html#user-defined-exceptions)

### 11.6 可以同时继承多个内建异常类吗？

不建议把它作为默认设计。

某些内建异常有特殊的底层内存布局，多重继承可能产生冲突，而且未来版本也可能变化。

更清楚的方式是设计自己的一个基类层级，再用属性表达不同细节。

不要为了让某个异常被两种大类同时捕获，就随意组合所有内建异常类。

## 第十二章：给自定义异常增加字段，错误报告才不需要拆字符串

### 12.1 把“哪一行、哪个字段”保存下来

导入 CSV 时，你希望知道第几条记录的哪个字段有问题。

可以这样定义：

```python
class RecordValidationError(Exception):
    def __init__(self, record_number, field, reason):
        self.record_number = record_number
        self.field = field
        self.reason = reason
        super().__init__(f"第 {record_number} 条记录的 {field}：{reason}")
```

`self.record_number` 是结构化数据。

`self.field` 是字段名。

`self.reason` 是可读原因。

`super().__init__(...)` 初始化异常基类，使普通 `str(error)` 和 `error.args` 也具有有意义的内容。

### 12.2 创建与捕获

```python
try:
    raise RecordValidationError(3, "title", "不能为空白")
except RecordValidationError as error:
    print(error)
    print(error.record_number)
    print(error.field)
```

输出：

```text
第 3 条记录的 title：不能为空白
3
title
```

上层需要定位表格行时，直接用 `record_number`。

需要显示提示时，用 `str(error)`。

不必从“第”和“条”之间切字符串。

### 12.3 属性保存原始值会不会更方便？

有时有用，但要考虑数据敏感性和大小。

一个错误对象如果保存整个上传文件、完整配置或大型 DataFrame，可能扩大内存与泄露范围。

很多时候只保存位置、字段、规则和短小脱敏摘要就够了。

不要把“方便调试”理解成把所有输入永久挂在异常上。

### 12.4 自定义异常必须用 dataclass 吗？

不必须。

普通类已经清楚表达构造过程和基类初始化。

dataclass 与 Exception 结合时，要注意自动生成的 `__init__` 是否按你期望初始化基类参数、字符串表现和序列化行为。

初学先写显式的小构造函数，更容易看懂每个字段从哪里来。

### 12.5 自定义异常能直接跨进程传递吗？

不能把“在本进程能 raise”自动等同于“任意序列化方式都能恢复”。

某些进程工具使用 pickle，重建异常时会使用构造参数；你保存的 args 与自定义 `__init__` 签名不匹配，可能造成恢复失败。

如果确实要跨进程传异常，应该为目标通信机制设计并测试，或者传递一个普通的结构化错误结果。

本篇的 `RecordValidationError` 主要用于同一进程内的批处理，不声称已经提供通用跨进程序列化协议。

## 第十三章：异常链——上层换一种说法，底层原因还能留下吗？

### 13.1 自动上下文是什么？

如果在处理一个异常时又抛另一个异常，Python 默认会记录它们之间的上下文关系。

```python
try:
    try:
        int("bad")
    except ValueError:
        raise RuntimeError("无法建立学习计划")
except RuntimeError as error:
    print(type(error.__context__).__name__)
```

输出：

```text
ValueError
```

默认回溯展示通常会同时显示两次失败，说明处理前一个异常时出现了后一个异常。

### 13.2 显式原因用 from

```python
class PlanInputError(Exception):
    pass


def parse_plan_days(text):
    try:
        return int(text)
    except ValueError as error:
        raise PlanInputError("学习计划的天数必须是整数") from error


try:
    parse_plan_days("bad")
except PlanInputError as error:
    print(error)
    print(type(error.__cause__).__name__)
```

输出：

```text
学习计划的天数必须是整数
ValueError
```

上层获得工具领域里的错误。

调试者仍能看到 `int()` 的原始 ValueError。

这适合在抽象边界转换错误，而不是每经过一层函数就强行包一层。

### 13.3 到底什么时候应该转换？

当你的函数对外承诺的是“加载学习配置”，而不想让调用者依赖内部究竟用了 JSON、TOML 还是数据库时，转换成配置异常有意义。

但你仍应保留 from 原因，支持诊断。

如果调用者本来就在使用一个透明的底层包装函数，没有额外领域含义，原样传播可能更清楚。

异常层级要帮助调用者理解接口，不能只是让回溯变得更长。

### 13.4 `from None` 是删除原始异常吗？

它主要抑制默认回溯展示中的原始上下文。

上下文对象仍可能留在 `__context__`，并不是彻底擦除。

```python
try:
    try:
        int("bad")
    except ValueError:
        raise PlanInputError("天数格式错误") from None
except PlanInputError as error:
    print(error.__cause__ is None)
    print(type(error.__context__).__name__)
    print(error.__suppress_context__)
```

输出：

```text
True
ValueError
True
```

如果你的目的是给普通用户简短提示，通常可以在最终显示层只输出消息，不一定需要在底层到处使用 from None。

保留诊断证据往往更有价值。

### 13.5 异常链与多次独立错误一样吗？

不一样。

异常链表达“这次更高层失败的原因或上下文”。

同时验证五个独立字段，得到五个错误，不应该硬串成一个假因果链。

独立错误可以用列表、结构化报告或 3.11 的 ExceptionGroup 表示，后面会分别讲。

## 第十四章：给异常加注释，不必每层重新定义一个类

### 14.1 Python 3.11 的 add_note

有时你不想换掉异常类型，只想补充当前处理哪个文件或哪个任务。

可以使用 `add_note()`。

```python
try:
    try:
        int("bad")
    except ValueError as error:
        error.add_note("处理文件：practice/plan.json")
        error.add_note("处理字段：days")
        raise
except ValueError as error:
    print(error.__notes__)
```

输出：

```text
['处理文件：practice/plan.json', '处理字段：days']
```

默认回溯展示会在异常消息之后显示这些注释。

原异常仍是 ValueError。

### 14.2 note 是字符串还是任意对象？

要求字符串。

不要直接放 Path 对象、字典或整数。

需要时先转换成适合公开的文字。

它是面向人的补充说明，不是结构化字段的替代品。

如果程序要判断字段名，仍应保存在明确属性或结构化结果中。

### 14.3 哪些内容适合写 note？

当前批次名称。

已脱敏的相对文件路径。

正在执行的阶段，例如“解析配置后，尚未发布报告”。

可能的检查方向，例如“此文件必须是 UTF-8”。

不适合把整个秘密配置或几兆原始响应塞进去。

### 14.4 为什么不用每层都 raise RuntimeError？

如果错误类别没有变化，只是补充位置，重新包装会增加层数，让调用者更难按原类型捕获。

add_note 保留类型和原回溯，同时添上操作上下文。

如果抽象确实改变，例如底层解码失败变成工具配置错误，则明确的异常转换仍然合理。

## 第十五章：读取一份学习计划配置，分清四种不同失败

### 15.1 先说输入契约

配置文件是一份 UTF-8 JSON 对象。

只允许 `title` 和 `days` 两个字段。

title 必须是去掉两端空白后非空的字符串。

days 必须是真正整数，范围 `1` 到 `365`，不接受 bool。

文件不存在、读不了、文本解码失败、JSON 语法错误、字段值错误，都不能全部显示为“天数不对”。

### 15.2 先定义本工具的配置异常

```python
from pathlib import Path
import json


class PlanConfigError(Exception):
    """学习计划配置无法按约定加载。"""


class PlanConfigReadError(PlanConfigError):
    pass


class PlanConfigFormatError(PlanConfigError):
    pass


class PlanConfigValidationError(PlanConfigError):
    pass
```

读取失败、格式失败、字段验证失败属于同一配置错误体系。

如果上层只需要一个友好失败出口，捕获 PlanConfigError。

如果界面要高亮字段错误，则可以单独处理验证类型。

### 15.3 加载函数

```python
def load_plan_config(path):
    path = Path(path)
    try:
        text = path.read_text(encoding="utf-8")
    except UnicodeError as error:
        raise PlanConfigFormatError("配置必须使用 UTF-8 编码") from error
    except OSError as error:
        raise PlanConfigReadError(f"无法读取配置文件：{path.name}") from error

    try:
        payload = json.loads(text)
    except json.JSONDecodeError as error:
        raise PlanConfigFormatError(
            f"JSON 语法错误：第 {error.lineno} 行，第 {error.colno} 列"
        ) from error

    if not isinstance(payload, dict):
        raise PlanConfigValidationError("配置顶层必须是对象")
    expected = {"title", "days"}
    missing = expected - payload.keys()
    extra = payload.keys() - expected
    if missing:
        raise PlanConfigValidationError(f"缺少字段：{', '.join(sorted(missing))}")
    if extra:
        raise PlanConfigValidationError(f"未知字段：{', '.join(sorted(extra))}")

    title = payload["title"]
    days = payload["days"]
    if not isinstance(title, str) or not title.strip():
        raise PlanConfigValidationError("title 必须是非空白字符串")
    if type(days) is not int:
        raise PlanConfigValidationError("days 必须是整数，不能是布尔值")
    if not 1 <= days <= 365:
        raise PlanConfigValidationError("days 必须在 1 到 365 之间")
    return {"title": title.strip(), "days": days}
```

### 15.4 为什么不把整个函数放进一个 except Exception？

因为读取、解析和规则验证是三种不同阶段。

读不到文件时，JSON 解析根本没发生。

JSON 语法通过时，字段仍可能无效。

函数内部意外出现 NameError，则通常是程序缺陷，不应伪装成用户的配置问题。

### 15.5 在临时目录试一份正确配置

依赖上面的类与函数定义。

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-config-") as directory:
    path = Path(directory) / "plan.json"
    path.write_text('{"title": "  Python 学习  ", "days": 30}', encoding="utf-8")
    print(load_plan_config(path))
```

输出：

```text
{'title': 'Python 学习', 'days': 30}
```

返回的是经过规则确认和标题规范化的新字典。

它不是原始输入一定无条件照搬。

### 15.6 如果 days 是 true 呢？

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-config-") as directory:
    path = Path(directory) / "plan.json"
    path.write_text('{"title": "Python", "days": true}', encoding="utf-8")
    try:
        load_plan_config(path)
    except PlanConfigValidationError as error:
        print(error)
```

输出：

```text
days 必须是整数，不能是布尔值
```

JSON 的 true 进入 Python 以后是 bool。

这也是本篇没有直接用 `isinstance(days, int)` 的原因。

### 15.7 如果输入文件特别大呢？

当前加载函数把整份文件读入内存，没有声称为任意尺寸文件设计。

给真实工具使用时，可以增加字节上限，在读取阶段用有界二进制读取再解码。

不能先整体 `read_text()`，最后检查字符串长度，再说已经防止大文件占用内存。

异常处理告诉你怎么报告超限，但资源上限还要在实际读取方式中落实。

## 第十六章：文件资源什么时候释放？不要在每个 except 里重复 close

### 16.1 最朴素的 finally 写法

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-file-") as directory:
    path = Path(directory) / "data.txt"
    path.write_text("hello", encoding="utf-8")
    source = path.open("r", encoding="utf-8")
    try:
        content = source.read()
        print(content)
    finally:
        source.close()
    print(source.closed)
```

输出：

```text
hello
True
```

文件成功打开以后，读取失败也会经过 finally 安排关闭。

注意打开操作发生在 try 之前：如果打开本身失败，source 根本没得到文件对象，就不应尝试关闭一个不存在的资源。

### 16.2 更常用的 with 写法

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-file-") as directory:
    path = Path(directory) / "data.txt"
    path.write_text("hello", encoding="utf-8")
    with path.open("r", encoding="utf-8") as source:
        print(source.read())
    print(source.closed)
```

输出相同：

```text
hello
True
```

文件对象的上下文协议负责正常和异常离开时关闭资源。

这比在每个 except 后都复制 source.close 更不容易漏掉分支。

### 16.3 with 会自动吞掉所有文件异常吗？

普通文件上下文不会这样做。

它关闭文件，但读取失败仍会继续向外传播。

清理和恢复是两种职责。

你可以在外层捕获文件错误，决定怎样提示用户。

不能因为文件已经关闭，就把本次读操作当成成功。

### 16.4 多个文件怎么一起管理？

固定数量可以在一个 with 语句中写多个上下文管理器。

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-two-files-") as directory:
    root = Path(directory)
    source_path = root / "input.txt"
    target_path = root / "output.txt"
    source_path.write_text("hello", encoding="utf-8")
    with source_path.open(encoding="utf-8") as source, target_path.open("x", encoding="utf-8") as target:
        target.write(source.read().upper())
    print(target_path.read_text(encoding="utf-8"))
```

输出：

```text
HELLO
```

如果第二个打开失败，已经进入的第一个上下文也会得到退出处理。

如果资源数量运行时才确定，可以考虑 ExitStack；后面会把它和失败清理联系起来。


## 第十七章：异常不会自动撤销已经做过的事

### 17.1 列表追加一半，后来失败

```python
results = []
try:
    for text in ["10", "20", "bad", "40"]:
        results.append(int(text))
except ValueError:
    print("转换中途失败")

print(results)
```

输出：

```text
转换中途失败
[10, 20]
```

异常停止了当前正常流程，没有把之前的 append 撤销。

如果这份列表已经被别的代码引用，别的代码也能看到这部分修改。

### 17.2 想全部成功后才更新原状态呢？

先在局部准备新结果，全部成功后再发布。

```python
original = [1, 2]
texts = ["10", "20", "bad"]
try:
    prepared = [int(text) for text in texts]
except ValueError:
    print("保留原结果")
else:
    original[:] = prepared

print(original)
```

输出：

```text
保留原结果
[1, 2]
```

`prepared` 的构造失败，发布语句没有执行。

这是一种很有用的组织方式：准备与验证在前，改变共享状态在后。

它不是适用于所有对象的自动事务，但能减少很多半成品状态。

### 17.3 文件写了一半，也不会自动复原

如果你用 `w` 打开已有报告，旧内容可能在写入之前就已经被截断。

后面再出异常，except 不会帮你找回原来字节。

所以要保护旧报告，不能只在写入附近加一个 try。

需要改变实际写入与发布流程。

### 17.4 一个只替换完整结果的练习函数

下面函数允许覆盖指定练习报告。

先在同目录创建临时文件，完整写入并关闭，然后使用 `os.replace()` 发布。

```python
from pathlib import Path
import os
import tempfile


def replace_practice_report(path, lines):
    path = Path(path)
    temporary_path = None
    try:
        with tempfile.NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            newline="\n",
            dir=path.parent,
            prefix=f".{path.name}.",
            suffix=".tmp",
            delete=False,
        ) as target:
            temporary_path = Path(target.name)
            for line in lines:
                if not isinstance(line, str):
                    raise TypeError("报告每一行必须是字符串")
                target.write(line)
                target.write("\n")
        os.replace(temporary_path, path)
        temporary_path = None
    finally:
        if temporary_path is not None:
            try:
                temporary_path.unlink()
            except FileNotFoundError:
                pass
```

这里没有自动创建父目录，要求调用者事先明确准备目标位置。

`os.replace()` 允许替换已经存在的目标，不能把本函数拿去指向不允许覆盖的文件。

### 17.5 用失败输入检查旧报告是否保住

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-publish-") as directory:
    path = Path(directory) / "report.txt"
    path.write_text("OLD\n", encoding="utf-8")
    try:
        replace_practice_report(path, ["first", 123, "third"])
    except TypeError:
        print("生成失败")
    print(path.read_text(encoding="utf-8"), end="")
    print(len(list(Path(directory).glob("*.tmp"))))
```

输出：

```text
生成失败
OLD
0
```

目标文件没被提前截断。

临时文件在正常异常展开中得到清理。

### 17.6 这个函数保证断电后一定保留新报告吗？

没有作这种承诺。

同文件系统替换提供的可见性与断电后的持久性是不同问题；更严格持久化还涉及刷新、同步与平台文件系统规则。

临时文件清理本身也可能因为权限或系统问题失败，并不是一定不会再抛异常。

本例验证的是“准备阶段抛普通 Python 异常时，旧目标不被提前破坏”，结论应保持在这个范围内。

### 17.7 数据库、网络与打印同样适用吗？

数据库可以用它自己的事务把适用修改组合起来。

已经发送的邮件、外部 HTTP 请求和终端打印，不会因本地异常自动撤销。

跨系统任务要考虑幂等、补偿、任务记录与重试边界。

先问“哪些效果已经发生”，比只问“异常有没有捕获”更接近真实可靠性。

## 第十八章：批处理遇到坏数据，是立刻停止还是收集后继续？

### 18.1 这首先是业务选择

如果一份账单必须全部正确，发现一条坏数据就停止并保留原状，可能最合理。

如果你在扫描几百份独立笔记，某一份编码错了，不一定要放弃其他文件。

两种策略都能使用异常，但最终结果状态不同。

不要在实现中随手加 continue，替用户悄悄决定“允许部分成功”。

### 18.2 一个明确允许部分成功的解析函数

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ParseIssue:
    record_number: int
    message: str


def parse_independent_days(texts):
    values = []
    issues = []
    for record_number, text in enumerate(texts, 1):
        try:
            value = int(text)
        except (ValueError, TypeError):
            issues.append(ParseIssue(record_number, "必须能转换成整数"))
            continue
        if not 1 <= value <= 365:
            issues.append(ParseIssue(record_number, "必须在 1 到 365 之间"))
            continue
        values.append(value)
    return values, issues
```

这里每条记录独立，函数明确返回成功值与问题列表。

没有假装整批完全成功。

### 18.3 查看返回结果

```python
values, issues = parse_independent_days(["10", "bad", "0", "30"])
print(values)
for issue in issues:
    print(issue.record_number, issue.message)
```

输出：

```text
[10, 30]
2 必须能转换成整数
3 必须在 1 到 365 之间
```

成功结果和失败记录都保留下来，调用者可以生成汇总。

如果真实输入契约只允许字符串，应在转换前拒绝其他类型；当前函数特意允许 int() 接受的输入，并用返回值表达其政策。

### 18.4 为什么这里没有把异常对象放进 issues？

普通报告只需要位置和可读原因。

它不需要长期保存每条回溯。

同时，面向用户的规则消息比 `int()` 原始英文错误更稳定。

如果开发者还需要底层诊断，可以在调试模式记录，或者给内部问题对象增加独立诊断字段。

### 18.5 如果问题特别多，列表会不会无限增长？

会有这个风险。

大文件导入可以只收集前若干个详细问题，同时统计总问题数，或按流写入独立报告。

需要让用户知道“这里只展示前 100 条”，不能截断后说“总共只有 100 条问题”。

错误数据本身也能造成资源压力，所以异常处理路径同样要有规模意识。

### 18.6 哪些异常不适合在每条记录内忽略？

比如程序代码出现 AttributeError，磁盘无法继续读取，数据库连接失效，或者用户要求取消。

这些通常不是某一条记录自身的问题。

把所有异常都变成一条 issue 然后继续，可能制造几万条重复假错误。

记录级恢复只捕获你真正预期并能够隔离的失败类别。

## 第十九章：命令行入口应该怎样处理可预期失败？

### 19.1 底层函数不必直接退出整个程序

`load_plan_config()` 负责加载配置，失败抛 PlanConfigError。

它不应该在内部随便 `sys.exit()`，因为将来可能被测试、桌面界面或别的模块调用。

最外层命令行入口才决定消息显示与退出状态。

### 19.2 一个可测试的 main

依赖第十五章的配置类和加载函数。

```python
import argparse
from pathlib import Path
import sys


def plan_main(argv=None):
    parser = argparse.ArgumentParser(description="读取一份练习学习计划")
    parser.add_argument("config", type=Path)
    args = parser.parse_args(argv)

    try:
        plan = load_plan_config(args.config)
    except PlanConfigError as error:
        print(f"无法生成计划：{error}", file=sys.stderr)
        return 2

    print(f"{plan['title']}：{plan['days']} 天")
    return 0
```

脚本入口可以另写：

```python
if __name__ == "__main__":
    raise SystemExit(plan_main())
```

这个入口片段需要与函数放在同一文件中。

### 19.3 为什么 main 返回整数，而不是自己到处 exit？

返回值容易测试。

你可以传入一个参数列表，检查返回码与输出，不必真的关闭测试进程。

最外层的一次 SystemExit 把结果交给操作系统。

这是把业务调用与进程生命周期分开的一种简单方式。

### 19.4 如果输入参数本身不合法呢？

argparse 有自己的 usage 提示和 SystemExit 行为。

`--help` 通常是正常帮助退出，不应当作程序缺陷记录成严重错误。

因此不要在最外面使用裸 except 把所有 SystemExit 都拦下来显示“运行崩溃”。

配置内容错误与命令行参数语法错误，可以有不同来源但都向用户给出明确说明。

### 19.5 程序意外出现 NameError 要怎么显示？

开发过程中通常让回溯暴露，尽快修复。

正式工具如果需要统一崩溃报告，可以在最外层捕获 Exception，记录完整诊断并返回失败状态。

但提示要说明是意外程序错误，而不是谎称用户配置有问题。

也不要捕获以后返回 `0`，那会让自动化调用者误以为任务成功。

### 19.6 错误打印到 stdout 还是 stderr？

通常把正常机器结果放 stdout，把诊断放 stderr。

这样调用者可以把结果重定向成文件，而不会混进一行“读取失败”。

日志和退出码各有用途：消息供人理解，状态码供调用者判断。

不要只打印失败文本却仍用成功码退出。

## 第二十章：记录异常时，怎样既有消息，又保留回溯？

### 20.1 print(error) 只是一种很薄的信息

它通常只显示异常字符串。

函数调用路径、源代码位置和异常链不一定一起显示。

开发者要排查问题时，常常还需要回溯。

### 20.2 在处理块里用 logging.exception

```python
import io
import logging

stream = io.StringIO()
logger = logging.getLogger("notes.exception.practice")
logger.setLevel(logging.ERROR)
logger.propagate = False
handler = logging.StreamHandler(stream)
handler.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))
logger.addHandler(handler)

try:
    try:
        int("bad")
    except ValueError:
        logger.exception("计划天数解析失败")
    log_text = stream.getvalue()
    print("ValueError" in log_text)
    print("Traceback" in log_text)
finally:
    logger.removeHandler(handler)
    handler.close()
```

输出：

```text
True
True
```

这里用内存流观察结果，避免在练习中生成长期日志文件。

`logger.exception()` 通常在 except 中调用，会按当前异常信息记录回溯。

### 20.3 `logger.error("失败")` 呢？

它只记录你给出的普通消息，不会因为处在 except 中就自动带上回溯。

可以使用 `exc_info=True` 明确要求异常信息。

`logger.exception()` 是这个常见用途的便捷接口。

不要为了得到回溯，把异常字符串拼很多次，它仍然不是完整调用栈。

### 20.4 同一个异常每层都记录一次，有必要吗？

往往会得到三四份重复回溯。

底层记录一次又重新抛，中层记录一次再包装，入口又记录一次，日志很快变得难读。

通常选择拥有足够上下文和最终处置责任的一层记录。

其他层可以补 note、转换有意义的异常，或者保持传播。

记录策略也属于接口设计，不是每见到 except 就必须打印。

### 20.5 我想把回溯保存成字符串呢？

```python
import traceback

try:
    int("bad")
except ValueError:
    detail = traceback.format_exc()

print("ValueError" in detail)
```

输出：

```text
True
```

`format_exc()` 适合当前处理中的异常。

如果你已经显式保存一个异常对象，可以使用 traceback 的相应格式化接口，而不是等到离开处理块以后仍假设当前异常存在。

同样注意消息中的秘密数据；格式化不是脱敏。

## 第二十一章：重试不是 except 后面加一个 continue 就结束

### 21.1 哪些失败可能值得重试？

短暂锁竞争、暂时不可用的连接、可恢复的服务状态，有时值得重试。

格式永远错误的整数文本、缺少必填字段、程序自己的 NameError，重复一百次也不会自然变好。

先分类，再决定是否重试。

### 21.2 重试必须有停止条件

最大尝试次数。

总截止时间。

每次等待与退避。

允许重做的操作范围。

最后失败如何保留原因。

这些都属于重试政策，不能只写 `while True` 然后希望某次碰巧成功。

### 21.3 用确定的本地函数模拟短暂失败

```python
class TemporaryPracticeError(Exception):
    pass


def retry_practice(operation, *, attempts=3, delay=0.1, sleep=None):
    import time
    import math

    if type(attempts) is not int or attempts < 1:
        raise ValueError("attempts 必须是正整数")
    if isinstance(delay, bool) or not isinstance(delay, (int, float)):
        raise ValueError("delay 必须是有限非负数")
    try:
        valid_delay = math.isfinite(delay) and delay >= 0
    except OverflowError:
        valid_delay = False
    if not valid_delay:
        raise ValueError("delay 必须是有限非负数")
    if sleep is None:
        sleep = time.sleep

    for attempt in range(1, attempts + 1):
        try:
            return operation()
        except TemporaryPracticeError as error:
            if attempt == attempts:
                error.add_note(f"已经尝试 {attempts} 次，仍未完成")
                raise
            sleep(delay)
```

这个函数只捕获自己明确认定可重试的 TemporaryPracticeError。

它不捕获所有 Exception。

这里通过 `math.isfinite()` 拒绝 NaN 与无穷大，也把超出浮点转换范围的大整数视为不适合等待的值。

有限延迟仍可能很长，真实工具还可以按产品要求设置上限；本例的调用始终使用短延迟。

### 21.4 为什么允许注入 sleep？

测试时不必真的等几秒。

你可以传入一个只记录等待时间的函数，验证退避政策。

下面依赖前面的类和重试函数：

```python
state = {"calls": 0}
waits = []


def operation():
    state["calls"] += 1
    if state["calls"] < 3:
        raise TemporaryPracticeError("模拟短暂不可用")
    return "done"


result = retry_practice(operation, attempts=3, delay=0.1, sleep=waits.append)
print(result)
print(state["calls"])
print(waits)
```

输出：

```text
done
3
[0.1, 0.1]
```

第三次成功后没有多睡一次。

这比让示例真的随机失败，更容易学习与验证。

### 21.5 最后一次失败时为什么裸 raise？

它保留最后一次失败的真实类型和回溯。

add_note 补充这次已经尝试几次。

如果调用者想区分“失败类别”，仍然可以捕获 TemporaryPracticeError。

你也可以设计 RetryExhaustedError，但要说明它是一个新的抽象边界，并通过 from 保留最后原因。

### 21.6 操作已经完成，回复丢了，再重试怎么办？

这是很重要的边界。

比如你发出一次写入请求，服务已经成功保存，但返回响应时连接断了。

父侧只看到超时，不知道远端是否完成。

再次执行可能产生重复记录。

所以重试写操作前要考虑幂等键、唯一约束或查询任务状态。

异常是你看到的失败，不必然等同于对方完全没有做事。

### 21.7 多步事务怎么重试？

通常需要把整体业务操作作为重试单位，而不是只重跑最后一条语句。

每次尝试前确保上次的事务、连接和临时状态已经按约定结束。

否则第二次尝试可能建立在第一次的半成品上。

“最多三次”只有在一次尝试的边界清楚时才有意义。

### 21.8 指数退避与随机抖动是必须的吗？

对于很多客户端竞争同一服务，它们有助于避免所有调用者同时再次请求。

但普通本地练习不需要一开始就引入复杂策略。

先明确错误可重试、次数有限、动作可重复，再按真实并发需求选择延迟与总预算。

算法名字不能代替业务安全性。


## 第二十二章：生成器里的异常，为什么创建时没报，遍历时才报？

### 22.1 生成器函数的执行是延迟的

```python
def parsed_numbers(texts):
    for text in texts:
        yield int(text)


numbers = parsed_numbers(["10", "bad", "30"])
print("生成器已创建")
print(next(numbers))
try:
    print(next(numbers))
except ValueError:
    print("第二次取值才发生转换错误")
```

输出：

```text
生成器已创建
10
第二次取值才发生转换错误
```

调用生成器函数时，通常只是得到生成器对象。

真正执行函数体发生在推进迭代时。

所以只把“创建生成器”包在 try 里，不一定能捕获后面遍历中发生的错误。

### 22.2 捕获位置要围住消费过程

```python
try:
    values = list(parsed_numbers(["10", "bad", "30"]))
except ValueError:
    print("整次消费没有完成")
```

输出：

```text
整次消费没有完成
```

`list()` 推进迭代，转换错误就在这个过程中向调用者传播。

如果你想记录坏行后继续，需要生成器自己按记录边界捕获，而不是期待外层 except 执行后自动恢复到失败的 yield 前面。

### 22.3 出现未处理异常后，生成器还能接着输出第三条吗？

通常这个生成器已经因异常退出，不能把它当作暂停在第二条等待你修好。

```python
numbers = parsed_numbers(["10", "bad", "30"])
print(next(numbers))
try:
    next(numbers)
except ValueError:
    print("生成器失败退出")
print(list(numbers))
```

输出：

```text
10
生成器失败退出
[]
```

如果业务希望独立记录继续，应在生成器内部明确恢复策略，或返回“成功/失败记录”对象。

### 22.4 StopIteration 是不是也要当普通错误提示？

它常用来表示迭代自然结束。

for 循环会按迭代协议处理结束，不把它打印成错误。

使用 `next(iterator, default)` 还可以给结束提供默认值。

```python
iterator = iter([7])
print(next(iterator, None))
print(next(iterator, None))
```

输出：

```text
7
None
```

不要把所有异常名都翻译成“程序坏了”，有些是协议控制信号。

### 22.5 为什么不要在生成器里随便 raise StopIteration？

生成器应通常通过 return 或自然走到末尾结束。

在现代 Python 中，生成器内部意外逃出的 StopIteration 会转成 RuntimeError，以避免某些内部错误悄悄表现成正常迭代结束。

```python
def broken_generator():
    yield 1
    raise StopIteration("错误地手动结束")


try:
    list(broken_generator())
except RuntimeError:
    print("生成器内部的 StopIteration 被视为错误")
```

输出：

```text
生成器内部的 StopIteration 被视为错误
```

读迭代相关代码时，区分“调用 next 得到正常结束”和“生成器体内错误抛 StopIteration”。

### 22.6 生成器持有文件时，什么时候关闭？

如果生成器内部用 with 打开文件，资源会在生成器结束或关闭时得到退出处理。

但调用者中途停止遍历，不应把清理时机寄托在垃圾回收碰巧发生。

可以使用明确的上下文或 `contextlib.closing()` 安排关闭支持 close 的生成器。

资源生命周期要覆盖整个消费阶段，而不只是对象创建那一行。

## 第二十三章：异步任务失败和取消，为什么不能吞掉 CancelledError？

### 23.1 async 函数也会抛普通异常

```python
import asyncio


async def parse_async_days(text):
    await asyncio.sleep(0)
    return int(text)


async def async_example():
    try:
        await parse_async_days("bad")
    except ValueError:
        print("await 处收到了转换异常")


asyncio.run(async_example())
```

在普通脚本中输出：

```text
await 处收到了转换异常
```

如果你在已有事件循环的 Notebook 环境，不要再嵌套 asyncio.run，应按该环境支持方式 await 顶层协程。

异步没有让错误消失，只是执行与等待的组织方式改变了。

### 23.2 取消不是普通输入错误

现代 asyncio 的 `CancelledError` 继承 BaseException。

任务取消时，协程应该及时退出并安排必要清理。

如果为了“不中断”把它吞掉，TaskGroup、超时上下文和上层取消协议可能无法按预期工作。

```python
import asyncio


async def cancellable_worker():
    try:
        await asyncio.sleep(10)
    finally:
        print("任务开始清理")


async def cancellation_example():
    task = asyncio.create_task(cancellable_worker())
    await asyncio.sleep(0)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("调用者确认取消")


asyncio.run(cancellation_example())
```

输出：

```text
任务开始清理
调用者确认取消
```

### 23.3 为什么先 sleep(0) 再 cancel？

它让事件循环有机会推进 worker，使其进入 try/finally 范围。

如果任务在协程体开始前就被取消，不能指望尚未进入的 finally 已经承担资源清理。

资源应在真正取得后立即安排清理，不能根据“我创建了 task”就假设协程已运行到某一行。

### 23.4 如果确实要捕获 CancelledError 呢？

可以为了补充记录或清理捕获，但通常应在完成必要工作后重新 raise。

不要改成返回一个普通成功值，让上层以为任务已经完成。

取消、失败与成功应保持清楚的状态差异。

### 23.5 create_task 以后不 await，会怎样？

任务可能在后台失败，而创建它的函数没有在原位置收到异常。

事件循环可能记录“任务异常未取回”等诊断。

这不是你已经处理了失败，而是你没有给任务安排结果责任。

真实程序要明确谁 await、谁收集异常、谁取消剩余任务。

如果任务是同一组工作，可以考虑 3.11 的 TaskGroup，让任务生命周期更集中。

### 23.6 异步超时与同步 timeout 是同一个异常吗？

不同 API 有自己的约定。

例如 asyncio 的超时接口、subprocess.TimeoutExpired 和第三方 HTTP 客户端的超时异常，并不是同一个通用类。

不要只看消息里都有 timeout，就用一个宽泛 except 随意处理。

先查实际接口，尤其注意取消是否被用作内部实现机制。[asyncio 取消说明](https://docs.python.org/3.14/library/asyncio-task.html#task-cancellation)

## 第二十四章：ExceptionGroup 与 except*，同时有几份独立失败怎么办？

### 24.1 普通异常链不适合假装多份独立错误

假设你分别校验标题、日期和天数，发现其中两项不合法。

它们不是“日期错误导致标题错误”，而是同一批工作的两个独立问题。

可以用普通问题列表返回。

如果调用协议希望整体仍通过异常传播，可以使用 Python 3.11 的 ExceptionGroup。

### 24.2 最小分组

```python
errors = [
    ValueError("title 不能为空"),
    TypeError("days 必须是整数"),
]
group = ExceptionGroup("计划字段验证失败", errors)
print(group.message)
print(len(group.exceptions))
```

输出：

```text
计划字段验证失败
2
```

ExceptionGroup 包含一组异常。

它自己仍是可抛出的对象，拥有整组的说明。

### 24.3 普通 except ExceptionGroup 会怎样？

它捕获整组，你再自己检查内部元素。

```python
try:
    raise ExceptionGroup("验证失败", [ValueError("bad title"), TypeError("bad days")])
except ExceptionGroup as error:
    print(len(error.exceptions))
```

输出：

```text
2
```

这并不会自动按照子异常类型拆成两次普通 except。

### 24.4 except* 才按子异常类别匹配

```python
try:
    raise ExceptionGroup(
        "验证失败",
        [ValueError("bad title"), TypeError("bad days")],
    )
except* ValueError as errors:
    print("值错误组：", len(errors.exceptions))
except* TypeError as errors:
    print("类型错误组：", len(errors.exceptions))
```

输出：

```text
值错误组： 1
类型错误组： 1
```

`errors` 仍然是一个匹配出来的异常组，不是某一条叶子异常。

对于嵌套组，还会保留相应组结构，因此 `len(errors.exceptions)` 不一定等于所有最深层叶子数量。

这个例子只有一层，所以计数直观。

### 24.5 为什么普通 except 只进一个，except* 能进多个？

普通 except 对当前抛出的异常对象做类型匹配，选择首个合适分支。

except* 把组里匹配的部分交给对应处理块，剩余部分还可以继续匹配后面的 except*。

没有处理的部分会继续向外传播。

这不是让一个普通异常反复进入所有分支，而是异常组拆分的专门语义。

### 24.6 如果只处理 ValueError，TypeError 会消失吗？

不会。

```python
try:
    try:
        raise ExceptionGroup("验证失败", [ValueError("title"), TypeError("days")])
    except* ValueError:
        print("标题问题已处理")
except ExceptionGroup as remaining:
    print(type(remaining.exceptions[0]).__name__)
```

输出：

```text
标题问题已处理
TypeError
```

这里外层普通 except 捕获剩余组。

同一个 try 结构不能把普通 except 和 except* 混写在同一层，所以例子使用嵌套结构。

### 24.7 except* 中可以直接 return 吗？

不能使用某些会从该处理块直接转移控制的 return、break、continue，语法有专门限制。

不要把普通 except 的所有控制流习惯机械搬过来。

可以在处理块里收集结果，等整个结构结束后再决定下一步。

学习时先掌握一层分组，再研究嵌套和重新抛出的细节。[except* 语义](https://docs.python.org/3.14/reference/compound_stmts.html#except-star)

### 24.8 BaseExceptionGroup 又是什么？

它允许包含更广的 BaseException 分支。

普通 ExceptionGroup 只包含 Exception 范围的异常。

业务字段验证通常用 ExceptionGroup 就足够，不需要把用户取消和字段错误随意合成一份可忽略报告。

### 24.9 我要不要把所有批处理都改成 ExceptionGroup？

不必。

如果本来就要生成一张逐行问题报告，普通结构化列表可能更直观。

如果多个并发任务共同失败，或者一套调用协议通过异常传播整批错误，异常组更贴合。

选择依据是调用者如何处理，不是新语法一定比旧列表高级。

## 第二十五章：上下文管理器会不会把异常吞掉？要看退出协议

### 25.1 文件 with 的表现不是所有 with 的统一规则

文件上下文通常关闭后继续传播异常。

自定义上下文管理器却可以选择抑制异常。

因此读代码时，不要看到 with 就认定里面的异常一定传播或一定被处理。

### 25.2 最小类协议

```python
class PracticeContext:
    def __enter__(self):
        print("进入上下文")
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        print("退出时有异常：", exc_type is not None)
        return False


try:
    with PracticeContext():
        raise ValueError("演示失败")
except ValueError:
    print("外层收到异常")
```

输出：

```text
进入上下文
退出时有异常： True
外层收到异常
```

`__exit__` 收到异常类型、异常对象和回溯。

没有异常时这些参数是 None。

返回假值，表示不抑制当前异常。

### 25.3 返回 True 呢？

上下文退出会认为异常已处理，外层可能按正常路线继续。

只有确实完成恢复或明确允许忽略时才应这样做。

例如一个上下文专门忽略“临时标记文件已不存在”，可以抑制那个具体类型。

把所有类型都 return True，会隐藏严重程序缺陷。

### 25.4 contextmanager 装饰器也遵守这个原则

```python
from contextlib import contextmanager


@contextmanager
def practice_phase(name):
    print("开始", name)
    try:
        yield
    except Exception:
        print("阶段失败", name)
        raise
    finally:
        print("阶段退出", name)


try:
    with practice_phase("读取配置"):
        raise ValueError("bad config")
except ValueError:
    print("调用者仍收到错误")
```

输出：

```text
开始 读取配置
阶段失败 读取配置
阶段退出 读取配置
调用者仍收到错误
```

yield 把执行权交给 with 的内部代码。

内部代码异常退出时，异常会在生成器的 yield 位置重新出现。

如果 except 只打印不 raise，上下文就可能表现为把异常处理掉了。

### 25.5 suppress 适合什么？

它明确抑制指定类别。

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    raise FileNotFoundError("练习：目标本来就不存在")
print("允许缺失的清理步骤已结束")
```

输出：

```text
允许缺失的清理步骤已结束
```

适合“删除本来就可能不存在的可丢弃标记”这类明确允许缺失的操作。

不适合 `suppress(Exception)` 包住整个报告生成器，然后只打印成功。

Python 3.12 扩展了 suppress 对异常组的支持，使用 3.11 时不能假定它会对组内叶子做相同拆分。

### 25.6 动态打开多个文件，可以用 ExitStack

```python
from contextlib import ExitStack
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-stack-") as directory:
    root = Path(directory)
    paths = [root / "first.txt", root / "second.txt"]
    paths[0].write_text("A", encoding="utf-8")
    paths[1].write_text("B", encoding="utf-8")
    handles = []
    with ExitStack() as stack:
        for path in paths:
            handles.append(stack.enter_context(path.open(encoding="utf-8")))
        print("".join(handle.read() for handle in handles))
    print(all(handle.closed for handle in handles))
```

输出：

```text
AB
True
```

如果第二个文件打开失败，已经登记的第一个文件仍会得到关闭安排。

资源数量不固定时，这比手工为每个索引写一套 finally 更清楚。

## 第二十六章：先检查再操作，能避免所有文件错误吗？

### 26.1 文件刚检查存在，下一瞬间也可能不在

你可能写出：

```text
如果路径存在：
    打开并读取。
否则：
    显示不存在。
```

问题是检查与打开不是同一个不可分割动作。

两者之间文件可能被删除、更名，权限可能改变。

所以检查存在不能替代实际打开时的错误处理。

### 26.2 EAFP 是什么？

它常被概括为“先尝试做，再处理具体失败”。

例如直接读取，并捕获真正的 FileNotFoundError。

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-eafp-") as directory:
    path = Path(directory) / "missing.txt"
    try:
        content = path.read_text(encoding="utf-8")
    except FileNotFoundError:
        print("文件不存在，请检查输入位置")
```

输出：

```text
文件不存在，请检查输入位置
```

没有先检查再猜测后续结果。

### 26.3 LBYL 又是什么？

它强调先检查前置条件再行动。

比如先验证用户输入天数在 `1..365`，再开展后续工作。

这也很合理。

两种思路不是要你永远二选一。

纯内存输入规则可以先检查，外部资源状态仍需要在真实操作处处理失败。

### 26.4 删除不存在的临时文件，可以忽略吗？

如果你明确把它当作可选清理，可以只忽略 FileNotFoundError。

权限不足、路径是目录、文件系统损坏等其他错误不应全部被吞掉。

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-clean-") as directory:
    path = Path(directory) / "optional-marker.txt"
    try:
        path.unlink()
    except FileNotFoundError:
        pass
    print("缺失标记按规则允许")
```

输出：

```text
缺失标记按规则允许
```

pass 并不天生错误，关键是你是否知道自己为什么允许这一种失败。

### 26.5 捕获 FileNotFoundError 就一定是最后文件名错吗？

不一定。

父目录缺失也可能造成找不到目标。

程序启动时指定的工作目录不存在，也可能产生类似系统异常。

要结合操作、路径与异常属性解释，不要自动提示“请新建这个文件”而不看上下文。

### 26.6 相对路径会不会影响异常解释？

会。

假设脚本在：

```text
D:\project\main.py
```

终端当前目录在：

```text
C:\Users\Alice
```

脚本执行：

```python
from pathlib import Path

config_path = Path("plan.json")
```

这个相对路径默认依赖进程工作目录，不因为脚本放在 project 就自动指向脚本目录。

因此找不到文件时，应先核对你实际尝试的位置。

如果工具约定配置相对于脚本，可以在普通脚本环境用 `Path(__file__).resolve().parent / "plan.json"` 明确构造；Notebook 等环境不一定有 __file__。

异常消息越贴近实际路径约定，用户越容易纠正问题。


## 第二十七章：警告、异常和返回状态，分别告诉调用者什么？

### 27.1 警告通常不立即终止当前操作

Python 的 warnings 模块可以报告弃用、兼容性或其他需要注意的情况。

默认过滤规则、警告类别和运行方式会影响它是否显示。

它不是普通 print，也不等于必然抛出一个异常。

```python
import warnings

with warnings.catch_warnings(record=True) as captured:
    warnings.simplefilter("always")
    warnings.warn("旧配置字段将在后续版本移除", DeprecationWarning)
    print("代码继续")

print(len(captured))
print(captured[0].category.__name__)
```

输出：

```text
代码继续
1
DeprecationWarning
```

这里使用捕获上下文，使示例不依赖当前终端默认过滤规则。

### 27.2 可以把警告当成错误检查吗？

可以在限定范围内调整过滤策略。

```python
import warnings

with warnings.catch_warnings():
    warnings.simplefilter("error", DeprecationWarning)
    try:
        warnings.warn("旧字段", DeprecationWarning)
    except DeprecationWarning:
        print("本次检查把弃用警告视为失败")
```

输出：

```text
本次检查把弃用警告视为失败
```

测试中这有助于提前发现即将失效的 API。

但不要在一个库导入时无条件修改整个进程的全局警告政策，影响所有调用者。

### 27.3 什么时候用返回 None，什么时候抛异常？

如果“没有找到笔记”是查询的正常结果，可以返回 None。

如果“数据库根本无法打开”，返回同一个 None 就会混淆两种状态。

接口应区分正常缺失与无法完成。

例如：

```python
def find_title(mapping, note_id):
    return mapping.get(note_id)


print(find_title({1: "open"}, 2))
```

输出：

```text
None
```

这里 mapping 已经存在，查询不到 key 属于明确允许的正常结果。

如果你设计的是 `require_title()`，语义要求目标必须存在，抛 KeyError 或自己的 NoteNotFoundError 也可以合理。

关键是名字、文档和调用方式保持一致。

### 27.4 为什么不要返回“错误字符串”冒充正常结果？

假设函数正常返回报告文本。

失败时也返回 `"读取失败"`，调用者可能把这句话当报告写入文件。

如果选择结果对象，应明确有成功状态和独立错误字段。

如果选择异常，应保证失败时不返回半真半假的业务值。

两种设计都可以做得清楚，最糟糕的是没有约定地混用。

### 27.5 一个函数有太多可能异常，是不是都写进文档？

不必枚举任何偶然程序缺陷。

但应该说明调用者需要主动处理的公开失败，例如配置格式错误、记录冲突、权限不足或超时。

对于你转换后的领域异常，要说明它携带哪些字段、是否保留底层原因、哪些状态可能已改变。

错误契约也属于函数接口的一部分。

## 第二十八章：完整实践——先检查整份 CSV，再发布一份成功报告

### 28.1 先把成功与失败规则写成人话

输入文件有三列：`title`、`days`、`completed`。

title 去掉两端空白后必须非空，同一文件内不能重复。

days 是 `1..365` 的整数文本。

completed 只能是文本 `0` 或 `1`。

如果出现字段问题，收集所有能继续独立判断的问题，一次报告。

如果文件读不了、编码不对或 CSV 结构无法继续解析，就停止读取并报告文件级失败。

只有整份输入通过后，才生成并替换练习报告。

旧报告在验证失败时保持不变。

### 28.2 为什么把文件级失败和记录级失败分开？

单条记录天数不对，后面的记录仍然可以解析。

但文件读取过程中发生系统错误，不能假装剩余所有记录只是空白。

CSV 引号结构破坏也可能让解析器无法可靠确认后续记录边界。

所以不是所有失败都能“收集后继续”。

### 28.3 定义规范化记录与问题

```python
from dataclasses import dataclass
from pathlib import Path
import csv


@dataclass(frozen=True)
class PlanRecord:
    title: str
    days: int
    completed: bool


@dataclass(frozen=True)
class ImportIssue:
    record_number: int
    field: str
    message: str


class PlanImportError(Exception):
    pass


class PlanImportReadError(PlanImportError):
    pass


class PlanImportValidationError(PlanImportError):
    def __init__(self, issues):
        self.issues = tuple(issues)
        super().__init__(f"输入有 {len(self.issues)} 项验证问题")
```

问题列表用普通结构化数据保存，而不是把独立问题伪造成一长串原因链。

这里不需要 except*，因为调用者本来就希望生成逐条报告。

### 28.4 逐条验证，但最终只返回完整成功集合

```python
def read_plan_records(path):
    records = []
    issues = []
    seen_titles = set()

    try:
        with open(path, "r", encoding="utf-8-sig", newline="") as source:
            reader = csv.DictReader(source, strict=True)
            if reader.fieldnames != ["title", "days", "completed"]:
                raise PlanImportReadError("表头必须依次为 title,days,completed")

            for record_number, row in enumerate(reader, 1):
                issue_start = len(issues)
                if None in row or any(value is None for value in row.values()):
                    issues.append(ImportIssue(record_number, "row", "字段数量与表头不一致"))
                    continue

                title = row["title"].strip()
                if not title:
                    issues.append(ImportIssue(record_number, "title", "不能为空白"))
                elif title in seen_titles:
                    issues.append(ImportIssue(record_number, "title", "与前面记录重复"))
                else:
                    seen_titles.add(title)

                days = None
                try:
                    days = int(row["days"])
                except ValueError:
                    issues.append(ImportIssue(record_number, "days", "必须是整数文本"))
                else:
                    if not 1 <= days <= 365:
                        issues.append(ImportIssue(record_number, "days", "必须在 1 到 365 之间"))

                completed_text = row["completed"].strip()
                if completed_text not in {"0", "1"}:
                    issues.append(ImportIssue(record_number, "completed", "只能是 0 或 1"))

                if len(issues) == issue_start:
                    records.append(PlanRecord(title, days, completed_text == "1"))
    except UnicodeError as error:
        raise PlanImportReadError("输入必须是 UTF-8 CSV") from error
    except csv.Error as error:
        raise PlanImportReadError("CSV 结构无法解析，请检查引号与换行") from error
    except OSError as error:
        raise PlanImportReadError("无法读取输入 CSV 文件") from error

    if issues:
        raise PlanImportValidationError(issues)
    return records
```

### 28.5 逐句看几个关键位置

`records` 暂存通过验证的规范化记录。

只要最后存在 issues，整个函数就不会返回这份部分成功列表。

调用者因此不会误把前几条成功当作整批成功。

`issue_start` 记录处理当前行之前的问题数。

当前行结束时没有增加问题，才创建 PlanRecord。

days 初始为 None；只有整数转换与范围检查都通过，才会进入合法记录构造，所以成功记录中的 days 是整数。

`completed_text == "1"` 把明确的文本协议转成 bool，不使用 `bool("0")`。

因为非空字符串 `"0"` 的布尔值同样是真，直接 bool 转换会得到错误业务状态。

### 28.6 为什么编号叫 record_number，不叫 line_number？

CSV 的带引号字段可以包含换行。

一条逻辑记录可能跨多个物理文本行。

`enumerate(reader, 1)` 这里统计数据记录序号，不是文件物理行号，也不包含表头。

如果你要报告物理行位置，可以进一步参考 csv reader 的 line_num，但要说明它表示已读取的物理行位置，不一定是整条记录开始行。

准确命名能避免用户照着错误行号找半天。

### 28.7 重复标题遇到前一条本身也无效，怎么算？

当前政策是：只要前面出现了同一个非空规范化标题，后续相同标题就报告重复，哪怕前条其他字段无效。

这能让用户一次看到输入内部的标题冲突。

也可以设计成只对完全合法的前条去重，但要明确告诉使用者。

这不是异常语法决定的行为，而是数据契约决定的行为。

### 28.8 生成报告的函数只接受已经验证的数据

依赖第十七章的 `replace_practice_report()`。

```python
def publish_plan_report(input_path, output_path):
    records = read_plan_records(input_path)
    lines = ["学习计划报告"]
    for record in records:
        state = "已完成" if record.completed else "未完成"
        lines.append(f"{record.title} | {record.days} 天 | {state}")
    replace_practice_report(output_path, lines)
    return len(records)
```

先读取并验证全部输入。

再准备报告行。

最后才进入临时写入与发布阶段。

如果验证失败，目标报告根本没有进入写入函数。

### 28.9 一次成功与一次失败，检查旧结果

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-import-") as directory:
    root = Path(directory)
    source = root / "plans.csv"
    report = root / "report.txt"
    source.write_text(
        "title,days,completed\nPython,30,0\nLinux,20,1\n",
        encoding="utf-8",
    )
    print(publish_plan_report(source, report))
    before = report.read_bytes()

    source.write_text(
        "title,days,completed\nPython,bad,2\nPython,0,0\n",
        encoding="utf-8",
    )
    try:
        publish_plan_report(source, report)
    except PlanImportValidationError as error:
        print(error)
        for issue in error.issues:
            print(issue.record_number, issue.field, issue.message)
    print("旧报告未变化：", report.read_bytes() == before)
```

输出：

```text
2
输入有 4 项验证问题
1 days 必须是整数文本
1 completed 只能是 0 或 1
2 title 与前面记录重复
2 days 必须在 1 到 365 之间
旧报告未变化： True
```

这不是只验证“抛出了一个异常”。

它同时验证问题位置、问题总数和旧文件保持原样。

### 28.10 如果读取过程中失败，会不会发布半份报告？

不会进入 publish 的写入阶段，因为 `read_plan_records()` 没有成功返回。

但这个函数使用内存暂存记录和所有问题，不适合无上限文件。

真实大文件工具应设字节、记录、字段长度与问题数量上限，或者采用分阶段临时存储。

异常类型设计与资源限制需要配合，不能只做好错误类就宣称支持任意规模。

### 28.11 如果报告输出路径没有权限呢？

验证已经成功，失败发生在后续文件写入或发布阶段。

这不是输入 CSV 验证失败，不应重新包装成 PlanImportValidationError。

命令行入口可以分别捕获 PlanImportError 和 OSError，向用户说明是输入还是输出问题。

不同阶段的错误边界保持清楚，用户才知道该修改哪一项。

### 28.12 我想把输出改成数据库，异常设计要推倒重来吗？

输入验证部分仍然可以复用。

发布阶段则改成一个明确数据库事务，所有记录写入与任务状态更新一起成功或回滚。

数据库异常要按业务含义转换，不能把所有完整性问题都显示成 CSV 格式错。

不同存储有不同原子性和失败残留，异常体系负责表达，不负责自动创造存储保证。

## 第二十九章：怎样验证异常路径？让失败发生得可重复

### 29.1 不能只测正常输入

如果你写了一套异常处理，却只运行过合法配置，它可能从未进入 except。

消息字段拼错、异常变量不存在、清理顺序错误，都可能直到真正失败才暴露。

要主动构造有意义的坏输入，观察失败以后状态。

### 29.2 不需要真的把磁盘写满

很多错误可以在边界注入。

例如把字符串列表里放一个整数，触发报告验证失败。

把 JSON 字段改成 bool，触发明确类型错误。

把操作函数替换成前两次抛临时错误、第三次成功的本地函数，验证重试次数。

这些实验确定、快速、不会伤害真实数据。

### 29.3 用标准库 unittest 检查异常类型

下面测试依赖第十五章的加载函数与异常定义。

```python
from pathlib import Path
from tempfile import TemporaryDirectory
import unittest


class PlanConfigTests(unittest.TestCase):
    def test_bool_days_is_rejected(self):
        with TemporaryDirectory(prefix="test-plan-") as directory:
            path = Path(directory) / "plan.json"
            path.write_text('{"title":"Python","days":true}', encoding="utf-8")
            with self.assertRaises(PlanConfigValidationError) as captured:
                load_plan_config(path)
            self.assertIn("不能是布尔值", str(captured.exception))

    def test_missing_file_has_cause(self):
        with TemporaryDirectory(prefix="test-plan-") as directory:
            path = Path(directory) / "missing.json"
            with self.assertRaises(PlanConfigReadError) as captured:
                load_plan_config(path)
            self.assertIsInstance(captured.exception.__cause__, FileNotFoundError)


suite = unittest.defaultTestLoader.loadTestsFromTestCase(PlanConfigTests)
result = unittest.TextTestRunner(verbosity=1).run(suite)
assert result.wasSuccessful()
```

测试输出会包含运行数量与 OK，具体耗时不固定。

这里不比较完整系统错误消息，因为那可能受到系统与版本影响。

### 29.4 为什么不只写 assertRaises(Exception)？

它太宽。

如果函数意外出现 NameError，测试也可能“因为确实抛了 Exception”而通过。

你想验证的是明确的公开错误契约，就应检查相应类型。

必要时还检查字段、异常链与副作用状态。

### 29.5 错误消息需要逐字断言吗？

用户界面确实承诺固定措辞时，可以检查自己的文案。

但对系统产生的完整路径、操作系统英文消息和底层库格式，逐字比较往往脆弱。

优先检查稳定类型和结构化属性。

消息只检查真正属于你接口契约的部分。

### 29.6 失败以后最值得检查什么？

原始输入没有被修改。

旧报告字节仍然相同。

临时资源按正常异常路径被关闭。

事务没有留下部分业务记录。

重试次数没有超限，最后原因仍能追溯。

命令行返回失败状态，没有把失败伪装为成功。

这些结果能比“except 被执行了”更完整地证明处理有意义。

### 29.7 测试能证明断电恢复吗？

普通 Python 单元测试不能自动证明操作系统崩溃和硬件故障下的所有行为。

如果任务真的需要这类保证，应设计相应故障注入、文件系统与恢复测试。

不要因为一个临时文件测试通过，就声称解决所有持久化风险。

把证据范围说准确，也能帮助你知道下一步真正缺什么。


## 第三十章：练习题与参考答案，先预测执行路线再运行

### 30.1 练习：except 执行以后，会不会接着执行 try 里剩下的行？

先预测下面的输出，再运行。

```python
events = []
try:
    events.append("A")
    int("bad")
    events.append("B")
except ValueError:
    events.append("C")
finally:
    events.append("D")
events.append("E")
print(events)
```

参考输出：

```text
['A', 'C', 'D', 'E']
```

A 在失败前完成。

B 被跳过，异常处理不会回到 try 失败点继续。

C 是匹配处理块。

D 是离开时的清理路径。

处理完成后，E 正常执行。

如果你实际想处理完以后继续下一条数据，应在记录循环中安排 try，而不是期待控制流自动回到同一个块中。

### 30.2 练习：只捕获转换错误，不吞掉报告错误

函数先把文本转整数，然后调用另一个可能抛 ValueError 的报告函数。

要求只有 int 失败时显示“数字格式错”，报告失败必须向外传播。

参考答案：

```python
def report_days(days):
    raise ValueError("报告模板不可用")


def convert_then_report(text):
    try:
        days = int(text)
    except ValueError:
        return "数字格式错"
    else:
        return report_days(days)


print(convert_then_report("bad"))
try:
    convert_then_report("30")
except ValueError as error:
    print(error)
```

输出：

```text
数字格式错
报告模板不可用
```

关键是报告调用不在同一个受保护的转换 try 内。

如果返回类型在真实项目里需要保持严格，可以把输入错误也设计为异常，而不是像这个小练习返回提示字符串。

### 30.3 练习：自定义字段异常，调用者不拆消息字符串

定义 `FieldError`，保存 field 和 rule。

调用者根据 field 打印高亮位置。

```python
class FieldError(Exception):
    def __init__(self, field, rule):
        self.field = field
        self.rule = rule
        super().__init__(f"{field} 不满足规则：{rule}")


try:
    raise FieldError("days", "1 <= days <= 365")
except FieldError as error:
    print("高亮字段：", error.field)
    print(error.rule)
```

输出：

```text
高亮字段： days
1 <= days <= 365
```

消息可以将来改成更自然的中文，而结构化字段仍然明确。

这正是属性比解析文案更适合作为程序接口的原因。

### 30.4 练习：保留导致配置失败的原始原因

底层 `int()` 抛 ValueError，上层改成 ConfigFieldError，要求 `__cause__` 仍是 ValueError。

```python
class ConfigFieldError(Exception):
    pass


def config_days(text):
    try:
        return int(text)
    except ValueError as error:
        raise ConfigFieldError("days 格式错误") from error


try:
    config_days("thirty")
except ConfigFieldError as error:
    print(type(error.__cause__).__name__)
    print(error)
```

输出：

```text
ValueError
days 格式错误
```

如果改成只打印原错误然后返回 0，调用者失去明确失败信号，还可能生成零天计划。

打印日志不是返回成功结果的理由。

### 30.5 练习：UTF-8 解码错，与 JSON 语法错分开检查

依赖第十五章加载函数。

先写入一个非法 UTF-8 字节，再写一份可解码但 JSON 不合法的文本。

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-encoding-") as directory:
    path = Path(directory) / "plan.json"
    path.write_bytes(b"\xff")
    try:
        load_plan_config(path)
    except PlanConfigFormatError as error:
        print(type(error.__cause__).__name__)

    path.write_text('{"title":', encoding="utf-8")
    try:
        load_plan_config(path)
    except PlanConfigFormatError as error:
        print(type(error.__cause__).__name__)
```

输出：

```text
UnicodeDecodeError
JSONDecodeError
```

对外都是配置格式不可用，但底层原因清楚保留。

如果界面需要分别提示编码与 JSON 语法，可以再细分对外类型，或增加结构化原因字段。

### 30.6 练习：重试三次仍失败，别多睡一次

依赖第二十一章重试函数。

要求 operation 恰好调用三次，sleep 恰好调用两次，最后异常仍是 TemporaryPracticeError。

```python
calls = []
waits = []


def always_unavailable():
    calls.append(1)
    raise TemporaryPracticeError("仍然不可用")


try:
    retry_practice(always_unavailable, attempts=3, delay=0.2, sleep=waits.append)
except TemporaryPracticeError as error:
    print(len(calls))
    print(waits)
    print(error.__notes__[-1])
```

输出：

```text
3
[0.2, 0.2]
已经尝试 3 次，仍未完成
```

最后一次已经没有下一次尝试，不需要再等待。

这类确定性的测试能让你准确检查重试政策，不必真的睡很久。

### 30.7 练习：不属于可重试类型的错误，应立刻暴露

如果 operation 抛 TypeError，要求只执行一次，不进入下一次尝试。

```python
calls = []
waits = []


def programming_failure():
    calls.append(1)
    raise TypeError("模拟调用者参数用错")


try:
    retry_practice(programming_failure, attempts=3, sleep=waits.append)
except TypeError:
    print(len(calls))
    print(waits)
```

输出：

```text
1
[]
```

函数没有承诺修复所有错误。

只重试明确分类的临时失败，使意外程序缺陷更快显现。

### 30.8 练习：用户取消不能被普通业务错误捕获

不实际触发键盘中断，用手动对象演示继承匹配。

```python
try:
    try:
        raise KeyboardInterrupt
    except Exception:
        print("普通错误分支")
except KeyboardInterrupt:
    print("中断到达外层")
```

输出：

```text
中断到达外层
```

外层只为演示而捕获中断。

真实命令行如果要显示“已取消”，可以在入口单独处理；不要捕获后再打印“全部成功”。

### 30.9 练习：异常组里只处理一类，剩余仍向外传播

要求先处理两个 ValueError，再让一个 TypeError 到达外层。

```python
try:
    try:
        raise ExceptionGroup(
            "三个问题",
            [ValueError("title"), TypeError("days"), ValueError("date")],
        )
    except* ValueError as group:
        print("已处理值错误：", len(group.exceptions))
except ExceptionGroup as remaining:
    print("剩余类别：", type(remaining.exceptions[0]).__name__)
```

输出：

```text
已处理值错误： 2
剩余类别： TypeError
```

如果把 except* 当普通 except，只盯着整个 ExceptionGroup 的类名，就很难理解为什么可以拆出一部分。

记住：匹配的是内部相应部分，未匹配部分不会自动消失。

### 30.10 练习：生成器转换错误发生在哪一次 next？

依赖第二十二章 `parsed_numbers()`。

要求第一次返回 1，第二次失败，之后生成器结束。

```python
values = parsed_numbers(["1", "bad", "3"])
print(next(values))
try:
    next(values)
except ValueError:
    print("第二次失败")
print(next(values, "结束"))
```

输出：

```text
1
第二次失败
结束
```

外层捕获没有把已退出的生成器重新激活。

如果业务需要跳过坏记录，必须在生成器内部设计每条记录的恢复路径。

### 30.11 练习：清理代码不能把失败覆盖成成功

给出一个函数，内部抛 ValueError，finally 只记录清理。

要求调用者仍收到 ValueError。

```python
events = []


def cleanup_without_hiding():
    try:
        raise ValueError("任务失败")
    finally:
        events.append("closed")


try:
    cleanup_without_hiding()
except ValueError:
    print("失败仍可观察")
print(events)
```

输出：

```text
失败仍可观察
['closed']
```

finally 不写 return，也不把原本异常转换成普通成功值。

如果清理本身可能失败，还需要决定如何同时保留主错误和清理错误，下一章会接着解释。

### 30.12 练习：命令行失败返回二，诊断放 stderr

依赖第十九章 `plan_main()`。

为了让练习在同一个进程中运行，用 redirect_stderr 捕获错误文本。

```python
from contextlib import redirect_stderr
from pathlib import Path
from tempfile import TemporaryDirectory
import io

with TemporaryDirectory(prefix="exception-main-") as directory:
    missing = Path(directory) / "missing.json"
    errors = io.StringIO()
    with redirect_stderr(errors):
        code = plan_main([str(missing)])
    print(code)
    print("无法生成计划" in errors.getvalue())
```

输出：

```text
2
True
```

我们直接调用 main 返回值，没有真的让当前进程退出。

测试与真实入口由此可以复用同一份逻辑。

### 30.13 练习：失败以后原报告必须逐字节不变

依赖第十七章的发布函数。

这次不只检查文件存在，而是检查字节相等。

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory(prefix="exception-unchanged-") as directory:
    path = Path(directory) / "report.txt"
    previous = "原报告\n  保留缩进和空格  \n".encode("utf-8")
    path.write_bytes(previous)
    try:
        replace_practice_report(path, ["新第一行", None])
    except TypeError:
        pass
    print(path.read_bytes() == previous)
```

输出：

```text
True
```

为什么不 `.strip()` 后再比较？

因为旧文件里的空格与换行也属于内容。

如果先清理两端空白，可能掩盖真实变化。

### 30.14 练习：哪些失败不应该变成“请修改输入”？

输入文本是 `bad`，转换整数失败，是用户可以修正的格式问题。

函数内部把变量名 days 拼成 dayz，产生 NameError，是程序缺陷。

报告输出目录没有写入权限，是运行环境或目标位置问题。

用户按取消，是明确的控制请求。

四者都能打断正常流程，但恢复行动不同。

先写出用户下一步应该做什么，再决定捕获类型和提示内容，通常比先套一个宽泛 except 更容易设计正确。

## 第三十一章：清理也失败时，应该怎样保留真正发生过的事情？

### 31.1 主操作与清理可能各有一次失败

假设读取数据失败，关闭某个特殊外部资源时也失败。

如果清理异常直接向外抛，最终回溯最后显示的可能是清理错误。

Python 通常会通过上下文保留此前异常，但普通用户只看最后一行，可能忽略真正的起点。

### 31.2 一个完全内存里的演示

```python
def main_and_cleanup_fail():
    try:
        raise ValueError("主操作失败")
    finally:
        raise RuntimeError("清理也失败")


try:
    main_and_cleanup_fail()
except RuntimeError as error:
    print(error)
    print(type(error.__context__).__name__)
```

输出：

```text
清理也失败
ValueError
```

这不表示主失败从未发生。

它被保留在新异常的上下文里。

### 31.3 我应该吞掉清理错误吗？

不能一概而论。

删除一个可丢弃临时标记失败，可以记录并按政策继续报告主错误。

数据库提交失败、输出文件关闭时写入失败，可能影响结果真实性，不应悄悄忽略。

要根据资源的业务重要性决定是否转换、记录、组合或优先传播。

### 31.4 可以用异常组同时表达两者吗？

在明确的业务边界里可以，把两个普通 Exception 放入 ExceptionGroup，并给出关系说明。

但不要为了机械统一而把取消、中断与普通业务错误随意组合。

也不要让所有 finally 都变成一套复杂异常树。

简单可靠的资源管理器、短清理路径和恰当日志，通常是第一步。

### 31.5 为什么初始化成功后要立刻登记清理？

取得第一个资源后，第二个资源初始化就可能失败。

如果你等所有初始化结束才写 try/finally，前面已经取得的资源可能漏掉清理。

with 与 ExitStack 的价值就在这里：每获得一项，就及时建立退出责任。

不要把“对象创建语句都写完”作为开始考虑失败的时间点。

### 31.6 错误处理函数自己还要访问不可靠网络吗？

最好避免把最基本的清理和失败反馈依赖在另一串可能无限等待的操作上。

发送远程错误报告可以有超时和兜底，但不能让它阻止本地资源结束。

如果诊断服务也坏了，原任务仍然应有明确失败状态。

失败路径往往比成功路径更需要简单、有限和可预测。

## 第三十二章：把这些语法放回你自己的工具里

### 32.1 我是不是应该每个函数都加 try？

不需要。

只有当前层能够恢复、补充有价值上下文、转换抽象或者清理资源时，try 才有明确作用。

否则让异常自然传播到有处理能力的一层，代码会更清楚。

函数越多不代表要有越多 except。

### 32.2 如果我不捕获，程序退出是不是很不专业？

开发阶段清楚的回溯往往比假装成功更有帮助。

正式工具则应在入口处理可预期业务错误，给用户明确提示和失败状态。

意外缺陷需要记录诊断并推动修复，而不是藏在“未知错误，请重试”后面永远不看。

专业程度不取决于有没有红字，而取决于失败是否可理解、状态是否可信、问题是否可定位。

### 32.3 我需要统一一个 AppError 吗？

如果多个模块共享一个稳定的应用边界，一个业务基类很有帮助。

但它不应该接管所有 Python 异常，也不需要把每个 TypeError 都转换成 AppError。

先定义调用者确实需要处理的几类错误，再把它们放进清楚层级。

异常体系应该随着真实需求增长，不要提前构造一个庞大错误编号框架。

### 32.4 如果第三方库升级改变异常类型怎么办？

公共接口依赖哪些错误类别，应写入测试与兼容策略。

在有意义的边界转换为自己的领域异常，可以减少调用者对底层实现的耦合。

但转换前仍要准确理解新版本语义，不能只是扩大成 except Exception。

底层错误不再属于可恢复情境时，旧的自动重试政策也可能需要调整。

### 32.5 本篇实际运行了什么？

关键示例在 Python 3.12.3 中验证。

包含整数验证、捕获顺序、else/finally 执行路线、自定义属性、异常链与 note、临时配置加载、UTF-8 与 JSON 失败、报告准备失败保持旧文件、批处理问题汇总、命令行错误状态、日志回溯、有限重试、生成器、asyncio 取消、异常组、上下文清理和 CSV 整批验证发布。

Python 3.14 的 finally 控制流 SyntaxWarning 依据官方语义标注，没有声称在本机 3.12 实际产生了 3.14 的编译警告。

系统产生的完整异常文案、临时路径、回溯行号与线程调度，不当作跨平台固定字符串。

### 32.6 想继续核对原始语义，具体从哪里看？

开始阅读可以用 [Python 官方错误与异常教程](https://docs.python.org/3.14/tutorial/errors.html)，它介绍基本传播、处理和用户自定义异常。

具体继承关系、属性和特殊错误查 [内建异常参考](https://docs.python.org/3.14/library/exceptions.html)。

try、else、finally 与 except* 的准确控制流查 [复合语句中的 try 说明](https://docs.python.org/3.14/reference/compound_stmts.html#the-try-statement)。

异常链的语法查 [raise 语句](https://docs.python.org/3.14/reference/simple_stmts.html#the-raise-statement)。

上下文退出与抑制规则查 [contextlib](https://docs.python.org/3.14/library/contextlib.html)。

异步任务取消查 [asyncio 任务取消](https://docs.python.org/3.14/library/asyncio-task.html#task-cancellation)。

警告过滤查 [warnings](https://docs.python.org/3.14/library/warnings.html)，回溯保存查 [traceback](https://docs.python.org/3.14/library/traceback.html)。

把这些语法放回最初的笔记脚本，你就可以具体回答：哪些输入不被接受，失败怎样到达入口，哪些资源必须释放，哪些结果尚未发布，哪些动作允许重做，以及调用者用什么判断这次工作没有完成。

这些问题有了清楚答案，异常处理就不再只是把报错藏起来，而成为你安排程序行为的一部分。
