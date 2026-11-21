# Python 类型注解与 typing 超详细教程：从看懂冒号，到泛型、协议和真正运行静态检查

## 零基础先导：类型注解为什么存在，又为什么不替你检查一切？

Python 平时是动态类型语言，你可以传不同类型的对象；大型项目中，函数参数的预期类型如果只靠猜，就很容易误用。

`typing` 是 Python 标准库的**类型提示工具集合**。它帮助表达“这里期望什么形状的数据”，主要服务于阅读、编辑器与静态检查工具；并**不等于自动在每次函数调用时强制检查**。

```python
def total(values: list[int]) -> int:
    return sum(values)

print(total([1, 2, 3]))  # 6
```

逐个解释：`values` 是参数名；冒号后的 `list[int]` 表示“预期包含整数的列表”；`-> int` 表示预期返回整数。这些注解不自动阻止你传 `["a", "b"]`；实际运行时可能由 `sum` 报错，而不是由注解完成验证。

为什么要更复杂的 `Optional`、`Union`、`Protocol`、`TypeVar`？因为真实函数还可能：允许 `None`、接收几种类型、只要求对象具有某些方法、或保持“输入什么类型就输出什么类型”的关系。它们是在表达**约束关系**，不是新的运行时容器。

现代 Python 中可以直接写 `int | None`、`list[str]` 等，旧教程中的 `Optional[int]`、`List[str]` 常涉及语法演进。`cast(...)` 主要让静态检查器改变对表达式的推断，**不是运行时的数据转换器**；需要实际校验外部 JSON 时，应单独设计验证逻辑或使用 Pydantic。

---

你可能已经看过：

```python
def load_names(path: str) -> list[str]:
    ...
```

也许能猜出它表示“传入路径，返回字符串列表”。

但继续看到 `T`、`Protocol`、`Callable`、`TypedDict`、`cast`，就容易觉得类型注解突然变成另一门语言。

其实它们大多在回答日常代码里的具体问题：这个值能不能是 None？函数会不会保留输入类型？字典有哪些键？我只需要一个能写文本的对象，为什么必须要求某个具体类？

本文从这些问题出发。

基础示例使用 Python 3.11 已支持的常用写法。

Python 3.12 的 type 语句与新泛型语法、Python 3.14 的注解求值变化，会在相应位置明确标注，不混在同一个兼容版本承诺里。

静态检查使用隔离环境中的 mypy，运行期例子使用 Python 本身。

两种检查分别说明，不把“Python 能执行”当作“静态类型已经检查通过”。

文中的 python 表示你选择的项目解释器；WSL 未激活虚拟环境、只有 python3 命令时，使用 python3 或虚拟环境解释器的绝对路径。

## 第一章：写了 int，为什么 Python 还允许传字符串？

### 1.1 一个很有解释力的小例子

```python
def repeat(value: int) -> int:
    return value * 2


print(repeat(3))
print(repeat("3"))
```

普通运行通常输出：

```text
6
33
```

第二次传入字符串，字符串乘二仍然合法，所以得到字符串重复。

冒号后的 int 没有自动在函数入口拦截它。

### 1.2 类型注解不是运行期强制转换

`value: int` 描述调用契约。

`-> int` 描述返回契约。

普通 Python 不会因为这些描述，就自动调用 int(value)，也不会每次都执行 isinstance。

你要做转换，仍然需要写转换代码。

你要做运行期拒绝，仍然需要明确判断，或使用 Pydantic 等验证工具。

### 1.3 那注解到底帮助谁？

帮助阅读代码的人理解接口。

帮助编辑器提供补全、定位和提示。

帮助静态检查器在运行前发现某些不符合契约的调用。

它不要求把 Python 变成另一种运行语言，而是在动态代码上提供额外可分析信息。

### 1.4 静态检查是什么时候发生？

你单独运行检查命令，或者编辑器在后台分析。

例如 mypy 读取源代码，检查调用类型与注解是否一致。

它不需要真正调用 repeat("3") 才能指出参数不合适。

但它也不能知道所有运行时事实，特别是外部 JSON、数据库和网络返回值。

静态分析与运行测试各有边界。

### 1.5 先记住这条主线

```text
类型注解：写下预期结构与关系。
静态检查器：分析代码是否符合这些描述。
运行期验证：检查实际收到的数据。
测试：执行具体场景并比较结果。
```

这四件事会配合，但不应互相冒充。

[Python 官方 typing 说明](https://docs.python.org/3.11/library/typing.html)也明确指出，普通运行时不会强制执行函数与变量注解。

## 第二章：类型提示从哪里来？为什么现代写法越来越简洁？

### 2.1 它不是最初 Python 的强制前提

Python 可以不写注解，仍然正常运行。

随着项目扩大，工具需要一种标准方式表达参数、返回值和复杂数据关系，于是相关注解约定与 typing 支持逐步发展。

typing 标准库模块在 Python 3.5 加入，后续版本继续完善语法和类型表达。

理解这段演进，可以解释为什么网上教程出现多种风格。

### 2.2 `List[str]` 与 `list[str]`

旧教程常写：

```python
from typing import List

names: List[str] = ["Alice", "Bob"]
```

现代 Python 3.9 及以上通常直接写：

```python
names: list[str] = ["Alice", "Bob"]
```

本文基线是 3.11，所以优先采用内置容器泛型写法。

不必为了普通 list 注解额外导入 List。

### 2.3 `Optional[str]` 与 `str | None`

旧写法：

```python
from typing import Optional

nickname: Optional[str] = None
```

现代写法：

```python
nickname: str | None = None
```

这里表达的类型选择大体相同。

竖线联合写法从 Python 3.10 起支持。

不是说旧代码立即无法运行，而是现代目标版本可以使用更直接的表达。

### 2.4 typing_extensions 是标准库吗？

不是，它是常见第三方兼容支持包。

某些新类型功能会在其中向旧 Python 提供回移版本。

但它不能让旧解释器突然理解所有新的语法。

例如 Python 3.11 解析不了 3.12 的 type 语句，你仅安装 typing_extensions 也不能改变解析器语法。

“提供一个名字”与“支持一份新语法”是不同层面。

### 2.5 学习时先确定目标 Python

项目如果支持 3.11，就优先使用能在 3.11 解析和导入的写法。

如果只支持 3.12，可以选择新的别名与泛型语法。

不要在一个文件里混写多个版本的示例，然后只在当前较新解释器运行一次，就承诺全部兼容旧版本。

后面会把新语法示例单独标注。

## 第三章：冒号、箭头和等号，分别在做什么？

### 3.1 变量注解

```python
count: int = 3
name: str = "Alice"
enabled: bool = True
```

冒号之后是类型描述。

等号之后是实际赋值。

两部分职责不同。

`count: int = "three"` 在普通 Python 中仍可能绑定字符串，但静态检查器应当指出与注解不符。

### 3.2 只写注解，不赋值呢？

```python
count: int
```

它不等于 `count = 0`。

在需要读取时，如果没有真正赋值，仍然可能遇到 NameError 等未绑定问题。

注解描述不自动替你创建某种默认对象。

如果你希望初始值为零，明确写 `count: int = 0`。

### 3.3 函数参数

```python
def greet(name: str) -> str:
    return f"你好，{name}"
```

`name: str` 描述参数。

`-> str` 描述返回值。

箭头不是在运行时执行一次字符串转换。

也不表示把函数对象永久限制成只能返回 str 的底层机器类型。

### 3.4 默认参数与注解怎样一起写？

```python
def repeat_text(text: str, times: int = 2) -> str:
    return text * times
```

类型在默认值前面。

`times: int = 2` 同时表达整数契约与缺失时使用二。

如果参数能传 None，需要在类型里明确描述，而不是仅因为给了默认值就推断“任何类型都可以”。

### 3.5 没有 return，应该写什么？

```python
def show_message(message: str) -> None:
    print(message)
```

没有显式返回其他值时，普通函数返回 None。

所以 `-> None` 表达“调用者不应期待一个其他结果对象”。

它不是说函数绝不产生输出或副作用。

上例仍然会打印。

### 3.6 构造方法也写 -> None 吗？

通常可以这样注解 __init__：

```python
class User:
    def __init__(self, name: str) -> None:
        self.name = name
```

__init__ 初始化已经创建的实例，不通过普通返回值把新实例交给调用者。

因此，不应把返回注解写成 User 只是因为外部调用 User("Alice") 得到 User 对象。

要区分类调用的整体行为与 __init__ 方法自身的返回契约。

## 第四章：list、dict、tuple、set，方括号描述什么？

### 4.1 list[str]

```python
names: list[str] = ["Alice", "Bob"]
```

表示列表中的元素预期是字符串。

不是列表只能有一个元素。

也不是运行时自动遍历检查每项。

方括号里的内容是元素类型参数。

### 4.2 dict[str, int]

```python
scores: dict[str, int] = {"Alice": 90, "Bob": 85}
```

第一个类型描述键，第二个描述值。

这并没有声明只允许 Alice 和 Bob 两个固定键。

如果需要固定字段名，后面会学习 TypedDict。

### 4.3 固定长度元组

```python
point: tuple[float, float] = (1.0, 2.0)
```

它描述两个位置，每个位置一个 float。

如果有三个坐标，要按真实结构改变类型。

不同位置也可以有不同类型：

```python
entry: tuple[str, int] = ("Alice", 90)
```

这与“任意数量的 str 或 int”不是一回事。

### 4.4 任意长度同类元组

```python
ids: tuple[int, ...] = (1, 2, 3)
```

这里省略号在类型表达中有专门含义，表示可变长度的同类元组。

不等于“代码还没写完”。

与函数体中的 `...` 占位用途，要按语境区分。

### 4.5 set[int]

```python
unique_ids: set[int] = {1, 2, 3}
```

它描述整数集合。

集合的去重、无固定索引等运行性质，仍然来自普通 set。

注解不会让集合保留输入重复次数或自动排序。

### 4.6 嵌套类型逐层读

```python
records: list[dict[str, int]] = [
    {"python": 90, "sql": 85},
    {"python": 80, "sql": 95},
]
```

从外向内读：一个列表，列表每项是字典，字典键是字符串、值是整数。

如果描述越来越深，阅读负担很高，可以考虑命名别名、TypedDict 或数据类。

不要把所有结构都塞进一行嵌套表达，追求最短字符数。

### 4.7 注解是不是容器创建语法？

`list[str]` 是类型表达的一种常见写法。

真正创建字符串列表仍然通常使用列表字面量或 list(...)。

不要把类型参数当成初始元素。

例如 `list[str]` 并不是自动生成了 `[""]`。

## 第五章：联合类型与 None，把两种“可选”分开

### 5.1 值可能属于不同类型

```python
def format_id(value: int | str) -> str:
    return str(value)
```

int | str 表示允许整数或字符串。

不是按位或计算两个普通数值，也不是执行时随机选择一个类型。

类型表达中的运算符含义由相应对象支持。

### 5.2 None 表示真实的一种值

```python
def find_name(user_id: int) -> str | None:
    if user_id == 1:
        return "Alice"
    return None
```

调用者必须处理“没有找到”的情况。

不能在没有判断时直接对结果调用字符串方法，并希望检查器总是假定它是 str。

### 5.3 可空不等于参数可以省略

```python
def show_nickname(nickname: str | None) -> None:
    print(nickname)
```

这个参数允许传 None，但调用时仍然需要提供。

如果希望省略，写默认值：

```python
def show_optional_nickname(nickname: str | None = None) -> None:
    print(nickname)
```

这与 Pydantic v2 中“Optional 不自动产生默认值”的区别很相通，但这里讨论的是普通 Python 参数规则。

### 5.4 有默认值，也不一定允许 None

```python
def connect(host: str = "localhost") -> str:
    return host
```

允许不提供 host。

但类型契约仍然要求显式提供时是 str。

`connect(None)` 不因为存在默认值就自动符合契约。

省略参数与明确 None，不是一件事。

### 5.5 为什么滥用联合类型会让调用者难用？

如果一个函数返回 `str | list[str] | dict[str, str] | None`，调用者每次都要判断许多形状。

有时这是实际协议需要。

但有时只是函数承担太多职责。

类型注解可以暴露设计复杂度，不一定要靠添加更多联合成员解决。

考虑统一返回模型、明确状态标签或拆分入口，可能更易用。

## 第六章：先真正运行一次静态检查

### 6.1 选择检查器，不靠眼睛猜

mypy 是常用的静态类型检查工具之一。

Pyright 等工具也有各自的规则、配置和诊断呈现。

本篇用 mypy 演练，不声称所有工具在每个高级边界上都给出完全相同结果。

在项目虚拟环境安装：

```bash
python -m pip install mypy
```

### 6.2 正确文件 good.py

```python
def double(value: int) -> int:
    return value * 2


answer: int = double(3)
print(answer)
```

运行程序：

```bash
python good.py
```

输出：

```text
6
```

静态检查：

```bash
python -m mypy --strict --python-version 3.11 good.py
```

通常显示一个源文件未发现问题。

这表示该检查配置下没有报告类型错误，不等于证明 double 的所有数学业务需求都正确。

### 6.3 错误文件 bad.py

```python
def double(value: int) -> int:
    return value * 2


answer: int = double("3")
```

普通 Python 仍然可以运行到赋值，把 "33" 绑定给 answer。

但静态检查应报告参数类型不合适。

```bash
python -m mypy --strict --python-version 3.11 bad.py
```

诊断会指出向需要 int 的参数传了 str，并以非零状态结束。

具体行号和文案可能随版本变化，稳定含义是参数契约冲突。

### 6.4 为什么要加 --strict？

它启用一组更严格的检查政策，有助于演练时发现未注解函数等问题。

实际旧项目不一定适合第一天全局开启最严格模式。

可以先检查稳定接口，逐步减少动态空白。

检查级别是工程选择，不是写了注解就只有唯一正确配置。

### 6.5 --python-version 代表什么？

告诉检查器目标 Python 版本，用于相应类型和语法规则分析。

它不是把当前运行的解释器降级为 3.11，也不替代真实旧版本运行检查。

尤其是新语法，目标版本与实际解释器支持需要分别确认。

### 6.6 检查通过后还需要测试吗？

需要。

一个函数声明返回 int，却把 left-right 写成 left+right，类型可能完全正确，数学行为却错。

静态类型擅长发现一部分结构与接口问题。

pytest 等测试执行具体场景，帮助检查结果、状态和错误路径。

可以结合 [pytest 教程](Python-pytest库.md) 把两类证据一起使用。

关于运行配置，可查[mypy 官方入门](https://mypy.readthedocs.io/en/stable/getting_started.html)。

## 第七章：类型推断，哪些地方不必重复写注解？

### 7.1 简单赋值通常能推断

```python
count = 3
name = "Alice"
```

检查器通常能推断出整数和字符串。

不需要每个局部变量都机械写一次相同信息。

注解更值得用于函数边界、复杂容器、允许多种类型的变量和公共数据结构。

### 7.2 空容器为什么有时需要帮助？

```python
names: list[str] = []
```

空列表没有元素可供推断。

后续使用有时能帮助工具推断，但明确注解能让意图更清楚，尤其是初始化后会跨多处分支写入的容器。

不要把空列表写成 list[Any] 只为了不让检查器追问。

### 7.3 初始 None 怎么办？

```python
selected: str | None = None
```

这样后续赋字符串与 None 都符合描述。

如果只让工具从初始 None 推断，某些后续赋值可能与你想象不同。

复杂变量生命周期中，显式联合类型通常更容易表达意图。

### 7.4 分支里可能没有赋值

类型描述不保证程序总会执行某个赋值分支。

如果只有条件成立才给 result 赋值，条件不成立就读取，仍然是逻辑问题。

可以提前初始化、明确返回或调整控制流。

不要用 cast 把未初始化问题伪装成类型已经确定。

### 7.5 reveal_type 是运行期函数吗？

类型检查器支持 reveal_type 等诊断辅助机制。

Python 3.11 的 typing 也提供相应入口，可以用于观察运行时类型信息，但静态诊断与运行打印的内容和目的不同。

学习时可以让检查器展示某表达式推断结果。

不要把临时诊断输出留在用户正常流程中，除非你确实在写调试工具。

## 第八章：类型缩小，为什么判断 None 后就能调用字符串方法？

### 8.1 从联合类型进入具体分支

```python
def display_name(name: str | None) -> str:
    if name is None:
        return "匿名用户"
    return name.upper()
```

函数入口时，name 可能是字符串或 None。

None 分支已经返回，所以最后一行面对的正常路径中，name 应该是字符串。

检查器可以分析这种控制流，把某个位置的类型范围缩小。

这叫类型缩小，不是运行时把 None 转成了字符串。

### 8.2 isinstance 也能帮助缩小

```python
def describe_value(value: int | str) -> str:
    if isinstance(value, str):
        return f"文本长度：{len(value)}"
    return f"整数的平方：{value * value}"
```

字符串分支可以使用字符串操作。

剩余分支按相应分析处理整数。

但 isinstance 的运行继承规则仍然存在，例如 bool 与 int 的关系，不能因为有注解就忽略。

### 8.3 真值判断会不会丢掉合法空值？

```python
def show_label(label: str | None) -> str:
    if label:
        return label
    return "默认标签"
```

空字符串也会走默认分支。

如果你只想处理 None，应写 `if label is None`。

类型分析可能认为代码类型合法，但业务语义仍由你决定。

这正是静态检查不能代替需求判断的具体例子。

### 8.4 assert 可以缩小，但不要代替外部校验

```python
def use_known_name(name: str | None) -> str:
    assert name is not None
    return name.upper()
```

检查器可以利用断言推断后续类型。

但普通 assert 在优化运行中可能被移除。

如果 None 来自不可信输入且必须拒绝，明确 raise 合适异常更稳妥。

测试中的 assert 用于检查结果，与业务必需校验的职责不同。

### 8.5 多线程修改会让某些假设失效

如果你先判断 `obj.value is not None`，随后别的代码修改 obj.value，之前的判断不一定保证稍后属性读取仍然相同。

把需要使用的值保存到局部变量，有时更容易表达稳定快照。

类型检查器的控制流推断不是自动并发锁。

共享可变状态仍需要真实同步与设计。

## 第九章：Any 与 object，看起来都“什么都能装”，为什么完全不同？

### 9.1 object 表示广泛对象，但不允许任意操作

```python
def describe_object(value: object) -> str:
    return str(value)
```

所有普通 Python 对象都可以作为 object 使用。

但你不能仅凭 object 注解，就静态确认它有 `.upper()` 或 `.append()`。

需要先做合适判断或使用更准确协议。

### 9.2 Any 更像关闭这一段检查约束

```python
from typing import Any


def dynamic_operation(value: Any) -> Any:
    return value.nonexistent_method()
```

检查器可能允许这段操作，因为 Any 告诉它缺少或放宽具体类型信息。

运行时如果对象没有这个方法，仍然会 AttributeError。

Any 不是“自动支持所有方法的超级对象”。

### 9.3 Any 为什么会传播？

一个来自动态来源的 Any，传给别的函数或访问属性后，可能让后续也失去明确检查。

因此，一个接口全部标 Any，通常不能提供你原本期待的类型保护。

它适合动态边界、兼容过渡与确实难以表达的少量区域，但最好让范围可见。

不要把所有红线都改成 Any，当作类型工作已经完成。

### 9.4 JSON 为什么经常带来 Any？

json.loads 可以得到字典、列表、字符串、数字、布尔或 None。

很多库的静态描述为了方便，会把解析结果表现为 Any。

这意味着检查器未必能替你确认键是否存在、值是不是所需类型。

你应在边界验证真实形状，再把可信结果交给内部函数。

### 9.5 不知道是什么，优先写 object 可以吗？

如果函数只存储、比较或返回未知对象，而不需要任意动态操作，object 往往比 Any 更能保持检查价值。

调用者可以传很多对象，但函数内部不能随意假定能力。

这迫使你在真正需要操作时说明理由。

Any 则允许更动态的互操作，两者用途不同。

### 9.6 `list[Any]` 与 Any 又有什么差别？

list[Any] 至少说明外层是列表，元素暂时不具体。

Any 则连外层能否索引、迭代等都可能放宽为动态。

同理，dict[str, Any] 仍然表达键为字符串的映射结构，但不保证固定键或具体值类型。

越靠近可靠内部边界，越值得逐步减少模糊范围。

## 第十章：cast，为什么它不能把字符串变成整数？

### 10.1 一个必须亲手运行的例子

```python
from typing import cast

value = cast(int, "123")
print(value)
print(type(value).__name__)
```

输出：

```text
123
str
```

cast 返回原对象，不执行 int 转换，也不验证实际类型。

它主要向静态工具表达“请按这个类型理解此处”。

### 10.2 真正转换怎么写？

```python
value = int("123")
print(type(value).__name__)
```

输出：

```text
int
```

这是普通 Python 转换函数。

可能成功，也可能对坏文本抛 ValueError。

cast 则没有替你获得这份运行保证。

### 10.3 什么情况下 cast 合理？

你已经通过外部机制确认某个事实，但检查器无法跟踪那份证明。

例如受控库边界返回较宽类型，你有具体、可复查的协议保证。

这时局部 cast 可以表达程序员掌握的信息。

但最好靠近证据位置，并解释为什么可信。

### 10.4 不要用 cast 修复真实错误

如果值其实可能是 None，你 cast 成 str，后面 `.upper()` 仍然可能崩溃。

如果外部 JSON 根本没验证，cast 成复杂 TypedDict 也只是改变工具看法。

这种做法可能让静态报告变绿，却减少程序真实保护。

优先做运行判断、解析或验证；只有检查器表达能力确实不足时再使用有依据的断言。

### 10.5 `# type: ignore` 也有类似责任

忽略注释可以暂时压制诊断。

应尽量指定错误类别，写清原因，避免全行或全文件广泛掩盖新问题。

如果一段代码需要很多 ignore，先检查注解是否错误、依赖类型资料是否缺失，或者设计是否真的不适合当前表达。

忽略是受控逃生口，不是默认开发方式。

## 第十一章：类型别名，让复杂结构有一个能读懂的名字

### 11.1 先看一条太长的签名

```python
def summarize(records: list[dict[str, int | str | None]]) -> dict[str, int]:
    return {"count": len(records)}
```

它能写，但每次阅读都需要重新拆外层和内层。

如果结构会重复使用，可以命名。

### 11.2 Python 3.11 的明确别名写法

```python
from typing import TypeAlias

CellValue: TypeAlias = int | str | None
Record: TypeAlias = dict[str, CellValue]
```

然后使用：

```python
def count_records(records: list[Record]) -> int:
    return len(records)
```

TypeAlias 表达这里在声明类型别名，而不是普通业务变量。

别名主要改善表达，不自动增加运行期验证。

### 11.3 别名会创建一种完全不同的运行类型吗？

普通别名通常不会。

如果 UserId: TypeAlias = int，UserId 与 int 的类型含义仍然紧密相同。

它不会自动防止把订单编号当成用户编号。

需要静态区分这种语义，可以了解 NewType，后面会单独讲。

### 11.4 Python 3.12 的 type 语句

下面代码需要 Python 3.12 及以上：

```python
# Python 3.12+
type CellValue312 = int | str | None
type Record312 = dict[str, CellValue312]
```

这是新的类型别名语法，会创建相应类型别名对象，并有自己的求值语义。

不是把之前的普通赋值随便加一个单词。

3.11 解释器连解析都不会通过，所以不能靠 if sys.version_info 把它包在同一个普通文件里就期待旧解释器略过语法错误。

### 11.5 为什么版本判断不能躲过语法解析？

Python 在执行 if 分支之前，需要先解析整个文件。

文件里有当前解释器不认识的语法，根本走不到条件判断。

要支持多个版本，应使用共同支持的语法，或把新语法隔离到不被旧解释器解析的模块与构建方式中。

这一点不仅适用于 type 语句，也适用于其他语言语法升级。

[Python 3.12 的 type 语句说明](https://docs.python.org/3.12/reference/simple_stmts.html#the-type-statement)解释了这个入口。

## 第十二章：NewType，用户编号和订单编号都是 int，怎么区分？

### 12.1 普通 int 丢失了语义差别

```python
def load_user(user_id: int) -> str:
    return f"user:{user_id}"
```

订单编号也是 int，检查器未必知道你把编号传错了。

如果这种错误实际常见，可以在静态层加入轻量语义区分。

### 12.2 定义两个不同名称

```python
from typing import NewType

UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)


def user_key(user_id: UserId) -> str:
    return f"user:{user_id}"


alice = UserId(1)
print(user_key(alice))
print(type(alice).__name__)
```

输出：

```text
user:1
int
```

运行期仍然是原来的整数对象表现。

静态工具则可以区分 UserId 与 OrderId 的使用关系。

### 12.3 会验证编号为正吗？

不会仅因为 NewType 名叫 UserId 就验证范围。

UserId(-1) 不自动抛出范围错误。

需要真实入口规则，另外执行检查或模型验证。

NewType 解决语义类型区分，不替代数据合法性规则。

### 12.4 为什么不总创建一个包装类？

真正包装类可以承载运行期行为，但需要对象结构与方法设计。

NewType 适合主要需要静态区分、又希望保留轻量运行表现的场景。

是否值得引入，取决于编号混淆造成的实际风险和代码规模。

不必给每个临时整数都发明一个新语义类型。

## 第十三章：Literal 与 Enum，有限选项怎样表达？

### 13.1 参数只允许两个模式

```python
from typing import Literal


def render_mode(mode: Literal["compact", "pretty"]) -> str:
    return mode
```

检查器可以发现 `render_mode("prety")` 这样的拼写问题。

Literal 描述一组具体允许值，而不是任意 str。

### 13.2 运行时会自动拒绝拼错吗？

不会。

普通函数的注解不自动执行值检查。

如果调用来自用户输入，需要明确运行验证。

这一点尤其容易因为 Pydantic 也支持 Literal 而混淆：Pydantic 会解释注解执行验证，普通函数不会。

同一份类型描述，被不同工具使用时，行为层面不同。

### 13.3 枚举提供真实运行对象

```python
from enum import Enum


class DisplayMode(str, Enum):
    COMPACT = "compact"
    PRETTY = "pretty"


def mode_label(mode: DisplayMode) -> str:
    return mode.value
```

枚举成员是真实运行对象。

你可以通过 DisplayMode("compact") 按枚举机制解析值，非法值会触发相应错误。

但函数参数注解本身仍不会自动帮你调用枚举构造器。

### 13.4 Literal 值与变量推断

一个变量写成 `mode = "compact"`，某些检查上下文可能把它推断为一般 str，而不是始终保持 Literal["compact"]。

如果你需要窄类型，可以明确注解或使用合适 Final 声明。

不要因为值看起来相同，就假定工具在所有上下文中拥有同样精确的信息。

需要时用 reveal_type 观察，而不是乱加 cast。

### 13.5 什么时候选哪一个？

少量字符串协议、希望调用简洁时，Literal 很合适。

需要真实枚举身份、集中成员定义、方法或运行期构造时，Enum 更有意义。

它们可以配合，但不是所有配置开关都必须同时用两套系统。

先看调用者怎样传值、外部数据怎样验证。

## 第十四章：TypedDict，字典有固定字段时怎样表达？

### 14.1 dict[str, object] 太宽了

一个用户记录固定有 id 和 name。

普通 dict[str, object] 只能表达键是字符串、值是广泛对象，无法清楚说明每个键的具体值类型。

TypedDict 用于描述这种固定形状字典。

### 14.2 最小示例

```python
from typing import TypedDict


class UserRecord(TypedDict):
    id: int
    name: str


def user_label(user: UserRecord) -> str:
    return f"{user['id']}:{user['name']}"


record: UserRecord = {"id": 1, "name": "Alice"}
print(user_label(record))
print(type(record).__name__)
```

输出：

```text
1:Alice
dict
```

运行时仍然是普通字典。

没有自动变成可以通过 record.name 访问的模型对象。

### 14.3 必需键与可选键

Python 3.11 可以使用 Required、NotRequired：

```python
from typing import NotRequired, TypedDict


class UserSummary(TypedDict):
    id: int
    name: str
    nickname: NotRequired[str | None]
```

nickname 可以缺失。

如果存在，允许字符串或 None。

这两个维度又一次出现：键是否存在，以及存在时值的类型。

### 14.4 `str | None` 不等于 NotRequired

如果你只写 `nickname: str | None`，默认 total=True 的 TypedDict 中，这个键仍然是要求存在的。

允许值为 None，与允许键缺失不同。

这与函数默认参数、Pydantic 字段的区别相呼应，但各工具有各自的语法入口。

理解维度，比死记某个 Optional 单词更可靠。

### 14.5 total=False

可以让模型中相应声明的键默认都不是必需：

```python
from typing import Required, TypedDict


class PartialUser(TypedDict, total=False):
    id: Required[int]
    nickname: str | None
```

这里 id 通过 Required 明确要求存在。

不要以为 total=False 意味着字典不能有任何键，或者所有值都自动变成 None。

它改变的是键存在性要求。

### 14.6 运行时可以 isinstance(record, UserRecord) 吗？

不能把 TypedDict 当成普通运行期结构验证类来依赖。

它主要服务静态类型描述。

想检查真实数据，需要自己判断、使用验证库，或借助 TypeAdapter 等支持工具。

不能只调用 `cast(UserRecord, raw_json)` 就宣称外部字典验证完成。

### 14.7 读取可选键要小心 KeyError

NotRequired 不表示 `record["nickname"]` 在运行时永远安全。

如果键缺失，普通 dict 索引仍然会 KeyError。

可以先做成员判断，或使用 get 并处理默认含义。

检查器对某些 TypedDict 访问策略的严格程度可能不同，运行期字典规则始终要理解。

### 14.8 TypedDict、dataclass 和 Pydantic 怎么区分？

TypedDict 保留字典形状，主要提供静态资料。

dataclass 帮助创建类式数据对象及相应生成方法，普通版本不自动验证全部注解。

Pydantic 解释类型描述，对实际输入执行转换与验证，并提供序列化等能力。

它们不是复杂程度排行榜，而是解决不同层面的问题。

## 第十五章：Iterable、Sequence、Mapping，为什么参数不一定写 list 或 dict？

### 15.1 我只想逐个读取元素

```python
from collections.abc import Iterable


def total_length(values: Iterable[str]) -> int:
    return sum(len(value) for value in values)
```

函数只需要能够迭代字符串。

列表、元组和某些生成器都能满足。

如果参数写 list[str]，可能无必要地排除其他可迭代输入。

注解应该描述真实需要的能力。

### 15.2 Iterable 不保证可以索引

你不能因为能 for 遍历，就直接假定 `values[0]` 有效。

生成器通常没有普通列表式索引。

如果函数需要长度与索引，可以考虑 Sequence。

类型越抽象，允许输入范围通常越广，但可直接依赖的能力也相应更有限。

### 15.3 Sequence[str]

```python
from collections.abc import Sequence


def first_or_none(values: Sequence[str]) -> str | None:
    if not values:
        return None
    return values[0]
```

这个函数需要判断长度与读取首项。

Sequence 比任意 Iterable 更贴近需求。

同时，它不要求调用者一定给一个可修改的 list。

### 15.4 Sequence 里的 str 陷阱

字符串本身也是字符序列。

某个参数写 Sequence[str]，不一定能静态排除把单个字符串当成一串字符串传入的情况。

如果业务必须接收“多个完整标签”，要考虑实际输入验证或更明确接口。

静态抽象能力匹配，不会自动理解你的自然语言“这是一个标签列表”的全部含义。

### 15.5 Mapping 与 MutableMapping

```python
from collections.abc import Mapping


def get_host(options: Mapping[str, str]) -> str:
    return options.get("host", "localhost")
```

这里只读取映射，不需要改它。

Mapping 表达读取能力。

如果确实要插入或删除键，则应考虑 MutableMapping 或具体可变类型。

不要在参数注解写只读接口，函数里却偷偷修改输入。

### 15.6 输入宽、输出准

一个常见原则是：输入按实际需要的最小能力描述，输出按你确实返回的具体保证描述。

例如接收 Iterable[str]，返回新 list[str]。

调用者知道可以传生成器，也知道结果可以索引与修改。

这比输入输出一律标 Iterable 更易用，但也不是要求每个函数都遵守唯一模板。

### 15.7 注解不会把可变对象变成只读

Mapping 或 Sequence 是静态接口约定。

实际对象如果是 dict 或 list，外部其他引用仍然可能修改它。

你只是说当前函数不应依赖修改能力，并没有在运行时冻结共享对象。

需要不可变快照时，真实复制或使用不可变结构仍然重要。

## 第十六章：TypeVar，怎样表达“返回类型跟着输入走”？

### 16.1 使用 object 会丢掉关系

```python
def identity_object(value: object) -> object:
    return value
```

它接受广泛对象，也返回广泛对象。

调用者传入 str 后，检查器仅从签名未必保留“结果仍是同一个 str 类型”的精确信息。

我们需要描述输入与输出之间的联系。

### 16.2 定义类型变量

```python
from typing import TypeVar

T = TypeVar("T")


def identity(value: T) -> T:
    return value
```

T 不是每次运行需要用户传入的普通业务参数。

它是类型关系里的变量。

对一次调用，检查器根据输入推断相应类型，再把关系应用到返回值。

### 16.3 具体调用怎样理解？

```python
name = identity("Alice")
count = identity(3)
```

第一处通常保留字符串信息。

第二处通常保留整数信息。

函数实现没有靠 if 判断 str 或 int，类型关系却比 object→object 更精确。

这正是泛型函数的一项常见用途。

### 16.4 一个更实际的例子：取非空列表首项

```python
from collections.abc import Sequence


def first(values: Sequence[T]) -> T:
    if not values:
        raise ValueError("序列不能为空")
    return values[0]
```

字符串列表返回字符串。

用户对象列表返回用户对象。

空序列仍然需要真实运行检查，因为类型参数并未自动证明长度大于零。

### 16.5 为什么不能随便 return 0？

如果签名承诺返回 T，而调用者可能传字符串序列，你却在某个分支返回整数零，就违反了关系。

检查器应指出问题。

泛型并不是“什么都可以”的另一个 Any。

它要求实现对所有允许的 T 保持相应关系。

### 16.6 T 的名字有魔法吗？

名字可以换，但约定常用 T、K、V 等表示类型参数。

TypeVar("T") 里的字符串名称通常与赋值名称匹配，便于工具识别与读者理解。

实际业务中也可以使用更有意义的名称，但不要把类型变量和普通变量混用。

### 16.7 什么时候不需要 TypeVar？

如果输入类型与输出类型没有相关约束，只是接受任意对象并返回字符串，object→str 更直接。

如果函数只支持明确的一种类型，直接写那种类型即可。

类型变量有价值的地方是关系，而不是让签名看起来更高级。

## 第十七章：bound 与有限约束，类型变量究竟能做什么？

### 17.1 为什么实现里不能对任意 T 调用 upper？

T 可能是整数、路径或其他对象。

没有更多约束，检查器不能确认它拥有字符串方法。

如果函数需要某种共同能力，要给类型变量边界，或使用协议。

### 17.2 bound 表达上界

```python
from typing import TypeVar

TextT = TypeVar("TextT", bound=str)


def preserve_text(value: TextT) -> TextT:
    return value
```

TextT 限制在 str 及相应子类型范围内。

它可以保留具体子类型关系。

但是你调用 value.upper() 的结果通常只保证是 str，不一定仍然是原子类型 TextT。

因此，返回 value.upper() 却标 TextT，可能是不成立的承诺。

### 17.3 有限约束表达几种允许族

```python
from typing import TypeVar

TextOrBytes = TypeVar("TextOrBytes", str, bytes)


def join_same_kind(left: TextOrBytes, right: TextOrBytes) -> TextOrBytes:
    return left + right
```

它表达左右两边需要保持同类文本族。

不能把 str 与 bytes 混在一起，指望通过联合类型自动解决拼接。

检查器会按受约束类型变量规则推断，而不是把每个参数独立理解成完全无关的 str|bytes。

### 17.4 bound 与 constraints 是同义词吗？

不是。

上界允许相应子类型并可保留更具体关系。

有限约束表达若干候选类型族，推断可能提升到候选中的相应类型，而不是完整保留任意子类。

初学时先围绕真实需求选择，不要把两个参数都当作“缩小范围”就随意交换。

### 17.5 多个参数共享 T，也不总能表达业务“完全同一个类型”

检查器有自己的推断与共同上界规则。

如果你要求精确运行类型完全一致，静态同一个 T 不一定足够表达所有边界。

必要时在运行期检查 type，或用有限约束、重载及更明确接口设计。

类型表达的实际能力要通过检查器样例确认，不能仅从自然语言解释推断绝对保证。

## 第十八章：Generic，怎样让自己的容器也写成 Box[int]？

### 18.1 普通容器类会丢掉元素信息

假设一个盒子存放一个值。

如果全部标成 object，取出后调用者需要再次判断具体类型。

泛型类让类实例也携带静态元素关系。

### 18.2 Python 3.11 的写法

```python
from typing import Generic, TypeVar

ItemT = TypeVar("ItemT")


class Box(Generic[ItemT]):
    def __init__(self, value: ItemT) -> None:
        self.value = value

    def get(self) -> ItemT:
        return self.value
```

使用：

```python
number_box = Box(3)
text_box = Box("Alice")
print(number_box.get())
print(text_box.get())
```

输出：

```text
3
Alice
```

检查器可以分别推断盒子内容类型。

### 18.3 Box[int] 会运行时检查赋值吗？

普通泛型类不会自动插入字段验证。

如果你在动态代码里把错误类型赋给 value，仍然可能发生。

Generic 提供静态参数化关系，不是 BaseModel 的替代品。

不要把“可以写方括号”误认为“创建时自动检查所有内容”。

### 18.4 方法也可以维护类型关系

```python
class MutableBox(Generic[ItemT]):
    def __init__(self, value: ItemT) -> None:
        self._value = value

    def get(self) -> ItemT:
        return self._value

    def set(self, value: ItemT) -> None:
        self._value = value
```

如果一个盒子已经被看作 MutableBox[int]，set 传字符串应当被检查器拒绝。

运行时仍然只是普通方法调用，是否校验由实现决定。

### 18.5 Python 3.12 的新泛型语法

下面单独要求 Python 3.12 及以上：

```python
# Python 3.12+
class Box312[T]:
    def __init__(self, value: T) -> None:
        self.value = value

    def get(self) -> T:
        return self.value


def identity312[T](value: T) -> T:
    return value
```

它减少了一些显式 TypeVar 与 Generic 的样板。

但不是说 3.11 的写法错误，或者两套语法可以在所有细节上随便混用。

项目目标版本与检查工具支持要一起确认。

### 18.6 类型参数不是普通类级共享字段

不要把泛型 T 当作每个实例自动存储的一份真实类型对象。

运行时如果确实需要构造某个类型，通常要明确传入类对象或工厂。

例如参数 `factory: Callable[[], T]`，再真正调用 factory()。

仅凭一个类型变量名称，不应猜测可以随时调用 T() 创建对象。

## 第十九章：可变容器为什么通常不能随便协变？

### 19.1 一个听起来合理、实际危险的想法

Dog 是 Animal 的子类。

那 list[Dog] 是不是可以传给要求 list[Animal] 的函数？

如果函数会往列表添加 Cat，原来的狗列表就混入猫了。

调用者还以为列表里全是 Dog，后面可能调用狗特有行为。

### 19.2 类型系统需要防止这种写入漏洞

普通可变 list 的类型参数通常按不变关系处理。

不能只根据元素继承关系，就认为两个可变容器也拥有同样的子类型关系。

这不是检查器故意挑刺，而是写入能力带来的真实风险。

### 19.3 只读接口可以更宽松

如果函数只读取 Animal，不往里添加其他动物，可以接收 Sequence[Animal]。

这样狗序列可以在只读能力范围内使用。

```python
from collections.abc import Sequence


class Animal:
    def speak(self) -> str:
        return "sound"


class Dog(Animal):
    def speak(self) -> str:
        return "woof"


def sounds(animals: Sequence[Animal]) -> list[str]:
    return [animal.speak() for animal in animals]
```

这再次说明，参数选择能力接口，不只是为了好看，还影响类型兼容关系。

### 19.4 自定义协变参数什么时候合适？

当类型参数主要用于输出、对象不接受会破坏安全的相应写入时，可以研究 covariant=True。

如果同一个类型参数又用于任意 set 方法输入，通常不能随意声明协变。

先理解读写方向，再选择方差。

不要为了让一条调用通过就改变类型变量方差，导致类整体契约不成立。

### 19.5 逆变为什么常与函数参数有关？

如果某个回调能处理任何 Animal，它当然也能处理 Dog。

反过来，只会处理 Dog 的回调，不适合被当作“任何动物都能处理”。

这体现了输入方向与输出方向的关系不同。

初学时不必一开始背全部方差术语，但遇到 Callable、Protocol 和泛型兼容错误时，记住“它是在读出还是接收写入”很有帮助。

## 第二十章：Protocol，我只需要一个会写文本的对象

### 20.1 为什么不直接要求具体文件类？

一个输出函数只调用 writer.write(text)。

真实文件、StringIO、你自己写的收集器都可能支持。

如果注解死绑某个具体类，就会排除这些合理对象。

Protocol 用于描述“需要哪些成员与签名”，而不是强制每个对象继承同一个业务父类。

### 20.2 定义文本写入协议

```python
from typing import Protocol


class TextWriter(Protocol):
    def write(self, text: str) -> int:
        ...


def emit_title(writer: TextWriter, title: str) -> None:
    writer.write(title + "\n")
```

协议方法中的省略号表示这里主要提供接口描述。

不是让 emit_title 运行时自动生成写入实现。

真正执行的仍然是传入对象的 write 方法。

### 20.3 不继承也可以满足

```python
class MemoryWriter:
    def __init__(self) -> None:
        self.parts: list[str] = []

    def write(self, text: str) -> int:
        self.parts.append(text)
        return len(text)


writer = MemoryWriter()
emit_title(writer, "Python笔记")
print(writer.parts)
```

输出：

```text
['Python笔记\n']
```

MemoryWriter 没有显式继承 TextWriter。

只要静态结构符合协议，检查器就可以接受。

这常被称为结构化子类型，贴近 Python 的鸭子类型使用习惯。

### 20.4 参数名称也可能影响兼容

如果协议方法允许关键字调用，参数名称可能成为签名契约的一部分。

一个实现把参数改名，可能在关键字调用下不兼容。

如果你只要求位置参数，可以在协议里使用位置专用参数 `/` 表达。

不要觉得“都接收一个字符串”就完全忽略关键字规则。

### 20.5 属性协议怎么描述？

可以在协议里声明属性或 property。

如果是只读需要，property 往往比可写普通属性更能表达意图，并影响兼容关系。

属性类型、是否可写和成员行为，都属于接口。

不要仅列出一个名称，就以为已经描述了对象全部业务含义。

### 20.6 runtime_checkable 会完整验证方法签名吗？

不会。

加上 runtime_checkable 后，可以在相应限制下对协议执行运行期 isinstance 等检查。

但它主要检查成员存在等结构信息，不等价于静态工具对完整参数与返回签名的分析。

更不会调用每个方法验证真实行为。

一个对象有 write 属性，不代表它一定能正确写入字符串并返回合理数量。

### 20.7 协议与抽象基类如何选择？

如果希望对象显式继承、共享实现或有运行期抽象实例化限制，ABC 等机制可能更适合。

如果只需要静态表达能力，允许外部类无需修改就参与，Protocol 很自然。

两者可以合作，不必为了“高级”把所有父类都改成协议。

官方[Protocol 说明](https://docs.python.org/3.11/library/typing.html#typing.Protocol)及[mypy 协议文档](https://mypy.readthedocs.io/en/stable/protocols.html)提供了更多兼容细节。

## 第二十一章：Callable，参数是一个函数时怎样注解？

### 21.1 一个真正会用到的回调

你希望转换一批字符串，但转换方式由调用者提供。

```python
from collections.abc import Callable, Iterable


def transform_texts(
    values: Iterable[str],
    transform: Callable[[str], str],
) -> list[str]:
    return [transform(value) for value in values]


print(transform_texts(["alice", "bob"], str.upper))
```

输出：

```text
['ALICE', 'BOB']
```

Callable[[str], str] 表示可调用对象接收一个字符串，返回字符串。

外层第一个方括号描述参数列表，后面描述返回类型。

### 21.2 传函数与调用函数不要混淆

调用 transform_texts 时传 `str.upper`，是传可调用对象。

如果先写 `str.upper("alice")`，你传的是已经得到的字符串结果。

类型提示能帮助发现这种把值当函数的错误，但首先仍要理解普通 Python 调用语法。

### 21.3 多个参数怎么写？

```python
from collections.abc import Callable

BinaryOperation = Callable[[int, int], int]


def apply_numbers(operation: BinaryOperation, left: int, right: int) -> int:
    return operation(left, right)
```

参数顺序属于契约。

Callable 这里没有表达参数名称、默认值和复杂关键字规则的全部细节。

如果需要更精确回调签名，可以使用带 __call__ 的 Protocol。

### 21.4 Callable[..., str] 是什么意思？

省略号表示参数部分没有用这份表达具体限制。

它不是“任意参数组合都肯定能在运行期成功调用”。

返回值描述仍然是 str，但输入检查能力被放宽。

如果你明明知道参数结构，却全部写省略号，就可能漏掉调用错误。

### 21.5 可调用对象不一定是 def 函数

实现 __call__ 的实例、某些类和内置对象也可能可调用。

Callable 描述的是调用能力，而不只是函数声明语法。

这与 Protocol 的结构化思路很接近。

你不必为了给回调传一个带状态对象，就强行把它拆成全局函数。

## 第二十二章：overload，返回类型为什么可以跟参数形状变化？

### 22.1 一个容易写得太宽的函数

函数传整数索引时，返回单个名称。

传切片时，返回名称列表。

如果只写 `int | slice -> str | list[str]`，调用者即使传了明确整数，也可能只得到宽联合信息。

重载可以描述不同调用形状与返回类型之间的关系。

### 22.2 Python 的 typing.overload 写法

```python
from typing import overload


@overload
def select_name(values: list[str], index: int) -> str:
    ...


@overload
def select_name(values: list[str], index: slice) -> list[str]:
    ...


def select_name(values: list[str], index: int | slice) -> str | list[str]:
    return values[index]
```

前面几份签名主要给静态工具使用。

最后一份是实际运行实现。

不要把它理解成普通 Python 根据签名自动挑一个 def 函数执行。

### 22.3 实际调用

```python
names = ["Alice", "Bob", "Carol"]
print(select_name(names, 1))
print(select_name(names, slice(0, 2)))
```

输出：

```text
Bob
['Alice', 'Bob']
```

运行实现仍然依靠普通 list 索引行为处理。

静态检查器则根据 overload 签名，为两次调用提供更精确结果。

### 22.4 Literal 可以与重载配合

例如 `as_bytes=True` 返回 bytes，False 返回 str。

可以用 Literal[True] 与 Literal[False] 描述两个分支。

但如果调用者传入的是一个一般 bool 变量，重载集合还需要考虑这个宽输入是否有合适签名。

不要只为字面量示例写两份漂亮签名，却漏掉真实变量调用。

### 22.5 实现必须覆盖所有声明

如果重载声明接受 slice，但实现只处理 int，就形成虚假契约。

检查器会检查部分实现兼容性，但真实分支行为仍然要测试。

尤其是返回值类型虽然一样、业务结果却错时，静态分析不一定发现。

### 22.6 什么情况下不必 overload？

如果输入变化并不需要改变返回类型，普通联合参数足够。

如果返回类型始终相同，不必为了多个值重复一组签名。

如果关系可以用 TypeVar 简洁表达，泛型可能更适合。

重载主要用于多个离散调用形状，不是函数参数多就必须使用。

## 第二十三章：ParamSpec，装饰器怎样保留原函数的参数？

### 23.1 最宽松写法丢掉了什么？

你写一个记录调用的装饰器。

如果注解是 Callable[..., Any]→Callable[..., Any]，检查器很难在装饰后继续准确约束原来的参数和返回值。

运行行为虽然可以正确，但类型保护被装饰器打断。

ParamSpec 用于描述并转发一整个参数规范。

### 23.2 先看完整的 3.11 写法

```python
from collections.abc import Callable
from functools import wraps
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")


def traced(function: Callable[P, R]) -> Callable[P, R]:
    @wraps(function)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"调用：{function.__name__}")
        return function(*args, **kwargs)
    return wrapper
```

P 表示参数结构。

R 表示返回类型。

wrapper 收到与原函数对应的参数，再原样转发。

### 23.3 为什么不是 *args: P？

`P.args` 描述位置参数收集部分。

`P.kwargs` 描述关键字参数收集部分。

它们作为 ParamSpec 的配套表达使用，不是普通 list 和 dict 的类型别名。

写法的目的，是把原函数的参数关系保留下来，而不是粗略声明“这里有一堆对象”。

### 23.4 装饰一个有明确签名的函数

```python
@traced
def greet_user(name: str, times: int = 1) -> str:
    return " ".join([f"你好，{name}"] * times)


print(greet_user("Alice", times=2))
```

输出：

```text
调用：greet_user
你好，Alice 你好，Alice
```

静态检查器仍然应当知道 name 是 str、times 是 int、结果是 str。

错误传 `times="2"` 应报告问题，而不是因为经过装饰器就全部放行。

### 23.5 wraps 负责什么？

functools.wraps 帮助保留名称、文档和被包装函数关联等元信息。

ParamSpec 负责静态参数关系描述。

两个工具经常配合，但不能互相替代。

只写 wraps 不意味着检查器一定拥有你希望的完整泛型签名。

### 23.6 Concatenate 是什么用途？

当装饰器给原函数增加或消耗前面的某个参数，可以使用 Concatenate 表达这种参数变化。

例如某个包装器自动提供上下文对象，调用者只传剩余参数。

但它对参数形状有具体限制，不是任意修改关键字列表的通用工具。

没有真实需求时，先掌握保持原签名的 ParamSpec 模式，不必为每个装饰器引入额外上下文机制。

### 23.7 同步与异步装饰器不能只复制签名

普通 wrapper 如果直接返回协程，没有 await，运行时行为与真正异步包装可能不同。

需要根据原函数类型和资源生命周期设计。

注解可以描述 Awaitable[R] 等结果，但它不会自动把同步 wrapper 变成 async def。

装饰器的真实控制流仍然需要结合已有装饰器与异步教材理解。

## 第二十四章：Self、type[T] 和类方法，返回“当前类”怎样表达？

### 24.1 链式方法返回自身

```python
from typing import Self


class Builder:
    def __init__(self) -> None:
        self.parts: list[str] = []

    def add(self, text: str) -> Self:
        self.parts.append(text)
        return self
```

Self 从 Python 3.11 的 typing 提供。

当子类继承这个方法时，结果能更好保留相应子类信息。

比把所有返回固定写成 Builder 更贴近“返回调用对象自身”的关系。

### 24.2 Self 不会帮你创建正确对象

如果你承诺返回 Self，却在方法里永远创建基类 Builder，子类调用时可能不符合承诺。

类型关系仍然需要实现维护。

返回 self 是常见自然情况，但不是看见链式调用就可以随便标 Self。

### 24.3 类对象与实例对象不同

```python
class User:
    def __init__(self, name: str) -> None:
        self.name = name


def user_class_name(user_type: type[User]) -> str:
    return user_type.__name__
```

type[User] 表示类对象的相关类型描述。

User 注解通常表示 User 实例。

传 User 与传 User("Alice") 是不同对象层次。

### 24.4 工厂保留子类类型

```python
from typing import TypeVar

UserT = TypeVar("UserT", bound=User)


def create_user(user_type: type[UserT], name: str) -> UserT:
    return user_type(name)
```

输入一个符合构造约定的 User 子类，返回对应实例类型。

但实际子类如果改变构造参数，运行时仍可能出问题。

类型检查器对构造契约的处理有具体规则，不应把 type[T] 视为“所有子类任何构造都自动兼容”的证明。

如果需要明确工厂签名，直接接收 Callable[[str], UserT] 往往更贴近所需能力。

### 24.5 classmethod 返回 Self

类方法常用于替代构造。

```python
from typing import Self


class NamedItem:
    def __init__(self, name: str) -> None:
        self.name = name

    @classmethod
    def from_text(cls, text: str) -> Self:
        return cls(text.strip())
```

cls 是实际参与调用的类。

返回 cls(...) 比硬编码 NamedItem(...) 更容易保留子类构造意图。

但仍要保证子类构造规则适合这份协议。

## 第二十五章：TypeGuard 与穷尽检查，把判断结果告诉工具

### 25.1 普通 bool 辅助函数未必传递类型信息

你写一个函数检查对象是否是字符串列表。

如果只标返回 bool，检查器不一定能自动理解 True 分支意味着参数已经成为 list[str]。

TypeGuard 可以表达这种自定义缩小承诺。

### 25.2 一份必须真的检查元素的判断

```python
from typing import TypeGuard


def is_string_list(value: object) -> TypeGuard[list[str]]:
    return isinstance(value, list) and all(isinstance(item, str) for item in value)


def join_if_strings(value: object) -> str:
    if is_string_list(value):
        return ",".join(value)
    return "不是字符串列表"
```

True 分支中，检查器可以按 list[str] 理解 value。

但这份承诺由你实现。

如果判断函数永远 return True，运行时并不会自动补上元素检查。

### 25.3 空列表算字符串列表吗？

上例 all 对空迭代返回 True，因此空列表通过。

这通常符合“所有已有元素都是字符串”的逻辑。

如果业务要求至少一项，要额外检查长度。

类型缩小本身不自动携带非空长度保证。

### 25.4 可变别名仍可能破坏假设

你检查列表全部是 str 后，另一个引用可能 append 整数。

如果两者并发或在回调中交错，先前检查并不等于永久冻结。

运行期共享可变对象的风险，不会因为 TypeGuard 就消失。

需要快照时，复制与不可变结构仍然有价值。

### 25.5 TypeIs 是另一种新工具吗？

TypeIs 在 Python 3.13 的 typing 中提供，旧版本可按支持情况使用 typing_extensions。

它与 TypeGuard 在缩小规则、否定分支和子类型限制等方面不同，不能只换名字就假定语义完全相同。

本文 3.11 基础示例使用 TypeGuard。

需要 TypeIs 时，应核对目标解释器、扩展包和检查器版本，并针对正反分支做小实验。

### 25.6 Literal 状态的穷尽检查

```python
from typing import Literal, assert_never

Status = Literal["queued", "done"]


def status_label(status: Status) -> str:
    if status == "queued":
        return "等待中"
    if status == "done":
        return "已完成"
    assert_never(status)
```

当所有允许状态都已处理，最后分支理论上不可到达。

以后你给 Status 增加 "failed"，却忘记处理，检查器可以利用 assert_never 提醒遗漏。

### 25.7 Never 与 NoReturn

一个函数总是抛异常、退出或不返回，可以使用 NoReturn 等表达不正常返回的契约。

Never 还用于表示没有可能值的类型位置。

它们与 `-> None` 不同。

None 表示正常返回 None；不返回表示控制流不会正常把结果交回调用者。

如果函数标了不返回却真的执行到普通 return，就是错误承诺。

## 第二十六章：迭代器、生成器与异步返回，注解写的是哪一层？

### 26.1 普通生成器函数调用后得到什么？

```python
from collections.abc import Iterator


def positive_ids(limit: int) -> Iterator[int]:
    for value in range(1, limit + 1):
        yield value
```

函数体 yield 的元素是 int。

函数调用返回迭代器对象，不是一个已经生成完的 int。

所以整体返回注解通常写 Iterator[int]。

### 26.2 Iterable 与 Iterator 怎样选？

Iterator 更明确包含逐步 next 消费、通常一次性状态等迭代器能力。

Iterable 更宽，只承诺能取得迭代器参与遍历。

函数如果真的返回生成器，Iterator[int] 通常清楚。

如果你不想承诺更具体能力，Iterable[int] 也可能满足公开接口需求。

### 26.3 Generator 的三个类型参数

完整生成器协议可以涉及 yield 元素、send 送入值和最终 return 值。

例如 Generator[YieldType, SendType, ReturnType]。

如果只是普通 for 产生元素，不使用 send 与最终返回值，Iterator[T] 更简单。

不要为了全面就让每个入门生成器都背上三层类型参数。

需要高级交互时，再结合已有生成器教材逐一对应运行行为。

### 26.4 async def 的返回注解不是 Coroutine 外壳吗？

```python
async def fetch_count() -> int:
    return 3
```

这里 `-> int` 描述等待这个协程后得到的结果。

普通调用 fetch_count() 得到协程对象；await 之后才得到整数。

不要因为调用阶段得到协程，就把所有 async def 的返回注解都机械写成 Coroutine[..., int]，从而多套一层含义。

### 26.5 参数接收“可以等待的对象”怎么写？

可以使用 collections.abc.Awaitable[T] 等接口描述。

如果需要特定协程能力，再考虑 Coroutine 的更具体参数。

同样遵循真实所需能力，而不是把名字最长的类型当作最准确。

### 26.6 异步生成器

```python
from collections.abc import AsyncIterator


async def async_ids(limit: int) -> AsyncIterator[int]:
    for value in range(1, limit + 1):
        yield value
```

这里同时出现 async def 与 yield，是异步生成器相关行为。

消费需要 async for 等适合异步迭代的方式。

它不同于“返回一个普通列表的协程”。

注解应描述实际调用与消费协议。

## 第二十七章：Annotated、Final 与 ClassVar，不要只看名字猜运行效果

### 27.1 Annotated 在基础类型旁附加元数据

```python
from typing import Annotated

PositiveCount = Annotated[int, "must be positive"]
```

这份字符串元数据本身不会让普通 Python 拒绝负数。

它只是类型描述旁的附加资料，具体工具可以选择解释。

Pydantic 会解释某些受支持的 Field 元数据；另一个工具可能完全不使用这份字符串。

### 27.2 元数据没有统一魔法

你写 Annotated[int, "最大100"]，检查器不会自动读中文产生数值范围证明。

真正约束来自相应工具明确支持的机制和运行规则。

因此，读到 Annotated 时，除了看第一项类型，还要问后面的元数据由谁处理。

没有解释者，就不应假定有验证行为。

### 27.3 Final 限制重新绑定的静态意图

```python
from typing import Final

DEFAULT_PORT: Final[int] = 8000
```

检查器可以提醒你不要再次给 DEFAULT_PORT 赋另一个值。

但普通运行时不会仅因为 Final 就自动禁止赋值。

它也不等于 C 语言那样的底层内存只读保护。

### 27.4 Final 的列表还能 append 吗？

Final 关注相应名称重新绑定等约束，不是递归冻结对象内容。

如果绑定的是可变列表，列表自身仍有普通可变行为。

这与 Pydantic frozen 的浅层限制、Mapping 的只读接口描述，都需要分别理解。

“名称不可重绑”“接口不提供写入”“对象实际不可变”，是不同保证。

### 27.5 ClassVar 表达类级字段

```python
from typing import ClassVar


class Worker:
    kind: ClassVar[str] = "exporter"

    def __init__(self, name: str) -> None:
        self.name = name
```

它告诉静态工具 kind 主要属于类级约定。

不应简单当作每个实例独立声明的普通字段。

某些框架或 dataclass 也会据此区分处理，但具体运行效果取决于使用者。

### 27.6 `@final` 会阻止运行期继承吗？

它主要让静态工具知道某个类不应再派生，或某个方法不应重写。

普通运行时不因此自动执行完整继承禁令。

如果你需要运行期限制，必须使用相应真实机制。

类型提示中的限制词，经常表达工具契约，不是操作系统式权限屏障。
