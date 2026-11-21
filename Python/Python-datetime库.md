# Python `datetime` 库详细教程：日期、时刻、时间差、格式化和实际数据处理

## 阅读前先建立完整的时间模型：不要直接背格式化符号

### 0.1 人说的“时间”其实有四种不同问题

你会说“今天是 11 月 3 日”“开会是下午 3 点”“从现在起再等两个小时”“美国和中国显示的钟表时间不同”。它们看似都是“时间”，本质上却分别是：

| 真实问题 | Python 中常用的对象 |
|---|---|
| 某个日历日期是什么？ | `datetime.date` |
| 一天中的钟表时刻是什么？ | `datetime.time` |
| 哪一天的哪一刻？ | `datetime.datetime` |
| 两个时间之间相隔多久？ | `datetime.timedelta` |

`datetime` 这个名字来自 date（日期）与 time（时间）。注意一个极容易混淆的地方：**`datetime` 既是模块名称，也是模块里面一个类的名称**。

```python
import datetime                 # datetime 指模块
today = datetime.date(2026, 11, 3)
moment = datetime.datetime(2026, 11, 3, 15, 30)
print(today)                    # 2026-11-03
print(moment)                   # 2026-11-03 15:30:00
```

你也可能看到 `from datetime import datetime`，此时同名的 `datetime` 指类，不再是之前的模块名。这两种写法都合法，读代码时先看 import。

### 0.2 为什么不用字符串就够了？

`"2026-11-03"` 只是字符；计算十天以后，需要理解每月天数、闰年、跨年等。`date` 对象可以与 `timedelta` 一起计算：

```python
from datetime import date, timedelta

start = date(2026, 11, 3)            # 先构造一个日历日期
later = start + timedelta(days=10)   # “天数”是一个持续时长
print(later)                         # 2026-11-13
```

`timedelta(days=10)` 不是“11 月 13 日”；它只表示一段时长。与具体日期相加才会得到另一个日期。

### 0.3 最重要的误区：本地时间、UTC 和时区不是一回事

`datetime(2026, 11, 3, 9, 0)` 只保存年月日时分秒，**没有说明它是上海上午九点还是纽约上午九点**。这叫“naive”（不带时区信息）的 datetime。

```python
from datetime import datetime, timezone

naive = datetime(2026, 11, 3, 9, 0)      # 无时区信息
aware = datetime(2026, 11, 3, 9, 0, tzinfo=timezone.utc)  # 明确是 UTC 09:00
print(naive.tzinfo)  # None
print(aware.tzinfo)  # UTC
```

两个数值完全相同的 `09:00` 如果解释在不同地区，**它们可能指不同的真实瞬间**。跨地区时间计算时，应尽量保存明确的时间点与时区信息。不要随意把 naive datetime 直接视作 UTC。

### 0.4 要求当前 UTC 时间，推荐明确告诉 Python 时区

```python
from datetime import datetime, timezone

now_utc = datetime.now(timezone.utc)  # 当前 UTC 时刻，带时区信息
print(now_utc.isoformat())            # 例如 2026-11-03T...+00:00
```

`datetime.now()` 不带参数时，常得到本地钟表表示，但结果默认没有 `tzinfo`；`datetime.utcnow()` 返回 naive UTC，现代代码通常更适合上面的显式时区写法。**示例输出会随你实际执行的时刻变化。**

### 0.5 地区时区为什么不直接写成 `+08:00`？

固定偏移表示某一段时差，不能独立表达“纽约随着夏令时改变偏移”这样的历史与日期规则。你要的是地区规则，就用 `zoneinfo.ZoneInfo`：

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

moment_utc = datetime(2026, 11, 3, 12, 0, tzinfo=timezone.utc)
new_york = moment_utc.astimezone(ZoneInfo("America/New_York"))
print(new_york.isoformat())  # 在该日期显示对应的纽约当地时间
```

`astimezone` 在这里是**同一个真实瞬间的显示转换**，不是把数字简单减去几个小时得到毫无时区信息的新对象。有些系统（尤其是缺少 IANA 时区数据的环境）需安装 `tzdata`；这在 `zoneinfo` 教程中详细说明。

### 0.6 格式化与解析是相反方向

```python
from datetime import datetime

dt = datetime(2026, 11, 3, 15, 30)
text = dt.strftime("%Y-%m-%d %H:%M")   # datetime 对象 → 文本
again = datetime.strptime(text, "%Y-%m-%d %H:%M")  # 文本 → datetime 对象
print(text)                            # 2026-11-03 15:30
print(again)                           # 2026-11-03 15:30:00
```

记名字：`str` 是 string（字符串）；`f` 记 format（格式化），`p` 记 parse（解析）。`strftime` 不会自动添加时区；`strptime` 也不会凭空知道原文本的时区。

### 0.7 学习路线与易混点

先学 `date` → `timedelta` → `datetime` → 字符串格式化，再学习 UTC、aware/naive 与 `ZoneInfo`。`datetime` 负责“日历与日期时刻运算”，`time` 库里的 `time.monotonic()`、`perf_counter()` 则更适合测量经过的时长。**不要拿日历时间差来做严谨性能计时**。

---

你可能会写 `print("2026-11-02")`，也能把一个日期保存成字符串。

但如果要计算七天以后、比较两个截止时间、把接口里的日期读进来，或者区分北京时间和纽约时间，字符串就不够表达全部规则了。

`datetime` 提供的是有日期时间含义的对象。

这一篇按实际问题展开，不要求你第一次阅读就记住所有格式码。

代码主要面向 Python 3.11 及以上；示例日期是固定教学输入，并不表示程序运行时一定得到同一天。

## 第一章：先分清“日期”“时刻”和“经过多久”

### 1.1 它们看起来都和时间有关，为什么还要分类型？

假设你记录：

```text
生日：2006-05-20
上课时间：08:30
作业截止：2026-11-02 23:59
本次学习时长：45 分钟
```

生日通常只需要年月日。

每天的上课时间可能只需要时分。

截止时间需要日期与时间结合，还可能需要时区。

学习时长则是一段间隔，不是某一天的时刻。

这些信息的操作规则不同，因此标准库提供不同类型。

### 1.2 常用类型是什么？

`date` 表示日期。

`time` 表示一天中的时间组成。

`datetime` 组合日期和时间。

`timedelta` 表示时间差。

`timezone` 表示固定 UTC 偏移时区。

`tzinfo` 是时区信息的抽象基础，普通应用通常使用具体时区实现。

### 1.3 模块和类都叫 datetime，会不会混淆？

第一种导入：

```python
import datetime

moment = datetime.datetime(2026, 11, 2, 12, 30)
```

前一个 `datetime` 是模块，后一个是类。

第二种导入：

```python
from datetime import datetime

moment = datetime(2026, 11, 2, 12, 30)
```

这次把类名直接导入当前命名空间。

也可以给模块别名：

```python
import datetime as dt

moment = dt.datetime(2026, 11, 2, 12, 30)
```

这三种都常见。

选择一种在当前文件中保持一致，避免中途把 `datetime` 从模块名覆盖成实例变量。

## 第二章：日期 `date`，先从没有时区的日历问题开始

### 2.1 创建一个明确日期

```python
from datetime import date

learning_day = date(2026, 11, 2)

print(learning_day)
print(learning_day.year)
print(learning_day.month)
print(learning_day.day)
```

输出：

```text
2026-11-02
2026
11
2
```

构造参数按年、月、日排列。

月份从一到十二，不是从零开始。

### 2.2 不能随意构造不存在的日期

```python
from datetime import date

date(2026, 2, 29)
```

2026 不是闰年，因此抛 `ValueError`。

相比普通字符串 `"2026-02-29"`，日期对象在构造时就能识别基本日历合法性。

它不会因为你“想表示月底”就自动调整成二月二十八日。

### 2.3 获取今天

```python
from datetime import date

print(date.today())
```

得到运行环境本地日期。

这里的“本地”由实际进程与操作系统环境决定，不自动知道业务用户位于哪个时区。

服务器在 UTC，而用户在另一个地区时，两者的“今天”可能不同。

需要特定地区日期，应先取得该地区的 aware datetime，再取 `.date()`，后面会讲。

### 2.4 星期几如何表示？

```python
from datetime import date

day = date(2026, 11, 2)
print(day.weekday())
print(day.isoweekday())
```

这一天是星期一，输出：

```text
0
1
```

`weekday()` 以周一为零，周日为六。

`isoweekday()` 以周一为一，周日为七。

两种编号都合理，但你需要知道接下来的数组或业务规则用哪一种。

### 2.5 七天以后，为什么不用给 day 直接加七？

```python
from datetime import date, timedelta

start = date(2026, 11, 28)
end = start + timedelta(days=7)
print(end)
```

得到：

```text
2026-12-05
```

日期运算会跨月、跨年并处理闰年规则。

如果只写 `start.day + 7`，得到的是整数三十五，不是一份合法的新日期。

### 2.6 一个月以后，为什么没有 `timedelta(months=1)`？

因为日历月份长度不同。

一月三十一日的“一个月以后”该是二月最后一天，还是应该拒绝，取决于业务政策。

`timedelta` 用固定天数、秒数和微秒等表达间隔，不替你决定这种日历规则。

月度账期、每月提醒等需求应明确“月底如何处理”，或使用专门日历运算工具。

不能把三十天无条件叫作一个月。

## 第三章：日期加时间，创建 `datetime`

### 3.1 明确构造

```python
from datetime import datetime

moment = datetime(2026, 11, 2, 14, 30, 15)
print(moment)
```

输出：

```text
2026-11-02 14:30:15
```

参数依次是年、月、日、时、分、秒，后面还可以指定微秒和时区信息。

小时通常是零到二十三，不是上午下午两套数字自动判断。

### 3.2 取得当前时间

```python
from datetime import datetime

now = datetime.now()
print(now)
```

这是当前本地时间，通常没有附带明确的时区对象。

它适合某些纯本地展示，但跨系统保存和比较时容易产生歧义。

需要明确 UTC 时刻，常用：

```python
from datetime import datetime, timezone

now_utc = datetime.now(timezone.utc)
print(now_utc.isoformat())
```

Python 3.11 起也可使用 `datetime.UTC` 模块常量别名，注意导入方式与模块、类命名区别。

### 3.3 `utcnow()` 为什么不作为新代码首选？

它返回表示 UTC 内容的 naive datetime，时区信息没有附上。

这可能在后来被当成本地时间解释。

Python 3.12 起已将 `datetime.utcnow()` 标为弃用，建议用 `datetime.now(timezone.utc)` 取得明确 aware 对象。[官方 datetime 说明](https://docs.python.org/3.14/library/datetime.html#datetime.datetime.utcnow)

### 3.4 日期和时间部分怎样取出来？

```python
from datetime import datetime

moment = datetime(2026, 11, 2, 14, 30)
print(moment.date())
print(moment.time())
```

分别得到日期对象和时间对象。

如果希望时间部分保留相应时区资料，可以了解 `timetz()`，不要假设 `.time()` 会保留全部原信息。

### 3.5 两个对象怎样组合？

```python
from datetime import date, time, datetime

day = date(2026, 11, 2)
clock = time(8, 30)
moment = datetime.combine(day, clock)
print(moment)
```

得到：

```text
2026-11-02 08:30:00
```

这适合把日历选中的日期与每天固定上课时间组合。

还没有明确时区时，它仍然只是一个没有完整全球定位语义的本地钟表读数。

## 第四章：`timedelta`，经过多久要怎样计算？

### 4.1 从两个日期或时刻相减

```python
from datetime import datetime

start = datetime(2026, 11, 2, 9, 0)
end = datetime(2026, 11, 2, 10, 45)

duration = end - start
print(duration)
print(duration.total_seconds())
```

输出：

```text
1:45:00
6300.0
```

相减得到 `timedelta`，不是自动变成小时数字。

`total_seconds()` 将整段间隔换算成秒，包含天数和小数部分。

### 4.2 为什么 `.seconds` 不是总秒数？

```python
from datetime import timedelta

duration = timedelta(days=2, seconds=30)
print(duration.days)
print(duration.seconds)
print(duration.total_seconds())
```

得到：

```text
2
30
172830.0
```

`days` 和 `seconds` 是规范化组成部分。

总秒数要把天数也算进去。

### 4.3 负间隔更能看出区别

```python
from datetime import timedelta

duration = timedelta(seconds=-1)
print(duration.days)
print(duration.seconds)
print(duration.total_seconds())
```

输出：

```text
-1
86399
-1.0
```

负一秒被规范化为负一天加八万六千三百九十九秒。

因此，拿 `.seconds` 直接判断“两个时刻相差多少秒”，很容易在跨天或负值时出错。

### 4.4 构造参数可以组合

```python
from datetime import timedelta

study = timedelta(hours=1, minutes=30)
break_time = timedelta(minutes=10)
print(study + break_time)
```

得到一小时四十分钟。

`weeks`、`days`、`hours`、`minutes` 等最终会规范化成内部表示。

这适合表达固定经过时长，不负责日历月份规则。

### 4.5 用 date 加二十三小时，为什么没变成下一天？

`date` 只有日期，没有时分秒。

与 `timedelta` 运算时主要使用天数组成，不能期望它保存小时部分。

需要具体时刻加小时，先使用 `datetime`。

对于日期计划，尽量明确传 `days`，更容易让读者理解意图。

## 第五章：字符串和日期对象，两个方向怎么转换？

### 5.1 输出 ISO 形式

```python
from datetime import date, datetime, timezone

print(date(2026, 11, 2).isoformat())
print(datetime(2026, 11, 2, 12, 30, tzinfo=timezone.utc).isoformat())
```

得到：

```text
2026-11-02
2026-11-02T12:30:00+00:00
```

这是字符串表示，不是改变对象里的时刻。

### 5.2 从已约定 ISO 形式读取

```python
from datetime import date, datetime

day = date.fromisoformat("2026-11-02")
moment = datetime.fromisoformat("2026-11-02T12:30:00+08:00")

print(day.year)
print(moment.utcoffset())
```

`fromisoformat()` 接受标准库支持的 ISO 风格形式，但不是所有 ISO 8601 写法的万能解析器。

它也可能接受比你公开接口契约更宽松的形式，所以“函数能解析”不自动等于“符合业务规定格式”。

Python 3.11 扩展了若干可接受形式，包括常见 `Z` UTC 结尾；需要兼容更旧版本时要核对。

### 5.3 格式不一样，用 `strptime()`

```python
from datetime import datetime

moment = datetime.strptime("2026/11/02 14:30", "%Y/%m/%d %H:%M")
print(moment)
```

格式字符串逐段说明输入如何对应年月日时分。

`strptime` 可以按 string parse time 理解。

输入与格式不匹配，通常抛 `ValueError`，不会自动猜测你本来想用哪一种日期。

### 5.4 想按自己喜欢的样式输出呢？

```python
from datetime import datetime

moment = datetime(2026, 11, 2, 14, 30)
print(moment.strftime("%Y年%m月%d日 %H:%M"))
```

得到：

```text
2026年11月02日 14:30
```

`strftime` 可以按 string format time 理解。

一个负责解析，一个负责格式化，方向不要反。

### 5.5 常用格式码

| 格式码 | 含义 | 示例 |
| --- | --- | --- |
| `%Y` | 四位年份 | 2026 |
| `%m` | 月份 | 11 |
| `%d` | 日期 | 02 |
| `%H` | 二十四小时制小时 | 14 |
| `%M` | 分钟 | 30 |
| `%S` | 秒 | 15 |
| `%f` | 微秒 | 123456 |
| `%z` | UTC 偏移 | +0800 |
| `%a`、`%b` | 星期或月份名称缩写 | 受区域设置影响 |
| `%%` | 百分号字符 | % |

月份 `%m` 与分钟 `%M` 很容易写反。

不要只看输出“像一个日期”就认为格式完全正确。

### 5.6 为什么只写月日的解析可能出问题？

缺失年份时，某些解析规则会使用默认年份。

遇到二月二十九日等闰日，就可能出现你没想到的失败。

需要解析完整业务日期时，尽量要求提供年份。

较新版本也对只含日月、不含年份的部分用法提出弃用提醒，应按目标版本文档核对。

## 第六章：naive 和 aware，为什么“几点钟”还不够？

### 6.1 同一个钟表读数，可以对应不同真实时刻

```text
2026-11-02 09:00
```

这是北京的上午九点，还是纽约的上午九点？

如果只给出年月日和时分，没有上下文，就不能唯一定位到全球时间线上。

没有足够时区资料的对象称为 naive，带有可用偏移信息的对象称为 aware。

严格来说，aware 要求 `tzinfo` 不为 `None`，并且它返回的 `utcoffset()` 也不为 `None`。

### 6.2 固定偏移怎么写？

```python
from datetime import datetime, timedelta, timezone

utc_plus_8 = timezone(timedelta(hours=8))
meeting = datetime(2026, 11, 2, 9, 0, tzinfo=utc_plus_8)

print(meeting.isoformat())
print(meeting.astimezone(timezone.utc).isoformat())
```

输出：

```text
2026-11-02T09:00:00+08:00
2026-11-02T01:00:00+00:00
```

两行描述同一个真实时刻，只是钟表显示方式不同。

### 6.3 `replace(tzinfo=...)` 和 `astimezone()` 有什么区别？

假设你手里有一个 naive 值：

```python
naive = datetime(2026, 11, 2, 9, 0)
```

而数据来源明确说，它表示 UTC+8 的本地上午九点。

可以附上这个已知解释：

```python
aware = naive.replace(tzinfo=utc_plus_8)
```

钟表数字仍然是九点。

现在要把同一时刻显示成 UTC：

```python
utc_value = aware.astimezone(timezone.utc)
```

显示变为凌晨一点。

`replace()` 是替换组成资料，`astimezone()` 是按时区换算同一真实时刻。

如果原值本来已经表示 UTC 九点，你却直接把 `tzinfo` 改成 UTC+8，就把它重新解释成了另一个时刻。

### 6.4 我能不能把所有 naive 值直接标成 UTC？

只有数据契约明确规定它们原本就是 UTC 时，才有依据这样做。

否则这是人为猜测，可能让记录平移数小时。

对于外部接口，最好要求带偏移或明确文档约定；对旧数据迁移，应单独说明采用了什么解释。

### 6.5 为什么 naive 与 aware 不能随便比较大小？

一个值没有明确全球定位，另一个有，直接比较很容易产生无依据的假设。

Python 对这类排序或相减通常抛 `TypeError`。

应先根据真实来源统一语义，再比较。

不要只为消除异常而给所有值随手 `.replace(tzinfo=...)`。

### 6.6 地区时区为什么不能只写固定 `-05:00`？

纽约可能因日期采用不同 UTC 偏移，历史规则也会变化。

固定偏移只能表达“始终相差这么多小时”的时区模型。

要根据地区与日期确定偏移，使用 `zoneinfo.ZoneInfo`。

本篇先建立概念，完整夏令时和歧义处理见独立的 `zoneinfo` 教材。

## 第七章：时间戳与日期对象，两个方向如何转换？

### 7.1 Unix 时间戳通常以秒为单位

时间戳是相对于纪元的数值表示。

普通 POSIX 时间戳从 1970 年一月一日 UTC 开始计秒，通常不把闰秒当作额外累积秒处理。

```python
from datetime import datetime, timezone

moment = datetime(2026, 11, 2, 12, 0, tzinfo=timezone.utc)
timestamp = moment.timestamp()
restored = datetime.fromtimestamp(timestamp, tz=timezone.utc)

print(restored == moment)
```

通常为真。

### 7.2 毫秒时间戳为什么变成离谱日期或报错？

有些接口返回毫秒，数值大约是秒时间戳的一千倍。

如果直接交给按秒解释的 `fromtimestamp()`，就会指向极远时间，甚至超出平台可表示范围。

明确来源后先换算：

```python
seconds = milliseconds / 1000
moment = datetime.fromtimestamp(seconds, tz=timezone.utc)
```

不要只靠位数永远猜单位，协议应该说明是秒、毫秒、微秒还是纳秒。

### 7.3 为什么 naive 的 `.timestamp()` 容易误用？

它通常按系统本地时间解释 naive 对象。

同一组年月日时分，在不同系统本地配置下可能得到不同数值。

需要稳定转换时，先明确时区语义，使用 aware 对象。

### 7.4 文件修改时间怎样显示？

```python
from datetime import datetime, timezone
from pathlib import Path

path = Path("notes.txt")
info = path.stat()
modified = datetime.fromtimestamp(info.st_mtime, tz=timezone.utc)
print(modified.isoformat())
```

这显示文件系统记录的修改时间。

它不等于可信的业务创建时间，也不保证文件内容没有被复制工具保留旧时间。

### 7.5 测量程序耗时也用 datetime 相减吗？

显示事件发生在什么时候，可以用 datetime。

测量一段程序经过多久，通常更适合 `time.perf_counter()` 或单调时钟。

系统校时可能调整墙上时钟，用它计算短时耗时容易受到影响。

时刻记录和持续时间测量，需要不同的工具。

## 第八章：`replace()`、排序和不可变对象

### 8.1 改成当天零点

```python
from datetime import datetime

moment = datetime(2026, 11, 2, 14, 30, 15, 123456)
midnight = moment.replace(hour=0, minute=0, second=0, microsecond=0)

print(moment)
print(midnight)
```

原对象仍然表示下午的时间，新对象表示同一天零点。

日期时间对象是不可变的，`replace()` 返回新对象。

### 8.2 为什么 `replace(day=31)` 可能失败？

它不是“自动滚到下个月”的运算。

如果当前月份没有三十一日，替换后的组成无效，就会报错。

跨日使用 `timedelta`，改具体字段使用 `replace()`，不要混淆。

### 8.3 字符串日期能直接排序吗？

统一、补零、相同偏移表示的 ISO 字符串在一些受控场景可按字典序获得合理顺序。

但格式混用、偏移不同或位数不一致时，字符串顺序不一定等于真实时间顺序。

更稳妥的是解析成具有一致语义的对象，再按需要转换到 UTC 比较。

### 8.4 同一个地区跨夏令时，相减一定是实际经过秒数吗？

两个 aware datetime 若使用同一个 `tzinfo` 对象，标准库有保持本地钟表运算的规则，偏移变化不一定按你期待的实际经过秒数处理。

要计算实际经过时长，可以先分别转换到 UTC，再相减。

要安排“明天本地同一钟点”，则是另一种日历需求。

独立 `zoneinfo` 文档会用具体跨夏令时日期实验，避免把两种问题都叫“一天”而混淆。

## 第九章：完整实践一——生成一周的阅读日期

### 9.1 需求

从明确的开始日期起，生成七天日期和星期名称。

输出中不依赖本机语言区域的英文或中文星期缩写。

### 9.2 实现

```python
from datetime import date, timedelta


def reading_days(start, count):
    if type(count) is not int or count < 0:
        raise ValueError("count 必须是非负整数")

    for offset in range(count):
        yield start + timedelta(days=offset)


weekday_names = ["星期一", "星期二", "星期三", "星期四", "星期五", "星期六", "星期日"]

for day in reading_days(date(2026, 11, 2), 7):
    print(day.isoformat(), weekday_names[day.weekday()])
```

### 9.3 为什么第一天加零？

`range(7)` 产生零到六。

所以包含开始日，一共七天，不包含第七天之后的那个日期。

日期范围是否包含起点终点，要明确说明，避免差一天的常见错误。

### 9.4 为什么使用生成器？

每次产生一个日期，调用者可以逐个处理。

这也接上你已经学过的 `yield`：日期计算与迭代机制可以自然结合。

只有七天时，保存为列表也完全合理；使用生成器不意味着必须为了性能把所有小集合都改写。

## 第十章：完整实践二——解析带偏移的学习记录并计算时长

### 10.1 输入格式

每次学习有开始和结束时间，必须明确偏移。

例如：

```json
{
  "start": "2026-11-02T09:00:00+08:00",
  "end": "2026-11-02T09:45:00+08:00"
}
```

### 10.2 完整代码

```python
from datetime import datetime, timezone


def parse_instant(text):
    if not isinstance(text, str):
        raise ValueError("时间必须是字符串")

    try:
        moment = datetime.fromisoformat(text)
    except ValueError as error:
        raise ValueError("时间不符合支持的 ISO 格式") from error

    if moment.tzinfo is None or moment.utcoffset() is None:
        raise ValueError("时间必须包含明确偏移")

    return moment.astimezone(timezone.utc)


def session_minutes(record):
    start = parse_instant(record["start"])
    end = parse_instant(record["end"])

    if end < start:
        raise ValueError("结束时间不能早于开始时间")

    return (end - start).total_seconds() / 60


record = {
    "start": "2026-11-02T09:00:00+08:00",
    "end": "2026-11-02T09:45:00+08:00",
}

print(session_minutes(record))
```

输出：

```text
45.0
```

### 10.3 为什么先转成 UTC？

我们计算的是时间线上的实际经过时长。

先统一 UTC 可以避免本地钟表偏移变化干扰意图，也方便不同来源偏移的记录比较。

这不表示用户界面必须显示 UTC，展示时仍可以转回用户地区。

### 10.4 这个函数有没有验证全部记录结构？

没有。

它示范日期解析与时长计算，假设调用者已经提供 `start` 和 `end` 键。

完整数据入口还应检查顶层对象、必填键、额外字段和最大允许时长等规则。

JSON 解析、日期解析和业务模式校验分别负责不同层次。

## 第十一章：完整实践三——生成按日期命名的报告

### 11.1 先决定“哪一个地区的日期”

```python
from datetime import datetime, timedelta, timezone
from pathlib import Path

business_zone = timezone(timedelta(hours=8))
now = datetime.now(business_zone)
filename = f"report-{now.date().isoformat()}.txt"
output_path = Path("datetime_practice") / filename

output_path.parent.mkdir(parents=True, exist_ok=True)
with output_path.open("x", encoding="utf-8", newline="\n") as target:
    target.write(f"生成时刻：{now.isoformat()}\n")
```

这个例子采用固定 UTC+8 业务日历。

如果业务地区有夏令时，应改用相应 `ZoneInfo`。

### 11.2 为什么先取一次 `now`？

如果分别调用多次“现在”，恰好跨过午夜，文件名日期与正文时刻可能落在不同天。

一次取得相关基准，再从同一个对象提取日期和格式化内容，更容易保持一致。

### 11.3 同一天重复运行呢？

本例使用 `x`，同名文件已存在就拒绝覆盖。

你可以改成追加、加入时间或序号，但要明确政策。

仅在文件名加入秒，不保证同一秒内多个任务不会冲突；需要唯一创建时仍要靠实际创建步骤处理。

## 第十二章：练习与参考答案

### 12.1 练习一：跨月日期

给定 2026 年十一月二十八日，计算七天以后。

```python
from datetime import date, timedelta

answer = date(2026, 11, 28) + timedelta(days=7)
assert answer == date(2026, 12, 5)
```

重点是日期对象负责跨月规则，不手工给日字段加七。

### 12.2 练习二：负时间差

解释 `timedelta(seconds=-1).seconds` 为什么不是负一，并取得真正总秒数。

```python
from datetime import timedelta

duration = timedelta(seconds=-1)
assert duration.days == -1
assert duration.seconds == 86399
assert duration.total_seconds() == -1
```

组成字段与总量不同，正负值和跨天情形都要考虑。

### 12.3 练习三：读取并输出另一种格式

```python
from datetime import datetime

value = datetime.strptime("2026/11/02 08:30", "%Y/%m/%d %H:%M")
assert value.strftime("%Y-%m-%d %H:%M") == "2026-11-02 08:30"
```

确认月份和分钟格式码没有写反。

### 12.4 练习四：同一个时刻的两种偏移

```python
from datetime import datetime, timedelta, timezone

beijing = datetime(2026, 11, 2, 9, tzinfo=timezone(timedelta(hours=8)))
utc_value = beijing.astimezone(timezone.utc)

assert utc_value.hour == 1
assert utc_value == beijing
```

显示小时不同，不表示真实时刻不同。

### 12.5 练习五：拒绝没有时区的外部记录

使用前面的 `parse_instant()`，分别传入带偏移和不带偏移的字符串。

带偏移应返回 UTC aware datetime；不带偏移应报告输入约定错误。

不要为了让所有输入通过，就猜测系统本地时区。

### 12.6 练习六：生日年龄为什么不能只减年份？

年份差还没有考虑今年生日是否已经到来。

```python
from datetime import date


def age_on(birthday, today):
    if today < birthday:
        raise ValueError("参考日期不能早于出生日期")
    before_birthday = (today.month, today.day) < (birthday.month, birthday.day)
    return today.year - birthday.year - int(before_birthday)


assert age_on(date(2006, 5, 20), date(2026, 5, 19)) == 19
assert age_on(date(2006, 5, 20), date(2026, 5, 20)) == 20
```

这个算法采用普通月日比较；二月二十九日出生在非闰年的庆祝或法定年龄政策可能有特定规定，业务应另行明确。

## 第十三章：实际选择接口时，先问清楚含义

只处理年月日，考虑 `date`。

记录一个真实发生时刻，考虑带明确时区的 `datetime`。

表示固定经过多久，使用 `timedelta`，取得总量时注意 `total_seconds()`。

生成可交换文本，优先约定清楚的 ISO 形式和偏移。

处理地区历史与夏令时，搭配 `zoneinfo`。

测量程序运行耗时，转到 `time.perf_counter()` 或单调时钟。

最重要的是先确定你处理的是日历日期、本地钟表时间、全球时刻，还是经过时长，再决定用什么对象和运算。

配套阅读：[Python-time库.md](Python-time库.md)、[Python-zoneinfo库.md](Python-zoneinfo库.md)、[Python-json库.md](Python-json库.md)。

官方参考：[datetime](https://docs.python.org/3.14/library/datetime.html)、[aware 与 naive](https://docs.python.org/3.14/library/datetime.html#aware-and-naive-objects)、[格式解析与格式码](https://docs.python.org/3.14/library/datetime.html#strftime-and-strptime-behavior)。
