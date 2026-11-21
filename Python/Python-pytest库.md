# Python pytest 超详细教程：从第一个断言，到文件、命令行、HTTP 与数据库的可靠测试

## 零基础先导：为什么“我手动运行过一次”不等于程序可靠？

你写了一个 `add(a,b)`，第一次试 `add(1,2)` 得到了 3。可是其他输入怎么办？下次改代码会不会把旧功能弄坏？测试的目的是：**把预期行为写成可以重复检查的条件**。`pytest` 是常用第三方测试框架，帮你发现测试函数、运行它们、显示失败差异，并管理测试用的准备与清理。

### 第一个测试只需要两个文件

`calc.py`：

```python
def add(a, b):
    return a + b
```

`test_calc.py`：

```python
from calc import add       # 导入要测试的函数

def test_add():
    assert add(2, 3) == 5 # 预期结果是 5；不成立就报告失败
```

先执行安装 `python -m pip install pytest`，然后在文件所在目录执行 `python -m pytest -q`。

- `test_calc.py`：默认文件发现规则会识别 `test_*.py` 或 `*_test.py`。
- `test_add`：默认以 `test_` 开头的函数会被收集为测试。
- `assert`：Python 自带的断言语句。`pytest` 会改进失败时的展示，但不是 `assert` 的发明者。
- 如果 `add(2, 3)` 变成 6，测试失败，帮你定位“实际与预期”不一致。

### 为什么后面还会有 fixture、parametrize、monkeypatch？

当多个测试都需要临时文件/数据库连接时，用 `fixture` 统一准备和清理；当同一个函数要测多个输入时，用 `@pytest.mark.parametrize` 避免机械复制；当测试不应该真的发送邮件或网络请求时，用 `monkeypatch` 临时替换依赖。**这些都是为测试复杂度增加服务的，不是写第一个测试的必要前提。**

注意：**pytest 检查的是你写出来的预期**。如果你断言了错误的业务规则，测试全绿也不代表需求正确；也不能把线上真实数据库当作临时练习环境随意清理。

---

你写完了一个函数，手动运行了一次，结果正确。

过几天，你为另一个需求修改了其中两行。

新的场景能用了，原先那个场景却悄悄坏了。

如果每次都靠记忆手动输入、肉眼比较输出，很快就会漏掉一些情况。

pytest 帮助你把“我希望程序怎样表现”写成可以重复执行的检查。

它是常用的第三方测试工具，不是大型业务框架。

本文以 Python 3.11 及以上的常用语法、pytest 9.2.0 的实际核验环境讲解。

绝大多数基础内容也适用于较早的现代 pytest，但版本有关的行为会单独提醒。

文中的 `python` 代表你为项目选择的解释器。若 WSL 未激活虚拟环境且只提供 `python3`，相应命令使用 `python3 -m pytest`，或直接使用虚拟环境内的解释器路径。

我们先让一个只有几行的测试真正跑起来，再逐步讨论文件、环境变量、外部请求、日志和 SQLite。

## 第一章：测试到底在检查什么？

### 1.1 “没有报错”只是最弱的一种观察

假设你要计算购物车总价：

```python
def calculate_total(prices):
    return sum(prices)
```

手动试一下：

```python
print(calculate_total([10, 20, 30]))
```

输出：

```text
60
```

你知道这一组输入现在能得到六十。

但你还不知道空列表怎样处理、负数是否允许、价格单位是否统一，以及将来改代码后还能否得到同样结果。

测试首先要把预期说清楚。

### 1.2 测试不是把代码运行一遍就结束

真正的检查至少需要两件事：执行行为，以及比较预期。

```python
result = calculate_total([10, 20, 30])
assert result == 60
```

`assert` 表达“我要求这个条件成立”。

条件为真时，程序继续。

条件为假时，抛出 AssertionError。

pytest 会识别这类失败，并报告是哪一项预期没有满足。

### 1.3 预期从哪里来？

从需求和协议来。

如果价格列表可以为空，空列表总价是零，测试就应明确写零。

如果空购物车是业务错误，则应该测试它抛出约定异常。

不能只是看当前函数返回什么，就照抄什么当“正确答案”。

否则实现写错，测试会忠实地把错误固定下来。

### 1.4 为什么测试代码也可以有 bug？

因为它也是 Python 程序。

测试可能漏掉关键分支、比较了错误变量，或者根本没被发现执行。

因此，一份测试显示 passed，不等于程序所有行为都正确。

它表示已执行的那些检查，在当前环境和输入下通过。

之后我们会反复问：如果故意把实现改坏，这个测试能不能发现？

这比只追求绿色数量更有用。

### 1.5 pytest 与 unittest 是什么关系？

unittest 是标准库中的测试工具。

pytest 是第三方项目，支持简洁的函数测试、普通断言、参数化和丰富夹具等，也能运行许多 unittest 测试。

你可以在现有项目约定下选择，不需要为了学 pytest 先否定所有其他工具。

本篇主要采用 pytest 自己的常见写法，从普通函数入门。

## 第二章：先运行一个完整的小项目

### 2.1 建立练习目录

先准备：

```text
pytest-demo/
├── calculator.py
└── test_calculator.py
```

把下面内容写入 `calculator.py`：

```python
def add(left, right):
    return left + right
```

再把下面内容写入 `test_calculator.py`：

```python
from calculator import add


def test_add_two_positive_numbers():
    result = add(2, 3)
    assert result == 5
```

先不要添加复杂目录、插件和配置。

我们需要先确认测试发现、导入和断言这条最短路径正常。

### 2.2 安装在你真正使用的环境里

在项目虚拟环境中执行：

```bash
python -m pip install pytest
```

然后检查：

```bash
python -m pytest --version
```

当前核验环境输出：

```text
pytest 9.2.0
```

为什么反复写 `python -m`？

因为电脑可能有多个 Python 和多个 pytest 命令。

这种写法明确使用当前 Python 环境中的 pytest 模块，减少“包装在另一个环境”的混乱。

在 WSL 中，如果你的解释器命令是 python3，按相同方式使用 `python3 -m pytest`。

### 2.3 运行测试

在 pytest-demo 目录执行：

```bash
python -m pytest -q
```

通常会看到类似：

```text
.                                                                        [100%]
1 passed in 0.01s
```

时间和进度排版与环境有关，不要把它们当成固定输出。

`.` 表示一个测试通过。

`1 passed` 表示一个测试项通过，不是“整个软件已得到完备证明”。

### 2.4 这里发生了什么？

pytest 寻找符合规则的测试文件。

导入测试模块。

发现符合命名规则的测试函数。

调用函数，执行 add，再检查断言。

断言没有失败，测试被记为通过。

这个过程可以反复运行，不需要你每次手工输入二和三。

### 2.5 如果我只是双击 test_calculator.py 呢？

普通执行 Python 文件，只会按普通模块规则执行其中顶层语句。

定义了一个 test_ 函数，不意味着普通解释器会自动调用它。

pytest 运行器才会按照自己的测试发现与执行流程调用这些函数。

因此，用 `python test_calculator.py` 没有输出，不足以证明那个测试真的执行了。

### 2.6 先故意写错一次

把测试里的五改成六：

```python
from calculator import add


def test_add_reports_wrong_expectation():
    assert add(2, 3) == 6
```

运行后应出现失败报告，指出实际结果五与预期六不相等。

理解失败报告，是测试入门的一部分。

这段故意错误的测试用于观察失败，完成练习后应改回正确预期或移出正式通过集合。

不要把它混在正常套件中，又用忽略失败的命令掩盖结果。

## 第三章：pytest 怎么找到我的测试？

### 3.1 常见默认命名规则

常见默认情况下，文件采用 `test_*.py` 或 `*_test.py`。

函数通常以 `test_` 开头。

测试类通常以 `Test` 开头，且不定义普通构造器来承载测试准备。

例如：

```python
class TestCalculator:
    def test_add_zero(self):
        assert 3 + 0 == 3
```

初学时直接使用函数已经足够。

类可以用于组织相关测试，但不是每个测试都必须写类。

### 3.2 文件名叫 tests.py 为什么没发现？

默认规则未必把这个名字当作你想要的测试模块。

你可以明确传入文件，或者调整配置中的发现规则。

但入门阶段最省心的选择是遵守 `test_xxx.py` 与 `test_xxx`。

先避免自己制造一套没有必要的新命名协议。

### 3.3 怎样只看发现了什么，而不执行？

```bash
python -m pytest --collect-only -q
```

它会列出收集到的测试项。

例如：

```text
test_calculator.py::test_add_two_positive_numbers
```

这特别适合排查“我明明写了测试，却没有跑”的问题。

但收集阶段仍然会导入模块、加载相关配置和插件。

如果你的模块顶层有副作用，单纯 collect-only 也可能触发它们。

“没有执行测试函数”不等于“什么代码都不会运行”。

### 3.4 什么是 node id？

刚才这段：

```text
test_calculator.py::test_add_two_positive_numbers
```

就是常见节点标识形式。

文件路径在前，函数或类成员在后，用双冒号连接。

你可以直接运行：

```bash
python -m pytest test_calculator.py::test_add_two_positive_numbers -q
```

这样只执行这一项。

参数化测试还可能在后面带有方括号中的案例标识。

### 3.5 为什么导入阶段就报错，测试还没开始？

pytest 必须先导入测试模块，才能知道其中有哪些测试对象。

如果测试文件顶层导入了不存在的包，或者被测模块导入时直接启动服务器，就可能在收集阶段失败。

这叫收集错误，不是某个 assert 比较失败。

把可执行入口放在合适的 main 防护中，避免导入时读真实文件、发请求或运行任务，有助于测试，也有助于正常模块复用。

### 3.6 一个文件没有测试，退出码也重要

pytest 常用退出状态包括全部通过的零，以及失败、使用错误、没有收集到测试等不同情况。

尤其“没有测试被执行”不能自动当作通过。

自动化脚本需要检查退出状态，而不是只检查控制台有没有红色。

确切含义可查[官方退出状态说明](https://docs.pytest.org/en/stable/reference/exit-codes.html)。

## 第四章：assert 怎样写得既准确又好读？

### 4.1 比较具体结果

```python
def normalize_name(value):
    return value.strip().casefold()


def test_normalize_name_removes_outer_spaces():
    assert normalize_name("  Alice  ") == "alice"
```

这里直接写出输入和预期。

失败时，你能看见实际字符串与预期字符串的差别。

比起只断言“结果不为空”，它检查了真正承诺的规范化行为。

### 4.2 布尔、成员与结构

```python
def test_basic_assertion_forms():
    values = [2, 4, 6]
    assert 4 in values
    assert 3 not in values
    assert len(values) == 3
    assert values == [2, 4, 6]
```

这些不是要求一个测试里永远写四种断言，而是展示不同表达方式。

实际项目应围绕同一项行为组织断言。

如果失败点分别对应完全无关的功能，就应该拆开，让报告更清楚。

### 4.3 `is` 与 `==` 不要混用

```python
def test_none_and_boolean_identity():
    value = None
    enabled = False
    assert value is None
    assert enabled is False
```

`is` 检查对象身份。

`==` 检查相等性。

None 通常用 `is None`。

数值和字符串的值比较通常用 `==`。

不要因为小整数缓存让某段 `a is b` 偶然通过，就把它当成数值比较语法。

### 4.4 自定义断言消息什么时候有用？

```python
def test_record_count():
    records = ["a", "b"]
    assert len(records) == 2, "导入后应保留两条有效记录"
```

消息可以补充业务语境。

但如果消息只是“断言失败”，没有增加帮助。

pytest 会对测试中的普通 assert 提供丰富解释，没必要把每个比较都封装成返回 bool 的黑盒函数。

### 4.5 为什么失败报告能显示中间值？

pytest 会对受管理的测试模块中的断言做相应处理，以便失败时展示表达式信息。

因此，你经常能看到列表差异、字典差异和实际调用结果。

但这个能力不意味着所有第三方辅助模块里的 assert 都自动获得完全相同的解释。

复杂共享断言辅助模块需要按工具机制考虑，普通入门先保持断言接近测试函数。

### 4.6 业务验证能不能也只用 assert？

不推荐把必须执行的业务校验仅写成 assert。

Python 优化运行会影响普通断言执行，业务契约不应因此消失。

测试代码使用断言来检查程序，是测试工具的正常模式。

业务函数面对非法输入，应明确抛出 ValueError、自定义异常等合适错误。

测试验证“这个错误会发生”，而不是让业务本身依赖测试断言机制。

[官方断言说明](https://docs.pytest.org/en/stable/how-to/assert.html)还介绍了更详细的失败解释与比较行为。

## 第五章：一个失败报告，应该从哪里读？

### 5.1 先看是哪一项失败

报告会指出测试文件和函数名称。

这通常比先盯着最后一行 traceback 更容易建立语境。

如果测试名是 `test_x1`，读者很难知道在检查什么。

如果叫 `test_empty_cart_has_zero_total`，失败含义会更清楚。

测试名是给未来排错的人看的，不只是满足发现规则。

### 5.2 再看断言两侧

例如：

```text
assert 5 == 6
```

说明执行走到了比较位置，实际值与预期不一致。

这与导入错误、夹具准备失败、文件打不开等情况不同。

先确定失败阶段，再决定检查被测业务、测试预期还是环境准备。

### 5.3 失败和错误都要处理

控制台中的 FAILED 常对应执行测试时预期不成立等问题。

ERROR 常可能发生在收集或准备、清理等阶段。

不要看到“没有 FAILED”就直接认为成功；一个夹具根本没准备好，测试主体可能完全没执行。

查看完整汇总与进程退出状态。

### 5.4 怎样让输出更详细？

```bash
python -m pytest -v
```

会更明确显示每个测试项。

想缩短 traceback：

```bash
python -m pytest --tb=short
```

想在第一项失败后停止：

```bash
python -m pytest -x
```

这些选项控制观察与执行策略，不改变业务是否正确。

排查时可以缩短反馈循环，确认修复后仍需要运行与改动范围有关的完整检查。

### 5.5 为什么不要一开始就捕获所有异常？

如果测试写成：

```python
def test_bad_testing_style():
    try:
        int("abc")
    except Exception:
        pass
```

它没有检查异常类型，也没有在完全不抛异常时失败。

以后实现返回一个错误默认值，这项测试仍可能通过。

异常测试应该有明确的“必须抛出”和“抛出什么”预期。

下一章用 pytest.raises 解决。

## 第六章：pytest.raises，怎样测试本来就应该失败的输入？

### 6.1 先写有明确契约的函数

```python
def parse_quantity(text: str) -> int:
    value = int(text)
    if value < 0:
        raise ValueError("数量不能为负数")
    return value
```

数字文本允许转换。

负数不允许。

零是否允许？允许，因为判断是小于零，而不是小于等于零。

这类边界就是测试应该帮你固定的需求。

### 6.2 用 with 表达预期异常

```python
import pytest


def test_negative_quantity_is_rejected():
    with pytest.raises(ValueError):
        parse_quantity("-1")
```

如果代码抛出符合预期的异常，测试通过。

如果没有抛出，测试失败。

如果抛出不符合预期的异常，通常也会失败，并保留异常信息。

### 6.3 match 参数是正则表达式

```python
def test_negative_quantity_message():
    with pytest.raises(ValueError, match="数量不能为负数"):
        parse_quantity("-1")
```

这里检查消息含有相应模式。

match 使用正则规则，不是所有字符都按字面比较。

如果消息中有括号、点号等特殊字符，而你确实想按字面匹配，可以先用 re.escape 处理预期字符串。

不要把第三方错误完整文本当成永远稳定接口，除非你有明确理由。

### 6.4 需要检查异常对象呢？

```python
def test_exception_object_can_be_inspected():
    with pytest.raises(ValueError) as captured:
        parse_quantity("-1")
    assert str(captured.value) == "数量不能为负数"
```

`captured.value` 是实际异常实例。

注意读取位置在 with 之后。

如果异常发生，with 内异常语句后面的普通语句不会继续执行。

### 6.5 with 范围为什么要尽量小？

不要把读文件、准备数据、调用目标函数和多次无关操作全部放进去。

如果前置准备就抛出同类型异常，测试可能误以为目标行为正确。

把准备放在前面，只把你希望抛异常的操作放在 raises 内。

这样“谁抛出的错误”更清楚。

### 6.6 父类异常会接受子类吗？

通常会按异常继承关系匹配。

如果你写 `pytest.raises(Exception)`，范围太宽，许多编程错误也可能被当作预期行为。

优先选择契约明确的异常类型。

如果确实要求精确类型而非其子类，可以进一步检查 `type(captured.value) is ExpectedError`。

是否需要如此严格，取决于对外异常契约。

### 6.7 怎么测试本来不应该抛错的输入？

直接调用并断言结果即可。

不需要给所有正常调用都套一个“没有异常”的特殊包装。

意外异常本来就会让测试失败，并显示 traceback。

例如：

```python
def test_zero_quantity_is_allowed():
    assert parse_quantity("0") == 0
```

简单直接的表达通常最好读。

## 第七章：浮点、顺序和集合，预期比较要符合数据含义

### 7.1 浮点数为什么不能总精确等号？

```python
print(0.1 + 0.2)
```

常见输出：

```text
0.30000000000000004
```

这是二进制浮点表示带来的现象。

如果业务采用浮点近似计算，可以使用 pytest.approx 描述容差。

```python
import pytest


def test_floating_point_sum():
    assert 0.1 + 0.2 == pytest.approx(0.3)
```

但不要为了掩盖真实误差，无条件把容差设得特别大。

### 7.2 abs 与 rel 怎么理解？

abs 是绝对容差。

rel 是相对容差。

例如测量值允许绝对误差不超过千分之一，可以明确表达：

```python
def test_measurement_with_absolute_tolerance():
    assert 1.0005 == pytest.approx(1.0, abs=0.001, rel=0)
```

接近零时，相对误差概念往往不如绝对误差直观。

测试容差应该来自算法或测量需求，不是失败后随手放宽到能过。

### 7.3 金额应该用 approx 糊过去吗？

如果系统约定使用整数分或 Decimal 精确十进制金额，应按这个契约精确比较。

不要因为浮点测试麻烦，就把所有财务误差都当成允许近似。

测试选择也在反映你的数据表示设计。

### 7.4 顺序到底重要不重要？

文件系统遍历结果通常不保证固定顺序。

如果需求只是得到相同名称集合，可以比较集合，或排序后比较。

```python
def test_names_when_order_is_irrelevant():
    actual = ["b.txt", "a.txt"]
    assert set(actual) == {"a.txt", "b.txt"}
```

但集合会丢掉重复次数。

如果重复也重要，可以使用排序后的列表或 Counter，按需求决定。

### 7.5 如果结果必须按时间排序呢？

那就应该比较顺序。

把 actual 转成 set 再比较，会让“排序坏了”的实现仍然通过。

测试技巧不能替代需求。

一段比较表达式是否合适，首先取决于你真正承诺了什么。

## 第八章：参数化，把不同输入写成可辨认的测试项

### 8.1 三个几乎相同的测试

你可能想检查数量解析：

```text
"0" → 0
"12" → 12
" 12 " → 12
```

三次准备、调用与断言结构相同，只是数据不同。

可以使用 parametrize。

### 8.2 最小写法

```python
import pytest


@pytest.mark.parametrize("text, expected", [
    ("0", 0),
    ("12", 12),
    (" 12 ", 12),
])
def test_parse_quantity_valid_inputs(text, expected):
    assert parse_quantity(text) == expected
```

装饰器里的字符串列出参数名称。

每个元组按对应位置提供一组参数。

pytest 为每组生成独立测试项。

某一组失败，报告会指出对应案例，而不是只知道一个大循环在中途失败。

### 8.3 为什么不是普通 for 循环？

普通循环也能检查多个输入。

但一次失败通常会中断该测试后面的循环，报告粒度也只有一个测试函数。

参数化让案例成为独立项，便于筛选、显示和单独重跑。

如果数据本来就是一个整体序列的状态变化，则可能仍然适合在同一个测试里顺序执行。

不要机械地把所有循环都换成参数化。

### 8.4 给案例起容易看懂的名字

```python
@pytest.mark.parametrize("text, expected", [
    pytest.param("0", 0, id="zero"),
    pytest.param("12", 12, id="positive"),
    pytest.param(" 12 ", 12, id="outer-spaces"),
])
def test_quantity_with_readable_ids(text, expected):
    assert parse_quantity(text) == expected
```

失败节点会带有案例标识。

在大型输出里，outer-spaces 比一长串自动生成值更容易理解。

不要在 id 中放密码、令牌或大段个人信息，测试报告可能被保存在共享系统中。

### 8.5 错误输入也可以参数化

```python
@pytest.mark.parametrize("text", ["-1", "abc", ""])
def test_parse_quantity_rejects_invalid_text(text):
    with pytest.raises(ValueError):
        parse_quantity(text)
```

这几组都约定 ValueError，但来源可能分别是业务范围检查与 int 解析。

如果你需要区分错误类别和消息，可以为每组添加相应预期，而不是只断言所有异常都一样。

### 8.6 参数列表里的可变对象会自动复制吗？

不要这样假设。

参数值可能以原对象传入测试，测试中的原地修改会留下影响。

特别是相同列表或字典被多组引用时，要清楚对象共享关系。

如果每次都需要新对象，可以传不可变输入描述，在测试内构造，或使用合适夹具工厂。

### 8.7 多层参数化会产生什么？

独立装饰器可以组合形成笛卡尔积。

两种模式乘三种输入，可能产生六项。

这对确实独立的维度很有用。

但如果某些组合业务上根本无意义，直接列有效案例通常更清楚。

不要为了增加数字而把所有开关机械相乘。

官方的[参数化说明](https://docs.pytest.org/en/stable/how-to/parametrize.html)可以帮助你进一步了解案例标记与夹具参数化。

## 第九章：fixture 是什么？为什么测试函数的参数会自己出现？

### 9.1 先看重复准备的问题

几个测试都需要一个初始购物车。

你可以在每个函数里写相同字典。

但准备逻辑越来越长时，重复会增加修改成本，也会掩盖各个测试真正关注的行为。

fixture 可以把“测试依赖怎样准备”组织成明确资源。

### 9.2 第一个夹具

```python
import pytest


@pytest.fixture
def cart():
    return {"items": [], "currency": "CNY"}


def test_new_cart_is_empty(cart):
    assert cart["items"] == []


def test_new_cart_uses_cny(cart):
    assert cart["currency"] == "CNY"
```

fixture 常译为夹具。

可以先把它理解为测试所需要的准备或资源提供机制。

不要把英文名称想得太神秘。

### 9.3 为什么不用自己调用 cart()？

测试参数中的 `cart` 告诉 pytest：这项测试需要一个叫 cart 的夹具。

pytest 按夹具解析规则找到对应提供者，执行它，并把结果交给测试参数。

这是测试运行器参与的依赖机制。

不是 Python 普通函数参数自动根据名称寻找全局变量。

如果你脱离 pytest 手工调用测试函数，Python 仍然需要你自己提供参数。

### 9.4 return 的是什么，就拿到什么

上例返回 dict，所以测试里的 cart 是字典。

可以返回 Path、数据库连接、客户端、小对象，甚至返回一个用于创建对象的函数。

夹具没有规定必须返回某种特殊基类。

这也是它容易与普通 Python 代码配合的原因。

### 9.5 两个测试修改同一份列表吗？

默认 function 作用域下，每个测试项通常会获得一次新的夹具执行结果。

上面的 cart 每次执行创建一个新字典和新列表。

因此，一个测试往 items 里追加内容，不应该自然污染另一个测试的新 cart。

但如果夹具返回的是模块级同一个全局对象，即使函数重新执行，也可能仍然共享同一份数据。

作用域与对象具体创建位置，需要一起看。

### 9.6 不使用 fixture 也可以吗？

当然。

一两行准备，直接写在测试里往往最清楚。

当准备逻辑需要复用、依赖其他资源或保证清理时，夹具更有价值。

不要为了学习装饰器就把每个字符串和数字都变成一个夹具，让读者必须到处跳转才能知道输入是什么。

## 第十章：夹具依赖、缓存和作用域

### 10.1 一个夹具也可以需要另一个夹具

```python
import pytest


@pytest.fixture
def base_url():
    return "https://books.example"


@pytest.fixture
def client_options(base_url):
    return {"base_url": base_url, "timeout": 3.0}


def test_client_configuration(client_options):
    assert client_options["base_url"] == "https://books.example"
```

client_options 的参数 base_url 也会按夹具解析。

pytest 根据依赖准备资源，而不需要测试手工一层层调用。

如果依赖关系形成循环，应该修正设计，而不是试图依靠函数定义前后顺序碰运气。

### 10.2 同一项测试中重复依赖，会准备两次吗？

在同一作用域的正常解析中，夹具结果会被缓存复用。

例如测试需要 cart，另一个它依赖的夹具也需要 cart，通常拿到的是同一次准备结果。

这样依赖图中的资源可以保持一致。

但这也意味着其中一个依赖修改了可变对象，另一个使用者可能看得到。

共享是一项机制，也是一项需要理解的责任。

### 10.3 function 作用域

默认作用域通常是 function。

每个测试项独立准备与清理。

参数化后的每个案例也是独立测试项，因此通常分别获得资源。

这适合可变数据、临时文件和需要强隔离的连接状态。

### 10.4 module 与 session

可以写：

```python
@pytest.fixture(scope="session")
def reference_codes():
    return frozenset({"PY", "SQL", "HTTP"})
```

session 通常在一次 pytest 会话中共享。

module 通常在同一测试模块范围共享。

此外还有 class、package 等作用域。

作用域更大，准备次数可能更少，但共享状态持续更久。

只读、昂贵的资源比较适合谨慎扩大作用域；会被测试修改的数据需要特别小心。

### 10.5 把数据库连接设为 session 就更快吗？

可能减少建立连接的成本。

但不同测试可能共享未提交事务、表内容或连接状态，导致顺序依赖。

如果一个测试必须先跑，另一个才会通过，测试隔离就有问题。

先明确数据重置与事务策略，再决定是否共享连接。

性能改进需要真实测量，不能以破坏可重复性为代价盲目扩大作用域。

### 10.6 为什么 session 夹具不能直接依赖普通 tmp_path？

普通 tmp_path 是按测试项提供的临时路径。

一个生命周期更长的 session 夹具，不能自然依赖一个更短生命周期的 function 夹具来持有资源。

这种作用域不匹配通常会报错。

需要会话级临时目录时，可以使用 tmp_path_factory 等适合的入口，后面会详细解释。

### 10.7 参数化会怎样影响共享？

参数化夹具可能在不同参数组合下产生不同实例。

因此，“session 作用域”并不总能简化成“这个函数整次运行只调用一次”的绝对口号。

理解时应同时看作用域、参数化与依赖组合。

复杂共享资源要通过可观察日志或小测试确认生命周期，而不是背一句话替代实际机制。

## 第十一章：yield 夹具，资源到底什么时候关闭？

### 11.1 return 只能提供值，清理怎么办？

打开文件、数据库连接或客户端后，测试结束时需要释放。

你可以在每个测试里写 with，但共同准备越来越多时，夹具能把生命周期集中管理。

```python
import sqlite3
import pytest


@pytest.fixture
def database():
    connection = sqlite3.connect(":memory:")
    try:
        yield connection
    finally:
        connection.close()
```

测试中：

```python
def test_database_can_select(database):
    result = database.execute("SELECT 1").fetchone()
    assert result == (1,)
```

### 11.2 yield 前后分别是什么？

yield 前面是准备阶段。

yield 出去的 connection 交给测试。

测试完成或失败后，pytest 会继续相应清理过程。

finally 里的 close 用来释放连接。

你不需要在每个测试里重复记住关闭同一个夹具资源。

### 11.3 测试断言失败也会清理吗？

在正常 pytest 异常展开过程中，会执行已经建立资源的相应清理。

这正是使用结构化生命周期的意义。

但进程被强制终止、机器断电等情况，不能承诺普通 Python 清理代码一定执行。

不要把夹具清理当作跨所有故障场景的数据恢复系统。

### 11.4 如果 yield 前面已经报错呢？

没有执行到 yield，后面普通清理段可能没有机会按你想象进入。

因此，资源刚获得就用 try/finally 或 with 包住，更容易保证此前已获得的资源能在后续准备失败时释放。

不要先连续创建五种外部资源，最后才统一写 yield，而中途任意一步失败都无人清理前面几项。

### 11.5 为什么推荐一次准备一个可清楚撤销的资源？

资源越多，半完成状态越难处理。

把数据库连接、临时目录和客户端拆成有依赖的夹具，每个夹具负责自己的生命周期，通常比巨大的万能 setup 更清晰。

这不是要求所有准备都必须拆分，而是让每个资源的所有者容易识别。

### 11.6 多个夹具的清理顺序

依赖资源通常需要先于依赖者建立，清理时按适合依赖关系的逆向过程展开。

例如客户端依赖服务器，通常先关闭客户端，再关闭服务器。

如果清理依赖某个具体顺序，就应该通过夹具依赖关系表达。

不要依靠参数列表排列、函数名字排序或文件中的定义位置来隐含控制关键顺序。

### 11.7 还有 addfinalizer 吗？

可以通过 request 等机制登记清理函数。

它适合某些动态资源管理场景，但登记时机必须正确：已经登记的清理可能在之后准备失败时仍被调用。

普通场景优先采用可读的 yield 配合 with 或 try/finally。

先把一种生命周期写明白，再研究更低层的动态机制。

更多细节见[官方夹具生命周期说明](https://docs.pytest.org/en/stable/how-to/fixtures.html)。

## 第十二章：conftest.py，为什么不用到处 import 夹具？

### 12.1 多个文件需要同一份准备

项目逐渐变成：

```text
project/
├── app.py
└── tests/
    ├── conftest.py
    ├── test_files.py
    └── test_config.py
```

把共享夹具放在对应测试目录的 conftest.py：

```python
import pytest


@pytest.fixture
def sample_settings():
    return {"encoding": "utf-8", "limit": 10}
```

同目录及相应子目录中的测试可以请求它。

通常不需要手工 `from conftest import sample_settings`。

### 12.2 conftest 是普通模块名巧合吗？

它是 pytest 认识的特殊配置与局部插件入口文件名。

作用范围与目录结构有关。

你把夹具放在哪一层，会影响哪些测试能找到它。

因此，不建议为了方便把所有夹具都堆在项目根目录，让每个测试隐含获得一整套与自身无关的环境修改。

### 12.3 子目录可以覆盖同名夹具吗？

可以在相应范围提供更局部的定义。

这有助于不同测试目录定制资源。

但同名层层覆盖过多时，读者很难知道某个参数到底来自哪里。

夹具名称应该表达用途，覆盖应有明确原因，而不是把它当作隐藏全局变量系统。

### 12.4 autouse=True 为什么要克制？

自动使用夹具不需要测试显式写参数。

这对统一设置某些必要环境有帮助，但也会增加隐含行为。

一个看起来只检查字符串的测试，可能在背后启动服务器或修改环境变量，排错就不再直观。

优先显式请求与测试有关的资源。

只有确实需要统一执行，且范围清楚时，再使用 autouse。

### 12.5 夹具找不到，从哪里检查？

检查名字是否一致。

检查 conftest 所在层级与测试目录。

检查是否真的由当前 pytest 运行这份文件。

可以用：

```bash
python -m pytest --fixtures -q
```

查看可用夹具资料。

不要一遇到 fixture not found 就把系统路径到处 append；先定位发现范围和命名问题。

## 第十三章：tmp_path，把文件测试从真实目录里搬出来

### 13.1 为什么不直接写 D 盘上的测试文件？

如果测试写死某个真实路径，在另一台电脑上可能不存在。

重复运行可能覆盖旧数据。

多个测试还可能争用同一个文件。

tmp_path 为每个测试项提供独立的临时目录，返回 pathlib.Path 对象。

### 13.2 第一个文件测试

```python
def test_write_utf8_file(tmp_path):
    target = tmp_path / "notes.txt"
    target.write_text("你好，Python\n", encoding="utf-8")
    assert target.read_text(encoding="utf-8") == "你好，Python\n"
```

`tmp_path` 不是你自己定义的局部变量，也不需要先导入。

它是 pytest 提供的内置夹具。

你在参数中请求它，pytest 准备路径并传入。

### 13.3 它是不是字符串？

通常是 Path 对象。

所以可以使用 `/` 拼路径、mkdir、write_text 等方法。

如果某个老接口明确只接受字符串，可使用 str(target)。

但许多现代标准库接口已经支持 PathLike，无需一律提前转字符串。

### 13.4 临时目录会不会立刻删除？

不要把具体保留策略写成“测试结束瞬间一定清空”。

pytest 有自己的临时目录管理与历史保留行为，可能留下近期目录帮助排错。

你的测试应该依赖它提供独立可用位置，而不是依赖旧测试留下的目录。

敏感数据也不应仅因为目录叫 temporary 就被视为自动满足所有保留政策。

### 13.5 怎样测试目录创建？

```python
def ensure_report_directory(base):
    target = base / "reports" / "daily"
    target.mkdir(parents=True, exist_ok=True)
    return target


def test_report_directory_can_be_prepared_twice(tmp_path):
    first = ensure_report_directory(tmp_path)
    second = ensure_report_directory(tmp_path)
    assert first == second
    assert first.is_dir()
```

这项测试关注可重复调用及真实目录存在。

它没有只比较返回字符串，因此能发现实现忘记 mkdir 的问题。

### 13.6 怎样检查错误时旧文件没有被破坏？

先在临时目录写入一份旧内容。

调用会失败的更新操作。

捕获约定异常，再读取目标确认旧内容仍在。

这是很有价值的失败路径检查。

“成功能写文件”与“失败不毁掉旧文件”，是两项不同承诺。

后面完整实践会真正实现并测试这种行为。

### 13.7 tmp_path_factory 又是什么？

它适合显式创建临时目录，例如会话级只读数据准备。

```python
import pytest


@pytest.fixture(scope="session")
def large_reference_file(tmp_path_factory):
    folder = tmp_path_factory.mktemp("reference")
    target = folder / "codes.txt"
    target.write_text("PY\nSQL\nHTTP\n", encoding="utf-8")
    return target
```

这个例子数据很小，只是展示接口。

真实项目不要因为名字叫 large 就以为一定需要 session 共享；根据准备成本和可变性决定。

使用细节见[官方临时目录说明](https://docs.pytest.org/en/stable/how-to/tmp_path.html)。

## 第十四章：monkeypatch，怎样临时改变环境又自动恢复？

### 14.1 函数读取环境变量

```python
import os


def get_batch_size():
    return int(os.getenv("APP_BATCH_SIZE", "20"))
```

你希望测试默认值与用户配置。

直接修改自己终端的真实长期环境，不利于独立重复运行。

monkeypatch 可以在测试范围内临时改动，并在合适清理阶段恢复。

### 14.2 设置与删除环境变量

```python
def test_batch_size_from_environment(monkeypatch):
    monkeypatch.setenv("APP_BATCH_SIZE", "50")
    assert get_batch_size() == 50


def test_batch_size_uses_default(monkeypatch):
    monkeypatch.delenv("APP_BATCH_SIZE", raising=False)
    assert get_batch_size() == 20
```

为什么默认值测试要先删除变量？

因为运行测试的电脑可能本来就设置了它。

测试不应该依赖开发者此刻碰巧有哪些环境变量。

`raising=False` 允许变量本来就不存在，而不把这一点当作错误。

### 14.3 setenv 的值应该是什么？

环境变量值通常使用字符串。

你要模拟外部环境中的 `"50"`，而不是直接把 Python 整数塞给程序绕过真实来源。

测试输入应该尽可能符合真实边界的形态。

这与“为了方便直接注入内部结果”有所区别。

### 14.4 临时切换工作目录

```python
from pathlib import Path


def load_local_message():
    return Path("message.txt").read_text(encoding="utf-8")


def test_relative_file_uses_current_directory(tmp_path, monkeypatch):
    (tmp_path / "message.txt").write_text("hello", encoding="utf-8")
    monkeypatch.chdir(tmp_path)
    assert load_local_message() == "hello"
```

如果代码使用相对路径，这项测试能够明确控制基准目录。

monkeypatch 会参与恢复工作目录。

但工作目录是进程级状态，测试如果同时自己启动多个共享进程状态的线程，仍需理解并发影响。

### 14.5 临时修改字典

可以使用 setitem、delitem 等入口处理映射。

适合临时配置字典，而不是把修改散落在测试后手工恢复。

但如果你只改变嵌套列表内部内容，需要理解恢复针对的是哪一级绑定；不要假定所有深层对象都自动深拷贝。

依旧要关注引用与可变性。

### 14.6 setattr 在哪里替换？

它可以替换属性，例如把某模块中用于读取时间的函数换成固定版本。

关键不只是替换什么函数，而是被测代码到哪里查找这个名称。

下一章会专门通过两个文件解释这个最常见的 mock 错位问题。

### 14.7 如果只想在一小段里生效呢？

可以使用 monkeypatch.context() 建立更窄的修改范围。

尤其是修改标准库或 pytest 自己可能用到的功能时，范围越窄越容易控制影响。

不要为了测试一个 read_text 调用，就全局把 open 替换成永远报错，导致测试运行器自己的文件读取也出问题。

更局部地替换你自己的边界函数，往往更容易理解。

官方[monkeypatch 说明](https://docs.pytest.org/en/stable/how-to/monkeypatch.html)列出了具体入口与恢复行为。

## 第十五章：Mock 与 patch，为什么“我明明替换了”还是调用真的？

### 15.1 先把普通导入讲清楚

有两个文件。

`clock_source.py`：

```python
def current_timestamp():
    import time
    return time.time()
```

`report.py`：

```python
from clock_source import current_timestamp


def build_report_header():
    return {"generated_at": current_timestamp()}
```

report 导入时，把相应函数对象绑定到自己模块的 current_timestamp 名称。

之后调用时，查找的是 report 里的这个名称。

### 15.2 应该替换使用位置的名称

```python
import report


def test_report_header_uses_supplied_clock(monkeypatch):
    monkeypatch.setattr(report, "current_timestamp", lambda: 123.0)
    assert report.build_report_header() == {"generated_at": 123.0}
```

这里替换 report.current_timestamp。

如果你只修改 clock_source.current_timestamp，report 已经保存的绑定未必随之改变。

这就是常说的“在被查找的位置 patch”。

不是所有情况都只替换最初定义所在模块。

### 15.3 如果是 import clock_source 呢？

如果 report 使用模块导入，再调用 clock_source.current_timestamp，那么查找路径又不同。

你应该读实际导入写法和调用表达式，顺着名称找到查找位置。

不要把一条字符串 patch 路径背成通用公式。

这本质上仍然是 Python 的名称绑定知识。

### 15.4 Mock 对象有什么额外能力？

标准库 unittest.mock 提供 Mock、MagicMock、patch 等工具。

pytest 可以与它们配合，不要求你额外安装 pytest-mock 才能开始模拟。

例如：

```python
from unittest.mock import Mock


def notify_user(sender, address, message):
    sender(address, message)


def test_notification_passes_expected_arguments():
    sender = Mock()
    notify_user(sender, "alice@example.test", "任务完成")
    sender.assert_called_once_with("alice@example.test", "任务完成")
```

Mock 记录调用次数和参数。

这对通知这类“输出动作”有帮助，因为函数本身不一定返回可直接比较的结果。

### 15.5 return_value 与 side_effect

```python
from unittest.mock import Mock


def test_mock_return_and_failure():
    fetch = Mock(return_value={"id": 1})
    assert fetch() == {"id": 1}

    broken_fetch = Mock(side_effect=TimeoutError("读取超时"))
    with pytest.raises(TimeoutError):
        broken_fetch()
```

return_value 指定调用结果。

side_effect 可以模拟异常等行为。

但一个 mock 允许你写出什么响应，不代表真实服务真的拥有那样的协议。

模拟数据也需要基于真实契约设计。

### 15.6 spec 和 autospec 为什么有用？

过于宽松的 Mock 可能接受不存在的方法或错误参数。

spec、autospec 等机制可以帮助约束接口，减少测试替身与真实对象脱节。

但它们不是万能保证：动态属性、复杂描述器和外部协议仍要理解。

小函数可以优先用签名清楚的普通替身，再按需要使用更复杂的 Mock 功能。

### 15.7 为什么不要把所有内部函数都 mock 掉？

如果被测函数做三次调用，你把三次调用都替换成固定值，最后只验证相加，可能几乎没检查真实业务逻辑。

适合模拟的是慢、不稳定、具有副作用或明确属于外部边界的部分。

核心计算与数据变换尽量用真实实现运行。

替身越多，越要问自己：当前通过到底证明了哪一段真实行为？

## 第十六章：capsys，怎样检查 print 而不靠肉眼看控制台？

### 16.1 一个最简单的命令行输出函数

```python
import sys


def greet(name):
    if not name:
        print("名字不能为空", file=sys.stderr)
        return 2
    print(f"你好，{name}")
    return 0
```

它同时有返回状态与输出流两种可观察行为。

只检查 return 0，可能漏掉正常结果被错误输出到 stderr 的问题。

只检查文字，又可能漏掉退出状态错误。

### 16.2 读取捕获结果

```python
def test_greet_writes_stdout(capsys):
    status = greet("小林")
    captured = capsys.readouterr()
    assert status == 0
    assert captured.out == "你好，小林\n"
    assert captured.err == ""
```

capsys 是 pytest 的内置夹具。

readouterr 返回对象，里面有 out 和 err。

它们对应捕获到的标准输出和标准错误文本。

### 16.3 为什么预期里有换行？

普通 print 默认在末尾增加换行。

所以比较完整输出时，应该把这部分算进去。

如果一律 `.strip()`，可能隐藏多余空白、额外空行等输出格式错误。

是否忽略两侧空白，应按协议决定，不能只是为了让比较通过。

### 16.4 错误路径也要验证流向

```python
def test_empty_greeting_writes_stderr(capsys):
    status = greet("")
    captured = capsys.readouterr()
    assert status == 2
    assert captured.out == ""
    assert captured.err == "名字不能为空\n"
```

这项测试能发现错误提示混入正常数据流的情况。

对于标准输出要交给另一个程序解析的命令行工具，这一点非常重要。

例如 JSON 输出不能前面夹一行“开始运行”。

### 16.5 readouterr 是一直读取完整历史吗？

它会取得当前捕获内容，并使后续读取对应之后新产生的内容。

因此，可以在同一测试中分阶段调用，但应知道每次比较的是哪一段。

如果你第一次读取后丢掉了结果，第二次再期待全部历史，就可能误解行为。

### 16.6 capfd 与 capsys 有什么区别？

capsys 面向 Python 层标准流捕获的常见需求。

capfd 面向文件描述符层面的捕获，能涉及某些直接写底层描述符的输出。

具体选择取决于输出通过哪一层发生。

例如 C 扩展或某些子进程的底层输出，不一定适合只用 Python sys 流替换理解。

### 16.7 subprocess.run(capture_output=True) 呢？

子进程的输出已经被 subprocess 捕获到 CompletedProcess 中，就直接检查 result.stdout 与 result.stderr。

不要同时期待这些内容自然出现在当前进程的 capsys 里。

先画清楚“哪个进程写到哪里、由谁捕获”，会比反复添加 -s 更有效。

### 16.8 `-s` 是做什么？

```bash
python -m pytest -s
```

可以关闭通常的全局输出捕获，方便临时观察打印。

但如果测试本来使用捕获夹具，应按夹具机制理解其行为，不能简单认为加 -s 后所有捕获接口都无意义。

调试输出是观察工具，稳定测试仍应通过断言表达预期。

官方[输出捕获说明](https://docs.pytest.org/en/stable/how-to/capture-stdout-stderr.html)解释了不同层次与模式。

## 第十七章：caplog，日志怎么测才不依赖终端格式？

### 17.1 日志与 print 是不同通道

假设业务函数记录被跳过的行：

```python
import logging

logger = logging.getLogger("importer")


def accept_row(row):
    if not row.get("title"):
        logger.warning("跳过缺少标题的记录")
        return False
    return True
```

你希望失败时留下可定位日志。

用 capsys 不一定是最合适的方式，因为 logging 有自己的记录、处理器与格式机制。

### 17.2 设置需要捕获的级别

```python
import logging


def test_missing_title_is_logged(caplog):
    with caplog.at_level(logging.WARNING, logger="importer"):
        accepted = accept_row({"title": ""})
    assert accepted is False
    assert "跳过缺少标题的记录" in caplog.messages
```

at_level 在明确范围里设置相关捕获级别。

caplog.messages 提供消息内容列表。

这避免把时间戳、终端颜色和日志文件路径当作固定预期。

### 17.3 如果需要检查日志来源与级别？

```python
def test_warning_record_metadata(caplog):
    with caplog.at_level(logging.WARNING, logger="importer"):
        accept_row({})
    assert any(
        record.name == "importer" and record.levelno == logging.WARNING
        for record in caplog.records
    )
```

records 中是日志记录对象，可以检查 name、levelno 等资料。

比起在一整段格式化字符串里猜“WARNING”出现在哪，更贴近日志真实语义。

### 17.4 要精确断言一条日志吗？

如果协议就是“这个动作只应产生一次警告”，可以检查次数。

但如果测试范围还包含其他合理日志，不应无条件要求全部日志列表只有一项。

把关注范围缩到对应 logger 或筛选相关记录，更容易避免与无关调试日志耦合。

### 17.5 日志里有异常信息怎么检查？

logging.exception 或 exc_info=True 会把异常信息放进记录。

你可以检查相关记录的 exc_info 是否存在，或检查你自己保证的业务消息。

不要默认要求整段 traceback 的文件绝对路径和行号永久不变。

那些细节适合排错，不一定适合作为稳定业务契约。

### 17.6 为什么重新配置 root logger 后突然捕不到？

如果被测代码大幅替换全局日志处理器，可能影响 pytest 用于捕获的处理器。

应用的日志配置应该有清楚入口，不要每个业务函数都强行全局重置 logging。

测试日志时也应避免无必要地改动整个进程日志系统。

优先检查业务模块对应记录，而不是依赖终端最终看起来是什么样。

相关行为见[官方日志说明](https://docs.pytest.org/en/stable/how-to/logging.html)。

## 第十八章：warnings，不要把弃用提示永远压掉

### 18.1 警告不是普通异常

有的 API 仍能运行，但会通过 warnings 提醒未来变化。

例如：

```python
import warnings


def old_parser(text):
    warnings.warn("请改用新的解析入口", DeprecationWarning, stacklevel=2)
    return text.strip()
```

你希望兼容入口继续工作，同时发出弃用提醒。

### 18.2 pytest.warns

```python
import pytest


def test_old_parser_emits_deprecation_warning():
    with pytest.warns(DeprecationWarning, match="新的解析入口"):
        result = old_parser("  hello  ")
    assert result == "hello"
```

它检查指定范围出现符合预期的警告。

并不因为警告出现就表示功能失败。

这里返回行为和提醒行为都属于契约，因此分别断言。

### 18.3 不希望出现警告怎么办？

可以按项目政策把特定警告提升为错误，或者在局部捕获记录后检查。

不要一遇到大量警告就把所有 warnings 全局 ignore。

那可能把未来版本升级的重要信号一起隐藏。

需要过滤时，尽量限定警告类型、来源和原因，并给后续维护者留下解释。

### 18.4 为什么本机与 CI 警告数量不同？

依赖版本、环境、导入顺序和已注册的警告过滤规则都可能影响观察。

测试应对自己承诺的行为作明确范围控制。

如果真正关注某个兼容入口，直接在调用范围内测试它，不必把整个进程出现过的所有警告当作一份固定列表。

## 第十九章：skip 与 xfail，怎样表达“暂时不能通过”？

### 19.1 skip 表示当前不执行

例如某项测试只适用于 Windows：

```python
import sys
import pytest


@pytest.mark.skipif(sys.platform != "win32", reason="检查 Windows 专有路径行为")
def test_windows_specific_condition():
    assert sys.platform == "win32"
```

在其他平台跳过，有明确理由。

但这项示例只展示标记机制，没有真正覆盖某项 Windows 文件行为；真实项目应断言那份具体契约。

不要为了让报告好看，把普遍存在的功能错误都标记成 skip。

### 19.2 xfail 表示已知预期失败

它适合表达一项已知问题或尚未实现但已经写明的行为预期。

例如你希望修复前能保留回归样例。

```python
@pytest.mark.xfail(reason="已登记的旧格式解析问题", strict=True)
def test_known_legacy_bug():
    assert "legacy".upper() == "legacy"
```

这里故意构造失败，只用于说明机制。

strict=True 使意外通过成为需要处理的结果，提醒你移除过期标记或重新检查原因。

### 19.3 XPASS 为什么也值得关注？

原先预期失败的测试现在通过，可能说明 bug 已经修好。

也可能说明测试已经不再检查目标问题。

如果一直忽略 XPASS，xfail 就可能永久留在项目里，让真实状态越来越难判断。

显式采用 strict 政策有助于避免这类遗忘。

### 19.4 跳过不等于验证通过

报告里有十项 passed、五项 skipped，含义是十项执行通过，五项未执行。

不能对外汇报“十五项功能全部通过”。

同样，xfail 也不是普通成功的替代颜色。

验收时应分别说明运行范围和未覆盖原因。

### 19.5 缺少可选依赖时怎么办？

某些测试组可以使用 importorskip 等入口按约定跳过。

但必需依赖没装，不应该悄悄跳过所有核心测试后仍宣称系统可用。

是否允许跳过，要从项目安装与支持契约判断。

一项测试机制本身不替你决定验收标准。

## 第二十章：自定义标记与筛选，测试多了怎么只跑相关部分？

### 20.1 按名字筛选

```bash
python -m pytest -k "quantity and not negative"
```

-k 使用相应表达式根据测试名称等信息筛选。

它不是任意 Python 代码，也不是普通正则表达式接口。

筛选后注意汇总里 deselected 的数量，确认你真的执行了想检查的范围。

### 20.2 按标记筛选

可以给一组真实数据库集成测试加上 integration 标记：

```python
import pytest


@pytest.mark.integration
def test_database_round_trip():
    import sqlite3
    with sqlite3.connect(":memory:") as connection:
        assert connection.execute("SELECT 1").fetchone() == (1,)
```

选择：

```bash
python -m pytest -m integration
```

排除：

```bash
python -m pytest -m "not integration"
```

标记名表达组织政策，不改变测试函数实际行为。

### 20.3 自定义标记应该登记

可以在 pytest 配置中声明 marker 名称与用途。

这样团队读者知道 integration 代表什么，也能发现拼写错误。

否则 `integraiton` 这样的错误可能变成一组未按预期筛选的测试。

配合严格标记选项，可尽早报告未知标记问题。

### 20.4 直接传路径往往最明确

```bash
python -m pytest tests/test_config.py -q
```

如果你清楚改动范围，直接文件或节点标识比复杂筛选表达式容易复查。

但最终需要覆盖哪些邻近行为，仍应根据改动影响决定。

不能因为只跑一项比较快，就把它当作所有功能回归。

### 20.5 上次失败重跑

```bash
python -m pytest --lf
```

它利用缓存重新选择先前失败的项。

适合调试，但缓存是历史辅助信息。

修好以后，应运行与改动有关的正常范围，避免只盯着过去失败项，漏掉新引入的问题。

## 第二十一章：配置文件、项目布局与导入问题

### 21.1 为什么要有统一配置？

每个人都手工输入一长串参数，很容易不一致。

把稳定政策写进项目配置，运行方式更可复现。

例如测试目录、默认选项、自定义标记，都可以集中说明。

但不应该把所有可能选项都开上；每个默认政策都应有用途。

### 21.2 一个常见 pyproject.toml 配置

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers"
markers = [
  "integration: exercises several real local components together",
]
```

这里用 pytest 广泛采用的 ini_options 表达方式。

testpaths 指定默认收集入口。

addopts 提供默认命令选项。

markers 为自定义标记写说明。

如果项目已经有配置文件，不要随意再加一份互相竞争的配置；先确认当前识别了哪一份。

### 21.3 rootdir 是什么？

pytest 会确定一个根目录，用于相应配置、节点标识与缓存等用途。

它不意味着所有 Python 导入都被自动解决。

“rootdir 正确”与“模块能在当前解释器中导入”仍然是两件事。

遇到 ModuleNotFoundError 时，也要检查项目是否安装、启动位置、包结构和导入模式。

### 21.4 一个小项目与一个可安装包，做法可以不同

本文最初的平铺目录，方便从练习目录直接运行。

实际可安装包可以采用 src 布局，通过项目虚拟环境的可编辑安装等方式让测试导入真实包。

不要长期依赖测试文件里四处 `sys.path.append("../../...")` 解决结构问题。

那会使导入依赖启动位置，还可能导入错同名模块。

### 21.5 pytest 命令与 python -m pytest 有细微差别吗？

它们在多数日常用法中接近，但 Python 模块启动方式与当前目录导入处理仍可能带来差别。

因此，项目应固定一种清楚方式，并结合自己的打包布局验证。

学习时优先使用 `python -m pytest`，也便于确认解释器一致。

具体导入模式可查[官方导入机制说明](https://docs.pytest.org/en/stable/explanation/pythonpath.html)。

### 21.6 为什么不要给自己的文件取名 pytest.py？

你可能遮蔽真正的 pytest 包。

同理，json.py、httpx.py、typing.py 等名字也容易与常用模块冲突。

如果报错说某个属性不存在，先检查导入对象的 __file__，确认是否误导入了本地同名文件。

不要第一反应就卸载重装所有依赖。

### 21.7 配置能保证所有开发者执行完全一样吗？

它能减少差异，但依赖版本、系统、环境变量和插件仍可能不同。

锁定必要依赖、使用隔离环境、记录测试命令，才能更接近可复现结果。

测试报告应该同时说明范围与环境，而不只是抄一行 passed。

关于布局和测试实践，可结合[官方集成建议](https://docs.pytest.org/en/stable/explanation/goodpractices.html)阅读。

## 第二十二章：完整实践一，先明确一个笔记导出工具的契约

### 22.1 不从测试 API 开始，从真正工作开始

我们要写一个小工具。

输入是 JSON 笔记列表，每条笔记有正整数 id 和非空白 title。

输出是规范化后的 JSON 报告。

如果输入错误，不应破坏已有报告。

这同时涉及纯数据处理、真实文件读取和失败后的文件状态。

正好可以看见不同测试方式分别解决什么问题。

### 22.2 项目结构

```text
notes-export-demo/
├── notes_export.py
└── tests/
    └── test_notes_export.py
```

下面的业务代码按步骤放进 notes_export.py。

测试代码放进 tests/test_notes_export.py。

本文会逐段解释每个函数，不把一大段没有说明的程序丢到文末。

### 22.3 先定义自己的输入错误

```python
class NotesInputError(ValueError):
    pass
```

为什么不直接到处使用 Exception？

因为调用者需要区分可预期的用户输入错误与程序编写错误。

这个自定义类型仍然继承 ValueError，表达“输入值不符合约定”。

它并不自动决定错误怎样打印，显示政策留给命令行入口。

### 22.4 只处理内存数据的核心函数

```python
def normalize_notes(raw):
    if not isinstance(raw, list):
        raise NotesInputError("顶层必须是笔记列表")

    normalized = []
    seen_ids = set()
    for index, item in enumerate(raw):
        if not isinstance(item, dict):
            raise NotesInputError(f"第 {index + 1} 条笔记必须是对象")

        note_id = item.get("id")
        title = item.get("title")
        if type(note_id) is not int or note_id <= 0:
            raise NotesInputError(f"第 {index + 1} 条笔记编号必须是正整数")
        if not isinstance(title, str) or not title.strip():
            raise NotesInputError(f"第 {index + 1} 条笔记标题不能为空")
        if note_id in seen_ids:
            raise NotesInputError(f"笔记编号重复：{note_id}")

        seen_ids.add(note_id)
        normalized.append({"id": note_id, "title": title.strip()})
    return normalized
```

这里没有使用文件、网络或数据库。

所以可以直接用普通输入检查，无需 mock。

### 22.5 为什么 id 使用 type(...) is int？

Python 中 bool 是 int 的子类。

如果你只用 isinstance(True, int)，结果为真。

本协议要求真实整数编号，不接受 JSON 的 true 当作一号笔记。

因此，这里有意采用精确类型判断。

这是业务选择，不是所有整数校验都必须照抄的统一规则。

### 22.6 为什么返回新字典？

函数对标题做 strip，但没有原地修改调用者的原字典。

这样外部调用者仍能保留原始输入，用于对照、错误反馈或其他处理。

规范化后的结果只包含 id 和 title，额外输入字段按当前协议忽略。

如果你的配置要求陌生字段必须拒绝，应加上相应规则与测试，不能偷偷改变协议后只改测试预期。

### 22.7 第一项测试检查真实规范化

测试文件先导入：

```python
import json
import pytest
from notes_export import (
    NotesInputError,
    normalize_notes,
    read_notes,
    export_notes,
    main,
)
```

此时 read_notes、export_notes、main 还没写完，所以先继续补完整业务模块，再运行测试。

第一项测试是：

```python
def test_normalize_notes_strips_title_without_mutating_input():
    raw = [{"id": 1, "title": "  Python笔记  ", "internal": "ignored"}]
    result = normalize_notes(raw)
    assert result == [{"id": 1, "title": "Python笔记"}]
    assert raw[0]["title"] == "  Python笔记  "
```

同一项行为包含两份承诺：结果正确，原始输入不被改变。

如果你把实现改成直接修改 item["title"]，第二个断言应当发现。

这不是在镜像实现，而是在保护调用者可观察的契约。

### 22.8 错误输入选哪些才有意义？

```python
@pytest.mark.parametrize("raw", [
    {"id": 1, "title": "不是列表"},
    [None],
    [{"id": 0, "title": "无效编号"}],
    [{"id": True, "title": "布尔编号"}],
    [{"id": 1, "title": "   "}],
    [{"id": 1, "title": "A"}, {"id": 1, "title": "B"}],
])
def test_normalize_notes_rejects_invalid_contract(raw):
    with pytest.raises(NotesInputError):
        normalize_notes(raw)
```

每组对应不同契约：顶层形状、条目形状、编号范围、bool 边界、空白清理和集合重复。

并不是随便生成六个不相关值凑测试数量。

如果某条规则被删除，应该至少有一项失败。

### 22.9 空列表要不要拒绝？

当前约定允许导出空报告。

```python
def test_empty_note_list_is_valid():
    assert normalize_notes([]) == []
```

为什么单独写？

因为后续有人可能把“列表不能为空”当成理所当然加进去。

测试让当前需求明确可见。

你也可以选择相反政策，但应先确定需求，再修改实现与对应检查。

## 第二十三章：读取真实 JSON 文件，测试格式错误和大小限制

### 23.1 文件入口函数

继续放在 notes_export.py：

```python
import json
from pathlib import Path


def read_notes(path: Path, max_bytes: int = 1024 * 1024):
    if max_bytes < 0:
        raise ValueError("max_bytes 不能为负数")
    with path.open("rb") as stream:
        raw_bytes = stream.read(max_bytes + 1)
    if len(raw_bytes) > max_bytes:
        raise NotesInputError("输入文件过大")

    try:
        raw = json.loads(raw_bytes.decode("utf-8"))
    except (UnicodeDecodeError, json.JSONDecodeError) as error:
        raise NotesInputError("输入文件不是有效的 UTF-8 JSON") from error
    return normalize_notes(raw)
```

本例不把 OSError 统一包成输入内容错误。

文件不存在、权限不足等，由上层决定怎样显示。

格式错误转换成 NotesInputError，同时通过 from 保留底层原因。

### 23.2 为什么不是直接 Path.read_text？

read_text 对小文件很方便。

这里的协议额外要求限制读取量，因此先按字节最多读取上限加一。

不是说所有文件读取都必须这么写，而是当前需求确实包含大小上限。

测试也应该检查这项新增保证。

### 23.3 成功路径使用真正的临时文件

```python
def test_read_notes_from_utf8_file(tmp_path):
    source = tmp_path / "notes.json"
    source.write_text(
        json.dumps([{"id": 1, "title": "  中文标题  "}], ensure_ascii=False),
        encoding="utf-8",
    )
    assert read_notes(source) == [{"id": 1, "title": "中文标题"}]
```

没有 mock open。

因为临时目录中的小文件读写成本低，也正是这个函数的边界。

这项测试同时运行编码、文件读取、JSON 解析与业务规范化。

### 23.4 非 JSON 和非 UTF-8 分别试

```python
@pytest.mark.parametrize("content", [b"{broken", b"\xff\xfe"])
def test_read_notes_rejects_format_and_encoding_errors(tmp_path, content):
    source = tmp_path / "bad.json"
    source.write_bytes(content)
    with pytest.raises(NotesInputError, match="UTF-8 JSON"):
        read_notes(source)
```

两种输入失败原因不同，但按本工具对外约定映射到同一类格式错误。

如果你需要给用户区分编码与 JSON 语法，应改进错误类型或消息，并相应修改测试。

测试不能替你决定最终用户该看到多少技术细节。

### 23.5 大小限制要测边界

```python
def test_read_notes_accepts_exact_byte_limit(tmp_path):
    source = tmp_path / "empty.json"
    source.write_bytes(b"[]")
    assert read_notes(source, max_bytes=2) == []


def test_read_notes_rejects_one_byte_over_limit(tmp_path):
    source = tmp_path / "large.json"
    source.write_bytes(b"[] ")
    with pytest.raises(NotesInputError, match="过大"):
        read_notes(source, max_bytes=2)
```

两个文件在 JSON 语法上都可以表示空列表，尾部空格合法。

所以第二项失败确实来自大小规则，而不是因为我们恰好写了一份坏 JSON。

这就是设计边界样例时要避免“同时触发多个错误、却不知道到底测到了谁”。

### 23.6 文件不存在为什么不必 mock？

```python
def test_read_notes_preserves_missing_file_error(tmp_path):
    with pytest.raises(FileNotFoundError):
        read_notes(tmp_path / "missing.json")
```

独立临时目录中不存在这个名字，真实文件系统会提供预期错误。

这比把所有 Path.open 都替换成 mock 更直接，也能检验我们的异常转换是否意外吞掉了文件系统错误。

## 第二十四章：先写临时文件再替换，如何验证失败没有毁掉旧输出？

### 24.1 我们真正需要的承诺是什么？

成功时，目标文件包含完整新报告。

输入验证失败时，不改旧报告。

最终替换失败时，不改旧报告，并清理本次临时文件。

这些承诺都能通过临时目录里的可观察状态检查。

不必用口号说“原子安全”，而不说明具体保护了哪个阶段。

### 24.2 写入函数

继续放进 notes_export.py：

```python
import os
import tempfile


def write_report_atomically(notes, target: Path):
    target.parent.mkdir(parents=True, exist_ok=True)
    temporary_path = None
    try:
        with tempfile.NamedTemporaryFile(
            mode="w", encoding="utf-8", newline="\n",
            dir=target.parent, prefix=f".{target.name}.", suffix=".tmp",
            delete=False,
        ) as stream:
            temporary_path = Path(stream.name)
            json.dump(notes, stream, ensure_ascii=False, indent=2)
            stream.write("\n")
        os.replace(temporary_path, target)
    finally:
        if temporary_path is not None:
            temporary_path.unlink(missing_ok=True)
```

临时文件建在目标所在目录，便于保持同一文件系统的替换条件。

使用唯一名称，避免多次普通调用都争抢同一个固定 `.tmp`。

先离开 with 关闭文件，再替换，也更符合跨平台文件打开语义。

### 24.3 finally 为什么在成功时也执行？

finally 本来就会在正常异常展开中执行。

替换成功后，原临时路径通常已经不存在。

unlink(missing_ok=True) 允许它已经不存在。

替换失败时，临时文件仍可能存在，则尝试删除。

清理本身仍可能因为文件系统错误失败；本例不把它描述为所有故障下的绝对保证。

### 24.4 这是不是断电持久性保证？

不是。

同一文件系统支持相应语义时，替换有助于避免读者看到部分写入目标。

但断电后的持久性还涉及 fsync、目录元数据、存储系统等。

本例关注普通工具的完整写入和失败保护，不声称覆盖机器突然断电、恶意目录竞争或多人协作数据库一致性。

测试范围也应按这个边界汇报。

### 24.5 合并读入与输出

```python
def export_notes(source: Path, target: Path):
    notes = read_notes(source)
    write_report_atomically(notes, target)
    return len(notes)
```

先完整读取并验证，再写报告。

输入失败时，还没开始更新目标。

顺序本身就是一项重要设计：不要先截断旧文件，再发现输入根本不合法。

### 24.6 成功更新测试

```python
def test_export_creates_normalized_report(tmp_path):
    source = tmp_path / "source.json"
    target = tmp_path / "reports" / "result.json"
    source.write_text('[{"id":1,"title":"  Python  "}]', encoding="utf-8")
    assert export_notes(source, target) == 1
    assert json.loads(target.read_text(encoding="utf-8")) == [
        {"id": 1, "title": "Python"},
    ]
    assert list(target.parent.glob(f".{target.name}.*.tmp")) == []
```

测试检查真正输出内容，而不只是 os.replace 被调用一次。

同时检查本次临时文件没有残留。

返回数量也被验证，因为它是函数对调用者提供的信息。

### 24.7 输入错误时保留旧报告

```python
def test_invalid_input_keeps_existing_report(tmp_path):
    source = tmp_path / "source.json"
    target = tmp_path / "result.json"
    source.write_text('[{"id":0,"title":"bad"}]', encoding="utf-8")
    target.write_bytes(b"old-report")
    with pytest.raises(NotesInputError):
        export_notes(source, target)
    assert target.read_bytes() == b"old-report"
```

这里不需要模拟任何东西。

真实坏输入足以触发失败路径。

如果实现先打开 target 的 w 模式再验证，旧内容会被截断，这项测试就能发现。

### 24.8 替换失败怎么稳定重现？

用真实磁盘权限模拟，在不同系统和管理员权限下可能不稳定。

这里把外部替换动作局部换成明确失败的函数。

```python
import notes_export


def test_replace_failure_keeps_old_report_and_cleans_temp(tmp_path, monkeypatch):
    source = tmp_path / "source.json"
    target = tmp_path / "result.json"
    source.write_text('[{"id":1,"title":"new"}]', encoding="utf-8")
    target.write_bytes(b"old-report")

    def fail_replace(source_path, target_path):
        raise PermissionError("模拟目标不可替换")

    with monkeypatch.context() as patch:
        patch.setattr(notes_export.os, "replace", fail_replace)
        with pytest.raises(PermissionError, match="不可替换"):
            export_notes(source, target)

    assert target.read_bytes() == b"old-report"
    assert list(tmp_path.glob(f".{target.name}.*.tmp")) == []
```

模拟范围很窄，真实写临时文件与 finally 清理仍然执行。

退出 context 后，replace 已恢复，再检查其他文件状态。

### 24.9 这项模拟测试证明了什么？

它证明当替换动作抛出这个错误时，当前代码会保留旧报告并执行相应清理。

它不证明所有操作系统的权限错误都能被提前检测，也不证明真实 os.replace 在每一种网络文件系统上具备完全相同语义。

测试的结论要紧贴实际执行的边界。

这比简单说“异常处理全部测试通过，因此绝对不会丢数据”更准确。

## 第二十五章：命令行入口，既测函数，也测真的进程

### 25.1 为什么 main 最好能接受参数列表？

如果 main 只能直接读 sys.argv，测试需要临时修改进程状态。

让 main(argv=None) 接受可注入参数，既符合 argparse 常用方式，也方便测试。

普通命令行运行时不传 argv，仍由解析器读取实际启动参数。

### 25.2 补上入口代码

继续放入 notes_export.py：

```python
import argparse
import sys


def main(argv=None):
    parser = argparse.ArgumentParser(description="规范化并导出笔记 JSON")
    parser.add_argument("source", type=Path)
    parser.add_argument("target", type=Path)
    args = parser.parse_args(argv)

    try:
        count = export_notes(args.source, args.target)
    except (NotesInputError, OSError) as error:
        print(f"导出失败：{error}", file=sys.stderr)
        return 2

    print(json.dumps({"count": count}, ensure_ascii=False))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

成功时 stdout 输出一行 JSON。

错误时 stderr 输出说明，返回二。

这给终端用户与调用它的其他程序提供明确结果。

### 25.3 进程内检查成功输出

```python
def test_main_success_returns_machine_readable_json(tmp_path, capsys):
    source = tmp_path / "input.json"
    target = tmp_path / "output.json"
    source.write_text("[]", encoding="utf-8")
    status = main([str(source), str(target)])
    captured = capsys.readouterr()
    assert status == 0
    assert json.loads(captured.out) == {"count": 0}
    assert captured.err == ""
```

我们解析输出 JSON，而不是把无关空格缩进当作固定格式。

但明确要求 stderr 为空，避免成功输出混入异常提示。

### 25.4 进程内检查输入失败

```python
def test_main_invalid_input_reports_error_to_stderr(tmp_path, capsys):
    source = tmp_path / "bad.json"
    target = tmp_path / "out.json"
    source.write_text("not json", encoding="utf-8")
    status = main([str(source), str(target)])
    captured = capsys.readouterr()
    assert status == 2
    assert captured.out == ""
    assert "导出失败" in captured.err
    assert not target.exists()
```

这不只是测试一行 print。

它真实运行读取与错误转换，并确认目标未被创建。

### 25.5 参数缺失会返回二，还是抛出 SystemExit？

argparse 默认会对参数使用错误发起 SystemExit。

这个动作发生在本例的业务 try 前面。

所以可以写：

```python
def test_main_missing_arguments_uses_argparse_exit(capsys):
    with pytest.raises(SystemExit) as captured:
        main([])
    assert captured.value.code == 2
    assert "usage:" in capsys.readouterr().err.lower()
```

不要假定所有失败都通过 main 的 return 2 返回。

真实控制流需要读具体代码。

### 25.6 为什么还要用 subprocess 跑一次？

进程内调用检查函数行为，但没有真的经过解释器启动、文件入口和进程退出状态。

可以补一项完整命令行检查。

```python
import subprocess
import sys
from pathlib import Path
import notes_export


def test_cli_process_round_trip(tmp_path):
    source = tmp_path / "input.json"
    target = tmp_path / "output.json"
    source.write_text("[]", encoding="utf-8")
    script = Path(notes_export.__file__).resolve()

    result = subprocess.run(
        [sys.executable, str(script), str(source), str(target)],
        capture_output=True, text=True, encoding="utf-8", timeout=10,
    )
    assert result.returncode == 0, result.stderr
    assert json.loads(result.stdout) == {"count": 0}
    assert target.exists()
```

使用 sys.executable，保证子进程使用当前测试环境的解释器。

使用参数列表，不拼接 shell 命令字符串。

timeout 防止测试无限等待，但不应把它当作性能要求“必须十秒内完成一切复杂系统工作”。

### 25.7 进程内和进程外测试各要多少？

大部分分支可以在进程内快速检查。

少量完整进程测试确认入口与输出协议接得起来。

没必要对每个数字都重新启动一次解释器，也不能永远只测内部函数就宣称命令行已验证。

按边界分配测试，通常比所有案例都走最重路径更有效。

## 第二十六章：SQLite 测试，用真实小数据库检查数据行为

### 26.1 数据库 API 为什么不一定都要 mock？

SQLite 可以在临时文件或内存中运行。

对一个简单仓储函数，使用真实 SQLite 能检查 SQL、参数绑定、唯一约束和事务行为。

如果只 mock execute 返回固定内容，SQL 拼错也可能照样通过。

我们先用真实本地数据库。

### 26.2 业务模块 library_store.py

创建文件：

```python
import sqlite3


def create_schema(connection):
    connection.execute(
        "CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT NOT NULL)"
    )
    connection.commit()


def add_book(connection, book_id, title):
    with connection:
        connection.execute(
            "INSERT INTO books (id, title) VALUES (?, ?)",
            (book_id, title),
        )


def list_books(connection):
    return connection.execute(
        "SELECT id, title FROM books ORDER BY id"
    ).fetchall()
```

参数使用问号占位，而不是把标题拼入 SQL。

list_books 明确 ORDER BY，因此顺序属于当前返回契约。

### 26.3 夹具负责创建与关闭

放在测试文件或该目录的 conftest.py：

```python
import sqlite3
import pytest
from library_store import create_schema


@pytest.fixture
def book_database(tmp_path):
    path = tmp_path / "library.sqlite3"
    connection = sqlite3.connect(path)
    try:
        create_schema(connection)
        yield connection
    finally:
        connection.close()
```

每个测试有独立数据库文件。

你可以在失败后观察实际文件，理解表结构与内容。

### 26.4 `with connection` 会关闭连接吗？

sqlite3 连接上下文主要管理相应事务提交或回滚。

不应把它理解成通用的退出时关闭连接。

所以夹具中仍然显式 connection.close()。

这与普通文件 with 的关闭行为不同，学习时要按具体对象协议理解，不能只看语法外形相似。

### 26.5 检查真正写入与读取

```python
from library_store import add_book, list_books


def test_books_round_trip_in_id_order(book_database):
    add_book(book_database, 2, "SQL")
    add_book(book_database, 1, "Python")
    assert list_books(book_database) == [(1, "Python"), (2, "SQL")]
```

先插二，再插一。

这样能发现函数只依赖偶然插入顺序，而没有保证排序的问题。

测试样例的顺序是有意设计的。

### 26.6 重复主键后，旧记录应保留

```python
def test_duplicate_id_does_not_replace_existing_book(book_database):
    add_book(book_database, 1, "原书名")
    with pytest.raises(sqlite3.IntegrityError):
        add_book(book_database, 1, "新书名")
    assert list_books(book_database) == [(1, "原书名")]
```

这里既检查数据库约束错误，也检查失败后的状态。

如果有人把 INSERT 改成 INSERT OR REPLACE，可能不再抛异常，也可能覆盖旧内容，测试就会揭示契约变化。

### 26.7 标题包含引号会怎样？

```python
def test_title_is_bound_as_data(book_database):
    title = "Alice's Python; DROP TABLE books;"
    add_book(book_database, 1, title)
    assert list_books(book_database) == [(1, title)]
```

标题被作为参数数据保存，不应被当作 SQL 程序执行。

这项检查不能证明所有输入安全政策都完备，但能保护当前函数的参数绑定契约。

### 26.8 内存库与文件库怎样选？

内存库适合快速、连接内的数据库逻辑检查。

临时文件适合需要测试重开连接、文件位置或持久结果的场景。

注意普通 `:memory:` 数据库通常属于相应连接，不是所有连接自动共用一个全局内存数据库。

如果你换连接后数据消失，先理解连接与数据库生命周期。

### 26.9 SQLite 通过能证明 PostgreSQL 也一样吗？

不能。

SQL 方言、类型、并发、锁和事务行为都有差别。

如果正式系统使用另一种数据库，关键数据库行为需要对相应数据库版本执行适当集成测试。

本篇的 SQLite 示例只声称验证真实 SQLite 的这份小契约。

## 第二十七章：HTTP 请求测试，使用 MockTransport 保留真实客户端逻辑

### 27.1 为什么不让测试总访问公网？

公网服务会变、会限流、会断网，也可能需要凭据。

一项单元或本地集成测试，不应因为网络此刻不可达就随机失败。

但如果把整个 HTTP 客户端全部替换成一个返回字典的 Mock，又可能漏掉路径、参数、状态与 JSON 处理错误。

HTTPX 的 MockTransport 可以在传输边界提供受控响应，保留上层客户端调用。

### 27.2 业务函数 book_client.py

```python
import httpx


def fetch_book_titles(client: httpx.Client, page: int):
    if page < 1:
        raise ValueError("页号必须从一开始")
    response = client.get("/books", params={"page": page})
    response.raise_for_status()
    payload = response.json()
    return [item["title"] for item in payload["items"]]
```

这个例子只演示请求路径、状态与简单返回提取。

真实业务结构严格校验，可以再结合 [Pydantic 教程](Python-Pydantic库.md)。

不要把这里的简单列表推导当作所有外部 JSON 都已完整验证。

### 27.3 用 handler 检查真实构造的请求

```python
import httpx
from book_client import fetch_book_titles


def test_http_client_sends_path_and_query():
    seen_requests = []

    def handler(request):
        seen_requests.append(request)
        return httpx.Response(200, json={
            "items": [{"title": "Python"}, {"title": "SQL"}],
        })

    with httpx.Client(
        base_url="https://books.example",
        transport=httpx.MockTransport(handler),
    ) as client:
        assert fetch_book_titles(client, 2) == ["Python", "SQL"]

    assert len(seen_requests) == 1
    assert seen_requests[0].method == "GET"
    assert seen_requests[0].url.path == "/books"
    assert seen_requests[0].url.params["page"] == "2"
```

page 原来是整数，进入 URL 查询参数后成为相应文本表示。

这项测试能发现路径写错、漏传页号、方法变错等问题。

没有实际访问 example 域名。

### 27.4 HTTP 错误应先于业务提取

```python
def test_http_status_error_is_preserved():
    transport = httpx.MockTransport(
        lambda request: httpx.Response(503, json={"error": "unavailable"})
    )
    with httpx.Client(base_url="https://books.example", transport=transport) as client:
        with pytest.raises(httpx.HTTPStatusError):
            fetch_book_titles(client, 1)
```

如果实现忘记 raise_for_status，可能在读取 items 时变成 KeyError。

测试预期 HTTPStatusError，可以发现这种错误层次变化。

### 27.5 模拟传输超时

```python
def test_http_transport_timeout_is_preserved():
    def handler(request):
        raise httpx.ReadTimeout("模拟读取超时", request=request)

    with httpx.Client(
        base_url="https://books.example",
        transport=httpx.MockTransport(handler),
    ) as client:
        with pytest.raises(httpx.ReadTimeout):
            fetch_book_titles(client, 1)
```

这验证调用者面对传输异常时没有被函数错误吞掉。

它不证明实际 TCP 连接在某个秒数后真的超时。

MockTransport 不需要真实网络，也不自动重现完整 socket 超时机制。

### 27.6 页号错误应该完全不发请求

```python
def test_invalid_page_is_rejected_before_transport():
    calls = []
    transport = httpx.MockTransport(
        lambda request: calls.append(request) or httpx.Response(200, json={"items": []})
    )
    with httpx.Client(base_url="https://books.example", transport=transport) as client:
        with pytest.raises(ValueError, match="从一开始"):
            fetch_book_titles(client, 0)
    assert calls == []
```

这项测试不仅要求错误，还要求没有外部调用。

“结果失败”与“失败前没有产生不必要副作用”，是不同保证。

### 27.7 这算单元测试还是集成测试？

名称因团队边界定义不同而有所差异。

更实际的说明是：运行真实 HTTPX 客户端和业务提取逻辑，用进程内传输替代真实网络。

这样读者知道哪些部分真实执行，哪些部分受控模拟。

比只贴一个 unit 或 integration 标签更准确。

## 第二十八章：异步函数怎么测？先分清 pytest 自带什么

### 28.1 async def 直接调用不会完成业务

```python
async def double_later(value):
    return value * 2
```

普通调用得到协程对象，而不是已经算好的结果。

测试必须让协程在合适事件循环中运行。

不能写 `assert double_later(3) == 6`，然后以为是 pytest 不支持普通断言。

这里首先是异步 Python 的执行模型问题。

### 28.2 少量独立协程可以明确运行

```python
import asyncio


def test_simple_coroutine_with_explicit_runner():
    result = asyncio.run(double_later(3))
    assert result == 6
```

这是一个普通同步测试函数，内部明确使用 asyncio.run。

适合独立小场景。

如果当前线程已经在运行事件循环，就不能再随意嵌套 asyncio.run；项目中的异步测试环境需要统一设计。

### 28.3 大量异步测试通常需要插件

pytest-asyncio、AnyIO 等生态工具能提供专门的异步测试支持。

它们不是“只安装 pytest 就自动拥有的所有能力”。

项目选择哪个，应结合现有异步库、事件循环生命周期和版本约定。

不要把不同插件的标记、模式和夹具规则混成一套。

本篇不要求额外安装异步测试插件，核心实践全部可以用当前隔离环境运行。

### 28.4 客户端与事件循环生命周期要配合

异步客户端可能绑定相关异步资源。

不要在一次 asyncio.run 创建资源，又在另一次全新的事件循环中继续使用它，最后出现已关闭循环等问题。

测试夹具作用域也要与插件的循环作用域匹配。

如果不清楚，先让资源在同一明确的异步入口中创建、使用和关闭，再考虑跨测试共享。

### 28.5 并发测试不要只靠很短的 sleep

想验证两个任务协作，可以用 Event、Queue 等同步机制控制关键时点。

只 sleep 零点零一秒后假定另一个任务一定执行到了某行，容易受到机器负载影响。

测试应该等待一个明确事件，而不是猜测操作系统调度速度。

并发测试需要超时来防止无限挂起，同时给出有意义的失败说明。

相关语言基础可以结合 [异步、协程与事件循环教程](Python-异步、协程、事件循环与线程.md)。

## 第二十九章：随机、时间和顺序，为什么有的测试今天过明天不过？

### 29.1 不稳定测试会削弱整个套件的价值

一项测试偶尔失败、重跑又过，团队逐渐习惯忽略红色。

之后真正的回归也可能被当作“再跑一次就好”。

所以测试自身的稳定性值得维护。

但稳定不是把重试次数无限加大，而是找出未受控依赖。

### 29.2 当前日期不要散落在预期中

如果函数根据今天生成文件名，测试直接拿真实当前时间比较，跨午夜时可能出现两次读取属于不同日期。

更清楚的设计是把“今天”作为参数，或注入一个时钟函数。

```python
from datetime import date


def daily_filename(today: date):
    return f"report-{today.isoformat()}.json"


def test_daily_filename_for_known_date():
    assert daily_filename(date(2026, 11, 3)) == "report-2026-11-03.json"
```

这里无需模拟整个 datetime 模块。

因为真正需要变化的事实只是日期。

### 29.3 随机输入也可以受控

可以使用独立 Random 实例和固定种子来复现实验。

但“固定种子”不等于所有 Python 版本下所有随机相关算法表现永久不变。

如果测试关注的是范围、不重复或格式，就应该断言这些性质，而不是无必要地锁死一长串随机结果。

生成安全令牌也不应为了测试方便改用不适合的随机算法。

### 29.4 文件系统顺序不要当成排序保证

listdir、glob 等结果顺序可能受平台与文件系统影响。

如果需求不关心顺序，就按集合或排序后比较。

如果需求明确要求排序，业务函数应该排序，测试也应检查排序。

不要由测试偷偷排序后掩盖实现没有排序的问题。

### 29.5 测试之间不能依赖共享残留

一个测试创建全局用户，另一个假定它已经存在，这样单独运行第二项就失败。

夹具应明确提供每项测试所需状态。

如果确实要检查多步业务流程，可以把它们组织为一项有清楚阶段的场景测试。

不要用测试函数名称的字母顺序表达业务执行顺序。

### 29.6 网络端口怎样避免冲突？

本地服务器测试可以让系统分配空闲端口，例如监听端口零，再读取实际地址。

不要所有测试都写死八千端口，要求用户关闭正在运行的项目。

但“先探测一个空闲端口，关闭探测 socket，再启动服务器”之间也有竞争窗口。

直接保留真正绑定的服务器资源更清楚。

本篇 HTTP 练习使用进程内传输，因此不需要占用网络端口。

### 29.7 测试依赖本机语言和时区吗？

错误文案、日期格式和本地时间可能受环境影响。

如果协议要求 UTC 或固定格式，业务代码与测试都应明确表达。

如果测试只是为了检查内容含义，不必把操作系统本地化消息整句锁死。

把真正的协议与偶然显示细节区分开，测试会更稳定。

### 29.8 flaky 不是一个可以永久贴着的标签

可以临时记录问题与复现信息，但需要继续定位原因。

盲目给失败测试加重试，可能把真实竞争条件藏起来。

尤其是写入、副作用和幂等问题，重跑本身还可能改变状态。

对外汇报时，重试后通过应如实说明，不要把最初失败从证据中抹掉。

## 第三十章：测试设计，怎样避免“测试很多，却没有保护作用”？

### 30.1 从可观察契约分组

一个解析函数可能有正常转换、范围拒绝、格式拒绝三类契约。

一个文件更新函数还包括成功输出、输入失败保持旧值、写入失败清理等状态契约。

先列出这些行为，再决定使用多少案例。

不是每个内部 if 都机械对应一项，也不是只挑一条成功输入。

### 30.2 正常值、边界值与无效值

正整数范围一到五百，可以选择一、五百、零、五百零一，以及一个普通中间值。

列表可以选择空、一项、多项和重复项。

可空字段可以分别检查省略、None、空字符串和正常内容。

这些划分来自行为差异，不是所有函数都要测试一套相同数据矩阵。

### 30.3 一个测试只写一个 assert 吗？

不必。

同一行为的多个相关结果可以放在同一测试里。

例如导出失败后，异常、旧文件与临时文件清理共同描述这一失败处理。

但如果一个测试同时检查登录、报告生成和邮件通知三个独立流程，失败后只知道前面某一步中断，可能应该拆分。

“一个测试一个主题”比“一个测试只能一个 assert”更实用。

### 30.4 测试名称应该描述结果

`test_parse` 太宽。

`test_negative_quantity_is_rejected` 能说明输入方向和预期。

`test_replace_failure_keeps_old_report` 能说明为什么这个测试存在。

名称不必写成一整段小说，但应让维护者在报告里看出业务含义。

### 30.5 安排、执行、断言怎么分？

常见结构是先准备输入与资源，再调用目标，最后比较结果。

不一定每个测试都写三个注释标题。

短测试通过空行就能清楚区分。

如果需要大量注释才能看懂执行目标，可能是准备逻辑过于复杂，或一次测试承担了过多职责。

### 30.6 不要重新实现一遍同一算法当预期

如果实现是按某公式计算，你在测试里复制同样公式，两边可能有相同错误。

可以选人工容易核对的小输入，写明确结果，或用独立性质与可信参考作比较。

例如排序结果除了等于已知列表，还可以检查元素守恒；但不要用同一个被测排序函数生成 expected。

独立性是预期可信的重要来源。

### 30.7 回归测试应该复现真实 bug

修复一个“全空白标题被接受”的问题，应留下全空白输入的失败样例。

先确认旧实现下能暴露问题，再确认修复后通过。

如果测试在旧实现上也通过，可能没有触发原来的错误条件。

回归测试的意义是防止同类问题回来，而不只是给修复提交增加一个新文件。

### 30.8 测试内部实现细节有什么代价？

如果只要你把一个内部辅助函数改名，几十项测试就失败，而公开行为完全没变，测试可能与实现耦合过紧。

优先检查返回值、输出、状态与对外动作。

但某些性能或副作用契约确实涉及调用次数，例如不应重复发送通知；这时检查调用是合理的。

关键是调用次数本身是否属于重要行为，而不是“Mock 很方便所以都 assert_called”。

### 30.9 覆盖率百分之百能证明没有 bug 吗？

不能。

覆盖率告诉你哪些代码位置在测量中被执行，以及工具所统计的分支等情况。

执行到一行，不代表比较了正确预期，也不代表所有组合和业务含义都检查过。

pytest-cov 等是生态插件，不是本篇默认要求的核心依赖。

如果项目使用覆盖率，把它当作发现未触及区域的线索，而不是唯一质量分数。

### 30.10 不应该为了可测试性把设计改得很奇怪

适当注入时钟、HTTP 客户端、路径和配置，有助于让边界明确。

但不需要给一个简单加法函数套五层接口和工厂。

测试与实现都应服务实际需求。

如果替身数量和准备成本远超业务本身，先检查是否选错了测试边界。

## 第三十一章：运行与排错，怎样让每次验证都有可复查范围？

### 31.1 先收集，再执行

新项目可以先运行 collect-only 确认测试数量和路径。

然后执行对应文件。

遇到错误先读阶段：收集、准备、调用还是清理。

这比反复安装插件、修改 sys.path 和加忽略选项更有方向。

### 31.2 失败时保留能定位问题的信息

命令、解释器版本、依赖版本、失败节点与关键 traceback 都有价值。

但报告中不要包含凭据、真实用户数据和不必要环境变量。

需要交给他人排查时，最好提供最小复现输入与预期，而不只是截图一行红字。

### 31.3 不要吞掉 pytest 的退出码

自动化脚本里，如果执行测试后又执行一个成功命令，最后只检查末尾状态，可能错过测试失败。

运行器应正确传播 pytest 的非零状态。

测试输出被重定向到文件，也不代表测试已经成功。

退出状态与报告内容要一起看。

### 31.4 插件会改变测试环境

某些插件会提供夹具、自动标记或运行策略。

如果本机通过、干净环境失败，检查当前加载插件与项目依赖约定。

不要依靠自己电脑偶然装过的全局插件完成核心测试。

隔离环境中显式声明依赖更容易复现。

### 31.5 重复运行是在验证什么？

一次修复后重跑相关测试，是确认当前行为。

多次运行可以帮助发现部分不稳定问题，但十次都过也不能证明并发竞争绝不存在。

没有新改动、失败或未解疑问时，也不必为了显得认真无限重复同一组检查。

把精力放在尚未验证的边界更有价值。

### 31.6 并行执行不是核心命令默认就有

pytest-xdist 等插件可以提供多进程并行测试。

但共享端口、文件、数据库和全局服务可能因此暴露更多冲突。

先让每项测试资源独立，再考虑并行加速。

不能把执行变快当成自动解决隔离问题。

本篇练习按普通顺序运行即可，不需要安装并行插件。

## 第三十二章：练习与参考答案，把知识变成自己的检查

### 32.1 练习一，修复一个永远绿灯的异常测试

现有测试是：

```python
def test_rejection_incomplete():
    try:
        parse_quantity("-1")
    except ValueError:
        pass
```

如果 parse_quantity 错误地返回负数，这项测试仍然通过。

请修复，并检查预期错误消息。

参考答案：

```python
def test_negative_quantity_requires_error():
    with pytest.raises(ValueError, match="数量不能为负数"):
        parse_quantity("-1")
```

raises 同时要求“必须发生异常”和“异常符合预期”。

普通 try 版本如果需要等效效果，至少还要在 else 中主动让测试失败。

### 32.2 练习二，零到底是否属于合法范围？

为 parse_quantity 写参数化测试：零、一、带两侧空格的十二。

参考答案：

```python
@pytest.mark.parametrize("text, expected", [
    ("0", 0),
    ("1", 1),
    (" 12 ", 12),
])
def test_quantity_acceptance_boundary(text, expected):
    assert parse_quantity(text) == expected
```

然后把实现中的 `< 0` 暂时改成 `<= 0`。

零那组应该失败。

观察失败后再恢复实现，这能确认测试确实保护了边界。

### 32.3 练习三，列表顺序与重复次数

函数返回 `["b", "a", "a"]`。

需求只说顺序不重要，但重复次数必须保留。

你应该使用 set 比较吗？

不应该。

参考检查：

```python
from collections import Counter


def test_items_ignore_order_but_keep_multiplicity():
    actual = ["b", "a", "a"]
    assert Counter(actual) == Counter(["a", "a", "b"])
```

set 会把两次 a 合并，无法发现遗漏一次重复。

这道题强调比较方式必须符合数据含义。

### 32.4 练习四，环境变量默认值怎么可靠测试？

函数 get_batch_size 从 APP_BATCH_SIZE 读取，缺失时默认二十。

你的电脑可能已经设置这个变量。

参考答案：

```python
def test_default_batch_size_is_environment_independent(monkeypatch):
    monkeypatch.delenv("APP_BATCH_SIZE", raising=False)
    assert get_batch_size() == 20
```

不需要告诉用户先手工删除终端变量。

测试应自己准备需要的前提，并参与恢复。

### 32.5 练习五，为什么 patch 了定义模块仍没效果？

report 使用 `from clock_source import current_timestamp`。

你修改 clock_source.current_timestamp 后，report 仍调用旧函数。

请解释并修复。

参考答案：report 已经保存了相应名称绑定，调用时查找 report.current_timestamp。

因此，测试替换 report 模块中的名称。

如果导入写法改成模块导入再访问属性，则应按新的查找路径重新判断。

不要只背固定 patch 字符串。

### 32.6 练习六，文件错误时旧内容还在吗？

使用 tmp_path 创建旧报告和非法输入。

调用 export_notes，要求 NotesInputError，然后比较旧字节内容。

参考结构：

```python
def test_invalid_input_never_truncates_old_target(tmp_path):
    source = tmp_path / "input.json"
    target = tmp_path / "output.json"
    source.write_text("{}", encoding="utf-8")
    target.write_bytes(b"keep-me")
    with pytest.raises(NotesInputError):
        export_notes(source, target)
    assert target.read_bytes() == b"keep-me"
```

顶层对象不是本协议允许的列表，所以失败来自业务契约。

不要把目标放在真实笔记目录里测试。

### 32.7 练习七，SQLite 失败后还要看什么？

重复插入编号时，除了要求 IntegrityError，还要确认原记录没有被覆盖。

参考思路是先插入旧标题，再尝试重复编号，最后查询全部记录。

如果你只 mock execute 抛出错误，就没有真实检查数据库最终内容。

对这种便宜的小数据库，真实临时库往往更适合这项需求。

### 32.8 练习八，stdout 是给机器读的

命令行成功时输出 JSON，错误提示应写 stderr。

请在成功测试中解析 stdout，检查 stderr 为空。

参考答案就是前面的 test_main_success_returns_machine_readable_json。

不要只判断 stdout 中出现 count 字样；一大段混乱文本也可能包含它，却不是有效 JSON。

让真正的 JSON 解析器检查输出协议，比肉眼搜索关键字更可靠。

### 32.9 练习九，HTTP 500 应该是什么错误？

MockTransport 返回五百状态与 `{"error":"bad"}`。

要求函数先产生 HTTPStatusError，而不是 KeyError。

参考方式是构造响应后使用 pytest.raises(httpx.HTTPStatusError) 包住目标调用。

这能防止实现删除 raise_for_status 后，错误在更深一层变得误导。

### 32.10 练习十，不要用 sleep 猜一个任务完成

一个后台工作把结果放入队列。

测试目前 sleep 零点一秒后断言结果已经出现。

机器忙时偶尔失败。

改进方向是等待明确完成事件，或在有合理超时的队列读取中等待结果。

超时只是防止永久卡住，成功条件仍然来自实际事件。

不要直接把 sleep 改成十秒，当成问题已经解决。

### 32.11 练习十一，检查测试有没有真的被发现

你写了 `verify_total()`，放在 `checks.py`，运行 pytest 显示 no tests ran。

应该先做什么？

先检查默认发现命名规则，并用 collect-only 观察收集结果。

可以改成 test_total.py 与 test_total，或者在明确需求下调整配置。

不要只在文件底部手动调用函数，然后以为这与 pytest 正常发现完全一样。

### 32.12 练习十二，夹具作用域能解决所有慢问题吗？

把可变购物车夹具从 function 改为 session 后，第二项测试读到了第一项加入的商品。

为什么？

因为共享生命周期变长，测试状态没有隔离。

参考改进是恢复每项独立购物车，或保留昂贵只读基础数据、每项复制生成新的可变状态。

不要仅通过调整测试执行顺序让它碰巧通过。

### 32.13 练习十三，验证一个测试确实能抓 bug

选取 normalize_notes 的空白标题检查。

先运行正常测试集合，确认通过。

临时删除 `.strip()` 相关空白拒绝条件，再运行对应负例，确认失败。

恢复实现并重跑。

这个过程帮助你确认测试没有只是在旁边“陪跑”。

正式项目可采用更系统的变异测试工具，但初学者先掌握这种小实验就很有价值。

### 32.14 练习十四，xfail 已经过时了

某项标记 strict=True 的 xfail 测试突然 XPASS 并让运行失败。

不要立即把 strict=False 当成修复。

先检查问题是否真正修好，测试是否仍然覆盖原场景。

若已修好，移除过期标记，让它成为普通回归测试。

这道题训练你理解标记承诺，而不只是追求控制台全绿。

### 32.15 练习十五，给文件工具增加一个真正需求

为 notes_export 增加“标题长度最多八十个字符”的规则。

先写长度八十通过、八十一拒绝的两项检查，再实现规则。

明确长度是在 strip 前还是 strip 后计算。

如果你选择 strip 后，输入八十个正文字符加两侧空白仍应通过。

参考方向：先得到 normalized_title，再对这个值进行非空与长度检查，最后保存它。

这一练习没有提供整段复制答案，是为了让你把输入契约、边界样例和实现顺序一起想清楚。

## 第三十三章：常见追问，把几个容易混淆的概念分开

### 33.1 passed 越多，质量就一定越高吗？

不一定。

一百组等价中间值可能都没有触及真正错误边界。

少量设计准确的拒绝样例、失败状态检查和完整入口验证，可能更有保护作用。

数量是运行规模信息，不是独立质量结论。

### 33.2 每次改一行都要写新测试吗？

不必机械追求测试文件变化。

如果现有测试已经明确保护这项行为，运行合适范围即可。

如果修复的是缺失的契约或原来漏掉的 bug，通常值得添加回归样例。

测试应提供新证据，不是把实现文字换一种方式再写一遍。

### 33.3 我可以在测试里 print 吗？

可以临时调试。

但最终判断应由断言表达，不能只靠运行者记得看某一行输出。

失败时 pytest 通常会显示相关捕获信息；需要临时实时查看时，可以使用适合的捕获选项。

完成排错后清理无意义打印，有助于报告保持可读。

### 33.4 fixture 与普通辅助函数怎么选？

普通纯转换、构造少量数据，可以直接函数调用。

需要 pytest 管理作用域、依赖和清理的资源，适合夹具。

夹具不是为了替代所有普通函数。

如果你需要动态创建多份独立对象，可以让夹具返回一个工厂函数，但也要保持实际输入容易看见。

### 33.5 mock 会不会让测试不可信？

合适的替身可以让边界可控、避免真实副作用。

过多不符合协议的替身会让测试与现实脱节。

因此，既要有受控模拟检查异常分支，也要在合适层保留真实组件组合检查。

明确“哪些代码真跑了”比简单争论 mock 好坏更有用。

### 33.6 为什么不要在测试中访问真实生产数据库？

普通自动测试需要可重复、可清理的数据范围。

生产数据变化、权限、副作用与隐私都不适合混入一个默认会反复执行的套件。

真实环境验收有自己的授权、只读或隔离策略，应单独明确范围。

本篇所有数据库与文件实践都在临时位置进行，不需要碰用户的真实业务数据。

### 33.7 不安装插件，pytest 本身能做什么？

能完成本篇核心的发现、断言、异常、参数化、夹具、临时目录、环境替换、捕获、日志、标记和命令行运行等。

覆盖率、特定异步支持、并行执行和某些 mock 便利包装来自相应生态工具。

需要时再按项目契约选择依赖，不要把别人的全套插件配置当成入门必需品。

### 33.8 本文的实践是怎样验证的？

本文关键示例在隔离环境中以 pytest 9.2.0、Python 3.12 运行，业务示例使用兼容 Python 3.11 的常见语法。

检查包含真实临时文件、JSON、命令行子进程、SQLite 与进程内 HTTPX MockTransport，不依赖公网服务。

故意失败的演示单独检查其非零退出与失败原因，不混入应通过的完整实践集合。

外部数据库、异步插件、覆盖率插件和多进程插件的说明，没有被冒充为已经安装并完成验证的功能。

需要进一步查阅时，可以从[pytest 官方入门](https://docs.pytest.org/en/stable/getting-started.html)、[运行方式](https://docs.pytest.org/en/stable/how-to/usage.html)、[夹具](https://docs.pytest.org/en/stable/how-to/fixtures.html)与[测试不稳定问题](https://docs.pytest.org/en/stable/explanation/flaky.html)继续阅读。
