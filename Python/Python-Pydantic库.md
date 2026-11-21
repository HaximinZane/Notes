# Python Pydantic v2 超详细教程：从杂乱输入，到类型转换、字段校验与明确的数据契约

## 零基础先导：Pydantic 为什么不是“再写一遍类型注解”？

### 从一条来自外部的数据开始

你已经会写 `age: int`，但 Python 普通类型注解主要告诉读者和静态检查器“我期望它是整数”，**并不默认检查用户实际传入了什么**：

```python
def show_age(age: int):
    print(age)

show_age("二十")  # Python 不会只因为注解就自动拒绝字符串
```

假设你从网络响应、环境变量、表单、JSON 文件收到数据。它可能缺字段、类型不符、数值越界。你可以一项项写 `if`，但是字段很多时，会出现重复判断。**Pydantic 是一套运行时数据验证与解析工具**；它根据模型声明来验证输入，还能在允许的条件下进行类型转换。

```python
from pydantic import BaseModel, Field, ValidationError

class Student(BaseModel):
    name: str
    age: int = Field(ge=0, le=150)  # ge=greater or equal，le=less or equal

try:
    student = Student(name="小林", age="20")
    print(student.age)  # 默认宽松模式下，"20" 可以转换成整数 20
except ValidationError as exc:
    print(exc)
```

逐行理解：`BaseModel` 是 Pydantic 提供的模型基类；`class Student(...)` 声明结构；`name`、`age` 是字段；`Field` 添加数值范围规则；`Student(...)` 创建对象时才进行验证；失败时通常抛 `ValidationError`。

不要把“自动转换字符串 20”为整数误认为“始终严格要求原始输入就是 int”。如果你的接口要求**严格类型**，需要配置严格模式或字段的严格约束。Pydantic v2 与 v1 的部分接口差别很大，本教程后面的 `model_validate`、`model_dump` 应按 v2 理解。

最后区分：`dataclasses` 偏向减少数据类样板代码，`typing` 偏向表达类型意图，`Pydantic` 偏向在**运行时检查外部数据**。拿到合法的模型对象后，你依然要处理业务规则与数据库逻辑，Pydantic 不会替你完成一切。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

假设你正在写一个小脚本。

输入来自 JSON 文件、HTTP 响应、环境变量，或者用户提交的字典。

你希望年龄是整数，姓名不能为空，列表里每一项都有合法编号，而且开始日期不能晚于结束日期。

最初你可能写几个 `if`。

字段增加后，判断会散落在不同函数里；某个入口忘记校验，后面就出现难以定位的错误。

Pydantic 主要帮助你在数据进入程序时，把结构、类型、转换和验证规则组织清楚。

它是第三方库，不是大型 Web 框架，也不要求你先使用 FastAPI。

本文使用 **Pydantic v2**，实际接口验证基线为 **2.14.2**，示例语法以 Python 3.11 及以上为主。

v1 的旧写法会在必要处指出，但不会把两代接口混成一套。

## 第一章：我已经写了类型注解，为什么还需要验证库？

### 1.1 先看一段非常普通的 Python

```python
def repeat(value: int) -> int:
    return value * 2


print(repeat(3))
print(repeat("3"))
```

通常输出：

```text
6
33
```

你可能会问：参数不是已经标了 `int` 吗？为什么还能传字符串？

因为普通 Python 类型注解主要提供类型信息，并不会自动在每次调用时执行运行期检查。

字符串乘以二，本来就是合法的字符串重复操作，所以函数可以继续执行。

### 1.2 注解有什么用？

它能帮助读者、编辑器和静态检查工具理解预期类型。

如果你使用静态类型检查器，这种不符合注解的调用可以在运行前被发现。

但程序面对外部 JSON、环境变量和 HTTP 数据时，仍然可能收到与预期不同的真实值。

因此，类型描述与运行期数据入口验证需要分别理解。

### 1.3 我自己写 if，不也能检查吗？

当然可以。

```python
def validate_age(value):
    if type(value) is not int:
        raise ValueError("年龄必须是整数")
    if not 0 <= value <= 150:
        raise ValueError("年龄超出允许范围")
    return value
```

少量字段时，这种写法很清楚。

但假设还有姓名、邮箱、地址、多个订单项、可选日期，以及字段之间的关系，就会出现大量重复判断。

Pydantic 提供声明模型与校验流程，帮助你把这些规则集中组织，并生成结构化错误。

### 1.4 它会帮我决定所有业务规则吗？

不会。

它不知道你的年龄上限应当是一百五十，还是某个特定业务范围。

也不知道空字符串是否有效、未知字段应该拒绝还是忽略、金额单位是什么。

这些契约仍然由你定义。

库负责按你的规则执行相应转换与验证，而不替你发明业务含义。

## 第二章：Pydantic 是什么？它处在数据流程的哪一步？

### 2.1 先把数据流画清楚

```text
文件、HTTP、环境变量等外部来源
    ↓ 读取或传输
原始字节／文本
    ↓ JSON等格式解析，或Pydantic对应入口
Python数据／JSON输入
    ↓ Pydantic模型验证与转换
符合模型约定的结果对象
    ↓ 业务计算
输出模型或明确序列化
```

HTTP 状态检查、JSON 语法解析和业务模型验证不是同一件事。

你需要在正确的位置处理每一层问题。

### 2.2 项目名称与用途

Pydantic 是这个验证库的项目名称。

学习时更值得记住它的正式能力：基于 Python 类型注解组织数据验证和序列化。

v2 的实现还使用底层验证引擎完成大量工作，但普通使用者首先面对的是 Python 模型接口。[官方模型说明](https://docs.pydantic.dev/latest/concepts/models/)

不必为了会写第一个模型，先研究底层 Rust 实现。

### 2.3 验证会不会改变输入？

验证过程可能执行类型转换和你定义的规范化。

例如字符串 `"20"` 在允许转换的整数模型中，可以变成整数 `20`。

模型输出的重点是符合声明约定，不是逐字节保留输入的原始表现。

如果你需要审计原始数据，应另外保留原始字节或原始对象，不要把规范化结果当作原始证据副本。

### 2.4 它不是数据库约束的替代品

模型可以检查一条记录的字段，但不能仅凭一个内存对象保证数据库里的用户名全局唯一，也不能自动处理多个并发请求的余额竞争。

数据库约束、事务和应用授权仍有独立职责。

入口校验有价值，但它只是完整数据系统的一层。

## 第三章：安装与版本，先避免 v1、v2 混用

### 3.1 安装

```bash
python -m pip install "pydantic>=2,<3"
```

实际项目应该遵守已有依赖管理方式，不要给系统 Python 随意覆盖包。

可以在项目虚拟环境中安装，或者按项目的 uv、Poetry 等约定添加依赖。

### 3.2 验证当前导入

```python
import sys
import pydantic

print(sys.executable)
print(pydantic.__version__)
print(pydantic.__file__)
```

如果版本是一点几，下面某些 v2 接口就不会按相同方式提供。

如果你自己有一个 `pydantic.py`，也可能遮蔽实际包。

### 3.3 常见代际差异

v2 常见接口包括：

```text
model_validate
model_validate_json
model_dump
model_dump_json
model_json_schema
field_validator
model_validator
ConfigDict
```

旧教程中可能出现 `parse_obj()`、`dict()`、`json()`、`validator()` 和旧式内部 `Config` 等。

有些保留兼容入口，有些已弃用或行为变化。

本篇直接采用 v2 主接口，遇到旧项目则应按官方迁移说明处理，不要只替换一个方法名就假定全部语义一致。

## 第四章：第一个模型，逐字拆开

### 4.1 先写完整小例子

```python
from pydantic import BaseModel


class Student(BaseModel):
    name: str
    age: int


student = Student(name="小林", age="20")

print(student.name)
print(student.age)
print(type(student.age).__name__)
```

通常输出：

```text
小林
20
int
```

### 4.2 `class Student(BaseModel)` 是什么意思？

`class` 定义一个类。

`Student` 是你为数据模型取的名称。

括号里的 `BaseModel` 表示继承 Pydantic 的模型基类。

因此，这个类不只是一个普通存放属性的空壳，还参与 Pydantic 提供的验证、错误处理和序列化流程。

### 4.3 `name: str` 是普通赋值吗？

它声明字段 `name` 的预期类型。

这行没有写默认值。

Pydantic 在构建模型时读取这些声明，建立相应字段规则。

普通类中的注解本身不会自动做到这一切，关键还在于它是一个 Pydantic 模型。

### 4.4 为什么 age 传字符串，结果成了整数？

默认情况下，一些合理转换是允许的。

输入字符串 `"20"` 可以解释为整数，于是模型结果存放整数 `20`。

但 `"twenty"` 不能按这个规则变成整数，应该触发验证错误。

允许哪些转换，是库的具体规则；你也可以通过严格模式和自定义规则收紧。

### 4.5 模型对象是字典吗？

不是。

你通常通过属性访问字段：

```python
student.name
```

需要字典输出时：

```python
data = student.model_dump()
```

不要把模型对象和它导出的字典混为同一个东西。

### 4.6 创建模型之后，原字典会被原地改成模型吗？

看：

```python
raw = {"name": "小林", "age": "20"}
student = Student.model_validate(raw)

print(raw["age"])
print(type(raw["age"]).__name__)
print(type(student.age).__name__)
```

这个简单例子中，原字典的 `age` 仍是字符串，模型里的字段是整数。

模型验证生成符合规则的结果，不等于把你的变量绑定和全部嵌套对象都统一原地修改。

更复杂的共享、实例重用和自定义验证器副作用，后面还会讨论。

## 第五章：输入错误时，怎样读懂 ValidationError？

### 5.1 故意传一次错误

```python
from pydantic import BaseModel, ValidationError


class Student(BaseModel):
    name: str
    age: int


try:
    Student(name="小林", age="twenty")
except ValidationError as error:
    print(error)
```

错误通常会指出字段 `age`、输入内容以及不能解析成整数的原因。

具体显示格式可能随版本略有变化，不需要背整段错误文本。

### 5.2 想给程序处理错误呢？

```python
try:
    Student(name="小林", age="twenty")
except ValidationError as error:
    for item in error.errors():
        print(item["loc"])
        print(item["type"])
        print(item["msg"])
```

`loc` 表示位置。

`type` 是错误类型标识。

`msg` 是可读说明。

嵌套结构中，位置还可能包含字段名、数组索引和更深层路径。

### 5.3 能不能一次报告多个错误？

很多情况下可以收集多个字段问题，而不是只看到第一个。

```python
try:
    Student(name=None, age="bad")
except ValidationError as error:
    print(error.error_count())
```

这个例子可能同时包含姓名类型和年龄解析错误。

但自定义模型级验证的执行顺序、前一步是否已经失败，也会影响后面有哪些规则被执行。

不要假定任何失败情况下所有验证器都会完整运行。

### 5.4 错误里会包含原始输入吗？

可能会。

这有利于调试，但输入可能包含秘密或个人信息。

对外展示时，可以只保留位置、类型和经过处理的简短消息。

```python
try:
    Student(name=None, age="bad")
except ValidationError as error:
    safe_errors = [
        {"loc": item["loc"], "type": item["type"], "message": item["msg"]}
        for item in error.errors(include_input=False, include_url=False)
    ]
    print(safe_errors)
```

自定义错误消息本身也不要拼进真实密码或令牌。

### 5.5 为什么不能直接把 `errors()` 任意交给 `json.dumps()`？

自定义验证错误的上下文可能包含异常对象等非 JSON 原生值。

需要 JSON 错误文本时，可以使用 Pydantic 提供的错误序列化入口，或主动构建你允许的简单字段结构。

不要为了编码失败就无条件 `default=str` 把所有内部对象公开出去。

## 第六章：必填、默认值和 None，三个问题分别问

### 6.1 没写默认值，通常就是必填

```python
class Profile(BaseModel):
    name: str
```

创建时不提供 `name`，通常报告缺失字段。

字段类型告诉你值应是什么，是否有默认值告诉你缺失时能否自动补上。

### 6.2 `str | None` 为什么仍然会说必填？

看这个模型：

```python
from pydantic import BaseModel, ValidationError


class Profile(BaseModel):
    nickname: str | None


try:
    Profile()
except ValidationError as error:
    print(error.errors()[0]["type"])
```

通常输出：

```text
missing
```

`str | None` 表示允许值是字符串或 None。

它没有自动提供一个缺失时使用的默认值。

所以“允许空值”和“允许不提供”必须分开理解。

### 6.3 如果确实想让它可以省略呢？

```python
class Profile(BaseModel):
    nickname: str | None = None
```

这次明确给出默认值 None。

```python
print(Profile().nickname)
print(Profile(nickname=None).nickname)
```

两次字段结果都可以是 None，但是否显式提供过这个字段，仍然可以通过模型信息区分。

### 6.4 普通默认值

```python
class PageOptions(BaseModel):
    page: int = 1
    page_size: int = 20
```

调用者省略时使用默认值。

显式提供不合法值时，仍然应该验证，不会因为有默认值就自动忽略错误输入。

### 6.5 缺失与明确 null 对更新有什么影响？

假设接口用于更新昵称。

没提供昵称，可能表示保持旧值。

提供 `null`，可能表示主动清除。

虽然最终字段都可以是 None，更新逻辑却需要知道这次到底有没有提供。

```python
from pydantic import BaseModel


class ProfilePatch(BaseModel):
    nickname: str | None = None


first = ProfilePatch()
second = ProfilePatch(nickname=None)

print(first.model_fields_set)
print(second.model_fields_set)
```

通常分别得到空集合和包含 `nickname` 的集合。

这为后面的 `exclude_unset=True` 提供基础。

### 6.6 `Optional[str]` 是不是更“可选”？

`typing.Optional[str]` 主要等价于允许 `str | None` 的类型表达。

它并不额外表达默认值。

在 v2 模型中，仍然要明确写 `= None` 才表示缺失时使用该默认值。

不要仅凭英文 optional 的日常含义，推断字段一定可以省略。

## 第七章：Field 到底是在做什么？

### 7.1 先约束一个年龄

```python
from pydantic import BaseModel, Field


class Student(BaseModel):
    name: str
    age: int = Field(ge=0, le=150)
```

`Field(...)` 用于提供字段默认值、约束和元数据等资料。

`ge` 来自 greater than or equal，表示大于等于。

`le` 来自 less than or equal，表示小于等于。

因此，这里允许零和一百五十这两个边界。

### 7.2 `gt` 与 `ge` 有什么区别？

```text
gt=0：严格大于零，不包括零。
ge=0：大于等于零，包括零。
```

同理，`lt` 不包括边界，`le` 包括边界。

金额、数量和页号等字段经常会用到这类规则。

### 7.3 同时有默认值怎么办？

```python
class PageOptions(BaseModel):
    page: int = Field(default=1, ge=1)
    page_size: int = Field(default=20, ge=1, le=100)
```

`default` 处理缺失。

`ge`、`le` 处理验证规则。

两类配置放在同一个 Field 里，但职责不同。

### 7.4 `Field(...)` 的省略号是什么意思？

你可能看到：

```python
name: str = Field(..., min_length=1)
```

在这里省略号用于表达必填，而不是“后面代码自己补”的注释。

为了减少初学混淆，本篇多数必填字段直接不提供默认值：

```python
name: str = Field(min_length=1)
```

这仍然是必填字段。

### 7.5 说明文字会自动变成验证器吗？

```python
age: int = Field(description="年龄必须合法")
```

`description` 主要提供描述元数据。

它不会读懂中文“合法”并自动推导零到一百五十的范围。

需要实际限制，仍要写 `ge=0`、`le=150` 或对应验证规则。

这也是 Schema 文档与运行期规则需要区别的一处。

## 第八章：默认值看起来写对了，为什么还可能留下错误类型？

### 8.1 默认值不会总是经过普通输入验证

看这个例子：

```python
from pydantic import BaseModel


class Options(BaseModel):
    retries: int = "3"


options = Options()
print(repr(options.retries))
print(type(options.retries).__name__)
```

在默认配置下，你可能看到：

```text
'3'
str
```

这与显式输入 `Options(retries="3")` 后得到整数的情况不同。

原因是默认值默认不总经过同样的验证流程。

因此，模型作者自己写下的默认值也需要认真检查。

### 8.2 如果希望默认值也验证呢？

```python
from pydantic import BaseModel, Field


class Options(BaseModel):
    retries: int = Field(default="3", validate_default=True)


options = Options()
print(options.retries)
print(type(options.retries).__name__)
```

这次可以按字段规则把默认字符串转换成整数。

如果默认内容根本不能转换，也应该在模型创建时报告错误。

### 8.3 整个模型统一设置

```python
from pydantic import BaseModel, ConfigDict


class Options(BaseModel):
    model_config = ConfigDict(validate_default=True)
    retries: int = "3"
```

这比给每个字段重复写相同选项更集中。

但配置值仍然需要基于你想要的行为，不应看到一个选项就无条件在所有模型开启。

### 8.4 可变默认值为什么推荐 default_factory？

假设每个用户都有自己的标签列表。

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str
    tags: list[str] = Field(default_factory=list)


first = User(name="小林")
second = User(name="小周")
first.tags.append("python")

print(first.tags)
print(second.tags)
```

通常得到：

```text
['python']
[]
```

`default_factory=list` 表示需要默认值时调用 `list`，每次得到一个新列表。

这里传的是可调用对象 `list`，不是先调用得到的 `list()`。

### 8.5 Pydantic 的列表默认值是不是一定共享？

不能简单照搬普通函数可变默认参数的规则。

Pydantic 对一些不可哈希默认值会进行深拷贝等处理，因而 `tags: list[str] = []` 不必然表现为所有实例共用同一个列表。

但显式使用 `default_factory` 更容易表达“每次创建一份”的意图，也更方便处理动态默认值。

理解库的具体行为，比机械背一句“所有列表默认值都共享”准确。[字段默认值说明](https://docs.pydantic.dev/latest/concepts/fields/#mutable-default-values)

### 8.6 动态时间默认值

```python
from datetime import datetime, timezone
from pydantic import BaseModel, Field


class Event(BaseModel):
    message: str
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
```

工厂在需要默认值时执行。

如果你直接在类定义处写 `datetime.now(...)`，时间可能在类定义时就已经确定，后面实例继续使用那个早已产生的默认对象。

所以要分清“现在调用一次得到值”和“保留一个以后调用的工厂”。

## 第九章：Annotated，怎样把可复用约束放进类型描述？

### 9.1 先看没有别名的写法

```python
class Page(BaseModel):
    page: int = Field(ge=1)
    page_size: int = Field(ge=1, le=100)
```

如果多个模型都要使用同样的“正整数 ID”，重复 Field 会逐渐增多。

可以使用 `Annotated` 组织元数据。

### 9.2 定义一个可复用约束类型

```python
from typing import Annotated
from pydantic import BaseModel, Field

PositiveId = Annotated[int, Field(gt=0)]


class User(BaseModel):
    id: PositiveId


class Book(BaseModel):
    id: PositiveId
```

`Annotated` 的第一项是基础类型。

后面附带可以由相关工具解释的元数据。

Pydantic 读取其中的 Field 规则；普通 Python 不会因为你定义了这个别名就自动检查每个整数变量。

### 9.3 它与新建一个真正的类一样吗？

不是必须这样理解。

这里主要是在类型描述上附加验证资料，方便重用。

不要以为 `PositiveId` 一定是一个能在所有地方当作普通类进行 `isinstance()` 判断的运行期类型。

类型表达与运行对象体系不是完全同一层。

### 9.4 为什么字段默认值通常仍写在赋值位置？

对默认值、某些别名和编辑器构造签名等信息，静态工具的理解方式有差别。

基础项目可以把常用约束放进 Annotated，把字段默认值明确写出来：

```python
PositiveCount = Annotated[int, Field(ge=0)]


class CounterOptions(BaseModel):
    retries: PositiveCount = 0
```

这样既能复用规则，也能让读者清楚看到省略时使用什么。

## 第十章：字符串校验，为什么“有长度”还不够？

### 10.1 最简单的长度约束

```python
from pydantic import BaseModel, Field


class Note(BaseModel):
    title: str = Field(min_length=1, max_length=80)
```

这限制字符串长度，但一个只包含空格的标题仍然有长度。

如果业务要求去掉两端空白后不能为空，需要明确规范化规则。

### 10.2 使用 StringConstraints

```python
from typing import Annotated
from pydantic import BaseModel, StringConstraints

CleanTitle = Annotated[
    str,
    StringConstraints(strip_whitespace=True, min_length=1, max_length=80),
]


class Note(BaseModel):
    title: CleanTitle


note = Note(title="   Python 文件处理   ")
print(repr(note.title))
```

通常得到：

```text
'Python 文件处理'
```

这里明确把空白清理纳入字段规则，而不是在程序某个不相关角落补一次 `.strip()`。

### 10.3 如果输入只有空格呢？

经过清理后变成空字符串，会违反最小长度要求。

可以主动验证：

```python
from pydantic import ValidationError

try:
    Note(title="   ")
except ValidationError:
    print("空白标题被拒绝")
```

### 10.4 `pattern` 会自动完整匹配吗？

不要把正则查找与完整字符串契约混为一谈。

如果你希望编号严格由两个大写字母和四个数字组成，可以明确写边界：

```python
from pydantic import BaseModel, Field


class Ticket(BaseModel):
    code: str = Field(pattern=r"^[A-Z]{2}[0-9]{4}$")
```

这样 `AB1234` 满足规则，而多出前后字符的输入不应被当成同一个合法编号。

### 10.5 它与 re 模块的所有语法都相同吗？

Pydantic v2 默认使用的正则引擎与 Python `re` 并非完全相同。

某些回溯引用、环视等特性可能不被默认引擎支持。

复杂正则需求应核对目标版本的支持范围，或在自定义验证器里明确使用你需要的处理方式。

不要把一条在 `re.fullmatch()` 中能运行的表达式，未经验证就直接搬进 Field。

### 10.6 字符数与人眼看到的字形数量

字符串长度通常按 Python Unicode 字符串的码点数量理解。

组合字符和某些表情可能由多个码点组成。

如果业务要求限制“用户看到的字形数量”，普通 min_length/max_length 不能自动替你完成全部 Unicode 分段语义。

大多数普通标题限制可以先使用码点数量，但应知道单位是什么。

## 第十一章：列表、字典和元组，类型描述怎样逐层起作用？

### 11.1 `list[int]` 不只检查外面是列表

```python
from pydantic import BaseModel


class Scores(BaseModel):
    values: list[int]


scores = Scores(values=["95", "88", 100])
print(scores.values)
```

默认允许的转换会逐项应用，结果可以是整数列表。

如果其中一项是 `"bad"`，错误位置通常包含那个元素的索引。

### 11.2 限制列表数量

```python
from pydantic import BaseModel, Field


class Scores(BaseModel):
    values: list[int] = Field(min_length=1, max_length=10)
```

这里的长度约束针对列表元素数量，不是每个整数的位数。

每项的数值范围要放进元素类型：

```python
from typing import Annotated
from pydantic import BaseModel, Field

Score = Annotated[int, Field(ge=0, le=100)]


class Scores(BaseModel):
    values: list[Score] = Field(min_length=1, max_length=10)
```

外层约束列表数量，内层约束每项范围。

### 11.3 字典的键和值都能描述

```python
class Counts(BaseModel):
    values: dict[str, int]
```

它描述键为字符串、值为整数的映射。

是否允许具体未知键名，仍然取决于模型结构和业务规则。

`dict[str, int]` 不能自动知道只允许 `python`、`linux` 两个分类。

### 11.4 固定长度元组

```python
class Point(BaseModel):
    position: tuple[float, float]
```

这里要求两项，对应两个浮点字段。

这与可变长度 `tuple[float, ...]` 不同。

序列结构的长度和每项类型都应按你实际需要表达。

### 11.5 集合会不会悄悄去重？

```python
class Tags(BaseModel):
    values: set[str]
```

集合语义会去重，并且不提供列表那样的顺序契约。

如果输入重复项本身应当报错，而不是自动合并，就不能只靠 set 类型表达。

需要保留列表，再定义重复检测规则。

### 11.6 巨大容器是否需要上限？

需要根据输入风险和业务规模决定。

类型正确的一百万个元素，仍然会消耗明显资源。

模型规则可以限制元素数量，但原始请求体读取、JSON 解析和外部系统限制也要一起考虑。

不能把一个 max_length 当成整条输入链路的完整资源沙箱。

## 第十二章：嵌套模型，让每层数据都具有明确结构

### 12.1 从一个地址对象开始

```python
from pydantic import BaseModel, Field


class Address(BaseModel):
    city: str = Field(min_length=1)
    street: str = Field(min_length=1)


class User(BaseModel):
    name: str
    address: Address
```

构造时可以传嵌套字典：

```python
user = User.model_validate({
    "name": "小林",
    "address": {"city": "上海", "street": "示例路 1 号"},
})

print(type(user.address).__name__)
print(user.address.city)
```

`address` 会按 Address 规则验证并构造成对应模型。

### 12.2 为什么不直接写 `address: dict`？

普通 dict 只表达它是映射，不说明必须有哪些字段、字段类型和范围。

嵌套模型把这层契约写清楚。

以后查看 User，你就能知道地址还有一套结构，而不必追踪程序里所有取键代码。

### 12.3 列表里放模型

```python
class Team(BaseModel):
    name: str
    members: list[User]
```

每个成员都按 User 验证，成员的地址继续按 Address 验证。

错误位置可能类似：

```text
members → 1 → address → city
```

这比只告诉你“整个 team 不合法”更容易找到问题。

### 12.4 嵌套模型会自动继承所有配置吗？

不要这样假设。

每个模型可以有自己的 ConfigDict。

父模型的配置不会把所有独立定义的子模型行为随意改成完全相同。

如果你希望项目模型共用某种政策，可以定义一个受控基类，让需要该政策的模型继承它。

### 12.5 输入已经是模型实例，还会重新完整验证吗？

默认实例重验证政策可能允许直接使用已有模型实例，而不是每次都重新检查全部内部状态。

如果实例之前被绕过验证地修改，不能只因为再次传入外层模型就认为所有错误都会被发现。

后面关于赋值校验与 `revalidate_instances` 会继续展开。

## 第十三章：严格模式，究竟严格在哪里？

### 13.1 我不想把字符串数字自动转成整数

可以在验证调用时要求严格：

```python
from pydantic import BaseModel, ValidationError


class Count(BaseModel):
    value: int


try:
    Count.model_validate({"value": "3"}, strict=True)
except ValidationError:
    print("严格输入拒绝字符串整数")
```

正常整数 `3` 则可以接受。

### 13.2 某个字段严格

```python
from pydantic import BaseModel, Field


class Count(BaseModel):
    value: int = Field(strict=True)
```

这样把政策放进字段定义，而不是要求每个调用者记得传参数。

### 13.3 整个模型严格

```python
from pydantic import BaseModel, ConfigDict


class Count(BaseModel):
    model_config = ConfigDict(strict=True)
    value: int
```

严格规则按各类型的具体语义执行，不应简单翻译成“任何类型都绝对不做任何转换”。

日期、UUID 和 JSON 表示等情况需要特别理解输入来源。

### 13.4 Python 字典与 JSON 输入为什么可能不同？

```python
from datetime import date
from pydantic import BaseModel, ConfigDict, ValidationError


class Appointment(BaseModel):
    model_config = ConfigDict(strict=True)
    day: date


try:
    Appointment.model_validate({"day": "2026-11-02"})
except ValidationError:
    print("Python严格输入拒绝日期字符串")

appointment = Appointment.model_validate_json('{"day":"2026-11-02"}')
print(appointment.day)
```

JSON 本身没有 Python `date` 对象这种原生值，所以 JSON 路径中某些类型允许按规定字符串表示解析。

这不与严格模式矛盾，而是严格政策考虑了输入格式能够表达什么。

因此，`json.loads()` 后再 `model_validate(..., strict=True)`，不必然与直接 `model_validate_json(..., strict=True)` 在所有类型上完全一样。[严格模式官方说明](https://docs.pydantic.dev/latest/concepts/strict_mode/)

### 13.5 布尔值能不能作为严格整数？

Python 中 bool 是 int 子类，但严格整数验证会按它自己的规则拒绝布尔值冒充普通整数。

```python
Count.model_validate({"value": True})
```

在前面严格 Count 模型中，应当报告验证错误。

这对计数、ID 和金额最小单位等字段很重要。

### 13.6 严格模式还能被自定义转换影响吗？

如果你在 before 验证器里主动把某个字符串转成整数，再交给字段验证，严格字段看到的已经是转换后的整数。

严格模式不会撤销你自己明确定义的预处理。

所以真正的输入契约需要把配置与自定义验证器一起审阅。

## 第十四章：Literal、枚举和联合类型

### 14.1 只允许几个固定值

```python
from typing import Literal
from pydantic import BaseModel


class DisplayOptions(BaseModel):
    theme: Literal["light", "dark"] = "light"
```

`Literal` 表达允许的具体值集合。

输入 `"blue"` 应当被拒绝。

这是类型层面的有限选项描述，不是让代码根据任意字符串自动导入某个类。

如果选项是数字，还要继续问：你想限制数值，还是连输入的精确类型也限制？

以本篇验证的 Pydantic v2 为例：

```python
from typing import Literal
from pydantic import BaseModel


class NumericVersion(BaseModel):
    version: Literal[1]


print(NumericVersion(version=True).version)
print(NumericVersion(version=1.0).version)
```

这两个输入都可能被接受，并得到整数一。

`Literal[1]` 不能单独当成“输入必须是精确的 `int`，而且值为一”的保证。

如果你的文件协议明确禁止布尔值和浮点版本号，可以在 `before` 验证器里检查原始类型，再交给 Literal 检查允许的数值。

第二十九章的配置实践会这样处理版本字段；不要把数值选项和字符串选项的边界完全等同。

### 14.2 枚举适合可复用选项

```python
from enum import Enum
from pydantic import BaseModel


class TaskStatus(str, Enum):
    pending = "pending"
    done = "done"


class Task(BaseModel):
    status: TaskStatus = TaskStatus.pending
```

模型结果中的字段可能是枚举成员。

导出为 JSON 兼容结构时，再按相应序列化规则转成值。

不要把“输入是字符串”和“模型内部保存字符串子类枚举成员”完全混为一谈。

### 14.3 `int | str` 会怎么选择？

联合类型允许多个分支。

Pydantic v2 的默认联合选择有自己的匹配策略，不能只靠“写在左边就必然先强制转换成功”来推断所有结果。

如果业务明确区分数字 ID 和文字 ID，最好设计不含歧义的模型，或者明确指定所需联合政策并验证边界。

### 14.4 多种记录最好加标签

```python
from pathlib import Path
from typing import Annotated, Literal
from pydantic import BaseModel, Field


class FileTask(BaseModel):
    kind: Literal["file"]
    path: Path


class TextTask(BaseModel):
    kind: Literal["text"]
    content: str


TaskInput = Annotated[FileTask | TextTask, Field(discriminator="kind")]


class Job(BaseModel):
    task: TaskInput
```

输入：

```python
job = Job.model_validate({
    "task": {"kind": "text", "content": "学习数据校验"}
})

print(type(job.task).__name__)
```

通常得到 `TextTask`。

### 14.5 为什么有标签更容易理解？

`kind` 直接说明采用哪一种结构。

错误信息也更容易围绕选中的分支展开。

如果没有标签，两个结构存在很多相似字段，验证器可能需要尝试多个分支，调用者也更难知道一个数据究竟想表示什么。

这属于协议设计改进，不只是库语法技巧。

## 第十五章：字段验证器，什么时候需要自己写函数？

Field 已经覆盖了许多数值和长度限制。

但一些业务规则无法只用范围表达，例如用户名不能包含某个保留词，文件任务名称不能只剩空白，或输入兼容旧格式。

这时可以定义字段验证器。

### 15.1 最简单的 after 验证器

```python
from pydantic import BaseModel, field_validator


class UserName(BaseModel):
    name: str

    @field_validator("name")
    @classmethod
    def reject_reserved_name(cls, value):
        if value.casefold() in {"admin", "system"}:
            raise ValueError("这个名称是保留名称")
        return value
```

默认字段验证模式是 after。

对于这个字符串字段，正常类型解析完成后，再把结果交给函数检查。

### 15.2 为什么要返回 value？

验证器的返回值会成为后续结果。

如果你只是检查后忘记 `return value`，普通 Python 函数默认返回 None。

字段结果就可能被你自己的回调错误改变。

因此，“检查失败时抛错，检查成功时返回值”是必须清楚的约定。

### 15.3 `@classmethod` 与 cls 是什么？

这类验证器通常按类方法方式组织。

`cls` 表示当前模型类，不是某个已经完成的实例。

装饰器表达式的应用顺序仍遵循 Python 装饰器规则。

你可以先按官方推荐排列写，再结合已有装饰器笔记理解它为何如此组织。

### 15.4 为什么通常抛 ValueError？

它适合表达“输入值不符合这条规则”。

Pydantic 会把相应错误整合进验证错误结构。

不要把编程上的 TypeError 当成通用输入拒绝机制。

v2 中验证器内部某些 TypeError 不会像旧习惯那样统一包装为 ValidationError，真实 bug 应当暴露并修正。

### 15.5 可以使用 assert 吗？

有些示例用断言表达规则。

但 Python 优化模式可能移除 assert。

必须执行的业务验证，使用明确的 `if ...: raise ValueError(...)` 更容易保证意图。

不要让用户输入是否合法取决于程序是否带了 `-O` 启动。

## 第十六章：before 与 after，先后顺序为什么会改变结果？

### 16.1 before 先看原始输入

```python
from pydantic import BaseModel, field_validator


class TagsInput(BaseModel):
    tags: list[str]

    @field_validator("tags", mode="before")
    @classmethod
    def split_comma_text(cls, value):
        if isinstance(value, str):
            return value.split(",")
        return value
```

这允许一个兼容输入：

```python
obj = TagsInput(tags="python,backend")
print(obj.tags)
```

得到：

```text
['python', 'backend']
```

before 在列表字段的正常验证之前运行，先把某种旧形式整理成可以交给字段规则处理的结构。

### 16.2 before 能假设 value 已经是列表吗？

不能。

它面对的是尚未按字段类型处理的输入，可能是字符串、数字、None 或其他对象。

因此要先做类型判断。

不要在 before 中直接对任意输入调用 `.split()` 或列表方法，导致本来应报告输入错误的地方变成意外属性错误。

### 16.3 after 适合检查已经转换后的值

```python
from pydantic import BaseModel, field_validator


class EvenNumber(BaseModel):
    value: int

    @field_validator("value", mode="after")
    @classmethod
    def require_even(cls, value):
        if value % 2:
            raise ValueError("必须是偶数")
        return value
```

允许默认转换时，字符串 `"4"` 先转为整数，再做偶数检查。

`"bad"` 在整数解析阶段就失败，after 不会拿到一个伪装成整数的字符串继续取模。

### 16.4 如果 after 清理字符串，长度规则会再跑一遍吗？

不要假设会。

考虑：字段先通过最小长度检查，after 再把三个空格 `.strip()` 成空字符串。

如果 after 直接返回这个空字符串，之前的长度约束不一定重新执行。

因此，清理应放在合适阶段，或在清理后明确检查。

例如：

```python
from pydantic import BaseModel, field_validator


class CleanName(BaseModel):
    name: str

    @field_validator("name", mode="after")
    @classmethod
    def clean_and_check(cls, value):
        cleaned = value.strip()
        if not cleaned:
            raise ValueError("清理后姓名不能为空")
        return cleaned
```

前面使用 StringConstraints 的写法也是一种集中表达选择。

### 16.5 可以原地修改传入列表吗？

需要谨慎。

输入对象可能还被调用者使用，也可能参与联合类型其他分支尝试。

如果 before 先原地改了一半，再抛错误，外部原始数据可能已经被改变。

通常更清楚的是构造新列表或新字典，成功后再交给后续验证。

这与普通 Python 可变对象和共享引用知识直接相关。

### 16.6 默认字段会不会执行我的验证器？

如果字段缺失，采用默认值，默认值又没有开启验证，相应流程可能不会按你想象执行。

需要默认值也经过清理或检查时，明确 `validate_default=True`。

不要只测试“调用者显式传了值”的一条路径。

## 第十七章：plain 与 wrap，为什么初学者不应随手使用？

### 17.1 plain 可以跳过普通类型验证

```python
from pydantic import BaseModel, field_validator


class LooseNumber(BaseModel):
    value: int

    @field_validator("value", mode="plain")
    @classmethod
    def accept_directly(cls, value):
        return value
```

这段示例故意展示危险的宽松行为。

```python
obj = LooseNumber(value="not-an-integer")
print(type(obj.value).__name__)
```

可能仍得到 `str`。

因为 plain 模式可以终止相应普通验证流程。

### 17.2 这是不是 Pydantic 没遵守注解？

是你自己选择了允许绕过正常处理的扩展入口。

所以不能只看字段写着 `int`，就忽略验证器模式对最终保证的影响。

普通规则优先使用 after 或 before，让基础类型处理继续清楚可见。

### 17.3 wrap 可以包围内部验证

```python
from pydantic import BaseModel, field_validator


class WrappedName(BaseModel):
    name: str

    @field_validator("name", mode="wrap")
    @classmethod
    def observe_validation(cls, value, handler):
        print("进入字段验证")
        result = handler(value)
        print("字段验证完成")
        return result
```

`handler` 表示继续内部验证流程的入口。

你可以在之前和之后做受控工作。

但如果完全不调用 handler，或者把失败随意改成一个默认结果，就可能改变本来应该保证的规则。

### 17.4 验证器顺序怎样判断？

不同模式与 Annotated 元数据的组合有明确的执行次序。

一般理解是 before／wrap 从外层进入，after 在相应内部处理后执行，但复杂排列不能只靠英文名字猜测。

需要复杂组合时，写一个记录顺序的小实验，对照官方规则验证。

不要在十个验证器相互影响后，靠反复换装饰器顺序碰运气。

### 17.5 多个简单规则是否一定要拆成很多验证器？

不一定。

把高度相关的清理与检查写在一个清楚函数里，可能更容易解释。

只有当规则确实有独立重用、顺序或职责需求时，再拆分组合。

代码层次越多，不自动意味着设计越好。

## 第十八章：跨字段关系，为什么适合 model_validator？

### 18.1 开始日期不能晚于结束日期

单个字段都可能是合法日期，但它们组合起来不合理。

```python
from datetime import date
from typing import Self
from pydantic import BaseModel, model_validator


class DateRange(BaseModel):
    start: date
    end: date

    @model_validator(mode="after")
    def require_ordered_dates(self) -> Self:
        if self.start > self.end:
            raise ValueError("开始日期不能晚于结束日期")
        return self
```

使用：

```python
period = DateRange(start="2026-11-01", end="2026-11-30")
print(period.start)
```

### 18.2 为什么这里使用 self？

after 模型验证器面对的是已经完成相应字段处理的模型实例。

所以可以读取 `self.start` 和 `self.end`。

成功时返回该实例。

这和字段类方法验证器中的 `cls` 不同，不要机械地给所有验证器都加相同签名。

### 18.3 如果 start 根本不是合法日期呢？

字段处理已经失败，模型 after 阶段通常不会再拿着一个完整合法实例执行这条关系检查。

因此，你不应要求每次错误报告都同时出现所有可能业务关系错误。

前置结构不成立时，后续关系检查可能没有合理输入。

### 18.4 为什么不在 end 字段验证器里读 start？

也可以通过 ValidationInfo 等机制取得先前已验证的数据，但会受字段定义顺序和前面字段是否成功影响。

跨多个字段的不变量，用模型 after 验证器通常更直接。

它表达“这些字段作为一个整体必须满足关系”，阅读时不需要猜测某个字段是否已经处理。

### 18.5 before 模型验证器适合什么？

它可以在整个模型字段处理前观察原始输入，例如兼容旧键名、拆分旧结构或拒绝某种输入形状。

但输入不一定总是字典，特别是启用属性读取或实例验证等路径时。

应先检查输入形状，避免把所有值都当 dict 原地修改。

### 18.6 模型验证器里能查数据库吗？

技术上你可以调用其他代码，但网络或数据库副作用会让验证成本、失败方式和可重复性变得复杂。

普通模型验证通常适合检查本地结构与确定性规则。

用户是否存在、库存是否足够、唯一性是否满足等，应在合适业务与事务范围处理。

不要把每次序列化或重验证都可能触发的外部操作藏进一个看似纯数据模型。

## 第十九章：ValidationInfo、字段顺序和 context

### 19.1 查看先前已验证字段

```python
from pydantic import BaseModel, ValidationInfo, field_validator


class ConfirmedValue(BaseModel):
    value: str
    confirmation: str

    @field_validator("confirmation")
    @classmethod
    def must_match(cls, value, info: ValidationInfo):
        original = info.data.get("value")
        if original is not None and value != original:
            raise ValueError("两次内容不一致")
        return value
```

`info.data` 不是“整个原始字典永远都在这里”。

它主要包含当前流程中已完成验证的相关字段资料。

### 19.2 是输入字典顺序决定的吗？

字段定义顺序很重要。

不要以为调用者把某个键先放进 dict，验证器就一定先完成那一项。

如果你把 confirmation 定义在 value 前面，上面的相互访问假设就需要重新考虑。

跨字段关系通常用 model_validator 更清楚。

### 19.3 context 能传什么？

可以在验证调用时提供受控上下文，例如本次允许的分类集合。

```python
from pydantic import BaseModel, ValidationInfo, field_validator


class CategoryInput(BaseModel):
    category: str

    @field_validator("category")
    @classmethod
    def check_allowed(cls, value, info: ValidationInfo):
        context = info.context or {}
        allowed = context.get("allowed_categories")
        if allowed is not None and value not in allowed:
            raise ValueError("分类不在本次允许范围内")
        return value


obj = CategoryInput.model_validate(
    {"category": "python"},
    context={"allowed_categories": {"python", "linux"}},
)
```

### 19.4 context 会自动写进模型字段吗？

不会因为传了它就自动成为序列化字段。

它是这次验证过程的附加资料，与你声明的数据结构不同。

但你的验证器如何使用它，会影响结果，因此调用者也应知道这个依赖。

### 19.5 普通构造调用能直接传 context 吗？

不要把 `context=` 随意当成普通字段以外的特殊参数传给所有 BaseModel 构造器。

本篇采用明确支持上下文的 `model_validate(..., context=...)` 入口。

若需要自定义其他构造路径，必须根据官方机制设计，不能靠多传一个同名键猜测它会自动生效。

## 第二十章：别名，为什么外部 userName 可以对应内部 user_name？

### 20.1 先看两边命名习惯

外部 JSON 可能使用：

```json
{"userName": "小林"}
```

Python 代码更希望写：

```python
user.user_name
```

可以用 alias 连接两种名称。

```python
from pydantic import BaseModel, Field


class UserAlias(BaseModel):
    user_name: str = Field(alias="userName")


user = UserAlias.model_validate({"userName": "小林"})
print(user.user_name)
```

### 20.2 内部字段真的改名了吗？

没有。

Python 属性仍然是 `user_name`。

alias 影响相应输入和输出命名处理，不是让你通过任意外部键访问实例属性。

### 20.3 导出时默认用哪个名字？

```python
print(user.model_dump())
print(user.model_dump(by_alias=True))
```

通常分别是：

```text
{'user_name': '小林'}
{'userName': '小林'}
```

因此，定义了别名不代表每次导出都自动使用别名，输出政策仍要明确。

### 20.4 两种输入名字都希望接受呢？

在较新的 v2 版本中，可以明确配置按字段名和别名验证。

```python
from pydantic import BaseModel, ConfigDict, Field


class FlexibleUser(BaseModel):
    model_config = ConfigDict(validate_by_name=True, validate_by_alias=True)
    user_name: str = Field(alias="userName")
```

`validate_by_name` 等配置从 v2.11 起提供，本篇实际版本支持。

较早 v2 项目可能使用 `populate_by_name`，应根据项目版本和迁移说明选择，不要在所有 v2 版本上假设新配置都存在。

### 20.5 只影响输入或只影响输出

```python
class ExternalUser(BaseModel):
    user_name: str = Field(
        validation_alias="legacyName",
        serialization_alias="displayName",
    )
```

输入按 `legacyName` 取得数据。

需要按输出别名序列化时，使用对应 `by_alias=True` 政策。

这种设计适合旧接口兼容和新输出契约，但名称越多越需要写清楚，不应把模型变成无法判断真实字段来源的映射迷宫。

### 20.6 AliasChoices 和 AliasPath

某些输入在迁移期间允许两个旧键名，或者值藏在嵌套对象中。

可以用受控的别名选择与路径：

```python
from pydantic import AliasChoices, AliasPath, BaseModel, Field


class ImportedUser(BaseModel):
    name: str = Field(validation_alias=AliasChoices("name", "full_name"))
    city: str = Field(validation_alias=AliasPath("address", "city"))
```

这不执行任意对象属性读取代码，而是按相应输入别名规则定位数据。

如果多个候选键同时存在且冲突，应明确你希望的政策，并用测试验证，不能默默假设库替你判断哪个是真实来源。

## 第二十一章：extra，陌生字段是忽略、保存还是拒绝？

### 21.1 一个很容易漏掉的拼写错误

模型定义 `page_size`。

输入却写成：

```json
{"pagesize": 50}
```

如果陌生字段被忽略，程序可能继续使用默认每页二十条，用户还以为设置已经生效。

因此，未知字段政策很重要。

### 21.2 默认忽略与明确拒绝

```python
from pydantic import BaseModel, ConfigDict


class Options(BaseModel):
    model_config = ConfigDict(extra="forbid")
    page_size: int = 20
```

输入 `pagesize` 时应报告额外字段错误。

适合用户手工配置和需要尽早发现拼写错误的入口。

### 21.3 允许并保留额外字段

```python
class FlexibleRecord(BaseModel):
    model_config = ConfigDict(extra="allow")
    id: int
```

额外数据可以进入模型的相应额外字段存储，并参与输出。

如果外部输入含秘密或不希望公开字段，允许所有额外内容后直接导出，可能扩大输出范围。

因此，allow 不是没有后果的“提高兼容性”开关。

### 21.4 子模型是否也拒绝额外字段？

如果子模型独立定义且没有采用同样配置，不能只凭父模型 forbid 就推断所有深层对象也相同。

项目可以定义基类集中政策，让需要的模型继承。

但输入模型、内部模型与外部输出模型未必都适合同一政策，应按用途选择。

### 21.5 未知字段与字典内容不同

一个模型有字段 `metadata: dict[str, str]`，metadata 里面的自由键是这个字段契约的一部分。

它们不自动等同于模型顶层的 extra 字段。

理解“哪一层声明了固定结构，哪一层允许自由映射”，才能正确设计配置和接口。

## 第二十二章：日期、金额、路径和 URL，为什么不要全写成 str？

### 22.1 看起来都是字符串，含义却完全不同

外部输入可能是：

```json
{
  "published_on": "2026-11-03",
  "price": "19.90",
  "output": "reports/result.json",
  "homepage": "https://example.com/books"
}
```

如果全部声明为 `str`，模型主要能确认它们是字符串。

但是，“明年十三月”“金额里夹着汉字”和“不是 URL 的随便一句话”，需要更具体的规则才能区分。

类型描述越贴近业务，后面的代码越少需要重复解析。

### 22.2 date 与 datetime 怎样选择？

只有年月日，例如生日、出版日期，可以使用 `datetime.date`。

涉及某一具体时刻，例如采集时间、任务结束时间，可以使用 `datetime.datetime`。

```python
from datetime import date, datetime
from pydantic import BaseModel


class Publication(BaseModel):
    published_on: date
    collected_at: datetime


item = Publication(
    published_on="2026-11-03",
    collected_at="2026-11-03T10:30:00+08:00",
)
print(type(item.published_on).__name__)
print(item.collected_at.utcoffset())
```

通常输出：

```text
date
8:00:00
```

`+08:00` 表示输入携带八小时的 UTC 偏移。

验证后得到 datetime 对象，你可以调用日期时间方法，而不用在每个业务函数里再次拆字符串。

### 22.3 写了 datetime，就保证有时区了吗？

不保证。

普通 datetime 类型可以涉及有时区和没有时区的对象。

如果你的数据必须明确表示时间线上的时刻，可以采用更具体的类型：

```python
from pydantic import AwareDatetime, BaseModel


class CollectedRecord(BaseModel):
    collected_at: AwareDatetime
```

试着输入：

```python
record = CollectedRecord(collected_at="2026-11-03T10:30:00+08:00")
print(record.collected_at.tzinfo is not None)
```

输出：

```text
True
```

如果改成没有偏移的 `"2026-11-03T10:30:00"`，应当拒绝。

但“有时区”不等于“已经统一为 UTC”。

需要统一保存时，明确调用 `astimezone(timezone.utc)`，或在经过验证的业务处理阶段做规范化。

不要看见日期时间字符串就自行加上 UTC 标记；缺失信息时，应先确定输入协议。

### 22.4 金额为什么常用 Decimal？

二进制浮点数不适合直接表达所有十进制金额。

Pydantic 可以把十进制字符串转换成 Decimal，并施加位数约束。

```python
from decimal import Decimal
from pydantic import BaseModel, Field


class PriceTag(BaseModel):
    amount: Decimal = Field(ge=0, max_digits=8, decimal_places=2)
    currency: str = "CNY"


price = PriceTag(amount="19.90")
print(price.amount)
print(type(price.amount).__name__)
```

通常输出：

```text
19.90
Decimal
```

`max_digits` 约束数字总位数，`decimal_places` 约束小数位数，具体计算按库的 Decimal 规则执行。

这些规则用于拒绝不符合精度契约的数据，不应被误认为自动采用了你想要的财务舍入政策。

### 22.5 三位小数会自动四舍五入吗？

不应这样假设。

如果约定最多两位小数，输入 `"19.999"` 应先作为超出约束处理。

若业务允许舍入，应明确说明采用哪一种 Decimal 舍入方式，在哪一步执行，并测试临界值。

“允许输入三位小数但规范成两位”与“拒绝三位小数”，是两份不同的数据协议。

另外，若最初传来的已经是二进制 float，最初的十进制书写信息可能已经丢失。

从 JSON 或 CSV 读取货币时，保留十进制文本再转 Decimal，通常更容易控制来源含义。

### 22.6 Path 类型会自动创建目录吗？

不会。

```python
from pathlib import Path
from pydantic import BaseModel


class ExportOptions(BaseModel):
    output: Path


options = ExportOptions(output="reports/result.json")
print(isinstance(options.output, Path))
```

输出：

```text
True
```

这里只是形成路径对象。

不代表文件存在，也不代表目录已经建立，更不代表程序拥有写入权限。

相对路径仍然需要基准目录。

配置文件放在 `/home/alice/project/config.json`，并不意味着其中的 `reports/result.json` 自动相对配置文件解释。

如果你希望这样，应在读取配置后明确使用配置文件的父目录组合路径。

这与 [pathlib 教程](Python-pathlib库.md) 中“路径对象不等于文件系统对象已经存在”的原则一致。

### 22.7 FilePath 与 DirectoryPath 检查到了什么？

它们可以要求验证时相应路径存在并符合文件或目录类型。

这适合本地工具的输入文件。

但一次验证通过，只说明检查发生时满足条件。

在真正打开前，文件仍可能被移动、替换或权限发生变化。

因此，实际文件操作仍要处理 OSError，不能拿模型验证替代真实打开的错误处理。

另外，路径存在也不等于路径属于你允许访问的目录。

访问范围与符号链接政策，需要结合真实应用单独设计。

### 22.8 HttpUrl 为什么不能直接当普通字符串拼接？

```python
from pydantic import BaseModel, HttpUrl


class Website(BaseModel):
    homepage: HttpUrl


site = Website(homepage="https://example.com")
print(str(site.homepage))
```

常见结果会带有规范化后的结尾斜杠：

```text
https://example.com/
```

v2 的 URL 类型不是普通 str 的简单同义词。

如果另一个库明确要字符串，使用 `str(site.homepage)`。

同时注意：合法 HTTP URL 不等于可信目标。

它不替你决定该不该访问内网地址、是不是授权范围内的域名，也不会自动请求这个网页验证它真的存在。

### 22.9 EmailStr 是内置可用吗？

EmailStr 属于常用扩展类型，但需要邮件地址验证相关的可选依赖。

如果项目需要它，按官方安装方式添加对应 extra，例如：

```bash
python -m pip install 'pydantic[email]>=2,<3'
```

本篇核心实践不依赖邮件验证扩展，因此不会要求你仅为了运行其余例子安装它。

邮件格式校验也不能证明邮箱归当前用户所有。

真正的所有权确认通常还要经过验证邮件等业务流程。

### 22.10 SecretStr 是不是加密？

不是。

```python
from pydantic import BaseModel, SecretStr


class Credentials(BaseModel):
    token: SecretStr


credentials = Credentials(token="example-token")
print(credentials.token)
print(credentials.token.get_secret_value())
```

典型输出：

```text
**********
example-token
```

遮蔽普通显示，有助于减少调试打印时的意外暴露。

但真正内容仍在内存中，可通过专门方法取得。

不要把这个类型描述成加密存储，也不要为了“看看是否验证成功”打印真实生产令牌。

这些常用类型的实际转换范围，还应结合[标准类型说明](https://docs.pydantic.dev/latest/api/standard_library_types/)和[网络类型说明](https://docs.pydantic.dev/latest/api/networks/)查看。

## 第二十三章：model_dump 与 model_dump_json，输出到底是什么？

### 23.1 验证成功，不代表已经得到 JSON 文本

先看完整模型：

```python
from datetime import date
from decimal import Decimal
from pathlib import Path
from pydantic import BaseModel


class ExportRecord(BaseModel):
    title: str
    published_on: date
    price: Decimal
    output: Path


record = ExportRecord(
    title="Python笔记",
    published_on="2026-11-03",
    price="19.90",
    output="reports/result.json",
)
```

现在的 `record` 是模型对象。

里面有 date、Decimal、Path 等 Python 对象。

要保存成 JSON，需要明确选择序列化入口。

### 23.2 默认 model_dump 返回 Python 字典

```python
payload = record.model_dump()
print(type(payload).__name__)
print(type(payload["published_on"]).__name__)
```

输出：

```text
dict
date
```

默认 Python 模式尽可能保留 Python 表示。

所以“返回 dict”不等于“其中所有值已经是 JSON 原生类型”。

直接交给标准库 `json.dumps(payload)`，可能因为 date 等对象而失败。

### 23.3 想得到能交给 JSON 编码器的字典呢？

```python
payload = record.model_dump(mode="json")
print(type(payload).__name__)
print(type(payload["published_on"]).__name__)
print(payload["price"])
```

通常输出：

```text
dict
str
19.90
```

`mode="json"` 让支持的类型按 JSON 兼容形式导出。

这里仍然是字典，尚未变成一整段 JSON 文本。

例如把数据交给 HTTPX 的 `json=` 参数时，这种输出常常合适。

HTTP 客户端负责最终编码，模型负责字段表现。

### 23.4 如果想直接得到 JSON 字符串？

```python
text = record.model_dump_json(indent=2)
print(type(text).__name__)
print(text)
```

第一行输出：

```text
str
```

后面是完整 JSON 文本。

这适合写文本文件，或者交给明确接受已编码 JSON 的接口。

不要看名称里带 JSON，就猜返回 bytes；BaseModel 的这个方法返回字符串。

稍后会看到 TypeAdapter 的同类输出接口返回 bytes，需要按具体 API 区分。

### 23.5 什么叫 JSON 被编码了两次？

假设你写：

```python
import json

text = record.model_dump_json()
encoded_again = json.dumps(text, ensure_ascii=False)
print(encoded_again[:30])
```

`text` 已经是 JSON 文本。

再次 `json.dumps(text)`，会把这整段文本当作一个普通字符串编码，出现额外引号和转义。

接收方解析一次后得到的可能是字符串，而不是原本希望的对象。

因此，要么用 `model_dump(mode="json")` 配合 JSON 编码器，要么直接使用 `model_dump_json()` 的结果。

先问“当前变量是字典，还是 JSON 文本”，能避免很多莫名其妙的转义问题。

### 23.6 `dict(record)` 为什么与 model_dump 不完全一样？

Python 的 `dict(model)` 可以利用模型提供的迭代行为取得字段。

但它不是完整的递归序列化接口。

嵌套模型可能仍保留模型对象；自定义序列化、过滤等也不能简单视为同一件事。

当目的就是按模型规则导出时，明确使用 `model_dump()`，阅读代码的人更容易判断行为。

### 23.7 如何保存中文 JSON？

```python
from pathlib import Path

Path("record.json").write_text(
    record.model_dump_json(indent=2),
    encoding="utf-8",
)
```

这段代码会写入或覆盖当前工作目录中的 `record.json`。

只在练习目录执行。

更可靠的正式配置写入方式，会在后面的完整实践中使用临时文件与替换。

注意，文件编码属于文件接口的职责；不是因为模型里有中文，就需要给每个字段单独指定 UTF-8。

### 23.8 序列化会再验证全部字段吗？

不要这样依赖。

如果你绕过验证或在创建后把字段改成错误类型，导出过程可能警告、失败，或者按当前值产生输出。

它不等价于把所有数据重新送回模型完整验证一遍。

需要重新确认契约时，使用清楚的验证入口，而不是指望 model_dump 自动修复模型。

## 第二十四章：导出过滤，未提供、等于默认值、值为 None 有什么区别？

### 24.1 先看同一个结果的两种来源

```python
from pydantic import BaseModel


class DisplayOptions(BaseModel):
    theme: str = "light"
    nickname: str | None = None
    enabled: bool = True


first = DisplayOptions()
second = DisplayOptions(theme="light", nickname=None, enabled=False)
```

`first` 什么都没提供。

`second` 显式提供了默认主题、空昵称，以及 False。

字段结果有重叠，但输入意图不同。

### 24.2 exclude_unset=True

```python
print(first.model_dump(exclude_unset=True))
print(second.model_dump(exclude_unset=True))
```

通常输出：

```text
{}
{'theme': 'light', 'nickname': None, 'enabled': False}
```

这个选项关注“字段是否被显式设置”，不是简单判断值是否为真。

因此，False、零、空字符串和 None 都可能需要保留。

做部分更新时，这种区别非常重要。

### 24.3 exclude_none=True

```python
print(second.model_dump(exclude_none=True))
```

输出中会去掉值为 None 的昵称。

但 False 仍然应该保留。

它关注当前值，不关注这次有没有显式提供。

如果 None 代表“主动清除”，更新时盲目 exclude_none 就会丢失这个操作意图。

### 24.4 exclude_defaults=True

```python
print(second.model_dump(exclude_defaults=True))
```

等于字段默认值的项会按规则被排除。

这与 exclude_unset 不同。

即使显式提供了 `theme="light"`，它仍然与默认值相等，因而可以被排除。

不要用“省略无用内容”这个模糊说法代替三个选项的具体语义。

### 24.5 include 与 exclude

```python
print(second.model_dump(include={"theme", "enabled"}))
print(second.model_dump(exclude={"nickname"}))
```

字段集合在这里表达输出范围。

适合明确导出一部分资料。

需要稳定的对外协议时，也可以定义单独输出模型，让输出允许什么字段更加直观。

如果只是不断往 exclude 里加秘密字段，新增字段时容易忘记同步政策。

### 24.6 嵌套字段怎么选？

假设模型有 `address` 子模型，里面有 city 与 street。

可以按层级描述选择范围：

```python
selected = user.model_dump(include={"name": True, "address": {"city"}})
```

这段代码假设 `user` 来自前面具有对应字段的嵌套模型示例。

`"address": {"city"}` 表示在 address 内进一步选择 city。

它不是把字符串 `"address.city"` 当成通用路径解析。

列表各元素的过滤还可以使用相应索引或 `__all__` 规则；只有实际需要时再学习，避免为几个字段写出比模型还难理解的过滤结构。

### 24.7 Field(exclude=True) 有什么用途？

```python
from pydantic import BaseModel, Field


class InternalAccount(BaseModel):
    username: str
    password_hash: str = Field(exclude=True)
```

正常导出时，该字段按排除规则处理。

但是模型里依然持有它，Python 代码仍可能访问。

这仍然不是加密，也不是限制 Python 调用者访问属性的权限系统。

敏感输出策略要同时关注序列化、日志、异常消息和自定义代码，而不能只看一个参数。

## 第二十五章：验证完成以后，模型还能被改坏吗？

### 25.1 默认赋值不一定重新验证

```python
from pydantic import BaseModel


class Counter(BaseModel):
    count: int


counter = Counter(count=3)
counter.count = "wrong"
print(type(counter.count).__name__)
```

默认设置下，可能输出：

```text
str
```

模型创建时通过了验证，不代表对象从此拥有不可破坏的运行期类型屏障。

这与普通 Python 动态对象的可修改性有关。

### 25.2 想在属性赋值时验证怎么办？

```python
from pydantic import BaseModel, ConfigDict


class CheckedCounter(BaseModel):
    model_config = ConfigDict(validate_assignment=True)
    count: int
```

现在执行不合法赋值会触发 ValidationError。

适合需要在对象存活期间保持字段规则的部分场景。

但不要因此就假定所有深层变化也都被拦截。

### 25.3 列表 append 会触发字段赋值验证吗？

看：

```python
from pydantic import BaseModel, ConfigDict, Field


class TagCollection(BaseModel):
    model_config = ConfigDict(validate_assignment=True)
    tags: list[str] = Field(default_factory=list)


collection = TagCollection()
collection.tags.append(123)
print(collection.tags)
```

列表原地变化没有经过 `collection.tags = ...` 这个字段替换操作。

因此，不能依靠 validate_assignment 为普通 list 的每个方法自动插入检查。

需要更稳定的数据结构时，可以考虑不可变容器、受控方法、重新构造模型等设计。

### 25.4 frozen=True 是深层冻结吗？

```python
from pydantic import BaseModel, ConfigDict, Field


class FrozenRecord(BaseModel):
    model_config = ConfigDict(frozen=True)
    name: str
    tags: list[str] = Field(default_factory=list)
```

它会限制通常的模型属性赋值。

但内层列表仍然可能被修改。

所以 frozen 更接近模型自身赋值层面的限制，而不是递归地把所有可达对象变成不可变。

如果你希望标签也不可原地增加，可以考虑 tuple[str, ...]，并理解其中元素本身是否仍然可变。

### 25.5 已经是模型实例，再 model_validate 一次就安全了吗？

不一定。

Pydantic 对已有模型实例有重验证政策。

默认情况下，传入同类型实例可能直接复用，不一定完整重新验证其内部当前值。

如果实例可能被修改，应了解 `revalidate_instances` 配置，并在正确的模型类型上设置。

```python
from pydantic import BaseModel, ConfigDict


class RecheckedCounter(BaseModel):
    model_config = ConfigDict(revalidate_instances="always")
    count: int
```

这个选项针对相应模型实例的验证处理，不是你在外层任意位置写上就能替所有不同子类统一开启。

### 25.6 model_copy(update=...) 会验证新值吗？

这是一个特别常见的误区。

```python
original = Counter(count=3)
changed = original.model_copy(update={"count": "not-a-number"})
print(type(changed.count).__name__)
```

更新数据不经过普通的完整字段验证。

因此，model_copy 适用于你已经信任并自行保证正确的数据复制场景，不能当成通用外部输入更新接口。

需要验证更新结果时，可以先导出原数据、合并经过明确政策选择的更新，再重新验证完整对象。

后面会写一个完整例子。

### 25.7 deep=True 又解决什么？

它主要影响复制深度。

浅复制中，某些嵌套对象仍然共享引用。

深复制会尝试复制嵌套对象，减少相应共享。

但“深复制”不等于“重新运行全部验证”，也不等于解决数据库事务或外部资源复制问题。

复制、校验和持久化是不同问题。

### 25.8 model_construct 是快速创建模型的常规入口吗？

它是绕过通常验证流程的受信任构造入口。

你可能在特殊性能路径、已经验证过的数据处理或低层代码中见到。

普通用户输入、HTTP 响应和配置文件，优先使用正常验证入口。

不要只因为名字里有 construct 就认为它更正式、更安全。

而且 v2 的正常验证已经有优化，绕过验证是否更快也需要测量，不能靠直觉决定牺牲保证。

## 第二十六章：TypeAdapter，没有一个完整对象时怎么办？

### 26.1 输入顶层就是列表

假设文件内容是：

```json
[1, "2", 3]
```

你希望验证它是整数列表，但不想为了它加一层：

```json
{"items": [1, "2", 3]}
```

这时可以使用 TypeAdapter。

### 26.2 最小用法

```python
from pydantic import TypeAdapter

adapter = TypeAdapter(list[int])
values = adapter.validate_python([1, "2", 3])
print(values)
```

输出：

```text
[1, 2, 3]
```

TypeAdapter 接收一个类型表达，为它提供验证和序列化等能力。

你不必先定义 BaseModel 子类。

### 26.3 JSON 入口

```python
values = adapter.validate_json('[1, "2", 3]')
print(values)
```

这里验证的是 JSON 文本对应的输入。

与 BaseModel 的两种入口一样，来源模式可能影响某些严格转换行为。

不要仅凭最终都是 Python 对象，就假定所有入口语义完全一致。

### 26.4 列表里的模型也可以吗？

```python
from pydantic import BaseModel, Field, TypeAdapter


class ImportedBook(BaseModel):
    id: int = Field(gt=0)
    title: str = Field(min_length=1)


books_adapter = TypeAdapter(list[ImportedBook])
books = books_adapter.validate_python([
    {"id": 1, "title": "Python"},
    {"id": "2", "title": "Linux"},
])
print(books[1].id)
print(type(books[1]).__name__)
```

输出：

```text
2
ImportedBook
```

错误位置也可以包含顶层列表索引，再进入模型字段。

这对批量导入和 HTTP 顶层数组很实用。

### 26.5 TypeAdapter.dump_json 返回什么？

```python
encoded = books_adapter.dump_json(books, indent=2)
print(type(encoded).__name__)
```

输出：

```text
bytes
```

它与 BaseModel.model_dump_json 返回 str 的约定不同。

需要写二进制文件时可以直接写 bytes；需要文本时明确 `.decode("utf-8")`。

不要通过名字相似就统一猜测返回类型。

### 26.6 为什么不要在每一行循环里重新创建适配器？

TypeAdapter 构建时需要分析类型表达并准备相应规则。

如果所有输入都使用同一契约，通常可以在循环外创建一次，重复使用它验证不同数据。

这既减少重复准备，也让数据协议集中可见。

当然，缓存与共享不等于把每次输入结果混在一起；适配器与验证结果对象是两回事。

### 26.7 想把顶层列表本身定义成一个有名称的模型呢？

可以了解 RootModel。

```python
from pydantic import RootModel


class BookIds(RootModel[list[int]]):
    pass


ids = BookIds.model_validate([1, "2", 3])
print(ids.root)
print(ids.model_dump_json())
```

通常输出：

```text
[1, 2, 3]
[1,2,3]
```

根值存放在 `.root`。

序列化不必为了 Python 属性名称就额外输出一个 root 包装键。

TypeAdapter 适合为任意类型表达提供适配能力；RootModel 适合确实希望有模型类身份和相应模型方法的根值结构。

选哪个取决于实际代码组织，不需要为了显得面向对象就把每个 list 都包装成类。

## 第二十七章：计算字段与自定义序列化，输入和输出可以不完全相同

### 27.1 订单总价为什么不必让用户填写？

如果数量与单价已经验证，总价可以计算。

让用户同时提交总价，还需要处理用户算错或故意篡改的情况。

```python
from decimal import Decimal
from pydantic import BaseModel, Field, computed_field


class OrderLine(BaseModel):
    unit_price: Decimal = Field(ge=0)
    quantity: int = Field(gt=0)

    @computed_field
    @property
    def total(self) -> Decimal:
        return self.unit_price * self.quantity


line = OrderLine(unit_price="12.50", quantity=2)
print(line.total)
print(line.model_dump(mode="json"))
```

通常输出：

```text
25.00
{'unit_price': '12.50', 'quantity': 2, 'total': '25.00'}
```

`property` 使它按属性读取。

`computed_field` 告诉模型序列化系统把这项计算结果纳入相应输出。

### 27.2 它是普通输入字段吗？

不要这样看。

输入和输出契约可能不同。

计算字段主要用于从模型状态产生输出信息，并不自动把外部提交的同名内容当成可信计算结果。

需要控制额外输入时，仍然明确 extra 政策。

如果使用缓存计算属性，还要理解对象修改后缓存是否失效；对初学实践，先采用简单 property 更容易保持含义清楚。

### 27.3 field_serializer 做什么？

它用于控制某字段导出时的表现。

例如标签内部保留列表，输出给一个旧工具时要求用逗号连接。

```python
from pydantic import BaseModel, field_serializer


class LegacyTags(BaseModel):
    tags: list[str]

    @field_serializer("tags", when_used="json")
    def tags_as_text(self, tags: list[str]) -> str:
        return ",".join(tags)


obj = LegacyTags(tags=["python", "linux"])
print(obj.model_dump())
print(obj.model_dump(mode="json"))
```

通常输出：

```text
{'tags': ['python', 'linux']}
{'tags': 'python,linux'}
```

`when_used="json"` 使这个规则在 JSON 模式等相应场景使用。

内部字段并没有因此自动永久改成字符串。

### 27.4 输出格式可逆吗？

不一定。

如果标签本身允许逗号，简单 join 后无法知道某个逗号来自分隔还是标签内部。

这个例子用于说明序列化机制，不代表通用标签协议就该如此设计。

自定义输出要考虑接收者能否无歧义解析，必要时保留标准 JSON 数组。

不能因为可以写 serializer，就无条件把结构化数据压成难以还原的字符串。

### 27.5 model_serializer 是什么时候需要？

当整个模型输出结构需要统一改变时，可以使用模型序列化器。

但它的影响范围很大。

如果只是一个字段格式变化，字段序列化器更局部；如果输入与输出本来就是两份独立协议，单独定义输出模型也可能更直观。

模型序列化器可以让 model_dump 产生与你原本想象不同的整体结构，因此调用代码不应盲目假定所有自定义模型都永远导出同一形状的 dict。

先明确接收方契约，再决定用哪种扩展工具。

### 27.6 校验器与序列化器的边界

校验器回答：“输入怎样变成合格的内部数据？”

序列化器回答：“内部数据怎样表现为输出？”

它们可以配合，但不应互相代替。

例如密码强度不能只在导出时检查；对外遮蔽也不能只靠输入字段验证。

把两个方向画清楚，会比记住装饰器名称更有帮助。

## 第二十八章：JSON Schema 是什么？能代替运行校验吗？

### 28.1 给模型生成一份结构说明

```python
from pydantic import BaseModel, Field


class PageContract(BaseModel):
    page: int = Field(ge=1, description="从一开始的页号")
    page_size: int = Field(default=20, ge=1, le=100)


schema = PageContract.model_json_schema()
print(schema["properties"]["page"]["minimum"])
print(schema["required"])
```

通常输出：

```text
1
['page']
```

得到的 schema 是 Python 字典，内容按照 JSON Schema 相关规范表达数据结构与部分约束。

它可以交给文档工具、表单系统或支持相应规范的验证器使用。

### 28.2 生成 Schema 会验证一份实际输入吗？

不会。

`model_json_schema()` 在描述规则。

`model_validate(...)` 在对具体输入执行验证。

你生成了一份“页号至少为一”的说明，不代表外部用户提交的零已经被检查过。

要验证，仍要把实际值送入验证流程。

### 28.3 自定义 Python 规则都能完整写进 Schema 吗？

不能这样保证。

例如开始日期不能晚于结束日期、列表内部编号不能重复，或者依赖 context 的动态允许分类，都可能无法直接从普通字段 Schema 完整体现。

因此，拿 JSON Schema 在另一个系统验证通过，不自动证明它执行了你的所有 Python 验证器。

当协议需要跨语言复用时，应明确哪些规则由 Schema 表达，哪些是附加业务约束，并分别测试。

### 28.4 validation 与 serialization 模式

输入允许的形状与输出形状可能不同。

例如 Decimal 的 JSON 表现、计算字段、自定义序列化都可能影响描述。

可以按用途生成：

```python
input_schema = OrderLine.model_json_schema(mode="validation")
output_schema = OrderLine.model_json_schema(mode="serialization")
print("total" in input_schema["properties"])
print("total" in output_schema["properties"])
```

在前面订单行模型中，通常分别为：

```text
False
True
```

这就是“输入契约”和“输出契约”需要区分的具体表现。

### 28.5 description、examples 和 default 是说明还是动作？

description 和 examples 主要是给使用者与工具看的元数据。

它们不会自动执行示例，也不会从自然语言推导业务程序。

Schema 中的 default 也不意味着所有下游 JSON Schema 验证器都会替你自动填值。

在 Pydantic 模型构造中使用默认值，是 Pydantic 自己的运行行为；导出的 Schema 被其他工具怎样解释，还要看那个工具。

### 28.6 $defs 与 $ref 是什么意思？

模型嵌套时，相同结构不必复制很多遍。

Schema 可以把可重用定义放在 `$defs` 中，再通过 `$ref` 引用。

它类似于“这里使用前面已定义的 Address 结构”，不是说程序需要访问一个叫 `$defs` 的真实目录。

因此，不要靠删除引用、手动拼字典就假定得到了等价 Schema；调整 Schema 需要尊重相应规范和下游工具能力。

更完整的规则见[官方 JSON Schema 说明](https://docs.pydantic.dev/latest/concepts/json_schema/)。

## 第二十九章：完整实践一，读取一份真正会出错的配置文件

### 29.1 先确定需求，而不是先写一个万能模型

你正在写一个把文章导出为 JSON 的命令行工具。

用户可以在配置文件中设置输出位置、每批处理数量，以及是否覆盖旧文件。

你希望遇到这些问题时尽早说明原因：

- 把 `batch_size` 拼成了 `batchsize`。
- 批量数量是零或过大。
- 输出文件名为空。
- 配置版本不是程序认识的版本。
- 文件内容根本不是 JSON。

这些问题分别发生在文件读取、格式解析和模型验证层。

下面把它们接起来，但不把所有异常压成一句“配置不合法”。

### 29.2 建立一个独立练习目录

```text
pydantic-config-demo/
├── config.json
└── config_demo.py
```

先创建 `config.json`：

```json
{
  "version": 1,
  "output": "reports/articles.json",
  "batch_size": 50,
  "overwrite": false
}
```

JSON 中布尔值写成小写 `false`。

不是 Python 的 `False`。

也不要在 JSON 里写注释或结尾多余逗号。

格式工具能否解析成功，与模型字段是否合格，是先后两个问题。

### 29.3 先写模型这一小段

把下面代码放入 `config_demo.py`：

```python
from pathlib import Path
from typing import Literal
from pydantic import BaseModel, ConfigDict, Field, field_validator


class ExportConfig(BaseModel):
    model_config = ConfigDict(extra="forbid", validate_default=True)

    version: Literal[1]
    output: Path
    batch_size: int = Field(default=50, ge=1, le=500, strict=True)
    overwrite: bool = Field(default=False, strict=True)

    @field_validator("version", mode="before")
    @classmethod
    def require_integer_version(cls, value):
        if type(value) is not int:
            raise ValueError("版本必须是真正的整数")
        return value

    @field_validator("output", mode="before")
    @classmethod
    def require_nonblank_path(cls, value):
        if isinstance(value, str) and not value.strip():
            raise ValueError("输出路径不能为空")
        return value
```

为什么版本写 `Literal[1]`？

因为这个程序只理解第一版结构，而不只是希望 version 是任意整数。

这里还通过 `before` 验证器要求原始值是精确的整数。

因此 `true`、`1.0` 与 `"1"` 都不会被当成合法版本号，而普通整数 `1` 才会继续进入 Literal 的固定值检查。

先检查输入类型，再检查允许的版本值，两项约定各有明确位置。

为什么 output 不直接写成 FilePath？

因为输出目标可能还不存在。

输入文件要求存在与输出文件允许新建，是不同场景。

为什么对数量和开关使用字段级 strict？

这份 JSON 配置规定数量应写数字，开关应写 JSON 布尔值。

你希望把 `"50"`、`"false"` 等错误书写直接报告出来。

同时，Path 仍需要从 JSON 字符串转换；没有为了几个字段的政策把整个模型设成不加区分的严格 Python 输入。

### 29.4 路径空字符串为什么要在 before 检查？

Path 接受某些字符串并形成路径对象。

空字符串可能被解释为当前目录形式，而不是自动代表“没有输出文件”。

因此，我们在原始字符串还在时检查空白。

但这里没有把所有字符串都 `.strip()` 后悄悄覆盖。

实际文件名是否允许前后空格，要由工具协议决定。

空白检查、规范化与禁止某些路径，是三份不同政策，不应混成一个含糊的“清理”。

### 29.5 再写读取函数

继续放在同一文件中：

```python
import json


class ConfigLoadError(ValueError):
    pass


def load_export_config(path: Path, max_bytes: int = 128 * 1024) -> ExportConfig:
    try:
        with path.open("rb") as stream:
            raw = stream.read(max_bytes + 1)
    except OSError as error:
        raise ConfigLoadError(f"无法读取配置文件：{path}") from error

    if len(raw) > max_bytes:
        raise ConfigLoadError("配置文件超过大小限制")

    try:
        text = raw.decode("utf-8")
        data = json.loads(text)
    except (UnicodeDecodeError, json.JSONDecodeError) as error:
        raise ConfigLoadError("配置文件不是有效的 UTF-8 JSON") from error

    return ExportConfig.model_validate(data)
```

`path.open("rb")` 先按字节读取。

`read(max_bytes + 1)` 最多多读一个字节，用于判断是否超过上限。

这里没有先 `stat()` 再无界 `read()`，避免把“检查时的大小”误当成之后读取的硬上限。

不过这只是限制读取内容量，并不是针对所有特殊设备或恶意文件系统的完整沙箱。

### 29.6 为什么 ConfigLoadError 与 ValidationError 分开？

ConfigLoadError 主要用于本例中的文件和格式问题。

ValidationError 则保留模型给出的结构化字段错误。

用户看到“文件无法读取”，应检查路径和权限。

看到 `batch_size` 的范围错误，应修改配置内容。

这比捕获所有 Exception 然后统一说“请重装 Pydantic”更有帮助。

`raise ... from error` 保留原因链，开发时仍能追踪底层错误。

对最终用户怎样显示消息，可以在命令行入口决定。

### 29.7 输出路径相对谁？

现在明确约定：相对配置文件所在目录。

```python
def resolve_config_output(config: ExportConfig, config_path: Path) -> Path:
    output = config.output.expanduser()
    if not output.is_absolute():
        output = config_path.resolve().parent / output
    return output
```

如果你的配置文件位于：

```text
/home/alice/project/config.json
```

其中写：

```text
reports/articles.json
```

那么得到的输出路径指向：

```text
/home/alice/project/reports/articles.json
```

即使你从 `/home/alice` 启动脚本，也不会突然把相对输出解释到另一个目录。

这里返回一个路径对象，仍然没有创建目录或写文件。

### 29.8 补上能运行的入口

```python
from pydantic import ValidationError


def main() -> int:
    config_path = Path(__file__).resolve().with_name("config.json")
    try:
        config = load_export_config(config_path)
    except ConfigLoadError as error:
        print(error)
        return 2
    except ValidationError as error:
        for item in error.errors(include_input=False, include_url=False):
            location = ".".join(str(part) for part in item["loc"]) or "<整体>"
            print(f"配置字段 {location}：{item['msg']}")
        return 2

    output = resolve_config_output(config, config_path)
    print(f"每批处理：{config.batch_size}")
    print(f"允许覆盖：{config.overwrite}")
    print(f"输出位置：{output}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

运行：

```bash
python config_demo.py
```

输出位置随你的练习目录变化，其余两行应类似：

```text
每批处理：50
允许覆盖：False
输出位置：你的练习目录/reports/articles.json
```

这个入口只显示最终配置，没有进行真正导出。

先确认读取、解释和错误反馈正确，再往后连接写文件操作，排查会更清楚。

### 29.9 把数量改成字符串会怎样？

改成：

```json
"batch_size": "50"
```

这个字段使用 strict=True，所以应拒绝，而不是悄悄转成整数。

如果你的工具是为了兼容旧配置，明确想接受字符串数量，可以删除这个字段级严格选项，并把接受规则写进文档和测试。

两种选择都可以有合理场景，关键是政策明确，不能一边说“严格数字”一边默默接受所有文本。

### 29.10 把键拼错会怎样？

改成：

```json
"batchsize": 50
```

extra="forbid" 会让这个陌生字段被报告。

同时，真正的 batch_size 缺失时可能使用默认值。

因此，拒绝陌生键特别有助于识别这种“看起来能运行，但设置根本没生效”的问题。

### 29.11 配置值验证完，文件操作就一定成功吗？

仍然不一定。

输出父目录可能不可写，磁盘可能满，目标可能在真正写入前发生变化。

Pydantic 负责数据契约，文件系统负责真实操作结果。

完整工具应在执行输出时继续处理相应异常，而不是把第一次模型验证当成后续所有动作的成功证明。

## 第三十章：环境变量和 settings，为什么 BaseModel 不会自动读整个系统？

### 30.1 环境变量通常是字符串

你可能设置：

```text
APP_BATCH_SIZE=30
APP_DEBUG=false
```

在普通环境变量读取中，得到的通常是字符串 `"30"` 与 `"false"`。

这和 JSON 里的数字与布尔值来源不同。

因此，环境变量入口经常需要一份明确的转换政策。

### 30.2 少量配置可以自己收集再验证

```python
import os
from pydantic import BaseModel, Field


class RuntimeOptions(BaseModel):
    batch_size: int = Field(default=20, ge=1, le=500)
    debug: bool = False


def load_runtime_options() -> RuntimeOptions:
    raw = {}
    if "APP_BATCH_SIZE" in os.environ:
        raw["batch_size"] = os.environ["APP_BATCH_SIZE"]
    if "APP_DEBUG" in os.environ:
        raw["debug"] = os.environ["APP_DEBUG"]
    return RuntimeOptions.model_validate(raw)
```

这里选择了允许 Pydantic 按相应规则解析字符串。

没有把缺失环境变量先写成 None。

因为缺失应让模型默认值生效；显式 None 是另一种输入。

### 30.3 `bool("false")` 为什么不合适？

普通 Python 的 bool 看的是对象真值。

非空字符串 `"false"` 的真值是 True。

因此，不应为了“转成布尔”就自己写：

```python
print(bool("false"))
```

输出：

```text
True
```

Pydantic 的 bool 验证有自己的接受范围，可以识别相应文本值，但你仍需要了解它允许哪些拼写。

如果你的协议只接受小写 true/false，就应该明确写更窄的解析函数或验证规则，而不是假定所有库的默认宽松策略恰好符合需求。

### 30.4 BaseSettings 为什么导入不到了？

v2 中，settings 管理相关能力位于独立的 `pydantic-settings` 包。

旧教程中的 `from pydantic import BaseSettings` 不能照搬成现代 v2 的标准写法。

如果项目确实需要多来源配置管理，可以单独安装：

```bash
python -m pip install pydantic-settings
```

然后按其当前版本文档使用。

本篇不把这个独立工具展开成另一份完整教材，也不把它作为运行前面模型示例的强制依赖。

### 30.5 什么时候需要它？

如果配置同时来自构造参数、环境变量、dotenv 文件和秘密目录，而且需要明确优先级、前缀、嵌套键和自定义来源，专门的 settings 工具会比手工散落读取更合适。

但“使用了 settings”不代表优先级可以靠猜。

哪个来源覆盖哪个来源、空字符串怎样处理、哪些文件会被读取，都要按选定版本确认。

尤其不要在排查配置时把全部环境变量直接打印出来，里面可能含有凭据。

对 settings 的边界与当前接口，可查[官方 settings 说明](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)。

### 30.6 为什么不让一个模型同时兼容所有来源？

配置文件可能要求 JSON 数字，环境变量却天然是文本。

为了兼容两者而全局变得非常宽松，可能让用户文件里的拼写错误被悄悄转换。

一个常见选择是：分别做来源解析，再汇合到清楚的内部配置模型。

也可以统一采用宽松模型，但必须把转换范围作为正式协议。

不要为了减少一两个类，丢掉你真正想要的输入语义。

## 第三十一章：完整实践二，HTTP 成功返回之后继续检查业务数据

### 31.1 HTTP 200 不等于内容合格

某个接口返回：

```json
{
  "items": [
    {"id": 1, "title": "Python笔记", "price": "19.90"}
  ],
  "next_page": null
}
```

你希望编号是正整数，标题不是空白，价格非负。

HTTPX 可以负责发送、状态和传输。

Pydantic 负责返回内容是否符合你的业务结构。

这两层各自失败时，要能区分原因。

### 31.2 定义响应模型

```python
from decimal import Decimal
from typing import Annotated
from pydantic import BaseModel, ConfigDict, Field, StringConstraints, model_validator

NonblankTitle = Annotated[
    str,
    StringConstraints(strip_whitespace=True, min_length=1, max_length=200),
]


class RemoteBook(BaseModel):
    model_config = ConfigDict(extra="ignore")
    id: int = Field(gt=0, strict=True)
    title: NonblankTitle
    price: Decimal = Field(ge=0, max_digits=10, decimal_places=2)


class BookPage(BaseModel):
    items: list[RemoteBook] = Field(max_length=100)
    next_page: int | None = Field(default=None, ge=1)

    @model_validator(mode="after")
    def require_unique_ids(self):
        ids = [book.id for book in self.items]
        if len(ids) != len(set(ids)):
            raise ValueError("同一页中出现重复书籍编号")
        return self
```

RemoteBook 明确忽略额外字段，是因为这个读取场景允许服务端增加我们暂时不用的信息。

如果协议必须完全固定，可以改成 forbid。

别把配置文件的陌生键政策机械复制到所有第三方响应；两种来源的演进需求可能不同。

### 31.3 为什么重复编号用模型验证器？

每个 id 单独看都可能合法。

问题在于多个条目的组合。

所以用模型级关系检查表达更自然。

这里检查的是“同一页”，没有声称保证所有分页之间都不重复。

跨页去重需要在分页采集逻辑中维护已见编号，或者根据真实接口协议决定是否允许重复。

### 31.4 写一个可注入客户端的小函数

```python
import httpx


def fetch_book_page(client: httpx.Client, page: int) -> BookPage:
    response = client.get("/books", params={"page": page})
    response.raise_for_status()
    return BookPage.model_validate(response.json())
```

为什么把 client 当参数，而不是函数里每次新建？

调用者可以复用连接、统一设置超时，也可以在测试中替换传输层。

函数只做这一个数据入口的工作。

`response.json()` 负责 JSON 解析。

`BookPage.model_validate(...)` 负责模型契约。

如果你改成直接 `model_validate_json(response.content)`，则走 JSON 验证来源路径，前面严格模式中讲过，两种来源对部分类型的规则可能不同。

### 31.5 不访问公网，也能演练

```python
import httpx


def book_handler(request: httpx.Request) -> httpx.Response:
    if request.url.path != "/books":
        return httpx.Response(404, json={"error": "not found"})
    return httpx.Response(200, json={
        "items": [{"id": 1, "title": "  Python笔记  ", "price": "19.90"}],
        "next_page": None,
    })


transport = httpx.MockTransport(book_handler)
with httpx.Client(transport=transport, base_url="https://books.example") as client:
    page = fetch_book_page(client, 1)
    print(page.items[0].title)
    print(type(page.items[0].price).__name__)
```

输出：

```text
Python笔记
Decimal
```

这里的 `.example` 域名由 MockTransport 在进程内处理，不进行真实公网请求。

它验证的是我们构造的请求与响应处理逻辑，不是证明某个真实网站可用。

关于 Client 与 MockTransport 的细节，可结合 [HTTPX 教程](Python-HTTPX库.md) 阅读。

### 31.6 如果服务端把 title 变成空格呢？

把返回值改成：

```json
{"id": 1, "title": "   ", "price": "19.90"}
```

清理空白后长度为零，应该触发 ValidationError。

这是实际有意义的拒绝测试。

如果只测试一条格式完美的记录，很难知道你的清理与长度约束是否真的按希望配合。

### 31.7 如果状态码是 500 呢？

`raise_for_status()` 应先抛出 HTTPStatusError，后面的模型验证不会继续。

你不应该把服务器错误响应的 `{"error": ...}` 强行解释成 BookPage，再只告诉用户缺少 items。

正确顺序让错误含义更贴近真实原因。

### 31.8 如果返回的是 HTML 呢？

JSON 解析会失败。

模型还没开始处理，不能把这种错误说成“price 类型不正确”。

实际客户端可以在业务入口把传输错误、HTTP 状态错误、JSON 格式错误、模型契约错误分别转换成适合调用者的领域错误。

转换时保留异常链；不要为了让控制台安静就返回一个空列表伪装成功。

### 31.9 已验证模型怎样发给另一个 JSON 接口？

使用：

```python
payload = page.model_dump(mode="json")
```

再交给接收 Python 对象并负责 JSON 编码的 `json=` 参数。

不要把 `page.model_dump_json()` 得到的字符串直接作为 `json=` 值，否则整段 JSON 文本可能再次作为字符串编码。

如果确实要发送已经编码好的 JSON 文本，应按 HTTP 客户端的 content 等接口与 Content-Type 约定操作。

### 31.10 类型合法不代表业务动作已经得到授权

即使模型里有 `user_id=1`，也只说明它符合你声明的编号格式。

当前调用者是否有权操作这个用户，仍然需要业务授权判断。

数据验证、身份认证和权限控制会配合，但不能互相取代。

## 第三十二章：完整实践三，部分更新时保留 None、False 和零的含义

### 32.1 先看最容易写错的一行

你可能写：

```python
changes = {key: value for key, value in raw.items() if value}
```

这会同时丢掉 None、False、零和空字符串。

但它们在更新协议里可能都有明确含义。

是否更新字段，不能简单用真值来决定。

### 32.2 原始完整模型

```python
from pydantic import BaseModel, ConfigDict, Field


class AccountState(BaseModel):
    model_config = ConfigDict(extra="forbid")
    nickname: str | None = None
    enabled: bool = True
    quota: int = Field(default=10, ge=0, strict=True)
```

这里零配额是合法值。

None 昵称表示已经清空昵称。

enabled=False 表示主动关闭。

### 32.3 更新模型只描述这次允许提交什么

```python
class AccountPatch(BaseModel):
    model_config = ConfigDict(extra="forbid")
    nickname: str | None = None
    enabled: bool | None = Field(default=None, strict=True)
    quota: int | None = Field(default=None, ge=0, strict=True)
```

这个版本允许更新入口收到 None。

但对 enabled、quota，完整状态模型不允许 None。

因此，合并后必须再次用完整模型确认状态。

这是一个明确的两阶段设计：更新模型负责输入形状与允许字段，完整模型负责最终状态的全部约束。

如果希望更早拒绝某字段的显式 None，也可以在更新模型添加对应验证器；规则必须与文档约定一致。

### 32.4 合并时只使用本次显式提供字段

```python
def apply_account_patch(current: AccountState, raw_patch: dict) -> AccountState:
    patch = AccountPatch.model_validate(raw_patch)
    changes = patch.model_dump(exclude_unset=True)
    merged = current.model_dump()
    merged.update(changes)
    return AccountState.model_validate(merged)
```

`exclude_unset=True` 保留显式 None，也保留 False 和零。

`merged.update(changes)` 只改变这次提交的字段。

最后重新验证完整结果，避免使用不验证更新值的 model_copy(update=...)。

### 32.5 运行一次真的更新

```python
current = AccountState(nickname="小林", enabled=True, quota=10)
updated = apply_account_patch(current, {
    "nickname": None,
    "enabled": False,
    "quota": 0,
})
print(updated.model_dump())
print(current.model_dump())
```

通常输出：

```text
{'nickname': None, 'enabled': False, 'quota': 0}
{'nickname': '小林', 'enabled': True, 'quota': 10}
```

这个例子构造新状态，原模型保持原值。

这方便调用者在确认成功之前保留旧状态，也方便比较前后变化。

### 32.6 空字典表示什么？

```python
unchanged = apply_account_patch(current, {})
print(unchanged.model_dump() == current.model_dump())
```

输出：

```text
True
```

没提供任何字段，所以不进行字段更新。

这与显式 `{"nickname": None}` 不同，后者会清空昵称。

### 32.7 显式 quota=None 呢？

最终 AccountState 不允许 None，所以完整结果验证应失败。

程序没有因为更新模型允许 None，就把最终状态悄悄变成不符合内部约定的对象。

设计更新协议时，最好逐字段写清楚“省略”“null”“空字符串”“零”的含义，而不是用一个 Optional 词试图覆盖所有语义。

### 32.8 这解决并发更新了吗？

没有。

这个函数只处理内存中的字段合并与校验。

如果多个调用者同时修改数据库，同样可能出现最后写入覆盖前一份修改的问题。

版本号、条件更新、事务或乐观锁等需要在持久化层设计。

不要把“合并后验证成功”当作并发一致性的证明。

## 第三十三章：批量导入，整批拒绝还是逐条报告？

### 33.1 两种需求都很常见

一个配置清单中任何一条不合法，整个任务就不应启动。

一份人工整理的 CSV 中，可能希望保留合法行，同时导出错误清单让人修正。

这两种需求不能用同一个“跳过错误就行”随意决定。

使用 TypeAdapter(list[Model]) 可以按整个集合验证。

逐条调用 Model.model_validate 则方便把每一条输入与错误绑定。

### 33.2 先为每条输入保留来源位置

```python
from pydantic import BaseModel, Field


class ImportRow(BaseModel):
    id: int = Field(gt=0)
    title: str = Field(min_length=1)
    quantity: int = Field(ge=0)
```

本例故意把数量字符串转换视为允许行为，因为 CSV 常常读出字符串。

这和前面 JSON 配置中的严格数量不同。

来源不同，契约应明确设计。

### 33.3 逐条验证函数

```python
from pydantic import ValidationError


def validate_rows(rows):
    accepted = []
    rejected = []
    seen_ids = set()

    for row_number, raw in enumerate(rows, start=2):
        try:
            item = ImportRow.model_validate(raw)
        except ValidationError as error:
            rejected.append({
                "row": row_number,
                "errors": [
                    {"loc": list(detail["loc"]), "type": detail["type"]}
                    for detail in error.errors(include_input=False, include_url=False)
                ],
            })
            continue

        if item.id in seen_ids:
            rejected.append({"row": row_number, "errors": [
                {"loc": ["id"], "type": "duplicate_id"},
            ]})
            continue

        seen_ids.add(item.id)
        accepted.append(item)

    return accepted, rejected
```

为什么 start=2？

本例约定 CSV 第一行是表头，第一条记录从第二行开始。

准确地说，这个计数把表头占用的一项算进去，第一条数据记录编号为二。

它是带表头偏移的**逻辑记录序号**，不能普遍当作文件物理行号。

CSV 中一个带引号的字段可以包含换行，一条记录就可能占据多个物理行。

例如，第一条数据的标题占两行，第二条数据实际从文件第四行开始，本函数仍然给第二条数据编号为三。

如果你需要物理行定位，应在 CSV 读取层同时保存 `reader.line_num` 等信息，并说明它表示读完当前记录时的物理行位置。

不能只靠 `enumerate(..., start=2)` 恢复带多行字段的准确物理行号。

如果输入来自 JSON 数组，这个来源定位方式就应改成数组索引，而不是生搬硬套“第二行”。

### 33.4 验证结果不必包含全部原始数据

错误报告只保留上面约定的记录序号、字段位置和错误类型。

这既便于程序处理，也减少把敏感原始输入复制进日志的机会。

如果确实需要错误原文，应该按业务数据保留政策另外保存，不能自动把所有输入都打印到共享控制台。

本例中的 duplicate_id 是我们为业务重复问题定义的报告类型，不要误认为它一定是 Pydantic 内置错误名称。

### 33.5 运行一次混合输入

```python
rows = [
    {"id": "1", "title": "Python", "quantity": "3"},
    {"id": "2", "title": "Linux", "quantity": "-1"},
    {"id": "1", "title": "重复记录", "quantity": "0"},
]
accepted, rejected = validate_rows(rows)
print([item.id for item in accepted])
print([item["row"] for item in rejected])
```

输出：

```text
[1]
[3, 4]
```

第二条记录数量为负，被字段规则拒绝。

第三条记录单独字段都合法，但编号与已接受记录重复，被集合层面的业务规则拒绝。

### 33.6 为什么 seen_ids 放在成功后才更新？

本例政策是：只有已接受记录占用编号。

某条记录字段失败，不阻止后续修正后的同编号记录进入。

如果你希望“原文件出现过相同编号就整批拒绝”，应采用不同的预扫描或集合规则。

代码中的集合更新位置，其实就在表达导入政策。

这比看到 set 就只理解成“提高速度”更深入一步。

### 33.7 CSV 读取怎样接进来？

```python
import csv
from pathlib import Path


def validate_csv(path: Path):
    with path.open("r", encoding="utf-8", newline="") as stream:
        reader = csv.DictReader(stream)
        return validate_rows(reader)
```

这段函数复用上面逐条验证入口。

但是 CSV 语法错误、编码错误、重复表头、缺失表头、额外列等，还需要结合你实际 CSV 协议处理。

Pydantic 不会因为输入来自 DictReader，就自动知道第一行表头是否符合你的文件格式要求。

请结合 [CSV 教程](Python-csv库.md) 区分文件语法层与记录契约层。

### 33.8 “逐条报告”是不是就不应该抛异常？

不应把所有异常吞掉。

本例只把预期的字段 ValidationError 转成错误记录。

程序编写错误、读取失败等仍应暴露或在合适层处理。

否则一个验证器拼错变量名产生 NameError，你可能把整份文件全部当成“用户填错”，排查方向就完全错了。

## 第三十四章：如何为模型写真正有意义的检查？

### 34.1 不只看创建对象没有报错

如果只验证一份完美输入，你无法确认负数会被拒绝、未知键会被发现、None 与省略有没有区别。

把契约拆成可观察问题，测试才有价值。

例如标题清理应该先去两侧空白，再拒绝空白标题。

至少要有正常标题、带空白标题、全空白标题三个方向。

不是为了增加测试数量，而是因为它们触发不同规则路径。

### 34.2 用普通断言先检查核心行为

下面片段沿用 RemoteBook：

```python
from pydantic import ValidationError

book = RemoteBook(id=1, title="  Python  ", price="0.00")
assert book.title == "Python"
assert book.price == 0

try:
    RemoteBook(id=1, title="   ", price="0.00")
except ValidationError as error:
    assert error.errors()[0]["loc"] == ("title",)
else:
    raise AssertionError("全空白标题应该被拒绝")
```

测试代码里的 assert 用于验证程序行为。

前面提醒“不用 assert 承担必须执行的业务校验”，与这里并不矛盾。

正式运行时依赖的业务规则必须稳定执行；测试运行器本来就把断言当作检查工具。

### 34.3 边界值为什么比随便挑几个数字更有意义？

数量下限是一，测试一应通过、零应失败。

上限是五百，测试五百应通过、五百零一应失败。

只测十、二十、三十，可能完全碰不到 `ge` 与 `gt` 写反的问题。

对时间范围，同一天是否允许也需要明确：start==end 是通过还是拒绝？

测试应从业务契约推导，而不是照着实现里的每个 if 复制一份。

### 34.4 错误消息应该整段精确匹配吗？

通常优先验证稳定含义，比如字段位置和错误类型。

完整英文错误字符串可能因版本、文案和格式细节改变。

你真正希望保证的，往往是“错误被定位到 quantity，而不是代码崩溃”以及“零不被接受”。

如果你有对外承诺的中文错误文案，则可以针对自己的错误转换层测试精确输出。

不要把第三方库内部文案误当作你必须永久复制的业务接口。

### 34.5 哪些危险入口值得特别检查？

model_copy(update=...) 不校验、plain 验证器可以绕过字段类型、默认值默认可能不验证、原地修改容器不触发字段赋值验证。

这些都容易让读者以为拥有更强保证。

测试这些边界，可以防止未来重构时再次把“模型实例”误认为“任何时刻都绝对合法”的对象。

下一份 [pytest 教程](Python-pytest库.md) 会进一步解释如何组织这些检查、夹具和失败报告。

## 第三十五章：练习，先判断数据协议，再写代码

### 35.1 练习一，昵称究竟可不可以省略？

写两个模型。

第一个要求调用者必须提供 nickname，但允许其值为 None。

第二个允许完全不提供 nickname，缺失时使用 None。

分别验证空字典与 `{"nickname": None}`，解释差异。

参考答案：

```python
from pydantic import BaseModel


class RequiredNickname(BaseModel):
    nickname: str | None


class DefaultNickname(BaseModel):
    nickname: str | None = None
```

空字典不能构造第一个模型，因为没有默认值。

第二个模型可以。

显式 None 对两个模型都符合字段类型。

这道题检查的是“可空”与“可缺失”两种维度，而不是 Optional 单词翻译。

### 35.2 练习二，价格不能静悄悄四舍五入

设计 PriceInput，价格使用 Decimal，必须非负且最多两位小数。

输入 `"0.00"`、`"12.34"` 应通过，`"-0.01"`、`"12.345"` 应拒绝。

参考答案：

```python
from decimal import Decimal
from pydantic import BaseModel, Field


class PriceInput(BaseModel):
    amount: Decimal = Field(ge=0, max_digits=10, decimal_places=2)
```

然后分别构造四次输入。

不要只写模型而不运行负例。

如果你另外加了舍入验证器，可能已经改变题目要求；这道题要求拒绝额外小数，而不是把它修正成另一个值。

### 35.3 练习三，逗号文本和列表兼容

某个旧文件把标签存成 `"python, linux"`，新文件存成 `["python", "linux"]`。

要求两种形式都转换为清理后的字符串列表。

全空白项应该拒绝，不要悄悄删除。

参考答案：

```python
from typing import Annotated
from pydantic import BaseModel, StringConstraints, field_validator

TagText = Annotated[str, StringConstraints(strip_whitespace=True, min_length=1)]


class TagInput(BaseModel):
    tags: list[TagText]

    @field_validator("tags", mode="before")
    @classmethod
    def split_legacy_text(cls, value):
        if isinstance(value, str):
            return value.split(",")
        return value
```

输入 `"python, linux"` 得到两项。

输入 `"python,,linux"` 会产生一个空项，随后被 TagText 规则拒绝。

这里 before 只负责兼容容器外形，元素清理交给元素类型规则，职责比较清楚。

### 35.4 练习四，两种日期入口为什么不一样？

给模型设置 strict=True，字段是 date。

分别调用 model_validate 传入日期字符串字典，以及 model_validate_json 传入同样日期的 JSON。

观察两种结果，解释原因。

参考模型：

```python
from datetime import date
from pydantic import BaseModel, ConfigDict


class StrictDateInput(BaseModel):
    model_config = ConfigDict(strict=True)
    day: date
```

Python 输入路径的字符串不是 date 实例，严格验证会拒绝。

JSON 本身没有 date 对象，严格 JSON 模式对日期等类型仍允许相应文本表示。

所以严格不是一句“绝不转换”能完整定义的；类型和输入来源都需要考虑。

### 35.5 练习五，订单行编号不能重复

定义订单包含多个 OrderItem。

每个 item_id 为正整数，quantity 大于零。

整个订单内 item_id 不允许重复。

参考答案：

```python
from pydantic import BaseModel, Field, model_validator


class OrderItem(BaseModel):
    item_id: int = Field(gt=0)
    quantity: int = Field(gt=0)


class OrderInput(BaseModel):
    items: list[OrderItem] = Field(min_length=1)

    @model_validator(mode="after")
    def no_duplicate_item_ids(self):
        ids = [item.item_id for item in self.items]
        if len(ids) != len(set(ids)):
            raise ValueError("订单中不允许重复商品编号")
        return self
```

单个商品编号合法，不代表组合没有重复。

这道练习要求把字段规则、容器规则与模型整体规则分别放在合适位置。

若业务其实允许同一商品多行，你就不该照搬本规则，而应说明合并数量或保留明细的政策。

### 35.6 练习六，导出时保留显式清空

当前昵称是“小林”。

用户提交 `{"nickname": None}`，表示清除昵称。

你要从更新模型导出本次提交内容，应该用 exclude_unset 还是 exclude_none？

参考答案是 exclude_unset=True。

它会保留这次显式提供的 None。

exclude_none=True 会把这个值删掉，使“清除昵称”的意图丢失。

再用空字典与 enabled=False 试一次，确认没有把省略和假值混起来。

### 35.7 练习七，Schema 没写出的规则怎么办？

给 DateRange 生成 JSON Schema。

观察 start 与 end 的字段说明。

问自己：下游只拿到这份 Schema，是否一定知道 start 不得晚于 end？

不能这样保证。

Python 模型验证器里执行的关系规则，不一定完整转换成通用 Schema 约束。

如果你要把这份协议交给另一个语言实现，需额外说明关系并准备共享样例，分别验证两端行为。

### 35.8 练习八，修复一个“看起来安全”的复制

代码是：

```python
original = CheckedCounter(count=3)
copy = original.model_copy(update={"count": "wrong"})
```

为什么 validate_assignment=True 不能自动保证这次 update 被校验？

因为 model_copy 的 update 路径不是普通受验证字段赋值的替代接口，更新内容按受信任数据处理。

一种修复是：

```python
merged = original.model_dump()
merged.update({"count": "wrong"})
checked = CheckedCounter.model_validate(merged)
```

这次应该在正常验证入口拒绝错误。

不要把 deep=True 加上去当成答案；深拷贝解决共享问题，不解决更新内容验证问题。

### 35.9 练习九，为什么错误行不占用编号？

回看 validate_rows。

如果第二行 id=1 但 quantity=-1，第三行 id=1 且 quantity=2，本例是否接受第三行？

按当前政策，可以接受。

因为第一条失败记录没有加入 seen_ids。

如果实际需求是“原始文件任何重复都不允许”，需要修改算法及对应测试，而不是说其中一种行为天生正确。

这道题训练你从语句位置看出隐藏的业务政策。

### 35.10 练习十，给 HTTP 模型加失败样例

为 fetch_book_page 至少构造四种 MockTransport 返回值：正常页、500 状态、200 但不是 JSON、200 JSON 中 price 为负。

预期分别是正常 BookPage、HTTP 状态错误、JSON 解析错误、模型验证错误。

如果四种情况最后都返回空列表，你的调用者就失去了判断真实结果的能力。

这道练习的重点不是背异常名，而是保持错误发生层次和业务含义。

## 第三十六章：实际使用中，你可能还会追问的问题

### 36.1 我应该给项目里每个字典都套一个模型吗？

没有必要。

Pydantic 特别适合稳定的数据边界：外部输入、配置、导入记录、接口契约、需要统一错误反馈的数据结构。

函数内部临时统计计数的小字典，可能用清楚的普通 Python 就够了。

模型的价值来自明确契约，而不是类的数量。

### 36.2 已经有 dataclass，还需要 BaseModel 吗？

标准 dataclass 主要帮助组织类字段、生成构造等方法，本身不因为类型注解就自动进行完整运行期验证。

Pydantic 也提供数据类相关工具，但它与直接继承 BaseModel 的方法集合和使用方式并不完全相同。

选择时先问：是否需要解析外部输入、结构化错误、序列化和 Schema？

不要把“都能写 name: str”当成行为完全一样的证据。

### 36.3 TypedDict 是不是更轻量的运行期验证模型？

普通 TypedDict 主要描述字典形状，属于静态类型工具。

它不会单独在每次创建 dict 时检查键和值。

你可以借助 TypeAdapter 等工具对相应类型表达执行验证，但这是额外库提供的行为。

“类型描述”与“谁解释并执行这个描述”始终要分别看。

### 36.4 自定义普通类字段，为什么报无法生成 Schema？

Pydantic 不会自动知道所有任意 Python 类怎样解析、验证和序列化。

可以选择把它转换成受支持的数据结构、定义明确的自定义集成，或在受控模型中使用 arbitrary_types_allowed 等配置。

但允许任意类型时，往往只是接受相应实例，不意味着库已经深入验证其全部内部状态或知道怎样导出 JSON。

不要为了消除错误就无条件开启选项，然后以为仍然获得了相同保证。

### 36.5 `from_attributes=True` 是把对象所有属性都读进来吗？

它允许按字段声明从对象属性等相应来源取得数据。

例如把一个普通对象转换成模型。

但读取属性可能触发 property 逻辑，某些对象还可能延迟加载外部数据。

使用前需要知道输入对象是什么、字段访问是否有副作用，而不是把它当作无条件安全的深度复制工具。

对简单字典输入，不必为了“功能更强”先开启属性来源处理。

### 36.6 可以把验证器写成 async 吗？

不要期待普通 model_validate 会自动 await 你写的异步函数。

常规验证流程以同步、确定性的数据处理为主要使用方式。

需要远程检查或数据库查询时，把异步操作放在明确业务流程中，取得必要事实后再做模型或业务判断。

把网络调用藏进验证器会使调用者难以判断耗时、资源与错误来源。

### 36.7 为什么输入数字不会总自动转成字符串？

不要把“宽松转换”理解成任意类型之间都随意转换。

每个类型有自己的接受表和配置。

例如你不能仅因为整数能解析数字字符串，就反推所有字符串字段都会自动对任意对象调用 str。

如果协议允许数字转文本，应明确配置或自定义规则，并测试是否会接收本来想拒绝的数据。

不确定时查看[官方转换表](https://docs.pydantic.dev/latest/concepts/conversion_table/)，再用最小输入验证当前版本。

### 36.8 ValidationError 能不能直接给最终用户看？

学习阶段直接打印很方便。

正式工具通常需要把技术字段路径转换成用户能理解的信息，并控制原始输入暴露。

同时，不要把所有细节删到只剩“输入有误”。

保留来源位置、字段、错误类别和可操作说明，用户才知道下一步改哪里。

### 36.9 版本升级最应该检查什么？

检查项目是否锁定 v2 的具体范围，是否依赖默认转换、别名配置、联合选择、错误细节和自定义验证器顺序。

先运行与真实协议相关的边界测试，再看弃用警告与官方迁移说明。

不要因为一个简单 Student 示例还能创建，就证明所有业务契约在新版本下完全一致。

### 36.10 性能优化应该从哪里开始？

先确认工作量：重复构建 TypeAdapter、在验证器里做慢 I/O、无必要地多次解析同一份 JSON，可能比某个字段约束本身更值得改。

针对真实输入测量，再决定是否复用适配器、减少重复转换或调整边界。

不要在没有测量时先改成 model_construct，放弃验证可能让后续错误代价更大。

性能与正确性的权衡应明确可验证，而不是从函数名猜谁更快。

### 36.11 我需要把所有校验都放在模型里吗？

字段格式和对象内确定关系适合集中在模型。

依赖身份、数据库当前状态、外部权限或事务时机的判断，通常属于应用业务流程。

模型与业务函数合作，才能形成完整行为。

把所有事情塞进一个 BaseModel，不会自动使系统更可靠，也会让简单数据读取触发难以察觉的外部动作。

### 36.12 这篇文档的验证范围是什么？

本文以 Pydantic 2.14.2 的实际行为核对关键入口与边界，Python 示例采用兼容 3.11 的常用语法。

本地检查覆盖必填与 None、默认值校验、严格 Python/JSON 来源差异、嵌套错误、验证器、别名与额外字段、序列化、复制与赋值边界、TypeAdapter，以及本地配置和模拟 HTTP 实践。

依赖真实网络、可选邮件验证扩展、独立 pydantic-settings 包及外部数据库的操作，没有冒充为已经完成的公网或生产验证。

需要查参数细节时，优先结合[BaseModel API](https://docs.pydantic.dev/latest/api/base_model/)、[字段](https://docs.pydantic.dev/latest/concepts/fields/)、[验证器](https://docs.pydantic.dev/latest/concepts/validators/)和[序列化](https://docs.pydantic.dev/latest/concepts/serialization/)说明阅读。

阅读官方文档时也先确认版本；旧问题里的 v1 答案不一定适用于这里采用的 v2。
