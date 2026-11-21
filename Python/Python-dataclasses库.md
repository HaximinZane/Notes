# Python `dataclasses` 库超详细教程：从一条数据，到有名字、有规则、可以比较和复制的对象

## 零基础先导：数据类不是一种新的字典

`dataclasses` 的出发点是：很多 Python 类不是用来实现复杂算法，而只是把几项有名字的数据放在一起。你可以手写 `__init__`、`__repr__` 等；`@dataclass` 可以依据**字段注解**生成常见方法，减少重复模板代码。

### 先看没有装饰器的写法

```python
class Book:
    def __init__(self, title, price):
        self.title = title
        self.price = price
```

每新增字段，都可能要同步修改初始化与显示代码。使用标准库：

```python
from dataclasses import dataclass

@dataclass                # 根据下面声明的字段生成一些常见方法
class Book:
    title: str            # 字段名 title，期望字符串
    price: int            # 字段名 price，期望整数

book = Book("Python", 39) # 调用生成的初始化方法
print(book)               # Book(title='Python', price=39)
```

这里的冒号类型注解并不等于运行时自动强制检查：`Book("Python", "便宜")` 默认也能构造，类型提示主要供阅读和静态工具使用。若你需要**验证外部输入**，还要自己加检查或使用 Pydantic 等工具。

后面理解 `field(default_factory=list)` 时，先想“如果所有对象共用一个可变列表会怎样”；理解 `frozen=True` 时，区分“禁止普通属性赋值”和“深层对象绝对不可变”。这才是装饰器参数存在的原因。

---

你可能已经会创建类，也知道 `__init__` 用来初始化对象。

但是，实际写脚本时，经常会遇到一种看起来很普通、写起来却很重复的类。

它主要保存几项数据。

例如，一条笔记有标题、分类和是否完成；一项下载任务有地址、保存位置和重试次数。

你想创建对象、打印对象、比较两份数据，有时还想生成修改后的新对象。

这些需求没有复杂的继承体系，却仍然可能让你反复编写初始化、显示和比较代码。

`dataclasses` 就是为这一类常见工作准备的标准库工具。

本篇以 Python 3.11 及以上的常用能力为基础，涉及更新版本的地方会单独说明。

例子会逐步从一张便签发展为文件导入、配置处理和统计工具。

同一小节内的短代码可以接着运行；标明“完整程序”的代码包含所需导入和入口，适合单独保存执行。

## 第一章：我已经会用字典，为什么还要有数据类？

### 1.1 先从你可能真的会写的数据开始

假设你正在做一个个人笔记工具。

一条笔记可以先写成字典：

```python
note = {
    "title": "学习 pathlib",
    "category": "Python",
    "completed": False,
}

print(note["title"])
```

输出：

```text
学习 pathlib
```

这完全可以使用。

字典适合保存键值关系，不需要为了学习一个新库，就把所有字典都换成类。

但是，当这份结构被多个函数反复使用时，你可能开始遇到一些麻烦。

有的地方把 `completed` 写成 `complete`。

有的地方忘记提供分类。

有的地方把是否完成填成字符串 `"false"`。

不同函数之间又没有明确说明，一条笔记究竟应该有什么字段。

这些问题有的可以通过类型注解、验证函数或约定解决，有的适合通过一个有名字的类型来组织。

### 1.2 如果先用普通类写呢？

可以这样写：

```python
class Note:
    def __init__(self, title, category, completed=False):
        self.title = title
        self.category = category
        self.completed = completed


note = Note("学习 pathlib", "Python")
print(note.title)
```

同样输出标题。

这时，`Note` 给这组数据起了一个名字。

看到参数类型是 `Note`，你就能联想到它代表一条笔记，而不只是某个任意字典。

但是如果直接打印：

```python
print(note)
```

默认显示往往类似：

```text
<__main__.Note object at 0x...>
```

地址部分会变化。

它告诉你这是哪个类的对象，却没有直接显示标题和完成状态。

你可能继续补一个 `__repr__()`。

之后又希望两条内容一样的笔记能比较相等，于是继续补 `__eq__()`。

这些代码本身不难，只是容易随着字段增加而重复修改。

### 1.3 数据类帮你自动生成哪些东西？

先改写成：

```python
from dataclasses import dataclass


@dataclass
class Note:
    title: str
    category: str
    completed: bool = False


note = Note("学习 pathlib", "Python")
print(note)
```

输出：

```text
Note(title='学习 pathlib', category='Python', completed=False)
```

你没有亲自写 `__init__()`，仍然可以传入标题和分类。

你没有亲自写 `__repr__()`，仍然可以看到有意义的对象内容。

再创建一条：

```python
other = Note("学习 pathlib", "Python")
print(note == other)
print(note is other)
```

输出：

```text
True
False
```

两份内容相等，但它们不是同一个对象。

`dataclasses` 根据声明的字段自动生成常见方法，让这类代码保持一致。

它没有替你设计业务规则，也不会自动推断一条笔记是否合理。

### 1.4 为什么这个库叫 `dataclasses`？

data 是数据，class 是类。

数据类这个名字，强调类的一个常见用途：表达一份具有明确字段的数据。

Python 3.7 将它加入标准库，设计记录可以参考 [PEP 557](https://peps.python.org/pep-0557/)。

在这之前，开发者已经使用普通类、具名元组以及第三方工具处理类似需求。

`dataclasses` 选择沿用普通 `class` 语法，再通过装饰器生成方法。

所以它很容易接到你已经学习过的类、装饰器和魔法方法上。

## 第二章：逐字看懂第一个 `@dataclass`

### 2.1 第一行到底导入了什么？

```python
from dataclasses import dataclass
```

左边的 `dataclasses` 是模块名称，带复数形式。

右边的 `dataclass` 是我们要使用的装饰器名称。

两者不是拼写不一致。

模块还提供 `field()`、`fields()`、`replace()`、`asdict()` 等工具，后面会分别解释。

它属于标准库，正常环境不需要另外执行 `pip install dataclasses`。

你如果看到旧项目安装同名包，先核对它是不是为了兼容更早的 Python；不要把旧项目的兼容方案直接搬进新项目。

### 2.2 `@dataclass` 在什么时候生效？

看这一段：

```python
@dataclass
class Point:
    x: int
    y: int
```

先创建类，再把类交给装饰器处理，最后将处理后的结果绑定给名称 `Point`。

可以用下面的写法帮助理解：

```python
class Point:
    x: int
    y: int


Point = dataclass(Point)
```

这发生在执行类定义的阶段，不是每次创建 `Point(1, 2)` 时重新分析一遍类正文。

普通情况下装饰器修改并返回原类；使用 `slots=True` 时会返回一个新类，这个特殊情况后面再讲。

因此，不要把“装饰器一定返回一个嵌套函数”当成统一规则。

装饰器语法关心的是：表达式求值得到可调用对象，再用它处理被装饰对象。

### 2.3 冒号后面的 `int` 是在给字段赋值吗？

```python
x: int
```

这是变量注解，表示这个字段预期保存整数。

它本身不是：

```python
x = int
```

后者会把整数类型对象赋给名称 `x`，含义完全不同。

也不是：

```python
x = int()
```

后者调用 `int()`，得到整数零。

在数据类里，注解帮助装饰器找出哪些名称属于字段，并帮助编辑器或静态检查工具理解数据结构。

### 2.4 创建对象时，参数按什么顺序对应？

```python
point = Point(10, 20)
print(point.x)
print(point.y)
```

输出：

```text
10
20
```

生成的初始化参数通常按照字段声明顺序排列。

所以这里十对应 `x`，二十对应 `y`。

你也可以写得更明确：

```python
point = Point(x=10, y=20)
```

字段变多，或者几个字段含义相近时，关键字参数通常更容易读。

### 2.5 我想自己看生成的构造参数呢？

可以借助标准库的签名查看工具：

```python
import inspect

print(inspect.signature(Point))
```

常见输出类似：

```text
(x: int, y: int) -> None
```

签名显示构造对象时接受哪些参数。

这里的返回注解来自初始化方法的语义；你调用 `Point(...)` 仍然会得到 `Point` 实例，并不是得到 `None`。

类调用创建实例与 `__init__()` 的返回值，是两个相关但不同的概念。

## 第三章：写了类型，为什么传错了也可能成功？

### 3.1 先做一个容易让初学者意外的实验

```python
from dataclasses import dataclass


@dataclass
class Reading:
    title: str
    minutes: int


reading = Reading(title=123, minutes="四十分钟")
print(reading)
```

默认情况下，这个构造过程可能正常完成，显示：

```text
Reading(title=123, minutes='四十分钟')
```

因为普通数据类不会根据字段注解自动执行通用的运行时类型检查。

生成的初始化方法主要负责把收到的值放到对应属性上。

### 3.2 那注解到底有什么用？

它有几个实际作用。

装饰器根据注解识别字段。

编辑器可以提供补全和错误提示。

静态类型检查工具可以分析调用位置是否符合约定。

阅读代码的人可以知道这个字段预期是什么。

这些作用都很重要，但它们不等于每次运行时自动转换和验证输入。

特别是从 JSON、CSV、命令行或环境变量拿到的数据，仍然需要明确处理。

### 3.3 如果想在创建时立即检查呢？

可以写 `__post_init__()`：

```python
from dataclasses import dataclass


@dataclass
class Reading:
    title: str
    minutes: int

    def __post_init__(self):
        if not isinstance(self.title, str) or not self.title.strip():
            raise ValueError("title 必须是非空字符串")
        if type(self.minutes) is not int or self.minutes < 0:
            raise ValueError("minutes 必须是非负整数")


reading = Reading("学习数据类", 40)
print(reading.minutes)
```

输出四十。

`__post_init__()` 由生成的初始化方法在字段赋值后调用。

它是你放置规则的一个入口，不是数据类自动理解所有规则的证明。

### 3.4 为什么这里用 `type(value) is int`？

因为 `bool` 是 `int` 的子类。

如果你写：

```python
isinstance(True, int)
```

结果是真。

但是学习时长为 `True` 不是我们想接受的输入。

这个例子选择要求精确的整数类型，因此排除了布尔值。

这是一项业务约定。

如果你的模型需要接收其他整数实现，应使用适合的协议或类型检查方式，不要把精确类型判断到处机械照搬。

### 3.5 创建时检查过，之后就永远不会出错吗？

不一定。

普通数据类的字段仍然可以重新赋值：

```python
reading.minutes = -100
```

这次赋值不会自动重新调用 `__post_init__()`。

所以“构造时满足规则”与“整个生命周期始终满足规则”需要分开设计。

你可以使用冻结对象、受控修改方法、属性或其他验证模型。

选哪一种，取决于对象的职责，而不是看到验证需求就不断往一个方法里叠代码。

## 第四章：必填字段、默认字段和 `None`

### 4.1 没有默认值，代表构造时必须提供

```python
from dataclasses import dataclass


@dataclass
class Task:
    title: str
    completed: bool = False


first = Task("整理笔记")
second = Task("复习文件操作", completed=True)
print(first)
print(second)
```

输出：

```text
Task(title='整理笔记', completed=False)
Task(title='复习文件操作', completed=True)
```

`title` 没有默认值，所以必须提供。

`completed` 默认是假，可以省略。

如果写 `Task()`，通常会因为缺少必需参数而抛 `TypeError`。

这个参数数量错误与字段值验证错误，不是同一件事。

### 4.2 为什么必填字段通常要写在默认字段前面？

普通函数就有类似规则：

```python
def create_task(title, completed=False):
    return title, completed
```

如果普通位置参数中，默认参数后面又放一个无默认参数，会让调用规则不合法。

数据类生成初始化方法时，也必须形成合法的函数签名。

因此，下面是一个预期失败的定义：

```python
from dataclasses import dataclass

try:
    @dataclass
    class BadTask:
        completed: bool = False
        title: str
except TypeError:
    print("普通必填字段不能放在默认字段后面")
```

这里失败发生在处理类定义时，而不是等你创建对象才出现。

后面会讲关键字专用字段，它提供另一种组织参数的方法。

### 4.3 写 `str | None`，是不是就能省略参数？

不是。

```python
from dataclasses import dataclass


@dataclass
class Note:
    title: str
    description: str | None


note = Note("学习 dataclasses", None)
print(note.description)
```

输出：

```text
None
```

注解表达预期值可以是字符串或者 `None`。

它没有为参数提供默认值。

如果想省略时使用 `None`，要明确写：

```python
@dataclass
class Note:
    title: str
    description: str | None = None
```

“允许为空”与“允许不传”是两个问题。

这个区分在 Pydantic、函数参数以及接口模型中也会反复出现。

### 4.4 空字符串和 `None` 有什么不同？

假设 `description=None` 表示没有填写描述。

而 `description=""` 表示明确提供了空文本。

你的工具也可以选择把两者视为同一种业务状态，但要主动规定。

不要随手用 `value or default` 合并它们，因为这还会把零、空列表和 `False` 一并当成需要替换的值。

例如，更新任务时，用户明确传入 `completed=False`，通常不应被当成“没有提供”。

这会在后面的部分更新实践中具体出现。

## 第五章：为什么列表默认值要用 `default_factory`？

### 5.1 我想让每条笔记都带一个标签列表

很容易想到：

```python
from dataclasses import dataclass

try:
    @dataclass
    class BadNote:
        title: str
        tags: list[str] = []
except ValueError:
    print("可变默认值需要重新设计")
```

在这里使用的现代 Python 中，这个定义会被拒绝。

原因要从普通类的默认对象共享理解。

类定义执行时，`[]` 只创建一次。

如果生成的初始化方法直接把这个列表当默认值，多次构造就可能共享同一个列表。

你给一条笔记追加标签，另一条笔记也跟着变化，这通常不是你想要的。

### 5.2 正确的常见写法

```python
from dataclasses import dataclass, field


@dataclass
class Note:
    title: str
    tags: list[str] = field(default_factory=list)


first = Note("学习 open")
second = Note("学习 pathlib")
first.tags.append("文件处理")

print(first.tags)
print(second.tags)
print(first.tags is second.tags)
```

输出：

```text
['文件处理']
[]
False
```

每次需要默认标签时，调用 `list()` 创建一个新列表。

所以默认值彼此独立。

### 5.3 为什么传 `list`，而不是 `list()`？

`default_factory` 要接收一个不需要参数就能调用的对象。

`list` 是可调用的类型对象。

`list()` 是已经调用后的结果，是一个列表。

下面写法方向错误：

```python
field(default_factory=list())
```

你交进去的是列表结果，而不是未来用来创建列表的工厂。

看到 factory，可以把它理解成“需要值的时候，再调用这个东西生成”。

它不要求一定是 `def` 定义的函数；类型对象、适当的 `partial` 或其他可调用对象也可以。

### 5.4 如果默认列表不是空的呢？

可以写一个返回新对象的函数：

```python
from dataclasses import dataclass, field


def default_tags():
    return ["未分类"]


@dataclass
class Note:
    title: str
    tags: list[str] = field(default_factory=default_tags)


print(Note("第一条").tags)
```

输出：

```text
['未分类']
```

短小的构造也常写成：

```python
tags: list[str] = field(default_factory=lambda: ["未分类"])
```

这里的 lambda 只负责每次创建一个新列表。

如果默认结构复杂，用有名字的函数通常更容易解释和测试。

### 5.5 使用工厂，就一定没有共享了吗？

仍然要看工厂返回什么。

如果它总返回同一个全局列表，共享问题仍然存在：

```python
shared_tags = []


def bad_factory():
    return shared_tags
```

“每次调用函数”不等于“每次创建新对象”。

另外，即使默认值彼此独立，你主动传入同一个列表，也会共享：

```python
tags = ["Python"]
first = Note("第一条", tags)
second = Note("第二条", tags)
print(first.tags is second.tags)
```

输出为真。

数据类不会默认复制调用者提供的所有对象。

要隔离外部列表，需要在明确的位置复制；要表达不允许改变的标签集合，也可以选择元组。

### 5.6 它怎么判断哪些默认值危险？

Python 3.11 起，数据类通过拒绝不可哈希的直接默认值，近似防止常见可变默认值问题。

这个规则不是一个能识别所有可变对象的完整系统。

有的可变自定义对象仍然可哈希，有的不可哈希对象未必符合你对“可变”的直觉。

所以理解对象共享仍然必要，不能只依靠类定义有没有报错判断设计是否正确。

## 第六章：`field()` 是在放一个字段值吗？

### 6.1 它更像字段的说明书

```python
from dataclasses import dataclass, field


@dataclass
class User:
    name: str
    password: str = field(repr=False)


user = User("Alice", "practice-password")
print(user)
```

输出：

```text
User(name='Alice')
```

`password` 仍然存在。

只是生成的 `repr` 不显示这个字段。

`field()` 向数据类装饰器提供配置，不是说最终 `user.password` 应该保存一个 `Field` 对象。

### 6.2 隐藏显示，等于数据保密吗？

不等于。

你仍然可以访问：

```python
print(user.password)
```

其他导出工具也可能读出这个属性。

尤其是 `asdict(user)` 默认会包含它。

因此，`repr=False` 主要减少普通对象显示时的意外暴露，不是加密、权限控制或完整脱敏策略。

对于真正的秘密，不要把演示中的明文密码模型当成实际密码存储方案。

这里用这个字段只是为了说明显示策略和实际数据之间的区别。

### 6.3 `default` 和 `default_factory` 能同时写吗？

不能。

```python
from dataclasses import field

try:
    field(default=[], default_factory=list)
except ValueError:
    print("默认值和默认工厂只能选择一种")
```

一个表达“直接使用这个值”，一个表达“需要时调用工厂”。

同时提供，会让默认值的来源不明确。

对于不可变的简单值，直接写 `completed: bool = False` 通常最清楚。

需要更多字段选项时，再写 `field(default=False, repr=False)` 之类组合。

### 6.4 常用选项分别控制哪件事？

`init=False` 控制字段是否进入生成的初始化参数。

`repr=False` 控制字段是否进入自动生成的对象显示。

`compare=False` 控制字段是否参与生成的相等与排序方法。

`hash=False` 可以排除生成哈希时使用的某字段，但必须理解它与相等规则的关系。

`kw_only=True` 要求这个字段通过关键字传入。

`metadata=` 给字段附加自定义说明，数据类本身不会自动执行这些说明里的业务规则。

这些开关各有作用，不应该因为某个字段“比较特殊”就把所有开关都设为假。

后面会在具体场景里分别验证。

## 第七章：对象还可以有方法和属性吗？

### 7.1 数据类仍然是普通的 Python 类

```python
from dataclasses import dataclass


@dataclass
class Purchase:
    name: str
    unit_price_cents: int
    quantity: int = 1

    def total_cents(self):
        return self.unit_price_cents * self.quantity


purchase = Purchase("笔记本", 1290, 3)
print(purchase.total_cents())
```

输出：

```text
3870
```

数据类可以包含方法、文档字符串和其他普通类行为。

我们把金额按分保存，是这个小例子的单位约定，避免把二进制浮点误差与数据类概念搅在一起。

你也可以在其他模型中选择 `Decimal`，但那时要另外规定输入、计算与舍入规则。

### 7.2 如果我想写成属性访问呢？

```python
from dataclasses import dataclass


@dataclass
class Purchase:
    name: str
    unit_price_cents: int
    quantity: int = 1

    @property
    def total_cents(self):
        return self.unit_price_cents * self.quantity


purchase = Purchase("笔记本", 1290, 3)
print(purchase.total_cents)
```

同样输出三千八百七十。

这次没有调用括号，因为 `total_cents` 是属性。

属性每次读取时重新计算，不会因为用了数据类就自动缓存。

### 7.3 总价应该保存成字段，还是临时计算？

如果计算非常便宜，又直接依赖可变字段，属性通常简单。

假设你保存了一个普通 `total_cents` 字段，后来改了数量，却忘记同步总价，就会出现数据之间不一致。

使用属性时：

```python
purchase.quantity = 4
print(purchase.total_cents)
```

结果随当前数量重新计算。

如果计算昂贵、结果来自外部系统，或者你要记录历史结算值，保存成字段也可能合理。

关键是先说清楚：这个值是当前数据推导出来的，还是独立的历史事实。

### 7.4 属性会出现在 `fields()` 和 `asdict()` 里吗？

普通 `@property` 不会自动成为数据类字段。

它不会因为你能通过点号访问，就被所有数据类工具当成存储字段。

如果输出报告时需要总价，可以在明确的导出函数中加入它。

这通常比期待“任何属性都自动序列化”更容易控制接口。

同样，方法名称也不会变成字段。

字段来自相应声明规则，不是对象上所有能访问到的名称的集合。

## 第八章：`__post_init__()` 不只是“再初始化一次”

### 8.1 它在正常构造流程中的位置

先看一份可以直接观察顺序的代码：

```python
from dataclasses import dataclass


@dataclass
class Note:
    title: str

    def __post_init__(self):
        print("此时已经能读取标题：", self.title)


note = Note("学习初始化")
print("构造结束")
```

输出：

```text
此时已经能读取标题： 学习初始化
构造结束
```

生成的初始化方法先给字段赋值，再调用 `__post_init__()`。

因此，它适合做依赖多个输入字段的检查，以及某些派生字段的初始化。

### 8.2 如果想去掉标题两端的空格呢？

```python
from dataclasses import dataclass


@dataclass
class Note:
    title: str

    def __post_init__(self):
        if not isinstance(self.title, str):
            raise TypeError("title 必须是字符串")
        self.title = self.title.strip()
        if not self.title:
            raise ValueError("title 不能只有空白")


print(Note("  学习 dataclasses  ").title)
```

输出：

```text
学习 dataclasses
```

这是一种输入规范化。

但不要对所有字段都机械调用 `strip()`。

正文缩进、代码片段或用户明确保存的空格，可能是有意义的数据。

是否清理，要与字段含义对应。

### 8.3 我自己写了 `__init__()`，它还会自动调用吗？

如果你自己定义初始化方法，数据类不会默认替你在这个方法末尾补上 `__post_init__()` 调用。

看一个完整例子：

```python
from dataclasses import dataclass


@dataclass
class Note:
    title: str

    def __init__(self, title):
        self.title = title

    def __post_init__(self):
        print("检查标题")


note = Note("不会自动调用检查")
print(note.title)
```

这里只打印标题。

如果你接管了构造过程，就应自己明确调用哪些步骤。

不要一边完整接管 `__init__()`，一边仍假设所有自动初始化约定都保留。

### 8.4 初始化里应该顺便读取网络、创建文件吗？

通常先谨慎考虑。

如果创建一个数据对象就会发请求或改文件，打印调试、复制替换和测试都可能变得难以控制。

例如，后面的 `replace()` 会重新调用构造过程。

如果 `__post_init__()` 负责发邮件，生成一份修改后的对象就可能再次发邮件。

对主要表达数据的类，常见做法是让构造负责数据与规则，把外部操作放在清楚命名的函数或服务方法里。

这样看到 `Note(...)` 时，读者更容易理解它的成本和副作用。

## 第九章：`init=False`，为什么有字段却不能传参数？

### 9.1 一个从已有字段推导出来的值

假设矩形保存宽、高，并在创建时计算面积。

```python
from dataclasses import dataclass, field


@dataclass
class Rectangle:
    width: int
    height: int
    area: int = field(init=False)

    def __post_init__(self):
        if self.width < 0 or self.height < 0:
            raise ValueError("宽和高不能为负数")
        self.area = self.width * self.height


rectangle = Rectangle(3, 4)
print(rectangle)
```

输出：

```text
Rectangle(width=3, height=4, area=12)
```

`area` 仍然是字段，所以会参与默认显示和比较。

但是它没有进入生成的初始化参数。

因此，你不需要写 `Rectangle(3, 4, 12)`，也不能用这种方式随意提供一个不匹配的面积。

### 9.2 `init=False` 会自动计算这个字段吗？

不会。

真正计算面积的是你写在 `__post_init__()` 中的赋值。

如果你声明了没有默认值的 `init=False` 字段，却始终没有赋值，读取它时可能抛 `AttributeError`。

生成的 `repr` 需要读取这个字段时，也可能暴露问题。

所以 `init=False` 只改变构造参数的安排，不是一个自动计算开关。

### 9.3 创建后改宽度，面积会跟着变吗？

对上面的实现，不会。

```python
rectangle.width = 10
print(rectangle.area)
```

仍然是十二。

我们保存的是初始化时计算的值，之后并没有自动建立依赖更新系统。

如果希望面积永远反映当前宽高，更自然的方案是用上一章介绍的属性。

如果矩形创建后不应该变化，可以使用冻结数据类。

如果宽高变化必须触发更多规则，就设计一个清楚的修改方法。

同样的 `init=False`，在不同生命周期政策下可能合理，也可能成为陈旧数据的来源。

### 9.4 带工厂的 `init=False` 又有什么用？

例如，每个任务需要一个内部列表，不允许调用者在构造时传入：

```python
from dataclasses import dataclass, field


@dataclass
class Task:
    title: str
    messages: list[str] = field(
        default_factory=list,
        init=False,
        repr=False,
        compare=False,
    )


task = Task("扫描目录")
task.messages.append("开始扫描")
print(task)
print(task.messages)
```

输出：

```text
Task(title='扫描目录')
['开始扫描']
```

生成的初始化方法会调用工厂，为每个对象创建这个内部列表。

它不作为外部构造参数，不进入常规显示，也不参与自动比较。

这里每个选项都对应明确需求，不能只因为它叫内部字段，就推断 Python 阻止外部访问。

调用者仍然能通过属性读取或修改它。

## 第十章：`InitVar`，只在创建时使用的输入

### 10.1 有些参数不应该长期保存在对象里

假设你收到一段带空格的原始标题，需要清理以后再保存。

你希望创建对象时接受原始输入，但对象最终只保留规范标题。

可以使用初始化专用变量：

```python
from dataclasses import InitVar, dataclass, field, fields


@dataclass
class CleanTitle:
    raw_title: InitVar[str]
    title: str = field(init=False)

    def __post_init__(self, raw_title):
        if not isinstance(raw_title, str):
            raise TypeError("原始标题必须是字符串")
        title = raw_title.strip()
        if not title:
            raise ValueError("标题不能为空")
        self.title = title


item = CleanTitle("  学习 InitVar  ")
print(item)
print([field_info.name for field_info in fields(item)])
```

输出：

```text
CleanTitle(title='学习 InitVar')
['title']
```

`raw_title` 进入构造参数，并传给 `__post_init__()`。

它不是一个真正的数据类存储字段，不会出现在 `fields()` 中。

### 10.2 它的名字为什么叫 InitVar？

可以理解成 initialization variable，初始化阶段使用的变量。

普通字段会生成类似 `self.title = title` 的赋值。

初始化专用变量主要负责把构造输入交给后处理方法。

除非你自己再赋给属性，否则不会因为声明了 `InitVar` 就自动在实例上保存一份相同名称的数据。

### 10.3 有多个 `InitVar`，方法参数怎样对应？

它们按照声明顺序传给后处理方法。

```python
from dataclasses import InitVar, dataclass, field


@dataclass
class ScaledValue:
    raw: InitVar[int]
    multiplier: InitVar[int]
    value: int = field(init=False)

    def __post_init__(self, raw, multiplier):
        self.value = raw * multiplier


print(ScaledValue(10, 3).value)
```

输出三十。

这里让方法形参名称与声明名称一致，方便人阅读。

你还应保证参数数量与顺序符合生成调用方式。

### 10.4 我是不是应该把所有临时参数都改成 InitVar？

不需要。

如果只是解析一份原始输入，单独写一个 `from_dict()`、`parse_record()` 或普通工厂函数，也可能更清楚。

`InitVar` 适合让构造过程确实需要的辅助输入参与后处理。

但如果对象创建依赖大量数据库连接、文件句柄和环境状态，构造过程可能已经承担过多工作。

还要注意，没有默认值的 `InitVar` 在调用 `replace()` 创建新对象时，通常也需要重新提供。

它不会凭空从旧实例恢复一份原本就没有保存的输入。

## 第十一章：`ClassVar`，哪些名称属于整个类？

### 11.1 我想给所有对象放一个统一版本号

```python
from dataclasses import dataclass, fields
from typing import ClassVar


@dataclass
class Note:
    format_version: ClassVar[int] = 1
    title: str
    completed: bool = False


note = Note("学习类变量")
print(note)
print(Note.format_version)
print([item.name for item in fields(Note)])
```

输出：

```text
Note(title='学习类变量', completed=False)
1
['title', 'completed']
```

`ClassVar` 告诉数据类：这个注解名称是类变量，不作为普通实例字段处理。

因此，它不会进入生成的初始化参数，也不会进入 `fields()` 的结果。

### 11.2 不写 `ClassVar` 会怎样？

如果你写：

```python
format_version: int = 1
```

它通常会被当作普通数据类字段。

这意味着每个实例的构造、显示和比较可能都包含它。

而且，如果它位于必填字段前面，还会触发默认字段顺序问题。

所以是否属于类，并不只看你把它写在类正文中；普通数据类字段本来也是在类正文中声明的。

要明确表达共享配置和实例数据的不同职责。

### 11.3 `ClassVar` 会让值绝对不能修改吗？

不会。

普通类属性仍然可能被重新赋值。

注解告诉工具和读者这个名称的预期职责，不是一个运行时的只读保护器。

如果类变量保存一个可变容器，仍然可能出现共享修改。

比如把默认标签放在 `ClassVar[list[str]]` 中，不会自动为每个实例复制一份。

因此，类变量适合表达类层面的共同信息，不适合伪装成每个实例独立的可变状态。

## 第十二章：自动生成的 `repr`，能代替正式输出吗？

### 12.1 `repr` 首先服务观察对象

```python
from dataclasses import dataclass


@dataclass
class Task:
    title: str
    completed: bool = False


task = Task("复习文件操作")
print(repr(task))
print(str(task))
```

在没有额外自定义 `__str__()` 的普通情况下，两者通常显示相同形式。

例如：

```text
Task(title='复习文件操作', completed=False)
Task(title='复习文件操作', completed=False)
```

这很适合调试。

但是用户界面可能更希望看到“未完成：复习文件操作”。

JSON 文件则需要明确字段结构。

这些输出目标不同，不应都依靠同一个调试表示。

### 12.2 如果我想为用户显示一句自然的话呢？

可以添加 `__str__()`：

```python
from dataclasses import dataclass


@dataclass
class Task:
    title: str
    completed: bool = False

    def __str__(self):
        state = "已完成" if self.completed else "未完成"
        return f"{state}：{self.title}"


task = Task("整理学习记录")
print(task)
print(repr(task))
```

输出：

```text
未完成：整理学习记录
Task(title='整理学习记录', completed=False)
```

一个负责自然显示，一个保留结构化调试信息。

数据类没有阻止你添加这类方法。

### 12.3 打印结果可以直接保存以后再解析吗？

不建议把默认 `repr` 当作长期交换格式。

字段顺序、类名、嵌套对象表示都可能随着代码变化。

它也不是 JSON。

更不能拿未知来源的这种字符串交给 `eval()`，试图恢复对象。

需要保存，就设计明确的 JSON、CSV 或其他数据格式，再写对应的导入与验证。

后面会给出完整例子。

## 第十三章：`==` 比内容，`is` 比身份

### 13.1 两个新对象为什么会相等？

```python
from dataclasses import dataclass


@dataclass
class Coordinate:
    x: int
    y: int


first = Coordinate(2, 3)
second = Coordinate(2, 3)

print(first == second)
print(first is second)
```

输出：

```text
True
False
```

默认 `eq=True` 会生成基于相应字段的相等方法。

所以两份坐标内容一样，就比较相等。

`is` 仍然问两个名称是否指向同一个对象。

自动比较不会把不同实例合并成一个实例。

### 13.2 两个类字段一样，就一定相等吗？

看这个完整例子：

```python
from dataclasses import dataclass


@dataclass
class ScreenPoint:
    x: int
    y: int


@dataclass
class MapPoint:
    x: int
    y: int


print(ScreenPoint(1, 2) == MapPoint(1, 2))
```

输出：

```text
False
```

生成的相等逻辑要求双方具有相同的具体类型。

屏幕像素坐标和地图坐标即使刚好是同样两个数字，也未必表达同一种数据。

如果你确实需要跨类型比较，应主动定义规则，不要指望字段名字相同就自动打通。

### 13.3 我有一个调试时间，不希望影响相等呢？

```python
from dataclasses import dataclass, field


@dataclass
class Document:
    title: str
    content: str
    inspected_at: str = field(default="", compare=False)


first = Document("笔记", "正文", "09:00")
second = Document("笔记", "正文", "10:00")
print(first == second)
```

输出为真。

`inspected_at` 仍然可以显示和访问，但不参与生成的比较。

这意味着，你在这个模型中把检查时间视为附带信息，而不是决定数据相等的核心内容。

选择 `compare=False` 时，最好能用这样一句具体的话解释原因。

### 13.4 `eq=False` 是不是强制不相等？

不是。

它表示数据类装饰器不生成相等方法。

最终比较行为由类自己、父类以及 Python 的普通规则决定。

如果没有另外定义，通常回到普通对象身份相关的默认行为。

同一个对象与自身比较仍然可能相等。

所以 `eq=False` 应理解为“不替你生成”，而不是“所有比较都返回假”。

### 13.5 比较浮点 NaN，需要特别注意吗？

需要。

NaN 的普通数值相等行为与常见数值不同。

而数据类生成的比较实现也存在版本变化：Python 3.13 调整了逐字段相等的实现，相比更早版本对共享特殊对象的处理可能出现差异。

常规业务字段比较不需要围绕实现细节编程。

如果 NaN 对模型没有合理含义，通常在入口明确拒绝非有限数值更容易维护。

如果你确实要表达特殊数值状态，应单独写出相等政策，并在目标 Python 版本运行验证，不能只根据一段旧版生成代码推断所有版本。

## 第十四章：`order=True`，排序顺序从哪里来？

### 14.1 默认数据类不自动提供大小排序

即使两份数据可以比较相等，也不代表它们天然知道谁大谁小。

例如，一条笔记标题相同可以比较内容，但按什么规则排队，仍然需要你决定。

如果确实希望按声明字段依次比较，可以开启：

```python
from dataclasses import dataclass


@dataclass(order=True)
class Rank:
    priority: int
    name: str


items = [Rank(2, "B"), Rank(1, "C"), Rank(1, "A")]
print(sorted(items))
```

输出：

```text
[Rank(priority=1, name='A'), Rank(priority=1, name='C'), Rank(priority=2, name='B')]
```

先比较优先级。

优先级相同，再比较名称。

它不是根据字段名称自动猜“priority 大的更重要，所以应该排前面”。

### 14.2 如果只想这一次按某字段排序呢？

通常用排序的 `key` 参数就够了：

```python
items = sorted(items, key=lambda item: (-item.priority, item.name))
```

这里优先级从大到小，名称作为后续顺序。

如果同一种数据在不同页面有不同排序方式，把所有排序都写进类的大小关系里反而不灵活。

`order=True` 适合这个类型确实有稳定、明确的比较含义。

### 14.3 不想让正文参与排序，怎么写？

```python
from dataclasses import dataclass, field


@dataclass(order=True)
class RankedNote:
    priority: int
    title: str
    body: str = field(compare=False)
```

这样正文被排除在生成的比较之外。

但注意，`compare=False` 同时影响生成的相等比较。

它不是一个“只排除排序、不排除相等”的专用参数。

如果正文应该决定内容是否相等，却不应该参与某次列表排序，使用 `sorted(..., key=...)` 通常更合适。

### 14.4 可以一边自动生成，一边手写部分大小方法吗？

开启 `order=True` 时，如果类已经定义相关排序方法，数据类会拒绝某些冲突组合。

也不能简单设置 `order=True, eq=False`，因为自动排序依赖相应的一致比较设计。

遇到复杂排序规则，先选择一种清楚的组织方式。

要么让数据类自动生成整套简单的字段比较。

要么自己定义明确的比较逻辑。

要么只在具体排序处提供键函数。

不需要为了省一行 `key=`，把类型的全局语义搞得难以理解。

## 第十五章：`frozen=True`，对象被冻结了什么？

### 15.1 先观察普通重新赋值

```python
from dataclasses import FrozenInstanceError, dataclass


@dataclass(frozen=True)
class Point:
    x: int
    y: int


point = Point(1, 2)

try:
    point.x = 10
except FrozenInstanceError:
    print("不允许这样修改冻结对象")
```

输出：

```text
不允许这样修改冻结对象
```

冻结数据类通过生成相应的属性赋值、删除限制，模拟只读实例。

常见用途是让一份创建好的值对象保持稳定，修改时创建一份新对象。

### 15.2 它会把里面的列表也冻结吗？

不会递归冻结。

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class Note:
    title: str
    tags: list[str] = field(default_factory=list)


note = Note("冻结实验")
note.tags.append("仍然可以修改列表")
print(note.tags)
```

输出：

```text
['仍然可以修改列表']
```

`note.tags = other_list` 与 `note.tags.append(...)` 不是同一种操作。

前者尝试重新绑定属性。

后者修改属性已经引用的那个列表。

冻结通常阻止前一种普通操作，并不让任意深层对象全部不可变。

### 15.3 如果标签本来也不该修改呢？

可以使用元组，并在边界明确转换：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Note:
    title: str
    tags: tuple[str, ...] = ()


note = Note("学习值对象", ("Python", "基础"))
print(note.tags)
```

这里的元组没有列表式追加方法。

但如果元组里面又存放列表，内部列表仍然可能变化。

所以不可变结构需要逐层看，而不是只看最外层有没有 `frozen=True`。

### 15.4 冻结对象怎样在初始化时生成派生字段？

如果你确实需要在后处理阶段赋值，可以使用底层属性赋值方式：

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class Rectangle:
    width: int
    height: int
    area: int = field(init=False)

    def __post_init__(self):
        if self.width < 0 or self.height < 0:
            raise ValueError("宽高不能为负数")
        object.__setattr__(self, "area", self.width * self.height)


print(Rectangle(3, 4))
```

输出面积为十二的对象。

这里是在受控构造过程中完成字段初始化。

它也说明冻结机制不是防止有意绕过的安全边界。

不要在普通业务代码里到处用这种方式修改已发布对象，否则冻结带来的理解优势就消失了。

### 15.5 冻结以后是不是就线程安全了？

不能只凭这个参数下结论。

如果对象内部包含可变列表、外部资源或共享缓存，相关状态仍然可能变化。

即使对象本身是不可变值，多个线程更新“当前对象引用”的完整业务流程，也可能需要协调。

冻结有助于减少某些共享修改，却不是自动提供锁、事务或跨线程协议。

我们使用它时，最好明确说“普通字段赋值被限制”，而不要扩大成“所有并发问题都已解决”。

## 第十六章：数据类能不能放进 `set`，或者当字典键？

### 16.1 先看一个默认数据类

```python
from dataclasses import dataclass


@dataclass
class Point:
    x: int
    y: int


point = Point(1, 2)

try:
    print(hash(point))
except TypeError:
    print("这个默认可变数据类不可哈希")
```

通常会进入错误分支。

默认数据类按字段比较相等，同时又允许字段变化。

把这种可变内容直接用作哈希容器的键，会使规则很容易被破坏。

因此，默认配置会将这类对象标记为不可哈希。

### 16.2 为什么字段变化会影响字典查找？

字典和集合依靠哈希值定位候选位置，再进行必要的相等检查。

如果一个键放入以后，决定它哈希值的内容发生变化，后续查找可能按新的哈希去另一个位置找。

对象仍然在那里，但查找规则已经不一致。

这不是一个适合靠运气观察“我这次似乎还能查到”来设计的机制。

核心要求是：作为键使用期间，哈希与相等所依赖的状态要保持合理、稳定，并符合相等对象哈希一致的约定。

### 16.3 冻结值对象通常怎样使用？

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Point:
    x: int
    y: int


visits = {Point(1, 2): 3}
print(visits[Point(1, 2)])
```

输出：

```text
3
```

在没有自定义哈希冲突配置的常见情况下，`eq=True` 配合 `frozen=True` 会生成哈希方法。

但它用到的字段仍然必须能完成哈希。

### 16.4 冻结对象里有列表，为什么还是不能哈希？

因为列表本身不可哈希。

生成哈希方法不等于把所有字段自动转换为可哈希结构。

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class Note:
    title: str
    tags: list[str] = field(default_factory=list)


try:
    hash(Note("学习哈希"))
except TypeError:
    print("生成了哈希方法，字段仍可能无法哈希")
```

如果标签属于内容身份的一部分，可以考虑使用不可变、可哈希的字段结构。

如果它只是附带的可变缓存，可能应排除比较与哈希，或者拆到另一个对象。

不要为了让报错消失，就随意排除实际决定数据身份的字段。

### 16.5 `unsafe_hash=True` 是不是推荐修复开关？

不是。

它可以强制生成哈希方法，但不会替你冻结数据，也不会修复字段本身的不可哈希问题。

名字里的 unsafe 已经提示，这需要开发者承担一致性设计。

日常脚本里，通常先判断你到底需要对象键，还是使用稳定 ID、字符串、整数或元组作为键就够了。

例如，任务数据经常会更新，而任务编号保持不变，那么用编号建立字典通常更直观。

## 第十七章：`replace()`，怎样生成修改后的新对象？

### 17.1 冻结对象不能直接改，那我想标记完成呢？

```python
from dataclasses import dataclass, replace


@dataclass(frozen=True)
class Task:
    title: str
    completed: bool = False


original = Task("复习 dataclasses")
finished = replace(original, completed=True)

print(original)
print(finished)
print(original is finished)
```

输出：

```text
Task(title='复习 dataclasses', completed=False)
Task(title='复习 dataclasses', completed=True)
False
```

`replace()` 根据旧对象的相关字段创建同类型的新对象，再使用你提供的变更值。

原对象保持原来的状态。

### 17.2 它是偷偷跳过构造过程直接复制吗？

不是。

它会调用相应的初始化过程，所以生成初始化方法通常还会调用 `__post_init__()`。

如果后处理方法验证标题不能为空，那么通过 `replace()` 创建空标题也会失败。

这常常是好事：新的值对象仍然经过相同的规则。

也因此，不要在后处理里偷偷做不应该重复的外部操作。

### 17.3 如果旧对象有 `init=False` 字段呢？

这些字段不会被当成普通构造参数从旧对象原样复制过去。

它们应在新对象初始化时重新产生，或者由你的专门复制方案负责。

对前面的冻结矩形，替换宽度后，面积会在新对象的后处理里重新计算。

```python
from dataclasses import dataclass, field, replace


@dataclass(frozen=True)
class Rectangle:
    width: int
    height: int
    area: int = field(init=False)

    def __post_init__(self):
        object.__setattr__(self, "area", self.width * self.height)


first = Rectangle(3, 4)
second = replace(first, width=5)
print(first.area)
print(second.area)
```

输出：

```text
12
20
```

不能把 `replace(first, area=999)` 当成绕过派生规则的普通用法；`init=False` 字段不接受这种替换参数。

### 17.4 新对象一定没有共享内部列表吗？

不一定。

```python
from dataclasses import dataclass, field, replace


@dataclass
class Note:
    title: str
    tags: list[str] = field(default_factory=list)


first = Note("第一版", ["Python"])
second = replace(first, title="第二版")
second.tags.append("共享实验")

print(first.tags)
print(first.tags is second.tags)
```

输出：

```text
['Python', '共享实验']
True
```

`replace()` 不是自动深拷贝所有字段。

没有替换的初始化字段，通常会把原值引用交给新构造过程。

如果需要隔离标签，可以明确写 `tags=list(first.tags)`，或者从一开始使用不可变标签结构。

### 17.5 我什么时候应该使用专门的方法？

如果一次状态变更需要检查多个字段、生成事件、判断允许的迁移，写 `complete_task()` 或对象上的业务方法可能更清楚。

`replace()` 只表达根据字段创建新值。

它不会自动知道“已归档任务不能重新开始”“结束日期不能早于开始日期”等完整业务规则。

可以让专门方法内部调用 `replace()`，同时把变更政策写在容易找到的位置。

## 第十八章：`fields()`，怎样查看字段，而不是猜对象内部？

### 18.1 我想知道这个模型有哪些字段

```python
from dataclasses import dataclass, field, fields


@dataclass
class Note:
    title: str
    completed: bool = False
    tags: list[str] = field(default_factory=list)


for info in fields(Note):
    print(info.name, info.init, info.repr, info.compare)
```

输出：

```text
title True True True
completed True True True
tags True True True
```

`fields()` 返回一个元组，里面是描述字段的 `Field` 对象。

你可以传类，也可以传这个类的实例。

字段说明与实例上的字段值，是两种不同东西。

### 18.2 那我怎么同时拿到名称和值？

```python
note = Note("学习字段检查", tags=["Python"])

for info in fields(note):
    print(info.name, getattr(note, info.name))
```

`info.name` 提供字段名称字符串。

`getattr(note, info.name)` 按名称读取该实例的属性值。

这是一种明确使用公开字段接口的方式，不需要依赖 `__dataclass_fields__` 的内部细节。

### 18.3 能不能直接用 `vars(note)`？

普通没有槽限制的实例，常常可以。

但 `vars()` 主要读取对象的实例属性字典，不等于专门的字段查询接口。

有的实例使用 `slots=True`，没有普通实例 `__dict__`。

有的实例又被动态加了额外属性，这些属性未必属于数据类字段。

如果你的问题明确是“数据类声明了哪些字段”，`fields()` 更直接。

如果你的问题是“当前实例字典里面到底有什么”，那是另外一种调试观察。

### 18.4 类属性和实例属性不一样，会影响观察吗？

会。

带简单默认值的字段，类上可能仍有对应默认属性。

没有默认值的字段，类上通常没有同名实际值。

使用工厂的字段，也不能期待在类上取到未来每个实例的独立默认对象。

所以不要用 `Note.title` 能不能访问来判断 `title` 是不是字段。

声明结构交给 `fields()`，实例数据从实例读取，含义更清楚。

## 第十九章：`asdict()` 和 `astuple()`，转换时有没有复制？

### 19.1 先看最普通的转换

```python
from dataclasses import asdict, astuple, dataclass


@dataclass
class Point:
    x: int
    y: int


point = Point(2, 3)
print(asdict(point))
print(astuple(point))
```

输出：

```text
{'x': 2, 'y': 3}
(2, 3)
```

一个保留字段名称，生成字典形式。

一个按字段顺序生成元组形式。

对于对外交换数据，字段名称通常比位置更容易理解。

### 19.2 嵌套数据类会怎样？

```python
from dataclasses import asdict, dataclass


@dataclass
class Author:
    name: str


@dataclass
class Note:
    title: str
    author: Author
    tags: list[str]


note = Note("学习转换", Author("Alice"), ["Python"])
data = asdict(note)
print(data)
```

输出：

```text
{'title': '学习转换', 'author': {'name': 'Alice'}, 'tags': ['Python']}
```

嵌套数据类会继续转换，列表、元组和字典也会递归处理。

其他对象会涉及深拷贝。

因此，它不只是把最外层字段引用装进一个字典那么简单。

### 19.3 改导出的标签，会改原对象吗？

对刚才这种普通列表结构：

```python
data["tags"].append("新增")
print(note.tags)
print(data["tags"])
```

输出：

```text
['Python']
['Python', '新增']
```

递归转换创建了相应的新容器。

但是，不能因此就把 `asdict()` 当成任意对象图的完美备份工具。

资源对象可能不支持深拷贝，循环引用可能导致问题，共享引用关系也不应该被当作其通用保真承诺。

### 19.4 如果我只想做浅层字典呢？

可以明确写：

```python
from dataclasses import fields

shallow = {info.name: getattr(note, info.name) for info in fields(note)}
print(shallow["tags"] is note.tags)
```

输出为真。

浅层字典保存字段值的原引用。

它适合某些只需要读取或传递的内部场景，但修改嵌套对象仍然会影响原数据。

选择浅层还是递归转换，应根据用途，而不是把“转换成字典”当成只有一种语义。

### 19.5 `repr=False` 的字段会不会自动排除？

不会。

`asdict()` 处理真实字段，不按默认显示开关决定是否导出。

所以字段里包含令牌、内部备注或本地路径时，应该设计明确的导出函数。

不要把“打印时没有显示”误解成“转成任何格式都不会出现”。

### 19.6 可以给 `dict_factory` 传自定义函数吗？

可以，它控制构造相应映射的方式。

但这不是一个对任意对象、任意敏感性都正确的自动转换入口。

对于简单固定的外部接口，直接写一个返回明确字段的函数，通常更容易审查。

后面导出笔记 JSON 时，我们会使用这种明确函数，而不是把内部结构全部无差别送出去。

## 第二十章：数据类与 JSON，中间还差哪些步骤？

### 20.1 我能直接 `json.dumps(note)` 吗？

标准 JSON 编码器默认并不理解所有数据类实例。

```python
import json
from dataclasses import dataclass


@dataclass
class Note:
    title: str


try:
    json.dumps(Note("学习 JSON"))
except TypeError:
    print("需要先明确转换规则")
```

所以，数据类本身不是一套自动 JSON 协议。

### 20.2 先 `asdict()` 不就够了吗？

对只包含 JSON 基本类型的简单模型，可能够用。

但是模型里经常有日期、路径、集合、枚举或其他对象。

`asdict()` 不会自动把每个日期变成你希望的 ISO 字符串，也不会自动决定路径使用绝对还是相对表示。

看这个例子：

```python
import json
from dataclasses import asdict, dataclass
from datetime import date
from pathlib import Path


@dataclass
class Report:
    day: date
    output: Path


report = Report(date(2026, 11, 3), Path("reports/day.json"))
data = asdict(report)
print(type(data["day"]).__name__)
print(type(data["output"]).__name__)
```

日期仍然是日期对象，路径仍然是相应路径对象。

你还需要决定怎样把它们放入外部格式。

### 20.3 显式导出函数怎样写？

```python
def report_to_dict(report):
    return {
        "day": report.day.isoformat(),
        "output": report.output.as_posix(),
    }


text = json.dumps(report_to_dict(report), ensure_ascii=False, indent=2)
print(text)
```

输出：

```json
{
  "day": "2026-11-03",
  "output": "reports/day.json"
}
```

这段函数明确表达日期格式和路径显示政策。

如果以后内部模型增加了缓存、临时状态或秘密字段，外部结构不会因此自动扩大。

### 20.4 读回来时，直接 `Report(**data)` 可以吗？

这只是把字典里的值按关键字传给构造过程。

如果 `data["day"]` 是字符串，它不会因为字段注解是 `date` 就自动变成日期。

如果 `data["output"]` 是字符串，它也不会自动变成 `Path`。

你需要显式解析：

```python
def report_from_dict(data):
    if not isinstance(data, dict):
        raise ValueError("报告顶层必须是对象")
    if set(data) != {"day", "output"}:
        raise ValueError("报告字段不符合约定")
    if not isinstance(data["day"], str):
        raise ValueError("day 必须是字符串")
    if not isinstance(data["output"], str):
        raise ValueError("output 必须是字符串")
    return Report(
        day=date.fromisoformat(data["day"]),
        output=Path(data["output"]),
    )
```

外部字典、领域对象和文件格式之间的边界，因此变得很明确。

如果这类验证规则越来越多，Pydantic 可能更适合承担输入模型工作；对应独立教材会详细说明。

## 第二十一章：关键字专用参数，让调用更容易读

### 21.1 两个布尔值放在位置参数里，容易看错

假设看到：

```text
ScanOptions("notes", True, False, 100)
```

第二项、第三项分别是什么意思？

你可能必须回到类定义才能确认。

如果这些参数本来就适合用名字表达，可以设置 `kw_only=True`。

```python
from dataclasses import dataclass


@dataclass(kw_only=True)
class ScanOptions:
    root: str
    recursive: bool = True
    include_hidden: bool = False
    limit: int = 100


options = ScanOptions(root="notes", include_hidden=True)
print(options)
```

所有这些字段在构造时都通过关键字提供。

这不是验证字段类型的功能，而是限制调用语法，减少参数位置误会。

### 21.2 能不能只让一部分字段必须写名字？

可以按字段设置：

```python
from dataclasses import dataclass, field


@dataclass
class ScanOptions:
    root: str
    recursive: bool = field(default=True, kw_only=True)
    limit: int = field(default=100, kw_only=True)


options = ScanOptions("notes", recursive=False, limit=10)
```

根目录还可以用位置参数，其余选项必须写名字。

这很像很多标准库函数把主要对象放前面，把控制选项放在关键字参数里。

### 21.3 `KW_ONLY` 那个下划线又是什么？

当后面一组字段都应该是关键字专用时，可以写：

```python
from dataclasses import KW_ONLY, dataclass


@dataclass
class ScanOptions:
    root: str
    _: KW_ONLY
    recursive: bool = True
    limit: int = 100


options = ScanOptions("notes", limit=10)
```

`_` 是惯用名称。

真正起作用的是 `KW_ONLY` 标记，标记后的相应字段成为关键字专用。

这个标记本身不是一份需要保存到实例的数据，也不应在同一个类中重复声明多个这样的分隔标记。

### 21.4 有默认值的字段前面，能不能放必填关键字字段？

关键字专用参数有不同的签名组织方式，可以表达一些普通位置参数不能表达的排列。

例如：

```python
from dataclasses import dataclass


@dataclass(kw_only=True)
class RequestOptions:
    timeout: float = 5.0
    endpoint: str


options = RequestOptions(endpoint="/books")
print(options.endpoint)
```

这里 `endpoint` 仍然是必填，只是必须按名字提供。

不要把“关键字参数可以有这种顺序”误解成所有默认字段顺序限制都消失了；普通位置参数仍然需要遵守相应规则。

## 第二十二章：`slots=True`，是在加速，还是在限制属性？

### 22.1 先观察普通实例能临时加属性

```python
from dataclasses import dataclass


@dataclass
class Point:
    x: int
    y: int


point = Point(1, 2)
point.description = "临时添加"
print(point.description)
```

在没有其他限制时，这通常可以运行。

很多普通对象使用实例字典保存属性。

### 22.2 使用槽之后呢？

```python
from dataclasses import dataclass


@dataclass(slots=True)
class Point:
    x: int
    y: int


point = Point(1, 2)

try:
    point.description = "临时添加"
except AttributeError:
    print("这个实例没有保存该额外属性的位置")
```

对这种简单的槽类，额外属性赋值会失败。

槽为预定属性安排存储方式，通常也能减少大量小实例的属性字典开销。

但具体内存与速度收益取决于解释器、字段、继承与使用方式，不能给所有项目承诺固定百分比。

### 22.3 `slots=True` 等于冻结吗？

不等于。

你仍然可以修改已经声明的可写字段：

```python
point.x = 100
print(point.x)
```

槽主要限制和安排属性存储；冻结主要限制普通属性赋值。

需要两种行为时，可以组合 `@dataclass(slots=True, frozen=True)`。

### 22.4 为什么不能再直接用 `vars(point)`？

简单槽实例可能没有普通 `__dict__`。

因此，依靠实例字典导出数据的通用代码可能失败。

这也是前面推荐用 `fields()` 查看数据类结构的原因。

不过，如果父类提供实例字典，子类即使用槽，也可能仍然具有相关动态属性能力。

所以不要只看当前一行装饰器，就对复杂继承对象的存储布局下绝对结论。

### 22.5 槽类有什么版本和继承注意事项？

数据类的 `slots=` 从 Python 3.10 提供。

它会创建并返回新类，这可能影响某些依赖原类身份的高级机制。

在这种类中使用无参数 `super()` 存在已知限制，遇到相关继承需求时，应核对目标版本文档并采用明确的父类调用或合适的双参数形式。

父类的 `__init_subclass__()` 如果要求必须提供参数，也可能与生成新槽类的过程发生冲突。

对于刚开始学习的数据模型，先用简单类完成需求；确实需要槽时，再把相关继承组合在目标环境验证。

### 22.6 `weakref_slot=True` 是什么？

Python 3.11 起，可以配合 `slots=True` 生成支持弱引用所需的槽。

```python
import weakref
from dataclasses import dataclass


@dataclass(slots=True, weakref_slot=True)
class Item:
    name: str


item = Item("缓存对象")
reference = weakref.ref(item)
print(reference() is item)
```

输出为真。

弱引用不会像普通强引用那样单独维持对象存活。

日常文件脚本通常不需要先为这个参数增加复杂度；只有在弱引用缓存等明确场景下，再学习它的生命周期含义。

## 第二十三章：继承以后，字段和初始化怎样组合？

### 23.1 子类可以添加字段

```python
from dataclasses import dataclass


@dataclass
class Entry:
    title: str


@dataclass
class TimedEntry(Entry):
    minutes: int


entry = TimedEntry("学习继承", 30)
print(entry)
```

输出：

```text
TimedEntry(title='学习继承', minutes=30)
```

装饰器收集数据类父类的字段，再结合当前类自己的字段，生成相应方法。

父类字段不是因为没有重复写在子类里就消失了。

### 23.2 父类有默认字段，子类再加必填字段呢？

仍然可能遇到普通参数顺序限制。

例如父类已经有 `completed=False`，子类后面再增加普通必填的 `minutes`，组合后可能形成不合法的签名。

可以调整结构，把某些字段设计成关键字专用，或者放弃不必要的继承。

不要为了继承一两个字段，给整套模型引入很难解释的构造规则。

### 23.3 组合是不是也可以？

当然。

```python
from dataclasses import dataclass


@dataclass
class Author:
    name: str


@dataclass
class Note:
    title: str
    author: Author


note = Note("组合示例", Author("Alice"))
print(note.author.name)
```

笔记拥有一个作者，不需要把笔记设计成作者的子类。

类之间的关系应该表达业务关系，而不是只为了少写几行字段声明。

### 23.4 父类 `__post_init__()` 会自动全部执行吗？

如果子类覆盖这个方法，普通方法覆盖规则仍然成立。

子类需要父类检查时，应明确调用：

```python
from dataclasses import dataclass


@dataclass
class Entry:
    title: str

    def __post_init__(self):
        if not self.title:
            raise ValueError("标题不能为空")


@dataclass
class TimedEntry(Entry):
    minutes: int

    def __post_init__(self):
        super().__post_init__()
        if self.minutes < 0:
            raise ValueError("时长不能为负数")
```

这个例子没有开启 `slots=True`，因此不要把它与上一章讨论的槽类限制混在一起。

父类负责共同规则，子类在其基础上检查自己增加的字段。

### 23.5 父类不是数据类，它的初始化会自动调用吗？

数据类生成的初始化方法不会自动替你调用任意普通父类的 `__init__()`。

如果父类有必要的初始化逻辑，需要在明确的位置调用。

这和收集数据类父类字段不是一件事。

所以混合继承时，先写一个最小例子确认状态在哪里创建，通常比根据“子类总会初始化父类”的模糊说法推断更可靠。

### 23.6 父子类可以随意混用冻结与可变吗？

数据类对冻结与非冻结数据类继承有相应限制，不能任意混合。

原因也容易理解：如果父类型被当成稳定值使用，子类型突然允许修改同样的状态，会破坏很多调用者假设。

日常建模时，通常让一组相关值对象保持一致政策。

如果不同对象职责差别很大，组合往往更清楚。

## 第二十四章：模式匹配能直接拆开数据类吗？

### 24.1 先用字段名匹配

Python 3.10 及以上可以使用结构模式匹配。

```python
from dataclasses import dataclass


@dataclass
class Task:
    title: str
    completed: bool = False


def describe(task):
    match task:
        case Task(title=title, completed=True):
            return f"已完成：{title}"
        case Task(title=title, completed=False):
            return f"待处理：{title}"
        case _:
            return "不是支持的任务"


print(describe(Task("学习模式匹配")))
```

输出：

```text
待处理：学习模式匹配
```

这里的 `Task(...)` 位于模式中，不是在那个位置创建新对象。

它用于匹配类型及相应属性。

### 24.2 为什么也能写位置形式？

数据类默认会生成相应的 `__match_args__`，告诉位置模式各项对应什么属性。

例如简单类可能支持：

```python
match Task("复习", True):
    case Task(title, True):
        print(title)
```

但位置形式依赖字段的对应关系。

字段很多，或者模型需要长期演进时，关键字模式更容易读，也更不容易因顺序调整引起误会。

### 24.3 关键字专用字段会进入位置匹配吗？

不会进入自动生成的位置匹配参数元组。

这与构造时要求显式字段名的设计是一致的。

你仍然可以通过字段名匹配相关属性。

如果完全不希望自动生成位置匹配规则，可以设置 `match_args=False`。

但这不代表类型不能再用关键字模式匹配。

## 第二十五章：字段的 `metadata`，能自动做验证吗？

### 25.1 先看怎样附加说明

```python
from dataclasses import dataclass, field, fields


@dataclass
class Reading:
    title: str = field(metadata={"label": "标题"})
    minutes: int = field(metadata={"label": "分钟数", "minimum": 0})


for info in fields(Reading):
    print(info.name, info.metadata.get("label"))
```

输出：

```text
title 标题
minutes 分钟数
```

`metadata` 是你或其他工具读取的扩展信息。

数据类不会因为里面有一个叫 `minimum` 的键，就自动执行非负检查。

### 25.2 为什么不自动理解 `minimum`？

因为这个映射没有一套由数据类统一规定的业务词汇。

你的工具可能用 `label` 生成表头，另一个工具可能用带命名空间的键保存格式化规则。

如果标准库擅自执行所有键，就会混淆不同工具的约定。

因此，写下元数据和实现读取它的代码，是两个步骤。

### 25.3 它是只读的吗？

字段公开的元数据映射会通过只读代理呈现。

但是不要把这当成递归冻结所有内部内容的承诺。

如果元数据值里放了列表或其他可变对象，仍然要考虑引用共享。

日常使用时，放简单、稳定的说明数据，最容易理解。

### 25.4 新版本的 `doc=` 是不是必须使用？

Python 3.14 为 `field()` 增加字段文档参数。

它适合保存字段说明，但不是旧版本都有的参数。

本篇完整程序以常见的 3.11 语法为基础，不依赖它运行。

如果项目需要兼容多个解释器版本，应先确认最低版本，而不是把最新版签名里的所有参数都写进去。

### 25.5 我可以运行时动态生成一个数据类吗？

模块提供 `make_dataclass()`。

```python
from dataclasses import make_dataclass

Point = make_dataclass("Point", [("x", int), ("y", int)])
point = Point(1, 2)
print(point)
```

这适合某些运行时才知道结构的工具。

普通固定模型直接用 `class` 通常更容易补全、阅读和维护。

不要为了显得灵活，把本来明确的几个字段改成层层动态生成。

Python 3.14 又增加了相关工厂的 `decorator=` 参数，旧环境不能直接照搬带该参数的例子。

## 第二十六章：完整实践一，把任务更新写成明确的数据变化

### 26.1 先说清楚我们想要什么

你有一个学习任务清单。

每个任务保存编号、标题、预计分钟数、完成状态和标签。

你希望已经创建的任务保持稳定，修改时返回新对象。

完成一个任务，不应该偷偷改掉其他地方正在查看的旧版本。

同一个编号不能重复添加，标签也不重复保存。

这里先只处理内存数据，不读文件、不访问网络，专心把对象行为讲清楚。

### 26.2 完整程序

保存为 `task_values.py`：

```python
from dataclasses import dataclass, replace


@dataclass(frozen=True, slots=True)
class StudyTask:
    task_id: int
    title: str
    estimated_minutes: int
    completed: bool = False
    tags: tuple[str, ...] = ()

    def __post_init__(self):
        if type(self.task_id) is not int or self.task_id <= 0:
            raise ValueError("task_id 必须是正整数")
        if not isinstance(self.title, str):
            raise ValueError("title 必须是字符串")
        title = self.title.strip()
        if not title or len(title) > 100:
            raise ValueError("title 清理后长度必须在 1 到 100 之间")
        if type(self.estimated_minutes) is not int:
            raise ValueError("estimated_minutes 必须是整数")
        if not 1 <= self.estimated_minutes <= 24 * 60:
            raise ValueError("estimated_minutes 必须在 1 到 1440 之间")
        if type(self.completed) is not bool:
            raise ValueError("completed 必须是真正的布尔值")
        if type(self.tags) is not tuple:
            raise ValueError("tags 必须是元组")
        if any(not isinstance(tag, str) or not tag.strip() for tag in self.tags):
            raise ValueError("每个标签必须是非空字符串")

        tags = tuple(dict.fromkeys(tag.strip() for tag in self.tags))
        object.__setattr__(self, "title", title)
        object.__setattr__(self, "tags", tags)

    def complete(self):
        return replace(self, completed=True)

    def with_tag(self, tag):
        return replace(self, tags=(*self.tags, tag))


def add_task(tasks, task):
    if any(item.task_id == task.task_id for item in tasks):
        raise ValueError(f"重复任务编号：{task.task_id}")
    return [*tasks, task]


def complete_task(tasks, task_id):
    if type(task_id) is not int or task_id <= 0:
        raise ValueError("查询任务编号必须是正整数")

    updated = []
    found = False

    for task in tasks:
        if task.task_id == task_id:
            updated.append(task.complete())
            found = True
        else:
            updated.append(task)

    if not found:
        raise KeyError(f"没有任务编号：{task_id}")

    return updated


def summarize_tasks(tasks):
    return {
        "total": len(tasks),
        "completed": sum(task.completed for task in tasks),
        "remaining_minutes": sum(
            task.estimated_minutes for task in tasks if not task.completed
        ),
    }


def main():
    tasks = []
    tasks = add_task(tasks, StudyTask(1, "  学习文件处理  ", 45, tags=("Python",)))
    tasks = add_task(tasks, StudyTask(2, "复习路径", 30, tags=("Python", "文件")))
    updated = complete_task(tasks, 1)

    print(tasks[0].completed)
    print(updated[0].completed)
    print(updated[1] is tasks[1])
    print(summarize_tasks(updated))


if __name__ == "__main__":
    main()
```

### 26.3 运行以后应该看到什么？

```text
False
True
True
{'total': 2, 'completed': 1, 'remaining_minutes': 30}
```

第一项说明旧任务仍然没有完成。

第二项说明新列表里的对应任务已经完成。

第三项说明没有变化的第二个任务可以继续共享同一个值对象。

最后的报告只把未完成任务的预计时长加起来。

### 26.4 为什么标题清理以后，还要再检查长度？

原始输入可能只是几个空格。

如果只检查原始字符串长度，它看起来不是空字符串，却没有实际标题。

先确认类型，再清理，再验证最终内容，是这里选择的顺序。

这个模型主动清理标题两端空白；它没有清理正文，因为根本没有正文这个字段。

规范化应围绕具体字段，不应该不分含义地套用到所有数据。

### 26.5 `tuple(dict.fromkeys(...))` 在做什么？

先把每个标签两端空白去掉。

再用字典键不重复、保持插入顺序的特性去重。

最后把键转成元组。

例如：

```python
task = StudyTask(3, "去重实验", 20, tags=(" Python ", "文件", "Python"))
print(task.tags)
```

输出：

```text
('Python', '文件')
```

这个例子把标签当作区分大小写的文本。

如果你希望 `python` 与 `Python` 等价，还需要明确大小写归一规则。

不要因为用了集合或字典，就假定已经理解所有人类文本的相等含义。

### 26.6 为什么标签修改也要走 `replace()`？

`with_tag()` 把旧标签与新标签组合，再创建新对象。

新对象会重新经过相同的验证和去重过程。

所以传入空标签会失败，重复标签不会增加条目。

你不需要在每个方法里复制一遍完整标签规则。

但是构造规则仍然应该保持没有外部副作用，否则每次替换都会重复产生这些副作用。

### 26.7 列表也没有原地修改，是不是复制了全部对象？

没有。

`complete_task()` 创建新列表，但对没有变化的任务继续保存原引用。

对已经设计为稳定值的对象，这是很常见的共享方式。

新列表与旧列表的容器不同，里面某些元素可以相同。

要解释复制，就必须说明复制的是哪一层。

### 26.8 为什么完成不存在的编号要抛异常？

因为调用者请求了一个明确动作，却没有对应目标。

悄悄返回原列表，容易让调用者误以为操作已经成功。

本例用 `KeyError` 表达编号查找失败。

查找之前，函数也要求编号是精确的正整数。

这样 `True` 或 `1.0` 不会因为与整数一比较相等，就意外完成编号为一的任务。

编号格式不合约定与格式正确但没有对应任务，分别通过 `ValueError` 和 `KeyError` 表达。

实际工具也可以返回专门的结果对象，但应该让“找到并完成”和“没有找到”可以区分。

### 26.9 这个内存模型解决数据库并发了吗？

没有。

它只清楚表达内存中的数据变化。

如果两个进程同时读取文件、各自更新再保存，后保存的一份仍然可能覆盖先保存的一份。

数据库事务、版本检查和文件发布政策属于下一层问题。

不要把不可变对象的局部好处扩大成整个存储系统的一致性保证。

## 第二十七章：完整实践二，从 JSON 文件导入任务，再可靠保存

### 27.1 文件格式先定下来

我们不直接保存对象的打印结果。

文件顶层明确包含格式版本和任务数组：

```json
{
  "version": 1,
  "tasks": [
    {
      "task_id": 1,
      "title": "学习文件操作",
      "estimated_minutes": 45,
      "completed": false,
      "tags": ["Python"]
    }
  ]
}
```

内部标签使用元组，JSON 数组读取后则是列表。

导入函数要明确完成这一步转换。

本实践拒绝未知字段、重复 JSON 键、重复任务编号，以及超过规定大小的文件。

这些限制服务于一份小型个人任务清单，不是所有任务系统的通用上限。

### 27.2 完整文件处理模块

把上一章的完整程序保存为 `task_values.py`。

下面保存为同目录的 `task_store.py`，明确导入刚才已经讲解过的模型。

```python
import json
import os
import tempfile
from pathlib import Path

from task_values import StudyTask


TASK_KEYS = {"task_id", "title", "estimated_minutes", "completed", "tags"}


def unique_object(pairs):
    result = {}
    for key, value in pairs:
        if key in result:
            raise ValueError(f"JSON 对象有重复键：{key}")
        result[key] = value
    return result


def reject_constant(text):
    raise ValueError(f"不接受非标准 JSON 数值：{text}")


def task_from_dict(data):
    if not isinstance(data, dict):
        raise ValueError("任务必须是对象")
    if set(data) != TASK_KEYS:
        raise ValueError("任务字段必须与文件格式完全一致")
    if not isinstance(data["tags"], list):
        raise ValueError("文件里的 tags 必须是数组")

    return StudyTask(
        task_id=data["task_id"],
        title=data["title"],
        estimated_minutes=data["estimated_minutes"],
        completed=data["completed"],
        tags=tuple(data["tags"]),
    )


def task_to_dict(task):
    return {
        "task_id": task.task_id,
        "title": task.title,
        "estimated_minutes": task.estimated_minutes,
        "completed": task.completed,
        "tags": list(task.tags),
    }


def ensure_unique_tasks(tasks):
    seen = set()
    for task in tasks:
        if task.task_id in seen:
            raise ValueError(f"重复任务编号：{task.task_id}")
        seen.add(task.task_id)


def load_tasks(path, max_bytes=1_000_000):
    if type(max_bytes) is not int or max_bytes < 0:
        raise ValueError("max_bytes 必须是非负整数")

    with Path(path).open("rb") as source:
        payload = source.read(max_bytes + 1)
    if len(payload) > max_bytes:
        raise ValueError("任务文件超过允许大小")

    data = json.loads(
        payload.decode("utf-8"),
        object_pairs_hook=unique_object,
        parse_constant=reject_constant,
    )
    if not isinstance(data, dict) or set(data) != {"version", "tasks"}:
        raise ValueError("文件顶层结构不合法")
    if type(data["version"]) is not int or data["version"] != 1:
        raise ValueError("不支持的文件版本")
    if not isinstance(data["tasks"], list) or len(data["tasks"]) > 10_000:
        raise ValueError("tasks 必须是最多一万条的数组")

    tasks = []
    for index, item in enumerate(data["tasks"]):
        try:
            tasks.append(task_from_dict(item))
        except (TypeError, ValueError) as error:
            raise ValueError(f"tasks[{index}]：{error}") from error

    ensure_unique_tasks(tasks)
    return tasks


def save_tasks(path, tasks):
    tasks = list(tasks)
    if len(tasks) > 10_000:
        raise ValueError("任务数量超过本例限制")
    if any(not isinstance(task, StudyTask) for task in tasks):
        raise TypeError("tasks 只能包含 StudyTask")
    ensure_unique_tasks(tasks)

    data = {"version": 1, "tasks": [task_to_dict(task) for task in tasks]}
    text = json.dumps(data, ensure_ascii=False, indent=2, allow_nan=False) + "\n"
    if len(text.encode("utf-8")) > 1_000_000:
        raise ValueError("导出结果超过本例文件大小限制")

    path = Path(path).absolute()
    path.parent.mkdir(parents=True, exist_ok=True)
    temporary = None

    try:
        with tempfile.NamedTemporaryFile(
            mode="w", encoding="utf-8", newline="\n", delete=False,
            dir=path.parent, prefix=path.name + ".", suffix=".tmp",
        ) as target:
            temporary = Path(target.name)
            target.write(text)
            target.flush()
            os.fsync(target.fileno())

        temporary.replace(path)
    finally:
        if temporary is not None:
            temporary.unlink(missing_ok=True)
```

### 27.3 怎样第一次实际运行？

再创建同目录的 `store_demo.py`：

```python
from pathlib import Path

from task_store import load_tasks, save_tasks
from task_values import StudyTask


path = Path("dataclass_practice/tasks.json")
tasks = [
    StudyTask(1, "学习文件操作", 45, tags=("Python",)),
    StudyTask(2, "复习日期", 20, completed=True),
]

save_tasks(path, tasks)
loaded = load_tasks(path)
print(loaded == tasks)
print(type(loaded[0].tags).__name__)
```

输出：

```text
True
tuple
```

文件保存数组，读回后恢复成内部元组。

数据相等，不表示仍然是原来的那几个对象；导入过程创建了新实例。

### 27.4 为什么不在每个函数里重复检查所有字段？

文件层负责顶层结构、版本、大小、JSON 对象形式以及数组形式。

模型负责一条任务内部的编号、标题、时长、状态和标签规则。

集合层负责跨任务的编号唯一性。

这样每条规则都有明确位置。

如果以后把任务改成从 CSV 导入，仍然可以复用同一个模型，而不需要再发明一套标题规则。

### 27.5 `**data` 为什么没有出现在这里？

完全可以在验证好字段后使用关键字解包。

这里选择显式写每个字段，是为了让 JSON 列表转元组的地方一眼可见，也让内部模型与外部字段的对应关系清楚。

如果以后外部格式把 `task_id` 改成 `id`，这个转换函数就是明确的适配位置。

不需要为了省几行就假设内部字段与外部协议永远同名。

### 27.6 缺失文件为什么没有自动返回空列表？

本例让 `FileNotFoundError` 保持可见。

第一次创建任务库，和用户给错路径导致没有找到文件，是两种不同情景。

调用者可以在明确的“初始化新任务库”操作里决定采用空列表。

底层读取函数不必擅自替所有调用者作这个决定。

### 27.7 文件里两条任务有同一个编号，为什么对象构造检查不够？

一条任务自己不知道别的任务有哪些编号。

模型只能检查编号本身是正整数。

跨记录唯一性需要在集合层判断。

同样，一个学生分数合法，不代表成绩表里没有重复学生；一条订单合法，也不代表同一订单号没有重复导入。

这个区分比把所有验证统称为“类型检查”更有用。

### 27.8 临时文件替换能保证什么？

先把完整文本生成好，避免序列化失败时先清空正式文件。

再写同目录临时文件，关闭以后替换正式路径，减少读者看见半份输出的机会。

但它不解决两个写入者互相覆盖、不保留全部原元数据，也没有为目录条目提供完整的崩溃持久性协议。

清理临时文件本身也可能失败。

如果你需要多条记录的并发更新和事务，前面的 SQLite 教材更适合继续学习。

### 27.9 如果不想覆盖已存在文件呢？

本例的 `save_tasks()` 明确使用替换政策。

如果业务要求“名字已存在就拒绝”，就必须重新设计最终发布动作。

不能只在前面加一个 `exists()`，就宣称在并发环境下绝不会覆盖。

同样，直接改成 `x` 可以提供排他创建，但写到一半失败又会留下部分目标。

保存政策需要同时考虑覆盖、完整性与并发边界，而不是把某一种模式当成适用于所有需求的答案。

## 第二十八章：完整实践三，把 CSV 文本变成学习记录

### 28.1 输入是字符串，对象需要的是日期和整数

假设 `sessions.csv` 内容如下：

```csv
day,title,minutes
2026-11-01,学习 open,40
2026-11-01,复习 pathlib,20
2026-11-02,学习 dataclasses,50
2026-11-02,错误记录,-5
```

我们希望逐条导入。

合法记录参与统计，错误记录保留来源行号和原因。

这次选择“继续导入其他行”的政策，与上一章整份任务文件必须全部合法的政策不同。

不是换了数据类以后，错误处理政策就会自动统一。

### 28.2 完整程序

保存为 `session_report.py`：

```python
import csv
from collections import defaultdict
from dataclasses import dataclass
from datetime import date
from pathlib import Path


@dataclass(frozen=True, slots=True)
class Session:
    day: date
    title: str
    minutes: int

    def __post_init__(self):
        if type(self.day) is not date:
            raise ValueError("day 必须是日期对象")
        if not isinstance(self.title, str) or not self.title.strip():
            raise ValueError("title 必须是非空字符串")
        if type(self.minutes) is not int or not 1 <= self.minutes <= 1440:
            raise ValueError("minutes 必须是 1 到 1440 的整数")
        object.__setattr__(self, "title", self.title.strip())


@dataclass(frozen=True, slots=True)
class RowProblem:
    line_number: int
    message: str


def parse_session(row):
    if any(value is None for value in row.values()):
        raise ValueError("记录缺少列")
    if None in row:
        raise ValueError("记录包含多余列")

    return Session(
        day=date.fromisoformat(row["day"].strip()),
        title=row["title"],
        minutes=int(row["minutes"].strip()),
    )


def load_sessions(path):
    sessions = []
    problems = []

    with Path(path).open("r", encoding="utf-8-sig", newline="") as source:
        reader = csv.DictReader(source, strict=True)
        if reader.fieldnames != ["day", "title", "minutes"]:
            raise ValueError("表头必须依次为 day,title,minutes")

        for row in reader:
            try:
                session = parse_session(row)
            except (TypeError, ValueError) as error:
                problems.append(RowProblem(reader.line_num, str(error)))
            else:
                sessions.append(session)

    return sessions, problems


def daily_totals(sessions):
    totals = defaultdict(int)
    for session in sessions:
        totals[session.day] += session.minutes
    return dict(sorted(totals.items()))


def main():
    sessions, problems = load_sessions("sessions.csv")
    print("合法记录：", len(sessions))
    print("错误记录：", len(problems))
    for day, minutes in daily_totals(sessions).items():
        print(day.isoformat(), minutes)
    for problem in problems:
        print(f"第 {problem.line_number} 行：{problem.message}")


if __name__ == "__main__":
    main()
```

### 28.3 对给定输入，输出应该怎样理解？

正常情况下会看到：

```text
合法记录： 3
错误记录： 1
2026-11-01 60
2026-11-02 50
第 5 行：minutes 必须是 1 到 1440 的整数
```

十一月一日两次学习加起来六十分钟。

十一月二日的负数记录没有进入统计，所以只保留五十分钟。

错误报告中的第五行包含了表头在内的物理行号。

### 28.4 为什么转换写在 `parse_session()`，规则写在模型里？

CSV 把字段交给我们时，它们主要是字符串。

`date.fromisoformat()` 和 `int()` 负责把文本解释为内部值。

模型再检查内部值是否符合范围与内容规则。

例如，`int("-5")` 可以成功，但负五分钟仍然违反本例规则。

能转换与业务合法不是同一件事。

### 28.5 为什么没有在每行里捕获所有异常？

我们只把预期的记录转换与验证问题作为坏行保存。

如果 CSV 自身出现严重语法错误，`csv.Error` 会从遍历处传播出来。

如果文件编码损坏，解码错误也不会被伪装成普通字段值错误。

因为这时“跳过一条就能安全继续”未必成立，解析器可能已经无法正确识别后续记录边界。

按错误层次选择能否继续，通常比把所有异常都塞进一个坏行列表更准确。

### 28.6 `reader.line_num` 是记录序号吗？

它表示解析器已经读取的物理行数量。

如果一个带引号的字段跨越多行，某条记录可能占据多个物理行。

因此，这里记录的是读取完当前记录时的行号，不能无条件等同于“第几条数据”。

需要更完整的定位时，还可以保存每条记录开始前的计数，或者单独提供记录序号并明确单位。

### 28.7 为什么错误也用数据类？

`RowProblem` 给错误结构一个名字，避免在多个函数之间传递含义不清的二元组。

看到 `problem.line_number`，比看到 `problem[0]` 更容易知道是什么。

但如果只在两行代码中临时使用一个简单二元组，未必需要专门创建类。

类型的价值来自表达职责和减少误解，而不是类越多越好。

### 28.8 这还是流式程序吗？

输入读取按记录逐步进行。

但是合法记录和错误记录都被保存进列表，所以总内存仍然随数据增长。

如果只需要每日总时长，可以边读边累计，不保存全部 `Session`。

如果错误特别多，可以只保存前若干条示例，并另计总错误数量，或者写入独立报告。

“使用迭代器读取”与“整个流程内存恒定”必须分开判断。

## 第二十九章：完整实践四，配置覆盖以后再生成明确对象

### 29.1 我有默认值、配置文件和命令行，最后应该保存什么？

假设一个目录扫描工具有三个选项。

扫描哪个目录。

是否递归。

最多处理多少个文件。

它们可能来自程序默认值、配置文件和命令行。

你可以在各处传一个随时变化的字典，也可以先合并来源，验证以后创建一个明确的配置对象。

本实践采用后一种方式。

生成配置对象以后，业务函数只接收这份最终配置，不需要再到处读取环境或重新判断命令行有没有传入。

### 29.2 路径相对谁，先用具体情景说清楚

你的配置文件位于：

```text
D:\project\config\scan.json
```

里面写：

```json
{"root": "../notes"}
```

这份工具约定，配置里的相对路径，以配置文件所在目录为基准。

因此，它表示：

```text
D:\project\notes
```

但是你在终端中位于：

```text
C:\Users\Alice
```

并通过命令行明确指定：

```text
--root drafts
```

命令行里的相对路径，则按这次终端的当前工作目录解释，表示：

```text
C:\Users\Alice\drafts
```

同样叫 `root`，不同来源有不同的基准。

这些规则需要转换层明确落实，数据类不会因为字段注解是 `Path` 就自动知道。

### 29.3 完整程序

下面例子不实际扫描文件，只生成最终配置，因此可以专心观察覆盖规则。

```python
from dataclasses import dataclass, replace
from pathlib import Path


@dataclass(frozen=True, kw_only=True)
class ScanOptions:
    root: Path
    recursive: bool = True
    limit: int = 100

    def __post_init__(self):
        if not isinstance(self.root, Path):
            raise TypeError("root 必须是当前平台的 Path")
        if type(self.recursive) is not bool:
            raise ValueError("recursive 必须是布尔值")
        if type(self.limit) is not int or not 1 <= self.limit <= 100_000:
            raise ValueError("limit 必须是 1 到 100000 的整数")


def normalize_changes(raw, base):
    if not isinstance(raw, dict):
        raise ValueError("配置来源必须是字典")
    unknown = set(raw) - {"root", "recursive", "limit"}
    if unknown:
        raise ValueError(f"未知配置项：{sorted(unknown)}")

    changes = dict(raw)
    if "root" in changes:
        value = changes["root"]
        if not isinstance(value, (str, Path)):
            raise ValueError("root 必须是路径文本或 Path")
        if isinstance(value, str) and not value.strip():
            raise ValueError("root 不能为空路径文本")
        path = Path(value)
        if not path.is_absolute():
            path = Path(base) / path
        changes["root"] = path.resolve()

    return changes


def build_options(default_root, file_values, config_path, cli_values, cwd):
    options = ScanOptions(root=Path(default_root).resolve())

    file_base = Path(config_path).resolve().parent
    from_file = normalize_changes(file_values, file_base)
    options = replace(options, **from_file)

    from_cli = normalize_changes(cli_values, Path(cwd).resolve())
    options = replace(options, **from_cli)
    return options


def main():
    from tempfile import TemporaryDirectory

    with TemporaryDirectory(prefix="dataclass-config-") as folder:
        base = Path(folder)
        file_values = {"root": "../notes", "recursive": True, "limit": 200}
        cli_values = {"recursive": False, "limit": 10}

        options = build_options(
            default_root=base / "default_notes",
            file_values=file_values,
            config_path=base / "config" / "scan.json",
            cli_values=cli_values,
            cwd=base / "terminal",
        )

        print(options.root == base / "notes")
        print(options.recursive)
        print(options.limit)


if __name__ == "__main__":
    main()
```

### 29.4 输出为什么是这三个值？

```text
True
False
10
```

配置里的根目录覆盖程序默认目录。

命令行没有提供 `root`，所以继续保留配置文件得到的位置。

命令行明确提供 `recursive=False`，因此正确覆盖原来的真值。

命令行明确提供 `limit=10`，因此覆盖配置文件里的二百。

### 29.5 为什么不能用 `cli_value or file_value`？

因为 `False` 是一个有效的明确设置。

如果用布尔真假选择来源，用户想关闭递归，结果可能又被文件中的真值覆盖回来。

本例直接看键有没有出现。

键没有出现，表示这个来源没有提供该项。

键出现而值是假，表示明确选择关闭。

这也是为什么命令行解析阶段常常需要保留“没有提供”与“提供了默认值”之间的区别。

### 29.6 两次 `replace()` 会不会多余？

它让每个来源应用以后，都经过相同模型规则。

如果配置文件写了错误类型，即使命令行后来可能覆盖它，我们也能尽早报告这份配置本身的问题。

这是一项明确政策。

另一种工具可能选择先合并所有来源，再只验证最终结果。

两种政策各有用途，关键是不要无意间让错误配置被静默掩盖。

### 29.7 为什么没有在模型里检查目录一定存在？

这份模型表达最终选项。

本例允许先生成配置，再由真正扫描的入口检查目录和权限。

路径可能在生成配置之后被创建，也可能在扫描之前消失。

即使模型构造时检查过存在，后续 I/O 仍然需要处理失败。

因此，对这种工具，把数据范围规则与具体资源访问分开，比较容易解释生命周期。

### 29.8 如果以后增加一个选项，需要改哪里？

先在模型里声明字段与默认值。

如果有范围规则，在模型验证里补上。

然后在转换层声明哪些外部字段被允许，以及是否需要解析。

最后为来源优先级和错误情况补一个真实例子。

字段增加会改变构造、比较、显示等自动行为，所以仍然需要检查整个模型的语义。

自动生成方法减少了机械重复，不代表模型演进从此不需要思考。

## 第三十章：字典、具名元组、数据类和 Pydantic，怎样选择？

### 30.1 我只是临时组织几个键值

字典很合适。

尤其是键本来就是动态的，例如“每个扩展名对应文件数量”。

这种关系不是固定的三四个字段，硬改成数据类反而不自然。

数据类擅长表达稳定结构，不负责取代所有映射。

### 30.2 我需要一个紧凑、具有元组行为的记录

具名元组或 `NamedTuple` 可以考虑。

它们通常保留元组的按位置访问、解包等行为。

数据类默认不因为只有两个字段，就自动支持 `x, y = point`。

你可以自行定义迭代行为，但那是额外接口选择。

如果调用方明确依赖元组兼容性，就应该把这个需求作为选择依据。

### 30.3 我的数据结构固定，还要添加方法

数据类经常很合适。

例如扫描结果、导入错误、配置值、任务记录和统计报告。

它们需要有名字的字段、有意义的显示和比较，可能还有少量与数据直接相关的方法。

这正是本篇反复使用的场景。

### 30.4 我主要面对大量外部、不可信或形态复杂的输入

如果你需要嵌套转换、详细错误路径、字段别名、严格模式、复杂联合类型或 JSON Schema，Pydantic 可以减少很多手写输入处理。

但它是第三方依赖，验证、转换与序列化也有需要学习的明确规则。

不能仅凭“这个库会验证”就假设所有默认值、后续修改和复制更新都会自动重新检查。

对应的 [Pydantic 教材](Python-Pydantic库.md) 会逐一做实验。

### 30.5 能不能外部用 Pydantic，内部用数据类？

可以。

例如，外部接口先把不规则输入解析成明确值，再创建内部领域对象。

但是每增加一层模型，就要维护字段转换和职责边界。

小脚本只有一份简单输入时，未必需要两套相似模型。

当两层确实承担不同职责，例如外部字段可变、内部规则稳定时，分开才更有价值。

### 30.6 普通类是不是就不值得用了？

当然仍然值得。

如果对象主要管理资源、连接、线程或复杂行为，普通类往往更直接。

例如，一个数据库客户端的核心问题是连接生命周期和执行动作，而不只是几个公开字段的相等比较。

给它自动生成按连接对象比较的 `__eq__()`，可能根本没有合理业务含义。

先看对象的职责，再选择是否自动生成某些方法。

## 第三十一章：怎样验证数据类，才不是只验证“代码能运行”？

### 31.1 先预测正常数据应该得到什么

拿第二十六章的模型举例。

输入标题两端带空格，构造后应清理。

输入重复标签，构造后应按原顺序去重。

调用 `complete()`，旧对象不变，新对象完成。

这些都是明确行为，适合写成断言。

```python
from task_values import StudyTask

task = StudyTask(1, "  学习对象  ", 30, tags=("Python", "Python"))
assert task.title == "学习对象"
assert task.tags == ("Python",)

finished = task.complete()
assert task.completed is False
assert finished.completed is True
assert task is not finished
```

这些断言验证的是约定，而不是去检查装饰器内部究竟用了哪几行实现。

### 31.2 再验证边界，而不只写一份正确输入

```python
from task_values import StudyTask

for minutes in (0, -1, 1441, True, "30"):
    try:
        StudyTask(1, "边界实验", minutes)
    except ValueError:
        pass
    else:
        raise AssertionError(f"意外接受了时长：{minutes!r}")
```

这里每一个输入都对应一个真实误用方向。

零与负数检查下界。

一千四百四十一检查上界。

布尔值检查 `bool` 与 `int` 的关系。

字符串检查模型有没有悄悄执行未约定的转换。

### 31.3 文件程序应该检查旧数据有没有保留

假设保存前发现重复任务编号。

不仅要观察它抛了异常，还应检查旧文件内容没有被清空或替换。

同样，下载、配置更新和 CSV 导出中，失败后的已有文件状态也是重要结果。

只断言“出现 ValueError”，没有检查关键资料是否保留，验证就可能遗漏真正关心的行为。

### 31.4 用临时目录，为什么更适合练习？

你可以为每次实验准备独立输入和输出。

实验结束自动清理，重复运行也不依赖上一轮留下的文件。

例如：

```python
from pathlib import Path
from tempfile import TemporaryDirectory

from task_store import load_tasks, save_tasks
from task_values import StudyTask

with TemporaryDirectory(prefix="dataclass-check-") as folder:
    path = Path(folder) / "tasks.json"
    expected = [StudyTask(1, "独立实验", 15)]
    save_tasks(path, expected)
    assert load_tasks(path) == expected
```

这个实验有真实文件读写，却不需要碰你的现有笔记库。

### 31.5 每个字段都要单独写一百个测试吗？

不需要为了数量重复相同结论。

先覆盖你自己规定的规则、重要边界和曾经容易误解的行为。

如果换了 Python 版本，尤其用了槽、继承或特殊比较对象，再针对相关差异运行验证。

对于数据类自动生成的基础机制，理解公开约定通常比把标准库内部实现重新测试一遍更有用。

## 第三十二章：练习一至四，把字段与默认值想清楚

### 32.1 练习一：创建一条书籍记录

要求：书名和作者必填，页数默认零。

创建两份内容相同的对象，分别比较 `==` 和 `is`。

先预测两个结果，再运行。

参考答案：

```python
from dataclasses import dataclass


@dataclass
class Book:
    title: str
    author: str
    pages: int = 0


first = Book("Python 基础", "Alice")
second = Book("Python 基础", "Alice")
print(first == second)
print(first is second)
```

结果依次是真、假。

如果你接着传入 `pages="很多"`，默认数据类不会自动替你验证这个字段。

这也是本练习需要继续思考的边界。

### 32.2 练习二：每个购物清单都有独立列表

要求：清单保存拥有者和商品名称列表。

只给第一份清单追加商品，第二份不受影响。

参考答案：

```python
from dataclasses import dataclass, field


@dataclass
class ShoppingList:
    owner: str
    items: list[str] = field(default_factory=list)


first = ShoppingList("Alice")
second = ShoppingList("Bob")
first.items.append("笔记本")
assert first.items == ["笔记本"]
assert second.items == []
assert first.items is not second.items
```

追问：如果构造时主动把同一个 `items` 列表传给两者呢？

那就会共享，因为工厂只负责未提供该值时的默认构造。

### 32.3 练习三：可为空和可省略

要求：用户昵称可以是字符串或 `None`。

第一版要求调用者必须明确提供，第二版允许省略。

参考答案：

```python
from dataclasses import dataclass


@dataclass
class RequiredNickname:
    nickname: str | None


@dataclass
class OptionalNickname:
    nickname: str | None = None


print(RequiredNickname(None))
print(OptionalNickname())
```

区别在默认值，不在联合类型注解。

### 32.4 练习四：让两个布尔开关必须写名字

要求：目录参数允许位置传入，递归与包含隐藏文件必须关键字传入。

参考答案：

```python
from dataclasses import KW_ONLY, dataclass


@dataclass
class Scan:
    root: str
    _: KW_ONLY
    recursive: bool = True
    include_hidden: bool = False


options = Scan("notes", recursive=False, include_hidden=True)
print(options)
```

这里限制的是调用方式，没有自动把字符串 `"false"` 转换为布尔值。

## 第三十三章：练习五至八，把比较、冻结和复制分开

### 33.1 练习五：检查时间不影响文档相等

要求：标题、正文都相同的文档比较相等，即使最近检查时间不同。

参考答案：

```python
from dataclasses import dataclass, field


@dataclass
class Document:
    title: str
    body: str
    checked_at: str = field(compare=False)


assert Document("A", "正文", "09:00") == Document("A", "正文", "10:00")
assert Document("A", "正文", "09:00") != Document("A", "修改", "09:00")
```

不要把正文也排除，否则两份真实内容不同的文档可能被误判为相等。

### 33.2 练习六：坐标放进集合去重

要求：坐标创建后不允许普通字段修改，相同数值去重。

参考答案：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Point:
    x: int
    y: int


points = {Point(1, 2), Point(1, 2), Point(2, 3)}
assert len(points) == 2
```

这里字段是整数，可以参与生成的哈希。

如果把某字段改成列表，就要重新考虑它的哈希与相等含义。

### 33.3 练习七：给不可变任务改标题

要求：原任务保持不变，生成新任务。

参考答案：

```python
from dataclasses import dataclass, replace


@dataclass(frozen=True)
class Task:
    title: str
    completed: bool = False


old = Task("第一版")
new = replace(old, title="第二版")
assert old.title == "第一版"
assert new.title == "第二版"
assert old is not new
```

如果有后处理验证，新对象仍然需要经过它。

### 33.4 练习八：观察浅层共享

要求：说明下面代码运行后，两个对象的标签分别是什么。

```python
from dataclasses import dataclass, replace


@dataclass
class Note:
    title: str
    tags: list[str]


first = Note("A", ["Python"])
second = replace(first, title="B")
second.tags.append("文件")
```

答案是两者都能看到新增的文件标签。

因为 `replace()` 创建新实例，但未替换的列表值仍然共享引用。

如果想隔离，可以构造 `replace(first, title="B", tags=list(first.tags))`。

这只是复制当前标签列表；如果列表元素又是可变对象，还要继续判断需要复制到哪一层。

## 第三十四章：练习九至十二，把转换放在正确边界

### 34.1 练习九：导出公开用户信息

要求：内部用户模型保存名称和令牌，导出时只包含名称。

不能仅依赖 `repr=False`。

参考答案：

```python
from dataclasses import dataclass, field


@dataclass
class User:
    name: str
    token: str = field(repr=False)


def public_user(user):
    return {"name": user.name}


user = User("Alice", "practice-token")
assert public_user(user) == {"name": "Alice"}
```

导出规则明确列出允许输出的字段，而不是假设某个显示选项能覆盖所有转换函数。

### 34.2 练习十：日期字符串转成对象

要求：输入字典包含 `day` 和 `minutes`，创建日期与整数类型的记录。

参考答案：

```python
from dataclasses import dataclass
from datetime import date


@dataclass
class Session:
    day: date
    minutes: int


def parse_session(data):
    if not isinstance(data, dict) or set(data) != {"day", "minutes"}:
        raise ValueError("记录字段不正确")
    if not isinstance(data["day"], str):
        raise ValueError("day 必须是日期字符串")
    if type(data["minutes"]) is not int or data["minutes"] < 0:
        raise ValueError("minutes 必须是非负整数")
    return Session(date.fromisoformat(data["day"]), data["minutes"])


session = parse_session({"day": "2026-11-03", "minutes": 30})
assert session.day == date(2026, 11, 3)
```

注意，本练习把规则放在解析函数里，因此直接调用 `Session(...)` 仍然可能绕过它。

如果模型本身也要保证构造规则，应把相应内部值检查放入模型。

### 34.3 练习十一：任务编号重复时报两处位置

要求：给定任务列表，重复编号错误要指出第一次和第二次出现的下标。

参考答案：

```python
def check_unique_ids(tasks):
    origins = {}
    for index, task in enumerate(tasks):
        if task.task_id in origins:
            first = origins[task.task_id]
            raise ValueError(
                f"编号 {task.task_id} 重复：tasks[{first}] 与 tasks[{index}]"
            )
        origins[task.task_id] = index
```

保存来源可以让错误真正帮助修复输入，而不只是告诉用户“数据不合法”。

下标从零开始，面向普通用户时可以改成从一开始的记录序号，但要写清楚单位。

### 34.4 练习十二：明确关闭选项不能被默认值覆盖

要求：文件配置为递归，命令行明确给假，最终必须是假。

参考答案：

```python
from dataclasses import dataclass, replace


@dataclass(frozen=True)
class Options:
    recursive: bool = True
    limit: int = 100


base = Options()
cli = {"recursive": False}
result = replace(base, **cli)
assert result.recursive is False
assert result.limit == 100
```

这个答案保留未提供字段，并接受明确的假值。

如果使用 `cli.get("recursive") or base.recursive`，就会把用户的关闭意图弄丢。

## 第三十五章：看到报错以后，应该先检查什么？

### 35.1 类定义时就出现默认参数顺序错误

先查看当前类及数据类父类中，是否存在普通必填字段排在默认字段之后。

不要只看子类的最后几行。

生成初始化参数时，继承来的字段同样参与排列。

如果这几个控制参数本来就适合写名字，可以考虑关键字专用设计。

### 35.2 创建对象成功，计算时却发现字段类型不对

先回忆：普通数据类没有自动做全面运行时验证。

追踪输入来自 JSON、CSV、命令行还是其他函数。

在转换边界检查实际值，再决定是显式解析、模型后处理，还是使用专门验证工具。

不要在发生错误的每个计算位置都临时加一层 `int(...)`，那会让接受规则分散，而且可能把原本应该拒绝的数据悄悄转换掉。

### 35.3 调用 `hash()` 报列表不可哈希

先确认是否真的需要把整个对象当键。

如果需要，检查参与比较与哈希的字段结构。

冻结对象的内部列表仍然不可哈希，`unsafe_hash=True` 也不会把列表变成元组。

修复应来自数据身份与可变性设计，而不是只切换一个装饰器参数。

### 35.4 `replace()` 报初始化专用参数缺失

查看类有没有无默认值的 `InitVar`。

它并没有被自动保存在旧实例中，新构造又需要这份输入，因此必须明确提供。

如果你的复制经常不应该重新要求原始输入，可以考虑保存必要的最终值，并通过专门工厂处理原始输入。

### 35.5 `asdict()` 在复杂资源字段上失败

先判断这个对象是不是本来就适合当作纯数据模型。

连接、锁、打开的文件或其他运行时资源，可能不能合理深拷贝，也未必适合导出。

可以把资源管理对象与需要保存的配置或结果拆开。

导出需要什么，就明确转换什么，而不是期待递归工具为所有对象推断交换形式。

### 35.6 更新一份对象，另一份也跟着变化

先检查共享的是哪个内部对象。

可能是显式传入同一个列表，也可能是工厂返回共享列表，还可能是 `replace()` 沿用了旧字段引用。

打印最外层两个对象的 `is` 不够，还要检查内部字段的 `is`。

确定共享层次以后，再选择复制容器、使用不可变结构，或者明确承认共享就是设计的一部分。

## 第三十六章：版本、性能和学习顺序，怎样落到自己的项目？

### 36.1 先确认最低 Python 版本

`dataclasses` 从 Python 3.7 进入标准库。

本篇大量例子还使用了 `str | None`、内置容器泛型、关键字专用字段和槽等较新的语法或能力。

因此，不能因为模块从三点七就存在，就断言本篇所有代码都能在三点七原样运行。

文中的完整程序按 Python 3.11 及以上设计。

`field(doc=...)`、`make_dataclass(decorator=...)` 等更新能力只在相关小节说明，没有混入基本实践依赖。

### 36.2 自动生成的方法一定比手写快吗？

不能这样概括。

它首先提供一致、便于维护的常见方法。

运行开销要看具体操作、字段与解释器。

`slots=True` 对大量小对象可能节省存储，但也改变属性能力。

`frozen=True` 带来明确的更新政策，也有相应初始化实现成本。

如果你的程序主要在等待磁盘和网络，先把数据流程写清楚，通常比围绕几个对象创建操作做微优化更有价值。

### 36.3 类字段多了，是不是应该一直加？

如果一个对象同时包含原始输入、解析后数据、界面状态、数据库连接、统计缓存和日志句柄，通常值得重新检查职责。

并不是数据类允许添加字段，就应该把一切放进去。

可以把配置、输入记录、计算结果和资源管理分别建模。

不过也不需要反过来为每一个数字建立一个类；以调用方能否理解和正确使用为准。

### 36.4 刚学完以后，最值得先实践什么？

从自己的脚本里挑一种已经反复出现的固定结构。

例如一条扫描错误、一项下载结果、一份统计摘要。

先声明字段和默认值，观察创建、打印和比较。

再为真正存在的输入规则补验证。

最后根据是否允许变化，选择普通对象、冻结对象或明确修改方法。

不需要一开始就把继承、弱引用、动态类和模式匹配全部塞进同一个练习。

## 第三十七章：需要复习时，用这一张表找入口

| 想解决的问题 | 常见入口 | 同时记住的边界 |
| --- | --- | --- |
| 为固定结构生成常见方法 | `@dataclass` | 默认不执行全面运行时类型验证 |
| 普通默认值 | `field_name: Type = value` | 普通必填参数与默认参数有顺序规则 |
| 每个实例独立的列表 | `field(default_factory=list)` | 显式传入相同列表仍会共享 |
| 构造后检查与规范化 | `__post_init__()` | 自定义初始化不会自动补调用 |
| 派生字段不接受构造参数 | `field(init=False)` | 仍需明确给它赋值 |
| 只在初始化阶段使用输入 | `InitVar` | 不是普通存储字段，替换时可能需重传 |
| 声明类层面的共同信息 | `ClassVar` | 不等于只读，也不自动复制可变内容 |
| 避免普通对象显示某字段 | `field(repr=False)` | 不会自动排除 `asdict()` 导出 |
| 排除附带信息的自动比较 | `field(compare=False)` | 同时影响自动相等与排序 |
| 稳定的字段顺序比较 | `order=True` | 多种展示排序通常更适合 `key=` |
| 限制普通字段重新赋值 | `frozen=True` | 不会递归冻结内部列表 |
| 使用稳定值作为键 | 冻结与可哈希字段设计 | 生成哈希不代表所有字段可哈希 |
| 创建修改后的新对象 | `replace()` | 会重新构造，通常共享未替换的内部引用 |
| 查看声明字段 | `fields()` | 与实例字典不是一回事 |
| 递归转换常见数据结构 | `asdict()`、`astuple()` | 不是任意资源对象的通用序列化 |
| 要求写出参数名称 | `kw_only=True`、`KW_ONLY` | 改变调用语法，不自动转换类型 |
| 固定属性存储 | `slots=True` | 不等于冻结，继承需要核对 |
| 导入或导出文件 | 明确转换与保存函数 | 格式、验证、覆盖和并发各有规则 |

本篇接口依据可继续核对 [Python 数据类文档](https://docs.python.org/3.14/library/dataclasses.html)；设计背景见第一章链接的 PEP 557。

如果接下来想把外部 JSON 转成带复杂约束的模型，可以阅读 [Pydantic](Python-Pydantic库.md)。

如果想理解这些注解如何被静态检查工具使用，可以阅读 [类型注解](Python-typing类型注解.md)。

如果想观察自动生成方法与普通魔法方法之间的关系，也可以回到已有的类与魔法方法笔记，对照本篇的具体输出再运行一次。
