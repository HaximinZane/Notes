# Python `functools` 库超详细教程：从函数对象与参数预绑定，到装饰器、缓存和类型分派

## 零基础先导：函数也可以被“加工”，为什么？

`functools` 是 function tools（函数工具）的意思。它处理的不是“读取文件”“发送网络请求”，而是**已有函数对象的组合、包装与复用**。

先区分两个表达式：

```python
def add(a, b):
    return a + b

f = add              # 不带括号：把函数对象赋给 f
result = add(2, 3)   # 带括号：立即调用，得到整数 5
```

如果你分不清它们，后面很容易把 `partial` 或装饰器看成魔法。

### 三种最常见的现实需求

**固定部分参数**：原本 `add(a, b)` 需要两个输入，但你想生成“固定加十”的新函数。

```python
from functools import partial

add_ten = partial(add, 10)  # 先绑定第一个参数为 10
print(add_ten(3))            # 等价于 add(10, 3)，得到 13
```

**缓存重复计算**：同样参数反复计算且结果稳定，可以用 `@lru_cache`；但文件状态、时间、随机数等变化时缓存可能不再反映最新值。

**保留被装饰函数的信息**：自己写包装函数时，`@wraps(original)` 可帮助保留原函数的名称、文档等元数据。它不意味着包装代码与原函数完全相同。

后文遇到 `reduce`、`singledispatch`，也按“原本要写什么重复代码 → 函数工具怎样替代 → 有什么代价”来学。普通 `for`、`if` 能清楚解决的问题，不必为了使用 `functools` 而强行使用。

---

你已经见过装饰器，也知道函数可以作为参数传给另一个函数。

接下来，实际写代码时很容易继续遇到这些问题。

有一个函数参数很多，我想固定其中几个，生成一个更方便调用的版本，可以吗？

装饰以后，为什么函数名称变成了 `wrapper`，帮助信息也不对了？

同样参数的计算重复执行，怎样保存结果？

用了缓存，为什么文件已经更新，程序却还返回旧内容？

不同类型的数据要用不同方式格式化，是不是只能写很长的 `if isinstance(...)`？

这些问题正是 `functools` 的常见用途。

本篇按 Python 3.11 及以上的常用接口展开。`Placeholder` 等 Python 3.14 新能力，以及 `cached_property` 在 3.12 的行为变化，会明确标注。

同一小节的短例子沿用本节变量；完整实践包含所需导入和输入。标注预期错误的代码只用于观察对应边界，不应当作正常流程连续执行。

## 第一章：为什么处理函数，也需要一个工具库？

### 1.1 `functools` 是什么含义？

可以把它拆成 function 和 tools，理解为函数工具。

模块里有些工具接收函数，返回一个新的可调用对象。

有些工具帮助保留函数的说明信息。

有些工具把函数调用结果缓存起来。

还有一些工具按参数类型选择实现。

它属于标准库，不需要额外安装：

```python
import functools
```

日常也经常直接导入所需名称：

```python
from functools import partial, wraps, lru_cache
```

### 1.2 先区分函数本身与调用结果

```python

def add(first, second):
    return first + second


operation = add
result = add(2, 3)

print(callable(operation))
print(result)
```

`operation` 保存函数对象。

`result` 保存调用后返回的五。

看到下面这行：

```python
partial(add, 10)
```

传入的是 `add` 本身。

如果误写：

```python
partial(add(10, 20), 30)
```

就先执行了加法，再把三十当作需要包装的对象，含义完全改变。

理解括号什么时候出现，是学习高阶函数的第一步。

### 1.3 什么叫高阶函数？

一个函数可以接收函数，或返回函数等可调用对象。

这样的用法经常被称为高阶函数。

例如：

```python

def apply_twice(operation, value):
    return operation(operation(value))


def add_one(value):
    return value + 1


print(apply_twice(add_one, 10))
```

输出十二。

`apply_twice()` 不需要知道具体是加一、平方还是其他变换，只需要调用收到的对象。

`functools` 把许多这样的常见组合方式组织成标准接口。

## 第二章：`partial()`——固定部分参数，生成方便调用的对象

### 2.1 先看一个有多个参数的函数

```python

def power(base, exponent):
    return base ** exponent


print(power(2, 3))
print(power(5, 2))
```

假设接下来经常计算平方。

你可以写一个新函数：

```python

def square(value):
    return power(value, 2)
```

也可以使用 `partial()`，预先提供指数参数。

```python
from functools import partial

square = partial(power, exponent=2)
print(square(5))
```

输出二十五。

### 2.2 为什么叫 partial？

partial 是“部分的”。

这里可以按“部分参数已经绑定好”理解。

你先提供一部分参数，后续调用时再提供其余参数。

它不是马上计算所有可能结果，也不是根据函数名称自动创造数学上的偏导数。

### 2.3 基本语法

```python
partial(func, *args, **keywords)
```

`func` 是原始可调用对象。

`args` 是预先绑定的位置参数。

`keywords` 是预先绑定的关键字参数。

返回 `partial` 对象，这个对象仍然可以调用。

```python
from functools import partial

add_ten = partial(add, 10)
print(add_ten(5))
```

相当于调用 `add(10, 5)`。

### 2.4 后续位置参数放在哪里？

通常追加到已绑定位置参数后面。

```python
from functools import partial


def show(first, second, third):
    return first, second, third


prepared = partial(show, "A")
print(prepared("B", "C"))
```

得到 A、B、C。

这也是旧版本中普通 `partial` 更适合固定前面位置参数，或通过名字固定其他参数的原因。

### 2.5 绑定关键字以后，还能覆盖吗？

可以由后续同名关键字覆盖。

```python
from functools import partial

square = partial(pow, exp=2)
print(square(5))
print(square(5, exp=3))
```

第二次使用新的指数三。

因此，`partial` 的关键字预绑定更像默认配置，不是不可更改的权限限制。

如果必须禁止调用者改变某个选项，应该写显式验证或封装接口。

### 2.6 为什么有时会报“同一参数得到了多个值”？

看自定义函数：

```python
from functools import partial


def greet(name, prefix):
    return f"{prefix}，{name}"


hello = partial(greet, name="小林")
# 预期 TypeError：位置参数还会先匹配 name。
hello("你好")
```

关键字绑定没有把原函数的位置参数规则重新排好。

后面的 `"你好"` 仍然先按第一个位置参数匹配 `name`，于是与已有关键字冲突。

可以写：

```python
print(hello(prefix="你好"))
```

或者调整原函数的参数设计，让常见预绑定方式更清楚。

### 2.7 `partial` 可以代替所有 `lambda` 吗？

它很适合“同一个函数，只提前给出部分参数”。

如果你需要重新排列参数、增加判断或执行多步逻辑，一个命名函数往往更清楚。

简单 `lambda` 也可以，但应考虑读者是否容易看懂和以后是否方便调试。

不要为了使用工具而把三步业务处理挤进一个难读的表达式。

## 第三章：`partial` 对象里保存了什么？

### 3.1 三个常用观察属性

```python
from functools import partial


def multiply(first, second, scale=1):
    return first * second * scale


operation = partial(multiply, 2, scale=10)

print(operation.func is multiply)
print(operation.args)
print(operation.keywords)
```

`func` 指向原可调用对象。

`args` 保存预绑定位置参数元组。

`keywords` 保存相应关键字字典。

它们有助于调试“这个回调究竟固定了什么”。

### 3.2 预绑定列表时，是复制还是引用？

```python
from functools import partial


def describe(values, prefix):
    return prefix, list(values)


numbers = [1, 2]
operation = partial(describe, numbers)
numbers.append(3)

print(operation("当前内容"))
```

调用时能看到后续增加的三。

预绑定不会自动深拷贝参数对象。

如果你希望冻结一份数据快照，可以先复制或转换成合适的不可变结构。

### 3.3 函数定义后，原变量换了，绑定会跟着换吗？

```python
from functools import partial


def show(value):
    return value


number = 10
operation = partial(show, number)
number = 20

print(operation())
```

仍然得到十。

创建 `partial` 时，把当时的对象引用保存进去。

后续让变量 `number` 指向另一个整数，不会重新修改已绑定参数。

这和“同一个列表对象内部被修改”需要分开理解。

### 3.4 能嵌套使用吗？

```python
from functools import partial


def combine(first, second, third):
    return first, second, third


first_stage = partial(combine, "A")
second_stage = partial(first_stage, "B")
print(second_stage("C"))
```

可以继续预绑定。

但层数多时，最好检查 `.func`、`.args` 和 `.keywords`，或用一个有意义名称的普通函数组织，避免调用契约变得隐蔽。

### 3.5 Python 3.14 的 `Placeholder` 解决什么？

旧式普通位置预绑定主要固定前面的位置。

Python 3.14 增加占位符，让你保留某些位置，先绑定后面的参数。

```python
# Python 3.14+
from functools import partial, Placeholder

remove_spaces = partial(str.replace, Placeholder, " ", "")
print(remove_spaces("a b c"))
```

调用时，输入字符串填入占位位置。

这个能力不能直接复制到 Python 3.11～3.13 中使用。

如果使用占位符，所有保留位置都必须在调用时得到填充；它也不能作为普通关键字参数值随意用来表示“以后再填”。

旧版本可以用明确的小函数表达相同需求，不必为了一个简写马上升级整个项目环境。

## 第四章：`partialmethod()`——类方法还涉及自动传入 `self`

### 4.1 先看一个通用方法

```python
class Reporter:
    def report(self, level, message):
        return f"[{level}] {message}"
```

你希望实例有 `info()` 和 `error()` 两种方便入口，自动固定级别。

```python
from functools import partialmethod


class Reporter:
    def report(self, level, message):
        return f"[{level}] {message}"

    info = partialmethod(report, "INFO")
    error = partialmethod(report, "ERROR")


reporter = Reporter()
print(reporter.info("开始处理"))
print(reporter.error("处理失败"))
```

### 4.2 为什么不是普通 `partial` 就够了？

普通函数放在类上，通过实例访问时，会参与方法绑定，把实例作为 `self`。

普通 `partial` 对象不按普通函数的同一方式自动成为绑定实例方法。

`partialmethod` 专门与描述符和方法绑定配合，让 `self` 的位置正确，再处理预绑定参数。

所以，“对象可以调用”和“对象在类上具有普通方法绑定语义”不是同一个条件。

### 4.3 已经拿到实例的绑定方法后呢？

这时普通 `partial` 可以很自然：

```python
from functools import partial

reporter = Reporter()
info = partial(reporter.report, "INFO")
print(info("一条消息"))
```

`reporter.report` 本来已经绑定实例。

你不再需要让类属性访问过程替你补 `self`。

这两个场景的差别发生在你保存的是未绑定方法定义，还是已绑定方法对象。

## 第五章：装饰以后，函数的名字为什么变了？

### 5.1 先写一个没有保留信息的包装器

```python

def announce(func):
    def wrapper(*args, **kwargs):
        print("准备调用")
        return func(*args, **kwargs)
    return wrapper


@announce
def greet(name: str) -> str:
    """向指定名字问好。"""
    return f"你好，{name}"


print(greet.__name__)
print(greet.__doc__)
```

你可能看到 `wrapper` 和空文档说明。

因为 `greet` 这个名称现在引用的是包装函数。

原函数并没有因为名字被替换，就自动把自己的说明资料转移给另一个对象。

### 5.2 `wraps()` 解决什么？

```python
from functools import wraps


def announce(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print("准备调用")
        return func(*args, **kwargs)
    return wrapper


@announce
def greet(name: str) -> str:
    """向指定名字问好。"""
    return f"你好，{name}"


print(greet.__name__)
print(greet.__doc__)
print(greet("小林"))
```

现在名称和文档能保留为原函数相关信息。

包装逻辑仍然存在，调用仍先经过 `wrapper`。

### 5.3 `@wraps(func)` 本身也是装饰器吗？

是。

它接收被包装函数资料，返回用于更新包装对象信息的装饰器。

放在内部 `wrapper` 上，是让这个新函数带上原函数的相关元信息。

这和业务装饰器 `announce` 的职责不同：一个负责行为，一个帮助维护可观察信息。

### 5.4 它会把包装函数的运行时参数检查也变成原函数吗？

不会自动把 `*args, **kwargs` 的包装代码改写成原函数同样的参数声明。

调试和检查工具可能通过 `__wrapped__` 追踪原函数，显示相应签名。

但真正执行的包装函数仍然运行你写的代码。

如果包装器先错误地处理参数，`wraps()` 不会帮你修复。

### 5.5 `__wrapped__` 是什么用途？

```python
print(greet.__wrapped__.__name__)
```

它提供对被包装对象的引用。

某些检查工具可以沿这条关系找到原函数。

必要时也可以直接调用原函数，但那会绕过包装逻辑，例如绕过计数、缓存或检查，应明确知道自己为什么这样做。

### 5.6 为什么保留名称和文档这么重要？

当很多函数都显示为 `wrapper`，日志定位、帮助信息、测试报告和调试器显示都会变得含糊。

保留元信息让包装后的函数仍然容易辨认。

但元信息不等于行为证明，看到名字保留正常，不表示包装器一定正确转发参数、返回值或异常。

## 第六章：`update_wrapper()` 与带参数装饰器的多种实现

### 6.1 `wraps()` 背后是什么操作？

`update_wrapper(wrapper, wrapped)` 直接更新包装对象的相关资料，并返回包装对象。

`wraps(wrapped)` 把它组织成适合放在 `@` 后面的形式。

```python
from functools import update_wrapper


def original(value):
    """返回传入值。"""
    return value


def wrapper(*args, **kwargs):
    return original(*args, **kwargs)


update_wrapper(wrapper, original)
print(wrapper.__name__)
```

### 6.2 带参数装饰器，一定必须写三层 `def` 吗？

三层函数是非常常见、容易解释的一种写法。

外层接收装饰器配置。

中层接收被装饰函数。

内层处理真正调用。

但这是实现组织方式，不是 Python 语法强制必须有三层嵌套 `def`。

语法真正需要的是：装饰器表达式求值得到一个可调用对象，再把被装饰对象交给它。

这个可调用对象可以来自函数、类实例、`partial` 等方式。

### 6.3 用 `partial` 组织一个带标签的装饰器

```python
from functools import partial, update_wrapper


def invoke_with_label(func, label, *args, **kwargs):
    print(f"[{label}] 开始")
    try:
        return func(*args, **kwargs)
    finally:
        print(f"[{label}] 结束")


def decorate_with_label(func, *, label):
    wrapper = partial(invoke_with_label, func, label)
    return update_wrapper(wrapper, func)


def labeled(label):
    return partial(decorate_with_label, label=label)


@labeled("greeting")
def greet(name):
    return f"你好，{name}"


print(greet("小林"))
```

这里没有三层嵌套函数。

`labeled()` 返回部分参数已绑定的装饰器。

装饰器收到 `greet` 后，再返回部分参数已绑定的调用对象。

`update_wrapper()` 负责让这个调用对象保留原函数资料。

### 6.4 为什么这里使用 `finally`？

我们希望普通异常传播时也打印结束标记。

这不表示异常被吞掉，原异常仍然向外传播。

如果你只把结束打印放在正常返回之后，就可能在异常路径看不到它。

### 6.5 这段替代实现适合所有方法吗？

它主要演示普通函数装饰。

返回 `partial` 对象时，类属性上的方法绑定行为还要另外考虑，不能把它当作所有函数包装器完全可互换的实现。

这正好与前面 `partialmethod` 的问题联系起来。

学习多种实现的目的，是理解语法要求和对象协议，不是宣称一种写法可以无条件替代所有其他写法。

## 第七章：`cache()`——相同参数的结果，能不能直接复用？

### 7.1 先观察重复计算

```python

def square(number):
    print("真正计算：", number)
    return number * number


print(square(5))
print(square(5))
```

两次都会进入函数体。

如果函数很慢，而相同输入应得到相同输出，你可能希望第二次直接使用先前结果。

### 7.2 加上缓存装饰器

```python
from functools import cache


@cache
def square(number):
    print("真正计算：", number)
    return number * number


print(square(5))
print(square(5))
```

第二次同样参数调用时，通常不再打印“真正计算”。

包装器根据参数查找已经保存的结果，命中后直接返回。

`cache` 从 Python 3.9 起提供，本文基础环境可用。

### 7.3 缓存是在装饰时计算的吗？

不是。

定义函数并应用装饰器时，建立了具有缓存能力的调用入口。

具体参数的结果要等真正调用后才产生。

你没有调用 `square(5)`，缓存就不会凭空知道五的平方是多少。

### 7.4 缓存保存在哪里？

它关联到这个包装函数对象，位于当前 Python 进程的运行状态中。

程序重启、重新建立包装对象后，不应期待旧结果自动从磁盘恢复。

`cache()` 不是持久化数据库，也不自动在多个进程之间共享。

### 7.5 怎样观察命中情况？

```python
print(square.cache_info())
```

你会看到命中、未命中、容量和当前条目数量等信息。

命中表示找到现成结果。

未命中表示需要执行原函数取得结果。

它们统计的是这个包装器的调用行为，不等于服务器请求总量或整个系统的缓存命中率。

### 7.6 想清空怎么办？

```python
square.cache_clear()
```

清理后，再调用相同参数通常会重新计算。

缓存统计也会相应重置。

如果其他地方还保存着先前返回对象的引用，清缓存不等于立即销毁那些外部仍在使用的对象。

## 第八章：`lru_cache()`——结果很多时，怎样限制条目数？

### 8.1 无界缓存有什么问题？

如果输入不断变化，`cache()` 会持续保存不同参数与结果。

每份结果可能只是一个整数，也可能是一段很大的文本。

条目持续增长，内存也可能持续增长。

`lru_cache()` 可以设置最多保存多少条调用结果。

### 8.2 LRU 是什么含义？

Least Recently Used，最近最少使用。

容量满后，长期没有访问的条目优先淘汰。

“最近使用”不仅包括新计算，也包括命中读取。

```python
from functools import lru_cache


@lru_cache(maxsize=2)
def transform(number):
    print("计算：", number)
    return number * 10


transform(1)
transform(2)
transform(1)
transform(3)
transform(2)

print(transform.cache_info())
```

一先创建，二再创建。

再次访问一，让一变成最近使用。

三加入时，二被淘汰。

随后再访问二，就需要重新计算。

### 8.3 `maxsize` 是字节数吗？

不是。

它限制缓存条目数量。

两百条很小结果与两百条巨大结果，占用可能相差非常多。

需要严格内存预算时，还要考虑键和值的体积，以及应用的存储策略。

### 8.4 `maxsize=None` 和 `cache()` 有什么关系？

`lru_cache(maxsize=None)` 表达不做 LRU 容量淘汰的缓存。

`cache()` 为这种常见无界缓存需求提供简洁入口。

普通代码应根据键空间是否有界、结果规模和生命周期选择，不必为了少几个字符无条件使用无界版本。

### 8.5 `maxsize=0` 又是什么？

这种设置不会保存可复用结果，但仍有包装和统计行为。

它可以用于某些配置切换或观察，不过如果完全不需要缓存，直接调用原函数通常更清楚。

### 8.6 省略参数和直接写装饰器可以吗？

常见写法是：

```python
@lru_cache(maxsize=128)
def calculate(value):
    return value * 2
```

现代 Python 也支持相应不传配置的形式，使用默认策略。

初学时明确写出 `maxsize`，更容易让读者知道应用选择了多大缓存。

## 第九章：缓存键——为什么列表不能直接当参数？

### 9.1 缓存要根据什么查找？

包装器需要把调用参数组织成可查找的键。

因此，参数需要满足可哈希要求。

```python
from functools import lru_cache


@lru_cache(maxsize=32)
def total(values):
    return sum(values)


print(total((1, 2, 3)))
# 预期 TypeError：列表不能直接作为这类缓存键。
print(total([1, 2, 3]))
```

元组如果包含不可哈希元素，也不能仅凭外层是元组就保证可用。

### 9.2 我转成元组就一定正确吗？

对简单数值序列，这通常可以表达一份不可变参数。

但要确认顺序是否重要、重复项是否重要、嵌套元素是否可哈希。

不要为了让函数接受缓存，就随手排序或去重，改变原计算含义。

例如 `[1, 2]` 和 `[2, 1]` 对求和相同，对生成顺序报告却可能不同。

### 9.3 位置调用和关键字调用会共用同一条目吗？

不要把所有“业务上等价”的调用都假设成同一个缓存键。

```python
calculate(10)
calculate(value=10)
```

不同调用形式可能形成不同缓存条目。

关键字顺序差异也可能影响条目组织。

如果你希望统一，应在外层规范化参数，再调用一个形式固定的内部缓存函数。

### 9.4 `typed=True` 做什么？

```python
from functools import lru_cache


@lru_cache(maxsize=32, typed=True)
def describe(value):
    return type(value).__name__, str(value)
```

它要求不同直接参数类型分别缓存。

但不要把它理解成递归检查所有嵌套容器内部元素类型。

`typed=False` 也不保证所有不同但相等的类型都无条件合并成同一条目，具体键路径存在差别。

如果类型影响业务结果，主动选择清楚的参数模型和 `typed` 政策，比依赖偶然命中更可靠。

### 9.5 对字典配置怎样构造稳定键？

如果配置明确是“字符串键、整数值”，可以转换成按键排序的元组：

```python

def configuration_key(config):
    if not isinstance(config, dict):
        raise TypeError("config 必须是字典")
    for key, value in config.items():
        if not isinstance(key, str) or type(value) is not int:
            raise ValueError("这里只接受字符串键和整数值")
    return tuple(sorted(config.items()))
```

这是一种针对明确模型的转换。

它不是适用于任意嵌套对象、循环引用、集合和资源对象的万能冻结器。

缓存键应体现函数真实依赖的数据，而不是只追求“现在不报 TypeError”。

## 第十章：哪些函数不适合直接缓存？

### 10.1 结果依赖外部变化

```python
from functools import cache

current_theme = "light"


@cache
def get_theme(user_id):
    return current_theme
```

第一次调用后，改变全局主题，同参数仍可能返回旧结果。

因为缓存键只看到 `user_id`，没有记录全局变量版本。

函数结果依赖哪些东西，都需要在设计时考虑进去。

### 10.2 当前时间和随机值

如果函数的目标就是每次取得新的当前时间或随机结果，缓存会改变它的用途。

把所有无参数函数都加上 `@cache`，不等于免费加速。

无参数调用只有一种键，第一次结果可能一直复用。

### 10.3 有副作用的函数

假设函数发送消息、写入文件或记录一次操作。

缓存命中时，函数体不再执行，相应副作用也不会再次发生。

如果业务要求每次调用都执行动作，就不能把调用结果缓存当成透明优化。

### 10.4 返回可变列表，调用者修改了怎么办？

```python
from functools import cache


@cache
def make_numbers(size):
    return list(range(size))


first = make_numbers(3)
first.append(99)
second = make_numbers(3)

print(second)
print(first is second)
```

返回的是同一个缓存对象，第二次能看到九十九。

缓存不自动替调用者创建独立副本。

可以约定缓存不可变元组，再在外层按需要复制成列表。

### 10.5 `None` 会被当成没有缓存吗？

正常缓存会保存成功返回的 `None`，不会因为它是假值就自动重新执行。

同理，零、空字符串和 `False` 也可以是合法缓存结果。

不要在手写缓存时使用 `if not cached_value` 判断是否命中，否则会把合法假值误认为缺失。

### 10.6 抛出的异常也会被缓存吗？

普通 `cache`、`lru_cache` 不会把一次抛异常的调用当作正常结果保存复用。

再次调用相同参数，通常仍会进入原函数并再次失败。

如果你想短期缓存“资源不存在”等负结果，应在应用协议中明确用什么值表示，以及多久失效，而不是假设异常自动缓存。

### 10.7 生成器函数为什么不适合直接这样缓存？

生成器函数调用返回生成器对象。

如果缓存保存这个对象，多次相同调用可能拿到同一个已经推进甚至耗尽的迭代器。

```python
from functools import cache


@cache
def numbers():
    yield 1
    yield 2


print(list(numbers()))
print(list(numbers()))
```

第二次可能为空。

如果你真正想复用有限结果，可以先明确收集为不可变数据，再按需生成新迭代器，承担相应保存成本。

## 第十一章：缓存失效、引用生命周期与线程

### 11.1 LRU 会不会过几分钟自动过期？

不会因为时间经过，就自动按你想象的 TTL 失效。

LRU 根据访问与容量淘汰，不是基于固定时间期限的缓存。

需要时间过期，应设计明确的过期记录或使用合适工具，并考虑时钟、并发和失败处理。

### 11.2 `cache_clear()` 是唯一可能的失效方式吗？

它是包装器提供的清理全部缓存入口。

你也可以让版本号成为键的一部分，或重新建立包装器。

标准公共接口没有给你一个通用的“直接删除任意内部键”操作，不能依赖私有实现随意操作。

如果业务频繁需要逐项删除，显式缓存容器可能更适合。

### 11.3 `cache_parameters()` 能修改容量吗？

它提供参数信息。

修改返回字典不代表实际缓存配置随之改变。

需要新的容量策略时，应建立相应新包装器，而不是把观察接口当成动态配置通道。

### 11.4 缓存为什么可能让对象一直活着？

键和值会被缓存引用，直到相应条目淘汰或清理。

如果方法缓存把 `self` 放进键，缓存也可能保留实例。

因此，对象“我已经不再使用了”不表示它马上失去所有引用。

分析内存时，需要把缓存作为引用来源之一。

### 11.5 方法缓存会自动随实例属性变化吗？

不会。

```python
from functools import lru_cache


class Multiplier:
    def __init__(self, scale):
        self.scale = scale

    @lru_cache(maxsize=16)
    def apply(self, value):
        return self.scale * value


operation = Multiplier(2)
print(operation.apply(10))
operation.scale = 3
print(operation.apply(10))
```

相同实例与相同参数仍可能命中旧结果。

如果实例可变，你需要失效政策，或者把相关不可变版本参数放入真正缓存的函数。

### 11.6 线程安全，是不是同一键永远只计算一次？

不是这个保证。

缓存维护的数据结构可以在并发使用中保持一致，但两个线程可能在首次结果尚未完成时，同时发现未命中，并各自执行原函数。

最终它们返回的值可能都正确，却执行了两次计算。

如果原函数有“必须只执行一次”的外部动作，不能仅靠 `lru_cache` 充当同步机制。

### 11.7 所以应该缓存纯计算吗？

结果只由明确参数决定、重复计算没有有害副作用的函数，通常更容易安全复用。

但“纯”是你对函数真实依赖的分析，不是加上装饰器以后自动获得的性质。

读文件、取环境配置、请求网络和读取可变实例属性，都需要另外考虑变化来源。

## 第十二章：`async def` 的函数能不能直接加 `@cache`？

### 12.1 调用异步函数，先得到什么？

```python
async def calculate(value):
    return value * 2
```

调用它先得到协程对象，结果需要通过 `await` 取得。

普通 `cache` 装饰器缓存的是调用直接返回的对象。

因此，直接这样写可能保存协程对象，而不是等待结束后的结果：

```python
# 反面示例：普通 functools 缓存不负责 await 语义。
from functools import cache


@cache
async def calculate(value):
    return value * 2
```

### 12.2 同一个协程对象可以反复 await 吗？

普通已完成协程不能当成可无限重用的结果值。

再次等待同一个已经运行过的协程，可能触发不能重用已等待协程的运行时错误。

所以，给异步函数随手套一个同步缓存装饰器，不能宣称已经实现异步结果缓存。

### 12.3 缓存 Task 就一定简单吗？

Task 涉及事件循环归属、取消、异常和并发等待等问题。

你还需要决定失败任务是否保留、取消是否影响其他等待者、应用关闭时如何清理。

这些都超出普通同步 `lru_cache` 自动处理的范围。

异步应用需要选择支持相应语义的缓存设计。

### 12.4 如果函数只是同步纯计算，放在异步程序里可以缓存吗？

可以按同步函数的规则分析。

异步应用里仍然可以调用普通同步计算函数，缓存它的普通结果。

但耗时 CPU 计算仍可能阻塞事件循环，缓存只减少命中时的重复工作，不会自动让首次计算变成非阻塞。

## 第十三章：`cached_property()`——对象的一项昂贵属性，能不能只算一次？

### 13.1 普通属性访问也可能触发计算

```python
class DataSet:
    def __init__(self, values):
        self.values = tuple(values)

    @property
    def total(self):
        print("计算 total")
        return sum(self.values)
```

每次访问 `dataset.total`，getter 都可能重新执行。

如果数据在实例生命周期里基本不变，计算又比较昂贵，就可以考虑属性缓存。

### 13.2 使用 `cached_property`

```python
from functools import cached_property


class DataSet:
    def __init__(self, values):
        self.values = tuple(values)

    @cached_property
    def total(self):
        print("计算 total")
        return sum(self.values)


dataset = DataSet([10, 20, 30])
print(dataset.total)
print(dataset.total)
```

通常只看到一次“计算 total”。

之后的读取直接取得已经保存的属性值。

### 13.3 缓存和普通 `@property` 的区别在哪里？

`cached_property` 在首次成功取得值后，把结果保存为实例上的同名普通属性。

之后同名属性读取优先得到这个保存值。

这和每次都进入只读 property getter 的机制不同。

### 13.4 我主动赋值，会怎样？

```python
dataset.total = 999
print(dataset.total)
```

通常会得到你新赋的九百九十九。

它允许普通属性写入，不是一个永久禁止覆盖的只读入口。

如果业务不允许修改，应另行设计对象接口，不能把 cached_property 当作访问控制机制。

### 13.5 如果原始数据变了呢？

```python
dataset.values = (100, 200)
print(dataset.total)
```

已经保存的值不会因为另一个属性被赋值就自动失效。

需要重新计算时，可以删除缓存属性：

```python
del dataset.total
print(dataset.total)
```

下一次读取会再次调用 getter。

### 13.6 Getter 抛异常，也会留下一个缓存值吗？

如果计算失败，没有正常得到结果，就不会按正常成功路径写入结果属性。

之后再次读取可能重新执行并再次失败。

因此，不应在 getter 中执行只能发生一次、失败后不能重复的外部动作。

### 13.7 使用 `__slots__` 的类为什么可能失败？

这个机制需要实例有可写的属性字典来保存同名结果。

只声明若干 slots 而不提供 `__dict__` 的类，不能直接假设兼容。

如果你使用其他属性缓存组织方式，还需要考虑实例可哈希、缓存生命周期和引用保留等区别。

### 13.8 Python 3.12 的锁变化是什么？

Python 3.12 以前，这个接口包含一个未公开约定的锁机制。

锁按属性组织，可能造成不同实例之间的竞争。

Python 3.12 起移除了这个锁。

因此，多个线程同时首次读取同一实例属性，getter 可能执行不止一次。

如果需要同步，应在适当位置自己安排锁，而不依赖旧版本实现细节。[版本依据：cached_property](https://docs.python.org/3.14/library/functools.html#functools.cached_property)

## 第十四章：`singledispatch()`——按第一个参数类型选择实现

### 14.1 如果格式化逻辑越来越多

```python

def describe(value):
    if isinstance(value, str):
        return f"文本：{value}"
    if isinstance(value, int):
        return f"整数：{value}"
    return repr(value)
```

少量分支这样写没有问题。

如果不同类型的处理逻辑独立、需要在不同位置扩展，单分派函数可以提供一个组织入口。

### 14.2 建立默认实现

```python
from functools import singledispatch


@singledispatch
def describe(value):
    return f"其他类型：{type(value).__name__}"
```

这个默认实现负责没有更具体注册项时的情况。

它不是自动验证所有输入正确，也不会自动把未知对象转成某种统一业务类型。

### 14.3 注册整数和字符串实现

```python
@describe.register(int)
def describe_integer(value):
    return f"整数：{value}"


@describe.register(str)
def describe_text(value):
    return f"文本长度：{len(value)}"


print(describe(10))
print(describe("Python"))
print(describe([1, 2]))
```

包装入口根据第一个参数的运行时类型选择实现。

调用者仍然统一使用 `describe(...)`。

### 14.4 `bool` 会走整数实现吗？

如果没有更具体的布尔实现，可能会沿类型继承关系选择整数实现，因为 `bool` 是 `int` 子类。

需要区别时就明确注册：

```python
@describe.register(bool)
def describe_boolean(value):
    return "布尔：真" if value else "布尔：假"
```

它不会因为你先注册整数，就永远阻止更具体的布尔分派。

### 14.5 它会根据第二个参数一起选择吗？

不会。

名称中的 single 指单分派，选择依据主要是第一个参数类型。

第二个及后续参数仍然正常传给选中的实现，但不一起决定多类型组合分派。

如果业务同时依赖多个对象的类型，可能需要显式分支或另外的设计。

### 14.6 能不能只靠类型注解注册？

现代 Python 支持按注册函数相关参数的类型注解推断。

```python
@describe.register
def describe_float(value: float):
    return f"浮点：{value:.2f}"
```

Python 3.11 还支持相应 Union 类型注解注册形式。

初学时明确写 `register(int)` 更直观，也便于分清运行时类型注册与一般静态类型提示。

### 14.7 `list[int]` 和 `list[str]` 可以作为两种分派类型吗？

不能仅靠这个工具直接按列表元素泛型区分。

运行时列表通常都是 `list` 类型。

你可以注册 `list` 实现，再在其中检查业务元素；或者定义具有明确身份的不同数据类型。

静态类型表达式不自动变成运行时所有值都携带的完整类型标签。

### 14.8 如何查看会选哪个实现？

```python
implementation = describe.dispatch(bool)
print(implementation.__name__)
print(describe.registry.keys())
```

`dispatch()` 接收类型，返回匹配实现。

`registry` 用于查看注册映射，不应把它当成可以随意写入的普通业务字典。

需要增加实现，使用公开的 `register()` 接口。

### 14.9 参数一定要按位置传吗？

这个分派入口需要取得用于选择的第一个位置参数。

不要假设只写 `describe(value=10)` 与普通函数完全一样。

为了保持清楚，按位置传入需要分派的主要对象，把其他控制选项作为明确关键字。

注册实现也应具有一致可用的附加参数契约，否则选中以后仍可能因为参数不匹配失败。

## 第十五章：`singledispatchmethod()`——方法按哪个参数分派？

### 15.1 `self` 总是同一类，拿它分派有用吗？

方法最前面通常是实例参数 `self`。

我们真正想区分的，是实例收到的业务值。

`singledispatchmethod` 按第一个非 self 或 cls 的业务参数类型组织分派。

```python
from functools import singledispatchmethod


class Converter:
    @singledispatchmethod
    def convert(self, value):
        raise TypeError(f"不支持类型：{type(value).__name__}")

    @convert.register
    def convert_integer(self, value: int):
        return str(value)

    @convert.register
    def convert_text(self, value: str):
        return value.strip()


converter = Converter()
print(converter.convert(123))
print(converter.convert("  hello  "))
```

### 15.2 和 `classmethod` 组合时要注意什么？

如果还要通过 `.register` 继续注册，`singledispatchmethod` 通常应放在外层，让这个注册入口仍可访问。

装饰器顺序会影响最终对象暴露什么能力。

不要把所有装饰器当成任意交换顺序都等价的标签。

### 15.3 它适合取代所有方法重载吗？

Python 中的方法组织还有普通分支、不同方法名、协议对象和继承等方式。

单分派适合“一个统一动作，对不同主要输入类型有独立实现”。

如果分支只差一行，或不同实现的业务参数完全不一致，强行分派可能让接口更难理解。

## 第十六章：`reduce()`——把一串值逐步归约成一个结果

### 16.1 先把过程写出来

输入：

```text
1、2、3、4
```

如果每次把已有结果和下一项相乘，就会经历：

```text
1 * 2 -> 2
2 * 3 -> 6
6 * 4 -> 24
```

`reduce()` 把这个“二元操作持续归约”的过程组织起来。

```python
from functools import reduce

print(reduce(lambda left, right: left * right, [1, 2, 3, 4]))
```

### 16.2 初始值有什么作用？

```python
print(reduce(lambda left, right: left + right, [1, 2, 3], 10))
```

从十开始，依次加入一、二、三，得到十六。

Python 3.11～3.13 中，为了兼容应把初始值作为第三个位置参数。

Python 3.14 起也支持 `initial=` 关键字形式。

### 16.3 空输入呢？

没有初始值时，无法取得第一次累计状态，空输入会报告错误。

提供初始值，则可以直接得到初始值。

```python
from functools import reduce

print(reduce(lambda left, right: left + right, [], 0))
```

### 16.4 哪些时候不要勉强使用它？

求和可以直接 `sum()`。

全部满足或任意满足，可以考虑 `all()`、`any()`。

把列表逐个拼接成大列表，可能产生大量重复复制，`chain` 等工具往往更贴近连接需求。

普通几行循环如果更清楚，就保留循环。

`reduce()` 的价值是表达确实适合二元归约的过程，不是把所有循环都压缩成一行。

## 第十七章：`cmp_to_key()`——已有两两比较函数，怎样交给排序？

### 17.1 排序通常更适合提供键

```python
students = [("小林", 95), ("小周", 88), ("小陈", 95)]
ordered = sorted(students, key=lambda item: (-item[1], item[0]))
```

一次给出每个元素的排序键，通常很直观。

如果你已经有一个接收两个对象的比较函数，返回负数、零或正数，就可以用 `cmp_to_key()` 适配。

### 17.2 一个明确的比较器

```python
from functools import cmp_to_key


def compare_scores(first, second):
    if first[1] != second[1]:
        return -1 if first[1] > second[1] else 1
    if first[0] < second[0]:
        return -1
    if first[0] > second[0]:
        return 1
    return 0


students = [("小林", 95), ("小周", 88), ("小陈", 95)]
print(sorted(students, key=cmp_to_key(compare_scores)))
```

### 17.3 为什么不能只返回 `True` 或 `False`？

旧式比较器需要表达小于、相等和大于三种关系。

布尔值只有两种，直接把 `first < second` 当返回值，不能正确表达负数方向。

这不是 `cmp_to_key()` 自动猜测的地方。

### 17.4 比较函数应该满足哪些一致性？

同一个对象与自己比较应当相等。

如果 A 小于 B，反向比较应体现相反关系。

如果 A 小于 B，B 小于 C，通常也应有 A 小于 C。

如果你的规则循环矛盾，排序工具不能替你修正业务关系。

### 17.5 为什么仍优先考虑键函数？

键函数通常更容易检查，也避免复杂比较逻辑反复执行。

`cmp_to_key()` 很适合适配已有比较器或确实依赖两者关系的规则。

它是桥接工具，不要求现代新代码优先回到旧式比较写法。

## 第十八章：`total_ordering()`——定义少量比较方法，补齐其他关系

### 18.1 一个类为什么会需要多个比较方法？

你可能希望任务对象支持 `<`、`<=`、`>` 和 `>=`。

这些关系如果使用相同排序键，手写全部方法容易重复。

`total_ordering` 可以基于一个主要顺序比较方法和相等方法，补齐其他常见关系。

### 18.2 一个完整例子

```python
from functools import total_ordering


@total_ordering
class Task:
    def __init__(self, priority, task_id):
        self.priority = priority
        self.task_id = task_id

    def _key(self):
        return self.priority, self.task_id

    def __eq__(self, other):
        if not isinstance(other, Task):
            return NotImplemented
        return self._key() == other._key()

    def __lt__(self, other):
        if not isinstance(other, Task):
            return NotImplemented
        return self._key() < other._key()

    def __repr__(self):
        return f"Task({self.priority!r}, {self.task_id!r})"


first = Task(1, "A")
second = Task(2, "B")

print(first < second)
print(first <= second)
print(first > second)
print(first >= second)
```

### 18.3 `NotImplemented` 是抛异常吗？

它是一个特殊返回值，表示这个比较实现不支持该类型组合，让 Python 继续尝试相应协议路径。

它与 `NotImplementedError` 异常不是同一个东西。

不要随手返回假来表示所有未知类型，尤其在比较协议里，这可能妨碍其他对象提供自己的匹配实现。

### 18.4 它会自动创造正确排序规则吗？

不会。

基础 `__eq__()` 和 `__lt__()` 必须使用一致的业务模型。

例如相等只看编号，顺序却把其他可变属性也算进去，就可能产生你不希望的关系。

工具减少重复方法，不替你决定对象身份和顺序。

### 18.5 排序等价与业务身份相同吗？

不一定。

一个任务的显示顺序可以按优先级和编号，而业务身份可能只看不可变编号。

如果要同时表达两种关系，应认真设计类的相等和排序契约，或者直接在 `sorted()` 中提供显示键，不让对象本身承诺全部顺序。

### 18.6 有什么代价？

自动生成的方法可能增加调用层次和性能开销，异常堆栈也更复杂。

通常先以可读和正确为主，只有实际测量表明比较是瓶颈时，再考虑手写完整关系。

它不会覆盖类或父类已经定义的方法，所以继承场景也需要核对实际生效的实现。

## 第十九章：完整实践一——相同配置写法不同，怎样仍然命中同一份计算？

### 19.1 先把实际问题说清楚

你正在写一个学习进度统计脚本。

每个科目都有一个权重，程序根据这些权重计算一项固定指标。

配置可能来自不同位置：

```python
first = {"python": 3, "linux": 2}
second = {"linux": 2, "python": 3}
```

这两份字典的插入顺序不同，但在这个业务里表达同一份权重配置。

如果直接把字典传给带 `lru_cache` 的函数，会因为字典不可哈希而失败。

如果简单使用 `str(config)` 当键，两个插入顺序又可能得到不同字符串。

我们需要先回答：这个计算究竟依赖什么？

它依赖科目名称和对应权重，不依赖字典的插入顺序。

因此，可以把配置转成按名称排列的键值对元组。

这一步是业务规范化，不只是为了躲过一个报错。

### 19.2 先设计允许的输入

为了让例子有明确边界，我们规定：

科目必须是非空字符串。

权重必须是非负整数，不接受布尔值。

空配置可以使用，它的总权重为零。

外层函数接受普通字典，内部缓存函数只接收规范化元组。

这样，验证、转换、计算三个动作可以分开观察。

### 19.3 完整程序

```python
from functools import lru_cache


def normalize_weights(weights):
    if not isinstance(weights, dict):
        raise TypeError("weights 必须是字典")
    pairs = []
    for subject, weight in weights.items():
        if not isinstance(subject, str) or not subject.strip():
            raise ValueError("科目名称必须是非空字符串")
        if type(weight) is not int or weight < 0:
            raise ValueError("权重必须是非负整数")
        pairs.append((subject, weight))
    return tuple(sorted(pairs))


@lru_cache(maxsize=64)
def score_from_key(pairs):
    print("真正汇总：", pairs)
    total = sum(weight for subject, weight in pairs)
    active = sum(weight > 0 for subject, weight in pairs)
    return total, active


def summarize_weights(weights):
    return score_from_key(normalize_weights(weights))


if __name__ == "__main__":
    score_from_key.cache_clear()
    print(summarize_weights({"python": 3, "linux": 2}))
    print(summarize_weights({"linux": 2, "python": 3}))
    print(score_from_key.cache_info())
```

运行时，真正汇总只出现一次。

主要结果是：

```text
真正汇总： (('linux', 2), ('python', 3))
(5, 2)
(5, 2)
CacheInfo(hits=1, misses=1, maxsize=64, currsize=1)
```

### 19.4 为什么不在缓存函数内部才转换字典？

因为缓存包装器在进入原函数体之前，就要处理调用参数并查找缓存。

如果你把不可哈希字典直接交给包装入口，它可能根本来不及执行函数体里的转换。

所以顺序应当是：

```text
外部字典
    ↓
验证并规范化
    ↓
不可变、可哈希的参数
    ↓
缓存查找
    ↓
未命中时进行计算
```

这也是许多缓存设计中外层函数和内部函数分开的原因。

### 19.5 为什么返回元组？

总权重与有效科目数量都是整数。

把它们放在元组里，可以避免调用者随手往结果里追加元素。

这里没有嵌套可变对象，所以这份结果比较适合共享。

如果返回字典，也能缓存，但调用者修改它就会影响之后得到同一对象的人。

你可以在外层按需要创建独立字典：

```python

def summarize_weights_as_dict(weights):
    total, active = summarize_weights(weights)
    return {"total_weight": total, "active_subjects": active}
```

每次创建的是新字典，内部仍然复用整数计算结果。

这说明缓存结果的复用与对外返回对象的独立性，可以在不同层处理。

### 19.6 如果科目大小写不一致呢？

比如 `Python` 和 `python`。

当前程序把它们当成不同名称。

是否合并，属于业务规则，不能随手调用 `.lower()` 就宣布问题解决。

如果你决定忽略大小写，还要决定同一个字典里同时出现两个形式时怎么办。

是拒绝，还是相加，还是后者覆盖前者？

这些选择会改变计算结果。

正确的规范化应当先有语义，再有代码。

### 19.7 原字典后来被修改，会污染已有键吗？

不会通过这份元组自动修改旧键，因为元组里保存的是字符串和整数这类不可变值。

后续调用外层函数时，会根据修改后的字典重新生成键。

```python
weights = {"python": 3}
print(summarize_weights(weights))
weights["python"] = 5
print(summarize_weights(weights))
```

两次分别对应不同权重快照。

但如果原字典在另一个线程转换期间被并发修改，外层转换本身仍需要调用者提供适当的同步。

“内部缓存线程安全”不会自动让外部可变输入的读取过程也变成一个原子快照。

## 第二十章：完整实践二——装饰器到底什么时候执行？用事件顺序看清楚

### 20.1 为什么背三层结构以后，仍然容易写错？

因为函数定义、装饰器构造、包装和真实调用，发生在不同时间。

如果只记外层、中层、内层的名字，很容易把运行顺序混在一起。

我们用一份事件列表记录发生了什么。

先定义一个装饰器工厂，再定义被装饰函数，最后调用两次。

### 20.2 不依赖真实时钟的演示

```python
from functools import wraps


events = []


def traced(label):
    events.append(("创建装饰器", label))

    def decorate(func):
        events.append(("包装函数", func.__name__))

        @wraps(func)
        def wrapper(*args, **kwargs):
            events.append(("调用开始", label))
            try:
                return func(*args, **kwargs)
            finally:
                events.append(("调用结束", label))

        return wrapper

    return decorate


@traced("greeting")
def greet(name):
    """根据名字生成问候语。"""
    events.append(("进入原函数", name))
    return f"你好，{name}"


if __name__ == "__main__":
    print(events)
    print(greet("小林"))
    print(greet("小周"))
    print(events)
```

函数定义完成、还没有调用 `greet()` 时，列表已经有：

```text
[('创建装饰器', 'greeting'), ('包装函数', 'greet')]
```

调用一次以后，才增加开始、原函数执行和结束这三条事件。

第二次调用不会重新创建整个装饰器工厂，而是继续使用已经得到的包装函数。

### 20.3 把 `@` 还原，会是什么？

可以按下面的关系理解：

```text
decorator = traced("greeting")
greet = decorator(原来的 greet 函数)
```

这不是要求你一定这样手写，而是帮助区分两个调用。

`traced("greeting")` 消费装饰器配置。

`decorator(原函数)` 消费需要包装的函数对象。

以后 `greet("小林")` 才是消费业务参数。

三种参数属于三个不同阶段。

### 20.4 为什么 `label` 后来还能使用？

包装函数引用了外层作用域中的 `label`。

只要相关函数对象仍然需要这个引用，Python 就会保留必要的闭包状态。

外层函数返回，不代表其内部被引用的值必须立刻消失。

这和普通“局部变量只在函数内能直接用名字访问”的说法并不矛盾。

关键在于另一个函数对象已经捕获了它需要访问的作用域单元。

### 20.5 闭包保存的是每次循环时的值吗？

这正是另一个常见疑问。

看下面的反面例子：

```python
operations = []
for factor in (1, 2, 3):
    operations.append(lambda value: factor * value)

print([operation(10) for operation in operations])
```

这里通常得到三十、三十、三十。

这些函数在调用时读取同一个后续已经变成三的名称，而不是自动为每次循环拍一张快照。

如果就是想提前固定参数，`partial()` 是很直接的表达方式：

```python
from functools import partial


def multiply(factor, value):
    return factor * value


operations = [partial(multiply, factor) for factor in (1, 2, 3)]
print([operation(10) for operation in operations])
```

现在得到：

```text
[10, 20, 30]
```

也可以用函数默认参数固定当前对象，但应该明白那是定义阶段求值的规则，而不是闭包在每次循环自动复制。

### 20.6 `wraps` 会替我保留闭包变量吗？

闭包是否存在，由函数引用了哪些外层名称决定。

`wraps` 主要维护包装对象的元信息和被包装对象关系。

它不会把一个错误绑定的循环变量自动修正，也不会把共享列表变成独立副本。

这是两个层面的机制。

## 第二十一章：完整实践三——写一个可验证的耗时记录装饰器

### 21.1 先说清楚我们要记录什么

有一个普通同步函数，你想知道每次执行花了多久。

你希望保留原函数名称、文档和返回值。

函数发生普通异常时，也希望记录这次失败，然后让原异常继续传播。

我们不把参数值全部打印出来，因为参数可能很大，也可能根本没必要出现在记录中。

这里只记录名称、耗时和是否正常完成。

### 21.2 为什么注入时钟？

真实运行可以使用 `time.perf_counter()`。

但如果写练习验证，只知道电脑“这次大概几毫秒”不方便检查。

把时钟作为装饰器工厂参数传入，就可以在练习里提供固定序列，得到稳定结果。

这不是为了让每个简单函数都复杂化，而是让时间依赖可以被明确替换。

### 21.3 完整实现

```python
from functools import wraps
from time import perf_counter


def measured(*, sink, clock=perf_counter):
    def decorate(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            started = clock()
            succeeded = False
            try:
                result = func(*args, **kwargs)
                succeeded = True
                return result
            finally:
                elapsed = clock() - started
                sink({
                    "function": func.__qualname__,
                    "elapsed": elapsed,
                    "succeeded": succeeded,
                })
        return wrapper
    return decorate


def run_timing_demo():
    records = []
    moments = iter((10.0, 10.25, 20.0, 20.5))

    @measured(sink=records.append, clock=lambda: next(moments))
    def divide(first, second):
        """执行普通除法。"""
        return first / second

    print(divide(8, 2))
    try:
        divide(8, 0)
    except ZeroDivisionError:
        print("调用者接到了除零异常")
    print(divide.__name__)
    print([(item["elapsed"], item["succeeded"]) for item in records])
    return divide, records


if __name__ == "__main__":
    run_timing_demo()
```

主要输出：

```text
4.0
调用者接到了除零异常
divide
[(0.25, True), (0.5, False)]
```

### 21.4 为什么先保存 `result`？

我们需要在返回之前把成功标记设为真。

如果直接在 `try` 中写 `return func(...)`，也可以执行 `finally`，但成功标记需要用其他方式安排。

这里的三步把“函数返回”“标记成功”“把结果交还调用者”写得清楚。

`finally` 不会因为前面 `return` 就被跳过。

### 21.5 如果 `sink` 自己失败呢？

这个简单版本要求接收记录的 `sink` 按约定正常工作。

如果 `sink` 在 `finally` 中抛异常，它可能取代原本要传播的异常，或使原本成功的调用变成失败。

因此，不能笼统宣称这个包装器在所有记录器故障下都完全透明。

在正式程序中，你要决定记录失败是必须阻断业务，还是可以另行处理。

如果要保护原业务异常，应该明确捕获记录过程预期会出现的异常，并安排可观察的处理方式。

不要写一个覆盖所有情况的 `except: pass`，然后假装记录永远成功。

### 21.6 如果被装饰的是 `async def` 呢？

当前包装器是普通同步函数。

它记录的是调用得到协程对象这一步的耗时，而不是协程真正执行完成的耗时。

要记录异步执行，需要 `async def wrapper` 并在其中 `await func(...)`。

这再次说明：保留函数名字不等于保留了所有执行语义。

### 21.7 可以测量装饰器本身的额外开销吗？

可以，但要用适合微基准的方式，区分单次时钟噪声和长期平均。

这里的固定时钟例子是行为验证，不是性能实验。

看到 `0.25` 不能得出真实机器一定需要四分之一秒。

程序测试和性能测量回答不同问题，不要把人为控制的结果当成实际跑分。

## 第二十二章：完整实践四——文件更新后，怎样避免读到旧的解析结果？

### 22.1 为什么“按文件路径缓存”容易过期？

假设你把配置保存在：

```text
project/settings.json
```

程序第一次读取这个路径，得到主题 `light`。

后来你把文件内容改为 `dark`。

路径字符串没有变化，因此按路径缓存的函数仍可能返回第一次的结果。

文件路径是位置标识，不是内容版本。

要解决这个问题，需要先决定你究竟希望省掉什么工作。

本例只缓存文本解析和验证，不缓存读取磁盘这一步。

每次读取当前文件文本，再用文本本身作为解析缓存键。

### 22.2 这是不是所有场景里最省内存的办法？

不是。

键里保存完整文本，会占用相应空间。

但对于体积较小、重复解析较多的配置文件，这个模型很容易理解和验证。

容量限制只能限制条目数，因此我们还给文本长度一个明确上限。

这个上限按字符计算，不是严格的字节内存预算。

### 22.3 完整程序

```python
import json
from functools import lru_cache
from pathlib import Path
from tempfile import TemporaryDirectory


@lru_cache(maxsize=16)
def parse_preferences(text):
    if len(text) > 100_000:
        raise ValueError("配置文本过长")
    data = json.loads(text)
    if not isinstance(data, dict):
        raise ValueError("配置顶层必须是对象")
    if set(data) != {"theme", "page_size"}:
        raise ValueError("配置字段必须恰好是 theme 和 page_size")
    theme = data["theme"]
    page_size = data["page_size"]
    if theme not in ("light", "dark"):
        raise ValueError("theme 只能是 light 或 dark")
    if type(page_size) is not int or not 1 <= page_size <= 100:
        raise ValueError("page_size 必须是 1 到 100 的整数")
    return theme, page_size


def load_preferences(path):
    text = Path(path).read_text(encoding="utf-8")
    return parse_preferences(text)


def run_preferences_demo():
    parse_preferences.cache_clear()
    with TemporaryDirectory() as directory:
        path = Path(directory) / "settings.json"
        path.write_text('{"theme":"light","page_size":20}', encoding="utf-8")
        print(load_preferences(path))
        print(load_preferences(path))
        path.write_text('{"theme":"dark","page_size":30}', encoding="utf-8")
        print(load_preferences(path))
        print(parse_preferences.cache_info())


if __name__ == "__main__":
    run_preferences_demo()
```

主要输出：

```text
('light', 20)
('light', 20)
('dark', 30)
CacheInfo(hits=1, misses=2, maxsize=16, currsize=2)
```

### 22.4 为什么相同文件仍然读取两次？

因为本例明确只复用解析结果。

第二次读取可以确认当前内容仍是原来那段文本。

然后解析缓存命中，省掉 JSON 解析和验证。

如果你的文件很小、解析很快，缓存可能并不值得。

本例的重点是把依赖关系设计正确，而不是承诺每个配置文件都应该缓存。

### 22.5 换成修改时间和文件大小作为键可以吗？

可以作为某些应用接受的变化指纹，但不能把它说成严格等价于内容身份。

不同内容可能具有相同大小。

时间戳也涉及精度、文件系统行为和主动设置。

而且取得属性与读取内容之间，文件可能继续变化。

如果你需要严格的一致性保证，需要更完整的更新与读取协议，例如由写入方提供不可变版本文件，再明确切换版本引用。

不要仅靠“我看了两次 `stat`”就宣称处理了所有并发更新问题。

### 22.6 文本只是空格改变，会命中吗？

本例不会自动合并。

JSON 里空格或字段排列改变，文本键就改变，即便解析后的值相同。

这是一项有意选择：我们缓存的是相同文本的解析。

如果先解析再规范化成别的键，解析这一步已经执行，优化目标就改变了。

理解缓存省掉哪一段工作，比盲目追求更高命中率重要。

### 22.7 验证代码为什么用 `type(page_size) is int`？

因为布尔值是整数子类。

`isinstance(True, int)` 为真，但业务上我们不希望 `true` 被当成页面条数一。

这是配置输入规则，不是要求所有 Python 程序都拒绝整数子类。

针对明确数据协议，精确类型检查有时是有意义的。

### 22.8 解析失败以后，下次能重新尝试吗？

可以。

这个函数没有把失败结果主动转成普通返回值缓存。

`json.loads()` 或验证抛出的异常不会作为成功结果保存。

修正文件后，下次读取会拿到新的文本，并重新解析。

不过读取文件的异常、编码错误和解析错误属于不同阶段，调用者应按需要给出清楚的提示，不能全部称为“缓存坏了”。

## 第二十三章：完整实践五——用两个线程亲眼看见“同一参数计算两次”

### 23.1 为什么不只加一句“可能重复”就结束？

因为“线程安全”这个词很容易被理解成“所有工作都排队，只会做一次”。

我们可以写一个受控例子，让两个线程在首次计算完成之前都进入函数体。

这样就能把缓存容器的一致性，与业务计算执行次数分开观察。

这里不用随机等待几秒钟来碰运气，而是使用一个两方屏障。

两个线程都到达屏障，才一起继续。

### 23.2 完整程序

```python
from concurrent.futures import ThreadPoolExecutor
from functools import lru_cache
from threading import Barrier, Lock


def demonstrate_duplicate_miss():
    barrier = Barrier(2)
    counter_lock = Lock()
    calls = 0

    @lru_cache(maxsize=8)
    def calculate(value):
        nonlocal calls
        with counter_lock:
            calls += 1
        barrier.wait(timeout=5)
        return value * value

    with ThreadPoolExecutor(max_workers=2) as pool:
        first = pool.submit(calculate, 7)
        second = pool.submit(calculate, 7)
        results = [first.result(timeout=10), second.result(timeout=10)]

    print(results)
    print("函数体执行次数：", calls)
    print("之后再调用：", calculate(7))
    print(calculate.cache_info())
    return results, calls, calculate.cache_info()


if __name__ == "__main__":
    demonstrate_duplicate_miss()
```

在正常完成的演示里，你会看到：

```text
[49, 49]
函数体执行次数： 2
之后再调用： 49
CacheInfo(hits=1, misses=2, maxsize=8, currsize=1)
```

### 23.3 屏障在这里起什么作用？

第一个线程进入原函数后，停在屏障处，暂时不能返回结果。

所以缓存里还没有可供第二个线程命中的完整结果。

第二个线程用相同参数调用，仍然未命中，也进入原函数。

两个线程都到了以后，屏障放行。

这样，我们不必依靠“电脑刚好够慢”来复现重复计算。

### 23.4 为什么计数器还要单独加锁？

计数器是我们自己写的共享状态。

缓存内部如何维护自己的字典，不等于它会替原函数里的每一个变量加锁。

我们使用 `Lock`，让这项观察动作有明确的并发语义。

锁只保护增加计数的短代码段，没有把整个计算包起来。

如果把锁一直持有到屏障等待结束，第一个线程会阻止第二个线程进入，反而可能互相等待。

同步工具的作用范围必须根据执行流程设计。

### 23.5 最后为什么只有一条缓存记录？

两次调用的键相同。

重复执行不意味着最终必须保留两个同键条目。

因此，未命中次数可以是二，而当前条目数是一。

这个例子也说明：只看最终字典大小，不能推断原函数究竟执行过几次。

### 23.6 如果计算真的只能执行一次呢？

例如创建一份不可重复的外部记录，或触发一次付款。

首先应该把“只发生一次”的正确性放在相应业务协议、事务或幂等设计中。

不要把进程里的函数缓存当作外部系统的唯一保护。

如果只是同一进程里昂贵纯计算的合并，可以另外设计按键的进行中任务与同步机制。

但那需要处理异常、等待者唤醒、锁表清理和递归调用等问题。

简单地给缓存外面套一把全局锁，虽然可能减少重复，却也会串行化不同键的计算，不一定适合真实负载。

### 23.7 `cache_clear()` 会取消已经开始的计算吗？

它不是任务取消接口。

如果另一个线程正在执行原函数，清理缓存不表示该函数马上停止，也不能据此推断后续一定没有新结果写回。

需要在运行中更新配置时，应定义明确的版本边界，而不是把“清了一下缓存”当成所有并发工作都结束的信号。

缓存控制和任务生命周期应分开设计。

## 第二十四章：完整实践六——成绩报告的属性缓存，什么时候该删除？

### 24.1 一个很实际的对象

你有一组成绩，需要多次显示平均分、最高分、最低分。

同一组数据不变时，报告页面反复访问这些统计量，不需要每次重新遍历。

但你也允许用户用一整组新成绩替换旧成绩。

因此，规则可以明确为：

创建对象时保存不可变成绩快照。

首次访问统计属性时计算。

替换整组成绩时，验证成功后再更新输入，并删除已有缓存。

不要让调用者拿着内部列表随意修改，而你却不知道什么时候该失效。

### 24.2 完整程序

```python
from functools import cached_property
from math import isfinite


def validated_scores(values):
    result = []
    for value in values:
        if isinstance(value, bool) or not isinstance(value, (int, float)):
            raise TypeError("成绩必须是普通整数或浮点数")
        try:
            number = float(value)
        except OverflowError as error:
            raise ValueError("成绩超出可处理范围") from error
        if not isfinite(number) or not 0 <= number <= 100:
            raise ValueError("成绩必须是 0 到 100 的有限数值")
        result.append(number)
    if not result:
        raise ValueError("至少需要一项成绩")
    return tuple(result)


class GradeReport:
    def __init__(self, values):
        self._scores = validated_scores(values)
        self.calculations = 0

    @property
    def scores(self):
        return self._scores

    @cached_property
    def statistics(self):
        self.calculations += 1
        return (
            sum(self._scores) / len(self._scores),
            min(self._scores),
            max(self._scores),
        )

    def replace_scores(self, values):
        replacement = validated_scores(values)
        self._scores = replacement
        self.__dict__.pop("statistics", None)


def run_grade_demo():
    report = GradeReport([80, 90, 100])
    print(report.statistics)
    print(report.statistics)
    print("计算次数：", report.calculations)
    report.replace_scores([60, 80])
    print(report.statistics)
    print("计算次数：", report.calculations)
    return report


if __name__ == "__main__":
    run_grade_demo()
```

输出：

```text
(90.0, 80.0, 100.0)
(90.0, 80.0, 100.0)
计算次数： 1
(70.0, 60.0, 80.0)
计算次数： 2
```

### 24.3 为什么先验证，再替换？

如果新数据无效，我们希望原来的合法报告仍然能使用。

所以先把新输入转换为 `replacement`。

只有转换成功，才修改对象状态。

这样，输入含负分或空序列时，不会把对象更新到半成品状态。

这是普通程序设计中很实用的一条顺序原则，与有没有缓存都有关。

### 24.4 为什么不是直接 `del self.statistics`？

如果还从未访问过这个属性，实例上可能没有已缓存的同名条目。

直接删除可能报告属性不存在。

从 `__dict__` 中使用带默认值的 `pop`，表达“有就删除，没有也正常”。

这段代码依赖这个类采用普通实例属性字典，正好符合 `cached_property` 的机制。

如果你重构了存储方式，也应该一起重新考虑失效实现。

### 24.5 为什么成绩保存为元组？

外部拿到 `scores` 后，不能像列表那样直接追加一项。

这让“更新数据必须经过替换方法”更容易遵守。

元组不是绝对防护墙，Python 调用者仍然可能主动修改带下划线的内部属性。

但这个接口把推荐使用路径表达清楚，避免日常代码无意绕过失效规则。

### 24.6 如果用户给 `statistics` 手动赋值呢？

`cached_property` 本身允许覆盖。

因此，这个简单类仍然可以被主动写入同名属性。

如果你希望公开只读统计量，普通 `property` 加内部缓存字段可能更合适。

工具选择要服从接口要求，而不是看到“缓存属性”就一定使用这个装饰器。

### 24.7 为什么这里没有加锁？

这个实践按单线程使用设计。

如果多个线程同时替换数据和读取统计，单靠删除属性不能保证读取的是完整一致的一组状态。

需要同步时，应保护数据更新与读取所依赖的整个状态关系。

只给计数器加锁，或者只把删除动作加锁，都不能自动解决读到新旧状态交错的问题。

这比笼统写一句“多线程注意安全”更具体：你要保护的，是输入快照与对应计算结果之间的关系。

## 第二十五章：完整实践七——为一组类型组织统一的文本展示入口

### 25.1 先不要把需求扩大成万能序列化器

你正在写一个终端中的小工具。

它需要展示整数、布尔值、文本，以及一个自定义任务对象。

你不需要导出所有 Python 对象，也不需要递归遍历任意复杂结构。

因此，我们建立一个统一的 `display()` 入口，再按主要输入类型注册实现。

格式化选项只有一个：是否使用紧凑形式。

每个实现都接受同样的附加关键字参数，避免分派成功后参数契约又不一致。

### 25.2 完整程序

```python
from functools import singledispatch


class LearningTask:
    def __init__(self, title, completed=False):
        if not isinstance(title, str) or not title.strip():
            raise ValueError("任务标题必须是非空文本")
        if type(completed) is not bool:
            raise TypeError("completed 必须是布尔值")
        self.title = title
        self.completed = completed


@singledispatch
def display(value, *, compact=False):
    raise TypeError(f"没有注册展示方式：{type(value).__name__}")


@display.register(int)
def display_integer(value, *, compact=False):
    return str(value) if compact else f"整数：{value}"


@display.register(bool)
def display_boolean(value, *, compact=False):
    word = "是" if value else "否"
    return word if compact else f"布尔值：{word}"


@display.register(str)
def display_text(value, *, compact=False):
    return value.strip() if compact else f"文本：{value}"


@display.register(LearningTask)
def display_task(value, *, compact=False):
    state = "完成" if value.completed else "待办"
    return value.title if compact else f"[{state}] {value.title}"


if __name__ == "__main__":
    values = [10, True, "  Python  ", LearningTask("阅读 functools")]
    for value in values:
        print(display(value))
    print(display(LearningTask("复习缓存", True), compact=True))
```

输出：

```text
整数：10
布尔值：是
文本：  Python  
[待办] 阅读 functools
复习缓存
```

文本行的原始空格被保留；紧凑形式才主动去掉两侧空白。

这是一项清楚的输入输出政策，不是 `singledispatch` 自动做的处理。

### 25.3 为什么注册顺序不决定布尔值是否变成整数？

分派会考虑运行时类型及相关继承关系。

已经存在更具体的 `bool` 实现时，会选择它，而不是简单按你书写代码的先后从上往下碰第一个 `isinstance`。

但这不代表任意多个抽象基类组合都一定存在唯一明显的答案。

复杂注册中可能存在无法唯一确定的关系，需要更具体的注册或更明确的模型。

初学时先把对象的实际类型、直接父类和默认处理路径弄清楚，就能解决大多数日常问题。

### 25.4 新类型可以在别处增加吗？

可以，只要注册到同一个通用函数对象上。

```python
class Tag:
    def __init__(self, text):
        self.text = text


@display.register(Tag)
def display_tag(value, *, compact=False):
    return value.text if compact else f"标签：{value.text}"
```

不需要重写原来所有类型的展示代码。

但注册代码需要实际被执行，例如对应模块已经被导入。

你把注册函数写在一个从未导入的文件里，Python 不会自动扫描整个项目替你完成注册。

### 25.5 如果某个插件又注册了同一种类型呢？

后续注册可能改变这个类型选择到的实现。

因此，注册表也是一份运行中的全局组织状态。

项目很大时，应明确由谁拥有某个类型的默认展示方式，避免多个模块在导入顺序中互相覆盖。

必要时使用不同的通用函数表达不同业务视角，而不是把所有展示需求都塞进同一注册表。

### 25.6 默认实现为什么直接抛异常？

因为本例希望不支持的类型立刻暴露出来。

如果默认写 `str(value)`，未知对象会继续输出，但也可能把遗漏的业务支持悄悄掩盖。

两种设计都可能合理，取决于工具希望严格还是宽松。

这里明确选择严格模式，调用者可以知道哪些输入属于协议。

### 25.7 使用缓存包装展示函数，会发生什么？

首先，自定义对象通常可能可变，内容改变后结果可能过期。

其次，缓存键里包含什么，取决于装饰器套用位置。

再次，新增注册后，旧的外层结果缓存不一定自动知道业务实现已经变了。

所以，不应该仅因为展示函数返回字符串就直接加缓存。

分派解决实现选择，缓存解决结果复用；组合时还要说明注册变化与对象变化怎样影响结果。

## 第二十六章：装饰器叠放顺序，为什么会改变你看到的调用次数？

### 26.1 先用两种顺序做实验

一个装饰器负责记录“经过这里一次”。

另一个装饰器负责缓存结果。

如果记录在缓存外面，每次经过公共入口都会记录。

如果记录在缓存里面，缓存命中时就不进入记录包装器。

```python
from functools import lru_cache, wraps


def counted(events):
    def decorate(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            events.append(func.__name__)
            return func(*args, **kwargs)
        return wrapper
    return decorate


def compare_decorator_order():
    outer_events = []
    inner_events = []

    @counted(outer_events)
    @lru_cache(maxsize=8)
    def first(value):
        return value * 2

    @lru_cache(maxsize=8)
    @counted(inner_events)
    def second(value):
        return value * 2

    first(10)
    first(10)
    second(10)
    second(10)
    print(len(outer_events), len(inner_events))
    return first, second, outer_events, inner_events


if __name__ == "__main__":
    compare_decorator_order()
```

输出：

```text
2 1
```

### 26.2 为什么从下面往上包装？

多个装饰器可以理解成函数调用嵌套。

```text
@A
@B
def operation(...): ...
```

大致形成：

```text
operation = A(B(原函数))
```

真正调用时，最外面的 A 入口先被经过。

“应用装饰器的组合顺序”与“调用时从外向内经过包装器”要一起看，才不会混乱。

### 26.3 `wraps` 会把缓存控制方法自动完整转发吗？

不要这样假设。

普通包装函数保留了元信息与 `__wrapped__`，不等于它自动代理被包装对象全部方法。

如果最外层是普通函数，你不一定能直接写 `first.cache_info()`。

可以明确沿 `__wrapped__` 找到内部缓存包装器，或者在设计装饰器时显式提供所需控制接口。

但不建议业务代码到处硬编码好几层 `.__wrapped__.__wrapped__`，因为包装结构一改就容易出错。

稳定的公开控制入口通常比依赖层数更容易维护。

### 26.4 `inspect.signature()` 为什么仍然像原函数？

常见检查行为会沿 `__wrapped__` 查看原函数签名。

这对帮助信息很有用。

你也可以用 `follow_wrapped=False` 观察包装器本身的签名。

```python
import inspect

first, second, outer_events, inner_events = compare_decorator_order()
print(inspect.signature(first))
print(inspect.signature(first, follow_wrapped=False))
```

通常前者显示业务参数 `value`，后者显示包装器的 `*args, **kwargs`。

元信息帮助工具理解你的意图，但运行时到底执行哪一层，仍需要看真实调用路径。[官方说明：inspect.signature](https://docs.python.org/3.12/library/inspect.html#inspect.signature)

## 第二十七章：完整实践八——目录统计可以归约，但要看清状态和输入

### 27.1 什么时候 `reduce` 的状态模型比较自然？

假设输入不是文件内容，而是一组已经取得的文件信息。

每项包含名称、字节数和是否跳过。

我们希望得到文件数量、总字节数和最大文件大小。

这三个数共同构成当前累计状态。

每收到一项，就把旧状态和当前记录归约成新状态。

这是 `reduce` 可以清楚表达的一类任务。

### 27.2 完整实现

```python
from functools import reduce


def combine_file_state(state, record):
    count, total_size, largest_size = state
    name, size, skipped = record
    if not isinstance(name, str):
        raise TypeError("文件名必须是文本")
    if type(size) is not int or size < 0:
        raise ValueError("文件大小必须是非负整数")
    if type(skipped) is not bool:
        raise TypeError("skipped 必须是布尔值")
    if skipped:
        return state
    return count + 1, total_size + size, max(largest_size, size)


def summarize_files(records):
    return reduce(combine_file_state, records, (0, 0, 0))


if __name__ == "__main__":
    records = [
        ("a.txt", 120, False),
        ("cache.tmp", 999, True),
        ("b.txt", 80, False),
    ]
    print(summarize_files(records))
    print(summarize_files([]))
```

输出：

```text
(2, 200, 120)
(0, 0, 0)
```

### 27.3 为什么初始状态的最大值用零？

因为本例文件大小明确非负。

没有文件时，我们约定最大大小为零。

如果统计的数值可以为负，或需要区分“确实存在一个零”和“根本没有元素”，初始状态就要重新设计，例如使用 `None` 表示尚未出现有效项。

初始值不是随便找个看起来方便的数字，它需要符合模型。

### 27.4 跳过项为什么仍然验证？

这里约定所有输入记录都必须满足同一结构与类型要求，跳过只影响汇总。

如果你希望跳过项完全不解析大小，可以把判断提前，但那是另一种输入协议。

无论选择哪种，都应该写清楚，否则读者会不知道无效数据究竟在什么阶段被接受或拒绝。

### 27.5 为什么没有真的扫描磁盘？

因为本例要把“取得文件信息”和“汇总已知记录”分开。

磁盘扫描可能失败，也可能遇到文件变化。

先有一份明确记录，再讨论归约，可以让这个函数容易测试和复用。

接入 `pathlib` 时，可以由另一层负责产生同样的记录格式，再把它交给汇总函数。

不要为了演示 `reduce`，让所有路径、异常、输出和累计状态挤进一个 lambda。

### 27.6 用普通循环会不会更好？

完全可能。

你可以用同一个 `combine_file_state()` 写循环：

```python

def summarize_files_with_loop(records):
    state = (0, 0, 0)
    for record in records:
        state = combine_file_state(state, record)
    return state
```

这个版本对很多初学者更直观。

理解 `reduce` 以后，你多了一种表达选择，不是失去了写循环的资格。

如果累计过程需要 `break`、多个异常分支或复杂日志，循环通常更容易维护。

## 第二十八章：完整实践九——排序规则相同，就一定应该让对象支持全部比较吗？

### 28.1 先区分两种需求

第一种需求是“这个页面按优先级展示任务”。

这往往只需要给 `sorted()` 提供键。

第二种需求是“这个数据类型本身定义了稳定的总顺序”。

这才更接近 `total_ordering` 的使用场景。

比如一个简化的版本编号，只由主版本、次版本和修订号构成。

本例不处理预发布标签、发行阶段、构建元信息，也不声称替代正式包版本规则。

我们只处理三个非负整数形成的版本三元组。

### 28.2 完整实现

```python
from functools import total_ordering


@total_ordering
class SimpleVersion:
    def __init__(self, major, minor, patch):
        parts = (major, minor, patch)
        if any(type(part) is not int or part < 0 for part in parts):
            raise ValueError("版本的三个部分必须是非负整数")
        self._parts = parts

    @property
    def parts(self):
        return self._parts

    def __eq__(self, other):
        if type(other) is not SimpleVersion:
            return NotImplemented
        return self._parts == other._parts

    def __lt__(self, other):
        if type(other) is not SimpleVersion:
            return NotImplemented
        return self._parts < other._parts

    def __repr__(self):
        major, minor, patch = self._parts
        return f"SimpleVersion({major}, {minor}, {patch})"


def run_version_demo():
    versions = [SimpleVersion(1, 10, 0), SimpleVersion(1, 2, 9), SimpleVersion(2, 0, 0)]
    print(sorted(versions))
    print(SimpleVersion(1, 2, 9) <= SimpleVersion(1, 2, 9))
    print(SimpleVersion(2, 0, 0) > SimpleVersion(1, 99, 99))
    print(SimpleVersion(1, 2, 3) == "1.2.3")


if __name__ == "__main__":
    run_version_demo()
```

主要输出：

```text
[SimpleVersion(1, 2, 9), SimpleVersion(1, 10, 0), SimpleVersion(2, 0, 0)]
True
True
False
```

### 28.3 为什么不能直接比较版本字符串？

文本比较依据字符序列。

字符串 `"1.10.0"` 和 `"1.2.9"` 的字符次序，不等于对应整数三元组的版本次序。

把业务结构解析成整数，是为了让比较规则与含义一致。

`total_ordering` 只补比较方法；它不会替你完成字符串解析，更不会自动知道一个点分字符串代表版本。

### 28.4 为什么这里使用精确类型判断？

为了把这个教学类型的比较协议限制在自身。

如果允许子类参与，还要考虑子类是否扩展了影响相等或排序的字段。

这个例子没有承诺支持这样的继承扩展。

它不是规定所有类的比较都必须用 `type(other) is ...`，而是在明确边界下选择一种一致的规则。

### 28.5 对未知类型，为什么等号结果是假，小于却可能报错？

相等比较在双方都不支持时，有相应的回退行为。

顺序比较没有“随便排一个位置”的通用默认规则。

因此，下面的顺序比较预期失败：

```python
# 预期 TypeError：没有定义版本对象与字符串之间的顺序。
print(SimpleVersion(1, 2, 3) < "1.2.4")
```

这比悄悄把任何对象都转换成字符串再比较更容易暴露误用。

### 28.6 对象可以当缓存键了吗？

还不行直接这么推断。

我们定义了相等比较，但没有定义匹配的哈希方法。

Python 会对这类情况采取相应的不可哈希处理，避免随意产生与相等规则矛盾的哈希行为。

如果要作为缓存键，最简单的是明确传入 `.parts` 这个整数元组。

你也可以进一步设计真正不可变、相等与哈希一致的类型，但不能只因加上 `total_ordering` 就宣称这些条件已经满足。

### 28.7 `total_ordering` 与 `dataclass(order=True)` 怎么选？

如果对象主要是一组字段，且字段顺序恰好符合比较语义，数据类可以减少很多定义。

如果需要手写比较规则，`total_ordering` 允许你提供核心比较实现。

两者都需要先明确字段含义、相等规则和可变性。

这个选择不影响一条最基础的判断：只有在对象确实适合承诺顺序时，才把顺序写进类型本身。

## 第二十九章：完整实践十——异步协程被缓存后，到底缓存了什么？

### 29.1 不靠猜测，直接观察对象身份

我们让一个异步函数返回数字的两倍。

给它加普通 `cache`，然后连续取两次相同参数的调用结果。

先不等待，看看是不是同一个对象。

再等待第一次，最后尝试等待第二次。

### 29.2 演示程序

```python
import asyncio
from functools import cache


@cache
async def cached_coroutine(value):
    return value * 2


async def demonstrate_coroutine_cache():
    cached_coroutine.cache_clear()
    first = cached_coroutine(5)
    second = cached_coroutine(5)
    print("同一个协程对象：", first is second)
    print("第一次等待：", await first)
    try:
        await second
    except RuntimeError as error:
        print(type(error).__name__)
    finally:
        cached_coroutine.cache_clear()


if __name__ == "__main__":
    asyncio.run(demonstrate_coroutine_cache())
```

主要输出：

```text
同一个协程对象： True
第一次等待： 10
RuntimeError
```

这个程序专门演示错误组合，不能直接作为异步缓存模板使用。

### 29.3 第一次为什么还能成功？

第一次得到的协程对象还没被执行完。

`await first` 正常驱动它完成，并得到十。

问题不是“异步函数永远不能返回正确结果”，而是包装器保存的是协程对象本身，而你真正想复用的是完成后的十。

这两个对象属于不同层次。

### 29.4 换成无参数函数会更安全吗？

不会。

无参数只是让所有调用更容易命中同一个键。

如果缓存对象本身有一次性执行状态，问题仍然存在。

相同道理也适用于前面已经耗尽的生成器对象。

缓存能保存引用，却不会自动把有状态协议变成纯数据。

### 29.5 业务结果已经得到以后，能不能存入普通字典？

可以自己存，但需要设计键、容量、失败、并发和失效规则。

如果多个调用同时未命中，是否允许分别发起同样的 I/O？

某个等待者取消时，其他等待者怎么办？

任务属于哪个事件循环，应用关闭时如何清理？

这些都是异步结果缓存的真实问题。

理解这些以后，再选择成熟实现或编写范围明确的版本，比误以为普通 `@cache` 自动等待结果更稳妥。

## 第三十章：练习题——先预测行为，再运行，再解释原因

下面每题都可以在独立脚本或交互环境中完成。

重点不只是得到一行结果，还要能说清楚：绑定发生在何时、缓存键包含什么、原函数有没有被真正调用。

### 30.1 练习一：固定编码函数的参数

你有一个函数：

```python

def encode_text(text, encoding="utf-8", errors="strict"):
    return text.encode(encoding, errors)
```

要求创建 `encode_utf8`，默认使用 UTF-8，并保留调用者传入 `errors` 的能力。

再回答：调用者是否还能使用关键字覆盖编码？

参考答案：

```python
from functools import partial

encode_utf8 = partial(encode_text, encoding="utf-8")
print(encode_utf8("你好"))
print(encode_utf8("abc", encoding="ascii"))
```

可以覆盖预绑定关键字。

如果接口要求编码不可变，应写一个只暴露文本与错误处理选项的新函数，而不是依赖 `partial` 拒绝覆盖。

本题提醒你：预配置与强制限制不是一回事。

### 30.2 练习二：循环里生成三种折扣计算器

折扣依次为原价的百分之九十、百分之八十、百分之七十。

不要使用读取同一个循环变量的晚绑定 lambda。

参考答案：

```python
from functools import partial
from decimal import Decimal


def discounted(rate, price):
    return price * rate


rates = [Decimal("0.9"), Decimal("0.8"), Decimal("0.7")]
operations = [partial(discounted, rate) for rate in rates]
print([operation(Decimal("100")) for operation in operations])
```

结果对应九十、八十、七十。

这里使用十进制定点含义明确的输入，避免把二进制浮点显示误差混进绑定机制练习。

真实金额还要规定舍入位数与方式，这不是 `partial` 自动完成的内容。

### 30.3 练习三：包装器忘记 `return` 会怎样？

```python

def broken(func):
    def wrapper(*args, **kwargs):
        func(*args, **kwargs)
    return wrapper
```

即便原函数返回一百，调用包装函数也可能得到 `None`。

请修复，并保留名称和说明。

参考答案：

```python
from functools import wraps


def forwarded(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

`wraps` 不会帮你补上遗漏的 `return`。

这个错误来自真实控制流，而不是元信息。

### 30.4 练习四：缓存能区分空结果与未命中吗？

编写一个总是返回 `None` 的缓存函数，记录真正执行次数，调用两次后应只执行一次。

参考答案：

```python
from functools import cache

calls = []


@cache
def missing_resource(name):
    calls.append(name)
    return None


assert missing_resource("a") is None
assert missing_resource("a") is None
print(len(calls), missing_resource.cache_info().hits)
```

输出：

```text
1 1
```

假值不是未命中的同义词。

如果自己用字典实现缓存，应该测试键是否存在，或使用专门的缺失哨兵。

### 30.5 练习五：异常被缓存了吗？

编写一个总是抛 `ValueError` 的缓存函数，调用两次并捕获错误，观察执行次数。

参考答案：

```python
from functools import cache

attempts = []


@cache
def fail(value):
    attempts.append(value)
    raise ValueError("演示失败")


for unused in range(2):
    try:
        fail(1)
    except ValueError:
        pass

print(len(attempts), fail.cache_info().currsize)
```

输出：

```text
2 0
```

这里的 `pass` 仅用于练习里明确捕获预期错误，方便继续观察计数。

正式业务中是否忽略错误，需要按照实际要求决定。

### 30.6 练习六：列表结果为什么被别人改了？

要求内部缓存一份零到 n 减一的数据，但每个调用者得到独立列表。

参考答案：

```python
from functools import lru_cache


@lru_cache(maxsize=32)
def number_tuple(size):
    if type(size) is not int or not 0 <= size <= 10_000:
        raise ValueError("size 必须是 0 到 10000 的整数")
    return tuple(range(size))


def number_list(size):
    return list(number_tuple(size))


first = number_list(3)
first.append(99)
print(number_list(3))
```

输出仍然是零、一、二。

外层复制并不免费，但它表达了每位调用者都可独立修改结果的协议。

这里限制输入大小，也避免一次请求建立极大的缓存对象。

### 30.7 练习七：LRU 到底淘汰谁？

容量为二，按顺序调用键 A、B、A、C、B。

预测命中和未命中次数。

参考答案是命中一次、未命中四次。

A 在第三次调用重新成为最近使用项，所以插入 C 时淘汰 B。

最后的 B 需要重新计算。

如果你只按插入顺序推断，会误把 A 淘汰，这恰好是普通先进先出与 LRU 的区别。

### 30.8 练习八：修改实例属性后，缓存值为什么不变？

对第十一章的 `Multiplier`，提出两种修复方式。

一种是明确在改变倍数后清理缓存，但要注意方法包装器可能服务同类多个实例，清理影响不一定局限当前实例。

另一种是把计算提取成按 `scale` 和 `value` 缓存的纯函数。

参考实现：

```python
from functools import lru_cache


@lru_cache(maxsize=64)
def scaled_value(scale, value):
    return scale * value


class Multiplier:
    def __init__(self, scale):
        self.scale = scale

    def apply(self, value):
        return scaled_value(self.scale, value)
```

此时倍数变化会产生不同键。

这个方案还避免内部缓存把每个实例本身作为键保留下来。

### 30.9 练习九：属性从未计算过，清理会不会失败？

创建一个 `GradeReport`，立即替换成绩，之后才访问统计量。

参考答案：不会因为“尚无 statistics 缓存条目”而失败。

原因是 `pop("statistics", None)` 允许键不存在。

但是新成绩本身仍然需要通过验证。

“允许尚未缓存”与“允许任意无效输入”是不同条件。

### 30.10 练习十：布尔值分派到哪里？

只注册整数实现时，调用 `display(True)` 会怎样？

在没有更具体注册、也没有其他冲突的基本模型下，会沿继承关系匹配整数实现。

再注册布尔实现后，则使用布尔实现。

本题要解释的是 `bool` 与 `int` 的类型关系，不是 `True` 显示成一的文本习惯。

### 30.11 练习十一：第一参数以外的类型能决定分派吗？

```text
convert(value, target_type)
```

如果你希望根据 `value` 和 `target_type` 的组合选择实现，单靠 `singledispatch` 不够。

它不会自动成为双分派。

可以让第一参数实现内部继续处理目标类型，或者把不同转换操作设计成不同入口。

参考答案不是某个神奇参数，而是识别工具实际提供的选择维度。

### 30.12 练习十二：空归约为什么需要初始值？

要求把一组集合求交集，输入可能为空。

先不要急着写代码。

空输入的交集在你的业务里应该是什么？

如果没有定义全集，直接凭空给一个空集合可能并不等于数学上所有集合交的中性元素。

因此，可以规定空输入拒绝，并对非空输入从第一项复制后继续求交。

参考实现：

```python

def intersect_all(groups):
    iterator = iter(groups)
    try:
        first = next(iterator)
    except StopIteration:
        raise ValueError("至少需要一组数据") from None
    result = set(first)
    for group in iterator:
        result.intersection_update(group)
    return result
```

这个题刻意使用普通循环，强调先定义空输入语义。

是否改写为 `reduce`，是后续表达选择。

### 30.13 练习十三：比较器返回布尔值为什么不够？

某同学写：

```python

def bad_compare(first, second):
    return first < second
```

请解释错在哪里。

当 first 大于 second 时返回假，也就是零，比较器会把“更大”误表达成“相等”。

当 first 小于 second 时返回真，也就是一，符号方向也与常见负数表示小于的契约相反。

可用以下简式表达三种方向：

```python

def compare_numbers(first, second):
    return (first > second) - (first < second)
```

但前提仍然是两个值可以进行一致的普通比较，不能由这行代码自动修复 NaN 或自定义矛盾关系。

### 30.14 练习十四：包装顺序与统计目标

你要统计的是“用户发起了多少次调用”，应把计数器放在缓存外还是里面？

参考答案是外面。

如果要统计“真正进入原计算逻辑多少次”，放在缓存内部更贴近目标。

不过并发情况下仍要考虑计数同步；调用数、命中数、实际计算数也不是天然相同。

### 30.15 练习十五：三层 `def` 是语法规则吗？

不是。

请用一句准确的话解释带参数装饰器。

参考表述：装饰器表达式先求值得到可调用对象，这个对象收到被装饰函数，并返回最终绑定到函数名上的对象。

三层嵌套函数是常见实现，它把配置、包装和调用三个阶段对应到三个函数。

可调用类、`partial` 或其他组织方式也可以满足相关协议，但需要另外确认方法绑定和元信息行为。

这比背“带参数装饰器必须三层”更准确，也更能解释实际代码。

## 第三十一章：回到日常代码——还有哪些细节会影响你真正使用这些工具？

### 31.1 递归函数加缓存，就不会遇到递归深度限制了吗？

不会。

缓存减少的是相同子问题的重复计算。

如果第一次计算仍然需要一层一层向下调用，调用栈深度仍然存在。

例如：

```python
from functools import cache


@cache
def triangular(number):
    if type(number) is not int or number < 0:
        raise ValueError("number 必须是非负整数")
    if number == 0:
        return 0
    return number + triangular(number - 1)


print(triangular(5))
```

得到十五。

但直接从一个极大数字开始调用，并不会因为加了缓存就自动转换为循环。

同一个函数在已有缓存基础上逐步扩展，与从空缓存一次走很深，也是不同运行状态。

如果问题本来有简单迭代或公式，应该优先考虑更合适的算法。

本例甚至可以直接使用 `number * (number + 1) // 2`，这时没有必要为了缓存而递归。

### 31.2 缓存后是不是每次都更快？

缓存查找、构造键、维护 LRU 顺序也有成本。

如果原函数只做一个很简单的加法，缓存包装可能比直接计算更慢。

如果绝大多数输入从不重复，又持续挤掉旧条目，维护缓存的成本也未必能换来足够收益。

因此，判断缓存是否有价值，需要结合真实输入分布。

`cache_info()` 可以提供一部分观察，但命中率高也不等于总收益一定大。

被省下的计算多贵、键和值占多少内存、应用会运行多久，都需要一起考虑。

### 31.3 怎样区分冷缓存与热缓存？

冷缓存可以理解为相关结果还没有建立。

热缓存表示常用结果已经存在。

如果性能测量只反复调用同一个参数，你主要测到的是热缓存命中速度。

如果真实程序每次处理全新输入，这个测量就无法代表实际情况。

可以分别记录首次计算、重复调用和代表性混合输入。

另外，清缓存会重置相应统计，比较时要确认起始状态一致。

不要把“在循环里调用一百万次同一个值”的结果直接当成整个程序的加速比例。

### 31.4 函数默认参数也会影响键的规范化吗？

看下面几种业务上可能等价的调用：

```python
from functools import lru_cache


@lru_cache(maxsize=32)
def multiply(value, scale=2):
    return value * scale


multiply(10)
multiply(10, 2)
multiply(value=10, scale=2)
```

不要假设缓存包装器会先按照你心里的“最终参数表”把它们全部合并。

如果这个差别确实影响应用，可以保留一个未缓存的公开入口，并只让内部函数接收一种固定形式。

```python
from functools import lru_cache


@lru_cache(maxsize=32)
def multiply_core(value, scale):
    return value * scale


def multiply(value, scale=2):
    return multiply_core(value, scale)
```

现在外层参数绑定完成后，总是按两个位置参数调用内部入口。

这个小改动通常比尝试依赖缓存内部键实现更清楚。

如果输入类型也需要合并，例如数字字符串转换成整数，还应该明确转换失败与数据范围规则。

### 31.5 自定义对象只要实现 `__hash__` 就适合缓存了吗？

还需要相等与哈希一致，而且对象在键中的行为应保持稳定。

如果你把可变属性纳入哈希，再修改这些属性，已有字典或缓存中的键可能不再按原来方式查找。

这不是 `lru_cache` 独有的问题，而是哈希容器的基本契约。

相等对象应当具有相同哈希值。

不相等对象仍然允许发生哈希冲突，字典会结合相等比较继续区分。

因此，不能用“哈希值相同”直接证明两个业务对象完全一样，也不能只存 `hash(obj)` 作为唯一内容身份。

简单、明确、不可变的数据键通常更容易正确使用。

### 31.6 为什么没有介绍一个万能的 `freeze()`？

因为任意 Python 对象可能包含循环引用、文件句柄、网络连接、生成器、可变集合和自定义相等规则。

把它们一律递归转成元组，往往无法保持原语义。

集合原本无序，字典的插入顺序在某些任务里又可能重要。

对象的属性字典也不一定完整表达其行为。

所以本篇宁可针对成绩、配置、版本号分别设计明确键，也不提供一个看似包办一切、实际容易误用的工具。

学会列出计算的真实依赖，本身就是很实用的能力。

### 31.7 `wraps` 默认到底复制哪些资料？

日常最明显的是名称、限定名称、模块、文档和注解等信息。

它还会更新包装对象相应的属性字典，并设置 `__wrapped__` 关系。

Python 3.12 起，默认复制项还包含泛型相关的 `__type_params__`。

这些默认行为对应 `update_wrapper` 的 `assigned` 与 `updated` 两类配置。

`assigned` 可以理解为把指定属性赋到包装对象上。

`updated` 可以理解为更新包装对象中指定的映射属性。

一般业务装饰器使用默认 `@wraps(func)` 就很好，不必手动复制一长串名称。

如果你确实要调整默认行为，可以查看常量：

```python
from functools import WRAPPER_ASSIGNMENTS, WRAPPER_UPDATES

print(WRAPPER_ASSIGNMENTS)
print(WRAPPER_UPDATES)
```

具体输出会随 Python 版本的默认项目变化，因此应观察当前解释器，而不是把一个版本的打印结果当作永久不变的语法。

元信息更新的参数与版本说明可以直接对照 [官方 update_wrapper 文档](https://docs.python.org/3.14/library/functools.html#functools.update_wrapper)。

### 31.8 原函数上有自定义属性，也会影响包装器吗？

默认会更新包装函数的属性字典，所以自定义属性可能被复制到包装对象上。

但这通常是对象引用层面的更新，不是深拷贝所有属性内容。

如果原属性引用一个列表，两个位置可能继续指向同一个列表。

此外，包装器自己也可能有同名属性，需要考虑更新顺序。

所以不要把 `wraps` 描述成“只改一个名字”，也不要把它描述成“完整克隆一个函数”。

它做的是一组明确的元信息维护操作。

### 31.9 `partial` 对象一定有原函数的 `__name__` 吗？

普通 `partial` 对象不会自动表现得和原函数具有完全相同的名称与文档。

你可以根据需要手动赋予说明，或使用 `update_wrapper`。

但如果这个部分函数已经固定了某些参数，直接复制原文档后，说明内容也可能需要补充。

例如原函数文档说调用者必须传三个参数，预绑定后实际上只需要一个，原说明就不够贴切。

元信息保留的目标是帮助人理解，并不要求机械复制后不做任何思考。

对于重要的公共接口，写一个有清楚签名和文档的普通包装函数，往往更适合阅读。

### 31.10 `singledispatch` 可以注册抽象基类吗？

可以支持相应抽象基类注册与继承关系分派。

例如使用 `collections.abc.Mapping` 表达映射接口，比只注册 `dict` 更宽泛。

```python
from collections.abc import Mapping
from functools import singledispatch


@singledispatch
def count_fields(value):
    raise TypeError("需要映射对象")


@count_fields.register(Mapping)
def count_mapping_fields(value):
    return len(value)


print(count_fields({"a": 1, "b": 2}))
```

输出二。

但宽泛注册不等于你可以假设每个实现都具有普通字典全部特性。

只应使用该接口真正承诺的操作。

如果多个互不包含的抽象基类都匹配，又没有明确更具体实现，可能出现歧义。

遇到这种情况，应通过明确注册或重新设计类型模型解决，而不是猜测导入顺序会选谁。

### 31.11 注册函数一定要使用同一个名字 `_` 吗？

不需要。

官方示例经常使用 `_`，强调实现通过注册表使用，而不是业务直接调用这个名字。

你也可以像本篇一样给每个实现取明确名称，便于单独验证。

`register()` 返回相应注册函数，因此可以保留直接调用具体实现的入口。

还可以用函数式方式注册：

```python

def display_bytes(value, *, compact=False):
    text = value.hex()
    return text if compact else f"字节：{text}"


display.register(bytes, display_bytes)
```

这里需要沿用第二十五章的 `display`。

函数式注册不需要在实现定义上加装饰器，但最终修改的是同一个注册表。

### 31.12 为 Union 注册，是否会在运行时识别所有复合注解？

不会。

Python 3.11 支持的 Union 注册，表示把相关具体类型分别纳入分派，而不是让所有静态类型系统表达式都成为运行时匹配规则。

例如可以明确写：

```python
from functools import singledispatch


@singledispatch
def number_kind(value):
    return "其他"


@number_kind.register
def number_kind_numeric(value: int | float):
    return "普通数值"


print(number_kind(1))
print(number_kind(1.5))
```

布尔值仍需考虑它与整数的继承关系。

`list[int]` 与 `list[str]` 的元素参数也仍然不是两个普通运行时列表类型。

这些边界与 [官方 singledispatch 文档](https://docs.python.org/3.14/library/functools.html#functools.singledispatch) 一致。

### 31.13 `partialmethod` 预绑定的可变对象，会每个实例一份吗？

不要这样假设。

如果你在类定义时建立一个列表并交给描述符，它仍然是那时建立的对象引用。

多个实例可能通过同一个定义使用它。

如果需要每实例独立数据，通常应该在 `__init__` 中创建实例属性，再让方法读取对应属性。

这与函数默认可变参数、类属性共享列表有相似的根源：对象在何时创建，哪些地方引用它。

`partialmethod` 主要解决方法绑定，不负责替你按实例复制任意参数。

### 31.14 `reduce` 的操作顺序能随便交换吗？

不能。

它按迭代器给出的顺序，持续把当前累计结果与下一项结合。

减法、除法、字符串连接和很多状态更新都依赖顺序。

甚至普通浮点加法，也不应在所有精度问题上假设重排完全无影响。

如果你要并行归约或先分组再合并，需要相应操作具有适合的结合性质，并考虑数值与业务语义。

`reduce` 没有自动证明这些性质。

### 31.15 学完这些工具，最常见的日常组合是什么？

批量回调参数总重复，可以先想 `partial`。

类上要派生几个固定选项的方法，可以想 `partialmethod`。

写装饰器时，通常想起 `wraps`，再检查参数转发、返回值、异常与同步异步语义。

同一份不可变输入反复做昂贵计算，可以评估 `lru_cache`，同时写明容量与失效。

对象数据近似不变、属性计算昂贵，可以评估 `cached_property`。

一个统一动作面对多种主要输入类型，可以评估 `singledispatch`。

已有旧式比较函数要交给排序，可以用 `cmp_to_key`。

对象确实具有一致顺序、手写比较重复时，可以考虑 `total_ordering`。

这些判断都从你遇到的问题出发，而不是从“我今天要用完这个库”出发。

### 31.16 哪些版本点需要真正记住？

本文主要代码可按 Python 3.11 及以上环境理解。

`cached_property` 从 3.8 起存在，`cache` 从 3.9 起存在，所以在本文基线上无需为这两个名称额外安装库。

`singledispatch` 对 Union 注解的相关支持从 3.11 起具备。

`cached_property` 的内部锁在 3.12 被移除，不能依靠它保证并发首次读取只计算一次。

`update_wrapper` 默认复制 `__type_params__` 是 3.12 的变化。

`Placeholder` 是 3.14 新能力。

`reduce` 接受 `initial=` 关键字也是 3.14 的变化；兼容较早版本时继续使用第三个位置参数。

遇到自己环境与示例不一致时，可以先查看：

```python
import sys

print(sys.version)
```

再对照 [Python 3.11 functools 文档](https://docs.python.org/3.11/library/functools.html)、[Python 3.12 functools 文档](https://docs.python.org/3.12/library/functools.html) 或 [Python 3.14 functools 文档](https://docs.python.org/3.14/library/functools.html) 中对应接口的版本说明。

阅读官方资料时，应直接查看你要使用的接口，而不是只看搜索结果摘要里的函数名称。

## 第三十二章：复习时，用这一张表找回选择思路

前面的例子已经逐步说明调用过程、边界和实践。

下表只用于你读完后快速找回相关工具；遇到细节问题，仍应回到相应章节的输入、输出与解释。

| 你正在遇到的问题 | 可以考虑的工具 | 使用前先确认什么 |
| --- | --- | --- |
| 同一函数总有几个参数重复 | `partial` | 参数位置、关键字覆盖、是否引用可变对象 |
| 类上要提供几个固定选项的方法 | `partialmethod` | `self` 绑定及共享参数对象 |
| 包装后名称、文档不清楚 | `wraps` / `update_wrapper` | 元信息不能修复错误控制流 |
| 有限种输入需要无界复用 | `cache` | 键空间和结果是否真的可控 |
| 结果复用但需要限制条目数 | `lru_cache` | 可哈希键、容量、失效、共享返回对象 |
| 单个实例的昂贵属性 | `cached_property` | 实例属性字典、数据变化、覆盖与并发 |
| 主要输入类型决定实现 | `singledispatch` | 第一参数、注册范围和附加参数契约 |
| 实例方法按业务输入类型选择 | `singledispatchmethod` | 第一个非 self/cls 参数和装饰器顺序 |
| 二元操作持续合并状态 | `reduce` | 初始值、空输入、顺序与可读性 |
| 适配已有两两比较函数 | `cmp_to_key` | 负零正契约与比较一致性 |
| 类有稳定顺序但比较方法重复 | `total_ordering` | 相等和顺序是否使用一致模型 |

你不需要一次把每个名称背下来。

下一次写包装函数时，能想到检查返回值和 `wraps`；下一次加缓存时，能先问“结果到底依赖什么、何时变化”，这些具体判断才会真正改变你写代码的方式。
