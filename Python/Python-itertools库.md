# Python `itertools` 库超详细教程：从按需取值，到截取、连接、分组、组合与数据管道

## 零基础先导：为什么“逐个产生”，有时比“一次性列表”更合适？

`itertools` 可理解为 iterator tools（迭代器工具）。它不是新的一套 `for` 语法，而是把经常写的**迭代模式**做成可复用的工具。

先看一件事：`range(1_000_000)` 能表示很多数，但不必提前创建一百万个整数列表。迭代器的基本思路是**需要下一个值时再提供**。这能避免许多不必要的中间列表，但不等于任何迭代过程都不会占内存。

```python
from itertools import islice

numbers = iter(range(1000000))  # 得到迭代器
first_three = list(islice(numbers, 3))  # 只取 3 个值
print(first_three)  # [0, 1, 2]
print(next(numbers)) # 3：前面三个已经被消耗
```

每行为什么存在？`iter(...)` 让你显式取得迭代器；`islice(..., 3)` 只从它取三个值；`list(...)` 才将这三个值收集到列表。**迭代器通常有消费状态**，不能假定第二次遍历还会从头开始。

后面常用函数按问题分类更容易记：`chain` 负责依次连接多个来源，`islice` 截取，`enumerate`（内置函数）配合遍历产生下标，`groupby` 对**连续相同键**分组，`product` 生成各维度的笛卡尔积。特别注意：`groupby` 不是 SQL 的“全局分组”；想把相同键聚在一起，通常要先排序或使用字典分组。`product` 组合数量可能呈乘法增长，使用前先想结果规模。

---

假设你已经知道，Python 可以使用 `for` 遍历列表，也可以用生成器逐个产生结果。

接下来，你可能会遇到这些问题。

一个很长的数据来源，我只想先看十条，应该怎样截取？

几份文件的记录想依次处理，是否必须先合成一个大列表？

相同用户的连续日志想归成一组，应该怎样做？

一个生成器想交给两处使用，直接赋给两个变量可以吗？

三个参数各有几种候选值，怎样遍历所有搭配？

这些问题都可以手写循环解决。

`itertools` 提供了一组围绕迭代器组织的工具，把许多常见循环模式表达得更直接。

这篇会先把取值过程讲清楚，再逐个学习常用工具，最后组成实际数据处理程序。

基础环境按 Python 3.11 及以上考虑；`batched()` 等较新接口明确标注版本。

同一小节的短例子可以接着执行，变量沿用本节先前定义；完整实践会给出自己的导入和输入。预期报错的例子只用于观察相应失败行为。

## 第一章：这个模块的名字和定位

### 1.1 `itertools` 可以怎么理解？

`iter` 与 iteration，也就是迭代有关。

`tools` 是工具。

把它理解为“迭代工具箱”就很直观。

它不是一个巨型数据框架，不需要建立应用服务器，也不要求你把数据先转换成特殊表格对象。

许多接口只接收普通可迭代对象，再返回一个迭代器。

```python
from itertools import islice

numbers = range(100)
first_five = islice(numbers, 5)

print(list(first_five))
```

输出：

```text
[0, 1, 2, 3, 4]
```

### 1.2 需要安装吗？

不需要。

它属于 Python 标准库：

```python
import itertools
```

或者只导入所需名称：

```python
from itertools import chain, groupby, islice
```

模块提供的许多名称是函数或可调用类型。

你调用它们得到工具对象，再通过迭代取得结果。

### 1.3 为什么不总写列表推导式？

列表推导式适合需要马上得到列表的场景。

如果你只想逐条处理数据，先保存全部结果可能不是必需的。

例如：

```python
squares = [number * number for number in range(1_000_000)]
```

会建立一百万个结果对应的列表。

如果后续只取前五个，前面的大部分计算和保存都没有被实际使用。

迭代工具允许你把处理过程按需连接起来。

但这不代表它们永远不缓存，或者使用后内存就一定固定。

本篇会特别说明哪些工具需要保存输入或等待后续消费者。

## 第二章：先分清可迭代对象、迭代器和一次取值

### 2.1 列表可以反复遍历，为什么生成器第二次空了？

先看列表：

```python
numbers = [10, 20, 30]

print(list(numbers))
print(list(numbers))
```

两次都得到相同元素。

再看迭代器：

```python
numbers = iter([10, 20, 30])

print(list(numbers))
print(list(numbers))
```

输出：

```text
[10, 20, 30]
[]
```

第二次没有重新开始。

这个对象保存了“下一次读到哪里”的状态，第一次已经消费到末尾。

### 2.2 `iter()` 和 `next()` 分别做什么？

```python
values = [10, 20, 30]
iterator = iter(values)

print(next(iterator))
print(next(iterator))
print(list(iterator))
```

输出：

```text
10
20
[30]
```

`iter(values)` 取得可以逐个取值的迭代器。

`next(iterator)` 取出下一项。

后面的 `list(iterator)` 继续消费剩余部分，没有回头重读已经取出的十和二十。

### 2.3 耗尽以后，还能调用 `next()` 吗？

继续调用通常会抛出 `StopIteration`。

如果你希望用默认值表示结束，可以：

```python
iterator = iter([10])

print(next(iterator, "结束"))
print(next(iterator, "结束"))
```

得到十和字符串“结束”。

真实数据本身也可能等于这个字符串，因此需要严格区分时，可以使用专门的哨兵对象。

### 2.4 把迭代器赋给两个变量，会有两个读取位置吗？

```python
source = iter([10, 20, 30])
first = source
second = source

print(next(first))
print(next(second))
```

输出十、二十。

两个变量引用的是同一个迭代器，位置共享。

赋值没有复制整个数据来源，也没有为你创建第二个独立游标。

稍后讲 `tee()` 时，我们会继续解决“两个消费者都想看同一批值”的需求。

### 2.5 对迭代器再次 `iter()`，会重新开始吗？

```python
source = iter([10, 20, 30])
print(iter(source) is source)
```

通常得到真。

迭代器的迭代接口返回自己，这是它能够直接放进 `for` 的原因之一。

如果想重新遍历列表，可以对原列表再创建一个迭代器。

如果来源是已经读过的网络流、文件流或生成器，则需要相应的重新打开、定位或重建机制。

### 2.6 如果我想先知道一共有多少条，再开始处理呢？

对于列表，直接 `len(values)` 不需要消费一个读取游标。

但一般迭代器不一定知道剩余总量，也通常没有同样的长度接口。

你可以数：

```python
source = iter([10, 20, 30])
count = sum(1 for item in source)

print(count)
print(list(source))
```

数完得到三，来源也已经耗尽。

如果后面还要使用，常见选择有三种。

来源有限而规模可控，就保存列表，再取长度并处理。

来源可重新打开，就明确执行两次读取，但要考虑两次之间数据可能变化。

可以在一次处理过程中累计总数，等处理结束再报告。

不必为了先显示一个精确总数，强行把未知大流全部缓存在内存。

## 第三章：惰性，到底是哪些事情暂时没有发生？

### 3.1 给生成过程加上观察输出

```python
from itertools import islice


def source_numbers():
    for number in range(5):
        print("产生：", number)
        yield number


source = source_numbers()
selected = islice(source, 2)

print("工具已经建立")
print(next(selected))
print(next(selected))
```

你会先看到“工具已经建立”，然后才看到对应元素产生。

`source_numbers()` 返回生成器时，函数体还没有逐项执行。

`islice` 建立时，也没有为了这个例子先把五个数全部保存下来。

消费者真正请求下一项，处理才向前推进。

### 3.2 如果只取一项，剩下会自动计算吗？

通常不会因为你建立了这个简单管道，就自动把后面所有项算完。

```python
source = source_numbers()
selected = islice(source, 4)

print(next(selected))
```

这次只要求第一个结果，来源只推进到需要的位置。

但不要把这个现象推广为任何工具建立时都绝不读取输入。

例如后面的 `product()` 会预先保存有限输入池。

### 3.3 为什么错误有时发生在 `for`，而不是建立工具那一行？

```python
from itertools import starmap

pairs = [(10, 2), (20, 0)]
results = starmap(lambda first, second: first / second, pairs)

print("已经建立结果迭代器")
print(next(results))
# 下一次取值预期触发 ZeroDivisionError。
print(next(results))
```

除法是在结果被请求时执行。

因此，错误也在消费阶段出现。

如果你只把 `results = ...` 放在 `try` 里，却在外面遍历，就可能没有捕获真正发生的错误。

### 3.4 惰性会不会让调试更难？

它让执行时间分散到消费处，因此需要更清楚地追踪数据流。

好办法是给关键阶段加小型观察函数，在少量受控数据上运行。

不要为了查看前几条而直接 `list()` 一个可能无限或巨大的来源。

也不要对同一个迭代器打印一次完整列表后，再期待业务循环还能读到原数据。

## 第四章：`islice()`——迭代器没有切片时，怎样只看一部分？

### 4.1 为什么不能直接 `source[:5]`？

列表有索引和切片接口。

生成器、文件等迭代器通常没有相同的切片能力。

你可以请求它们向前产生值，但不能假设它们支持随意返回任意历史位置。

`islice()` 提供按迭代顺序截取的方式。

### 4.2 最常见的两个调用形式

```python
islice(iterable, stop)
```

或者：

```python
islice(iterable, start, stop, step)
```

`start` 是跳过前面多少位置。

`stop` 是停止位置，不包含该位置。

`step` 是取值间隔。

```python
from itertools import islice

print(list(islice(range(10), 5)))
print(list(islice(range(10), 2, 8)))
print(list(islice(range(10), 2, 8, 2)))
```

输出：

```text
[0, 1, 2, 3, 4]
[2, 3, 4, 5, 6, 7]
[2, 4, 6]
```

### 4.3 想从第五项之后一直读到结尾呢？

结束位置可以使用 `None`：

```python
from itertools import islice

print(list(islice(range(8), 5, None)))
```

得到五、六、七。

如果来源本身无限，这个切片仍然可能无限，因此还需要下游终止条件。

### 4.4 可以像列表那样用负数索引吗？

不能直接照搬。

`islice()` 不支持普通列表切片那种负数起点、负数终点和负步长。

它只向前消费来源，不会自动保存全部内容后倒着取。

需要最后五项，可以维护 `deque(maxlen=5)`，但那仍然需要先消费完整有限输入才能知道最终五项。

### 4.5 被跳过的元素，还能从原迭代器再读到吗？

```python
from itertools import islice

source = iter(range(10))
print(list(islice(source, 2, 8, 2)))
print(next(source))
```

第一条输出二、四、六。

下一次原来源通常已经到八。

`islice` 为了走到相应位置，已经消费了中间没有交给你的元素。

它提供的是向前筛取，不是给来源做一份不影响游标的视图。

### 4.6 只创建但不消费，也会跳过去吗？

```python
source = iter(range(10))
selected = islice(source, 5)
print(next(source))
```

这次原来源仍从零开始。

建立工具与推进工具是两个时刻。

但现在你已经从原来源取走了零，后面再消费 `selected`，它会从当前剩余位置继续。

因此，不要在没有明确设计的情况下，让原来源和包装工具交替消费同一个状态。

## 第五章：`chain()`——几批数据怎样依次处理？

### 5.1 合并处理，不一定要先创建大列表

```python
from itertools import chain

morning = ["读文档", "写例子"]
afternoon = ["做练习", "整理笔记"]

for task in chain(morning, afternoon):
    print(task)
```

先处理第一批直到耗尽，再处理第二批。

`chain()` 不会在两批之间自动交错，也不会按时间排序。

它表达的就是按来源顺序连接。

### 5.2 与列表加法有什么区别？

```python
combined = morning + afternoon
```

会生成一个新列表，保存两批元素引用。

`chain(morning, afternoon)` 返回连接迭代器，不需要先建立完整结果列表。

如果你最后仍写 `list(chain(...))`，结果列表还是会被保存。

内存取舍取决于你如何消费结果，而不只取决于中间函数名称。

### 5.3 `chain()` 会自动递归展开所有嵌套吗？

不会。

```python
from itertools import chain

print(list(chain([1, [2, 3]], [4])))
```

得到：

```text
[1, [2, 3], 4]
```

内部列表仍然是一个元素。

这很有用，因为列表本身可能就是业务记录，不应该无条件拆散。

### 5.4 很多批数据放在一个列表里，怎样展开一层？

使用：

```python
from itertools import chain

batches = [[1, 2], [3], [], [4, 5]]
print(list(chain.from_iterable(batches)))
```

得到一到五。

`from_iterable()` 接受一个产生各批来源的外层可迭代对象，再依次消费其中每批。

### 5.5 为什么不总写 `chain(*batches)`？

如果 `batches` 已经是小型列表，这样写也能表达连接。

但调用时的 `*` 会先展开外层来源，把各项变成位置参数。

如果外层本来是一个很长或无限的生成器，提前展开就可能失去按需处理的优势，甚至永远无法完成调用。

`chain.from_iterable()` 可以让外层各批也按需取得。

### 5.6 字符串放进去，会怎样？

```python
print(list(chain.from_iterable(["AB", "CD"])))
```

得到字符 A、B、C、D。

因为每个内层字符串可迭代，迭代单位是字符。

如果你想把完整字符串作为元素，应提供包含字符串的记录序列，不要无意多展开一层。

## 第六章：`count()`、`repeat()` 和 `cycle()`——无限来源怎样安全使用？

### 6.1 `count()` 持续产生编号

```python
from itertools import count, islice

numbers = count(start=100, step=10)
print(list(islice(numbers, 5)))
```

输出：

```text
[100, 110, 120, 130, 140]
```

`start` 是起点，`step` 是步长。

它不像 `range()` 那样必须提供有限终点。

因此，使用前先想好谁负责停止。

### 6.2 直接 `list(count())` 会怎样？

它会试图持续收集没有自然终点的数据，可能不断占用内存。

不要在真实解释器里为了验证而运行这个表达式。

观察无限来源时，先用 `islice()`、有限 `zip()` 或明确的循环条件限制数量。

### 6.3 浮点步长一定精确吗？

```python
from itertools import count, islice

print(list(islice(count(0.0, 0.1), 5)))
```

浮点累计可能出现二进制表示误差。

对某些需求，可以根据整数索引计算 `start + step * index`，减少反复累计的误差形态，但它也不让所有十进制小数变得完全精确。

严格十进制业务应选择对应数值表示和规则。

### 6.4 `repeat()` 重复的是值，还是新对象？

```python
from itertools import repeat

print(list(repeat("ready", 3)))
```

得到三个字符串元素。

但它重复提供同一个对象引用，不是每次调用工厂生成新对象。

```python
from itertools import repeat

values = list(repeat([], 3))
values[0].append("changed")
print(values)
```

三个位置会看到同一列表的变化。

需要独立列表时，可以使用列表推导式逐次创建，或明确的生成器工厂。

### 6.5 不提供次数会怎样？

`repeat(value)` 会持续提供同一值。

常见用途是给 `map()` 或 `zip()` 的某一侧提供常量流。

```python
from itertools import repeat

print(list(map(pow, [2, 3, 4], repeat(2))))
```

平方计算在有限的数字来源耗尽后结束。

这里真正的停止条件来自另一侧有限输入。

### 6.6 `cycle()` 轮流重复一批值

```python
from itertools import cycle, islice

workers = cycle(["A", "B", "C"])
print(list(islice(workers, 8)))
```

输出：

```text
['A', 'B', 'C', 'A', 'B', 'C', 'A', 'B']
```

很适合简单轮转标签或分配顺序演示。

但它不会检查某个工作者是否忙碌，也不会自动管理任务完成与失败。

### 6.7 `cycle()` 会缓存输入吗？

会。

它在第一次遍历来源时保存遇到的元素，来源耗尽后再从保存内容重复。

如果输入很长，缓存也会很大。

如果输入本身无限，第一次遍历永远无法结束，还可能持续累积缓存。

因此，不要把无限生成器随意交给 `cycle()`。

### 6.8 空输入的循环会一直等待吗？

```python
from itertools import cycle

print(list(cycle([])))
```

得到空列表。

没有任何保存元素可以重复时，它会结束。

无限工具并不意味着所有输入都必然产生无限结果，仍要看具体定义。

## 第七章：筛选和截断，为什么需要不同函数？

### 7.1 “保留满足条件的元素”与“读到条件变化就停”

假设输入是：

```python
values = [1, 4, 6, 3, 8]
```

你可能有三种不同需求。

保留所有小于五的值，得到一、四、三。

只读取开头连续小于五的部分，得到一、四。

跳过开头连续小于五的部分，然后把后面全部保留，得到六、三、八。

这些结果不同，所以需要分别表达。

### 7.2 `filterfalse()`：保留条件为假的项

```python
from itertools import filterfalse

values = [1, 4, 6, 3, 8]
result = filterfalse(lambda value: value < 5, values)

print(list(result))
```

输出：

```text
[6, 8]
```

它对每个元素继续判断，保留条件结果为假的元素。

名称中的 false，就是这个方向。

与内置 `filter()` 相比，保留方向相反。

### 7.3 `predicate=None` 有什么含义？

```python
from itertools import filterfalse

print(list(filterfalse(None, [0, "", [], 1, "A"])))
```

使用对象自身的真假值判断，保留假值。

这不等于只保留 `False` 这个布尔对象。

零、空字符串、空列表等也具有假值语义。

如果业务只想识别某个特定状态，应写精确条件，不要把所有假值混在一起。

### 7.4 `takewhile()`：只取开头满足条件的连续前缀

```python
from itertools import takewhile

print(list(takewhile(lambda value: value < 5, [1, 4, 6, 3, 8])))
```

结果是一和四。

遇到六，条件第一次为假，就停止。

后面的三虽然小于五，也不会再进入结果。

因为这个工具的用途是截取连续前缀，不是遍历全体做过滤。

### 7.5 那个导致停止的六，还留在原来源里吗？

我们观察：

```python
from itertools import takewhile

source = iter([1, 4, 6, 3, 8])
head = list(takewhile(lambda value: value < 5, source))

print(head)
print(next(source))
```

输出：

```text
[1, 4]
3
```

六已经被取出来检查，因此被消费掉了。

它没有交给前缀结果，也不会自动退回原来源。

如果你需要把边界元素留给下一阶段，就要另外设计带回看缓冲的拆分过程，而不能直接认为 `takewhile()` 不会碰它。

### 7.6 `dropwhile()`：跳过开头前缀，之后全部保留

```python
from itertools import dropwhile

print(list(dropwhile(lambda value: value < 5, [1, 4, 6, 3, 8])))
```

得到：

```text
[6, 3, 8]
```

一和四被跳过。

六第一次不满足跳过条件，从它开始进入结果。

随后不再用这个条件排除三。

所以 `dropwhile()` 不是“删除所有小于五的数”。

### 7.7 如果条件一直为真，会发生什么？

有限来源会被全部跳过，然后结束。

无限来源若一直满足条件，消费者可能始终等不到第一个输出。

惰性不等于一定能很快给出结果，还要看工具为了找到可输出项需要读多少输入。

## 第八章：`compress()`——用另一串选择标记筛选数据

### 8.1 数据与标记一一对应

```python
from itertools import compress

names = ["小林", "小周", "小陈", "小何"]
selected = [True, False, True, False]

print(list(compress(names, selected)))
```

得到小林和小陈。

`compress(data, selectors)` 同时读取数据和标记，标记为真就保留对应数据。

### 8.2 为什么不直接写列表推导式？

你可以写：

```python
[name for name, keep in zip(names, selected) if keep]
```

当你需要最终列表时，这很清楚。

`compress()` 更直接表达“按另一来源的掩码选择”，并返回迭代器，便于继续接其他阶段。

选择应围绕代码读者是否容易看懂。

### 8.3 两边长度不一样呢？

任意一边先耗尽，就不再有完整对应项，结果也会停止。

```python
from itertools import compress

print(list(compress(["A", "B", "C"], [1, 0])))
```

只得到 A。

如果协议要求标记数量与数据数量完全一致，就要额外检查长度，或使用严格配对逻辑。

不能把较短输入导致的自然结束当作验证通过。

### 8.4 标记字符串 `"false"` 会被认为是假吗？

不会。

它是非空字符串，通常具有真值。

`compress()` 采用 Python 真假值规则，不负责解析配置文本。

如果标记来自 CSV 或环境变量，应先按明确规则转换成布尔值。

## 第九章：`map()` 与 `starmap()`——一条记录里的参数怎样传给函数？

### 9.1 普通 `map()` 每次传入一个元素

```python
values = [1, 2, 3]
print(list(map(lambda value: value * value, values)))
```

每次把一个数传给函数，得到平方。

`map()` 是内置工具，与 `itertools` 经常一起出现，理解它有助于理解后面的参数展开。

### 9.2 输入已经是参数元组，怎么办？

假设：

```python
pairs = [(2, 3), (3, 2), (10, 2)]
```

每条记录表示底数和指数。

希望依次计算：

```text
pow(2, 3)
pow(3, 2)
pow(10, 2)
```

可以使用：

```python
from itertools import starmap

pairs = [(2, 3), (3, 2), (10, 2)]
print(list(starmap(pow, pairs)))
```

输出：

```text
[8, 9, 100]
```

### 9.3 名称中的 star 是什么意思？

它可以联系调用函数时的星号解包：

```python
args = (2, 3)
print(pow(*args))
```

`starmap()` 每次取得一条记录，把其中各项作为位置参数展开给函数。

概念上类似：

```python
for args in pairs:
    result = pow(*args)
```

### 9.4 如果每条记录长度不对，会怎样？

被调用函数会在相应迭代步骤报告参数数量问题。

`starmap()` 不会自动补齐你缺少的参数，也不会根据字段名字猜测它们应该放到哪里。

如果记录来自外部文件，应先验证结构，再调用业务函数。

### 9.5 能直接展开关键字字典吗？

`starmap()` 展开的是每条记录的可迭代元素，作为位置参数。

把一个字典放进去，通常迭代出来的是键，不是自动执行 `function(**record)`。

需要关键字调用时，可以写一个明确的适配函数，或使用生成器表达式：

```python
records = [{"first": 2, "second": 3}, {"first": 4, "second": 5}]


def multiply(first, second):
    return first * second


results = (multiply(**record) for record in records)
print(list(results))
```

这段代码要求字典键正好符合函数参数政策。

多余或缺少键会导致相应错误，不应静默丢弃。

## 第十章：`accumulate()`——除了最后总和，还想看到每一步累计结果

### 10.1 `sum()` 只有最终值

假设每天学习分钟数是：

```python
minutes = [20, 30, 15, 40]
```

`sum(minutes)` 得到一百零五。

如果还想显示每天结束时累计多少，就需要保留中间结果。

### 10.2 默认是累计加法

```python
from itertools import accumulate

minutes = [20, 30, 15, 40]
print(list(accumulate(minutes)))
```

输出：

```text
[20, 50, 65, 105]
```

第一项是第一次累计结果，随后不断把新值加入已有累计值。

### 10.3 能指定初始累计值吗？

```python
from itertools import accumulate

changes = [10, -20, 5]
print(list(accumulate(changes, initial=100)))
```

输出：

```text
[100, 110, 90, 95]
```

初始值也作为第一项输出，所以结果比输入多一项。

空输入提供初始值时，也会输出这个初始值。

### 10.4 如果想追踪截至目前的最大值呢？

```python
from itertools import accumulate

scores = [60, 80, 70, 95, 90]
print(list(accumulate(scores, max)))
```

输出：

```text
[60, 80, 80, 95, 95]
```

第二个参数是一个接收“累计状态、新元素”的二元函数。

每次返回下一轮累计状态。

因此，它并不限于加法。

### 10.5 这会复制每个累计状态吗？

不会自动深拷贝。

如果你的累计函数反复修改并返回同一个列表，最终保存结果时，可能看到多个位置引用同一列表。

需要历史快照时，应让累计函数返回新的不可变状态或明确复制。

对于整数累计，值本身不可变，通常不需要担心这个问题。

### 10.6 与 `functools.reduce()` 有什么关系？

两者都可以通过二元操作归约序列。

`reduce()` 主要返回最终结果。

`accumulate()` 产生中间过程结果。

想画累计曲线、观察状态变化时，后者更贴近需求。

如果只要最终总和，内置 `sum()` 往往更清楚。

## 第十一章：`pairwise()`——相邻两条记录怎样配对？

### 11.1 比较前后两次读数

```python
from itertools import pairwise

readings = [10, 13, 12, 20]
print(list(pairwise(readings)))
```

输出：

```text
[(10, 13), (13, 12), (12, 20)]
```

每一项由相邻两个元素组成。

长度为四的有限输入，通常得到三对。

### 11.2 想计算相邻差值呢？

```python
from itertools import pairwise

readings = [10, 13, 12, 20]
differences = [current - previous for previous, current in pairwise(readings)]

print(differences)
```

得到三、负一、八。

可以用于记录增量、检测顺序变化和计算相邻时间间隔。

### 11.3 空输入或者只有一项呢？

没有完整相邻对，结果为空。

这通常比自己索引 `values[index + 1]` 更容易处理边界。

但如果业务规定至少需要两条数据，仍要主动验证，不能把空结果视为有效统计。

### 11.4 它和 `zip(values, values[1:])` 完全一样吗？

对普通小列表，结果含义相近。

但切片会生成相应副本，而且生成器不支持这种索引。

`pairwise()` 面向一般可迭代来源，只需保留用于相邻配对的必要状态。

它从 Python 3.10 起提供，本文基础环境可以使用。

## 第十二章：`groupby()`——相同键为什么没有全部合到一起？

### 12.1 先观察最容易误会的例子

```python
from itertools import groupby

for key, group in groupby("AAABBAC"):
    print(key, list(group))
```

输出：

```text
A ['A', 'A', 'A']
B ['B', 'B']
A ['A']
C ['C']
```

A 出现了两组。

这不是分组失败。

`groupby()` 按连续相邻的相同键分段，键改变就开始新组。

它不会自动把后来出现的 A 找回来，与最早的 A 合并。

### 12.2 和 SQL 的 GROUP BY 有什么区别？

SQL 常见聚合按键汇总全体匹配记录，不要求输入中相邻。

`itertools.groupby()` 是迭代过程中的连续分组。

如果你需要全体同键归组，可以先按相同键排序，再 `groupby()`；也可以使用 `defaultdict(list)` 收集。

两种方案的输入顺序、内存和输出方式不同。

### 12.3 给记录指定分组键

```python
from itertools import groupby

records = [
    {"class": "A", "name": "小林"},
    {"class": "B", "name": "小周"},
    {"class": "A", "name": "小陈"},
]

key_function = lambda record: record["class"]
ordered = sorted(records, key=key_function)

for class_name, group in groupby(ordered, key=key_function):
    print(class_name, [record["name"] for record in group])
```

排序和分组使用同一个键规则，才能让同组记录相邻。

如果排序按姓名，分组按班级，就不能期待相同班级自动成为一段。

### 12.4 排序会破坏原到达顺序吗？

会重新安排不同键的记录顺序。

Python 排序是稳定的，同键项保留相对顺序，但全局位置已经变化。

如果你处理的是连续状态、日志运行段或故障片段，排序反而可能破坏问题原本的时间结构。

所以要先问：我想要全局分类，还是连续分段？

### 12.5 `group` 是列表吗？

不是。

它也是迭代器，并且与外层分组工具共享底层来源。

我们观察：

```python
from itertools import groupby

groups = groupby("AAABB")
first_key, first_group = next(groups)
second_key, second_group = next(groups)

print(first_key)
print(list(first_group))
print(second_key)
print(list(second_group))
```

通常第一组再读取时已经空了。

外层为了进入 B 组，已经越过 A 段，旧组不能再回头独立读取。

### 12.6 如果要以后再使用每一组呢？

在当前组有效时，马上收集需要的数据：

```python
from itertools import groupby

saved = [(key, list(group)) for key, group in groupby("AAABB")]
print(saved)
```

或者只立即计算组统计，不保存完整成员。

```python
lengths = [(key, sum(1 for item in group)) for key, group in groupby("AAABB")]
```

选择取决于后面是否真的需要成员明细。

### 12.7 对组先求一次长度，再遍历成员，可以吗？

如果你通过消费迭代器求长度，组就已经耗尽。

需要两种结果时，可以先收集为列表，再 `len()` 和遍历。

或者在一次循环中同时累计你需要的所有统计。

这个问题与最早讲的“一次性迭代器”完全一致，只是现在藏在分组返回值里。

## 第十三章：`tee()`——让多个消费者观察同一来源

### 13.1 先看正确的分支方式

```python
from itertools import tee

source = iter([10, 20, 30])
first, second = tee(source, 2)

print(next(first))
print(next(first))
print(next(second))
```

得到十、二十、十。

这次两个分支拥有各自推进的位置。

### 13.2 它把原生成器完整复制了一份吗？

没有复制任意生成器的执行状态，也没有把来源重新运行两遍。

可以理解为：来源只向前取得值，较快分支先读过而较慢分支尚未读到的值，会被暂存以便后面补读。

这就是它能够提供相同元素序列的关键。

### 13.3 如果一个分支跑得很快，另一个一直不读呢？

缓存可能持续增长。

例如第一分支消费一百万条，第二分支还停在开头，就可能需要保留大量尚未被第二分支取走的数据。

所以 `tee()` 不等于零成本复制，也不能笼统称为恒定内存工具。

### 13.4 对象本身也复制了吗？

没有自动深拷贝每个元素。

```python
from itertools import tee

first, second = tee(iter([[1], [2]]))
item = next(first)
item.append(99)

print(next(second))
```

第二分支也可能看见同一个列表已经被修改。

如果分支应该相互隔离，需要另外设计不可变记录或复制策略。

### 13.5 创建分支后，还能继续直接消费 `source` 吗？

应避免这样混用。

直接从原来源取走的值没有按分支机制正常经过缓存，其他分支就可能丢失预期元素。

建立 `tee` 分支后，把后续读取责任交给分支，而不是再让原对象参与竞争消费。

### 13.6 多线程能各拿一个分支并行读吗？

`tee` 分支不是线程安全的。

即使原始来源可以安全读取，也不能推出共享这次 `tee()` 状态的分支支持并发使用。

需要多线程分发数据，应考虑明确同步的队列或消息传递设计。

### 13.7 什么时候列表反而更清楚？

来源有限且规模可控，又明确需要完整重读两次时，先保存为列表往往简单。

如果一个分支必须几乎全部读完后另一个才开始，`tee()` 的缓存优势也很有限。

它适合消费进度相对接近、确实希望按需分支的场景。

## 第十四章：`zip_longest()`——两边长度不一样，也想保留全部数据

### 14.1 普通 `zip()` 在短的一边结束

```python
names = ["A", "B", "C"]
scores = [95, 88]

print(list(zip(names, scores)))
```

C 没有对应分数，就没有进入结果。

如果你想把缺失也显示出来，可以使用：

```python
from itertools import zip_longest

print(list(zip_longest(names, scores, fillvalue="未提供")))
```

得到：

```text
[('A', 95), ('B', 88), ('C', '未提供')]
```

### 14.2 默认填充值为什么有时不够？

默认使用 `None`。

但真实输入本身也可能有 `None`，这时难以区分“原本空值”和“长度不足补位”。

可以使用专门哨兵：

```python
from itertools import zip_longest

missing = object()

for left, right in zip_longest([None, 2], [10], fillvalue=missing):
    if right is missing:
        print("右侧缺少对应项")
    else:
        print(left, right)
```

### 14.3 它会复制填充值吗？

不会为每个补位自动深拷贝。

如果用可变列表作为 `fillvalue`，多个缺失位置可能引用同一个对象。

一般使用不可变标记或专门哨兵更容易理解。

### 14.4 一边无限，另一边有限呢？

它会继续直到最长来源耗尽。

如果最长来源无限，结果也没有自然终点。

```python
from itertools import count, islice, zip_longest

pairs = zip_longest(["A", "B"], count(), fillvalue="缺失")
print(list(islice(pairs, 5)))
```

外面的 `islice()` 在这里承担结束责任。

### 14.5 如果长度不一致本来就是错误呢？

可以考虑内置 `zip(..., strict=True)`。

它表达“必须等长”的要求，Python 3.10 起可用。

`zip_longest()` 表达“保留全部并补位”，两者的业务政策不同，不能随意替换。

### 14.6 普通 `zip()` 结束时，会不会已经多读了另一边一项？

有可能，这与按顺序从各来源取下一项有关。

```python
left = iter([1, 2, 3])
right = iter(["A"])

print(list(zip(left, right)))
print(list(left))
```

前面的配对结果只有 `(1, "A")`。

但左边剩余通常是 `[3]`，二已经为了尝试组成下一对而被取出，随后才发现右边结束。

所以，“结果没有包含这个元素”不等于“原来源没有消费它”。

如果要继续使用配对后的剩余流，就必须考虑这个边界，不能凭输出长度推算每个输入游标的位置。

这也是为什么关键协议处理需要明确记录边界，而不只是把几个迭代工具连起来就认为所有未输出项都还在原处。

## 第十五章：`product()`——多个参数的所有搭配怎样生成？

### 15.1 先从两组选择开始

颜色有两种：

```python
colors = ["red", "blue"]
```

尺寸有三种：

```python
sizes = ["S", "M", "L"]
```

你希望列出每种颜色与每种尺寸的搭配。

```python
from itertools import product

for color, size in product(colors, sizes):
    print(color, size)
```

相当于一个嵌套循环。

第一组选择固定时，第二组依次变化；第二组跑完，再换第一组。

### 15.2 三个或者更多参数呢？

```python
from itertools import product

learning_rates = [0.01, 0.1]
batch_sizes = [16, 32]
seeds = [1, 2, 3]

for rate, batch_size, seed in product(learning_rates, batch_sizes, seeds):
    print(rate, batch_size, seed)
```

一共产生二乘二乘三，也就是十二种搭配。

不需要自己不断增加嵌套层级。

### 15.3 同一组候选重复选几次，怎样表达？

```python
from itertools import product

print(list(product("AB", repeat=2)))
```

输出：

```text
[('A', 'A'), ('A', 'B'), ('B', 'A'), ('B', 'B')]
```

`repeat=2` 表示把输入池重复作为两组位置选择。

这不是把最终结果列表再重复两遍。

### 15.4 返回结果是惰性的，输入也一定不预读吗？

这是一个必须讲清楚的例外。

`product()` 为了反复组合每一组候选，会先消费并保存各个有限输入池。

```python
from itertools import product


def candidates():
    print("准备 A")
    yield "A"
    print("准备 B")
    yield "B"


result = product(candidates(), [1, 2])
print("product 已建立")
print(next(result))
```

候选来源可能在“product 已建立”之前就被消费。

结果组合逐项产生，与输入候选池提前保存，是两个不同层次。

### 15.5 所以能传 `count()` 吗？

不应把无限来源直接传给需要完整保存候选池的 `product()`。

它可能一直消费而无法完成输入准备，还持续占用内存。

先明确限制候选数量：

```python
from itertools import count, islice, product

limited = islice(count(1), 3)
print(list(product(limited, ["A", "B"])))
```

这次输入池确定为一、二、三，是有限的。

### 15.6 为什么只用迭代器仍然可能计算很久？

假设十个参数，每个有十种选择。

候选池本身不大，但总搭配数是十的十次方。

不一次保存结果，解决的是结果列表内存，不会自动减少组合数量。

在参数搜索或测试矩阵中，先估算总量，再决定是否全部执行、抽样或限制预算。

## 第十六章：排列与组合，顺序和重复到底算不算不同？

### 16.1 三种常见问题

从 A、B、C 中选两人组成小组，A、B 与 B、A 是同一组。

从 A、B、C 中选冠军和亚军，A、B 与 B、A 不同。

两位密码，每位都可以从 A、B、C 选择，同一个字符可以重复。

这些需求分别对应不同工具。

### 16.2 `combinations()`：不重复选择位置，不强调选中后的排列

```python
from itertools import combinations

print(list(combinations("ABC", 2)))
```

输出：

```text
[('A', 'B'), ('A', 'C'), ('B', 'C')]
```

不会再产生 B、A，因为它与 A、B 属于同一种位置组合。

输出顺序依据输入顺序组织，不会自己把任意输入重新排序。

### 16.3 `permutations()`：不同顺序分别产生

```python
from itertools import permutations

print(list(permutations("ABC", 2)))
```

除了 A、B，也有 B、A。

每个位置从输入不同位置选择，不在同一个结果中重复使用相同输入位置。

不提供长度时，默认使用完整长度排列。

### 16.4 `combinations_with_replacement()`：允许重复选择

```python
from itertools import combinations_with_replacement

print(list(combinations_with_replacement("ABC", 2)))
```

结果包括 A、A，B、B，C、C。

但仍不把 A、B 与 B、A 当成两种不同排列。

如果顺序也不同算不同，就更接近 `product(..., repeat=...)` 的需求。

### 16.5 输入里有两个相等值，会自动去重吗？

```python
from itertools import combinations

print(list(combinations([1, 1, 2], 2)))
```

可能得到两个内容相同的 `(1, 2)`。

工具按输入位置区分元素，不自动按值去掉重复。

这对于“两个名字相同但身份不同的候选”可能正是正确行为。

如果业务要求按值唯一，应该先建立明确去重规则，再做组合。

### 16.6 为什么不直接把输入变成集合？

集合会丢掉重复和原始顺序，元素还必须可哈希。

如果你依赖输入顺序生成稳定报告，随手转集合可能改变结果排列。

字符串等可哈希数据想保留首次出现顺序时，可以考虑 `dict.fromkeys()` 去重。

如果记录身份由 ID 决定，就按 ID 去重，而不是把整个对象当成相同与否的唯一依据。

### 16.7 组合数量怎样估算？

```python
from math import comb, perm

print(comb(10, 3))
print(perm(10, 3))
```

十选三组合是一百二十种。

十个候选选三个有序位置，是七百二十种排列。

数量差异很快变大。

写程序之前先估算，比运行后发现生成了几十亿种结果更容易控制。

### 16.8 这些工具也需要有限输入吗？

排列和组合工具通常需要保存输入池，才能按位置生成后续结果。

不要把“结果一项项产生”理解为“可以直接接任何无限来源”。

面对来源类型不确定的通用函数，应明确要求有限候选集合，并设置合理规模限制。

## 第十七章：`batched()`——每次处理一小批记录

### 17.1 为什么要分批？

你可能需要每一百条数据保存一次，或者把大量请求分成小批提交。

逐条处理有时开销太大，一次全部处理又占用太多内存。

分批是在中间选择一个明确处理单位。

### 17.2 Python 3.12 起可以直接使用

```python
from itertools import batched

print(list(batched("ABCDEFG", 3)))
```

输出：

```text
[('A', 'B', 'C'), ('D', 'E', 'F'), ('G',)]
```

每一批是元组，最后一批可以不足指定长度。

`n` 必须是正数大小，不是“分成 n 组”。

### 17.3 最后一批必须完整，怎么办？

Python 3.13 起增加 `strict=True`：

```python
# Python 3.13+
from itertools import batched

# 预期 ValueError：最后一批不足三项。
print(list(batched("ABCDEFG", 3, strict=True)))
```

Python 3.12 有 `batched()`，但没有这个 `strict` 参数。

因此，不要把函数存在与某个新增参数也存在混为一谈。

### 17.4 为什么错误不是刚调用时就一定发现？

要知道最后一批是否完整，通常需要迭代到输入末尾。

前几批可能已经交给消费者，最后才发现不足。

如果消费者已经把前面批次写入数据库或发送出去，错误不会自动撤销已完成动作。

严格分批只检查批大小，不提供全任务事务。

### 17.5 Python 3.11 怎么练习分批？

可以写一个独立辅助函数，保持清楚的语义。

```python

def batch_records(iterable, size, *, strict=False):
    if type(size) is not int or size <= 0:
        raise ValueError("size 必须是正整数")

    batch = []
    for item in iterable:
        batch.append(item)
        if len(batch) == size:
            yield tuple(batch)
            batch = []

    if batch:
        if strict:
            raise ValueError("最后一批不足指定长度")
        yield tuple(batch)


print(list(batch_records(range(7), 3)))
```

这段教学实现不是修改标准库，也不声称与所有底层实现细节完全相同。

它让你在旧版本中理解按需组批、尾批和消费阶段报错。

### 17.6 想得到三批，而不是每批三项呢？

这是另一个问题。

如果输入总量已知，可以先计算每批大小和余数分配规则。

对于未知长度的流，想尽量等分成固定批数，通常需要先知道总长度或保存额外数据。

所以不要仅凭“分批”两个字，就把不同需求交给同一个参数。

## 第十八章：如何判断一个工具会保存多少状态？

### 18.1 不要只看返回类型

返回迭代器，说明结果可以逐步消费。

它不直接告诉你内部保存了多少输入。

`cycle()` 会保存第一轮元素。

`tee()` 会保存分支进度差需要的数据。

`product()`、排列和组合会保存有限输入池。

`groupby()` 的组共享来源，保存组列表则由你主动增加内存。

### 18.2 很多工具只需要少量当前状态

`pairwise()` 需要前一项。

简单 `accumulate()` 需要当前累计状态，但你自己选的状态对象仍可能越来越大。

`batched()` 需要当前批。

`islice()` 需要位置计数，并不为跳过元素自动保留可回看历史。

### 18.3 `sorted()` 放进管道会怎样？

```python
ordered = sorted(source, key=key_function)
```

普通全局排序需要取得有限输入并保存结果。

后面再接惰性 `groupby()`，也不能改变前面的排序已经收集了全部数据这一事实。

理解整条管道的内存，必须检查每个阶段。

### 18.4 下游保存结果也会改变整体内存

```python
results = list(pipeline)
```

这一步主动保存所有输出。

如果你本来只需要写文件、求和或逐条发送，可以直接消费，而不先收集列表。

但如果后续确实要随机访问或多次展示，保存列表就是合理选择，不必为了“流式”强迫所有任务都只读一次。

### 18.5 打开文件的时机也属于状态管理

迭代器可能在内存里还存在，但它依赖的文件已经关闭。

```python
# 反面例子：返回后文件已经关闭。
def bad_lines(path):
    with open(path, encoding="utf-8") as source:
        return iter(source)
```

需要由函数内部管理读取时，可以在 `with` 内逐行 `yield`，并让消费方明确管理生成器的提前关闭。

需要更直观的所有权时，就在调用者的 `with` 中建立和消费管道。

## 第十九章：完整实践一——把两份学习记录接成按需处理管道

### 19.1 问题从两份文件开始

你有周一和周二的学习日志，每行一个 JSON 对象。

想依次读取它们，检查字段，筛选至少二十分钟的记录，并且只预览前三条有效匹配。

目录关系是：

```text
practice/
├── monday.jsonl
└── tuesday.jsonl
```

我们希望连接的是“记录流”，而不是先把两个文件完整读入两个大列表再相加。

### 19.2 先把输入约定写清楚

每行包含 `name`、`topic`、`minutes`。

姓名和主题要求非空字符串。

分钟数要求一到一千四百四十之间的整数，不接受布尔值。

坏行记录到错误列表，然后继续。

“只预览前三条”意味着后面的记录可能尚未读取，因此错误列表也只包含已经实际遇到的错误。

这一点非常重要：提前停止的预览，不能假装已经审计完整文件。

### 19.3 完整程序

```python
from itertools import chain, islice
from pathlib import Path
from tempfile import TemporaryDirectory
import json


def parse_json_lines(stream, source_name, problems):
    for line_number, line in enumerate(stream, start=1):
        if not line.strip():
            problems.append({
                "source": source_name,
                "line": line_number,
                "message": "空白行",
            })
            continue

        try:
            obj = json.loads(line)
        except json.JSONDecodeError as error:
            problems.append({
                "source": source_name,
                "line": line_number,
                "message": error.msg,
            })
            continue

        yield source_name, line_number, obj


def validate_records(entries, problems):
    for source_name, line_number, obj in entries:
        try:
            if not isinstance(obj, dict):
                raise ValueError("记录必须是对象")

            name = obj.get("name")
            topic = obj.get("topic")
            minutes = obj.get("minutes")

            if not isinstance(name, str) or not name.strip():
                raise ValueError("name 必须是非空字符串")
            if not isinstance(topic, str) or not topic.strip():
                raise ValueError("topic 必须是非空字符串")
            if type(minutes) is not int or not 1 <= minutes <= 1440:
                raise ValueError("minutes 必须是 1 到 1440 的整数")
        except ValueError as error:
            problems.append({
                "source": source_name,
                "line": line_number,
                "message": str(error),
            })
            continue

        yield {
            "name": name.strip(),
            "topic": topic.strip(),
            "minutes": minutes,
            "source": source_name,
            "line": line_number,
        }


def preview_learning(first_path, second_path, limit=3):
    if type(limit) is not int or limit < 0:
        raise ValueError("limit 必须是非负整数")

    first_path = Path(first_path)
    second_path = Path(second_path)
    problems = []

    with first_path.open(encoding="utf-8") as first_stream:
        with second_path.open(encoding="utf-8") as second_stream:
            entries = chain(
                parse_json_lines(first_stream, first_path.name, problems),
                parse_json_lines(second_stream, second_path.name, problems),
            )
            validated = validate_records(entries, problems)
            selected = (
                record for record in validated
                if record["minutes"] >= 20
            )
            preview = list(islice(selected, limit))

    return {"preview": preview, "encountered_problems": problems}


def write_jsonl(path, records):
    with Path(path).open("w", encoding="utf-8", newline="\n") as target:
        for record in records:
            target.write(json.dumps(record, ensure_ascii=False, allow_nan=False))
            target.write("\n")


def main():
    with TemporaryDirectory(prefix="itertools-learning-") as directory:
        root = Path(directory)
        first = root / "monday.jsonl"
        second = root / "tuesday.jsonl"

        write_jsonl(first, [
            {"name": "小林", "topic": "Python", "minutes": 35},
            {"name": "小周", "topic": "SQL", "minutes": 10},
            {"name": "小陈", "topic": "Linux", "minutes": 25},
        ])
        write_jsonl(second, [
            {"name": "小林", "topic": "文件", "minutes": 30},
            {"name": "小周", "topic": "路径", "minutes": 40},
        ])

        result = preview_learning(first, second, limit=3)
        print(json.dumps(result, ensure_ascii=False, indent=2, allow_nan=False))


if __name__ == "__main__":
    main()
```

### 19.4 第一次请求结果时，哪一层先动？

最外层 `list(islice(...))` 要求第一个预览项。

`islice` 向筛选生成器要一个匹配项。

筛选生成器向校验阶段要记录。

校验阶段向 `chain` 要记录。

`chain` 从第一份文件的解析器取得第一行。

数据由底层产生，但推进请求来自下游消费者。

这个方向想清楚，惰性管道就不神秘了。

### 19.5 十分钟记录会占用一个预览名额吗？

不会。

先筛选，再在匹配结果上 `islice(..., 3)`。

因此限制的是三条通过筛选的结果，不是原始文件的前三行。

如果你把 `islice()` 放到筛选前面，含义就变成只检查前三条原始有效记录，最终匹配数可能少于三。

工具顺序就是业务顺序，不能随意交换。

### 19.6 第三条匹配来自第二份文件，意味着什么？

第一份文件要先被读取到需要的程度。

只有第一来源耗尽，`chain` 才进入第二来源。

它不会因为第二份文件里有更长学习时间，就提前把那条取出来。

想按时间排序，需要另外的排序或有序合并机制。

### 19.7 为什么文件保持在外层 `with` 中？

这段代码在文件仍打开的范围内，建立并消费所有依赖这些流的工具。

预览结束后离开上下文，两份文件都关闭。

返回的是已经收集的少量普通字典，不再依赖文件保持打开。

这样资源归属很清楚。

### 19.8 如果我想返回整个 `selected` 迭代器呢？

直接从这里返回是不合适的，因为函数离开 `with` 后文件会关闭。

调用者以后再迭代，底层读取可能失败。

你可以让调用者管理文件，或把打开和逐行产生过程设计成明确可关闭的生成器。

不要为了让返回值看起来流式，就隐藏它依赖的资源生命周期。

## 第二十章：完整实践二——连续状态日志怎样分段？

### 20.1 这次需要保留原顺序

假设系统状态依次是：

```text
OK、OK、ERROR、ERROR、OK、WARNING、WARNING
```

你想知道每个连续状态段从哪条到哪条、持续多少条记录。

这里相同状态在后面再次出现，应当是新的一段。

因此，直接 `groupby()` 很合适，先排序反而会把时间结构破坏。

### 20.2 完整程序

```python
from itertools import groupby
import json


def validate_events(events):
    previous_time = None

    for index, event in enumerate(events):
        if not isinstance(event, dict):
            raise ValueError(f"第 {index} 条不是对象")

        timestamp = event.get("time_seconds")
        state = event.get("state")

        if type(timestamp) is not int or timestamp < 0:
            raise ValueError(f"第 {index} 条时间必须是非负整数")
        if not isinstance(state, str) or not state:
            raise ValueError(f"第 {index} 条状态不正确")
        if previous_time is not None and timestamp < previous_time:
            raise ValueError("事件必须按时间非递减排列")

        previous_time = timestamp
        yield index, timestamp, state


def summarize_state_runs(events):
    checked = validate_events(events)
    runs = []

    for state, group in groupby(checked, key=lambda item: item[2]):
        first_index = None
        first_time = None
        last_index = None
        last_time = None
        count = 0

        for index, timestamp, ignored_state in group:
            if first_index is None:
                first_index = index
                first_time = timestamp
            last_index = index
            last_time = timestamp
            count += 1

        runs.append({
            "state": state,
            "first_index": first_index,
            "last_index": last_index,
            "first_time_seconds": first_time,
            "last_time_seconds": last_time,
            "observed_span_seconds": last_time - first_time,
            "record_count": count,
        })

    return runs


def main():
    states = ["OK", "OK", "ERROR", "ERROR", "OK", "WARNING", "WARNING"]
    events = [
        {"time_seconds": index * 5, "state": state}
        for index, state in enumerate(states)
    ]

    result = summarize_state_runs(events)
    print(json.dumps(result, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```

### 20.3 为什么没有把每组变成列表？

我们只需要首尾位置、首尾时间和数量。

一次遍历即可累计这些状态，没有必要保留所有成员。

如果后续要显示每段完整日志明细，再考虑保存列表。

这正是共享组迭代器的正确使用方式：当前组有效时，把需要的信息计算完。

### 20.4 `observed_span_seconds` 为什么这样命名？

它只表示这一组首条和末条观测之间的时间跨度。

假设最后一条 ERROR 在十五秒，下一条 OK 在二十秒，真正恢复可能发生在这五秒之间。

单凭离散观测，不能断言错误精确持续多久。

字段命名说明我们实际观察到了什么，避免把简单差值夸大成准确状态持续时间。

### 20.5 为什么要验证时间顺序，而不是自动排序？

乱序事件可能表示上游采集错误，或者来源合并政策不明确。

本例要求输入已经按时间顺序排列，所以发现乱序就报告。

如果你选择排序，应说明这会重新安排相同时间和不同来源的记录，并设置稳定的次级规则。

不能在没有说明的情况下悄悄改变日志顺序，再声称报告保留了原始经过。

### 20.6 只一条记录的状态段怎么办？

首尾时间相同，因此观测跨度为零。

这不证明该状态实际只存在零秒，只表示我们在这一段里只有一个观测点。

理解数据来源的采样方式，和理解 `groupby()` 的语法同样重要。

## 第二十一章：完整实践三——分批处理，并把失败边界记录清楚

### 21.1 分批不只是把列表切几段

你可能要把记录按批转换并输出。

需要决定：每批多少条，最后不足一批怎么办，某条非法时整批失败还是跳过，以及已经完成的批次是否保留。

这个练习采用清楚的政策：每批最多三条；非法记录单独报告；有效记录按批输出；结果在临时目标完成后才替换正式文件。

### 21.2 完整程序，兼容 Python 3.11

```python
from pathlib import Path
from tempfile import NamedTemporaryFile, TemporaryDirectory
import json


def batches(iterable, size):
    if type(size) is not int or size <= 0:
        raise ValueError("批大小必须是正整数")

    current = []
    for item in iterable:
        current.append(item)
        if len(current) == size:
            yield tuple(current)
            current = []
    if current:
        yield tuple(current)


def normalize_numbers(values, problems):
    for index, value in enumerate(values):
        if type(value) is not int:
            problems.append({"index": index, "message": "必须是整数"})
            continue
        if not 0 <= value <= 1_000_000:
            problems.append({"index": index, "message": "数值超出范围"})
            continue
        yield {"index": index, "number": value, "square": value * value}


def save_batched_report(target_path, values, batch_size=3):
    target_path = Path(target_path).absolute()
    target_path.parent.mkdir(parents=True, exist_ok=True)
    problems = []
    processed = 0
    batch_count = 0
    temporary_path = None

    try:
        with NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            newline="\n",
            dir=target_path.parent,
            prefix=target_path.name + ".",
            suffix=".tmp",
            delete=False,
        ) as target:
            temporary_path = Path(target.name)
            checked = normalize_numbers(values, problems)

            for batch_number, batch in enumerate(batches(checked, batch_size), start=1):
                document = {
                    "batch": batch_number,
                    "count": len(batch),
                    "records": list(batch),
                }
                target.write(json.dumps(document, ensure_ascii=False, allow_nan=False))
                target.write("\n")
                processed += len(batch)
                batch_count += 1

        temporary_path.replace(target_path)
    finally:
        if temporary_path is not None:
            temporary_path.unlink(missing_ok=True)

    return {
        "processed": processed,
        "batches": batch_count,
        "problems": problems,
    }


def main():
    with TemporaryDirectory(prefix="itertools-batches-") as directory:
        target = Path(directory) / "result.jsonl"
        values = [2, 3, "bad", 4, 5, True, 6, 7, 8]
        summary = save_batched_report(target, values, batch_size=3)

        print(json.dumps(summary, ensure_ascii=False, indent=2))
        print(target.read_text(encoding="utf-8"))


if __name__ == "__main__":
    main()
```

### 21.3 为什么先校验，再组批？

本例的批大小指有效记录数量。

如果先把原始数据分批，再在每批内部跳过坏项，最终批的有效数量就可能不同。

两种都可以，但要把“批次”究竟针对原始记录还是有效记录说清楚。

这里三个正常记录才构成一批，便于下游按有效数据处理。

### 21.4 为什么用 `enumerate()` 给批编号？

批编号从一开始，更适合报告阅读。

记录内部仍保存原始索引，方便追踪是哪一个输入产生结果。

批编号与原始行号是不同坐标，不应混用。

### 21.5 这里的临时替换保证了什么？

我们在生成全部输出后，关闭临时文件，再替换正式目标。

这能减少正式文件被读者观察到只写了一部分的风险。

它不保留原目标全部元数据，不解决多写入者竞争，也不是断电持久性或跨多个文件的事务保证。

本例重点是让管道中途失败时的发布时机清楚。

### 21.6 如果改成每批直接发给远端服务呢？

那么前几批成功后，后面失败，并不会因为 Python 抛异常就让远端自动撤销。

你需要根据服务契约设计幂等标识、重试和恢复位置。

分批工具只负责边界，不负责分布式业务事务。

### 21.7 为什么这个例子没有强求 `batched()`？

为了让 Python 3.11 也能运行完整实践，我们使用自己的有限辅助函数。

在 Python 3.12 及以上，可以换成 `itertools.batched()`，但仍要保留本例关于验证顺序、错误报告和发布时机的规则。

换工具不会自动替你完成业务决策。

## 第二十二章：完整实践四——生成有规模限制的参数试验矩阵

### 22.1 这个需求在哪里出现？

你可能想比较几种批大小、超时设置或文本切块长度。

每个参数有几个候选值，所有搭配都要跑一遍小测试。

这里先生成配置，再调用一个确定性评分函数模拟试验，避免依赖外部服务和费用。

### 22.2 先估算，再开始

```python
from itertools import product
from math import prod

choices = {
    "chunk_size": [200, 500, 1000],
    "overlap": [0, 50],
    "workers": [1, 2],
}

print(prod(len(values) for values in choices.values()))
```

得到十二种。

这个估算非常便宜，却能提前发现组合爆炸。

### 22.3 完整实现

```python
from itertools import product
from math import prod
import json


def prepare_choices(choices, max_configs):
    if not isinstance(choices, dict):
        raise TypeError("choices 必须是字典")
    if type(max_configs) is not int or max_configs <= 0:
        raise ValueError("max_configs 必须是正整数")

    names = tuple(choices)
    pools = []

    for name in names:
        if not isinstance(name, str) or not name:
            raise ValueError("参数名必须是非空字符串")
        values = choices[name]
        if not isinstance(values, (list, tuple)):
            raise ValueError("本程序只接受已知有限的列表或元组候选池")
        pools.append(tuple(values))

    total = prod(len(pool) for pool in pools)
    if total > max_configs:
        raise ValueError(f"预计 {total} 种配置，超过上限 {max_configs}")

    return names, tuple(pools), total


def iter_configurations(choices, max_configs=1000):
    names, pools, total = prepare_choices(choices, max_configs)

    for index, values in enumerate(product(*pools), start=1):
        yield {
            "trial": index,
            "total_trials": total,
            "parameters": dict(zip(names, values, strict=True)),
        }


def valid_text_config(config):
    chunk_size = config["chunk_size"]
    overlap = config["overlap"]
    workers = config["workers"]

    if any(type(value) is not int for value in [chunk_size, overlap, workers]):
        return False
    return chunk_size > 0 and 0 <= overlap < chunk_size and workers > 0


def simulate_trial(config):
    chunk_size = config["chunk_size"]
    overlap = config["overlap"]
    workers = config["workers"]

    estimated_chunks = (10_000 + chunk_size - overlap - 1) // (chunk_size - overlap)
    estimated_cost = estimated_chunks * (1 + overlap / chunk_size)
    estimated_speed = workers / estimated_chunks

    return {
        "estimated_chunks": estimated_chunks,
        "demonstration_cost": round(estimated_cost, 3),
        "demonstration_speed": round(estimated_speed, 6),
    }


def run_matrix(choices, max_configs=1000):
    if not isinstance(choices, dict) or set(choices) != {"chunk_size", "overlap", "workers"}:
        raise ValueError("这个文本试验要求 chunk_size、overlap、workers 三个参数")
    results = []
    rejected = []

    for trial in iter_configurations(choices, max_configs=max_configs):
        config = trial["parameters"]
        if not valid_text_config(config):
            rejected.append(trial)
            continue

        results.append({
            **trial,
            "result": simulate_trial(config),
        })

    return {"results": results, "rejected": rejected}


def main():
    choices = {
        "chunk_size": [200, 500, 1000],
        "overlap": [0, 50],
        "workers": [1, 2],
    }
    report = run_matrix(choices, max_configs=100)
    print(json.dumps(report, ensure_ascii=False, indent=2, allow_nan=False))


if __name__ == "__main__":
    main()
```

### 22.4 为什么只接受列表和元组候选？

这是这个应用主动制定的输入限制。

这样可以直接知道候选规模，也避免把无限生成器交给需要保存池的组合工具。

通用 `product()` 接受更广的可迭代对象，但你的业务封装可以提供更严格、容易理解的契约。

### 22.5 这些评分数字可以当作真实性能结论吗？

不能。

`simulate_trial()` 只是确定性的教学计算，用来展示每组参数如何形成一条结果。

真实文本切块质量、检索效果和并发性能需要实际输入、测量和评价指标。

示例字段因此明确带有 demonstration 或 estimated，不把公式演示伪装成实验结果。

### 22.6 参数之间的约束放在哪里？

`product()` 只负责生成所有位置搭配。

例如 `overlap < chunk_size` 是业务约束，需要额外检查。

你可以先生成再筛选，也可以在构造候选时减少不合法组合。

如果非法比例极高，盲目生成全部再排除可能很浪费，应重新设计搜索空间。

### 22.7 如果没有任何参数，为什么还可能有一项？

数学上零组选择的笛卡尔积包含一个空搭配。

`product()` 没有输入时会产生一个空元组。

这可用于“没有可变参数，执行一次默认试验”。

如果你的应用认为空参数表应报错，就在对应业务入口明确拒绝，而不是把底层规则当成错误。

这里通用的 `iter_configurations({})` 可以产生空配置，但特定文本试验的 `run_matrix()` 明确要求三个已约定参数，所以会拒绝空参数表。通用枚举能力与具体试验契约分别成立。

### 22.8 结果为什么保存列表？

这次报告规模有上限，而且后面可能排序比较，所以保存结果列表是明确选择。

大规模试验可以逐条写 JSON Lines，或记录到数据库，再按需要查询。

是否保存结果应看任务，而不是把所有中间迭代器都强行收集。

## 第二十三章：完整实践五——累计账目与相邻变化报告

### 23.1 我们处理什么？

一份练习账目有初始余额和一串整数分单位变动。

希望得到每步余额、相邻变动差值，以及截至每一步见过的最大余额。

使用整数最小单位，是为了把这篇重点放在迭代过程，避免用二进制浮点演示精确金额。

这不是完整金融账务系统，只是受控整数数据的累计实践。

### 23.2 完整实现

```python
from itertools import accumulate, pairwise
import json


def validate_changes(changes):
    for index, amount in enumerate(changes):
        if type(amount) is not int:
            raise ValueError(f"第 {index} 条变动必须是整数分")
        if not -1_000_000 <= amount <= 1_000_000:
            raise ValueError(f"第 {index} 条变动超出练习范围")
        yield amount


def build_balance_report(initial_cents, changes):
    if type(initial_cents) is not int:
        raise ValueError("初始余额必须是整数分")

    checked = list(validate_changes(changes))
    balances = list(accumulate(checked, initial=initial_cents))
    running_highs = list(accumulate(balances, max))

    steps = []
    for step, (before, after) in enumerate(pairwise(balances), start=1):
        amount = checked[step - 1]
        steps.append({
            "step": step,
            "change_cents": amount,
            "before_cents": before,
            "after_cents": after,
            "running_high_cents": running_highs[step],
        })

    return {
        "initial_cents": initial_cents,
        "final_cents": balances[-1],
        "changes_count": len(checked),
        "steps": steps,
    }


def main():
    report = build_balance_report(10_000, [2_000, -500, -3_000, 1_000])
    print(json.dumps(report, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```

### 23.3 为什么余额列表比变动列表多一项？

因为 `initial=initial_cents` 被作为第一项输出。

四条变动产生五个余额状态：操作前初始状态，加上每次操作后的状态。

`pairwise(balances)` 恰好产生四组前后状态，与四条变动对应。

这个对应关系解释清楚后，索引就不再是猜出来的。

### 23.4 为什么先保存 `checked`？

这个报告需要反复使用变动和余额，并随机访问对应位置。

输入规模受控时，保存列表比使用多个 `tee` 分支再让进度产生很大差异更直接。

如果只需要逐步输出，可以把当前余额、最大值和当前变动放在一次循环里，避免保存全部历史。

这是计算结果需求决定的数据结构选择。

### 23.5 没有变动时，`balances[-1]` 会失败吗？

不会，因为初始值仍然存在。

报告的最终余额就是初始余额，步骤列表为空。

如果没有提供 `initial`，空来源的累计结果也会为空，后续取末尾就需要额外处理。

### 23.6 怎样检查累计结果与原始变动一致？

对每组相邻余额，应该满足：

```text
after - before == change
```

这是一条与实现细节相对独立的业务不变量。

验证代码时，检查这种关系比只比较某个打印字符串更有意义。

## 第二十四章：完整实践六——两个分支同步消费，保留原文与规范化结果

### 24.1 一个来源，两种视角

假设你拿到一串原始用户名文本。

一条分支负责保留原始输入，另一条负责去掉两端空白、检查内容并生成规范化名字。

最后把原文、规范化结果和错误并排输出。

这个例子用 `tee()` 展示分支同步消费，也让你观察“共享来源”不等于“重复执行来源”。

### 24.2 完整程序

```python
from itertools import tee
import json


def original_names(values, observations):
    for index, value in enumerate(values):
        observations.append(index)
        yield value


def normalize_name(value):
    if not isinstance(value, str):
        return {"value": None, "error": "必须是字符串"}

    cleaned = value.strip()
    if not cleaned:
        return {"value": None, "error": "不能是空白名字"}
    if len(cleaned) > 40:
        return {"value": None, "error": "名字太长"}

    return {"value": cleaned, "error": None}


def make_name_audit(values):
    observations = []
    source = original_names(values, observations)
    raw_branch, normalization_branch = tee(source, 2)
    normalized = map(normalize_name, normalization_branch)
    results = []

    for index, (raw, result) in enumerate(
        zip(raw_branch, normalized, strict=True)
    ):
        results.append({
            "index": index,
            "original": raw,
            "normalized": result["value"],
            "error": result["error"],
        })

    return {
        "results": results,
        "source_read_indices": observations,
    }


def main():
    values = ["  Alice  ", "小林", "   ", 123]
    report = make_name_audit(values)
    print(json.dumps(report, ensure_ascii=False, indent=2, allow_nan=False))


if __name__ == "__main__":
    main()
```

### 24.3 为什么这次缓存进度差通常很小？

`zip()` 每轮从原文分支取得一项，再从规范化分支取得对应一项。

两边基本同步推进，不会故意让一边先跑完整个来源。

这与“先 `list(first_branch)`，再 `list(second_branch)`”的使用方式不同。

### 24.4 `source_read_indices` 为什么每个索引只出现一次？

底层生成器只产生每条值一次。

另一个分支需要时，取得的是共享分支机制保存的相应元素。

这个观察列表只服务小型教学验证，本身也会保存全部索引。

真实大型管道若不需要完整追踪，应改成计数或限量日志。

### 24.5 为什么规范化函数总返回一条结果？

为了保持两条分支一一对应。

坏名字也返回带错误的结果，没有被直接过滤掉。

如果规范化分支删除了某些项，再与原文分支简单 `zip()`，后面的行就可能错位。

这是数据管道中很重要的“基数变化”问题：过滤会改变条数，映射通常保持一项进、一项出。

### 24.6 这项任务一定需要 `tee()` 吗？

不一定。

一个普通循环里同时保留原文并调用规范化函数，也很清楚，甚至更简短。

这里使用分支，是为了帮助你理解已有独立处理阶段怎样在同一来源上配合。

工具知识应让你能够判断取舍，而不要求每个程序都绕成复杂管道。

### 24.7 `strict=True` 能发现内容错位吗？

它主要检查长度不一致，不知道两项在业务上是否属于同一记录。

如果两边长度一样却顺序不同，它仍然可能顺利结束。

需要跨阶段可靠关联时，应保留稳定记录 ID，并按 ID 验证对应关系。

## 第二十五章：需要边界元素时，怎样拆分前缀？

### 25.1 从 `takewhile()` 的消费行为继续问

前面看到：导致条件第一次为假的元素已经被消费。

如果你的协议分成“头部”和“正文”，那个边界元素恰好应该属于正文，就不能丢掉。

我们可以自己写一个小型拆分函数，把边界值接回剩余流。

### 25.2 实现

```python
from itertools import chain


def split_prefix(iterable, predicate):
    iterator = iter(iterable)
    prefix = []

    for item in iterator:
        if predicate(item):
            prefix.append(item)
        else:
            remainder = chain((item,), iterator)
            return prefix, remainder

    return prefix, iter(())


header, body = split_prefix(
    ["# title", "# author", "first paragraph", "second paragraph"],
    lambda line: line.startswith("#"),
)

print(header)
print(list(body))
```

### 25.3 为什么 `chain((item,), iterator)` 里有一个逗号？

`(item,)` 是只有一个元素的元组。

如果只写 `(item)`，通常只是括号包住原对象，不会创建单元素元组。

我们要先把边界值作为一整项交出去，再接原来源剩余部分。

若 `item` 是字符串，错误地直接把它当第一个可迭代来源，可能把边界行拆成字符。

### 25.4 为什么这个函数立即读取前缀？

它要返回一份已经确定的前缀列表，所以调用时必须走到边界或来源末尾。

如果前缀无限，函数永远无法返回。

如果前缀非常大，列表会占用相应内存。

这段辅助函数明确适合有限且规模合理的头部，不应被宣传为所有输入都恒定内存。

### 25.5 返回的正文可以重读吗？

不能默认重读。

正文仍是连接迭代器，消费一次就向前推进。

如果你要多次处理正文，要保存内容、重新打开来源或设计其他明确机制。

保住一个边界元素，没有改变迭代器整体的一次性语义。

## 第二十六章：管道顺序改变，结果为什么也改变？

### 26.1 先限制十条，再过滤，与先过滤再限制十条

假设来源有很多记录，其中只有部分是错误事件。

下面表达“只检查前十条原始记录，再找其中错误”：

```python
from itertools import islice

first_ten = islice(records, 10)
errors = (record for record in first_ten if record["level"] == "ERROR")
```

下面表达“持续查找，直到取得最多十条错误事件”：

```python
errors = (record for record in records if record["level"] == "ERROR")
first_ten_errors = islice(errors, 10)
```

前者最多读取十条原始记录。

后者为了找到十条错误，可能读取很多条；如果无限来源从来没有错误，还可能一直找不到足够结果。

同样两个工具，顺序决定不同资源边界和业务含义。

### 26.2 先全局排序，再分组，会增加什么成本？

全局排序需要取得有限输入。

因此，即使后面只想看第一个组，前面的 `sorted()` 通常也已经消费全部来源。

如果来源本来按组键有序，就可以直接连续分组，减少不必要的全局收集。

是否已经有序应由输入契约或验证确认，不能仅凭几条样例看起来整齐就假设。

### 26.3 `any()` 和 `all()` 也会推进来源吗？

会。

```python
source = iter([0, 1, 2, 3])

print(any(source))
print(next(source))
```

`any()` 遇到第一项真值就停止。

零是假，接着一是真，因此它已经消费零和一，下一项是二。

`all()` 遇到第一项假值就停止，也会留下尚未消费部分。

如果你把它们用作“提前检查一下”，必须意识到检查本身改变了迭代位置。

### 26.4 `value in iterator` 也只是看一眼吗？

不是。

```python
source = iter([10, 20, 30, 40])
print(20 in source)
print(list(source))
```

成员搜索会向前消费，直到找到目标或耗尽。

后面的列表通常只剩三十和四十。

如果需要不改变后续输入的查询，要保存有限数据、创建新来源，或使用明确分支机制并承担缓存成本。

### 26.5 记录错误的列表会不会破坏流式内存优势？

如果一直保存所有错误，会。

解决办法可以是只保留前一百个示例，同时累计总错误数。

```python
class ProblemCollector:
    def __init__(self, max_examples=100):
        if type(max_examples) is not int or max_examples < 0:
            raise ValueError("max_examples 必须是非负整数")
        self.max_examples = max_examples
        self.total = 0
        self.examples = []

    def add(self, problem):
        self.total += 1
        if len(self.examples) < self.max_examples:
            self.examples.append(problem)


collector = ProblemCollector(max_examples=2)
for number in range(5):
    collector.add({"line": number, "message": "示例错误"})

print(collector.total)
print(collector.examples)
```

这次总数为五，但只保存两个错误示例。

输出应明确区分总错误数和保留示例数，避免用户把列表长度当成完整错误数量。

### 26.6 在管道上加日志，应该放哪里？

你可以写观察生成器：

```python

def observe(iterable, label):
    for item in iterable:
        print(label, repr(item))
        yield item
```

放在过滤之前，看到的是原始进入项。

放在过滤之后，看到的是通过条件的项。

日志的位置同样体现你想观察哪个阶段。

真实数据可能含凭证或隐私，不应无条件 `repr()` 打印所有输入；教学观察只使用受控样例。

### 26.7 为什么不要用计时结论替代行为验证？

一个管道跑得很快，可能是因为它根本没有被消费。

只计时 `results = map(...)`，与计时 `list(map(...))`，测量的不是同一项工作。

比较性能时，要让两个方案真正执行相同的数据处理和输出要求。

先确认结果正确、消费范围正确，再讨论时间和内存。

## 第二十七章：动手练习与参考答案

### 27.1 练习一：只查看无限编号中的五项

要求：从一百开始，每次增加三，只取得五项，不允许把无限来源直接整体转列表。

```python
from itertools import count, islice

numbers = count(100, 3)
result = list(islice(numbers, 5))

print(result)
```

结果是一百、一百零三、一百零六、一百零九、一百一十二。

下一次从 `numbers` 读取，会继续到一百一十五。

`islice` 并没有复制原来源。

### 27.2 练习二：每隔一项取得一项，并观察原来源位置

```python
from itertools import islice

source = iter(range(12))
result = list(islice(source, 1, 9, 2))

print(result)
print(next(source))
```

结果是一、三、五、七。

切片被完整消费后，原来源已经向前经过停止边界之前的部分，下一项是九。

被跳过的偶数没有保存在可回看的视图里。

### 27.3 练习三：连接多批记录，保留记录内部结构

```python
from itertools import chain

batches = [
    [("A", 1), ("B", 2)],
    [],
    [("C", 3)],
]

records = list(chain.from_iterable(batches))
print(records)
```

展开一层批次以后，每个二元组仍然是完整记录。

如果继续错误地多展开一层，就会把名字和值拆散。

先画出输入嵌套层级，再决定展开几层，通常比不断试不同星号更清楚。

### 27.4 练习四：识别前缀截取和全体过滤

```python
from itertools import takewhile

values = [2, 4, 7, 3, 8]

prefix = list(takewhile(lambda value: value < 5, values))
all_matches = [value for value in values if value < 5]

print(prefix)
print(all_matches)
```

前缀是二、四。

全体过滤还会保留后面的三。

差别来自需求定义，不是哪个函数更“正确”。

### 27.5 练习五：把边界记录留给正文

使用前面的 `split_prefix()`，把注释行当作头部，把第一个非注释行及后续内容作为正文。

```python
lines = ["# title", "# version", "body 1", "body 2"]
header, body = split_prefix(lines, lambda line: line.startswith("#"))

assert header == ["# title", "# version"]
assert list(body) == ["body 1", "body 2"]
```

再用空输入、全部注释、第一行就是正文三种情况验证。

前缀拆分函数应该对这些边界都有明确结果。

### 27.6 练习六：用选择标记筛数据，并发现长度政策

```python
from itertools import compress

names = ["A", "B", "C", "D"]
flags = [1, 0, 1]

print(list(compress(names, flags)))
```

得到 A、C，D 没有对应标记而不进入结果。

如果你的任务要求每个名字必须有标记，可以先在有限序列中检查长度一致，或用严格配对实现验证。

不要把自然结束当成缺失标记已经被处理。

### 27.7 练习七：把成对参数传给计算函数

```python
from itertools import starmap


def rectangle_area(width, height):
    if width <= 0 or height <= 0:
        raise ValueError("边长必须为正")
    return width * height


sizes = [(2, 3), (4, 5), (1, 8)]
print(list(starmap(rectangle_area, sizes)))
```

结果为六、二十、八。

再加入一条只有一个元素的元组，观察参数错误发生在消费到那条记录时。

### 27.8 练习八：累计值和相邻差值互相检查

```python
from itertools import accumulate, pairwise

changes = [3, -1, 5]
values = list(accumulate(changes, initial=10))
recovered = [after - before for before, after in pairwise(values)]

assert values == [10, 13, 12, 17]
assert recovered == changes
```

这条验证关系独立于你用怎样的打印格式展示结果。

它帮助检查初始值和步数对应是否一致。

### 27.9 练习九：避免累计可变状态共享

下面使用不可变元组产生历史快照：

```python
from itertools import accumulate

states = list(accumulate(
    ["A", "B", "C"],
    lambda previous, item: previous + (item,),
    initial=(),
))

print(states)
```

每一步得到一个新元组。

如果改成反复修改并返回同一列表，保存的多个历史项可能最终指向同一个内容。

不过，不断拼接越来越长的元组也会增加复制成本，所以这只适合规模合理的历史快照需求。

### 27.10 练习十：连续分段与全局分类

```python
from itertools import groupby

values = ["A", "A", "B", "A", "B", "B"]

runs = [(key, list(group)) for key, group in groupby(values)]
classes = [(key, list(group)) for key, group in groupby(sorted(values))]

print(runs)
print(classes)
```

前者保留四个连续段。

后者得到全局 A 组和 B 组。

排序解决相邻条件，也改变原顺序，这正是两种报告的区别。

### 27.11 练习十一：证明旧组共享来源

```python
from itertools import groupby

outer = groupby("AAABB")
key_a, group_a = next(outer)
key_b, group_b = next(outer)

assert key_a == "A"
assert list(group_a) == []
assert list(group_b) == ["B", "B"]
```

如果你确实需要 A 组，应该在推进外层之前就处理或保存。

把所有组迭代器先保存起来，通常无法替代保存组内容。

### 27.12 练习十二：两个分支不会重新运行来源

```python
from itertools import tee

seen = []


def source():
    for number in range(3):
        seen.append(number)
        yield number


first, second = tee(source())
assert list(first) == [0, 1, 2]
assert list(second) == [0, 1, 2]
assert seen == [0, 1, 2]
```

这项练习为了观察缓存行为，故意让第一分支先读完。

来源很大时，这种进度差会使缓存明显增长，不应当作默认的大规模处理方式。

### 27.13 练习十三：区分真实 `None` 与长度不足

```python
from itertools import zip_longest

missing = object()
pairs = list(zip_longest([None, 2], [10], fillvalue=missing))

assert pairs[0] == (None, 10)
assert pairs[1][0] == 2
assert pairs[1][1] is missing
```

判断哨兵使用 `is`，表示对象身份。

不要把它转换成显示字符串再比较，那会让内部标记与普通业务文本混淆。

### 27.14 练习十四：按值重复与按位置组合

```python
from itertools import combinations, permutations, product

print(list(combinations("ABC", 2)))
print(list(permutations("ABC", 2)))
print(list(product("AB", repeat=2)))
```

请分别解释为什么第一项没有 B、A，第二项有，第三项允许 A、A。

再把输入改成 `"AAB"`，观察相同值来自不同位置时为什么还可能产生相同内容的结果。

### 27.15 练习十五：分批的尾批政策

使用前面兼容旧版本的 `batch_records()`：

```python
assert list(batch_records(range(7), 3)) == [
    (0, 1, 2),
    (3, 4, 5),
    (6,),
]

try:
    list(batch_records(range(7), 3, strict=True))
except ValueError:
    print("最后一批不完整，按约定报告")
```

如果不把结果转列表，而是逐批发送，前两批可能已经产生外部动作，最后才发现尾批问题。

这项练习应同时检查数据结果和副作用边界。

### 27.16 练习十六：短路检查消耗了多少？

```python
source = iter([0, 0, 5, 6])
assert any(source) is True
assert next(source) == 6
```

再设计一条包含假值的输入，观察 `all()` 在哪里停止。

最后试一次成员搜索 `5 in source`，记录剩余内容。

这些操作都在真正读取来源，不是无副作用的预览窗口。

### 27.17 练习十七：预览没有完成全文件审计

给前面的双文件预览程序增加第四条才出现的坏记录。

先预览前三条匹配，再把限制放大，比较 `encountered_problems`。

如果坏记录尚未被读取，它自然不会出现在已遇到错误列表中。

你应该在报告说明中区分“已读取部分的问题”和“完整输入的全部问题”。

### 27.18 练习十八：给参数矩阵设置预算

先提供两组三个值的参数，再设置 `max_configs=8`。

预计总量为九，程序应在执行试验前拒绝。

把上限改为十，再确认生成九条配置。

这样验证的是“先估算后执行”的资源政策，不只是 `product()` 能否输出元组。

## 第二十八章：如何把这些工具放回日常 Python 代码？

### 28.1 不必把每个循环都改成函数链

三四行直观循环能够清楚表达需求时，保留它完全合理。

如果一个管道连续嵌套十层，读者需要反复跳转才知道字段如何变化，代码就可能失去可读性。

可以给关键阶段取名字，例如解析、验证、筛选和输出。

工具让常见循环模式更明确，不要求把所有逻辑压成一行。

### 28.2 每个阶段最好说明输入和输出条数关系

映射通常一项进、一项出。

过滤可能减少条数。

分组把多条记录变成若干组。

展开会把一项批次变成多条记录。

组合可能让输出数量成倍增长。

理解这些关系，可以提前发现配对错位、计数误解和组合爆炸。

### 28.3 每个阶段还要说明排序关系

`chain` 保留来源先后连接。

`filter` 类工具保留被选元素的相对次序。

`sorted` 主动重排。

`groupby` 依赖当前相邻关系。

`product` 按输入池位置规则枚举。

如果你无法说明阶段对顺序做了什么，后面出现“相同数据结果却变了”就很难排查。

### 28.4 数据来源由谁关闭？

纯内存列表没有磁盘句柄生命周期问题。

文件、网络响应和数据库游标可能需要明确关闭。

当管道提前停止时，资源清理尤其需要安排。

把管道放在资源拥有者的上下文里消费，是容易理解的起点；封装成生成器时，要明确提前关闭责任。

### 28.5 应该怎样安排错误处理？

参数本身非法时，有些错误会在构造工具时出现。

输入转换、业务函数和严格尾批检查等错误，可能在迭代到对应位置时才出现。

因此，保护范围通常需要包含真正消费过程。

涉及外部写入时，还要说明前面已经处理的结果是否保留、如何恢复，以及重试是否会重复执行。

## 第二十九章：常用方法、版本与复习对照

| 任务 | 工具 | 结果与重要边界 |
| --- | --- | --- |
| 取得前若干项 | `islice(source, n)` | 推进原来源，不支持任意回看 |
| 按位置跳取 | `islice(source, start, stop, step)` | 跳过的元素也会被消费 |
| 依次连接来源 | `chain(first, second, ...)` | 不自动排序，不交错 |
| 外层批次也按需取得 | `chain.from_iterable(batches)` | 展开一层，不递归展开全部 |
| 持续编号 | `count(start, step)` | 需要明确停止条件 |
| 重复同一对象 | `repeat(value, times)` | 不会每次创建独立可变对象 |
| 循环一批输入 | `cycle(source)` | 保存第一轮元素，输入应合理有限 |
| 保留条件为假项 | `filterfalse(predicate, source)` | 检查全体实际消费项 |
| 截取连续前缀 | `takewhile(predicate, source)` | 消费导致停止的边界元素 |
| 丢弃连续前缀 | `dropwhile(predicate, source)` | 之后不再按同条件过滤 |
| 用选择流筛选 | `compress(data, selectors)` | 按真值选，短边结束 |
| 每条参数记录展开调用 | `starmap(function, records)` | 位置解包，不自动关键字调用 |
| 逐步累计 | `accumulate(source, function, initial=...)` | 保存当前累计状态，不深拷贝状态 |
| 相邻配对 | `pairwise(source)` | 3.10+，长度不足两项则空 |
| 连续相同键分组 | `groupby(source, key=...)` | 组共享来源，全局同键需另行组织 |
| 分出多个读取分支 | `tee(source, n)` | 进度差产生缓存，不支持分支并发安全 |
| 不等长配对并补位 | `zip_longest(..., fillvalue=...)` | 直到最长来源结束，注意无限输入 |
| 参数所有搭配 | `product(*pools, repeat=...)` | 预先保存输入池，输出数量可能巨大 |
| 按位置选组合 | `combinations(source, r)` | 相同值不同位置不自动去重 |
| 有序位置选择 | `permutations(source, r)` | 顺序不同算不同 |
| 允许重复选的组合 | `combinations_with_replacement(source, r)` | 不同于所有有序搭配 |
| 固定大小批次 | `batched(source, n)` | 3.12+，尾批默认可短 |
| 严格尾批 | `batched(..., strict=True)` | 3.13+，错误可能在后期消费时出现 |

`accumulate(initial=...)` 从 Python 3.8 起支持，本文基础环境可以使用。

`pairwise()` 从 Python 3.10 起支持。

`batched()` 从 Python 3.12 起支持，`strict` 参数在 3.13 增加。

旧版本练习可以使用前面明确给出的辅助函数，不要把它伪装成旧标准库原本提供的接口。

如果你已经读过迭代器和生成器笔记，可以把本篇看成“怎样组合已有迭代能力”的继续。

一个工具是否适合任务，可以从四个问题开始：结果什么时候产生，原来源推进了多少，内部需要保存哪些状态，以及谁负责终止和关闭。

官方参考：[itertools 模块](https://docs.python.org/3.14/library/itertools.html)、[迭代器类型](https://docs.python.org/3.14/library/stdtypes.html#iterator-types)、[内置 iter() 与 next()](https://docs.python.org/3.14/library/functions.html)、[groupby](https://docs.python.org/3.14/library/itertools.html#itertools.groupby)、[tee](https://docs.python.org/3.14/library/itertools.html#itertools.tee)、[batched](https://docs.python.org/3.14/library/itertools.html#itertools.batched)。
