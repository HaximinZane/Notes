# Python `time` 库详细教程：时间戳、耗时测量、等待与超时期限

## 零基础先导：同样叫“时间”，为什么要分几种时钟？

`time` 是 Python 的标准库，能读取系统时间、测量程序经过多久、让当前线程暂停。它与 `datetime` 的区别在于：`datetime` 更适合“2026 年 11 月 3 日下午三点”这种日历问题；`time` 更常遇到时间戳、时钟读数、等待与耗时测量。

### 先理解两个场景

**场景 A：你想记录现在是何时。**

```python
import time

now = time.time()    # 以 Unix 纪元为基准的秒数（浮点数）
print(now)
```

这类读数受系统时间校正影响，因此适合表示大致“何时发生”，却不一定适合精准量程序运行时长。

**场景 B：你想知道一个操作用了多少秒。**

```python
import time

start = time.perf_counter()    # ① 读取适合度量经过时长的时钟
time.sleep(0.2)               # ② 请求当前线程暂停约 0.2 秒
elapsed = time.perf_counter() - start  # ③ 两次读数相减
print(elapsed)
```

`perf_counter()` 的绝对值**没有直接的日历含义**，要用前后之差。`time.sleep(0.2)` 通常不会保证精准到毫秒，操作系统调度可能让实际等待更久；它并不是“把 CPU 消耗 0.2 秒来忙等”。

如果你在事件循环/协程中执行异步任务，不要随手调用阻塞的 `time.sleep()`，要按环境使用例如 `await asyncio.sleep(...)`。看超时逻辑时区分“某次调用等待多久”和“整个任务距离截止期限还有多久”。最后记住：**`time.time()` 可能因时钟调整跳变；`time.monotonic()` 适合衡量不能倒退的经过时间**。

---

你想知道一段程序运行多久，可能会先写：

```python
import time

start = time.time()
# 执行一些工作
end = time.time()
print(end - start)
```

很多时候它看起来可以工作。

但系统时间如果被校准，结果可能不再适合衡量经过时长。

`time` 提供不止一种时钟，正是因为“现在是几时”和“已经经过多久”是两个不同问题。

本文主要面向 Python 3.11 及以上，先解释选择，再连接实际计时、轮询、重试与测试。

## 第一章：先分清墙上时钟和经过时长

### 1.1 墙上时钟回答什么？

比如：某份文件在何时修改，一条事件何时发生。

我们需要能与年月日对应的时间值。

`time.time()` 常用于取得当前 Unix 时间戳，单位秒。

```python
import time

print(time.time())
```

结果是一个随运行时刻变化的浮点数。

不要把教程中某个具体数字当成每次都应该相同的输出。

### 1.2 为什么它不适合所有耗时测量？

系统时间可能因校时、管理员修改或其他机制调整。

如果你用两个墙上时钟读数相减，就可能把时钟调整也算进去。

测量请求耗时、算法耗时或等待期限，更希望时钟连续向前，不被设置日期直接拨回。

### 1.3 单调时钟解决什么问题？

单调时钟的读数不会因为设置系统墙上时间而倒退。

`time.monotonic()` 的起点不用于表示某个日历日期。

你通常关心两个读数的差，或者以当前读数加上一段间隔形成截止点。

### 1.4 `perf_counter()` 用来干什么？

它提供适合测量短时经过时长的性能计数器。

```python
import time

start = time.perf_counter()
value = sum(range(100_000))
elapsed = time.perf_counter() - start

print(value)
print(f"经过 {elapsed:.6f} 秒")
```

耗时包含程序实际等待期间经过的时间，不只是 CPU 真正在执行指令的时间。

数值每次会变化，受机器负载与缓存等因素影响。

### 1.5 名称分别怎么记？

`time` 取得可对应日历的墙上时间值。

`monotonic` 强调单调性，适合经过时长和期限。

`perf_counter` 来自 performance counter，常用于性能计时。

具体底层时钟来源取决于平台和 Python 实现，通常不需要自己绑定系统调用。[官方时钟接口](https://docs.python.org/3.14/library/time.html)

## 第二章：一次完整的耗时测量

### 2.1 计时边界放在哪里？

假设你要测量“读取、解析并汇总一个文件”的完整耗时。

计时开始应在读取之前，结束应在汇总完成之后。

如果只包住 `open()` 一行，测到的并不是完整处理过程。

```python
import json
import time
from pathlib import Path

path = Path("time_practice/input.json")
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text('[1, 2, 3, 4]', encoding="utf-8")

start = time.perf_counter()
with path.open(encoding="utf-8") as source:
    numbers = json.load(source)
total = sum(numbers)
elapsed = time.perf_counter() - start

print(total)
print(f"完整处理耗时：{elapsed:.6f} 秒")
```

示例创建已知练习文件，不应该直接覆盖自己的真实资料。

### 2.2 六位小数就代表微秒级准确吗？

不代表。

格式显示六位小数，只说明你选择了这个字符串格式。

时钟分辨率、测量开销、调度和系统状态都影响实际意义。

不要因为显示很多位，就声称测量具备同样精确的物理准确度。

### 2.3 为什么短程序测一次不够？

一次运行可能恰好遇到进程调度、缓存预热、后台任务或首次加载开销。

比较算法时应保证做同样的工作，多次观察，并考虑输入规模。

特别短的代码片段可以进一步使用 `timeit`，它会帮助组织重复测量；但是否读取真实数据、是否包含准备成本，仍需要你自己定义。

### 2.4 `_ns` 版本有什么作用？

```python
import time

start = time.perf_counter_ns()
value = sum(range(100_000))
elapsed_ns = time.perf_counter_ns() - start

print(elapsed_ns)
print(elapsed_ns / 1_000_000_000)
```

纳秒版本返回整数，避免用浮点表示长读数时的一些精度损失。

它不保证底层硬件真的每一纳秒都能给出独立有效测量。

单位与分辨率是两个概念。

## 第三章：`sleep()`，等待并不是精确调度

### 3.1 基本写法

```python
import time

print("开始等待")
time.sleep(0.1)
print("继续执行")
```

参数单位是秒，允许浮点小数。

它暂停当前线程的执行，不是让整个操作系统停止工作。

### 3.2 一定刚好睡这么久吗？

正常情况下会至少等到相应时机，但调度等原因可能让实际暂停更长。

信号处理若抛出异常等情况，也可能提前打断正常流程。

不要用 `sleep(0.001)` 就断言代码每毫秒严格运行一次。

### 3.3 负数和零

负数不是“回到过去”，通常会报错。

`sleep(0)` 的调度效果具有平台相关性，也不是一个保证清空所有异步任务的通用操作。

普通业务不应靠它解决线程同步问题。

### 3.4 异步函数里可以直接 sleep 吗？

`time.sleep()` 会阻塞当前线程。

如果这个线程正运行异步事件循环，其他依赖同一循环的任务也会受阻。

异步等待通常使用：

```python
import asyncio

async def pause_briefly():
    await asyncio.sleep(0.1)
```

`await` 让出事件循环调度机会，与阻塞线程的等待不同。

具体协程原理可以接着看目录中已有的异步教材。

## 第四章：CPU 时间与真实经过时间

### 4.1 `process_time()` 回答什么？

它测量当前进程消耗的 CPU 时间，通常不包含单纯睡眠等待。

```python
import time

wall_start = time.perf_counter()
cpu_start = time.process_time()

time.sleep(0.05)

wall_elapsed = time.perf_counter() - wall_start
cpu_elapsed = time.process_time() - cpu_start

print("实际经过：", wall_elapsed)
print("当前进程 CPU：", cpu_elapsed)
```

通常前者接近或超过等待时长，后者明显更小，但具体数值不固定。

### 4.2 它会把子进程 CPU 自动加进来吗？

不要这样假设。

当前进程的 CPU 计时不是整个外部作业树的统一资源账本。

如果你运行编译器、数据库或多个工作进程，要用适合平台和任务的资源统计机制。

### 4.3 `thread_time()` 呢？

它关注当前线程的 CPU 时间，在支持的平台上可用于更细粒度观察。

普通脚本计总耗时，先掌握 `perf_counter()` 即可。

多线程里测到某个线程 CPU 很少，也不代表整个程序没有消耗 CPU，其他线程或进程可能正在执行。

### 4.4 怎样查看当前时钟信息？

```python
import time

info = time.get_clock_info("perf_counter")
print(info.monotonic)
print(info.adjustable)
print(info.resolution)
print(info.implementation)
```

这些属性帮助你理解底层实现和分辨率。

`adjustable` 等字段有具体接口定义，不应该把它扩大解释为完全不受任何硬件或频率校准机制影响。

大多数业务只需要选择语义合适的公开时钟。

## 第五章：时间戳怎样变成日历组成？

### 5.1 UTC 与本地转换

```python
import time

timestamp = 0
print(time.gmtime(timestamp))
print(time.localtime(timestamp))
```

`gmtime()` 使用 UTC 视角。

`localtime()` 使用运行环境本地时区。

同一个数值可能显示成不同年月日时分，真实时刻仍然相同。

### 5.2 `struct_time` 是什么？

它提供年、月、日、小时、分钟、秒、星期等组成字段。

可以用属性访问：

```python
parts = time.gmtime(0)
print(parts.tm_year)
print(parts.tm_mon)
print(parts.tm_mday)
```

通常得到 1970、一、一。

它不是 `datetime` 对象，不具备完全相同的时区与日期运算接口。

### 5.3 `strftime()` 与 `strptime()`

```python
import time

parts = time.strptime("2026-11-02 14:30", "%Y-%m-%d %H:%M")
text = time.strftime("%Y/%m/%d %H:%M", parts)
print(text)
```

输出：

```text
2026/11/02 14:30
```

这里 `strptime()` 返回 `struct_time`，而 `datetime.strptime()` 返回 datetime。

接口名字相近，不表示返回类型相同。

### 5.4 `mktime()` 按什么时区解释？

它通常把输入结构当作本地时间转换成时间戳。

因此，不能把 UTC 的 `gmtime()` 结果随手交给它，就认为在任何本地时区都原样往返。

需要明确 UTC 语义时，使用带 `timezone.utc` 的 datetime 转换通常更清楚。

### 5.5 可以在程序里修改 `TZ` 再调用 `tzset()` 吗？

某些 Unix 平台提供这种能力，但它影响进程全局本地时区状态，且不是所有平台支持。

普通应用需要处理多个用户时区时，应给各个 datetime 明确时区，不宜通过反复改变全局环境实现。

本教程不要求修改用户系统或进程全局时区来完成练习。

## 第六章：完整实践——有总期限的轮询

### 6.1 需求

每隔一小段时间检查一次条件，直到成功或到达总期限。

不能因为系统墙上时间被拨回，就无意多等很久。

### 6.2 实现

```python
import time


def wait_until(predicate, timeout, interval=0.1, *, clock=time.monotonic, sleep=time.sleep):
    if timeout < 0:
        raise ValueError("timeout 不能为负数")
    if interval <= 0:
        raise ValueError("interval 必须大于零")

    deadline = clock() + timeout

    while True:
        if predicate():
            return True

        remaining = deadline - clock()
        if remaining <= 0:
            return False

        sleep(min(interval, remaining))
```

### 6.3 为什么每次计算剩余时间？

条件检查本身也会消耗时间。

如果每轮只睡固定间隔并累计睡眠次数，容易忽略检查成本。

截止点方式根据当前单调读数计算剩余时间，更接近总预算含义。

### 6.4 它能强行打断一个卡住的 predicate 吗？

不能。

如果 `predicate()` 自己阻塞很久，外层要等它返回后才能检查期限。

所以内部 I/O 也应该有合适超时或取消机制。

这份函数表达的是合作式检查总期限，不是对任意阻塞函数的强制抢占。

### 6.5 timeout 为零为什么仍检查一次？

代码先调用条件，再判断是否还可以等待。

这定义了“立即检查一次，不额外等待”的语义。

如果你的需求是预算已到就完全不调用条件，应调整检查顺序，并把规则写清楚。

## 第七章：不真的等待，也能验证时间逻辑吗？

### 7.1 注入可控时钟

```python
class FakeClock:
    def __init__(self):
        self.value = 0.0

    def now(self):
        return self.value

    def sleep(self, seconds):
        self.value += seconds


fake = FakeClock()

result = wait_until(
    lambda: fake.now() >= 0.3,
    timeout=1.0,
    interval=0.1,
    clock=fake.now,
    sleep=fake.sleep,
)

assert result is True
assert fake.now() <= 1.0
```

这里使用前面的 `wait_until()`。

假的 sleep 不暂停线程，只推进受控读数。

你可以很快测试成功、超时和边界，而不让测试真的睡很多秒。

### 7.2 这能代替真实网络超时测试吗？

不能。

它验证你自己的期限计算和轮询政策。

真实套接字是否遵守读取超时、子进程是否可取消，需要对应层次的实际测试。

测试替身让某一层更可控，不自动证明所有底层行为。

## 第八章：固定间隔循环，为什么会越跑越慢？

### 8.1 每次做完再 sleep，会累计任务耗时

```python
for _ in range(3):
    do_work()
    time.sleep(1)
```

如果 `do_work()` 用了零点二秒，两次开始之间就大约是一点二秒，而不是一秒。

这是一种“完成后再等待”的规则，可以合理使用，但要知道它的含义。

### 8.2 按计划开始点推进

```python
import time

interval = 0.1
next_start = time.monotonic()

for index in range(3):
    remaining = next_start - time.monotonic()
    if remaining > 0:
        time.sleep(remaining)

    print("执行轮次：", index)
    next_start += interval
```

这让开始计划相对于一个共同基准递增。

实际执行仍然受调度影响，不是硬实时保证。

### 8.3 如果任务比间隔还慢呢？

你需要决定：立即补跑、跳过过期轮次、降低频率，还是只保留最新任务。

不同政策会产生不同结果。

不要在没有上限的循环中不断积累补跑任务，让一次延迟变成永久拥塞。

这里的三轮示例只解释基准，不建立后台调度服务。

## 第九章：练习与参考思路

### 9.1 练习一：选对时钟

记录事件发生时刻，用墙上时钟或 aware datetime。

测量函数执行耗时，考虑 `perf_counter()`。

实现等待截止点，考虑 `monotonic()`。

统计当前进程 CPU 消耗，考虑 `process_time()`。

题目不是背函数名，而是解释每个值的语义是否匹配需求。

### 9.2 练习二：写一个返回耗时的包装函数

```python
import time


def measure_call(func, *args, **kwargs):
    start = time.perf_counter()
    result = func(*args, **kwargs)
    elapsed = time.perf_counter() - start
    return result, elapsed


value, elapsed = measure_call(sum, [1, 2, 3])
assert value == 6
assert elapsed >= 0
```

这个版本只在函数成功时返回二元组。

如果函数抛异常，异常会继续传播，调用者拿不到正常返回的耗时。

需要失败也记录时，可以在 `finally` 中交给日志或回调，而不是随意吞掉异常。

### 9.3 练习三：验证轮询超时

```python
fake = FakeClock()

result = wait_until(
    lambda: False,
    timeout=0.25,
    interval=0.1,
    clock=fake.now,
    sleep=fake.sleep,
)

assert result is False
assert abs(fake.now() - 0.25) < 1e-9
```

最后一次等待会使用剩余的零点零五秒，不再完整多睡一个间隔。

### 9.4 练习四：解释 CPU 与等待

测量一个短 `sleep()` 的 `perf_counter` 差与 `process_time` 差。

不要断言实际等待恰好等于传入值，也不要把 CPU 差必须为零写成测试。

合理测试应围绕语义和宽松边界，避免因系统调度产生脆弱结果。

## 第十章：常用接口对照

| 问题 | 接口 | 注意事项 |
| --- | --- | --- |
| 现在对应哪个 Unix 时间值 | `time.time()` | 墙上时钟可调整 |
| 整数纳秒时间戳 | `time.time_ns()` | 单位不等于实际精度 |
| 已经过多久或等待期限 | `time.monotonic()` | 不用于日历转换 |
| 测量短时经过 | `time.perf_counter()` | 包含等待经过时间 |
| 纳秒耗时读数 | `perf_counter_ns()` | 适合整数差值 |
| 当前进程 CPU | `process_time()` | 不是整个子进程树总消耗 |
| 当前线程 CPU | `thread_time()` | 需要平台支持 |
| 暂停当前线程 | `sleep(seconds)` | 不是精确调度 |
| UTC 日历组成 | `gmtime(timestamp)` | 返回 struct_time |
| 本地日历组成 | `localtime(timestamp)` | 使用运行环境本地时区 |
| 本地组成转时间戳 | `mktime(parts)` | 不要误当 UTC 逆变换 |
| 查看时钟性质 | `get_clock_info(name)` | 读定义而非猜底层 |

配套阅读：[Python-datetime库.md](Python-datetime库.md)、[Python-zoneinfo库.md](Python-zoneinfo库.md)、[Python-异步、协程、事件循环与线程.md](Python-异步、协程、事件循环与线程.md)。

官方参考：[time](https://docs.python.org/3.14/library/time.html)、[PEP 418 时钟设计](https://peps.python.org/pep-0418/)。
