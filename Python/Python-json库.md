# Python `json` 库超详细教程：从文本与对象的区别，到文件保存、接口数据和结构校验

## 零基础先导：JSON、Python 字典与 JSON 文件是三件事

**JSON** 是 JavaScript Object Notation 的缩写，如今主要作为跨语言的数据交换格式。它既不是 Python 专用语法，也不是一个自动保存数据的容器。“把字典变成 JSON 字符串”和“把 JSON 字符串解析成字典”是两个**方向相反**的操作。

```python
import json

student = {"name": "小林", "active": True}      # Python 对象：布尔值是 True
text = json.dumps(student, ensure_ascii=False) # 生成 JSON 字符串
print(text)                                     # {"name": "小林", "active": true}
again = json.loads(text)                        # JSON 字符串解析回 Python 对象
print(again["active"])                          # True
```

逐句看：`dumps` 把数据**序列化为字符串**；`loads` 从字符串**解析出对象**；`dump`/`load` 少一个 `s`，常配合文件对象，直接写入/读取文件。`ensure_ascii=False` 只是让非 ASCII 字符更直观地出现，写文件时仍然建议明确 `encoding="utf-8"`。

**不要拿 `str(dict)` 当 JSON**：Python 可能使用单引号、`True`、`None`，这些与 JSON 的规则不同。JSON 字符串中的数字不等于 Python 输入文件中的一切业务类型；日期对象、自定义类也不会默认神奇地自动序列化。网络请求库的 `response.json()` 也只是在“解析收到的正文”，并不意味着服务器给你的是安全可信的业务数据。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

适合已经掌握字典、列表、字符串和文件读写基础，正在接触配置文件、API 或数据交换的读者。

你很可能见过：

```python
import json

text = json.dumps({"name": "小林", "age": 20}, ensure_ascii=False)
data = json.loads(text)
```

这两行究竟做了什么？`text` 和 `data` 看起来内容差不多，为什么一个要用 `dumps()`，一个要用 `loads()`？`load()` 又为什么少一个 `s`？把字典保存成字符串之后，是否就能叫 JSON？

这一篇从数据表示的基础讲起，再逐步说明参数、返回值、异常、常见组合，以及完整的文件处理实践。代码以 Python 3.11 及以上为主要基础环境，标准库 `json` 不需要额外安装。

## 第一章：JSON 是什么？为什么项目中经常出现它？

### 1.1 同一份信息，可能有不同语言的表示

Python 可以这样表示一个学生：

```python
student = {
    "name": "小林",
    "age": 20,
    "active": True,
    "tags": ["Python", "backend"],
    "graduated_at": None,
}
```

但另一个程序可能用 JavaScript、Java 或 Go 编写。它不能直接使用当前 Python 进程里的字典对象，因为语言的类型、内存布局和运行环境不同。

你需要一种双方都理解的交换格式。JSON 就是常用选择之一。

它把有限几种通用数据结构写成有明确规则的文本。发送方按规则生成文本，接收方按规则解释文本，再转换成自己语言中的对象。

### 1.2 名称的由来

JSON 是 JavaScript Object Notation 的缩写，可以理解为源自 JavaScript 对象表示法的一种数据格式。

虽然名称带有 JavaScript，JSON 现在是与具体编程语言无关的交换格式。它只描述数据，不等于一段应当执行的 JavaScript 程序。[格式背景：RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)

### 1.3 JSON 与 Python 字典不是同一个东西

下面是 JSON 文本：

```json
{
  "name": "小林",
  "age": 20,
  "active": true,
  "tags": ["Python", "backend"],
  "graduated_at": null
}
```

`true`、`false`、`null` 是 JSON 的字面量。Python 中对应的是 `True`、`False`、`None`。

JSON 对象的键必须是字符串；字符串使用双引号。Python 字典的键规则更丰富，Python 字符串字面量也可以使用单引号。这些差异意味着不能把任意 Python 对象的打印结果直接当成 JSON。

### 1.4 为什么不直接用 `str(student)`？

```python
student = {"name": "小林", "active": True}
print(str(student))
```

通常会看到类似：

```text
{'name': '小林', 'active': True}
```

它是 Python 的对象显示形式，不是符合 JSON 规则的输出。

```python
import json

print(json.dumps(student, ensure_ascii=False))
```

才会生成相应 JSON 文本：

```json
{"name": "小林", "active": true}
```

不要用字符串替换把所有单引号改双引号、把 `True` 改成 `true`。字段内容本身可能含有引号或这些单词，手工替换会破坏数据。

## 第二章：三层数据一定要分清楚

### 2.1 Python 对象、JSON 字符串、网络或文件字节

```text
Python 对象：dict、list、str、int 等
    │ json.dumps()
    ▼
JSON 文本：Python str
    │ encode("utf-8") 或文本文件写入
    ▼
字节：bytes、磁盘内容或网络消息
```

反方向：

```text
字节
    │ 按对应编码解码
    ▼
JSON 文本 str
    │ json.loads()
    ▼
Python 对象
```

这些步骤在某些高层接口里可以连起来自动做，但概念上仍然不同。

### 2.2 最小实验

```python
import json

obj = {"message": "你好"}
text = json.dumps(obj, ensure_ascii=False)
payload = text.encode("utf-8")

print(type(obj))
print(type(text))
print(type(payload))

restored = json.loads(payload)
print(restored)
```

类型分别是字典、字符串和字节。`json.loads()` 可以接受 `str`，也可以接受符合其支持编码规则的 `bytes` 或 `bytearray`。

虽然它可以直接解析字节输入，HTTP、文件和协议层仍然可能有自己的编码约定。不要因此认为所有编码的任意文本字节都能直接成功解析。

### 2.3 看到花括号，不代表变量是字典

```python
import json

text = '{"count": 3}'
print(type(text))
```

`text` 是字符串。字符串的第一个字符虽然是 `{`，它仍然不是字典。

```python
obj = json.loads(text)
print(obj["count"])
```

解析之后才能按对象结构访问。

反过来，变量如果已经是字典，再调用 `json.loads(obj)` 会因输入类型不符合要求而报错。遇到问题先用 `type()` 确认当前层次，不要见到 JSON 就反复调用 `loads()`。

## 第三章：JSON 支持什么数据？

### 3.1 类型映射表

| JSON 类型 | 示例 | 默认解析到 Python | 常见 Python 生成来源 |
| --- | --- | --- | --- |
| object | `{"name":"小林"}` | `dict` | `dict` |
| array | `[1,2,3]` | `list` | `list`、`tuple` |
| string | `"hello"` | `str` | `str` |
| number，整数形式 | `123` | `int` | `int` |
| number，带小数或指数 | `1.5`、`1e3` | `float` | `float` |
| true / false | `true` | `True` / `False` | `bool` |
| null | `null` | `None` | `None` |

JSON 没有直接对应 Python 集合、元组类型标签、日期对象、路径对象、字节串或自定义类实例的独立类型。

### 3.2 顶层不一定是对象

以下都是合法 JSON 值的例子：

```json
[1, 2, 3]
```

```json
"hello"
```

```json
42
```

```json
null
```

因此：

```python
import json

result = json.loads("null")
print(result is None)
```

可以得到 `True`。解析成功不保证你拿到的一定是字典，后续应验证期望顶层类型。

### 3.3 元组会变成数组，再读回来是列表

```python
import json

original = {"position": (10, 20)}
text = json.dumps(original)
restored = json.loads(text)

print(restored)
print(type(restored["position"]))
```

`position` 恢复后是列表。JSON 保存的是通用数组，不保留“这是 Python tuple”的类型信息。

所以，`loads(dumps(obj))` 不保证对所有 Python 对象保持原类型和身份。这是一种格式转换，不是任意对象的完整克隆。

### 3.4 字典键最好主动限定为字符串

Python `json` 对部分非字符串基本键有转换支持，例如整数键可能变成字符串：

```python
import json

text = json.dumps({1: "one"})
print(text)
print(json.loads(text))
```

恢复后的键是 `"1"`，不是整数 `1`。

如果原字典同时具有整数 `1` 和字符串 `"1"` 等可能转换后冲突的键，交换语义会更加混乱。设计用于 JSON 的数据结构时，提前使用唯一字符串键，是最容易理解的约定。

## 第四章：`dumps`、`loads`、`dump`、`load`，到底怎样分清？

这四个名字只差几个字母，刚开始很容易混。

我们先不背表格，从两件具体的事出发。

第一件事：把内存中的 Python 数据，变成一段符合 JSON 规则的文本。

第二件事：把已经得到的 JSON 文本，解析成 Python 数据。

然后再考虑文本是放在变量里，还是来自文件流。

### 4.1 我已经有一个字典，怎样得到 JSON 字符串？

假设你有：

```python
student = {
    "name": "小林",
    "age": 20,
    "active": True,
}
```

这里的 `student` 是 Python 字典。

现在想生成一段可以保存或发送的 JSON 文本：

```python
import json

text = json.dumps(student, ensure_ascii=False)

print(text)
print(type(text))
```

通常输出：

```text
{"name": "小林", "age": 20, "active": true}
<class 'str'>
```

方向是：

```text
Python 字典
    │ json.dumps()
    ▼
JSON 文本，保存在 Python str 中
```

`dump` 可以理解为把数据导出成一种表示。

这里末尾的 `s` 提示 string，也就是返回字符串的接口。

#### 字典中的 `True` 怎么变成小写 `true` 了？

因为 JSON 与 Python 使用不同的布尔字面量。

编码器不是把整个字典用 `str()` 打印一遍，而是按 JSON 格式规则生成内容。

同样，Python 的 `None` 会写成 JSON 的 `null`，列表会写成数组。

#### `text` 已经是 JSON，为什么类型还是 str？

“JSON”说明这段文本遵守什么格式。

“str”说明它在当前 Python 程序中用什么对象保存。

就像一个字符串可以装电话号码，也可以装 SQL 或 HTML；它们的文本含义不同，但 Python 层面都可能由 `str` 保存。

所以，不存在“只要看见花括号，变量就自动变成字典”这回事。

#### 运行这行之后，磁盘上会有 JSON 文件吗？

还没有。

```python
text = json.dumps(student, ensure_ascii=False)
```

只是生成一个内存字符串。

你没有提供输出路径，也没有打开文件，编码器自然不知道要保存到哪个位置。

如果想写文件，稍后看 `dump()`，或者把这个字符串交给普通文本写入。

### 4.2 我收到一段 JSON 字符串，怎样重新取出字段？

假设现在拿到：

```python
text = '{"name":"小林","age":20,"active":true}'
```

注意：外层单引号属于 Python 字符串字面量。

字符串内容内部仍然是 JSON 要求的双引号和小写 `true`。

解析：

```python
import json

student = json.loads(text)

print(type(student))
print(student["name"])
print(student["age"] + 1)
print(student["active"])
```

通常得到：

```text
<class 'dict'>
小林
21
True
```

方向反过来：

```text
JSON 文本 str
    │ json.loads()
    ▼
Python 字典、列表或其他对应值
```

`load` 可以理解为把某种表示加载回程序数据。

末尾 `s` 提醒你，这里处理的是文本输入接口。

#### 为什么不能直接 `text["name"]`？

因为 `text` 仍然是字符串。

字符串索引按照字符位置工作，不会自行把内部花括号解释成字典结构。

你必须先解析，才得到按字段访问的数据对象。

#### 如果我已经有字典，再 `loads()` 一次呢？

```python
# 预期错误：student 已经是字典，不是待解析 JSON 文本。
json.loads({"name": "小林"})
```

通常会因输入类型不符合接口要求而报错。

这不是解析器不认识中文，而是你已经跨过了解析步骤，又把解析后的对象当作原始文本传入。

遇到混淆时，先打印：

```python
print(type(value))
```

确认目前拿到的是字符串、字节、字典还是响应对象，再决定下一步。

### 4.3 `dumps()` 和 `loads()` 能不能来回用？

对于普通 JSON 兼容结构，可以这样观察：

```python
import json

original = {"name": "小林", "scores": [95, 88], "active": True}
text = json.dumps(original, ensure_ascii=False)
restored = json.loads(text)

print(original == restored)
print(original is restored)
```

通常输出：

```text
True
False
```

数据值可以相等，但重新解析会创建另一组 Python 对象。

它没有保留原字典的对象身份。

#### 那是不是任意对象都能这样复制？

不能这样推广。

元组可能变成列表，非字符串字典键可能变成字符串，自定义类和日期对象默认还可能无法编码。

这里的往返遵守 JSON 类型规则，不是任意 Python 对象的完整克隆协议。

### 4.4 如果我想直接保存到文件呢？

这时可以使用少一个 `s` 的 `dump()`。

```python
import json
from pathlib import Path

student = {"name": "小林", "age": 20, "active": True}
path = Path("json_practice") / "student.json"
path.parent.mkdir(parents=True, exist_ok=True)

with path.open("w", encoding="utf-8", newline="\n") as target:
    json.dump(student, target, ensure_ascii=False, indent=2)
    target.write("\n")
```

我们逐步看职责。

```python
path = Path("json_practice") / "student.json"
```

构造输出地址。

```python
path.parent.mkdir(parents=True, exist_ok=True)
```

准备父目录。

```text
with path.open("w", encoding="utf-8", newline="\n") as target:
```

真正打开文本文件，并安排离开时关闭。

`w` 会覆盖已有练习文件，这个行为来自文件打开模式。

```python
json.dump(student, target, ensure_ascii=False, indent=2)
```

把对象编码成 JSON 文本，写入目标流。

`indent=2` 让布局更适合人工阅读。

#### 为什么 `dump()` 不直接收文件名？

它接收可写流对象，使你不仅可以写磁盘文件，也可以写内存文本流或其他符合写入接口的对象。

文件路径定位与打开，由调用者负责。

结构化格式生成，由 JSON 编码器负责。

这种分工让接口更容易与标准输入输出、临时文件和其他工具组合。

#### `json.dump(student, "student.json")` 可以吗？

不能把普通路径字符串直接当可写流使用。

字符串没有你需要的文本写入方法。

你应先打开，传入 `target`，或者先 `dumps()` 得到文本再写文件。

### 4.5 `dump()` 返回的是刚才写出的 JSON 字符串吗？

通常不是，返回值是 `None`。

```python
import json
from io import StringIO

with StringIO() as target:
    result = json.dump({"count": 3}, target)
    text = target.getvalue()

print(result)
print(text)
```

输出：

```text
None
{"count": 3}
```

想得到字符串，用 `dumps()`。

想把数据写进指定流，用 `dump()`。

不要因为函数已经写出了内容，就以为它还会把同样内容作为返回值交回来。

### 4.6 已经保存的 JSON 文件，怎样读回？

用 `load()`：

```python
import json
from pathlib import Path

path = Path("json_practice") / "student.json"

with path.open("r", encoding="utf-8") as source:
    student = json.load(source)

print(student["name"])
```

这里依赖前面已经创建好的练习文件。

`source` 是可读流，`load()` 从它取得内容并解析。

解析得到的字典不依赖文件继续保持打开，所以离开 `with` 后仍能访问字段。

#### 为什么 `json.load("student.json")` 不行？

`load()` 需要能够读取的流。

字符串路径只是地址，不是已经打开的文件。

它通常会在尝试读取接口时失败。

#### 那我把文件名交给 `loads()` 呢？

```python
json.loads("student.json")
```

这次解析器会把这几个字符当成 JSON 文本内容。

但 `student.json` 本身不是一份合法 JSON 文档，所以通常报告语法错误。

它不会因为字符串末尾是 `.json` 就自动去磁盘读取。

### 4.7 四个函数的输入和结果，一起整理

| 你现在有什么 | 你想得到什么 | 常用接口 |
| --- | --- | --- |
| Python 对象 | JSON 字符串 | `json.dumps(obj)` |
| JSON 字符串或支持的字节输入 | Python 对象 | `json.loads(text)` |
| Python 对象和可写文本流 | 将 JSON 写进流 | `json.dump(obj, target)` |
| 含 JSON 的可读流 | Python 对象 | `json.load(source)` |

先看转换方向，再看是否已经有流。

这比只凭哪个函数名字更熟悉来选择，更容易保持清楚。

### 4.8 最后再接一个容易问的问题：网络响应是不是也用 `loads()`？

如果你手里是一个 JSON 字符串，可以 `json.loads(text)`。

如果你使用 Requests 或 HTTPX，响应对象通常提供 `.json()`，由客户端按它的响应处理接口解析。

不要把整个 Response 对象直接交给 `json.loads()`，也不要把已经 `.json()` 得到的字典再解析一次。

始终确认当前变量处于哪一层，文件和 HTTP 场景的选择就能连起来理解。


## 第五章：`dumps()` 常用参数逐个解释

### 5.1 基本调用形式

```python
json.dumps(
    obj,
    ensure_ascii=True,
    indent=None,
    separators=None,
    sort_keys=False,
    allow_nan=True,
    skipkeys=False,
    default=None,
    check_circular=True,
)
```

这里列出主要常用参数，很多参数应使用关键字形式。并不是每次调用都要把它们写全。

### 5.2 `ensure_ascii`：控制非 ASCII 字符的表现

```python
import json

obj = {"message": "你好"}

print(json.dumps(obj))
print(json.dumps(obj, ensure_ascii=False))
```

默认输出可能把中文写为 `\u....` 转义；第二个调用保留中文的可读形式。

两种 JSON 文本解析后可以得到相同字符串。转义形式不是乱码，它是 JSON 允许的字符表达方式。

`ensure_ascii=False` 也不等于“输出编码设为 UTF-8”。`dumps()` 返回的是 `str`，真正变成字节时还要由文件或网络层编码。

### 5.3 `indent`：让结构更适合人阅读

```python
print(json.dumps({"name": "小林", "scores": [95, 88]}, ensure_ascii=False, indent=2))
```

`indent=2` 用两格作为每级缩进。它影响输出布局，不改变字段含义。

`indent` 也可以使用字符串，例如 `"\t"`。`indent=0`、负数或空字符串与完全紧凑单行之间还有换行布局区别，普通教材优先掌握正整数即可。

配置文件和手工检查报告通常使用缩进；带宽敏感的接口或逐行记录通常选择紧凑表示。

### 5.4 `separators`：控制逗号和冒号后的空白

```python
compact = json.dumps(
    {"a": 1, "b": [2, 3]},
    separators=(",", ":"),
)
print(compact)
```

输出类似：

```json
{"a":1,"b":[2,3]}
```

元组第一项是元素分隔符，第二项是键值分隔符。删除无意义空格能缩短输出，但不会压缩字符串字段内部的空格。

不要使用 `text.replace(" ", "")` 进行所谓 JSON 压缩，否则会把 `"Alice Smith"` 变成 `"AliceSmith"`。

### 5.5 `sort_keys`：按键排序输出

```python
text = json.dumps({"z": 1, "a": 2}, sort_keys=True)
print(text)
```

适合可读差异比较和部分测试场景。Python 字典保留插入顺序，但不同生成流程可能以不同顺序插入字段，排序可以减少这类布局差异。

`sort_keys=True` 不等于实现了完整的 JSON 规范化算法。数字表示、Unicode、转义等问题仍然存在，数字签名不能仅凭这个参数就自称跨语言规范一致。

### 5.6 `allow_nan`：默认行为与严格交换需求

Python 浮点可以表示 `nan` 和正负无穷。但严格 JSON 数字语法不允许这些特殊值。

```python
import json

print(json.dumps({"value": float("nan")}))
```

标准库默认可能输出 `NaN`。若要生成更严格的交换数据：

```python
json.dumps({"value": float("nan")}, allow_nan=False)
```

会抛出 `ValueError`。

接口双方若要求严格 JSON，建议明确拒绝这些值，或按业务规则用 `null`、错误状态等方式表达。不要在不了解接收方行为的情况下任意输出特殊数字。[兼容性依据：标准遵循与互操作](https://docs.python.org/3.14/library/json.html#standard-compliance-and-interoperability)

### 5.7 `skipkeys`：跳过非法键会丢数据

```python
import json

obj = {("x", "y"): "point", "name": "demo"}
```

默认编码会因元组键不符合支持范围而失败。`skipkeys=True` 会跳过相应键值项。

这并不是“自动正确修复”。如果那个字段重要，跳过就意味着数据丢失。日常配置和业务数据通常应保留默认报错，修正数据结构，而不是让问题静默消失。

### 5.8 `check_circular`：防止循环结构无法表示

```python
items = []
items.append(items)
```

这个列表包含自己。JSON 树形结构不能直接表达这样的对象引用循环。

默认循环检查会在序列化时报告问题。关闭检查不意味着 JSON 获得循环引用能力，而可能让递归不断进行，最终触发递归错误或更严重问题。

正常项目应把准备交换的数据整理成明确的无环结构，不要为了避开异常随意关闭检查。

## 第六章：`loads()` 与解析异常

### 6.1 最基本解析

```python
import json

text = '{"name":"小林","scores":[95,88],"active":true}'
student = json.loads(text)

print(student["name"])
print(student["scores"][0])
print(student["active"])
```

外层 Python 字符串使用单引号，是为了让内部 JSON 双引号更容易直接书写。JSON 文本自身仍然使用双引号。

`student["scores"][0]` 先从字典取出分数列表，再访问列表第一个元素。这是 Python 容器语法，不是 `json` 提供的特殊查询语言。

### 6.2 `JSONDecodeError` 带位置资料

```python
import json

text = '{"name": "小林",}'

try:
    student = json.loads(text)
except json.JSONDecodeError as error:
    print("原因：", error.msg)
    print("行号：", error.lineno)
    print("列号：", error.colno)
    print("字符位置：", error.pos)
```

尾随逗号不符合普通 JSON 语法，因此解析失败。

`lineno`、`colno` 便于人工定位，`pos` 是文档中的位置索引信息。不要把它直接等同于 UTF-8 原文件的字节偏移，中文等多字节内容会让字符位置与字节位置不同。

### 6.3 解析常见失败原因

| 错误写法 | 原因 | 改进方向 |
| --- | --- | --- |
| `{'name':'小林'}` | JSON 不用单引号字符串 | 使用真正 JSON 编码器生成 |
| `{"active": True}` | Python 布尔字面量 | 使用 `true` |
| `{"value": None}` | Python 空值字面量 | 使用 `null` |
| `{"a":1,}` | 尾随逗号 | 删除多余逗号 |
| `// comment` | 标准 JSON 无注释语法 | 配置需求另选支持注释的格式 |
| `{"a":1}{"b":2}` | 不是单一完整 JSON 文档 | 使用数组或明确定义记录协议 |

### 6.4 空白可以存在，额外文档不行

```python
json.loads('  {"a": 1}\n')
```

外围合法空白通常可以接受。

但是：

```python
json.loads('{"a": 1} trailing text')
```

会因为额外内容失败。`loads()` 预期解析一个完整文档，而不是只随意找第一段看起来像 JSON 的文本。

### 6.5 解析不是执行 Python 代码

不要用 `eval()` 读取外部 JSON。

JSON 有自己的语法和数据范围；`eval()` 可以执行 Python 表达式，含义和风险都完全不同。即使样例看起来只是字典，来自外部的文本也不应该交给 `eval()` 当作数据解析。

`ast.literal_eval()` 用于有限 Python 字面量解析，仍不是 JSON 的替代品，例如它不按 JSON 的 `true`、`null` 规则工作。格式已经约定 JSON，就使用 JSON 解析器。

## 第七章：文件读写的完整搭配

### 7.1 保存 UTF-8 JSON 文件

```python
import json
from pathlib import Path

settings = {
    "theme": "dark",
    "language": "中文",
    "page_size": 20,
}

path = Path("json_practice") / "settings.json"
path.parent.mkdir(parents=True, exist_ok=True)

with path.open("w", encoding="utf-8", newline="\n") as target:
    json.dump(settings, target, ensure_ascii=False, indent=2, allow_nan=False)
    target.write("\n")
```

`Path` 构造位置并创建父目录；`open()` 负责文本流；`json.dump()` 负责结构化编码。

末尾额外写一个换行，是为了文本工具阅读和版本差异显示方便。这个换行不是 JSON 值的一部分，不会改变下一次解析结果。

### 7.2 读取时分别处理不同问题

```python
import json
from pathlib import Path

path = Path("json_practice/settings.json")

try:
    with path.open(encoding="utf-8") as source:
        settings = json.load(source)
except FileNotFoundError:
    print("配置文件不存在")
except UnicodeDecodeError:
    print("文件内容不能按 UTF-8 解码")
except json.JSONDecodeError as error:
    print(f"JSON 格式错误：第 {error.lineno} 行，第 {error.colno} 列")
except OSError as error:
    print("其他文件访问错误：", error)
else:
    print(settings)
```

这些异常描述不同层次。把它们全部归为“JSON 格式错误”，会误导用户修改一份其实只是没有权限读取的文件。

### 7.3 `load()` 不意味着流式逐项解析巨型文档

虽然接口接收文件对象，标准库 `json.load()` 仍然会读取并解析完整文档，不是一个逐条产出数组元素的流式解析器。

数 GiB 的顶层数组可能同时带来文本存储和 Python 对象存储开销。需要处理大型记录集合时，可以重新选择按行 JSON 协议、数据库或专门的增量解析工具。

### 7.4 BOM 与内容编码

如果你的文本流用普通 UTF-8 解码保留了开头 BOM，`json.loads()` 对包含该字符的字符串可能拒绝。

来源明确是带 UTF-8 BOM 的文件时，可以用：

```python
with open("from_tool.json", encoding="utf-8-sig") as source:
    obj = json.load(source)
```

最好在格式约定中统一普通 UTF-8，减少工具之间的兼容分歧。编码错误和 JSON 语法错误仍然需要分别诊断。

## 第八章：解析成功之后，为什么还必须检查结构？

### 8.1 合法 JSON 可以装着错误业务数据

```json
{
  "name": 123,
  "age": "twenty",
  "active": "false",
  "scores": "95,88"
}
```

这份文本完全可能符合 JSON 语法。解析器负责确认双引号、冒号、括号和字面量等规则，不知道你的应用要求姓名必须是字符串、年龄必须是整数。

因此，不能把“`json.loads()` 没报错”写成“数据已经验证通过”。

### 8.2 先检查顶层类型

```python
import json

obj = json.loads('{"name":"小林","age":20}')

if not isinstance(obj, dict):
    raise ValueError("顶层必须是 JSON 对象")
```

如果直接写 `obj["name"]`，而顶层实际上是列表，就会出现与预期不符的错误。先检查结构，能给调用者更清楚的反馈。

### 8.3 必填字段与可选字段

```python
if "name" not in obj:
    raise ValueError("缺少 name 字段")

name = obj["name"]
city = obj.get("city", "未知")
```

`obj["name"]` 适合已经确认存在的必填字段；`get()` 可以提供缺失时默认值。

但需要注意：字段缺失和字段值为 `null` 不一样。

```python
first = {}
second = {"city": None}

print(first.get("city", "未知"))
print(second.get("city", "未知"))
```

前者返回默认值，后者返回 `None`。`get()` 不会把所有空值都替换为默认值。

### 8.4 缺失、空值、空字符串和零，分别表达什么？

| 数据 | 含义需要怎样约定 |
| --- | --- |
| 没有这个键 | 未提供、使用默认、保持旧值等 |
| `"field": null` | 明确为空、清除旧值或未知等 |
| `"field": ""` | 提供了空字符串 |
| `"field": 0` | 提供了数值零 |
| `"field": false` | 提供了布尔假 |

这些含义属于业务契约。不能简单使用 `value or default` 把它们都归为一类，否则合法的零和布尔假可能被覆盖。

### 8.5 `bool` 与 `int` 的关系会影响校验

在 Python 中：

```python
print(isinstance(True, int))
```

结果是 `True`，因为 `bool` 是 `int` 的子类。

如果 JSON 中的 `age` 必须是整数而不接受布尔值，简单 `isinstance(age, int)` 不够严格。可以使用：

```python
if type(age) is not int:
    raise ValueError("age 必须是整数，不能是布尔值")
```

或者明确排除 `bool`。对于刚解析的普通 JSON 基本类型，精确类型检查常能清楚表达约定；对于一般 Python 类层次，则要根据是否允许子类选择策略。

### 8.6 一个完整结构校验函数

```python
import math


def validate_student(obj):
    if not isinstance(obj, dict):
        raise ValueError("顶层必须是对象")

    allowed = {"name", "age", "active", "scores"}
    unknown = set(obj) - allowed
    if unknown:
        raise ValueError(f"不认识的字段：{sorted(unknown)}")

    required = {"name", "age"}
    missing = required - set(obj)
    if missing:
        raise ValueError(f"缺少字段：{sorted(missing)}")

    name = obj["name"]
    age = obj["age"]
    active = obj.get("active", True)
    scores = obj.get("scores", [])

    if not isinstance(name, str) or not name.strip():
        raise ValueError("name 必须是非空白字符串")
    if type(age) is not int or not 0 <= age <= 150:
        raise ValueError("age 必须是 0 到 150 之间的整数")
    if type(active) is not bool:
        raise ValueError("active 必须是布尔值")
    if not isinstance(scores, list):
        raise ValueError("scores 必须是数组")

    normalized_scores = []
    for index, score in enumerate(scores):
        if type(score) not in (int, float):
            raise ValueError(f"scores[{index}] 必须是数值")
        if isinstance(score, float) and not math.isfinite(score):
            raise ValueError(f"scores[{index}] 不能是非有限数值")
        if not 0 <= score <= 100:
            raise ValueError(f"scores[{index}] 超出允许范围")
        normalized_scores.append(score)

    return {
        "name": name.strip(),
        "age": age,
        "active": active,
        "scores": normalized_scores,
    }
```

### 8.7 为什么返回一份整理后的数据？

函数既完成校验，也明确输出需要的字段与默认值。这样后续业务不必反复判断某个可选字段是否缺失。

但“校验”和“规范化”应在说明中写清楚。例如这里对姓名使用 `strip()`，意味着两端空白不属于正式姓名；如果业务要求保留原始输入，就应另外保存原始数据，不要悄悄丢弃。

当模型数量变多、嵌套结构复杂时，可以使用 Pydantic 等第三方验证工具。工具能帮助组织规则，但不能替你决定规则本身。

## 第九章：嵌套字典、数组与逐层访问

### 9.1 不要把字段路径当成普通点号属性

假设解析结果是：

```python
obj = {
    "user": {
        "profile": {"city": "上海"},
        "roles": ["reader", "editor"],
    }
}
```

访问城市：

```python
city = obj["user"]["profile"]["city"]
```

不是 `obj.user.profile.city`。标准库 JSON 默认返回字典和列表，并没有把每个键变成 Python 对象属性。

### 9.2 每一层都可能不符合预期

如果 `profile` 是 `null`，继续 `["city"]` 就会失败；如果 `user` 缺失，第一步就是 `KeyError`。

少量字段可逐层检查：

```python
user = obj.get("user")
if not isinstance(user, dict):
    raise ValueError("user 必须是对象")

profile = user.get("profile")
if not isinstance(profile, dict):
    raise ValueError("user.profile 必须是对象")

city = profile.get("city")
if not isinstance(city, str):
    raise ValueError("user.profile.city 必须是字符串")
```

看起来比一行链式访问长，但它把每个失败点说清楚。这是数据入口校验经常值得写详细的地方。

### 9.3 不建议盲目连写 `.get(..., {})`

```python
city = obj.get("user", {}).get("profile", {}).get("city")
```

它只在缺失键时提供 `{}`。如果现有键值是 `None`、字符串或列表，下一次 `.get()` 仍可能失败。

这种写法适用于你已经掌握部分结构保证的场景，不能代替普遍的嵌套结构验证。

### 9.4 遍历对象与遍历数组不同

```python
for key, value in obj.items():
    print(key, value)
```

用于字典的键值对。

```python
for index, role in enumerate(obj["user"]["roles"]):
    print(index, role)
```

用于列表的索引和元素。

遍历字典时直接 `for item in obj` 得到键；遍历列表直接得到元素。很多所谓“JSON 遍历错误”，实际是没有分清解析结果的容器类型。

## 第十章：自定义编码——日期、路径、集合和类实例怎么办？

### 10.1 默认支持范围之外会报 `TypeError`

```python
import json
from datetime import datetime, timezone

obj = {"created_at": datetime.now(timezone.utc)}
json.dumps(obj)
```

`datetime` 对象没有 JSON 原生类型，默认编码器不能自行决定应该转成什么。

这其实有利于暴露设计问题。日期可以编码为 ISO 字符串，也可以编码为秒时间戳，两者语义和精度不同，程序不应擅自替你选择。

### 10.2 用 `default` 明确转换规则

```python
import json
from datetime import date, datetime
from pathlib import Path


def json_default(obj):
    if isinstance(obj, (datetime, date)):
        return obj.isoformat()
    if isinstance(obj, Path):
        return str(obj)
    if isinstance(obj, set):
        return sorted(obj)
    raise TypeError(f"不支持的类型：{type(obj).__name__}")


obj = {
    "birthday": date(2006, 5, 20),
    "file": Path("data/report.txt"),
    "tags": {"python", "backend"},
}

print(json.dumps(obj, default=json_default, ensure_ascii=False, indent=2))
```

集合排序在这里要求元素彼此可比较，示例都是字符串。混合不兼容类型的集合不能无条件 `sorted()`，应设计更明确的数据结构。

### 10.3 `default` 不是对所有值都调用

它主要处理编码器原本不认识的对象。普通 `str`、`int`、`list`、`dict` 等会按编码器自身规则处理，不会自动先交给 `default`。

因此，不能用这个回调假设所有字段都经过同一套过滤、脱敏或数值转换。需要处理所有字段时，应先显式遍历并生成用于输出的数据模型。

### 10.4 为什么回调最后要抛异常？

如果遇到不认识的类型却什么都不返回，Python 函数默认返回 `None`，编码器可能把它变成 JSON 的 `null`。

这样一个原本不支持的复杂对象就会悄悄丢失内容。明确 `raise TypeError(...)` 能保留失败信息，让调用者决定如何修正。

### 10.5 不要随手使用 `default=str`

```python
json.dumps(obj, default=str)
```

它很方便，但也可能把不该出现的对象转换成难以还原的显示文本。

例如集合变成 `"{'a', 'b'}"`，不是 JSON 数组；某个对象可能输出包含内存地址的表示；日期格式也未必符合双方约定。

用于临时诊断可以明确接受这种损失；用于正式交换协议，最好针对允许类型写清楚转换规则，拒绝其他对象。

### 10.6 为类实例生成明确字典

```python
class Student:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def to_dict(self):
        return {"name": self.name, "age": self.age}
```

```python
import json

student = Student("小林", 20)
text = json.dumps(student.to_dict(), ensure_ascii=False)
```

显式挑选字段的优点是避免把缓存、内部状态或秘密属性一并输出。不要为了省事无条件把所有对象的 `__dict__` 都发给外部。

### 10.7 子类化 `JSONEncoder`

```python
import json
from datetime import datetime


class AppEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        return super().default(obj)


text = json.dumps({"value": 1}, cls=AppEncoder)
```

`cls` 指定编码器类。适合项目想集中组织扩展规则时使用。规则很少时，一个普通 `default` 函数往往更容易阅读。

调用 `super().default(obj)` 会保留基类对未知对象的正常报错，而不是默默给未知类型返回 `None`。

## 第十一章：自定义解析——数值精度与对象钩子

### 11.1 `parse_float`：小数不一定要转成 float

```python
import json
from decimal import Decimal

obj = json.loads('{"price": 0.1, "tax": 0.2}', parse_float=Decimal)
print(obj["price"] + obj["tax"])
```

这里得到十进制精度语义下的 `Decimal` 值。默认 `float` 使用二进制浮点，并不能精确表示所有十进制小数。

`parse_float` 接收到对应数字的字符串形式，让你选择转换规则。它不处理 JSON 字符串中的 `"0.1"`，因为后者本来就是文本。

### 11.2 解析为 Decimal 后，编码也要考虑

标准库编码器默认不直接支持 `Decimal`。你需要决定输出数值、字符串或其他结构。

把 `Decimal` 转成 `float` 可能重新引入二进制浮点表示问题；转成字符串会改变 JSON 类型。金额系统应在协议里明确使用最小货币单位整数、十进制字符串或双方一致支持的数值方案。

不要只在读取一端改 `parse_float`，就宣称端到端精度问题已经全部解决。

### 11.3 `parse_int`

```python
import json

obj = json.loads('{"id": 123456}', parse_int=str)
print(obj)
```

这会把整数形式数字交给指定转换器。它不是把所有数值或所有字段名变成字符串，只影响对应解析入口。

Python 对超长整数文本还可能存在解释器级转换限制。不要为了接收任意巨大输入就随意关闭保护，应先约定合理长度和业务范围。

### 11.4 `parse_constant`：拒绝特殊浮点常量

严格解析时，可以拒绝 Python 默认接受的 `NaN`、`Infinity` 和 `-Infinity`：

```python
import json


def reject_constant(value):
    raise ValueError(f"不允许非标准数字：{value}")


obj = json.loads('{"value": 1.5}', parse_constant=reject_constant)
```

这个回调不会对 `true`、`false` 或 `null` 都执行，它针对的是相应非标准数值常量。

`JSONDecodeError` 与自定义回调抛出的 `ValueError` 需要在调用层按实际来源理解，不能假设所有失败都带同样的行列信息。

### 11.5 `object_hook`：对象字典解析后的转换

```python
import json


def convert_point(obj):
    if set(obj) == {"kind", "x", "y"} and obj["kind"] == "point":
        return (obj["x"], obj["y"])
    return obj


result = json.loads(
    '{"shape":{"kind":"point","x":10,"y":20}}',
    object_hook=convert_point,
)
print(result)
```

JSON 对象解析成字典后，钩子可以把它替换成另一个对象。嵌套对象也会受到影响。

这意味着钩子改变了默认“对象就是字典”的输出类型约定。使用它的函数应在文档里写明返回结构，不要让下游继续按未转换字典访问。

### 11.6 类型标记不能成为随意执行代码的入口

有些协议会加 `"kind": "point"` 这样的标记，用来选择允许的转换规则。这本身可以合理使用，但必须限制在明确的类型白名单内。

不要根据外部字段随意导入模块、反射任意类或执行任意函数。JSON 的文本解析不会直接执行代码，不表示你后续自己写的对象恢复逻辑天然安全。

## 第十二章：重复键、顺序与严格解析

### 12.1 重复键可能被覆盖

```python
import json

obj = json.loads('{"role":"reader","role":"admin"}')
print(obj)
```

标准库默认会保留后面的值，前面的同名字段不再作为独立项存在。

重复键会让不同系统产生分歧，尤其验证方与使用方对取值规则不一致时，很容易造成错误判断。正式协议最好拒绝重复键。

### 12.2 `object_pairs_hook` 可以检查原始键值对

```python
import json


def reject_duplicate_keys(pairs):
    obj = {}
    for key, value in pairs:
        if key in obj:
            raise ValueError(f"重复的 JSON 键：{key}")
        obj[key] = value
    return obj


result = json.loads(
    '{"name":"小林","age":20}',
    object_pairs_hook=reject_duplicate_keys,
)
```

这里回调拿到有顺序的键值对列表，可以在构造字典之前发现重复项。

如果同时提供 `object_pairs_hook` 和 `object_hook`，前者具有优先性。不要把两者当成必然依次执行的两条流水线。

### 12.3 顺序保留不等于顺序有业务语义

Python 字典保留插入顺序，JSON 编解码通常也会按相应容器顺序工作。

但 JSON 对象在交换语义上不应靠字段排列来决定业务意义。如果顺序重要，例如排行榜、步骤列表或时间线，应使用数组明确表达顺序。

```json
{
  "steps": ["下载", "验证", "解压", "安装"]
}
```

这比用对象键的偶然排列表示步骤更清楚。

### 12.4 一个较严格的解析入口

```python
import json


def reject_constant(value):
    raise ValueError(f"非标准数字：{value}")


def unique_object(pairs):
    result = {}
    for key, value in pairs:
        if key in result:
            raise ValueError(f"重复键：{key}")
        result[key] = value
    return result


def loads_strict(text):
    return json.loads(
        text,
        parse_constant=reject_constant,
        object_pairs_hook=unique_object,
    )
```

“较严格”指这里额外拒绝特殊数字和重复键，不代表它自动实现任意业务模式验证、体积限制或所有协议要求。

例如顶层必须是对象、字符串长度上限、数值范围和未知字段政策，仍然需要独立检查。

## 第十三章：Unicode、转义和两层字符串语法

### 13.1 Python 字符串转义与 JSON 转义分别发生

```python
import json

text = '{"message":"first\\nsecond"}'
obj = json.loads(text)
print(repr(obj["message"]))
```

Python 字面量中的 `\\n` 先形成 JSON 文本里的反斜杠加 `n`；JSON 解析器再把它解释成字段中的换行字符。

如果你只写 `\n`，Python 会先产生真实换行，直接放进 JSON 字符串内部通常违反控制字符转义规则。

两层规则一起出现容易混乱。最可靠的做法往往是先构造 Python 字典，再由 `dumps()` 生成 JSON，而不是手工拼接复杂的嵌套字符串。

### 13.2 引号与反斜杠也应交给编码器

```python
import json

obj = {
    "quote": '他说："你好"',
    "path": r"C:\notes\report.txt",
}

text = json.dumps(obj, ensure_ascii=False)
print(text)
print(json.loads(text) == obj)
```

编码器负责把引号、反斜杠和控制字符写成合法 JSON 表达。你不需要先手工把每个反斜杠替换一次，再让编码器重复转义。

### 13.3 打印显示和 `repr()` 显示不同

`print(text)` 输出字符串实际内容；`repr(text)` 输出适合观察转义的 Python 表示。

同一个字符串在这两种显示下，反斜杠数量看起来可能不同。调试时不要仅凭屏幕上看见几个斜杠就反复“修正”数据，应确认当前变量的类型与实际内容。

### 13.4 中文转义与表情字符

`ensure_ascii=True` 会使用 ASCII 范围内的转义表达非 ASCII 内容。某些非 BMP 字符可能以代理对形式出现，这是 JSON Unicode 转义表示相关机制。

日常阅读中文配置时，`ensure_ascii=False` 更直观；跨系统传输时，两种合法表示都需要接收方正确解码。

不要把所有 `\u` 片段当成应该用正则替换的“乱码”。合法 JSON 应交给解析器还原，而不是自己写一个不完整的 Unicode 替换器。

## 第十四章：连续记录与 JSON Lines

### 14.1 不能反复 `dump()` 就自动得到一份合法 JSON

```python
# 反面示例：把两个独立文档无分隔地写在一起。
with open("bad.json", "w", encoding="utf-8") as target:
    json.dump({"id": 1}, target)
    json.dump({"id": 2}, target)
```

结果类似：

```text
{"id": 1}{"id": 2}
```

这不是一个普通完整 JSON 文档。`json.load()` 不会自动把它解释成两个记录。

可以把多个对象放在数组里：

```json
[{"id":1},{"id":2}]
```

或者选择明确的逐行记录格式。

### 14.2 JSON Lines 的基本约定

每一行保存一个独立 JSON 值，通常使用 UTF-8，行之间用换行分隔。

```text
{"id":1,"message":"开始"}
{"id":2,"message":"结束"}
```

它不是“整个文件是一份 JSON 数组”，因此通常需要逐行调用 `loads()`。

记录中的字符串如果包含换行，`dumps()` 会在 JSON 字符串里进行转义，不会直接把它变成记录分隔的真实换行。不要使用 `indent` 让一条记录跨多行，否则就破坏了这个约定。

### 14.3 写入逐行 JSON

```python
import json

records = [
    {"id": 1, "message": "第一条"},
    {"id": 2, "message": "含有\n字段内部换行"},
]

with open("events.jsonl", "w", encoding="utf-8", newline="\n") as target:
    for record in records:
        text = json.dumps(
            record,
            ensure_ascii=False,
            separators=(",", ":"),
            allow_nan=False,
        )
        target.write(text + "\n")
```

### 14.4 逐行读取，错误带行号

```python
import json

with open("events.jsonl", encoding="utf-8") as source:
    for number, line in enumerate(source, start=1):
        if not line.strip():
            raise ValueError(f"第 {number} 行为空，不符合本程序记录约定")

        try:
            record = json.loads(line)
        except json.JSONDecodeError as error:
            raise ValueError(f"第 {number} 行 JSON 格式错误") from error

        print(record)
```

这里主动拒绝空白行。某些应用会选择跳过空行，也可以，但要写成协议约定，不能一边说“每行一个值”一边无声吞掉各种坏行。

### 14.5 逐行格式的优点和限制

优点是便于追加、逐条消费和定位记录错误，不需要先收集所有记录形成一个大数组。

限制是每条记录仍然需要整体解析；超长单行仍可能占很多内存；多个写入者并发追加也不自动形成完整事务；最后一行可能因为中断而不完整。

因此，JSON Lines 改变的是记录边界与处理方式，不是把所有持久化问题都解决了。

## 第十五章：完整实践——读取、验证并更新应用配置

### 15.1 先明确配置模型

我们给一个小程序设计以下配置：

```json
{
  "version": 1,
  "theme": "dark",
  "page_size": 20,
  "debug": false
}
```

约定：顶层必须是对象；只接受这四个字段；版本必须为一；主题只能是 `light` 或 `dark`；每页数量为一到一百的整数；调试标记必须为真正的布尔值。

配置文件不存在时允许采用默认值；配置存在却格式错误时必须报告，不能悄悄当成默认配置。这两个情形含义不同。

### 15.2 完整代码

```python
import json
import os
import tempfile
from pathlib import Path


DEFAULT_CONFIG = {
    "version": 1,
    "theme": "light",
    "page_size": 20,
    "debug": False,
}


def validate_config(obj):
    if not isinstance(obj, dict):
        raise ValueError("配置顶层必须是对象")

    unknown = set(obj) - set(DEFAULT_CONFIG)
    if unknown:
        raise ValueError(f"未知配置项：{sorted(unknown)}")

    result = DEFAULT_CONFIG | obj

    if type(result["version"]) is not int or result["version"] != 1:
        raise ValueError("不支持的配置版本")
    if not isinstance(result["theme"], str) or result["theme"] not in {"light", "dark"}:
        raise ValueError("theme 只能是 light 或 dark")
    if type(result["page_size"]) is not int or not 1 <= result["page_size"] <= 100:
        raise ValueError("page_size 必须是 1 到 100 的整数")
    if type(result["debug"]) is not bool:
        raise ValueError("debug 必须是布尔值")

    return result


def load_config(path):
    path = Path(path)

    try:
        with path.open(encoding="utf-8") as source:
            obj = json.load(source)
    except FileNotFoundError:
        return DEFAULT_CONFIG.copy()

    return validate_config(obj)


def save_config(path, obj):
    validated = validate_config(obj)
    text = json.dumps(
        validated,
        ensure_ascii=False,
        indent=2,
        allow_nan=False,
    ) + "\n"

    path = Path(path).absolute()
    path.parent.mkdir(parents=True, exist_ok=True)
    temp_path = None

    try:
        with tempfile.NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            newline="\n",
            dir=path.parent,
            prefix=path.name + ".",
            suffix=".tmp",
            delete=False,
        ) as target:
            temp_path = Path(target.name)
            target.write(text)
            target.flush()
            os.fsync(target.fileno())

        temp_path.replace(path)
    finally:
        if temp_path is not None:
            temp_path.unlink(missing_ok=True)


path = Path("json_practice") / "app_config.json"
config = load_config(path)
config["theme"] = "dark"
save_config(path, config)
print(load_config(path))
```

### 15.3 为什么默认值要复制？

`DEFAULT_CONFIG` 是一个可变字典。如果直接返回它，调用者修改主题就可能改变下一次读取时使用的全局默认配置。

这里值都是不可变基本对象，浅拷贝足够隔离顶层修改。如果默认配置里包含列表或嵌套字典，浅拷贝仍共享内部对象，需要更明确地构造新结构或使用适合的深拷贝方案。

这属于 Python 对象共享问题，不能因为函数名字叫“读取配置”就忽略它。

### 15.4 `DEFAULT_CONFIG | obj` 的含义

字典合并运算符生成新的字典，右侧同名项覆盖左侧。它适合表达“默认值加用户提供的顶层字段”。

这不是递归深度合并。假如一个字段本身是字典，右侧整个字段会替换左侧字段，不会自动逐层合并。

### 15.5 为什么序列化放在打开正式输出之前？

`json.dumps()` 可能因不支持的类型、循环结构或非有限数字失败。如果先用 `w` 打开正式配置，再调用编码器，失败时原配置就可能已经被清空。

先验证并生成字符串，让更多错误发生在修改文件之前。然后再写临时文件，成功后替换正式路径，进一步减少不完整配置被发布的风险。

### 15.6 为什么只对缺失文件使用默认值？

配置存在但损坏，可能是用户写错、程序中断或其他问题。自动忽略后继续运行，可能使用户以为自己设置了某个选项，实际程序却使用默认值。

因此，缺失与损坏应分开处理。这个例子保留 JSON、编码和权限错误，交给调用者报告。真实应用也可以备份坏文件并提供修复流程，但那是需要明确设计的行为。

### 15.7 临时替换仍然有哪些限制？

这个保存函数不保留原文件全部权限和元数据，不解决多个进程同时更新后相互覆盖的问题，也不提供目录元数据同步和跨文件事务。

若两个写入者同时读取旧配置、分别修改不同字段、再各自保存，后保存者仍可能覆盖前一个修改。需要协调时应使用锁、版本检查或数据库等合适机制。

## 第十六章：完整实践——批量验证一个目录中的 JSON 文件

### 16.1 任务目标

扫描练习目录中的 `.json` 普通文件，逐份验证语法和简单数据结构，输出一份报告。某份文件错误时继续处理其他文件，但最后明确返回是否全部成功。

假设每份文件要求具有字符串 `name` 和整数 `count`，不允许布尔值冒充整数。

### 16.2 完整代码

```python
import json
from pathlib import Path


def validate_record(obj):
    if not isinstance(obj, dict):
        raise ValueError("顶层必须是对象")
    if not isinstance(obj.get("name"), str) or not obj["name"].strip():
        raise ValueError("name 必须是非空字符串")
    if type(obj.get("count")) is not int or obj["count"] < 0:
        raise ValueError("count 必须是非负整数")
    return obj


def inspect_json_file(path):
    try:
        with path.open(encoding="utf-8") as source:
            obj = json.load(source)
        validate_record(obj)
    except json.JSONDecodeError as error:
        return {
            "path": str(path),
            "status": "invalid_json",
            "message": error.msg,
            "line": error.lineno,
            "column": error.colno,
        }
    except UnicodeDecodeError as error:
        return {"path": str(path), "status": "encoding_error", "message": str(error)}
    except OSError as error:
        return {"path": str(path), "status": "io_error", "message": str(error)}
    except ValueError as error:
        return {"path": str(path), "status": "schema_error", "message": str(error)}
    else:
        return {"path": str(path), "status": "ok"}


def validate_directory(folder):
    folder = Path(folder)
    if not folder.is_dir():
        raise NotADirectoryError(folder)

    results = []
    for path in sorted(folder.glob("*.json")):
        if path.is_file() and not path.is_symlink():
            results.append(inspect_json_file(path))

    return {
        "checked": len(results),
        "all_valid": all(item["status"] == "ok" for item in results),
        "results": results,
    }


report = validate_directory("json_practice/input")
print(json.dumps(report, ensure_ascii=False, indent=2, allow_nan=False))
```

### 16.3 为什么异常捕获有顺序？

`JSONDecodeError` 继承自 `ValueError`。如果先写 `except ValueError`，JSON 语法错误就会先落进那个分支，失去我们想保留的行列信息分类。

因此，较具体的异常放前面，较宽泛的放后面。这里 `ValueError` 主要接收我们自己的业务校验错误。

### 16.4 `all([])` 为什么为真？

如果目录中没有匹配文件，`results` 是空列表，`all(...)` 会返回 `True`。数学上这是“所有元素都满足条件”的空集规则。

但业务上你可能希望“没有任何输入”算失败。可以另加：

```python
all_valid = bool(results) and all(item["status"] == "ok" for item in results)
```

这不是 Python 的 bug，而是业务规则与通用逻辑函数默认含义不同。写报告时最好同时保留 `checked`，让调用者看清究竟检查了多少份。

### 16.5 输出目录与扫描目录分开

如果把报告也保存到输入目录，并使用相同 `.json` 后缀，下一次扫描可能把自己的报告当作待验证业务文件。

建议把输入放在 `input/`，报告放在 `reports/`，或明确排除输出路径。目录布局可以减少逻辑歧义，比每次运行后手动删除报告更可靠。

## 第十七章：完整实践——合并多份记录文件，并处理重复 ID

### 17.1 输入模型

每份输入文件的顶层是数组，每条记录包含：

```json
{"id": "s001", "name": "小林", "score": 95}
```

合并时不能因为某个 ID 重复就悄悄保留最后一条。这个练习选择发现重复就报告，因为 ID 是业务唯一标识。

### 17.2 实现

```python
import json
from pathlib import Path


def load_records(path):
    with path.open(encoding="utf-8") as source:
        records = json.load(source)

    if not isinstance(records, list):
        raise ValueError(f"{path}：顶层必须是数组")

    for index, record in enumerate(records):
        if not isinstance(record, dict):
            raise ValueError(f"{path}：第 {index} 项必须是对象")
        if not isinstance(record.get("id"), str) or not record["id"]:
            raise ValueError(f"{path}：第 {index} 项缺少字符串 id")
        if not isinstance(record.get("name"), str):
            raise ValueError(f"{path}：第 {index} 项 name 类型错误")
        if type(record.get("score")) is not int or not 0 <= record["score"] <= 100:
            raise ValueError(f"{path}：第 {index} 项 score 不合法")

    return records


def merge_records(paths):
    merged = []
    origins = {}

    for path in paths:
        path = Path(path)

        for index, record in enumerate(load_records(path)):
            record_id = record["id"]

            if record_id in origins:
                previous_path, previous_index = origins[record_id]
                raise ValueError(
                    f"重复 id {record_id!r}："
                    f"{previous_path}[{previous_index}] 与 {path}[{index}]"
                )

            origins[record_id] = (path, index)
            merged.append(record.copy())

    return merged


sources = sorted(Path("json_practice/parts").glob("*.json"))
records = merge_records(sources)
records.sort(key=lambda record: record["id"])

output = Path("json_practice/output/merged.json")
output.parent.mkdir(parents=True, exist_ok=True)

with output.open("w", encoding="utf-8", newline="\n") as target:
    json.dump(records, target, ensure_ascii=False, indent=2, allow_nan=False)
    target.write("\n")
```

### 17.3 为什么保存来源信息？

发现重复时只说“ID 重复”往往不足以修复数据。`origins` 保存第一次出现的位置，错误里同时显示两处来源，用户可以直接找到冲突文件和数组索引。

这里数组索引从零开始，是为了与 Python 访问方式一致。面向非程序用户时，也可以另外显示从一开始的记录序号，但应明确区分。

### 17.4 JSON 对象重复键与业务记录重复 ID 不同

`{"id":"a","id":"b"}` 是同一个 JSON 对象里的重复键问题，可以通过 `object_pairs_hook` 识别。

`[{"id":"a"},{"id":"a"}]` 中每个对象的键都没有重复，但两条记录在业务标识上冲突，需要业务层检查。

因此，严格解析器和唯一性检查分别解决不同问题。只配置一个钩子，不能自动覆盖所有数据一致性要求。

### 17.5 输出顺序是明确选择

这里按 ID 排序，方便比较。也可以保留输入文件顺序和各数组原顺序，但要把规则写清楚。

如果 ID 类似 `s2`、`s10`，字符串排序不一定是自然数字顺序。是否补零、提取编号或采用其他规则，应由 ID 格式约定决定。

## 第十八章：完整实践——把 JSON Lines 汇总为统计报告

### 18.1 每条输入事件

```json
{"level":"INFO","duration_ms":12.5,"service":"worker"}
```

要求：按日志级别计数，计算非负有限耗时的平均值。对格式错误的行记录问题，继续处理其他行，最后明确报告坏行数量。

### 18.2 完整代码

```python
import json
import math
from collections import Counter


def summarize_jsonl(path):
    counts = Counter()
    duration_total = 0.0
    duration_count = 0
    errors = []

    with open(path, encoding="utf-8") as source:
        for number, line in enumerate(source, start=1):
            try:
                if not line.strip():
                    raise ValueError("空白行")

                event = json.loads(line)
                if not isinstance(event, dict):
                    raise ValueError("记录必须是对象")

                level = event.get("level")
                duration = event.get("duration_ms")

                if not isinstance(level, str) or not level:
                    raise ValueError("level 必须是非空字符串")
                if type(duration) not in (int, float):
                    raise ValueError("duration_ms 必须是数值")
                if isinstance(duration, float) and not math.isfinite(duration):
                    raise ValueError("duration_ms 必须是有限数值")
                if not 0 <= duration <= 86_400_000:
                    raise ValueError("duration_ms 超出本程序允许范围")

            except (json.JSONDecodeError, ValueError) as error:
                errors.append({"line": number, "message": str(error)})
                continue

            counts[level] += 1
            duration_total += duration
            duration_count += 1

    return {
        "valid_records": duration_count,
        "invalid_records": len(errors),
        "counts_by_level": dict(counts),
        "average_duration_ms": (
            duration_total / duration_count if duration_count else None
        ),
        "errors": errors,
    }


report = summarize_jsonl("json_practice/events.jsonl")
print(json.dumps(report, ensure_ascii=False, indent=2, allow_nan=False))
```

### 18.3 为什么要给数值设置上限？

我们把一天以内的毫秒数作为示例业务限制。它既帮助发现错误单位，也避免接受不合理的大数值后，在后续浮点累计中出现转换或溢出问题。

这个上限不是日志耗时的通用标准。真实系统可以选择更合适范围，但应该明确，不要仅凭“是个数字”就接收任何数量级。

### 18.4 为什么平均值没有记录时使用 `null`？

没有合法耗时记录时，平均值并不存在。用 `None` 编码成 `null` 比填零更清楚，因为零表示真实平均耗时为零，与“没有数据”不同。

这也是前面强调缺失、空值和零需要区分的一个实际例子。

### 18.5 保存所有错误会不会消耗内存？

会。虽然事件是逐行处理的，但如果每一行都错，`errors` 列表就会不断增长。

可以只保存前一百个错误，同时累计总错误数；也可以把错误逐行写入独立报告。流式输入只是内存控制的一部分，累计输出数据同样需要考虑。

## 第十九章：完整实践——按大小限制读取 JSON 配置

### 19.1 为什么不能只靠文件大小预检查？

查询文件状态后，内容可能变化；某些路径也不是普通静态文件。你如果检查后仍然无上限 `read()`，读取步骤并没有真正遵守边界。

对于小型配置，按字节限制实际读取，再解码和解析，是容易理解的做法。

### 19.2 实现

```python
import json
from pathlib import Path


def load_small_json(path, max_bytes=1_000_000):
    if type(max_bytes) is not int or max_bytes < 0:
        raise ValueError("max_bytes 必须是非负整数")

    with Path(path).open("rb") as source:
        payload = source.read(max_bytes + 1)

    if len(payload) > max_bytes:
        raise ValueError("JSON 文件超过允许大小")

    text = payload.decode("utf-8")
    return json.loads(text)
```

### 19.3 为什么多读一个字节？

只读到上限数量，无法判断后面是否仍有内容。多读一个字节，既保留了明确上界，也能识别超限。

这个函数的限制单位是原始字节，字符数量和解析后对象的内存占用可能不同。一个字节规模有限的 JSON 仍然会产生额外容器和对象开销，因此不能把一 MiB 文件等同于一 MiB 内存。

### 19.4 为什么这里手动 `decode("utf-8")`？

这样明确约定输入必须是普通 UTF-8，并把字节大小限制放在解码之前。

如果业务允许 BOM，可以使用 `utf-8-sig`；如果协议要求其他编码，应明确修改约定。不能为了兼容所有未知来源就无限尝试编码并接受第一个不报错的结果。

### 19.5 长度限制不等于完整安全隔离

极深嵌套、复杂结构、超长字符串和后续业务算法都可能有开销。标准库解析器没有一个参数能同时为你设置所有业务深度、记录数和字段长度限制。

可以在数据结构校验时增加相应约束。对真正高风险输入，还要结合请求体限制、执行环境资源限制和服务整体设计。

## 第二十章：API 数据中的 JSON，与 HTTP 本身怎样区分？

### 20.1 HTTP 是传输协议，JSON 是内容格式

一个 HTTP 响应包括状态码、响应头和响应体。JSON 通常只是响应体的一种内容格式。

状态码成功，不保证响应体具有期望业务字段；响应体可以解析为 JSON，也不保证状态码成功。错误响应同样常用 JSON 描述错误。

### 20.2 不要重复编码

使用高层 HTTP 客户端时，常有 `json=` 参数：

```python
# 这是 Requests 风格示意，实际发请求会访问指定服务。
payload = {"name": "小林", "active": True}
# response = requests.post(url, json=payload, timeout=10)
```

客户端通常会负责序列化并设置合适的内容类型。

如果你先 `json.dumps(payload)`，又把得到的字符串传进 `json=`，可能把一个 JSON 对象编码成 JSON 字符串，形成双重编码。

```python
import json

payload = {"name": "小林"}
once = json.dumps(payload, ensure_ascii=False)
twice = json.dumps(once, ensure_ascii=False)

print(once)
print(twice)
print(type(json.loads(twice)))
```

最后解析一次得到的是字符串，而不是原字典。这类问题常见于服务之间没有明确当前变量层次。

### 20.3 `Content-Type` 不是业务校验

响应头声明 `application/json` 表达发送方的格式意图，但接收方仍要处理空内容、格式损坏和不符合结构的情况。

反过来，一个字符串即使看起来像 JSON，也不意味着你应该无视协议头和端点约定随意解析。接口契约应同时描述状态、头、内容结构和错误处理。

### 20.4 读取接口结果后不要随意把它当可信配置

JSON 解析不会验证来源、签名或授权。外部接口返回一个文件路径、命令名称或模板字段时，后续使用这些字段仍然需要业务验证。

“安全解析为数据”与“这个数据可以被授权用于任何操作”不是同一个保证。

## 第二十一章：序列化前的数据整理与脱敏

### 21.1 为什么不直接输出整个用户对象？

用户对象可能含有密码摘要、访问令牌、内部权限标记、缓存或其他不应公开的信息。

即使编码器能把对象变成 JSON，也不代表这些字段都适合交给调用者。应明确构造输出模型：

```python
user = {
    "id": "u001",
    "name": "小林",
    "password_hash": "example-only",
    "access_token": "example-only",
}

public_user = {
    "id": user["id"],
    "name": user["name"],
}
```

### 21.2 白名单比猜测黑名单更容易审阅

如果只删除几个已知秘密字段，后来新增一个内部字段时可能忘记排除。

按允许字段主动构建输出，能让审阅者直接看见外部协议有哪些内容。模型复杂时可以使用专门的序列化或验证工具，但原则仍是明确输出边界。

### 21.3 日志里的 JSON 也可能暴露信息

```python
print(json.dumps(request_data, ensure_ascii=False))
```

这行看起来只是调试，但如果 `request_data` 含凭证，就会进入日志。

序列化工具不负责判断字段是否敏感。记录日志前应选择必要字段，避免把完整请求体、环境变量或认证头无条件输出。

### 21.4 保留原始输入与生成整理结果要区分

某些审计或数据处理任务需要保留原始字节。JSON 解析再重新编码后，字段布局、空白、转义和数字显示可能变化，即使结构意义相同。

因此，原始文件与规范化输出应作为两个不同产物管理。不能用一次 `loads()` 加 `dumps()` 就宣称原始内容逐字节保真。

## 第二十二章：排序、规范化和比较两份 JSON

### 22.1 文本不同，不代表数据不同

下面两段文本的布局和字段排列不同：

```json
{"name":"小林","age":20}
```

```json
{
  "age": 20,
  "name": "小林"
}
```

普通结构比较可以先解析，再比较 Python 对象：

```python
import json

first = json.loads('{"name":"小林","age":20}')
second = json.loads('{"age":20,"name":"小林"}')

print(first == second)
```

结果通常为 `True`。字典相等主要比较键和值，不要求插入顺序相同。

### 22.2 数组顺序通常有意义

```python
print(json.loads('[1,2]') == json.loads('[2,1]'))
```

结果为 `False`。JSON 数组表达有序序列。

如果业务把某个数组当成无序集合，你需要按业务规则比较，不能直接把所有数组都排序。例如步骤、坐标、排名和时间线都不应随意重排。

### 22.3 Python 的数值相等可能比你的协议宽松

```python
print(1 == 1.0)
print(True == 1)
```

都可能为真。因此，直接比较解析后对象，也不一定足以判断协议层的类型完全一致。

如果你的模式严格区分布尔、整数和浮点表示，应在比较或验证时考虑精确类型，而不是只依赖 `==`。

### 22.4 用格式化输出改善版本差异

```python
import json


def pretty_json(text):
    obj = json.loads(text)
    return json.dumps(
        obj,
        ensure_ascii=False,
        indent=2,
        sort_keys=True,
        allow_nan=False,
    ) + "\n"
```

这会统一常见布局，使人工比较更容易。但它会改变原始文本表现，并可能受到默认解析数值和重复键处理的影响。

要整理重要原始资料时，先保留原文件，再生成格式化副本。不要未经说明地把格式化当成完全无损的原始字节处理。

### 22.5 为签名准备规范化数据需要更明确规范

如果服务两端要对 JSON 计算一致签名，仅仅 `sort_keys=True` 和删除空格未必足够。

还要约定数字表示、Unicode 字符处理、转义、重复键、非有限数值以及字节编码等规则。对于数字签名，应选择明确的规范化标准和经过验证的实现，而不是临时拼一个看起来稳定的字符串。

普通缓存键和调试比较可以采用较简单约定，但要知道它们的适用范围，不应扩大承诺。

## 第二十三章：完整实践——生成两个 JSON 对象的结构差异报告

### 23.1 需求先限定清楚

比较两个已解析的 JSON 值，报告哪些位置的类型或数值不同。对象按键比较，数组按索引比较，不尝试猜测数组记录的业务 ID。

为了让路径显示清楚，我们使用以 `$` 开头的教学路径表示。它只是本程序的显示格式，不声称兼容某一种标准 JSON 查询语言。

### 23.2 实现

```python
import json


def compare_json(first, second, path="$", differences=None):
    if differences is None:
        differences = []

    if type(first) is not type(second):
        differences.append({
            "path": path,
            "kind": "type_changed",
            "before_type": type(first).__name__,
            "after_type": type(second).__name__,
        })
        return differences

    if isinstance(first, dict):
        first_keys = set(first)
        second_keys = set(second)

        for key in sorted(first_keys - second_keys):
            differences.append({
                "path": f"{path}[{key!r}]",
                "kind": "removed",
                "before": first[key],
            })

        for key in sorted(second_keys - first_keys):
            differences.append({
                "path": f"{path}[{key!r}]",
                "kind": "added",
                "after": second[key],
            })

        for key in sorted(first_keys & second_keys):
            compare_json(
                first[key],
                second[key],
                path=f"{path}[{key!r}]",
                differences=differences,
            )

    elif isinstance(first, list):
        common_length = min(len(first), len(second))

        for index in range(common_length):
            compare_json(
                first[index],
                second[index],
                path=f"{path}[{index}]",
                differences=differences,
            )

        for index in range(common_length, len(first)):
            differences.append({
                "path": f"{path}[{index}]",
                "kind": "removed",
                "before": first[index],
            })

        for index in range(common_length, len(second)):
            differences.append({
                "path": f"{path}[{index}]",
                "kind": "added",
                "after": second[index],
            })

    elif first != second:
        differences.append({
            "path": path,
            "kind": "value_changed",
            "before": first,
            "after": second,
        })

    return differences


first = {"name": "小林", "scores": [95, 88], "active": True}
second = {"name": "小林", "scores": [96, 88, 100], "active": 1}

differences = compare_json(first, second)
print(json.dumps(differences, ensure_ascii=False, indent=2, allow_nan=False))
```

### 23.3 为什么默认参数不写 `differences=[]`？

可变默认参数会在函数定义时创建，并被多次调用共享。不同顶层比较可能把结果积累到同一个列表中。

使用 `None`，在第一次调用内部新建列表，再把这个列表明确传给递归调用，是常见的正确组织方式。

### 23.4 为什么先比较精确类型？

我们希望把布尔 `True` 与整数 `1` 标成类型变化，而不被 Python 的宽松数值相等掩盖。

这个比较器输入应为 JSON 基本类型组成的树。若你使用自定义解析钩子产生 `Decimal`、元组或自定义对象，需要重新定义类型与比较政策。

### 23.5 数组插入会产生多处差异

假如原数组 `["a","b"]` 在开头插入 `"x"`，变为 `["x","a","b"]`，按索引比较会看到多个值变化和尾部增加。

这不是程序算错，而是比较模型选择了索引语义。要按业务 ID 识别记录移动或修改，应先建立 ID 对应关系，不能把通用数组比较器当成懂业务的差异引擎。

### 23.6 深层递归有实际限制

这个教学实现使用递归，结构特别深时可能触发解释器递归限制。不要为了接受任意深度就随意把限制调得极高。

实际输入应有深度限制。需要处理非常深的受控结构时，可以设计显式栈的迭代算法；需要交换普通业务数据时，过深嵌套通常也会降低可维护性。

## 第二十四章：`JSONEncoder.iterencode()` 与 `raw_decode()`

这两个接口比四个核心函数更接近编码器和解码器的工作过程。先理解用途，再决定是否需要使用。

### 24.1 `iterencode()` 逐段产生输出文本

```python
import json
from io import StringIO

encoder = json.JSONEncoder(ensure_ascii=False, allow_nan=False)
obj = {"name": "小林", "values": [1, 2, 3]}

with StringIO() as target:
    for chunk in encoder.iterencode(obj):
        target.write(chunk)
    text = target.getvalue()

print(text)
```

每个 `chunk` 是输出片段，不保证对应一个完整字段、一行或一个数组元素。调用者应按字符串片段依次写入，不应自行把每段当成独立 JSON 解析。

### 24.2 逐段编码不等于输入对象不占内存

`iterencode()` 可以避免先拼出一个巨大的结果字符串，但输入的 Python 对象通常仍然已经存在。

如果你先用列表保存一百万条记录，再逐段编码，这个列表和其中对象仍然占内存。真正端到端的流式流程，需要同时考虑输入生成、记录边界和输出协议。

此外，编码中途遇到不支持的对象，可能已经写出一部分文本。需要完整发布的文件仍应采用临时结果策略。

### 24.3 `raw_decode()` 返回值和结束位置

```python
import json

decoder = json.JSONDecoder()
text = '{"a":1} trailing'

obj, end = decoder.raw_decode(text)

print(obj)
print(end)
print(text[end:])
```

它用于在字符串某个位置解码一个 JSON 值，并返回值结束位置。与 `loads()` 的完整文档要求不同，它可以把后续内容留给调用者处理。

### 24.4 不要把它当成自动协议分帧器

如果数据从网络分多次到达，当前字符串可能只含半个 JSON 值。你需要判断继续等待、输入非法、消息超限或连接结束等情况。

简单反复 `raw_decode()` 并不能自动给你完成所有缓冲、超时和边界处理。

文本前导空白的处理也需要按接口规则明确安排。对于普通文件，直接 `load()`；对于多条记录，优先选择有明确分隔的协议，通常更容易可靠实现。

## 第二十五章：命令行工具与独立排错步骤

### 25.1 `python -m json.tool`

保存一个练习 JSON 文件后，可以在终端运行：

```bash
python -m json.tool json_practice/settings.json
```

它会尝试验证并格式化输出。这里 `-m` 表示按模块执行当前解释器环境中的工具。

没有必要单独寻找一个名叫 `json` 的可执行程序。使用当前 Python 运行标准库工具，也便于确认你实际用的是哪个解释器。

### 25.2 输出到另一个文件

```bash
python -m json.tool json_practice/settings.json json_practice/settings_pretty.json
```

输入输出应使用不同路径，避免覆盖输入引发问题。编码、选项与错误输出细节应按目标 Python 版本帮助查看：

```bash
python -m json.tool --help
```

命令行工具方便检查语法，不会自动验证你的应用要求有哪些字段。

### 25.3 排错时先回答四个问题

第一，当前变量是什么类型：路径、字符串、字节、字典还是响应对象？

第二，你要做哪个方向：对象变文本，还是文本变对象？

第三，输入是否真是一个完整 JSON 文档，而不是 HTML 错误页、空响应或多个文档拼接？

第四，解析成功后，字段类型和业务值是否符合约定？

这四个问题通常比“试一下换成另一个 load 函数”更能快速定位错误。

### 25.4 对外报告不要无条件泄露完整输入

`JSONDecodeError` 持有文档和位置信息。调试时打印完整文档很方便，但输入中可能包含凭证或隐私。

用户提示可包含安全的文件名、行列和简短原因；内部日志也应按需要记录，避免把整个请求体直接输出。工具提供了信息，不代表每个地方都应把全部信息显示给所有人。

## 第二十六章：十六个常见误区逐个纠正

### 26.1 “JSON 就是字典”

JSON 是文本格式，顶层还能是数组、字符串、数值、布尔或空值。Python 字典只是默认对象映射之一。

### 26.2 “`dumps()` 会写文件”

它返回字符串。写文件用 `dump()` 配合流，或者手动写入字符串。

### 26.3 “`load()` 接收文件名”

它接收可读流。路径要先由 `open()` 或 `Path.open()` 打开。

### 26.4 “`s` 表示多个对象”

这里提示字符串接口。`loads()` 通常解析一个完整 JSON 文档，不是把多个拼接对象自动拆开。

### 26.5 “中文 `\u` 转义就是乱码”

合法转义是格式表示，解析后可恢复原字符。可读性需求可用 `ensure_ascii=False`。

### 26.6 “`ensure_ascii=False` 会设置 UTF-8 字节编码”

它控制 JSON 文本表现。字节编码由后续编码或文本流处理。

### 26.7 “能解析就说明业务数据正确”

语法、结构、字段值和权限来源是不同层次。需要独立验证。

### 26.8 “字段缺失和 `null` 一样”

字典缺键与键值为 `None` 可被区分；更新配置或 API PATCH 等场景往往具有不同含义。

### 26.9 “`default=str` 是万能可靠序列化”

它可能产生不可还原或不符合协议的字符串，掩盖意外类型。正式接口应写明确规则。

### 26.10 “重复键会自动报错”

默认常保留后值。严格需求应配置检查键值对的钩子。

### 26.11 “`sort_keys=True` 就是标准化签名”

排序只是规范化的一部分。真正签名需要双方明确且一致的完整规范。

### 26.12 “`load()` 天然逐条解析大数组”

接收文件对象不代表以记录为单位增量产出。大型数据要重新考虑格式和工具。

### 26.13 “反复 `dump()` 自动形成数组”

它不会添加数组括号和逗号。使用完整列表、明确数组写入逻辑或 JSON Lines。

### 26.14 “`NaN` 是严格合法 JSON 数字”

标准库默认宽松支持不等于格式标准允许。严格生成与解析需要相应约束。

### 26.15 “元组、集合和日期都能原样往返”

JSON 不保留任意 Python 类型。元组会按数组处理，集合和日期默认需要自定义转换。

### 26.16 “解析再编码会保留原始字节”

布局、转义、顺序和数字表示可能变化。需要原始证据时应另外保留原文件。

## 第二十七章：带参考答案的练习

### 27.1 练习一：写出四个核心函数的方向

请在不运行代码前，判断各变量类型：

```python
import json
from io import StringIO

obj = {"count": 3, "ready": True}
text = json.dumps(obj)
restored = json.loads(text)

with StringIO() as stream:
    result = json.dump(obj, stream)
    stream.seek(0)
    from_stream = json.load(stream)
```

`text` 是字符串；`restored` 与 `from_stream` 是字典；`result` 通常是 `None`。`stream.seek(0)` 必不可少，因为前面的写入把位置推进到了末尾。

### 27.2 练习二：保留中文并采用两格缩进

```python
import json

obj = {"title": "学习记录", "items": ["文件", "路径"]}
text = json.dumps(obj, ensure_ascii=False, indent=2, allow_nan=False)
print(text)
```

`indent` 决定布局，`ensure_ascii` 决定非 ASCII 文本表现，`allow_nan` 限制特殊浮点输出。这几个参数分别处理不同问题，不应当作一组没有含义的固定咒语。

### 27.3 练习三：拒绝布尔值充当数量

```python
import json

obj = json.loads('{"count": true}')
count = obj.get("count")

if type(count) is not int or count < 0:
    raise ValueError("count 必须是非负整数")
```

这段程序应该拒绝输入。不要为了让示例“看起来能成功”改成 `isinstance(count, int)` 而漏掉布尔子类关系。

### 27.4 练习四：区分缺失与明确空值

```python
missing = object()
obj = {"note": None}

value = obj.get("note", missing)

if value is missing:
    print("未提供 note")
elif value is None:
    print("明确提供了 null")
else:
    print("实际内容：", value)
```

`object()` 创建一个本次代码独有的哨兵对象，可以区分所有正常 JSON 值，包括 `None`。比较身份使用 `is`，不是把哨兵打印成字符串再比较。

### 27.5 练习五：拒绝重复键

```python
import json


def no_duplicates(pairs):
    result = {}
    for key, value in pairs:
        if key in result:
            raise ValueError(f"重复字段：{key}")
        result[key] = value
    return result


json.loads('{"id":1,"id":2}', object_pairs_hook=no_duplicates)
```

程序应抛 `ValueError`，而不是返回某个悄悄覆盖后的字典。这里抛出的错误来自自定义规则，不一定具有 `JSONDecodeError` 的行列属性。

### 27.6 练习六：把日期转换约定写明白

```python
import json
from datetime import datetime, timezone

obj = {
    "created_at": datetime(2026, 11, 2, 12, 0, tzinfo=timezone.utc).isoformat()
}

text = json.dumps(obj)
restored = json.loads(text)
print(type(restored["created_at"]))
```

恢复后仍是字符串，因为 JSON 不会根据字符串长得像日期就自动创建 `datetime`。需要调用者按字段契约解析，并处理时区和非法格式。

### 27.7 练习七：观察双重编码

```python
import json

obj = {"name": "小林"}
text = json.dumps(obj, ensure_ascii=False)
wrapped = json.dumps(text, ensure_ascii=False)

print(type(json.loads(text)).__name__)
print(type(json.loads(wrapped)).__name__)
```

分别得到 `dict` 和 `str`。如果接口本来期望对象，你不能把已经编码过的字符串继续当成对象交给自动 JSON 参数。

### 27.8 练习八：比较字节与字符长度

```python
import json

text = json.dumps({"message": "你好"}, ensure_ascii=False, separators=(",", ":"))
payload = text.encode("utf-8")

print(len(text))
print(len(payload))
```

两个汉字在 UTF-8 中占更多字节，因此两种长度通常不同。设置请求体上限、文件大小上限时，要明确单位是字节还是码点数量。

### 27.9 练习九：让损坏文件不覆盖旧配置

步骤是先读取旧配置，在内存里修改，完成验证和序列化，再写临时文件并替换。

可以在练习函数中故意加入一个不支持的 `set`，确认编码失败发生在正式文件替换之前。然后重新读取正式文件，检查旧内容仍然存在。

这项实验验证的是保存顺序和失败边界，不能通过仅断言 `dumps()` 返回字符串来代替。

### 27.10 练习十：用逐行格式处理记录

准备三行输入：一行合法对象，一行损坏 JSON，一行合法对象。程序应处理两条合法记录，并在错误报告中保留第二行的位置。

然后再写入一条包含字段内部换行的记录，使用 `dumps()` 输出，确认它仍然只占一条物理记录行。

这项实验帮助你理解 JSON 字符串内的转义换行与文件记录分隔的真实换行之间的差别。

## 第二十八章：为一份 JSON 接口写清楚数据契约

掌握函数调用之后，下一步通常不是寻找更多神奇参数，而是把发送方和接收方的约定写清楚。很多实际错误来自双方对同一字段的理解不同。

### 28.1 类型之外，还要有范围与单位

`{"timeout": 30}` 只告诉你有一个数字。它没有说明三十秒、三十毫秒还是三十分钟，也没有说明零是否表示立即超时或永不超时。

更清楚的字段名可以是 `timeout_seconds`，并说明允许范围。存储文件大小时用 `size_bytes`，不要让接收方猜测它是十进制 kB 还是二进制 KiB。

类型检查只能验证“看起来是个数”，无法自动消除单位歧义。

### 28.2 字符串需要说明规范

日期字符串要说明是否必须带时区；ID 要说明大小写敏感与否；文件路径要说明相对哪个基准；枚举值要列出允许项；空字符串是否有效也需要决定。

例如，`"created_at":"2026-11-02T12:00:00+00:00"` 比一个没有时区的本地时间更容易跨地区解释，但接收方仍需要按协议解析。

### 28.3 未知字段政策

严格拒绝未知字段能尽早发现拼写错误，例如把 `page_size` 写成 `pagesize`。但它也可能让新旧版本之间的兼容更困难。

忽略未知字段有利于向前兼容，却可能掩盖用户错误。选择哪种政策，应根据配置文件、公开 API 或内部事件的用途决定，并保持实现一致。

不要一个入口拒绝未知字段，另一个入口自动丢弃，最后却没有说明两种数据的契约不同。

### 28.4 版本字段与迁移

数据格式会变化。可以在顶层加入明确版本，解析后根据版本选择对应校验和迁移规则。

```json
{
  "schema_version": 1,
  "items": []
}
```

版本是你自己协议中的业务字段，不是 `json` 模块自动理解的特殊标记。加载器仍要检查它是否存在、类型是否正确、版本是否支持。

迁移应把旧模型转换成明确的新模型，再验证结果。重要资料应保留原始副本和迁移记录，不要直接在唯一原文件上试验未验证转换。

### 28.5 错误信息应帮助定位

“数据错误”往往太笼统。可以报告文件、记录索引、字段路径和规则，例如 `items[3].score 必须在 0 到 100 之间`。

但错误信息也要避免泄露不应展示的完整秘密字段。定位需要的是字段位置和规则，不一定需要把整份输入原样贴出来。

## 第二十九章：接口速查与学习顺序

| 需求 | 接口或组合 | 返回值与重点 |
| --- | --- | --- |
| 对象变 JSON 文本 | `json.dumps(obj)` | 返回 `str` |
| JSON 文本变对象 | `json.loads(text)` | 返回对应 Python 值 |
| 对象写入文本流 | `json.dump(obj, fp)` | 通常返回 `None` |
| 从流读取 JSON | `json.load(fp)` | 不自动按业务记录增量产出 |
| 中文可读输出 | `ensure_ascii=False` | 控制字符表现，不设置字节编码 |
| 美化布局 | `indent=2` | 增加缩进和换行 |
| 紧凑输出 | `separators=(",", ":")` | 删除结构分隔处的额外空白 |
| 按键排序 | `sort_keys=True` | 便于比较，不等于完整规范化 |
| 拒绝生成 NaN | `allow_nan=False` | 非有限数值导致错误 |
| 扩展未知类型 | `default=callback` | 返回可编码结构，未知类型应明确报错 |
| 统一扩展编码器 | `cls=EncoderClass` | 适合集中管理转换政策 |
| 十进制小数解析 | `parse_float=Decimal` | 编码回去的政策还需设计 |
| 定制整数解析 | `parse_int=callback` | 只处理对应数字入口 |
| 拒绝特殊数字输入 | `parse_constant=callback` | 不代替全部数值范围检查 |
| 对象字典后处理 | `object_hook=callback` | 可能改变默认返回类型 |
| 检查重复键 | `object_pairs_hook=callback` | 构造字典前检查键值对 |
| 定位语法问题 | `JSONDecodeError` | `msg`、`lineno`、`colno`、`pos` |
| 逐段生成文本 | `JSONEncoder.iterencode()` | 每段不保证独立 JSON 语义 |
| 解析前缀并取结束点 | `JSONDecoder.raw_decode()` | 还需调用者组织协议边界 |
| 命令行检查 | `python -m json.tool` | 验证语法并格式化 |

建议先掌握四个核心函数，再熟悉类型映射、UTF-8 文件搭配和结构校验。随后学习错误定位、严格解析和自定义类型转换。最后根据真实需求选择 JSON Lines、增量处理、版本迁移和保存策略。

你不必第一次读完就背下每个钩子。更重要的是在遇到需求时知道它属于哪一层：原始字节、JSON 语法、Python 容器、业务模式，还是文件发布与并发。

配套阅读：[Python-open文件处理.md](Python-open文件处理.md)、[Python-pathlib库.md](Python-pathlib库.md)。前者解释文本流和可靠保存，后者解释路径与目录管理；本篇解释流中的结构化数据如何编码、解析和验证。

官方参考：[Python json 接口](https://docs.python.org/3.14/library/json.html)、[RFC 8259 格式标准](https://www.rfc-editor.org/rfc/rfc8259)、[Python 命令行 JSON 工具](https://docs.python.org/3.14/library/json.html#command-line-interface)。教学代码与业务模型为本篇独立编写，示例中的字段范围和错误政策只属于相应练习，应按实际任务调整。
