# Python `collections` 库超详细教程：从计数、分组和队列，到有名字的记录与分层配置

## 零基础先导：为什么普通 `list` 和 `dict` 之外还需要工具？

`collections` 是 collection（集合、容器）相关工具的标准库模块；它不是替换所有列表和字典，而是针对经常重复出现的**容器操作模式**。

### 先自己写，再看库帮你省掉什么

任务：统计一段文本中每个单词出现几次。你可以写：

```python
words = ["python", "os", "python"]
counts = {}
for word in words:
    counts[word] = counts.get(word, 0) + 1  # 若不存在先当作 0
print(counts)  # {'python': 2, 'os': 1}
```

这就是 `Counter` 试图表达的工作：

```python
from collections import Counter

counts = Counter(["python", "os", "python"])  # 统计每个元素的出现次数
print(counts["python"])   # 2
print(counts["none"])     # 0：访问未出现的计数项时得到 0
```

不要把 `Counter` 理解成数据库或普通数学集合；它主要是**键 → 计数**的映射。

同样，`defaultdict(list)` 常用于“按类别收集元素”，`deque` 是适合两端追加/取出的双端队列，`namedtuple` 帮你给元组字段起名字，`ChainMap` 则让多层映射按优先次序查找。学习每个工具时先问：“如果只用 dict/list，我会反复写哪几行？”若没有这样的重复，继续使用普通容器完全合理。

---

假设你已经会使用列表和字典。

现在，你开始写一点实际程序。

需要统计文件扩展名出现了几次。

需要把学生按照班级分组。

需要保存最近二十条任务记录。

需要让命令行配置覆盖环境配置，再由默认配置兜底。

这些任务都能使用普通 `dict`、`list` 完成。

但写着写着，你会发现自己反复处理相同的细节：不存在的键先初始化，队列头部取出元素，维护固定长度，记录元组中哪个位置表示姓名。

`collections` 就提供了一些有明确用途的容器，帮助你把这些意图写清楚。

本篇面向 Python 3.11 及以上，新增版本差异会单独说明。

同一小节的短例子可以接着执行，变量沿用本小节先前的定义；标为完整程序的代码则包含自己的导入和输入。预期报错的反面例子只用于观察，不应和正常流程混在一起连续执行。

所有文件示例都使用练习目录或临时目录，不操作真实业务数据。

## 第一章：`collections` 到底是什么？

### 1.1 为什么已经有字典和列表，还要这个模块？

先看普通字典计数。

```python
words = ["python", "file", "python", "path", "file", "python"]
counts = {}

for word in words:
    if word not in counts:
        counts[word] = 0
    counts[word] += 1

print(counts)
```

输出：

```text
{'python': 3, 'file': 2, 'path': 1}
```

程序没有问题。

但你写了两种不同层次的逻辑。

一层是“我要数次数”。

另一层是“第一次遇到这个键，需要先放一个零”。

如果你的数据任务很多，后一层会不断重复。

使用专门的计数容器：

```python
from collections import Counter

words = ["python", "file", "python", "path", "file", "python"]
counts = Counter(words)

print(counts)
```

得到的计数含义相同。

容器把常见计数规则组织好了，你的代码可以直接表达“根据这些元素建立计数”。

### 1.2 名称怎样理解？

`collection` 可以理解为集合起来的一组对象。

这里的 `collections` 是模块名，不是特指 Python 的 `set` 集合类型。

它包含多个容器工具，各自面向不同需求。

`Counter` 面向次数。

`defaultdict` 面向缺失键的默认创建。

`deque` 面向两端快速加入和移除。

`ChainMap` 面向多个映射的联合查找。

`namedtuple` 面向有字段名称的轻量记录。

`OrderedDict` 面向需要主动重排顺序的映射。

不必为了“用了高级库”把所有列表都替换成 `deque`，也不必把所有字典都变成 `defaultdict`。

先看你需要哪种行为，再选择容器。

### 1.3 需要安装吗？

不需要。

它属于标准库：

```python
from collections import Counter, defaultdict, deque
```

这里导入了三个名称。

你也可以：

```python
import collections

counts = collections.Counter("banana")
```

两种写法都是正常的导入方式。

日常代码经常直接导入所需的类，避免每次写模块前缀。

## 第二章：`Counter`——把“出现多少次”作为容器的核心含义

### 2.1 传一个可迭代对象，会统计什么？

```python
from collections import Counter

counts = Counter("banana")
print(counts)
```

输出：

```text
Counter({'a': 3, 'n': 2, 'b': 1})
```

字符串可迭代，每轮产生一个字符。

因此，这里统计的是字符，不是整段字符串作为一个词出现一次。

如果你要统计单词，就先提供单词序列：

```python
text = "python file python path file"
counts = Counter(text.split())
print(counts)
```

`split()` 把文本分成单词列表。

`Counter` 统计这个列表的元素。

它不会自动知道英语分词、中文分词或标点规则。

### 2.2 如果我要统计整个文件名呢？

```python
names = ["a.txt", "b.txt", "a.txt"]
counts = Counter(names)
print(counts["a.txt"])
```

输出：

```text
2
```

每个列表元素就是一个计数对象。

这说明计数粒度由你交给它的迭代元素决定。

不要把 `Counter("a.txt")` 和 `Counter(["a.txt"])` 混为一谈。

### 2.3 访问没有出现过的元素，会怎样？

```python
from collections import Counter

counts = Counter(["apple", "apple", "banana"])

print(counts["orange"])
print("orange" in counts)
```

输出：

```text
0
False
```

下标访问缺失键得到零。

这次读取不会自动把 `orange` 插入计数器。

这和稍后要讲的 `defaultdict` 很不一样。

### 2.4 `counts.get("orange")` 也返回零吗？

不一定。

```python
print(counts.get("orange"))
print(counts.get("orange", 0))
```

分别得到：

```text
None
0
```

`Counter` 的缺失键规则由下标访问触发，不能把每一种字典方法都想成自动采用相同默认行为。

需要 `get()` 也返回零，就明确提供默认值。

### 2.5 怎样手动增加次数？

```python
from collections import Counter

counts = Counter()
counts["python"] += 1
counts["python"] += 1
counts["file"] += 1

print(counts)
```

第一次 `counts["python"]` 缺失时取得零。

再加一，写回结果。

于是你不需要先写“如果键不存在就初始化”。

### 2.6 元素必须满足什么条件？

`Counter` 的元素是字典键，因此必须可哈希。

字符串、整数和元素可哈希的元组都很常见。

列表不能直接作为键：

```python
# 预期 TypeError：列表不可哈希。
Counter([[1, 2], [1, 2]])
```

如果你要统计坐标，可以转成元组：

```python
points = [(1, 2), (1, 2), (3, 4)]
print(Counter(points))
```

把数据变成可哈希结构时，也要确认这个变换确实保留了你要比较的含义。

## 第三章：`Counter` 的更新、减法和最常见项

### 3.1 `update()` 是覆盖还是累加？

普通字典的 `update()` 会更新键对应的值。

`Counter.update()` 则用于增加计数。

```python
from collections import Counter

counts = Counter({"apple": 2, "banana": 1})
counts.update({"apple": 3, "orange": 2})

print(counts)
```

`apple` 得到五，不是三。

传入映射时，映射的值被理解为要增加的次数。

传入普通可迭代对象时，则每个元素增加一次。

```python
counts.update(["apple", "banana"])
```

这就是同一个接口根据输入结构体现的两种常见用法。

### 3.2 如果传入字符串，会怎样？

```python
counts = Counter()
counts.update("apple")
print(counts)
```

统计的是字母。

如果你想让单词 `apple` 增加一次，应传 `["apple"]`，或直接 `counts["apple"] += 1`。

这个问题与构造时一样：先确认迭代元素的粒度。

### 3.3 `subtract()` 能得到负数吗？

可以。

```python
from collections import Counter

stock = Counter({"pen": 3, "book": 2})
stock.subtract({"pen": 5, "book": 1})

print(stock["pen"])
print(stock["book"])
```

输出：

```text
-2
1
```

负数可以表示短缺、差额或其他你规定的含义。

`Counter` 不会强制所有值为非负整数，也不会自动理解“库存不得为负”的业务限制。

### 3.4 `subtract()` 和减号为什么结果不同？

```python
from collections import Counter

first = Counter({"pen": 3, "book": 2})
second = Counter({"pen": 5, "book": 1})

print(first - second)

copy = first.copy()
copy.subtract(second)
print(copy)
```

减号生成的多重集合结果会去掉零和负数项。

`subtract()` 会保留这些计数。

因此，如果你在做“差额报告”，不应随意用减号替代 `subtract()`，否则短缺项可能消失。

### 3.5 `most_common()`：次数最多的元素

```python
from collections import Counter

counts = Counter(["A", "B", "A", "C", "B", "A"])
print(counts.most_common(2))
```

输出：

```text
[('A', 3), ('B', 2)]
```

返回列表，列表中每项是“元素、计数”元组。

可以解包：

```python
for word, count in counts.most_common(2):
    print(word, count)
```

次数相同的项按照首次遇到的顺序排列。

如果你需要“次数降序、名称升序”，就自己表达第二排序规则：

```python
ranked = sorted(counts.items(), key=lambda item: (-item[1], item[0]))
```

这里要求元素能按照你定义的键比较。

### 3.6 零计数项会自动删除吗？

不会。

```python
counts = Counter({"apple": 2})
counts["apple"] = 0

print("apple" in counts)
print(len(counts))
```

这个键仍然存在。

需要删除时：

```python
del counts["apple"]
```

只想得到正计数项，可以使用一元加号：

```python
positive = +Counter({"a": 2, "b": 0, "c": -1})
print(positive)
```

这里的加号是 `Counter` 定义的运算行为，不是把每个值都加一。

### 3.7 `elements()` 与 `total()`

```python
from collections import Counter

counts = Counter({"a": 2, "b": 1, "c": 0, "d": -2})

print(list(counts.elements()))
print(counts.total())
```

`elements()` 按正整数计数重复产生元素，忽略非正计数。

`total()` 则求计数值之和，负数也参与。

因此，上例产生的元素数量与 `total()` 不必相等。

`total()` 从 Python 3.10 起提供，本文基础环境可用。

## 第四章：把计数与实际文件处理连接起来

### 4.1 统计练习目录的最后扩展名

```python
from collections import Counter
from pathlib import Path

folder = Path("collections_practice")
counts = Counter()

for path in folder.rglob("*"):
    if path.is_file() and not path.is_symlink():
        suffix = path.suffix.casefold() or "[无扩展名]"
        counts[suffix] += 1

for suffix, count in counts.most_common():
    print(suffix, count)
```

`rglob()` 提供候选路径。

`is_file()` 确认普通文件。

排除符号链接是这个示例的统计政策。

`suffix` 只取最后后缀，所以 `archive.tar.gz` 按 `.gz` 统计。

需要按复合后缀分类时，应明确定义 `.tar.gz` 等规则。

### 4.2 为什么 `Counter` 不负责扫描错误？

它只负责你交给它的数据。

扫描缺失目录、没有权限、文件在扫描后被移走，都属于文件系统层的问题。

如果你需要完整扫描报告，应搭配 `Path.walk(on_error=...)` 或其他明确记录错误的遍历方式。

不能因为最后打印了一个漂亮的计数器，就断定输入收集完整。

### 4.3 统计对象相等的含义需要提前决定

`"TXT"` 与 `"txt"` 是不同字符串。

先 `.casefold()` 可以把大小写差异归一。

但对文件名、用户 ID、语言标记等数据，是否应该归一，应按业务规则决定。

同理，`Counter([1, True])` 不能期待得到两个完全独立的键，因为 Python 的相等和哈希规则会使这两者产生同键行为。

计数器沿用字典键的语义，不会为业务自动创造另一套身份规则。

## 第五章：`defaultdict`——第一次遇到某个键，应该自动准备什么？

### 5.1 先看普通分组程序

假设数据是：

```python
students = [
    ("一班", "小林"),
    ("二班", "小周"),
    ("一班", "小陈"),
]
```

我们希望：

```text
一班 -> 小林、小陈
二班 -> 小周
```

普通字典写法：

```python
groups = {}

for class_name, student_name in students:
    if class_name not in groups:
        groups[class_name] = []
    groups[class_name].append(student_name)
```

第一次遇到一班，先创建空列表。

以后再遇到一班，往已有列表继续添加。

### 5.2 用 `defaultdict(list)` 表达这个规则

```python
from collections import defaultdict

students = [
    ("一班", "小林"),
    ("二班", "小周"),
    ("一班", "小陈"),
]

groups = defaultdict(list)

for class_name, student_name in students:
    groups[class_name].append(student_name)

print(dict(groups))
```

输出：

```text
{'一班': ['小林', '小陈'], '二班': ['小周']}
```

`list` 在这里是一个可调用对象。

缺失键需要默认值时，容器调用 `list()`，得到一个新的空列表。

### 5.3 为什么传 `list`，不是 `list()`？

因为容器需要的是“以后需要时，调用谁来创建默认值”。

传 `list`，是把工厂交给它。

传 `list()`，已经提前执行，得到一个列表对象；这个对象不能作为相应的默认工厂调用。

下面的写法会出问题：

```python
# 预期 TypeError：默认工厂必须可调用或为 None。
defaultdict([])
```

理解“函数本身”和“调用结果”，能同时帮助你理解回调、装饰器和很多库参数。

### 5.4 每个班级会不会共享同一个列表？

正常 `defaultdict(list)` 不会。

每次遇到新的缺失键，分别调用一次 `list()`。

```python
from collections import defaultdict

groups = defaultdict(list)
groups["A"].append("小林")
groups["B"].append("小周")

print(groups["A"] is groups["B"])
```

输出：

```text
False
```

但是，如果你自己写工厂返回同一个共享列表，就会共享：

```python
shared = []
groups = defaultdict(lambda: shared)
```

这次不是 `defaultdict` 自动复制，而是你主动要求每个键取得同一对象。

### 5.5 访问缺失键，会修改容器吗？

会，这一点很重要。

```python
from collections import defaultdict

groups = defaultdict(list)

print("A" in groups)
value = groups["A"]
print("A" in groups)
```

输出：

```text
False
True
```

下标读取触发默认创建，并把结果保存进去。

如果你只想查询而不插入，使用 `get()` 或成员判断：

```python
value = groups.get("B")
```

`get()` 不会调用默认工厂，也不会把 `B` 插进去。

### 5.6 `defaultdict(int)` 为什么适合累加？

```python
from collections import defaultdict

totals = defaultdict(int)

for category, amount in [("book", 20), ("pen", 5), ("book", 30)]:
    totals[category] += amount

print(dict(totals))
```

`int()` 不提供参数时得到零。

因此，首次访问缺失分类时，自动准备零，再进行累加。

这里的值可以表示金额、数量或耗时，但容器不负责验证单位和数值范围。

### 5.7 `defaultdict(set)` 适合去重分组

```python
from collections import defaultdict

courses = defaultdict(set)

for student, course in [("小林", "Python"), ("小林", "Python"), ("小林", "SQL")]:
    courses[student].add(course)

print(sorted(courses["小林"]))
```

同一课程重复出现，不会重复保存。

集合迭代顺序不是业务顺序，需要稳定显示时再排序。

如果你要保留首次出现顺序，又要去重，应使用其他明确结构，不能指望集合同时解决所有要求。

## 第六章：默认工厂、嵌套分组与 `setdefault()`

### 6.1 默认值想设成一百，怎么办？

工厂需要无参数调用，所以可以使用 `lambda`：

```python
from collections import defaultdict

scores = defaultdict(lambda: 100)
print(scores["new_student"])
```

这只是一个默认创建规则，不表示所有学生永远固定一百分。

之后仍然可以正常赋值。

### 6.2 如果默认值需要一个字典呢？

```python
stats = defaultdict(lambda: {"count": 0, "total": 0})

stats["A"]["count"] += 1
stats["A"]["total"] += 95
```

每次调用工厂都会构造新字典，因此不同分类拥有各自统计状态。

如果你需要再按日期分组，可以继续组合，但嵌套太深时应考虑命名函数或数据类，提高可读性。

### 6.3 命名工厂往往更容易调试

```python
from collections import defaultdict


def new_stats():
    return {"count": 0, "total": 0}


stats = defaultdict(new_stats)
```

看到 `new_stats`，就知道缺失分类会创建什么。

有些序列化机制对匿名函数和可导入函数有不同支持，也需要注意。

标准 JSON 保存时，通常应输出普通结构，不把工厂本身当作数据。

### 6.4 `setdefault()` 也能分组，怎样选？

```python
groups = {}

for class_name, student_name in [("A", "小林"), ("A", "小周")]:
    groups.setdefault(class_name, []).append(student_name)
```

`setdefault()` 在键不存在时插入指定默认值，返回对应值。

这一写法很适合局部简单分组。

但 `[]` 这个表达式每次调用前都会求值，即使该键已经存在，通常仍然先创建一个暂时不用的新列表。

`defaultdict` 的工厂在真正缺失时才调用，更适合持续采用同一默认策略的容器。

选择时也看代码读者是否容易理解，不必把两者绝对分出高低。

### 6.5 默认工厂抛异常会怎样？

异常会传播，不能理解为容器一定能给你一个默认值。

```python
from collections import defaultdict


def fail():
    raise RuntimeError("无法创建默认值")


values = defaultdict(fail)
```

调用 `values["x"]` 时会触发工厂，并抛出错误。

这也提醒我们：读取 `defaultdict` 缺失键可能有副作用，甚至执行你自己提供的复杂逻辑。

不要把它当成完全无状态的普通查询。

### 6.6 `dict(groups)` 是深拷贝吗？

不是。

它创建普通字典，但内部列表等对象仍然共享。

```python
from collections import defaultdict

groups = defaultdict(list)
groups["A"].append("小林")
plain = dict(groups)
plain["A"].append("小周")

print(groups["A"])
```

原容器也能看到新学生。

如果你希望隔离每组列表，可以主动复制各组：

```python
plain = {key: list(value) for key, value in groups.items()}
```

若元素本身还有嵌套可变对象，还需继续决定需要多深的隔离。

## 第七章：`deque`——为什么队列不总用列表？

### 7.1 先看一个排队取任务的情境

任务到达顺序是：

```text
下载资料
解析文本
生成报告
```

你希望先来的先执行。

使用列表可以写：

```python
tasks = ["下载资料", "解析文本", "生成报告"]
first = tasks.pop(0)
print(first)
print(tasks)
```

含义没有问题。

但列表底层适合按索引访问和尾部追加。

删除第零项时，后续元素通常需要移动位置。

如果你不断从队列头部取任务，列表不是最贴近这个工作方式的选择。

### 7.2 使用双端队列

```python
from collections import deque

tasks = deque(["下载资料", "解析文本", "生成报告"])
first = tasks.popleft()

print(first)
print(list(tasks))
```

输出：

```text
下载资料
['解析文本', '生成报告']
```

`deque` 来自 double-ended queue，也就是双端队列。

它适合在两端加入和移除元素。

`popleft()` 明确表达从左端取出。

### 7.3 四个最先掌握的方法

```python
from collections import deque

q = deque(["B", "C"])
q.append("D")
q.appendleft("A")

print(list(q))
print(q.pop())
print(q.popleft())
print(list(q))
```

输出：

```text
['A', 'B', 'C', 'D']
D
A
['B', 'C']
```

右端使用 `append()`、`pop()`。

左端使用 `appendleft()`、`popleft()`。

把队列方向固定下来，程序就容易理解。

### 7.4 空队列继续取，会怎样？

```python
from collections import deque

q = deque()
# 预期 IndexError：没有可取出的元素。
q.popleft()
```

单线程普通循环可以：

```python
while q:
    task = q.popleft()
    print(task)
```

但多个线程共享队列时，检查之后状态可能变化。

即使某些端点操作具有线程安全保证，多个操作组合起来也不会自动形成事务。

需要阻塞等待、任务完成通知等协作，通常应使用 `queue.Queue` 等专用同步工具。

### 7.5 `deque` 能通过索引读取吗？

可以。

```python
q = deque([10, 20, 30, 40])
print(q[0])
print(q[-1])
```

两端访问很适合。

中间随机访问的代价不像列表那样稳定为常数时间。

`deque` 也不直接提供列表那样的切片操作。

如果任务主要是随机索引和切片，列表往往更自然。

## 第八章：固定长度历史、旋转和批量加入

### 8.1 只保存最近三条记录

```python
from collections import deque

history = deque(maxlen=3)

for event in ["A", "B", "C", "D", "E"]:
    history.append(event)
    print(list(history))
```

输出：

```text
['A']
['A', 'B']
['A', 'B', 'C']
['B', 'C', 'D']
['C', 'D', 'E']
```

容量满后，在右端加入新项，会从左端自动丢弃最旧项。

这非常适合最近历史和有限窗口。

但“自动丢弃”也意味着它不适合要求一条任务都不能丢的待执行工作队列。

### 8.2 如果从左端加入，丢哪一边？

满容量时从左加入，就从另一端丢弃。

```python
q = deque([1, 2, 3], maxlen=3)
q.appendleft(0)
print(list(q))
```

结果是：

```text
[0, 1, 2]
```

规则与加入方向相配合。

### 8.3 `maxlen` 能随时改吗？

这个容量是创建时指定的属性，不是普通可写配置。

需要另一个容量时，通常构造新的 `deque`：

```python
larger = deque(q, maxlen=10)
```

构造新容器不会自动复制每个元素本身。

元素如果是可变对象，依然可能共享。

### 8.4 `extendleft()` 为什么顺序看起来反了？

```python
from collections import deque

q = deque(["D"])
q.extendleft(["A", "B", "C"])
print(list(q))
```

输出：

```text
['C', 'B', 'A', 'D']
```

可以按执行过程理解。

先把 A 放到左边。

再把 B 放到新的左边。

最后把 C 放到新的左边。

所以输入序列的相对顺序反转。

希望保持 A、B、C 在左边的顺序，可以先反向提供：

```python
q = deque(["D"])
q.extendleft(reversed(["A", "B", "C"]))
print(list(q))
```

### 8.5 `rotate()` 是怎么转的？

```python
q = deque(["A", "B", "C", "D"])
q.rotate(1)
print(list(q))
q.rotate(-2)
print(list(q))
```

正数向右旋转，右端元素依次移到左端。

负数向左旋转。

可以用于轮流处理多个来源、简单循环调度等。

它不会复制出更多元素，也不自动执行任务。

### 8.6 满容量时 `insert()` 会静默丢旧项吗？

不要把端点追加规则推广到所有方法。

有界 `deque` 已满时，`insert()` 插入可能抛出 `IndexError`。

如果需要稳定管理窗口，优先使用明确的端点操作，不要把队列当成随意插入的列表使用。

### 8.7 能在遍历时修改队列吗？

遍历过程中修改结构可能触发运行时错误，或让逻辑不可靠。

如果要边取边处理，就明确写消费循环：

```python
while q:
    item = q.popleft()
    print(item)
```

如果要遍历原快照，先保存你需要的副本。

“容器支持迭代”和“允许迭代时任意修改”是两个问题。

### 8.8 查找、删除和反转，也要分清是否原地修改

```python
from collections import deque

q = deque(["A", "B", "A", "C"])
print(q.count("A"))
print(q.index("B"))

q.remove("A")
print(list(q))

result = q.reverse()
print(list(q))
print(result)
```

`count()` 统计相等元素出现次数。

`index()` 返回第一次匹配的位置，找不到时抛出 `ValueError`。

`remove()` 删除第一个相等元素，不会一次删除全部匹配项。

`reverse()` 原地反转，并返回 `None`。

因此，不要写 `q = q.reverse()`，否则变量会变成 `None`。

这种“原地修改方法返回 None”的习惯，在列表等容器中也经常见到。

如果只是想反向遍历，不修改队列，可以使用：

```python
for item in reversed(q):
    print(item)
```

如果想得到反向列表，则进一步 `list(reversed(q))`。

这三种需求分别是修改容器、产生反向迭代器和收集反向结果，不必混在一起记忆。

## 第九章：`ChainMap`——多层配置怎样按优先级查找？

### 9.1 一个很常见的配置需求

默认配置：

```python
defaults = {"host": "127.0.0.1", "port": 8000, "debug": False}
```

用户配置：

```python
user = {"port": 9000}
```

本次命令行参数：

```python
command_line = {"debug": True}
```

我们希望优先使用命令行，其次用户配置，最后默认值。

### 9.2 把查找层叠起来

```python
from collections import ChainMap

defaults = {"host": "127.0.0.1", "port": 8000, "debug": False}
user = {"port": 9000}
command_line = {"debug": True}

config = ChainMap(command_line, user, defaults)

print(config["host"])
print(config["port"])
print(config["debug"])
```

输出：

```text
127.0.0.1
9000
True
```

查找从最左层开始，找到同名键就停止。

`ChainMap` 可以按“连起来的一组映射”理解。

### 9.3 它会复制所有字典吗？

不会。

底层映射以引用方式保留。

```python
user["port"] = 9100
print(config["port"])
```

你会看到新值。

这是一种动态视图，不是创建时的永久配置快照。

### 9.4 修改 `config["port"]` 会写到哪里？

默认只写第一层。

```python
config["port"] = 9200

print(command_line)
print(user)
```

即使原值来自第二层，赋值也不会自动“找到原来源再修改”。

它在第一层写入新覆盖值。

这对临时覆盖很方便，但使用前必须理解写入方向。

### 9.5 删除一个只存在于后层的键呢？

默认删除也只操作第一层。

如果键不在第一层，即使后层存在，也可能得到 `KeyError`。

不要把联合查找误认为所有写操作都会遍历全部层。

需要修改用户配置时，应明确操作用户字典，而不是猜测 `ChainMap` 会替你找到它。

### 9.6 为什么普通字典合并有时更简单？

```python
snapshot = defaults | user | command_line
```

这个表达式生成一份普通字典，右侧同名字段覆盖左侧。

适合本次读取后就希望固定配置的场景。

`ChainMap` 适合持续保留多层关系和动态更新。

它们解决的侧重点不同：一次合并快照，或多层查找视图。

### 9.7 `new_child()` 和 `parents`

```python
from collections import ChainMap

base = ChainMap({"theme": "light"})
local = base.new_child({"theme": "dark", "temporary": True})

print(local["theme"])
print(local.parents["theme"])
```

`new_child()` 在最前面增加新层，适合局部上下文覆盖。

`parents` 取得跳过第一层的链视图。

这与作用域查找很相似，但它本身并不修改 Python 的真实变量作用域。

### 9.8 迭代顺序为什么不直接按查找优先级？

`ChainMap` 的键迭代有自己的规则，按后层到前层的方式组织首次键位置。

而取值仍按前层优先查找。

因此，展示配置时不要靠迭代顺序暗示来源优先级。

需要解释每个值来自哪一层时，主动记录来源，比单独打印 `dict(config)` 更清楚。

## 第十章：`namedtuple`——元组里的第零项，到底是什么？

### 10.1 普通元组的问题

```python
student = ("小林", 20, "一班")
print(student[0])
print(student[2])
```

程序可以运行，但读者需要记住每个位置的含义。

数据一多，`row[2]` 很容易看不出是在访问班级还是分数。

### 10.2 给元组字段取名字

```python
from collections import namedtuple

Student = namedtuple("Student", ["name", "age", "class_name"])
student = Student("小林", 20, "一班")

print(student.name)
print(student.class_name)
print(student[0])
```

`namedtuple()` 是一个工厂函数，它返回一个新的元组子类。

这里 `Student` 保存这个类型。

`student` 才是创建出来的实例。

字段既能按名字访问，也保留元组的位置访问和解包能力。

### 10.3 第一个字符串必须与变量名相同吗？

通常应保持一致，便于显示和调试。

```python
Student = namedtuple("Student", "name age class_name")
```

字段也可以使用空白分隔字符串定义。

类型名称影响对象显示和相关类型信息，变量名则是当前作用域的绑定，两者概念上不同。

### 10.4 能修改字段吗？

元组本身不可变。

```python
student = Student("小林", 20, "一班")
# 预期 AttributeError：字段不能这样赋值。
student.age = 21
```

需要修改后的一份记录，可以：

```python
older = student._replace(age=21)
```

它返回新对象，不修改原实例。

### 10.5 字段里放列表，列表也不可变吗？

不会。

```python
from collections import namedtuple

Group = namedtuple("Group", "name members")
group = Group("A", ["小林"])
group.members.append("小周")
print(group)
```

元组固定的是字段引用，字段引用的列表仍然可以修改。

因此，不可变容器不等于整个对象图都不可变。

### 10.6 `_asdict()`、`_make()` 和 `_fields`

```python
Student = namedtuple("Student", "name age class_name")
student = Student._make(["小林", 20, "一班"])

print(student._asdict())
print(Student._fields)
```

`_make()` 根据已有可迭代记录构造实例。

`_asdict()` 返回字段名到值的普通字典。

`_fields` 提供字段名称元组。

虽然名称以下划线开头，这些是文档明确提供的公共辅助成员，不是让你直接修改的内部存储。

### 10.7 默认值怎么提供？

```python
Student = namedtuple(
    "Student",
    "name age class_name",
    defaults=[18, "未分班"],
)

print(Student("小林"))
```

默认值对应最右侧字段。

这里姓名必填，年龄和班级可以省略。

不要把可变列表当作共享默认值，除非你明确想共享；`namedtuple` 不提供数据类那种逐实例调用的 `default_factory` 参数。

### 10.8 和 `dataclass`、`typing.NamedTuple` 怎么选择？

`namedtuple` 适合轻量、固定字段、仍需要元组行为的记录。

`typing.NamedTuple` 使用类语法加类型注解，适合想明确字段类型的同类需求。

`dataclass` 更适合有默认工厂、更多方法或可变业务对象的模型。

类型注解本身通常不在运行时自动校验数据，别把命名记录当成数据验证器。

### 10.9 字段名来自外部表头，碰到关键字怎么办？

假设数据文件有这些列名：

```text
name,class,name
```

`class` 是 Python 关键字，`name` 又重复出现。

直接把它们作为命名元组字段并不符合规则。

可以让工厂重命名非法或重复名称：

```python
from collections import namedtuple

Record = namedtuple("Record", ["name", "class", "name"], rename=True)
print(Record._fields)
```

字段通常变成：

```text
('name', '_1', '_2')
```

这让类型能够建立，但原始列名与新字段名的对应关系需要你另外解释。

如果业务要求保留原表头、发现重复列必须报告，那么自动重命名未必是合适政策。

使用 `rename=True` 的意思是接受工厂生成的替代名称，不是自动证明表头正确。

外部数据模型最好先明确字段规则，再决定是拒绝、映射还是重命名。

## 第十一章：`OrderedDict`——普通字典已经有序，为什么它还在？

### 11.1 有序字典的“有序”是什么意思？

现代 Python 普通字典保留插入顺序。

这意味着键第一次插入的位置会影响迭代顺序。

但“保留插入顺序”不等于提供全部高效重排操作。

`OrderedDict` 在需要主动移动元素和从特定一端弹出项目时仍有用途。

### 11.2 `move_to_end()`：把已有键移到末尾

```python
from collections import OrderedDict

values = OrderedDict([("A", 1), ("B", 2), ("C", 3)])
values.move_to_end("A")

print(list(values))
```

输出：

```text
['B', 'C', 'A']
```

值没有改变，顺序改变。

如果希望移到开头：

```python
values.move_to_end("C", last=False)
```

### 11.3 `popitem(last=False)`：弹出最前面的项目

```python
key, value = values.popitem(last=False)
```

默认 `last=True` 则取末尾。

这为维护“最近使用顺序”提供了很方便的基础。

空容器调用会报错，需要按你的流程处理。

### 11.4 相等比较也有差别

两个 `OrderedDict` 相互比较时，顺序也参与相等判断。

与普通映射比较时，则采用相应映射比较规则。

不要把它概括为“任何情况下都按顺序比较”。

```python
from collections import OrderedDict

first = OrderedDict([("a", 1), ("b", 2)])
second = OrderedDict([("b", 2), ("a", 1)])

print(first == second)
print(first == dict(second))
```

常见结果分别为假和真。

### 11.5 只是保存配置字段，需要换成它吗？

通常没有必要。

普通字典已经满足很多字段顺序展示需求。

你需要移动、弹出特定端项目或特定顺序比较语义时，再选择 `OrderedDict`。

不要把存在于标准库里的每个容器都当成普通字典的升级替代。

## 第十二章：`UserDict`、`UserList`、`UserString` 和抽象接口

### 12.1 为什么会有包装类？

你可能想定义一个“自动规范化键的字典”或“只接收特定元素的列表”。

直接继承内置类型时，某些内置方法的内部路径不一定按你预期调用覆盖方法。

`UserDict` 等包装类通过一个明确的内部数据对象组织行为，让很多自定义容器更容易实现和理解。

### 12.2 一个规范化字符串键的简单映射

```python
from collections import UserDict


class LowerKeyDict(UserDict):
    def __setitem__(self, key, value):
        if not isinstance(key, str):
            raise TypeError("键必须是字符串")
        super().__setitem__(key.casefold(), value)

    def __getitem__(self, key):
        if not isinstance(key, str):
            raise TypeError("键必须是字符串")
        return super().__getitem__(key.casefold())

    def __delitem__(self, key):
        if not isinstance(key, str):
            raise TypeError("键必须是字符串")
        super().__delitem__(key.casefold())


values = LowerKeyDict()
values["Name"] = "小林"
print(values["NAME"])
```

这个例子只展示自定义行为思路。

完整容器还应检查成员判断、更新、复制等操作是否都符合你的规范化政策，不能只改两个方法就宣称任何接口都完全一致。

### 12.3 `collections.abc` 是另一个常见入口

```python
from collections.abc import Mapping, Iterable
```

这些名称用于表达抽象容器接口。

`Mapping` 关注键值映射能力。

`Iterable` 关注可迭代能力。

它们与 `Counter` 这种具体容器不同，不能把 `Mapping` 当成自动拥有业务数据的字典实例。

### 12.4 判断可迭代，`isinstance(..., Iterable)` 一定覆盖所有情况吗？

它能识别常见实现了相应接口的对象，但 Python 还存在通过旧式序列协议迭代的对象。

如果你真正需要执行迭代，尝试 `iter(obj)` 并处理适当异常，往往更直接表达需求。

接口判断和实际操作能力虽然相关，仍应按具体协议理解。

### 12.5 抽象接口不验证元素类型

一个对象是 `Mapping`，不说明键一定是字符串，也不说明值一定是整数。

一个对象可迭代，不说明它是有限序列，也不说明每次可以重新从头迭代。

因此，容器协议检查与业务数据校验仍然需要分开。

## 第十三章：完整实践一——把阅读日志变成一份统计报告

### 13.1 我们要处理什么？

假设你每天记录学习条目。

每条记录包含姓名、主题和分钟数。

现在想知道：哪个主题出现最多，每个人学了多少分钟，每个人接触了哪些主题，以及哪些记录有问题。

这个任务同时需要计数、累计和去重分组。

输入示例：

```python
records = [
    {"name": "小林", "topic": "Python", "minutes": 35},
    {"name": "小周", "topic": "SQL", "minutes": 40},
    {"name": "小林", "topic": "Python", "minutes": 25},
    {"name": "小林", "topic": "Linux", "minutes": 20},
]
```

### 13.2 先约定错误处理

姓名和主题必须是非空白字符串。

分钟数必须为一到一千四百四十之间的整数，不接收布尔值。

坏记录不计入统计，但保留索引和错误原因。

这些是练习的业务规则，不是 `Counter` 自动提供的规则。

### 13.3 完整程序

```python
from collections import Counter, defaultdict
import json


def validate_learning_record(record):
    if not isinstance(record, dict):
        raise ValueError("记录必须是字典")

    name = record.get("name")
    topic = record.get("topic")
    minutes = record.get("minutes")

    if not isinstance(name, str) or not name.strip():
        raise ValueError("name 必须是非空白字符串")
    if not isinstance(topic, str) or not topic.strip():
        raise ValueError("topic 必须是非空白字符串")
    if type(minutes) is not int or not 1 <= minutes <= 1440:
        raise ValueError("minutes 必须是 1 到 1440 的整数")

    return name.strip(), topic.strip(), minutes


def summarize_learning(records):
    topic_counts = Counter()
    minutes_by_person = defaultdict(int)
    topics_by_person = defaultdict(set)
    sessions_by_person = Counter()
    problems = []

    for index, record in enumerate(records):
        try:
            name, topic, minutes = validate_learning_record(record)
        except ValueError as error:
            problems.append({"index": index, "message": str(error)})
            continue

        topic_counts[topic] += 1
        minutes_by_person[name] += minutes
        topics_by_person[name].add(topic)
        sessions_by_person[name] += 1

    people = []
    for name in sorted(minutes_by_person):
        people.append({
            "name": name,
            "minutes": minutes_by_person[name],
            "sessions": sessions_by_person[name],
            "topics": sorted(topics_by_person[name]),
        })

    ranked_topics = [
        {"topic": topic, "sessions": count}
        for topic, count in sorted(
            topic_counts.items(),
            key=lambda item: (-item[1], item[0]),
        )
    ]

    return {
        "valid_sessions": topic_counts.total(),
        "invalid_sessions": len(problems),
        "topics": ranked_topics,
        "people": people,
        "problems": problems,
    }


def main():
    records = [
        {"name": "小林", "topic": "Python", "minutes": 35},
        {"name": "小周", "topic": "SQL", "minutes": 40},
        {"name": "小林", "topic": "Python", "minutes": 25},
        {"name": "小林", "topic": "Linux", "minutes": 20},
        {"name": "小周", "topic": "Python", "minutes": 30},
        {"name": "", "topic": "SQL", "minutes": 20},
        {"name": "小陈", "topic": "Linux", "minutes": True},
    ]

    report = summarize_learning(records)
    print(json.dumps(report, ensure_ascii=False, indent=2, allow_nan=False))


if __name__ == "__main__":
    main()
```

### 13.4 为什么用了两个 `Counter`？

主题次数与个人次数是两个不同统计维度。

`topic_counts[topic]` 回答某个主题有多少次学习记录。

`sessions_by_person[name]` 回答某个人有多少条有效学习记录。

即使两种值都是整数，也不应该塞进一个缺乏结构的字典，靠特殊字符串区分含义。

### 13.5 为什么分钟数使用 `defaultdict(int)`？

因为这是按键累计数值的常见模式。

`Counter` 也可以保存这样的累计值，但本例让 `Counter` 专门表达次数，让 `defaultdict(int)` 表达时长，变量语义更直接。

容器能力允许你做某件事，不表示所有含义都应挤到同一种类型里。

### 13.6 为什么主题集合输出前排序并变成列表？

集合帮助去重，但 JSON 默认没有集合类型。

输出报告时，`sorted(...)` 同时完成两件事：生成可序列化列表，以及确定显示顺序。

如果你需要保留首次接触主题的顺序，就应在收集阶段使用另一种保持顺序的去重策略，而不是最后排序。

### 13.7 这份报告应该检查什么？

有效记录数应为五。

小林累计八十分钟。

小周累计七十分钟。

Python 主题出现三次。

两个坏记录都应出现在问题列表里，不应悄悄计入有效数据。

这里明确使用 `type(minutes) is int`，所以布尔 `True` 不会因为是 `int` 子类就混入分钟数。

### 13.8 如果输入记录特别多，哪些部分还会增长？

主统计容器的大小主要随不同姓名和主题数量增长。

但 `problems` 会保存所有错误，如果每条都错，它就会随输入条数增长。

大型任务可以只保存前若干条错误，同时累计总错误数量，或把错误逐条写入独立文件。

不能只因为主循环逐条处理，就宣称整个程序恒定内存。

## 第十四章：完整实践二——扫描练习目录，统计扩展名并保存来源

### 14.1 只有总次数不够时怎么办？

你看到 `.txt` 有十份，接下来可能会问：“具体是哪十份？”

因此，我们同时维护扩展名计数、总字节数和对应文件列表。

这里会用到：

```text
Counter：数量

defaultdict(int)：累计大小

defaultdict(list)：记录对应文件
```

### 14.2 完整程序，Python 3.12+

```python
from collections import Counter, defaultdict
from pathlib import Path
from tempfile import TemporaryDirectory
import json


def collect_file_summary(root):
    root = Path(root).resolve(strict=True)
    if not root.is_dir():
        raise NotADirectoryError(root)

    counts = Counter()
    bytes_by_suffix = defaultdict(int)
    paths_by_suffix = defaultdict(list)
    problems = []

    def on_scan_error(error):
        problems.append({
            "path": str(error.filename),
            "message": str(error),
        })

    for current, directories, filenames in root.walk(on_error=on_scan_error):
        directories[:] = [
            name for name in directories
            if name not in {".git", ".venv", "__pycache__"}
        ]

        for filename in filenames:
            path = current / filename
            try:
                if path.is_symlink() or not path.is_file():
                    continue
                info = path.stat()
            except OSError as error:
                problems.append({"path": str(path), "message": str(error)})
                continue

            suffix = path.suffix.casefold() or "[无扩展名]"
            relative = path.relative_to(root).as_posix()

            counts[suffix] += 1
            bytes_by_suffix[suffix] += info.st_size
            paths_by_suffix[suffix].append(relative)

    categories = []
    for suffix in sorted(counts):
        categories.append({
            "suffix": suffix,
            "count": counts[suffix],
            "bytes": bytes_by_suffix[suffix],
            "files": sorted(paths_by_suffix[suffix]),
        })

    return {
        "root": str(root),
        "file_count": counts.total(),
        "logical_bytes": sum(bytes_by_suffix.values()),
        "categories": categories,
        "problems": problems,
    }


def prepare_demo(root):
    root = Path(root)
    (root / "data").mkdir(parents=True)
    (root / "src").mkdir()
    (root / "data" / "a.txt").write_text("abc", encoding="utf-8")
    (root / "data" / "b.TXT").write_text("你好", encoding="utf-8")
    (root / "src" / "main.py").write_text("print(1)\n", encoding="utf-8")
    (root / "README").write_text("demo\n", encoding="utf-8")


def main():
    with TemporaryDirectory(prefix="collections-files-") as temp_dir:
        root = Path(temp_dir) / "project"
        prepare_demo(root)
        report = collect_file_summary(root)
        print(json.dumps(report, ensure_ascii=False, indent=2, allow_nan=False))


if __name__ == "__main__":
    main()
```

### 14.3 先让样例自己创建输入，有什么好处？

你不用猜运行前目录里有哪些文件。

临时目录中的数据由代码明确创建，输入规模和内容都可控。

程序结束后，临时目录自动清理，避免把学习脚本误用于真实项目文件。

真实使用时，可以把 `collect_file_summary()` 单独拿出来处理你明确指定的目录。

### 14.4 `b.TXT` 为什么归入 `.txt`？

我们对后缀调用了 `casefold()`。

这是统计政策，目的是把常见扩展名大小写差异合并。

它没有重命名真实文件，报告的来源列表仍保留原始名字。

### 14.5 “字节总量”是否等于真实磁盘占用？

不等于。

这里累计普通文件的逻辑长度。

稀疏文件、压缩文件系统、硬链接和存储块分配都可能让实际占用不同。

不同硬链接名称在本例中分别计数，符号链接则跳过。

报告字段因此命名为 `logical_bytes`，避免暗示它是精确物理空间统计。

### 14.6 为什么先 `stat()` 成功再增加计数？

如果文件在扫描后消失，不能让“已计入数量”和“没有对应大小来源”悄悄混在一起。

本例选择只有成功取得所需状态的文件，才进入正式统计。

当然，你也可以分别统计“遇到的条目”和“成功读取的条目”，但字段名称必须说明两者不同。

### 14.7 如果只想要次数，还需要保存全部来源吗？

不需要。

`paths_by_suffix` 会保存每个文件名，文件多时占用明显内存。

如果任务只需要分类总数和总大小，删除这份来源索引即可。

容器组合应服务实际问题，不能因为 `defaultdict(list)` 很方便，就默认把所有中间数据永久存下来。

## 第十五章：完整实践三——有限历史窗口与滑动平均值

### 15.1 问题是什么？

假设程序不断产生耗时记录：

```text
10、20、30、40、50 毫秒
```

你只想显示最近三次平均耗时。

前面的旧数据会离开窗口，新数据会进入。

`deque(maxlen=3)` 很适合保存这个窗口。

但如果每次都对整个窗口求和，窗口很大时会重复计算。

我们可以同时维护累计和。

### 15.2 完整实现

```python
from collections import deque
import math


class MovingAverage:
    def __init__(self, size):
        if type(size) is not int or size <= 0:
            raise ValueError("size 必须是正整数")
        self._values = deque(maxlen=size)
        self._total = 0.0

    @property
    def size(self):
        return self._values.maxlen

    @property
    def count(self):
        return len(self._values)

    @property
    def values(self):
        return tuple(self._values)

    def add(self, value):
        if type(value) not in (int, float):
            raise TypeError("value 必须是普通整数或浮点数")
        try:
            value = float(value)
        except OverflowError as error:
            raise ValueError("value 超出本例浮点计算范围") from error
        if not math.isfinite(value):
            raise ValueError("value 必须是有限数值")

        if len(self._values) == self._values.maxlen:
            self._total -= self._values[0]

        self._values.append(value)
        self._total += value
        return self.average()

    def average(self):
        if not self._values:
            return None
        return self._total / len(self._values)

    def clear(self):
        self._values.clear()
        self._total = 0.0


def main():
    average = MovingAverage(3)

    print("空窗口：", average.average())
    for value in [10, 20, 30, 40, 50]:
        current = average.add(value)
        print("窗口：", average.values, "平均值：", current)

    average.clear()
    print("清理后：", average.values, average.average())


if __name__ == "__main__":
    main()
```

### 15.3 加入第四项前，为什么先看最左项？

当窗口已经有三项时，新项追加会让最左项自动丢弃。

如果你等追加完成后才找旧项，就已经拿不到它。

因此，先从累计和扣掉即将离开的值，再追加新值并增加累计和。

数据结构的自动行为有帮助，也要求你按照正确时机更新其他状态。

### 15.4 为什么 `values` 返回元组？

我们希望调用者能观察窗口，却不应直接修改内部队列，导致累计和与元素不一致。

返回一份元组快照，让外部读取更清楚。

这仍然是浅层快照，不过本例只允许普通数值，元素本身不可变，足以满足需求。

### 15.5 空窗口平均值为什么是 `None`？

没有数据时，平均值尚不存在。

返回零会让人以为真实平均耗时为零。

调用者可以把 `None` 显示为“暂无数据”。

如果你的应用希望空窗口抛异常，也可以，但要把接口约定写清楚。

### 15.6 浮点累计会不会有误差？

本例把允许的普通数值转换成 float 后计算；过大的整数会明确报告超出范围，而不是承诺支持任意精度金额。

浮点累计也会有误差。

反复加减浮点值，长期运行可能出现累计误差。

对精度要求高的任务，可以定期重新求和、采用适合的求和算法或选用明确的十进制类型。

即使单个输入有限，多个极大浮点数累加仍可能溢出；处理真实耗时等受控数据时，应另外限定合理数值范围。本例重点是窗口淘汰与累计状态的配合，不是任意精度数值算法。

`deque` 负责窗口顺序和容量，不负责数值分析的全部问题。

### 15.7 时间窗口和数量窗口一样吗？

不一样。

`maxlen=3` 表示最近三条，不是最近三秒。

如果记录产生速度不同，三条可能覆盖一毫秒，也可能覆盖十分钟。

时间窗口通常要保存时间戳，并按当前截止时间主动从左端淘汰旧记录。

## 第十六章：时间窗口中的事件计数

### 16.1 如果想知道最近十秒发生多少事件呢？

这次每条记录保存一个单调时钟数值。

队列按事件到达顺序排列。

每次查询时，把十秒以前的旧事件从左端移除。

```python
from collections import deque
import math
import time


class RecentEvents:
    def __init__(self, window_seconds, clock=time.monotonic):
        if type(window_seconds) not in (int, float):
            raise TypeError("窗口必须是普通数值")
        try:
            window = float(window_seconds)
        except OverflowError as error:
            raise ValueError("窗口超出浮点范围") from error
        if not math.isfinite(window) or window <= 0:
            raise ValueError("窗口必须是有限正数")
        self.window = window
        self.clock = clock
        self.events = deque()

    def _discard_old(self, now):
        cutoff = now - self.window
        while self.events and self.events[0] <= cutoff:
            self.events.popleft()

    def record(self):
        now = self.clock()
        self._discard_old(now)
        self.events.append(now)
        return len(self.events)

    def count(self):
        self._discard_old(self.clock())
        return len(self.events)
```

这里约定左边界不包含，也就是只保留满足“事件时间大于当前时间减窗口”的记录。

边界究竟包含不包含，应按实际统计规则说明。

### 16.2 为什么用 `monotonic()`？

我们关心经过时间，不需要把事件显示成某年某月某日。

系统墙上时间可能被校时或手动调整。

单调时钟更适合这类本进程内耗时窗口。

它的数值不能直接当作跨机器统一的 Unix 时间戳保存和比较。

### 16.3 不用真的等待十秒，也能测试吗？

可以通过注入时钟，让输入完全可控。

```python
class FakeClock:
    def __init__(self, now=0.0):
        self.now = now

    def __call__(self):
        return self.now

    def advance(self, seconds):
        self.now += seconds


clock = FakeClock()
recent = RecentEvents(10, clock=clock)

assert recent.record() == 1
clock.advance(4)
assert recent.record() == 2
clock.advance(6)
assert recent.count() == 1
clock.advance(4)
assert recent.count() == 0
```

这个例子同时展示一个有用的设计习惯：把外部变化来源作为依赖传入，让核心逻辑可以用具体输入验证。

### 16.4 它能直接用作正式服务限流器吗？

这个类适合单进程学习时间窗口。

它没有线程锁，没有跨进程共享，也没有分布式一致性机制。

大量事件还会让队列按窗口内事件数增长。

真实限流还涉及用户维度、存储、并发和故障政策，不能把一个教学队列直接当成完整服务基础设施。

## 第十七章：把 `Counter` 当成“有重复次数的集合”

### 17.1 集合只能说有没有，计数器还能说有几个

假设一个制作任务需要：

```python
required = {"screw": 4, "panel": 2, "motor": 1}
```

你手头有：

```python
available = {"screw": 6, "panel": 1, "motor": 1}
```

普通 `set` 只看到三种材料都存在，无法判断面板少了一个。

`Counter` 能表达每种材料的数量。

### 17.2 判断是否足够

```python
from collections import Counter

required = Counter({"screw": 4, "panel": 2, "motor": 1})
available = Counter({"screw": 6, "panel": 1, "motor": 1})

print(available >= required)
print(required - available)
```

第一项为假，第二项给出正数短缺：

```text
Counter({'panel': 1})
```

比较会把缺失元素按零计数处理。

这些丰富比较规则在 Python 3.10 起得到相应支持，本文基础环境可以使用。

### 17.3 零项是否影响相等？

```python
from collections import Counter

print(Counter(a=1) == Counter(a=1, b=0))
```

现代 Python 中结果为真。

对计数含义来说，缺失的 b 与显式零个 b 可以视为相同。

但键集合和 `len()` 仍可能不同。

因此，比较“计数相等”与比较“存储了完全相同的键”需要分别表达。

### 17.4 交集、并集、加法分别是什么？

```python
from collections import Counter

a = Counter(red=3, blue=1)
b = Counter(red=1, blue=2, green=1)

print(a + b)
print(a & b)
print(a | b)
```

加法合并计数。

交集按每个元素的较小计数取值。

并集按较大计数取值。

这些多重集合结果会保留正计数，零和负数不作为结果项保留。

所以它们很适合“有多少个相同元素”的集合式运算，未必适合保留所有正负差额的账本。

### 17.5 如果把浮点值放进计数器呢？

部分求和、更新和排序操作可以处理适当的数值。

但 `elements()` 需要按整数次数重复元素，浮点计数不适合直接用于这个接口。

容器允许保存一个值，不代表所有方法对这个值都有合理含义。

因此，数据模型应先约定计数到底是不是非负整数。

## 第十八章：完整实践四——用队列寻找最短学习依赖路径

### 18.1 为什么这是队列的自然用途？

假设你在整理知识关系。

从 Python 基础出发，可以学习文件处理或函数。

文件处理通向路径管理，函数通向装饰器。

你希望找到从某个主题到另一个主题，经过最少边数的路径。

如果每条边成本相同，可以使用广度优先搜索。

它按距离一层一层处理，因此很适合先入先出的队列。

### 18.2 输入关系

```python
graph = {
    "Python基础": ["函数", "文件处理"],
    "函数": ["装饰器", "生成器"],
    "文件处理": ["路径管理", "JSON"],
    "路径管理": ["自动化脚本"],
    "JSON": ["HTTP接口"],
    "装饰器": ["测试工具"],
    "生成器": ["数据管道"],
    "数据管道": ["自动化脚本"],
}
```

这里边只表示练习中设定的可达关系，不是严格课程先修证明。

### 18.3 完整实现

```python
from collections import deque


def shortest_path(graph, start, target, max_nodes=100_000):
    if type(max_nodes) is not int or max_nodes <= 0:
        raise ValueError("max_nodes 必须是正整数")

    queue = deque([start])
    parents = {start: None}

    while queue:
        current = queue.popleft()

        if current == target:
            path = []
            node = current
            while node is not None:
                path.append(node)
                node = parents[node]
            path.reverse()
            return path

        for neighbor in graph.get(current, ()):
            if neighbor in parents:
                continue

            if len(parents) >= max_nodes:
                raise ValueError("访问节点数量超过限制")

            parents[neighbor] = current
            queue.append(neighbor)

    return None


def main():
    graph = {
        "Python基础": ["函数", "文件处理"],
        "函数": ["装饰器", "生成器"],
        "文件处理": ["路径管理", "JSON"],
        "路径管理": ["自动化脚本"],
        "JSON": ["HTTP接口"],
        "装饰器": ["测试工具"],
        "生成器": ["数据管道"],
        "数据管道": ["自动化脚本"],
    }

    path = shortest_path(graph, "Python基础", "自动化脚本")
    if path is None:
        print("没有找到路径")
    else:
        print(" -> ".join(path))

    print(shortest_path(graph, "Python基础", "Python基础"))
    print(shortest_path(graph, "测试工具", "Python基础"))


if __name__ == "__main__":
    main()
```

### 18.4 为什么发现节点时就记录 `parents`？

如果等节点真正弹出时才标记，同一节点可能被多个前驱重复加入队列。

本例把“已经发现”和“已经安排处理”合并记录。

每个节点只第一次进入队列时保存父节点。

这既避免重复，也让最后能沿父指针恢复路径。

### 18.5 为什么不用 `pop()`？

从右端 `pop()` 会优先处理最近加入的节点，更接近深度优先顺序。

这样未必首先得到边数最少的路径。

`popleft()` 正好表达先加入先处理，让各距离层按顺序展开。

一个方法方向的变化，就可能改变算法性质。

### 18.6 为什么同样最短的路径可能有不同结果？

如果存在多条边数相同的路径，邻居遍历顺序会影响先发现哪一条。

输入邻居是列表时，顺序由你提供的列表决定。

如果改用集合，结果顺序就不应当作稳定业务保证。

需要稳定结果，应主动定义邻居排序或其他打破平局的规则。

### 18.7 这适用于带权路径吗？

如果一条边花一分钟，另一条花一小时，“经过边数最少”不等于“总成本最小”。

这个算法只按统一边成本理解。

带非负权重的最短路通常需要其他算法与优先队列，不能单靠把数字加进邻居元组就说问题解决。

### 18.8 为什么加入 `max_nodes`？

意外循环本身已经由发现集合避免，但非常大的图仍可能消耗大量内存。

访问数量上限让任务边界更明确。

这个上限不是算法正确性必需条件，而是应用对资源规模的一项约束。

## 第十九章：完整实践五——有来源说明的分层配置

### 19.1 只得到最终值，还可能想问什么？

配置显示端口九千。

你可能接着问：“这个值是默认的，还是用户文件改的，还是这次命令行覆盖的？”

`dict(ChainMap(...))` 能得到结果，却没有自动附上来源标签。

我们自己保留层名称，并在查找时记录来源。

### 19.2 完整程序

```python
from collections import ChainMap
import json


ALLOWED_KEYS = {"host", "port", "debug", "workers"}
DEFAULTS = {"host": "127.0.0.1", "port": 8000, "debug": False, "workers": 1}


def parse_bool(value):
    if type(value) is bool:
        return value
    if isinstance(value, str):
        normalized = value.strip().casefold()
        if normalized in {"true", "1", "yes"}:
            return True
        if normalized in {"false", "0", "no"}:
            return False
    raise ValueError("布尔配置只接受明确布尔值或约定字符串")


def parse_integer(value, name, lower, upper):
    if type(value) is int:
        number = value
    elif isinstance(value, str):
        try:
            number = int(value)
        except ValueError as error:
            raise ValueError(f"{name} 必须能转换为整数") from error
    else:
        raise ValueError(f"{name} 类型不正确")

    if not lower <= number <= upper:
        raise ValueError(f"{name} 必须在 {lower} 到 {upper} 之间")
    return number


def normalize_layer(mapping):
    if not isinstance(mapping, dict):
        raise TypeError("配置层必须是字典")

    unknown = set(mapping) - ALLOWED_KEYS
    if unknown:
        raise ValueError(f"未知配置项：{sorted(unknown)}")

    result = {}
    for key, value in mapping.items():
        if key == "host":
            if not isinstance(value, str) or not value.strip():
                raise ValueError("host 必须是非空字符串")
            result[key] = value.strip()
        elif key == "port":
            result[key] = parse_integer(value, key, 1, 65535)
        elif key == "workers":
            result[key] = parse_integer(value, key, 1, 128)
        elif key == "debug":
            result[key] = parse_bool(value)
    return result


def resolve_configuration(command_line, user_file, environment):
    layers = [
        ("command_line", normalize_layer(command_line)),
        ("user_file", normalize_layer(user_file)),
        ("environment", normalize_layer(environment)),
        ("defaults", DEFAULTS.copy()),
    ]

    chain = ChainMap(*(mapping for name, mapping in layers))
    values = dict(chain)
    sources = {}

    for key in values:
        for layer_name, mapping in layers:
            if key in mapping:
                sources[key] = layer_name
                break

    return {"values": values, "sources": sources}


def main():
    command_line = {"debug": True}
    user_file = {"port": 9000}
    environment = {"port": "8500", "workers": "4"}

    result = resolve_configuration(command_line, user_file, environment)
    print(json.dumps(result, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```

### 19.3 为什么环境数据也先转换？

真实环境变量通常是字符串。

`"false"` 是非空字符串，直接 `bool("false")` 会得到真。

因此，需要按明确文本集合转换，而不是把“非空”误当成“表示开启”。

这个示例使用普通字典模拟环境输入，避免为了展示配置而打印真实系统环境和其中可能包含的凭证。

### 19.4 为什么先把各层规范化，再建立链？

这样最终结果具有一致的类型。

端口不会因为来自文件就变整数，来自环境就仍然是字符串。

校验发生在入口，也能较早发现错误来源。

这个实现会检查被高优先级覆盖的低层字段；如果低层存在非法值，也会报告。

你可以选择只校验最终实际使用的值，但那是另一种政策，应明确说明。

### 19.5 为什么没有传入的字段才省略，而不把所有 `None` 都过滤掉？

本例把显式 `None` 当作非法配置值。

缺失意味着没有提供该层覆盖。

明确空值可能在别的协议中表示清除设置，不能一概等同于缺失。

设计配置工具时，这个区别必须先决定。

### 19.6 为什么结果转成普通字典？

我们在函数里规范化并解析一次，返回本次的配置快照。

后续原始输入修改，不应悄悄改变已经发给业务代码的配置。

如果你想保留动态链关系，也可以返回 `ChainMap`，但调用者就需要理解它的引用和写入语义。

## 第二十章：完整实践六——用 `OrderedDict` 理解最近最少使用缓存

### 20.1 缓存为什么需要顺序？

假设最多保存三份查询结果。

每次访问一个已有键，把它标为最近使用。

加入第四个不同键时，淘汰很久没有使用的那个。

这就是 LRU，也就是 Least Recently Used 的基本思路。

标准库 `functools.lru_cache` 已经提供函数缓存。

这里自己实现一个小容器，是为了理解顺序如何参与淘汰政策，并练习显式缓存管理。

### 20.2 完整实现

```python
from collections import Counter, OrderedDict


MISSING = object()


class SmallLRU:
    def __init__(self, maxsize):
        if type(maxsize) is not int or maxsize <= 0:
            raise ValueError("maxsize 必须是正整数")
        self.maxsize = maxsize
        self._data = OrderedDict()
        self._stats = Counter()

    def __len__(self):
        return len(self._data)

    def keys_oldest_first(self):
        return tuple(self._data)

    def get(self, key, default=MISSING):
        try:
            value = self._data[key]
        except KeyError:
            self._stats["misses"] += 1
            if default is MISSING:
                raise
            return default

        self._data.move_to_end(key)
        self._stats["hits"] += 1
        return value

    def put(self, key, value):
        existed = key in self._data
        self._data[key] = value
        self._data.move_to_end(key)
        self._stats["writes"] += 1

        if existed:
            self._stats["updates"] += 1

        if len(self._data) > self.maxsize:
            evicted = self._data.popitem(last=False)
            self._stats["evictions"] += 1
            return evicted

        return None

    def clear(self):
        self._data.clear()

    def statistics(self):
        return {
            "hits": self._stats["hits"],
            "misses": self._stats["misses"],
            "writes": self._stats["writes"],
            "updates": self._stats["updates"],
            "evictions": self._stats["evictions"],
            "size": len(self._data),
            "maxsize": self.maxsize,
        }


def main():
    cache = SmallLRU(3)
    cache.put("A", "result-A")
    cache.put("B", "result-B")
    cache.put("C", None)

    print(cache.keys_oldest_first())
    print(cache.get("A"))
    print(cache.keys_oldest_first())

    print("淘汰：", cache.put("D", "result-D"))
    print(cache.keys_oldest_first())
    print("C 的缓存值：", cache.get("C"))
    print("缺失默认：", cache.get("unknown", default="not-found"))
    print(cache.statistics())


if __name__ == "__main__":
    main()
```

### 20.3 为什么 `get()` 成功后要移动键？

LRU 关注最近使用时间，而不是最早写入时间。

A 最早创建，但刚刚再次访问，就应被视为最近使用。

所以 `move_to_end()` 放在命中之后。

如果只在写入时调整顺序，实现的就更接近按写入顺序淘汰，而不是最近访问顺序。

### 20.4 为什么默认值用专门的对象？

缓存本来允许保存 `None`。

如果拿 `None` 同时表示“没有传默认值”和“缓存结果为空”，接口含义容易混淆。

`MISSING = object()` 创建专用哨兵，比较时使用身份判断。

它表达的是代码内部的参数状态，不属于需要发送给外部的业务数据。

### 20.5 更新已有键，会不会误淘汰？

更新已有键不会增加容器大小，因此只有实际超过容量时才淘汰。

但它会把这个键移到最近使用端。

你可以自行测试连续 `put("A", ...)` 是否始终保持容量不变。

### 20.6 为什么 `clear()` 没有清统计？

本例把“清内容”和“累计运行统计”设计成两个含义。

真实接口也可以让 `clear()` 同时重置计数，或另外提供 `reset_statistics()`。

选择需要明确，避免调用者以为清缓存后命中次数也必然归零。

### 20.7 这个缓存会复制结果吗？

不会。

如果保存一个列表，取出来还是同一个列表对象。

调用者修改它，缓存中的内容也变化。

你可以在应用层约定缓存不可变结果，或者按需复制；容器不应擅自决定所有对象的复制策略。

### 20.8 能直接用于多线程服务吗？

这个实现没有锁。

一次查询包含查找、移动和更新统计多个步骤。

不能因为底层某些操作在某种解释器中通常看起来连续，就推断整个复合操作具备并发正确性。

生产代码还需要处理同步、过期时间、内存预算和加载失败政策。

## 第二十一章：完整实践七——用命名记录保存成绩，再按班级分组

### 21.1 为什么不把每条记录一直当列表？

假设原始数据每一行都是：

```text
编号、姓名、班级、分数
```

如果后面总写 `row[3]`，就要一直记住第三号位置是分数。

我们在入口完成验证，再转换成有名字的记录。

后续就可以写 `student.score` 和 `student.class_name`。

### 21.2 完整程序

```python
from collections import Counter, defaultdict, namedtuple
import json


Student = namedtuple("Student", "student_id name class_name score")


def parse_student(row):
    if not isinstance(row, (list, tuple)) or len(row) != 4:
        raise ValueError("每行必须恰好有四项")

    student_id, name, class_name, score_text = row

    for field_name, value in [
        ("student_id", student_id),
        ("name", name),
        ("class_name", class_name),
    ]:
        if not isinstance(value, str) or not value.strip():
            raise ValueError(f"{field_name} 必须是非空字符串")

    if type(score_text) is int:
        score = score_text
    elif isinstance(score_text, str):
        try:
            score = int(score_text)
        except ValueError as error:
            raise ValueError("score 必须是整数文本") from error
    else:
        raise ValueError("score 类型不正确")

    if not 0 <= score <= 100:
        raise ValueError("score 必须在 0 到 100 之间")

    return Student(
        student_id.strip(),
        name.strip(),
        class_name.strip(),
        score,
    )


def load_students(rows):
    students = []
    seen_ids = set()
    problems = []

    for index, row in enumerate(rows):
        try:
            student = parse_student(row)
            if student.student_id in seen_ids:
                raise ValueError(f"编号重复：{student.student_id}")
        except ValueError as error:
            problems.append({"row_index": index, "message": str(error)})
            continue

        seen_ids.add(student.student_id)
        students.append(student)

    return students, problems


def make_class_report(students):
    by_class = defaultdict(list)
    score_bands = Counter()

    for student in students:
        by_class[student.class_name].append(student)
        if student.score >= 90:
            score_bands["90-100"] += 1
        elif student.score >= 60:
            score_bands["60-89"] += 1
        else:
            score_bands["0-59"] += 1

    classes = []
    for class_name in sorted(by_class):
        members = by_class[class_name]
        total = sum(student.score for student in members)
        ranked = sorted(
            members,
            key=lambda student: (-student.score, student.student_id),
        )

        classes.append({
            "class_name": class_name,
            "count": len(members),
            "average_score": total / len(members),
            "students": [student._asdict() for student in ranked],
        })

    return {
        "student_count": len(students),
        "score_bands": {
            band: score_bands[band]
            for band in ["90-100", "60-89", "0-59"]
        },
        "classes": classes,
    }


def main():
    rows = [
        ["s001", "小林", "一班", "95"],
        ["s002", "小周", "二班", "88"],
        ["s003", "小陈", "一班", "100"],
        ["s004", "小何", "二班", "59"],
        ["s004", "重复编号", "二班", "90"],
        ["s005", "错误分数", "一班", "excellent"],
    ]

    students, problems = load_students(rows)
    report = make_class_report(students)
    report["problems"] = problems
    print(json.dumps(report, ensure_ascii=False, indent=2, allow_nan=False))


if __name__ == "__main__":
    main()
```

### 21.3 数据验证发生在哪里？

发生在 `parse_student()` 和 `load_students()`。

`namedtuple` 本身不检查分数范围，也不检查编号是否重复。

命名记录让通过验证的数据更容易使用，验证函数让外部数据进入内部模型前满足约定。

这两者各自有清楚职责。

### 21.4 为什么用编号打破分数相同的平局？

分数相同的学生也需要稳定显示顺序。

如果只按分数，Python 稳定排序会保留原输入顺序，但输入顺序可能来自不同文件或数据库查询。

我们把编号作为次级规则，报告更容易比较。

是否应该按姓名、学号或原顺序展示，属于实际产品需求，本例选择学号只是为了明确。

### 21.5 为什么班级平均值不会除以零？

班级键是在加入真实学生时创建。

因此，进入输出循环的班级分组至少有一个学生。

如果你先为了查询访问了 `by_class["三班"]`，就可能创建一个空分组，后面的这个推理就不成立。

这再次说明：`defaultdict` 的缺失读取会改变状态，查询动作需要谨慎。

### 21.6 为什么导出前使用 `_asdict()`？

`namedtuple` 本身是元组子类，直接 JSON 编码通常会变成数组，字段名字不会作为对象键保留。

`_asdict()` 明确生成字段名到值的映射，更适合可读报告。

但它不是深拷贝；本例字段都是不可变基本对象，所以共享引用不是问题。

### 21.7 如果以后增加字段，会发生什么？

构造函数、原始行格式、验证规则和输出契约都可能需要同步调整。

命名记录提高了可读性，没有取消数据模型维护成本。

变化很多的业务模型可以考虑数据类或验证模型，但仍应先理解字段与规则，而不是只更换工具名称。

## 第二十二章：常见疑问——这几个容器怎样放在一起理解？

### 22.1 `Counter` 和 `defaultdict(int)` 看起来都能计数，为什么要两个？

两者都能写 `counts[key] += 1`。

`Counter` 还直接表达计数集合的概念，并提供 `most_common()`、`elements()`、计数运算和丰富比较。

`defaultdict(int)` 更像是“遇到缺失键先准备零”的通用数值映射。

如果你的主要需求就是数出现次数，`Counter` 更直接。

如果你在构造一套有默认数值的状态映射，而不需要计数器特有操作，`defaultdict(int)` 也很自然。

### 22.2 分组究竟用列表还是集合？

需要保留重复记录和到达顺序，用列表。

需要去掉重复，只关心成员存在，用集合。

需要去重并保留首次顺序，可以用字典键或另外维护顺序列表和成员集合。

容器选择直接决定信息是否保留，因此应先问“重复项有没有意义”。

### 22.3 `deque(maxlen=...)` 是缓存吗？

它只是有限长度的双端序列。

它不知道哪个键被访问，也不知道过期时间。

LRU 缓存要根据访问行为调整顺序；时间缓存要根据时钟判断失效；任务缓存还可能需要错误政策。

不要因为都能限制数量，就把这些概念当成同一件事。

### 22.4 `ChainMap` 能深度合并配置吗？

默认按顶层键查找。

如果前层有 `database` 字典，后层也有 `database` 字典，取这个键会直接得到前层值，不会自动把两个内部字典逐项合并。

需要深度合并时，应为嵌套字段制定明确政策，例如列表替换还是追加、空值是否清除、未知字段是否允许。

这些规则没有一个对所有项目都正确的默认答案。

### 22.5 这些容器能保存到 JSON 吗？

多数继承自字典的容器可以按普通映射数据编码，但特殊默认工厂、链关系和缓存语义不会被 JSON 保留。

`deque` 一般应先转列表。

`namedtuple` 若直接按元组编码会变数组，想保留名字应转字典。

集合要决定是排序列表还是其他结构。

序列化保存的是你选择的数据表示，不是任意 Python 容器完整行为的快照。

### 22.6 复制这些容器，会复制里面的对象吗？

常见 `copy()` 和构造转换大多是浅拷贝。

你得到一个新外壳，内部引用仍可能共享。

如果里面是字符串和整数，一般容易管理；如果里面是列表、字典或类实例，就要明确谁可以修改。

深拷贝也不是所有对象都能随意使用的万能操作，资源对象、共享关系和业务身份都可能需要特别处理。

### 22.7 是否应该给函数参数写具体容器类型？

如果函数只需要读取键值，可以考虑接受 `Mapping` 这类抽象接口。

如果函数依赖 `most_common()`，那它确实需要计数器能力，写 `Counter` 更明确。

不要一边把参数标成任意映射，一边内部调用只有某个具体类才提供的方法。

类型提示应描述真实依赖，而不是只追求看起来宽泛或高级。

## 第二十三章：动手练习与参考答案

下面的练习建议先自己写，再看答案。

练习输入都很小，但它们对应真实项目中经常出现的组织方式。

### 23.1 练习一：统计后缀，同时保留没有出现的分类

要求：统计名称列表中的最后后缀，忽略大小写；输出 `.txt`、`.py`、`.json` 三个固定分类，即使没有出现也显示零。

```python
from collections import Counter
from pathlib import PurePosixPath

names = ["a.txt", "b.TXT", "main.py", "README"]
counts = Counter(
    PurePosixPath(name).suffix.casefold() or "[无扩展名]"
    for name in names
)

for suffix in [".txt", ".py", ".json"]:
    print(suffix, counts[suffix])
```

`.json` 下标读取得到零，但不会因此自动存入计数器。

可以在读取前后打印键列表验证这一点。

### 23.2 练习二：数量和净差额要分别报告

要求：比较预期与实际数量，输出正数短缺，并保留一份有正有负的完整差额。

```python
from collections import Counter

required = Counter(book=4, pen=3)
actual = Counter(book=2, pen=5)

shortage = required - actual
net = required.copy()
net.subtract(actual)

print(shortage)
print(net)
```

短缺报告会留下少了的书。

净差额还会保留笔数量多出的负值。

选错运算，报告含义就会变化。

### 23.3 练习三：按用户收集不重复主题

```python
from collections import defaultdict

pairs = [
    ("A", "Python"),
    ("B", "SQL"),
    ("A", "Python"),
    ("A", "Linux"),
]

groups = defaultdict(set)
for user, topic in pairs:
    groups[user].add(topic)

for user in sorted(groups):
    print(user, sorted(groups[user]))
```

如果改成 `defaultdict(list)` 加 `append()`，重复 Python 会保留。

两种结果都可能正确，要看你是在统计记录还是统计不同主题。

### 23.4 练习四：只查询，不让缺失键出现

```python
from collections import defaultdict

groups = defaultdict(list)
value = groups.get("unknown", [])
print(value)
print("unknown" in groups)
```

这次没有调用默认工厂。

提供给 `get()` 的空列表只是本次查询返回的默认对象，没有自动成为映射成员。

如果你修改它，也不会自动写回容器。

### 23.5 练习五：证明每组有独立列表

```python
from collections import defaultdict

groups = defaultdict(list)
groups["A"].append(1)
groups["B"].append(2)

assert groups["A"] == [1]
assert groups["B"] == [2]
assert groups["A"] is not groups["B"]
```

然后把工厂改成返回共享列表，观察为什么断言不再成立。

这项练习的关键是工厂每次创建新对象，而不是 `defaultdict` 会自动深拷贝任意结果。

### 23.6 练习六：最近三条与最早三条

```python
from collections import deque

recent = deque(maxlen=3)
for value in range(5):
    recent.append(value)

print(list(recent))
```

得到最近的二、三、四。

如果你需要“只收最早三条，之后拒绝”，应该主动检查并决定拒绝政策，不能使用自动淘汰旧项的同一规则。

### 23.7 练习七：理解左端批量加入

```python
from collections import deque

first = deque(["tail"])
first.extendleft(["A", "B", "C"])

second = deque(["tail"])
second.extendleft(reversed(["A", "B", "C"]))

print(list(first))
print(list(second))
```

先在纸上模拟每一次 `appendleft()`，再运行。

理解操作顺序比死记“这个函数会反转”更容易迁移到其他双端操作。

### 23.8 练习八：配置赋值发生在哪一层？

```python
from collections import ChainMap

local = {}
defaults = {"port": 8000}
config = ChainMap(local, defaults)

config["port"] = 9000

print(local)
print(defaults)
```

默认层仍为八千，第一层增加九千覆盖。

再尝试删除第一层的端口，随后读取会回退到默认层。

这说明删除覆盖值不等于从所有层删除这个配置项。

### 23.9 练习九：联合查找与快照

```python
from collections import ChainMap

user = {"theme": "light"}
chain = ChainMap(user, {"port": 8000})
snapshot = dict(chain)

user["theme"] = "dark"

print(chain["theme"])
print(snapshot["theme"])
```

链视图能看见后续修改，普通快照保留创建时的顶层值。

如果字段是可变嵌套对象，普通浅快照仍可能共享内部状态，需要继续分析。

### 23.10 练习十：从位置记录变成命名记录

```python
from collections import namedtuple

Task = namedtuple("Task", "task_id title done", defaults=[False])
task = Task("t001", "学习 collections")
updated = task._replace(done=True)

print(task.done)
print(updated.done)
print(updated._asdict())
```

原记录没有被修改。

`_replace()` 生成新记录，适合让“更新前”和“更新后”同时可观察。

### 23.11 练习十一：按最近使用顺序淘汰

要求：容量二；加入 A、B；访问 A；再加入 C，应该淘汰 B。

```python
from collections import OrderedDict

cache = OrderedDict()
cache["A"] = 1
cache["B"] = 2

value = cache["A"]
cache.move_to_end("A")

cache["C"] = 3
if len(cache) > 2:
    removed = cache.popitem(last=False)
    print("淘汰：", removed)

print(list(cache))
```

注意访问 `cache["A"]` 本身不会自动调整顺序。

移动是缓存政策的一部分，需要你明确执行。

### 23.12 练习十二：相等的值，顺序不同

```python
from collections import OrderedDict

first = OrderedDict([("a", 1), ("b", 2)])
second = OrderedDict([("b", 2), ("a", 1)])

assert first != second
assert dict(first) == dict(second)
```

这项练习帮助你看到类型定义的相等语义会影响结果。

不能只因为两个对象打印出相同键值，就认定所有容器都按同一规则比较。

### 23.13 练习十三：实现一个不会共享默认状态的统计工厂

```python
from collections import defaultdict


def new_bucket():
    return {"count": 0, "names": []}


buckets = defaultdict(new_bucket)
buckets["A"]["count"] += 1
buckets["A"]["names"].append("小林")

assert buckets["B"] == {"count": 0, "names": []}
assert buckets["A"]["names"] is not buckets["B"]["names"]
```

这里每次缺失都创建一套新字典和新列表。

如果把 `new_bucket()` 的返回值放到全局共享对象再重复返回，嵌套状态也会共享。

### 23.14 练习十四：对广度优先搜索增加一个环

在前面图中加入：

```python
graph["自动化脚本"] = ["Python基础"]
```

再次查询一条可达路径和一条不存在的路径。

程序应该正常结束，不应因为环而无限加入同一节点。

检查原因：节点首次入队时已经写入 `parents`，以后再遇到就跳过。

### 23.15 练习十五：给统计程序设计三个坏输入

分别使用空姓名、负分钟数、布尔分钟数。

确认它们都被记录为错误，而且不会增加主题次数或个人总分钟数。

这验证的是“验证通过后才修改统计状态”的顺序。

如果你先增加次数，再发现分钟数非法，就需要额外撤回，程序会更容易出错。

## 第二十四章：把容器行为画成几条清楚的边界

### 24.1 什么时候会执行用户代码？

创建和访问这些容器时，可能调用工厂、迭代器、键的哈希和相等方法等。

例如 `defaultdict` 缺失访问会执行工厂，`Counter(iterable)` 会消费输入迭代器。

因此，不应把任何容器操作都想成完全没有外部行为的简单内存读取。

实际接收自定义对象时，异常和副作用可能来自这些协议方法。

### 24.2 什么时候会丢弃信息？

集合分组会删除重复信息。

有界 `deque` 会丢弃旧端元素。

计数器的部分多重集合运算会去掉非正计数。

映射保存同名键会覆盖旧值。

你需要的是哪一种结果，决定你能否接受这种信息损失。

### 24.3 什么时候会保存引用？

容器通常保存对象引用，不自动深拷贝元素。

`ChainMap` 还明确保存底层映射引用。

所以，外部修改可能通过共享引用反映到容器中。

设计只读报告时，常把最终结果转换成由不可变值或独立列表组成的结构，明确交付边界。

### 24.4 什么时候应把数据存到数据库？

这些容器主要管理当前进程内的数据。

如果要求持久保存、多个进程协同、事务一致性或按复杂条件查询，就应考虑数据库或其他持久化系统。

把一个 `Counter` 转 JSON 保存可以满足小型统计快照，却不会自动提供数据库事务、索引和并发更新政策。

### 24.5 什么时候该保持普通列表和字典？

只有几个键、没有特殊默认规则的配置，用普通字典很清楚。

主要靠索引和切片处理的序列，用列表很自然。

固定长度记录需要可变字段和更多行为时，普通类或数据类可能更合适。

工具越贴近你的操作意图，解释成本通常越低。

## 第二十五章：常用接口复习表

| 需求 | 适合先考虑的接口 | 需要一起记住什么 |
| --- | --- | --- |
| 数出现次数 | `Counter(iterable)` | 元素粒度由输入决定，键需可哈希 |
| 增加一批计数 | `Counter.update()` | 映射值累加，普通迭代元素各加一次 |
| 保留完整差额 | `Counter.subtract()` | 可以保留零和负数 |
| 正数短缺 | `a - b` | 非正项从多重集合结果移除 |
| 次数排名 | `most_common(n)` | 平局采用首次遇到顺序 |
| 计数值总和 | `total()` | 负数也参与 |
| 根据次数展开 | `elements()` | 需要整数计数，忽略非正项 |
| 缺失时新建列表 | `defaultdict(list)` | 下标访问会插入，get 不触发工厂 |
| 缺失时准备零 | `defaultdict(int)` | 默认工厂是可调用对象 |
| 分组去重 | `defaultdict(set)` | 集合不保留业务排列顺序 |
| 左右两端操作 | `deque` | 中间随机访问不如列表贴切 |
| 最近若干记录 | `deque(maxlen=n)` | 自动丢旧项，不适合不能丢的任务队列 |
| 左端取出 | `popleft()` | 空容器会报错 |
| 循环旋转 | `rotate(n)` | 正数右移，负数左移 |
| 多层配置查找 | `ChainMap(first, second, ...)` | 左侧优先，写删只第一层 |
| 添加临时配置层 | `new_child()` | 保留父层引用 |
| 有名字的元组记录 | `namedtuple()` | 字段引用不可改，内部可变对象仍可能改变 |
| 生成更新后的记录 | `_replace()` | 返回新实例 |
| 导出字段字典 | `_asdict()` | 浅层转换，不是完整深拷贝 |
| 主动重排键 | `OrderedDict.move_to_end()` | 普通读取不自动移动 |
| 从指定端弹出项目 | `OrderedDict.popitem(last=...)` | 空容器报错 |
| 自定义容器包装 | `UserDict` 等 | 检查各公共操作是否符合统一规则 |

阅读到这里，你可以从自己的任务反推选择。

“我要数几次”，先看 `Counter`。

“第一次遇到键需要准备什么”，先看 `defaultdict`。

“我要从哪一端加入和取出”，先看 `deque`。

“这个值应该按哪些配置层查找”，先看 `ChainMap`。

“这条固定记录的每个位置叫什么”，先看 `namedtuple`。

“已有键需要按访问行为重新排队”，先看 `OrderedDict`。

官方参考：[collections 模块](https://docs.python.org/3.14/library/collections.html)、[Counter](https://docs.python.org/3.14/library/collections.html#collections.Counter)、[defaultdict](https://docs.python.org/3.14/library/collections.html#collections.defaultdict)、[deque](https://docs.python.org/3.14/library/collections.html#collections.deque)、[ChainMap](https://docs.python.org/3.14/library/collections.html#collections.ChainMap)、[namedtuple](https://docs.python.org/3.14/library/collections.html#collections.namedtuple)、[OrderedDict](https://docs.python.org/3.14/library/collections.html#collections.OrderedDict-objects)。

本篇的任务模型和示例数据用于学习，各项容量、分数字段、错误处理和配置优先级都是明确选择的练习政策，实际项目应按自己的契约调整。
