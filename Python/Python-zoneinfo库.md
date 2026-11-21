# Python `zoneinfo` 库详细教程：地区时区、夏令时、重复时间与跨地区实践

## 零基础先导：为什么时区不是简单的“加减八小时”？

你希望把某个 UTC 时间显示成上海、洛杉矶、纽约的当地时间。固定偏移 `+08:00` 能描述某一个确定的 UTC 偏移，却不能独立包含“某地区过去什么时候改过时区、未来什么时候切换夏令时”的日期规则。

`zoneinfo` 是 Python 3.9 起提供的标准库模块，用于访问 IANA 地区时区数据库的规则。`ZoneInfo("America/New_York")` 代表的是**纽约这个地区的时区规则**，不是一个永远固定的 `-05:00`。

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

instant = datetime(2026, 7, 1, 12, 0, tzinfo=timezone.utc)  # 明确的 UTC 瞬间
new_york = instant.astimezone(ZoneInfo("America/New_York")) # 同一瞬间换显示时区
print(new_york.isoformat())                                  # 当地日期、时间和偏移
```

`datetime(..., tzinfo=timezone.utc)` 创建带时区信息的时刻；`astimezone(...)` 是**转换同一个时间点的呈现**。不要把 `replace(tzinfo=...)` 当成相同的转换：`replace` 是换上时区解释，通常并不调整时钟上的数字。

如果 Windows 等环境缺少 IANA 时区数据库，虽然模块存在，`ZoneInfo(...)` 仍可能抛 `ZoneInfoNotFoundError`；可以按项目环境安装 `tzdata`。对于夏令时结束时“同一个当地钟表时间重复出现”的情形，还要理解 `fold`，不可凭字符串相同认定是同一真实时刻。建议先学习 `datetime` 教程里的 aware/naive，再看本篇夏令时章节。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

假设服务器记录了一个 UTC 时刻，你想把它显示成上海时间和纽约时间。

再假设你想安排“纽约每天上午九点提醒”，希望夏令时切换之后仍然是当地九点。

这两个需求都涉及地区时区。

只写固定 `+08:00` 或 `-05:00`，不一定足够表达某个地区随日期变化的规则。

`zoneinfo` 从 Python 3.9 起进入标准库，用 IANA 时区数据库为 datetime 提供具体地区规则。

本文代码以 Python 3.11 及以上为基础；示例日期用于解释已核对的规则，实际部署仍应保持时区数据更新。

## 第一章：为什么已经有 `timezone`，还需要 `ZoneInfo`？

### 1.1 固定偏移能表示什么？

```python
from datetime import timezone, timedelta

fixed = timezone(timedelta(hours=-5))
```

这个对象表达始终比 UTC 慢五小时的固定偏移。

它不知道某地区哪一天开始夏令时，也不知道历史上是否更改过时区规则。

### 1.2 地区时区带着日期规则

```python
from zoneinfo import ZoneInfo

new_york = ZoneInfo("America/New_York")
```

这个键指向 IANA 数据库中的地区规则。

实际偏移要结合日期和时刻确定。

因此，纽约在某些日期可能是 UTC-5，另一些日期可能是 UTC-4。

这不是同一个时区对象突然不一致，而是地区规则本来就随时间变化。

### 1.3 名称为什么使用 `地区/城市`？

时区键用于稳定指向一套地区规则，不等于当前显示的缩写。

`Asia/Shanghai`、`America/New_York` 是常见键。

`CST` 这类缩写在世界不同地区可能有不同含义，不适合作为普遍唯一标识。

别根据一个国家的名字，假设它全国都只有一套时区。

## 第二章：安装 Python 后，为什么仍然可能找不到时区？

### 2.1 模块代码与数据库是两部分

`zoneinfo` 本身属于标准库。

但它需要可用的时区数据。

通常先使用系统时区数据库，找不到时可以使用第一方维护的 `tzdata` 包。[官方数据来源说明](https://docs.python.org/3.14/library/zoneinfo.html#data-sources)

### 2.2 Windows 与 WSL 可能表现不同

某些 Windows 环境没有可供此接口直接使用的完整 IANA 数据库。

WSL Linux 则常由发行版提供系统数据。

因此，同一台电脑上两个 Python 环境可能有不同数据来源，不能因为 WSL 成功就保证 Windows 环境也成功。

### 2.3 `ZoneInfoNotFoundError`

```python
from zoneinfo import ZoneInfo, ZoneInfoNotFoundError

try:
    business_zone = ZoneInfo("Asia/Shanghai")
except ZoneInfoNotFoundError:
    print("当前环境缺少对应时区数据，请检查键名和数据安装")
```

可能是键名拼错，也可能是环境缺少数据。

若项目需要跨平台可靠使用，可以把 `tzdata` 纳入项目依赖：

```bash
python -m pip install tzdata
```

这条命令应在项目环境执行，不需要修改全局网络代理或系统时区。

### 2.4 时区数据会变化吗？

会。

地区规则包含历史与政策调整，数据更新可以影响未来时间解释。

长期运行的预约、排班或跨地区业务，应考虑数据版本与更新政策。

不要把某一年的偏移表复制进代码后，就宣称永远正确。

## 第三章：把同一时刻显示在两个地区

### 3.1 先取得一个明确 UTC 时刻

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

instant = datetime(2026, 1, 15, 1, 0, tzinfo=timezone.utc)

shanghai = instant.astimezone(ZoneInfo("Asia/Shanghai"))
new_york = instant.astimezone(ZoneInfo("America/New_York"))

print(instant.isoformat())
print(shanghai.isoformat())
print(new_york.isoformat())
```

在对应时区数据下，通常输出：

```text
2026-01-15T01:00:00+00:00
2026-01-15T09:00:00+08:00
2026-01-14T20:00:00-05:00
```

### 3.2 为什么纽约日期还是前一天？

UTC 凌晨一点，在 UTC-5 的地区对应前一天晚上八点。

时区转换可能改变日期，不只是把小时加减。

自己用 `hour - 5`，无法正确处理跨日、跨月和跨年。

`astimezone()` 会把这些关系一起处理。

### 3.3 三个对象相等吗？

这个普通非歧义示例中，它们表示同一个真实时刻，可以相等。

显示的日期与钟点不同，时间线位置相同。

因此，展示值和事件身份应分开理解。

### 3.4 如果想知道某个地区现在是几号呢？

```python
from datetime import datetime
from zoneinfo import ZoneInfo

now = datetime.now(ZoneInfo("America/New_York"))
print(now.date())
```

这比在服务器上直接 `date.today()` 更明确地表达“纽约的当前日期”。

不要把操作系统的默认本地时区假设成所有用户的业务时区。

## 第四章：给本地时间附上地区，与转换时区

### 4.1 数据明确说“纽约上午九点”

可以构造：

```python
from datetime import datetime
from zoneinfo import ZoneInfo

meeting = datetime(2026, 1, 15, 9, 0, tzinfo=ZoneInfo("America/New_York"))
print(meeting.isoformat())
```

这提供当地日期与钟点，再按地区规则解释。

### 4.2 如果原来已经是一个 UTC 时刻呢？

应该转换：

```python
local = utc_moment.astimezone(ZoneInfo("America/New_York"))
```

不要直接用 `replace(tzinfo=...)` 重贴标签。

`replace()` 保留钟表数字并更换解释，可能改变真实时刻。

`astimezone()` 保留真实时刻并换一种本地显示。

### 4.3 直接构造会验证这个钟点真的存在吗？

不一定。

夏令时向前跳时，某些本地钟点根本没有发生。

向后拨时，某些本地钟点又会发生两次。

`datetime(..., tzinfo=ZoneInfo(...))` 不会自动替你的应用做完整严格本地时间有效性判断。

接下来分别看这两种情况。

## 第五章：重复的一点三十分，`fold` 如何区分？

### 5.1 时钟向后拨，会出现两个同样的钟表读数

在适用规则下，纽约 2026 年十一月一日结束夏令时。

当地一点三十分会出现两次。

光保存：

```text
2026-11-01 01:30
```

无法判断对应哪一次。

### 5.2 使用 `fold=0` 与 `fold=1`

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

zone = ZoneInfo("America/New_York")
first = datetime(2026, 11, 1, 1, 30, tzinfo=zone, fold=0)
second = datetime(2026, 11, 1, 1, 30, tzinfo=zone, fold=1)

print(first.isoformat())
print(second.isoformat())
print(first.astimezone(timezone.utc).isoformat())
print(second.astimezone(timezone.utc).isoformat())
```

对应数据下输出：

```text
2026-11-01T01:30:00-04:00
2026-11-01T01:30:00-05:00
2026-11-01T05:30:00+00:00
2026-11-01T06:30:00+00:00
```

`fold=0` 选择较早的那一次。

`fold=1` 选择较晚的那一次。

这是 PEP 495 为本地时间歧义提供的标记。[设计说明](https://peps.python.org/pep-0495/)

### 5.3 `fold=1` 是不是永远表示冬令时？

不应该这样定义。

它表示重复区间中较晚的一次，具体偏移由地区规则决定。

历史变化不一定都符合简单的夏令时与冬令时二分。

### 5.4 为什么不要只用 `first == second` 判断是否同一时刻？

同一个 `tzinfo` 对象下，datetime 比较与运算对 fold 有特定规则，可能忽略这个区别。

需要比较真实时间线位置时，先转换为 UTC，或采用明确时间戳表示。

```python
assert first.astimezone(timezone.utc) != second.astimezone(timezone.utc)
```

不要把两个本地显示值的普通比较，扩大解释成完整的时间身份判定。

## 第六章：不存在的两点三十分，为什么仍能构造对象？

### 6.1 时钟向前跳，会跳过某段本地时间

在适用规则下，纽约 2026 年三月八日开始夏令时。

凌晨两点这一段会被跳过。

直接构造：

```python
from datetime import datetime
from zoneinfo import ZoneInfo

local = datetime(2026, 3, 8, 2, 30, tzinfo=ZoneInfo("America/New_York"))
print(local)
```

对象可能仍然被创建，并不会因为这段本地时间不存在就自动抛异常。

### 6.2 可以怎样观察问题？

从本地转到 UTC，再转回来：

```python
from datetime import timezone

round_trip = local.astimezone(timezone.utc).astimezone(local.tzinfo)

print(local.isoformat())
print(round_trip.isoformat())
```

你可能发现返回的本地钟点已经变成另一时间。

这提示：原来的本地读数并没有对应一个正常往返的真实时刻。

### 6.3 应用应该怎么处理？

常见政策包括拒绝、要求用户选择另一时间，或者明确平移到下一个有效时间。

这不是地区库能够替所有应用统一决定的业务规则。

预约系统、学习提醒和日志导入可能有不同要求，应写清楚。

### 6.4 一个区分零个、一个或两个候选时刻的教学函数

```python
from datetime import datetime, timezone


def local_time_candidates(naive, zone):
    if not isinstance(naive, datetime):
        raise TypeError("naive 必须是 datetime")
    if naive.tzinfo is not None:
        raise ValueError("请输入尚未附时区的本地日期时间")

    by_instant = {}

    for fold in (0, 1):
        candidate = naive.replace(tzinfo=zone, fold=fold)
        instant = candidate.astimezone(timezone.utc)
        restored = instant.astimezone(zone)

        if restored.replace(tzinfo=None) == naive:
            by_instant.setdefault(instant, candidate)

    return [by_instant[instant] for instant in sorted(by_instant)]
```

### 6.5 函数每一步在检查什么？

它先分别尝试两个 fold 解释。

把候选转换成 UTC，再按同一地区还原。

只有本地年月日时分秒能正常往返，才保留这个候选。

最后按 UTC 时刻去重，避免普通无歧义时间因为两个 fold 尝试被重复计算。

零个候选表示本例规则下没有有效本地时刻。

一个候选表示唯一。

两个候选表示需要进一步选择。

这是利用具体 `ZoneInfo` 和 datetime 往返规则的教学方案，不是所有自定义 `tzinfo` 实现都可以未经验证地照搬。

## 第七章：一天以后，与实际经过二十四小时

### 7.1 先看跨夏令时的中午

```python
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo

zone = ZoneInfo("America/New_York")
start = datetime(2026, 3, 7, 12, 0, tzinfo=zone)
next_local_noon = start + timedelta(days=1)

print(start.isoformat())
print(next_local_noon.isoformat())
print((next_local_noon - start).total_seconds() / 3600)
print((next_local_noon.astimezone(timezone.utc) - start.astimezone(timezone.utc)).total_seconds() / 3600)
```

对应数据下，后两项是二十四和二十三。

### 7.2 为什么会有两个答案？

同一地区对象上的本地日历运算保留了中午这个钟点。

但这一天时钟向前跳过一小时，时间线上实际经过二十三小时。

两种答案对应不同问题。

### 7.3 如果我要实际经过二十四小时呢？

```python
after_24_hours = (start.astimezone(timezone.utc) + timedelta(hours=24)).astimezone(zone)
```

先在 UTC 时间线上增加固定时长，再转换回本地展示。

它可能变成本地下午一点，而不是次日中午。

### 7.4 每天九点提醒应该选哪一种？

通常是按当地日历日期组合九点，再处理歧义或不存在时间政策。

并不是每次从上一次 UTC 时刻机械增加八万六千四百秒。

反过来，令牌有效三千六百秒等固定经过时长，也不应随本地钟表跳动改变。

真实调度还涉及系统可用性、任务恢复和是否补跑，本篇只解释时区计算层。

## 第八章：序列化时，偏移和地区名称分别保存什么？

### 8.1 ISO 字符串保留偏移，不自动保留地区规则

```python
text = meeting.isoformat()
```

例如包含 `-05:00`，能帮助确定这个具体时刻。

但下次解析时，通常不会自动恢复出 `America/New_York` 这个地区键和未来规则。

### 8.2 一个事件记录的常见模型

```json
{
  "occurred_at_utc": "2026-01-15T14:00:00+00:00",
  "display_zone": "America/New_York"
}
```

UTC 字段记录真实事件时刻。

地区键说明希望怎样展示。

如果存的是未来“每天当地九点”的规则，则还需要当地钟点、重复政策、歧义处理和地区键，不能只保存一次事件的 UTC 值就代表长期日程。

### 8.3 为什么不只存 `tzname()`？

缩写可能不唯一，也可能随日期变化。

`ZoneInfo.key` 更适合保存明确的 IANA 地区键。

这个键仍依赖时区数据库解释，长期业务应考虑数据更新政策。

## 第九章：缓存和数据更新，普通程序需要知道什么？

### 9.1 相同 key 常得到缓存对象

`ZoneInfo` 会缓存常规构造的地区对象。

这能避免每次重新加载相同规则，也会影响某些依赖 `tzinfo` 身份的 datetime 运算语义。

普通应用不需要为了“更新一下时间”频繁清空缓存。

### 9.2 更新数据库后，老对象会自动变成新规则吗？

不要这样假设。

已经创建的对象可能继续使用其加载的规则资料。

长期运行进程应按部署和重启策略处理数据更新，不宜在不了解影响时随意调用全局缓存清理。

### 9.3 `available_timezones()` 能做什么？

它可以取得当前环境可用的时区键集合。

```python
from zoneinfo import available_timezones

zones = available_timezones()
print("Asia/Shanghai" in zones)
```

可用集合反映当前数据环境，不代表你的业务应该让用户随意输入任何文件系统路径作为时区。

用户界面也可以提供经过筛选的地区选择，避免把几百个键直接扔给初学者。

## 第十章：练习与参考答案

### 10.1 练习一：同一时刻跨地区显示

把 `2026-01-15T01:00:00+00:00` 显示为上海和纽约时间。

```python
from datetime import datetime
from zoneinfo import ZoneInfo

instant = datetime.fromisoformat("2026-01-15T01:00:00+00:00")
assert instant.astimezone(ZoneInfo("Asia/Shanghai")).hour == 9
assert instant.astimezone(ZoneInfo("America/New_York")).day == 14
```

记得日期也可能改变。

### 10.2 练习二：重复与缺失

使用前面的 `local_time_candidates()`：

```python
from datetime import datetime
from zoneinfo import ZoneInfo

zone = ZoneInfo("America/New_York")
assert len(local_time_candidates(datetime(2026, 1, 15, 9), zone)) == 1
assert len(local_time_candidates(datetime(2026, 11, 1, 1, 30), zone)) == 2
assert len(local_time_candidates(datetime(2026, 3, 8, 2, 30), zone)) == 0
```

这些断言依赖对应地区数据规则，环境需具备 IANA 数据。

### 10.3 练习三：分别计算本地日历差与真实经过时长

复现跨三月八日的中午例子，解释为什么一个差是二十四小时，一个差是二十三小时。

答案应说清楚时区对象、钟表变化和 UTC 时间线，而不只是记住数字。

### 10.4 练习四：拒绝不认识的地区键

```python
from zoneinfo import ZoneInfo, ZoneInfoNotFoundError

try:
    ZoneInfo("Not_A_Real_Region/No_Such_City")
except ZoneInfoNotFoundError:
    print("请输入有效的地区时区键")
```

实际配置加载器应保留清楚的字段位置和用户提示，而不是悄悄回退到机器本地时区。

## 第十一章：实际使用时的思考顺序

先确定输入描述的是已经发生的真实时刻，还是一个当地日历安排。

已发生时刻最好带明确偏移，并可统一 UTC 保存。

当地安排要有地区键，并为重复或不存在钟点选择明确政策。

展示时用 `astimezone()` 换算，不随手重贴 `tzinfo`。

计算固定经过时长，优先在 UTC 时间线上比较。

时区数据缺失和键名错误要清楚报告，不能把服务器本地时区当作无声默认。

配套阅读：[Python-datetime库.md](Python-datetime库.md)、[Python-time库.md](Python-time库.md)。

官方参考：[zoneinfo](https://docs.python.org/3.14/library/zoneinfo.html)、[datetime 算术与比较](https://docs.python.org/3.14/library/datetime.html#datetime-objects)、[PEP 495](https://peps.python.org/pep-0495/)、[PEP 615](https://peps.python.org/pep-0615/)。
