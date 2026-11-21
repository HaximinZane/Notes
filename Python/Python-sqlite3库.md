# Python `sqlite3` 超详细教程：从一条笔记记录，到查询、约束、事务、备份与可维护的数据工具

## 零基础先导：为什么保存数据不总是 JSON 文件？

最开始你把笔记保存成 `notes.json` 就很方便。但当你想查找“数学分类且尚未完成的记录”、修改其中一条、保证编号唯一、在失败时撤销部分修改，就会发现自己不得不反复读取整个文件、遍历、改写、检查一致性。

**SQLite 是一种嵌入式关系型数据库引擎**；Python 的 `sqlite3` 模块是**调用 SQLite 的标准库接口**。与需要单独部署的某些数据库服务器不同，SQLite 经常直接操作本地数据库文件；不过多人并发、高网络负载等情况仍需要考虑其限制。

### 先用一个不创建磁盘文件的例子

```python
import sqlite3

with sqlite3.connect(":memory:") as conn:  # ① 内存中的临时数据库
    conn.execute("CREATE TABLE notes (id INTEGER PRIMARY KEY, title TEXT)")  # ② 建表
    conn.execute("INSERT INTO notes (title) VALUES (?)", ("学习 os",))          # ③ 写一条记录
    rows = conn.execute("SELECT id, title FROM notes").fetchall()              # ④ 查找
    print(rows)                                                                  # 例如 [(1, '学习 os')]
```

逐行解释：`:memory:` 是特殊数据库位置，并非普通磁盘路径；`CREATE TABLE` 定义表；`INSERT` 插入数据；`SELECT` 读取数据；`fetchall()` 取出结果行。`?` 是 SQL **值占位符**，让驱动安全绑定值，避免手工拼接用户字符串。`("学习 os",)` 中最后的逗号表示单元素元组。

**重要易错点**：`with sqlite3.connect(...) as conn` 主要管理事务的提交/回滚，**不保证在离开 with 时自动关闭连接**；要在合适位置调用 `conn.close()`，或使用 `contextlib.closing` 等方式管理连接资源。真实文件数据库如果并发读写，还要理解事务、锁与连接生命周期。不要把数据库的 `commit` 和 Python 文件的 `flush` 当成一回事。

---

假设你正在写一个学习笔记管理脚本。

最开始，你把标题、分类和完成状态保存到 CSV 或 JSON 文件里。

后来，你想快速查询“Python 分类中尚未完成的笔记”，按更新时间排序，并保证分类编号确实存在。

你还希望一次修改两条记录时，要么全部成功，要么保留原状。

这时，就可以考虑 SQLite。

这一篇从最小的内存数据库开始，逐步讲清 Python 的连接、游标、SQL 参数、查询结果和事务。

后面再把约束、关联、索引、并发、导入导出和备份接起来，做成能真正理解和修改的小工具。

本文以 **Python 3.11 可用的接口**作为基础；公共练习连接使用 `isolation_level` 管理写事务。Python 3.12 新增的 `autocommit` 接口放在明确标注的章节比较，不能直接复制到 3.11。

学习环境实际验证使用 Python 3.12.3、SQLite 3.45.1。

SQLite 引擎版本与 Python 版本不是同一个号码，较新 SQL 功能会注明要求。

所有写入、删除、迁移和备份示例只应作用于本篇的内存或临时练习数据库，不能拿真实业务数据库尝试。

## 第一章：SQLite 是什么？`sqlite3` 又是什么？

### 1.1 先区分数据库和 Python 模块

SQLite 是一个嵌入式关系数据库引擎。

它能管理表、执行 SQL、维护索引和事务。

Python 的 `sqlite3` 模块是调用这个引擎的接口，属于标准库中的可选组件，常见完整 Python 安装通常已经提供。

你写：

```python
import sqlite3
```

导入的是 Python 接口。

真正执行查询和维护数据库文件的底层引擎是 SQLite。[Python 官方定位](https://docs.python.org/3.14/library/sqlite3.html)

### 1.2 “嵌入式”是什么意思？

它通常运行在你的程序进程中，不需要先启动一个独立数据库服务器，再通过网络连接。

你可以打开一个磁盘文件：

```text
learning.db
```

也可以创建一个只在内存里的练习数据库。

这让个人工具、桌面应用、小型服务和测试很容易使用它。

### 1.3 名字里的 lite 是不是只能做玩具？

不能这样理解。

SQLite 的部署方式轻巧，但它拥有成熟的 SQL、事务与存储机制。

是否适合某个项目，取决于并发写入、部署拓扑、数据量和运维需求。

它也不能因为不需要服务器，就被当成普通文本文件随意在运行中复制和覆盖。

### 1.4 什么时候比 JSON 更方便？

如果你只是保存一小份配置，JSON 或 TOML 很合适。

如果数据是不断增加的记录，要反复按条件查询、排序、关联，并维护唯一性和一致性，数据库往往更容易表达。

例如：

```sql
SELECT title
FROM notes
WHERE category_id = ? AND completed = 0
ORDER BY updated_at DESC;
```

这条语句把查询意图集中表达出来。

当然，SQL 本身也需要学习，不能把数据库当成自动理解业务的黑盒。

## 第二章：先看版本和环境，避免把两个版本号混在一起

### 2.1 查询 Python 与 SQLite

```python
import sys
import sqlite3

print(sys.version)
print(sqlite3.sqlite_version)
print(sqlite3.sqlite_version_info)
```

Python 版本可能是 `3.12.3`。

SQLite 引擎版本可能是 `3.45.1`。

它们分别描述不同组件。

### 2.2 新语法由哪个版本决定？

`Connection.autocommit` 这样的 Python API 由 Python 版本决定。

`RETURNING`、`STRICT` 表等 SQL 能力主要由底层 SQLite 引擎版本决定。

因此，“我用了 Python 3.12”不自动等于“所有最新 SQLite SQL 都可用”。

部署到另一台机器时，应检查实际引擎版本和编译选项。

### 2.3 `sqlite3` 导入失败怎么办？

某些精简或自行构建的 Python 可能缺少该可选模块。

应查安装来源或发行版说明，补全对应支持。

不要随意安装一个同名第三方包就以为替换了标准库接口。

也要检查当前目录有没有自己命名的 `sqlite3.py` 遮蔽标准库。

### 2.4 本篇为什么采用显式事务模式？

Python 3.12 增加 `autocommit` 控制，旧的 `isolation_level` 方式仍然存在。

如果教程从头到尾依赖默认行为，却不解释模式，你很容易把 `commit()`、上下文管理和 SQLite 底层自动提交混为一谈。

我们先用 3.11 就能使用的明确初始化方式做练习，再比较 3.12 的新模式。

这样每个例子“什么时候开始事务、什么时候提交”都能说清楚。

## 第三章：第一个数据库，不创建任何真实文件

### 3.1 先建立一个连接辅助函数

```python
import sqlite3


def open_learning_database(path=":memory:"):
    connection = sqlite3.connect(path, isolation_level=None)
    try:
        connection.execute("PRAGMA foreign_keys = ON")
        connection.row_factory = sqlite3.Row
        connection.isolation_level = "DEFERRED"
    except BaseException:
        connection.close()
        raise
    return connection
```

这段可用于 Python 3.11，亦可用于保持旧事务控制的较新 Python。

先记住它用于本篇练习，下面逐层解释。

### 3.2 `":memory:"` 是什么意思？

这是 SQLite 的特殊数据库名称，表示创建内存数据库。

普通情况下，每次独立连接 `":memory:"` 会得到各自的数据库，不会因为名字一样就自动共享。

关闭连接后，没有额外保存机制的内存数据就不再作为磁盘文件保留。

因此很适合第一次实验。

### 3.3 连接是什么？

连接对象代表程序与某个数据库的一次会话。

它保存事务状态、部分配置、结果处理策略等。

它不等于数据库里的全部数据，也不是某一张表。

同一个数据库文件可以被多个连接打开，各连接有自己的状态和可见性规则。

### 3.4 为什么先 `isolation_level=None`？

我们先让连接处于 SQLite 的自动提交模式，方便在事务外启用外键约束。

某些 PRAGMA 在事务中设置不会按你期望生效，外键开关就是必须认真处理的例子。

设置完成后，再选择 `isolation_level="DEFERRED"`。

在这一旧式控制模式下，适用的 INSERT、UPDATE、DELETE、REPLACE 会隐式开启写事务；提交后不会像 3.12 的 `autocommit=False` 那样立即再开一个事务。

`with connection` 负责结束已有事务，本身不承诺给所有 SELECT 和 DDL 自动加 BEGIN。后面的迁移函数会显式处理 DDL 的事务。

### 3.5 `row_factory` 暂时起什么作用？

它决定后续查询行如何包装。

设为 `sqlite3.Row` 后，可以按列名访问结果，比在很多列之间反复记数字索引更方便。

后面会单独比较元组和 Row。

### 3.6 为什么这里捕获 `BaseException`？

只用于“创建连接后，初始化失败必须关闭”的清理范围。

清理后立即重新抛出，不吞掉退出或中断。

业务逻辑通常应捕获更具体的异常，不能把这段资源清理写法当成所有错误处理的固定模板。

### 3.7 最小完整使用

```python
connection = open_learning_database()

try:
    row = connection.execute("SELECT 1 + 2 AS result").fetchone()
    print(row["result"])
finally:
    connection.close()
```

输出：

```text
3
```

你已经执行了一条 SQL 查询，但还没有创建笔记表。

`finally` 保证正常异常展开时也安排关闭连接。

## 第四章：表、行和列，怎样对应笔记记录？

### 4.1 先设计一张很小的表

```sql
CREATE TABLE notes (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    completed INTEGER NOT NULL DEFAULT 0 CHECK (completed IN (0, 1))
);
```

表名是 `notes`。

每一行代表一条笔记。

每一列描述一个属性。

### 4.2 `id INTEGER PRIMARY KEY` 是什么？

`id` 是记录标识。

`INTEGER PRIMARY KEY` 在普通 SQLite rowid 表中有特殊含义，与行标识关联。

插入时通常可以不提供它，让引擎分配。

但不要把任意名字相近的 `INT PRIMARY KEY` 都当成完全相同的 rowid 别名行为。

类型名称与表形式会影响细节。

### 4.3 `TEXT NOT NULL` 为什么仍不保证标题有内容？

`NOT NULL` 拒绝 SQL 空值。

空字符串 `""` 不是 NULL，所以仍可能被接受。

如果标题不允许空白，应用层应验证，也可以用合适约束进一步表达数据库规则。

数据库约束与应用友好错误提示可以同时存在。

### 4.4 为什么布尔状态用整数？

SQLite 常用零和一表达布尔状态。

这里 `CHECK (completed IN (0, 1))` 限制只接受这两个值。

Python 布尔与整数存在类型关系，但 SQL 存储规则和 Python 对象规则不是一回事。

显式约束让数据库中的值更容易维持一致。

### 4.5 用 Python 创建表

```python
connection = open_learning_database()

try:
    connection.execute("""
        CREATE TABLE notes (
            id INTEGER PRIMARY KEY,
            title TEXT NOT NULL,
            completed INTEGER NOT NULL DEFAULT 0 CHECK (completed IN (0, 1))
        )
    """)
    connection.commit()
finally:
    connection.close()
```

这个例子关闭的是内存库，因此表不会在下一次独立连接中自动继续存在。

要继续插入查询，可以在同一个 `try` 范围里做，或把建表封装成函数。

## 第五章：`execute()`、游标与结果，不要只记方法名字

### 5.1 `execute()` 返回什么？

```python
cursor = connection.execute("SELECT 10 AS value")
```

返回一个游标对象。

游标负责这次语句执行和结果取得等状态。

它不是直接返回数字十，也不是直接返回所有行列表。

### 5.2 连接也有 execute，为什么还会看到 `cursor()`？

完整写法可以是：

```python
cursor = connection.cursor()
cursor.execute("SELECT 10 AS value")
row = cursor.fetchone()
```

连接上的便捷 `execute()` 会替你创建和使用相应游标。

普通单次查询使用便捷入口很清楚。

需要显式管理游标或重复组织某些操作时，也可以自己创建。

### 5.3 `fetchone()`

```python
row = cursor.fetchone()
```

取得下一行，结果已经读完时返回 `None`。

注意 `None` 表示没有下一行，不表示“这一行每个字段都是 NULL”。

一行里可以包含 `None` 字段，而这一行对象本身仍然存在。

### 5.4 `fetchall()`

```python
rows = cursor.fetchall()
```

取得剩余所有结果，放入列表。

小结果集很方便。

结果特别大时，会把全部剩余行保存在内存，应考虑逐行迭代或分批获取。

### 5.5 第一次 `fetchall()` 有结果，第二次为什么空？

游标也有结果消费位置。

已经取走的行不会自动重新出现。

要重新查询，可以再次执行语句，得到新的结果过程。

这和同一个文件对象读取到末尾后，第二次 `read()` 得到空内容，有相似的状态思路。

### 5.6 能直接遍历游标吗？

可以。

```python
for row in connection.execute("SELECT id, title FROM notes ORDER BY id"):
    print(row["id"], row["title"])
```

每轮取得一行，不需要先 `fetchall()`。

但执行与结果处理仍然占用连接资源，长时间持有读结果也可能影响事务与并发，后面会解释。

### 5.7 `fetchmany()` 是分页查询吗？

它从当前结果中取下一批行。

```python
cursor = connection.execute("SELECT id, title FROM notes ORDER BY id")

while True:
    batch = cursor.fetchmany(100)
    if not batch:
        break
    for row in batch:
        print(row["id"], row["title"])
```

这与在 SQL 中多次使用 `LIMIT/OFFSET` 重新查询不同。

前者消费同一个结果过程，后者每次执行新的查询，期间数据库状态可能发生变化。

## 第六章：参数化查询——为什么值一定要单独传？

### 6.1 标题里出现单引号

假设笔记标题是：

```text
Alice's Python notes
```

如果你手工拼接：

```python
# 错误示范：不要把外部值直接拼进 SQL。
sql = f"INSERT INTO notes(title) VALUES ('{title}')"
```

标题里的单引号会与 SQL 字符串边界冲突。

更严重的是，外部值可能被解释成 SQL 结构，形成注入问题。

### 6.2 问号占位符

```python
cursor = connection.execute(
    "INSERT INTO notes(title, completed) VALUES (?, ?)",
    ("Alice's Python notes", 0),
)
```

SQL 结构和数据值分开传入。

库负责把值绑定到相应位置，不需要你手工给标题加引号和转义。

### 6.3 问号外面要不要再写引号？

不要这样写：

```text
INSERT INTO notes(title) VALUES ('?')
```

引号里面的问号是 SQL 字符串内容，不再是你想要的参数位置。

正确写法中，占位符作为 SQL 语法的一部分，不由你额外包成文字。

### 6.4 只有一个参数，为什么写 `(title,)`？

```python
connection.execute("SELECT id FROM notes WHERE title = ?", (title,))
```

这是单元素元组。

如果写 `(title)`，它仍然只是原对象加括号。

字符串本身又可迭代，可能被错误解释成多个字符参数，导致参数数量不匹配。

你也可以提供一个元素的列表，但元组写法很常见。

### 6.5 命名参数

```python
connection.execute(
    "INSERT INTO notes(title, completed) VALUES (:title, :completed)",
    {"title": "学习事务", "completed": 0},
)
```

字典键写 `title`，SQL 占位符写 `:title`。

命名形式适合字段较多时减少位置混淆。

不要把其他数据库驱动的 `%s` 占位符直接抄到这里，具体参数风格由驱动接口决定。

### 6.6 参数能替换表名和列名吗？

不能把它当成任意 SQL 片段替换器。

下面不是动态选择表的正确方式：

```text
SELECT * FROM ?
```

占位符主要绑定数据值，不把一个字符串变成标识符或完整表达式。

动态列名和排序方向应采用严格白名单。

### 6.7 白名单选择排序列

```python
allowed_order = {
    "title": "title",
    "id": "id",
}

requested = "title"
if requested not in allowed_order:
    raise ValueError("不支持的排序字段")

column = allowed_order[requested]
query = f"SELECT id, title FROM notes ORDER BY {column}, id"
rows = connection.execute(query).fetchall()
```

这里插入的是程序自己定义的受控片段，不是任意用户输入。

筛选值仍然继续使用参数绑定。

### 6.8 `LIKE` 模式也可以绑定吗？

可以。

```python
keyword = "Python"
pattern = f"%{keyword}%"

rows = connection.execute(
    "SELECT id, title FROM notes WHERE title LIKE ? ORDER BY id",
    (pattern,),
).fetchall()
```

但 `%`、`_` 是 LIKE 模式字符。

参数化保护 SQL 结构，不自动把用户输入中的模式字符变成普通文字。

如果需求是字面子串，应按明确转义规则处理，或选择适合的字面搜索方式。

## 第七章：插入记录，怎样取得新编号？

### 7.1 插入一条笔记

假设前面已经建立 `notes` 表。

```python
cursor = connection.execute(
    "INSERT INTO notes(title, completed) VALUES (?, ?)",
    ("学习 SQLite", 0),
)
new_id = cursor.lastrowid
connection.commit()

print(new_id)
```

在新建普通练习表中，第一条通常得到编号一，但不要把“下一条永远等于某个固定数字”写进业务逻辑。

### 7.2 `lastrowid` 是这次 execute 的返回值吗？

不是。

`execute()` 返回游标。

`lastrowid` 是游标提供的插入行编号信息，适用于相应普通插入场景。

不要把它当成批量 `executemany()` 的全部 ID 列表，也不要在不符合适用条件的语句后随意读取并假设它刚更新。

### 7.3 `INTEGER PRIMARY KEY` 必须加 AUTOINCREMENT 吗？

通常不必为了自动编号就加。

SQLite 普通整数主键已经能自动分配 rowid。

`AUTOINCREMENT` 有更具体的“不重用某些历史已用编号”等语义和额外成本，不是“自动生成 ID”的唯一开关。

如果编号有跨系统永久身份含义，应单独设计业务标识，不宜只依赖你对行号增长的直觉。

### 7.4 批量插入

```python
records = [
    ("Python 文件处理", 0),
    ("pathlib 路径", 1),
    ("SQLite 事务", 0),
]

with connection:
    connection.executemany(
        "INSERT INTO notes(title, completed) VALUES (?, ?)",
        records,
    )
```

`executemany()` 对多组参数重复执行同一条语句。

它不是接受任意多段不同 SQL 的接口。

### 7.5 批量插入天然是一个事务吗？

事务边界取决于连接模式和你安排的提交范围。

本篇基础连接采用旧式 `DEFERRED` 控制，INSERT 开启事务；外面的 `with connection` 在正常退出时提交这个事务，异常时回滚。

如果换成 SQLite 自动提交模式，不能只凭 `executemany()` 名称就推断一批操作必然全部同时提交。

后面会用失败实验说明。

## 第八章：查询记录，先把筛选、排序和限制讲清楚

### 8.1 查询指定列

```python
rows = connection.execute(
    "SELECT id, title, completed FROM notes ORDER BY id"
).fetchall()
```

`SELECT` 后面列出你需要的字段。

`FROM` 指明来源表。

`ORDER BY` 明确顺序。

### 8.2 不写 ORDER BY，就按插入顺序吗？

不能依赖这种保证。

你在小样本里看到的顺序可能只是当前执行计划和存储状态的结果。

创建索引、修改查询或数据库版本变化后，顺序可能不同。

如果顺序对用户有意义，就显式写出，并在必要时增加唯一的次级排序键。

### 8.3 查询未完成笔记

```python
for row in connection.execute(
    "SELECT id, title FROM notes WHERE completed = ? ORDER BY id",
    (0,),
):
    print(row["id"], row["title"])
```

这里 `0` 是数据值，所以用参数。

表名、列名和比较结构是我们定义的 SQL。

### 8.4 查询一条，找不到怎么办？

```python
note_id = 999
row = connection.execute(
    "SELECT id, title FROM notes WHERE id = ?",
    (note_id,),
).fetchone()

if row is None:
    print("没有这条笔记")
else:
    print(row["title"])
```

不要直接 `fetchone()["title"]`，除非调用前已经有明确存在性保证，或者你就是希望不存在时报错。

### 8.5 计数与平均

```python
row = connection.execute(
    "SELECT COUNT(*) AS total, SUM(completed) AS completed_count FROM notes"
).fetchone()
```

`COUNT(*)` 统计行。

空集合上的 `SUM()` 等聚合可能返回 NULL，而不是零。

需要零时，可以明确使用 `COALESCE`：

```sql
SELECT COUNT(*) AS total,
       COALESCE(SUM(completed), 0) AS completed_count
FROM notes;
```

默认值应符合业务含义，不要把所有不存在的平均值也无条件改成零。

### 8.6 `LIMIT` 与 `OFFSET`

```python
rows = connection.execute(
    "SELECT id, title FROM notes ORDER BY id LIMIT ? OFFSET ?",
    (20, 40),
).fetchall()
```

表示按明确顺序跳过前四十条，再最多取二十条。

参数应在应用层验证为合理非负整数，避免接受意外范围。

### 8.7 大偏移为什么可能慢？

数据库仍可能需要扫描或跳过很多前序记录。

对于按唯一递增键翻页，可以采用游标式条件：

```python
rows = connection.execute(
    "SELECT id, title FROM notes WHERE id > ? ORDER BY id LIMIT ?",
    (last_seen_id, 20),
).fetchall()
```

这要求排序和游标字段有明确规则。

数据在分页期间变化时，两种方法也有不同表现，不能把翻页仅理解为界面页码计算。

## 第九章：更新与删除，为什么 WHERE 特别重要？

### 9.1 标记一条笔记完成

```python
with connection:
    cursor = connection.execute(
        "UPDATE notes SET completed = ? WHERE id = ?",
        (1, note_id),
    )

print(cursor.rowcount)
```

`WHERE id = ?` 限定目标。

不写 WHERE，可能把整张表都更新。

因此，学习修改语句时，先用相同条件 SELECT 看目标，是理解范围的好习惯。

但 SELECT 预检查不是并发安全保证，真正更新仍要通过条件和事务约束实现。

### 9.2 `rowcount=0` 是语法失败吗？

不一定。

SQL 可以成功执行，但没有行匹配目标条件。

应用可以把它解释为“记录不存在”，或者在乐观并发条件下解释为“状态已经变化”。

不要只检查有没有异常，就向用户报告“更新成功一条”。

### 9.3 条件更新状态

```python
with connection:
    cursor = connection.execute(
        "UPDATE notes SET completed = 1 WHERE id = ? AND completed = 0",
        (note_id,),
    )
```

这里把“原来必须未完成”的条件放进同一个更新语句。

比先读取状态、再无条件更新，更能准确表达当前状态要求。

### 9.4 删除一条

```python
with connection:
    cursor = connection.execute("DELETE FROM notes WHERE id = ?", (note_id,))
```

这是数据库记录删除，不是把文件送进回收站。

提交之后是否还能恢复，取决于你有没有备份、审计或软删除设计。

本篇只在练习数据中演示。

### 9.5 软删除是什么？

可以用 `deleted_at` 或状态字段标记，不立即物理删除。

但之后所有相关查询都要明确排除或处理已删除记录，唯一性约束和恢复规则也会更复杂。

它不是加一个布尔字段就自动拥有完整回收站。

如果业务没有这种需求，不必为了“看起来专业”引入大量状态。

## 第十章：NULL、默认值和类型，为什么 SQLite 与 Python 不完全一样？

### 10.1 SQL NULL 对应 Python None

```python
row = connection.execute("SELECT NULL AS value").fetchone()
print(row["value"] is None)
```

输出：

```text
True
```

SQL 空值不等于空字符串、数字零或布尔假。

### 10.2 为什么 `column = NULL` 找不到想要的结果？

SQL 的 NULL 比较具有三值逻辑。

检查空值应使用：

```sql
WHERE optional_field IS NULL
```

检查非空：

```sql
WHERE optional_field IS NOT NULL
```

不要把 Python 的 `== None` 习惯机械搬到 SQL 文本。

### 10.3 DEFAULT 什么时候使用？

如果 INSERT 省略某列，且它有默认值，就可以采用相应默认。

但你显式传入 `None`，表示 SQL NULL，不等于要求数据库忽略这个字段再采用 DEFAULT。

如果列 `NOT NULL`，显式 NULL 可能直接违反约束。

### 10.4 SQLite 的类型亲和性

SQLite 的常规表采用动态类型与类型亲和性规则。

声明列为 INTEGER，不代表它具有某些强静态类型数据库完全相同的拒绝行为。

值可能根据规则转换，也可能以其他存储类别保存。

所以重要数据应结合约束、应用验证，必要时使用支持版本的 STRICT 表。

### 10.5 Python 类型的常见映射

`None` 对应 NULL。

`int` 对应整数。

`float` 对应浮点。

`str` 对应文本。

`bytes` 对应 BLOB。

这份基本映射不自动涵盖任意日期、路径、集合和自定义对象。

### 10.6 金额怎么办？

可以根据协议使用最小货币单位的整数，例如分。

也可以使用经过明确约定的十进制文本和 Decimal 逻辑。

不要因为列名写了 `DECIMAL`，就未经验证地认为 SQLite 常规存储自动提供所有期望的精确十进制运算语义。

### 10.7 STRICT 表

SQLite 3.37.0 起支持 STRICT 表，对允许类型和存储约束提供更严格规则。

```sql
CREATE TABLE measurements (
    id INTEGER PRIMARY KEY,
    value REAL NOT NULL
) STRICT;
```

它需要目标引擎支持，不能仅根据 Python 版本猜测。

即使使用 STRICT，业务范围、单位、唯一标识和权限仍然需要另外表达。

## 第十一章：约束应该写进数据库，还是只写 Python 判断？

### 11.1 两层约束各有作用

Python 校验可以提供友好的字段错误，尽早发现问题。

数据库约束维护所有写入路径共同遵守的最后一道结构规则。

如果另一个脚本也修改数据库，只在某个界面表单里检查就不够。

### 11.2 常见约束

```sql
CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE CHECK (length(trim(name)) > 0)
);
```

`NOT NULL` 拒绝空值。

`UNIQUE` 约束名字唯一。

`CHECK` 约束非空白文本。

### 11.3 `UNIQUE` 怎样处理 NULL？

SQLite 的相应唯一性规则允许某些多 NULL 情况。

如果你要求必填且唯一，就应把 `NOT NULL` 与 `UNIQUE` 一起使用。

不能只写 UNIQUE，就假设这列一定有值。

### 11.4 CHECK 会把 NULL 当成失败吗？

CHECK 的表达式如果结果为 NULL，并不等同于结果为零而违反约束。

因此，想强制提供值时仍需要 NOT NULL。

这又一次说明：SQL 空值逻辑与普通 Python 布尔直觉不同。

### 11.5 违反约束会抛什么？

通常是 `sqlite3.IntegrityError`。

```python
try:
    with connection:
        connection.execute("INSERT INTO categories(name) VALUES (?)", ("Python",))
        connection.execute("INSERT INTO categories(name) VALUES (?)", ("Python",))
except sqlite3.IntegrityError:
    print("分类名称重复，当前事务未按正常路径提交")
```

不要无条件捕获后继续报告“已新增分类”。

是否已有相同分类可复用，应该通过明确查询或 UPSERT 政策处理。

## 第十二章：关联表与外键，分类编号为什么不能随便写？

### 12.1 给笔记增加分类

现在你希望每条笔记属于一个分类。

可以建立两张表：

```sql
CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE CHECK (length(trim(name)) > 0)
);

CREATE TABLE notes (
    id INTEGER PRIMARY KEY,
    category_id INTEGER NOT NULL,
    title TEXT NOT NULL CHECK (length(trim(title)) > 0),
    completed INTEGER NOT NULL DEFAULT 0 CHECK (completed IN (0, 1)),
    FOREIGN KEY (category_id) REFERENCES categories(id)
);
```

`category_id` 保存分类的编号，不在每条笔记中重复存放分类名称。

### 12.2 为什么要写 FOREIGN KEY？

假设分类表中只有编号一和二。

你却插入一条 `category_id=999` 的笔记。

如果没有外键规则，数据库可能接受这条孤立引用。

外键表达：这个值必须对应父表中允许的记录，除非你明确设计可空关系等其他语义。

### 12.3 建表写了外键，为什么无效引用仍然能插入？

SQLite 外键执行需要在每个相关连接上明确启用，并核实结果。

```python
connection.execute("PRAGMA foreign_keys = ON")
row = connection.execute("PRAGMA foreign_keys").fetchone()
print(row[0])
```

通常期望得到一。

但还有一个非常关键的条件。

### 12.4 如果已经在事务里面，开关可能没按预期改变

下面的 Python 3.12+ 例子值得亲自运行：

```python
import sqlite3

connection = sqlite3.connect(":memory:", autocommit=False)
try:
    print(connection.in_transaction)
    connection.execute("PRAGMA foreign_keys = ON")
    print(connection.execute("PRAGMA foreign_keys").fetchone()[0])
finally:
    connection.close()
```

`autocommit=False` 会维持事务管理，连接一开始就处于相应事务状态。

在已开始的多语句事务中，切换外键开关不会按普通直觉生效。

所以本篇辅助函数先在自动提交模式下启用外键，再切换事务模式。[SQLite 外键启用规则](https://www.sqlite.org/foreignkeys.html#fk_enable)

### 12.5 每个连接都要设置吗？

应该在连接初始化时明确设置并确认。

不要只在创建数据库文件的那一次执行，以为以后所有新连接都永久继承了应用期望的连接级开关。

共享一个文件与共享所有会话设置，是两个不同概念。

### 12.6 删除分类时，笔记怎么办？

你需要先决定业务政策。

禁止删除仍有笔记引用的分类。

把分类里的笔记一并删除。

或者把分类引用设为空，前提是模型允许空值。

外键的 `ON DELETE` 动作可以表达相应策略，但不能代替你决定哪一种符合业务。

### 12.7 CASCADE 很方便，为什么也要小心？

```sql
FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE
```

它会让父记录删除影响子记录。

如果用户以为只是“删除分类标签”，实际却删掉了所有笔记，就说明模型与界面含义没有对齐。

本篇笔记工具采用默认限制策略，让有子记录的分类删除失败，用户先明确处理笔记去向。

### 12.8 JOIN 把两张表查询出来

```python
rows = connection.execute("""
    SELECT n.id, n.title, n.completed, c.name AS category_name
    FROM notes AS n
    JOIN categories AS c ON c.id = n.category_id
    ORDER BY n.id
""").fetchall()

for row in rows:
    print(row["id"], row["category_name"], row["title"])
```

`AS n` 和 `AS c` 给表取短别名。

`ON` 明确关联条件。

不要遗漏连接条件，让每条笔记与每个分类组合成笛卡尔积，再以为数据库复制了记录。

## 第十三章：事务——为什么两条正确 SQL 放在一起，仍然可能需要额外安排？

### 13.1 用一个转移任务的情境理解

假设工具要把一条笔记从“待整理”分类转到“已整理”，并增加一条事件记录。

第一条 SQL 更新笔记。

第二条 SQL 写事件。

如果第一条成功，第二条因约束失败，你希望最后状态是什么？

通常你希望两条一起成功，或者都不留下半套修改。

这就是事务要解决的一类问题。

### 13.2 提交和回滚

`commit()` 表示提交适用事务中的更改。

`rollback()` 表示回滚适用事务中的更改。

本篇现代模式下，连接会在提交或回滚后继续保持新的事务管理状态，因此你可能看到 `in_transaction` 很快仍为真。

不要把它误解为上一次提交完全没有发生。

### 13.3 最直接的结构

```python
try:
    connection.execute(
        "UPDATE notes SET category_id = ? WHERE id = ?",
        (target_category_id, note_id),
    )
    connection.execute(
        "INSERT INTO events(note_id, message) VALUES (?, ?)",
        (note_id, "已移动分类"),
    )
    connection.commit()
except BaseException:
    connection.rollback()
    raise
```

这里清理范围中的 `BaseException` 用于回滚后立即重新抛出，避免中断时留下未结束事务状态。

普通业务错误提示可以在更外层捕获具体异常。

### 13.4 使用 `with connection`

```python
with connection:
    connection.execute(
        "UPDATE notes SET category_id = ? WHERE id = ?",
        (target_category_id, note_id),
    )
    connection.execute(
        "INSERT INTO events(note_id, message) VALUES (?, ?)",
        (note_id, "已移动分类"),
    )
```

在适用的事务模式中，正常退出时提交，异常退出时回滚。

它让一个业务操作的事务范围更容易看清。

### 13.5 进入 with 之前的修改也可能被一起提交

如果你在同一个当前事务中，已经做了别的写入，再进入 `with connection`，上下文退出处理面对的是这个连接的事务，不是只认识缩进块里的几行代码。

所以应把事务边界从业务入口就安排清楚。

不要把 `with connection` 当成一个自动新建、完全隔离于之前工作的独立子事务。

### 13.6 异常在块内被你吞掉，会怎样？

```python
with connection:
    connection.execute("INSERT INTO categories(name) VALUES (?)", ("Python",))
    try:
        connection.execute("INSERT INTO categories(name) VALUES (?)", ("Python",))
    except sqlite3.IntegrityError:
        print("忽略重复")
```

外层上下文看到正常离开，可能提交前面成功的修改。

如果你的原意是“任何一步失败就整组撤销”，不要在事务块内把异常吞掉后装作正常完成。

把具体错误处理放在事务上下文之外，通常更容易保持整体回滚语义。

### 13.7 回滚能撤销文件、网络请求和打印吗？

不能。

数据库事务主要管理数据库状态。

你在中间发出的邮件、写到其他文件的内容或调用外部接口造成的效果，不会因为 `rollback()` 自动撤回。

跨系统一致性需要另外设计，不应该把本地数据库事务扩大成所有副作用的万能撤销按钮。

### 13.8 为什么事务不应该开很久？

如果事务中等待用户输入、下载大文件或执行长时间计算，就会延长持有快照和锁相关资源的时间。

其他写入者可能一直等待，WAL 也可能因为长读事务而难以回收某些历史页。

常见做法是在事务外准备和验证数据，在事务内完成尽量短而明确的数据库修改。

## 第十四章：`with connection` 会关闭连接吗？

### 14.1 一个很常见的误会

文件对象的 `with` 会关闭文件。

于是很多人以为：

```python
with sqlite3.connect("practice.db") as connection:
    ...
```

离开之后连接也一定自动关闭。

但 SQLite 连接的上下文管理主要处理事务，不承担同样的自动关闭职责。

### 14.2 明确关闭

```python
connection = open_learning_database()
try:
    with connection:
        connection.execute("CREATE TABLE sample(value INTEGER)")
finally:
    connection.close()
```

外层资源生命周期与内层事务范围分别表达。

如果同一个连接要完成多次独立业务操作，也可以在外层保持连接，内部多次使用清楚的事务块。

### 14.3 用 `contextlib.closing()`

```python
from contextlib import closing

with closing(open_learning_database()) as connection:
    with connection:
        connection.execute("CREATE TABLE sample(value INTEGER)")
```

`closing()` 在退出时调用 `close()`。

内层连接上下文继续负责事务。

看起来两层 `with`，其实管理的是两种不同事情。

### 14.4 关闭会自动提交吗？

不要依赖关闭替代提交。

未提交事务在关闭时通常不会被当成“用户希望保存”而自动提交，可能回滚。

要保留更改，应明确提交或采用已解释清楚的事务上下文。

### 14.5 游标要不要关闭？

显式长期持有的游标应在不再需要时关闭，尤其没有完整消费结果时。

连接关闭也会结束相关资源，但不应让一个大结果游标无意义地存活很久。

使用 `cursor.close()` 只关闭游标，不等于关闭整个数据库连接。

## 第十五章：Python 3.12+ 的 autocommit，名字容易带来哪些误解？

### 15.1 三种主要取值

`autocommit=False`：采用 Python DB-API 风格的事务管理，保持事务，提交或回滚后会进入新的事务。

`autocommit=True`：SQLite 自动提交模式，连接的 `commit()`、`rollback()` 不按前一种模式承担提交开放事务的作用。

`autocommit=sqlite3.LEGACY_TRANSACTION_CONTROL`：采用较早 Python 的事务控制方式，主要结合 `isolation_level`。

具体默认值应查目标 Python 版本，不要把历史默认当成未来永久规则。[Python 事务控制说明](https://docs.python.org/3.14/library/sqlite3.html#transaction-control)

### 15.2 `autocommit=False` 为什么 SELECT 之后也有事务？

它希望连接处于持续受管理的事务状态。

这与旧模式“某些 DML 语句前才隐式 BEGIN”的行为不同。

因此，迁移代码时不能只改一个参数，就假设 PRAGMA、显式 BEGIN 和上下文嵌套行为完全不变。

### 15.3 `in_transaction` 与 autocommit 是同一个状态吗？

不是。

`autocommit` 是 Python 连接采用的事务控制政策。

`in_transaction` 反映底层 SQLite 是否处于相应活动事务状态。

它们相关，但一个是控制模式，一个是当前底层状态，不能当成同一个布尔开关。

### 15.4 切换 autocommit 可能有副作用

切换模式不是无害的显示设置。

例如切换到自动提交模式可能提交待处理事务，切回管理模式会开启相应事务。

不要在业务代码中为了解决一个锁错误随意反复切换。

先明确你要提交、回滚还是结束连接，再采用一致政策。

### 15.5 自动提交模式适合什么？

某些连接初始化 PRAGMA、显式 SQL 事务控制或特定数据库维护动作需要事务外执行。

但这不意味着普通多语句业务修改都应该无条件自动提交。

如果每条语句分别提交，中途失败就可能留下部分业务状态。

### 15.6 执行显式 BEGIN 时要留意当前模式

在 `autocommit=False` 已经存在事务时，再执行：

```sql
BEGIN;
```

可能得到“无法在事务内再开始事务”的错误。

你需要选一种清楚的控制方式，不要把所有教程的 BEGIN、with、isolation_level 和 autocommit 示例叠到同一个连接上。

## 第十六章：如果我的 Python 还是 3.11 呢？

### 16.1 没有同样的 autocommit 参数

Python 3.11 的 `sqlite3.connect()` 不支持 3.12 新增的相应参数。

可以采用旧接口，并明确理解其行为。

### 16.2 回看基础连接的兼容初始化

```python
import sqlite3


def open_legacy_database(path=":memory:"):
    connection = sqlite3.connect(path, isolation_level=None)
    try:
        connection.execute("PRAGMA foreign_keys = ON")
        connection.row_factory = sqlite3.Row
        connection.isolation_level = "DEFERRED"
    except BaseException:
        connection.close()
        raise
    return connection
```

先用 `isolation_level=None` 让 SQLite 处于自动提交模式，启用外键。

再选择旧式隐式事务控制。

### 16.3 它与现代 False 模式完全一样吗？

不完全一样。

旧控制主要在适用的 INSERT、UPDATE、DELETE、REPLACE 等 DML 语句前隐式开启事务。

某些 SELECT、DDL 等操作的事务时机不同，不能因为 `with connection` 表面写法相似，就认为所有语句被包裹方式相同。

需要明确把建表和写入都放进一个显式事务时，应按所选模式组织并验证。

### 16.4 `DEFERRED`、`IMMEDIATE` 和 `EXCLUSIVE`

它们涉及 SQLite BEGIN 的不同锁取得策略。

`DEFERRED` 延后到实际需要时取得相应锁。

`IMMEDIATE` 试图更早开始写事务，减少某些“先读后升级写锁”的竞争，但会更早占据写入机会。

`EXCLUSIVE` 在不同日志模式下有具体差异，不应只凭名字推断所有读取者行为。

初学先采用一种明确方式，再在真实并发需求下研究更细的锁策略。

### 16.5 Python 3.12+ 如果想采用新控制方式呢？

可以明确另写一个现代连接函数：

```python
def open_modern_database(path=":memory:"):
    connection = sqlite3.connect(path, autocommit=True)
    try:
        connection.execute("PRAGMA foreign_keys = ON")
        connection.row_factory = sqlite3.Row
        connection.autocommit = False
    except BaseException:
        connection.close()
        raise
    return connection
```

这段明确要求 Python 3.12+。

初始化先在事务外开外键，再进入 `autocommit=False`。后续即使提交或回滚，也会自动开始新的事务。

它可以用于前面的普通增删改查，但读事务持续时间、显式 BEGIN 与模式切换仍要按第十五章判断。

### 16.6 为什么教程不把兼容分支藏在一行代码里？

因为兼容的不只是参数是否存在，还有事务时机和状态含义。

如果工具支持多版本，应有测试覆盖目标行为，文档明确支持范围。

单纯 `try/except TypeError` 改用另一个参数，并不能证明行为等价。

## 第十七章：SAVEPOINT——一个事务里，想撤销其中一小段怎么办？

### 17.1 BEGIN 不能直接无限嵌套

普通事务不能简单一层层重复 BEGIN。

需要局部回退时，可以使用保存点。

```sql
SAVEPOINT optional_part;
```

表示在当前事务中建立一个可命名的位置。

### 17.2 一个可运行的小实验

```python
connection = open_learning_database()
try:
    connection.execute("CREATE TABLE values_demo(value INTEGER UNIQUE)")
    connection.commit()

    with connection:
        connection.execute("INSERT INTO values_demo(value) VALUES (1)")
        connection.execute("SAVEPOINT optional_part")
        try:
            connection.execute("INSERT INTO values_demo(value) VALUES (2)")
            connection.execute("INSERT INTO values_demo(value) VALUES (1)")
        except sqlite3.IntegrityError:
            connection.execute("ROLLBACK TO SAVEPOINT optional_part")
        finally:
            connection.execute("RELEASE SAVEPOINT optional_part")

        connection.execute("INSERT INTO values_demo(value) VALUES (3)")

    print([row[0] for row in connection.execute("SELECT value FROM values_demo ORDER BY value")])
finally:
    connection.close()
```

预期保留：

```text
[1, 3]
```

### 17.3 为什么回滚后还要 RELEASE？

`ROLLBACK TO` 撤销保存点之后的修改，但保存点本身还在。

`RELEASE` 结束相应保存点范围。

它不是无条件提交整个外层事务。

外层后来失败，已经 RELEASE 的局部工作仍然可能随外层整体回滚。

### 17.4 保存点名字能来自用户输入吗？

不要随意拼接外部名字到 SQL 结构。

示例使用程序内部固定名称。

如果封装通用嵌套保存点工具，应生成受控且不冲突的标识，并处理异常清理。

### 17.5 什么时候不该使用局部回退？

如果业务要求整组操作严格同时成功，局部吞掉错误会改变契约。

保存点适合你明确允许可选子步骤失败的情境。

不要因为“不想看到报错”就把每条写入包成一个自动忽略失败的小保存点。

## 第十八章：结果行与自定义转换，为什么有时拿到元组，有时可以按列名？

### 18.1 默认结果通常是元组

如果没有设置行工厂：

```python
import sqlite3

connection = sqlite3.connect(":memory:", isolation_level="DEFERRED")
try:
    row = connection.execute("SELECT 1 AS id, 'Python' AS title").fetchone()
    print(row)
    print(row[1])
finally:
    connection.close()
```

通常得到元组和第二项标题。

列多时，记住索引对应关系容易出错。

### 18.2 `sqlite3.Row`

本篇初始化设置：

```python
connection.row_factory = sqlite3.Row
```

让后续相应游标产生 Row 对象。

可以按索引，也可以按列名访问：

```python
row[0]
row["id"]
```

转为普通字典：

```python
record = dict(row)
```

### 18.3 两个查询列都叫 id，会不会混淆？

关联查询时应明确取别名。

```sql
SELECT n.id AS note_id, c.id AS category_id, n.title
FROM notes AS n
JOIN categories AS c ON c.id = n.category_id;
```

这样结果模型更清楚，也避免按同名字段访问产生不直观结果。

### 18.4 `SELECT *` 为什么不总是好选择？

表结构变化时，返回列数量和顺序也会变化。

两个表 JOIN 后，重复列名更多。

接口或导出依赖固定字段时，明确列名更容易审阅。

临时交互式查看可以用星号，但正式数据边界最好选择真正需要的列。

### 18.5 日期会自动变成 datetime 吗？

不要依赖这种默认保证。

Python 旧版曾提供一些默认适配器与转换器，这些默认行为从 Python 3.12 起已弃用。

而且默认时间戳转换对时区等含义可能不符合你的需求。

更清楚的做法是明确保存约定，例如 UTC ISO 文本或整数时间戳，再在应用层按契约解析。

### 18.6 自定义适配与转换何时值得使用？

当项目有统一模型，并清楚知道数据库列类型声明与转换规则，可以使用 `register_adapter()`、`register_converter()` 和 `detect_types`。

但注册行为与全局配置有关，复杂项目应避免各模块互相覆盖不一致规则。

对于初学笔记工具，显式生成字符串、读取后显式解析，更容易看清数据流。

## 第十九章：索引与 EXPLAIN QUERY PLAN，为什么加索引有时快，有时没帮助？

### 19.1 查询越来越慢

笔记只有十条时，扫描全表几乎感觉不到。

有十万条时，你经常按分类和完成状态查询，就可能需要合适索引。

例如：

```sql
SELECT id, title
FROM notes
WHERE category_id = ? AND completed = ?
ORDER BY id;
```

### 19.2 创建匹配查询的索引

```sql
CREATE INDEX idx_notes_category_completed_id
ON notes(category_id, completed, id);
```

索引维护一份适合按这些字段查找的结构。

它不是复制一张供你手动编辑的表，也不保证所有查询都变快。

### 19.3 索引为什么有成本？

插入、更新和删除相关记录时，数据库还要维护索引。

索引占磁盘，也会增加部分写入工作。

所以不要为了“保险”给每个字段随意建立十几个重复索引。

先观察真实查询条件、排序和数据分布。

### 19.4 看查询计划

```python
plan = connection.execute(
    "EXPLAIN QUERY PLAN SELECT id, title FROM notes WHERE category_id = ? AND completed = ? ORDER BY id",
    (1, 0),
).fetchall()

for row in plan:
    print(tuple(row))
```

它提供高层执行策略信息，例如扫描表还是使用某个索引查找。

输出文本可能随 SQLite 版本变化，官方不建议应用把它当成稳定机器协议。[EXPLAIN QUERY PLAN 官方说明](https://www.sqlite.org/eqp.html)

### 19.5 看到 SCAN 就一定是坏事吗？

不是。

表很小，或者查询要读取大部分记录时，全表扫描可能合理。

查询规划器会结合统计和成本选择。

性能判断应结合实际耗时、数据规模和计划，而不是只看到某个单词就修改配置。

### 19.6 联合索引的列顺序

索引从左侧字段开始组织，查询只使用后面某列时，效果未必与你想的一样。

等值条件、范围条件和排序之间也会相互影响。

不要把 `ON(a,b,c)` 当成同时拥有三份任意独立索引的简单合成。

学习时可以分别执行几种 WHERE 条件，观察计划，再用代表性数据测量。

### 19.7 有唯一约束，还要再建同样索引吗？

唯一约束和主键通常已经带来相应索引或底层结构。

再建完全重复索引可能没有收益。

可以通过 PRAGMA 和模式查询检查现有结构，但不要在不理解业务约束时随意删除索引。

## 第二十章：并发、锁和 WAL，为什么会出现 database is locked？

### 20.1 两个连接不是同时无限写入

SQLite 允许多个连接读取。

写入仍有协调和锁规则，不能因为打开了十个连接就让十个写事务随意并行修改同一文件。

如果一个事务长时间未提交，另一个写入者可能等待，最终遇到忙或锁相关错误。

### 20.2 `timeout` 的作用

```python
connection = sqlite3.connect("practice.db", timeout=5.0, isolation_level="DEFERRED")
```

它控制遇到相应锁竞争时等待的超时设置，不是每条查询的严格总执行时限。

计算特别昂贵的查询也不会仅靠这个参数在五秒时自动中断。

把锁等待超时和查询运行截止时间分开理解。

### 20.3 第一反应应该检查什么？

是否有未提交或未回滚的写事务。

是否在事务里面等待网络、睡眠或人工输入。

是否有长期未消费完的读结果。

是否多个工具同时修改同一个文件。

不要看到锁错误就先删除数据库旁边的日志文件。

这些文件可能参与正确恢复和当前状态，随意删除可能破坏一致性。

### 20.4 WAL 是什么？

WAL 来自 Write-Ahead Logging，预写日志。

它采用不同的提交记录与检查点机制，在许多场景中改善读写并发。

开启可以使用：

```sql
PRAGMA journal_mode = WAL;
```

应在适合的事务外连接初始化或维护阶段执行，并检查返回模式。

内存数据库和某些环境不一定采用你请求的模式。

### 20.5 WAL 是否允许多个写事务同时随意执行？

仍然不是无限并发写入。

WAL 常让读者与写者更容易并行，但同时写入仍然需要协调。

如果业务写入竞争持续很高，应分析事务长度、写入调度和系统架构，而不只是继续增加连接数。[SQLite WAL 机制](https://www.sqlite.org/wal.html)

### 20.6 `-wal` 和 `-shm` 是垃圾文件吗？

它们可能属于正在运行的数据库状态和共享机制。

不能因为只认识 `.db` 主文件，就把旁边文件当成临时垃圾删除。

尤其备份或移动运行中的 WAL 数据库时，单独拷贝主文件可能遗漏尚在日志中的已提交内容。

应采用数据库支持的备份方式，或在明确停机和状态确认下执行一致的文件级备份流程。

### 20.7 长读事务为什么也可能有影响？

读者可能持有某个历史快照，检查点不能随意丢弃它还需要的页面版本。

如果一直不结束，WAL 文件可能增长，后续维护受到影响。

因此，读操作虽然不修改记录，也应该及时消费结果、结束事务和释放连接。

### 20.8 WAL 能放在任意网络共享盘上吗？

WAL 依赖相应共享内存和文件系统锁等条件，不适合把普通本机部署假设直接搬到任意网络文件系统。

数据库文件所在位置、挂载行为和多进程访问方式，都需要符合 SQLite 的支持边界。

在 WSL、Windows 工具和共享挂载盘混合访问时，也应明确谁持有连接，避免凭历史成功经验断定当前组合安全可靠。

### 20.9 `check_same_thread=False` 能解决线程安全了吗？

默认连接有同线程使用检查。

关闭检查只移除这一层限制，不自动让你的事务操作变成并发安全的业务序列。

多个线程共享同一个连接，仍然可能交叉操作事务、游标和状态。

常见选择是每个工作线程管理自己的连接，或把写入集中到一个明确协调的执行者。

### 20.10 `sqlite3.threadsafety` 告诉你什么？

它反映底层构建对应的线程安全能力等级。

这不等于所有 Python 层共享连接的使用方式都自动符合你的业务要求。

底层保护与应用事务边界是不同层次，仍要根据设计进行同步。

## 第二十一章：只读连接、数据库路径和误创建文件

### 21.1 路径拼错，为什么出现一个新的空数据库？

普通连接某个不存在的数据库路径时，可能尝试创建新文件。

如果你本来只是查看已有数据库，拼错名字却创建了一份空库，就会看到“no such table”，误以为数据被清空。

先打印明确路径，确认它是预期文件。

### 21.2 要求目标已经存在

SQLite URI 形式可以设置模式：

```python
from pathlib import Path
import sqlite3

path = Path("practice.db").resolve()
uri = path.as_uri() + "?mode=ro"
connection = sqlite3.connect(uri, uri=True, isolation_level=None)

try:
    rows = connection.execute(
        "SELECT name FROM sqlite_schema WHERE type = 'table' ORDER BY name"
    ).fetchall()
    print(rows)
finally:
    connection.close()
```

`mode=ro` 表示只读打开，目标不存在时不会按普通读写模式自动创建。

URI 必须按接口规则构造，不能把任意查询字符串直接拼接到未编码路径上。

### 21.3 只读表示完全没有环境要求吗？

仍然需要能访问相应文件和目录，WAL 等模式还有自己的只读条件。

只读连接也可能持有读事务或快照，不能无限期不释放。

### 21.4 当前工作目录仍然影响相对路径

脚本位于：

```text
D:\tools\inspect_db.py
```

终端位于：

```text
C:\Users\Alice
```

如果代码写 `sqlite3.connect("practice.db")`，通常按当前工作目录寻找或创建。

它不自动以脚本目录为基准。

这与 `open()`、`pathlib` 的相对路径问题一致，数据库接口也没有例外。

### 21.5 连接不会自动创建缺失父目录

数据库文件不存在可以创建，不表示整条目录层级都会创建。

需要时在明确的写入初始化流程中：

```python
path.parent.mkdir(parents=True, exist_ok=True)
```

只读查看工具则不应该为了“路径不存在”先创建目录并新建空数据库，那会掩盖输入错误。

## 第二十二章：备份与导出，为什么不能把运行中的数据库当普通文本复制？

### 22.1 单个主文件未必包含全部当前状态

WAL 模式下，已提交修改可能尚在日志中。

其他日志模式也涉及事务和恢复状态。

正在写入时直接复制主文件，可能取得不一致或不完整版本。

因此，优先使用支持数据库一致性语义的备份 API。

### 22.2 `Connection.backup()`

```python
import sqlite3

source = open_learning_database("practice.db")
target = sqlite3.connect("practice_backup.db", isolation_level=None)

try:
    # 先结束本连接中尚未完成的工作；备份不替你决定是否提交业务修改。
    source.commit()
    source.backup(target)
finally:
    target.close()
    source.close()
```

这里只操作专门练习库。

目标路径若已有内容，备份会按 API 语义写入目标数据库，因此不能随便指向真实文件。

### 22.3 源连接为什么先处理事务状态？

备份不是“自动提交所有用户工作”的业务按钮。

源连接若正持有自己的写事务，可能导致备份等待等不符合你直觉的行为。

先明确提交或回滚，再按已知状态调用，能让练习过程可预测。

对复杂运行系统，可以使用独立连接和经过验证的在线备份流程。

### 22.4 分批备份与进度

下面是替换前一个完整示例中 `source.backup(target)` 的片段，须放在 `try` 内、连接关闭之前。

```python
def show_progress(status, remaining, total):
    print("剩余页：", remaining, "总页数：", total)

source.backup(target, pages=100, progress=show_progress, sleep=0.01)
```

这里的单位是数据库页面，不是业务记录条数。

回调不应该在每一页里执行昂贵网络操作，拖长备份或引入难以诊断的副作用。

### 22.5 SQL 文本导出 `iterdump()`

下面片段同样要求 `source` 连接仍然打开；不能接在前例 `finally` 已经关闭连接之后执行。

```python
with open("practice_dump.sql", "x", encoding="utf-8", newline="\n") as target:
    for statement in source.iterdump():
        target.write(statement + "\n")
```

它产生 SQL 文本形式，适合某些迁移、检查和恢复用途。

这不是 CSV，也不是逐字节复制原数据库布局。

恢复时执行的是 SQL，输入必须可信并经过适当验证，不能随意执行来源不明的脚本。

### 22.6 备份完成，怎样验证？

用另一个连接打开备份，检查关键表、记录数量和代表性查询。

根据任务要求运行 `PRAGMA integrity_check` 或相关检查。

还要实际测试恢复流程，而不是只确认备份文件大小大于零。

存在一个文件，不等于你已经拥有可用恢复能力。[SQLite 备份机制说明](https://www.sqlite.org/backup.html)

## 第二十三章：完整实践——建立一个小型笔记任务库

前面已经分别理解连接、约束和事务。

现在把它们接成一个完整的小模型。

本章继续使用 `open_learning_database()`，基础代码在 Python 3.11 就有相应接口，并已经启用外键。建表函数会显式开启需要的 DDL 事务。

### 23.1 需求先说清楚

分类名字唯一且不能为空白。

每条笔记有标题、分类、完成状态和 UTC 更新时间。

同一分类中，不允许出现相同标题。

标记完成时，同时写一条附属事件记录。

分类仍被笔记引用时，删除分类应失败。

本章不会设计用户登录、网络接口或大型框架，只实现数据库层的小工具函数。

### 23.2 建表函数

```python

def create_notes_schema(connection):
    statements = [
        """
        CREATE TABLE IF NOT EXISTS categories (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL UNIQUE CHECK (length(trim(name)) > 0)
        )
        """,
        """
        CREATE TABLE IF NOT EXISTS notes (
            id INTEGER PRIMARY KEY,
            category_id INTEGER NOT NULL,
            title TEXT NOT NULL CHECK (length(trim(title)) > 0),
            completed INTEGER NOT NULL DEFAULT 0 CHECK (completed IN (0, 1)),
            updated_at TEXT NOT NULL,
            revision INTEGER NOT NULL DEFAULT 1 CHECK (revision >= 1),
            UNIQUE (category_id, title),
            FOREIGN KEY (category_id) REFERENCES categories(id)
        )
        """,
        """
        CREATE TABLE IF NOT EXISTS note_events (
            id INTEGER PRIMARY KEY,
            note_id INTEGER NOT NULL,
            message TEXT NOT NULL,
            created_at TEXT NOT NULL,
            FOREIGN KEY (note_id) REFERENCES notes(id) ON DELETE CASCADE
        )
        """,
        """
        CREATE INDEX IF NOT EXISTS idx_notes_category_completed_id
        ON notes(category_id, completed, id)
        """,
    ]

    with connection:
        if not connection.in_transaction:
            connection.execute("BEGIN")
        for statement in statements:
            connection.execute(statement)
```

### 23.3 为什么这里逐条 execute，而不是随意拼脚本？

每个字符串是程序内部完整的一条受控语句。

事务范围清楚，不依赖你对 `executescript()` 在不同事务模式下的隐式处理记忆。

后面还会单独说明脚本接口的边界。

### 23.4 `IF NOT EXISTS` 能自动升级旧表吗？

不能。

它主要避免同名对象已存在时报对应创建错误。

如果旧 `notes` 表缺少 `revision` 列，这条语句不会自动把它补上。

重复初始化与数据库迁移是两个不同需求。

### 23.5 UTC 时间函数

```python
from datetime import datetime, timezone


def utc_now_text():
    return datetime.now(timezone.utc).isoformat(timespec="microseconds")
```

统一时区和固定表现形式，便于比较和解释。

它不表示数据库自动验证了每个字符串都是合法 UTC 时间，所有写入路径仍应遵循契约。

如果系统对时间顺序有更强要求，还要考虑时钟调整和业务事件序列，不能只靠墙上时钟戳解决全部排序问题。

### 23.6 创建分类

```python

def create_category(connection, name):
    if not isinstance(name, str) or not name.strip():
        raise ValueError("分类名称不能为空白")
    name = name.strip()

    with connection:
        cursor = connection.execute(
            "INSERT INTO categories(name) VALUES (?)",
            (name,),
        )
        category_id = cursor.lastrowid

    return category_id
```

应用层先提供友好错误。

数据库 UNIQUE 仍防止其他连接或其他代码路径插入重复名称。

函数遇到重复时保留 `IntegrityError`，由调用者决定提示还是复用已有分类。

### 23.7 新增笔记

```python

def create_note(connection, category_id, title):
    if type(category_id) is not int or category_id <= 0:
        raise ValueError("分类编号必须是正整数")
    if not isinstance(title, str) or not title.strip():
        raise ValueError("笔记标题不能为空白")

    title = title.strip()
    now = utc_now_text()

    with connection:
        cursor = connection.execute(
            """
            INSERT INTO notes(category_id, title, completed, updated_at)
            VALUES (?, ?, 0, ?)
            """,
            (category_id, title, now),
        )
        note_id = cursor.lastrowid
        connection.execute(
            "INSERT INTO note_events(note_id, message, created_at) VALUES (?, ?, ?)",
            (note_id, "创建笔记", now),
        )

    return note_id
```

笔记与创建事件处在同一事务。

如果事件写入失败，这次创建也不应通过正常路径留下半套状态。

### 23.8 查询未完成列表

```python

def list_pending_notes(connection, category_name=None, limit=20):
    if type(limit) is not int or not 1 <= limit <= 1000:
        raise ValueError("limit 必须是 1 到 1000 的整数")

    if category_name is None:
        cursor = connection.execute(
            """
            SELECT n.id, n.title, c.name AS category, n.updated_at, n.revision
            FROM notes AS n
            JOIN categories AS c ON c.id = n.category_id
            WHERE n.completed = 0
            ORDER BY n.id
            LIMIT ?
            """,
            (limit,),
        )
    else:
        if not isinstance(category_name, str) or not category_name.strip():
            raise ValueError("分类名称必须是非空字符串")
        cursor = connection.execute(
            """
            SELECT n.id, n.title, c.name AS category, n.updated_at, n.revision
            FROM notes AS n
            JOIN categories AS c ON c.id = n.category_id
            WHERE n.completed = 0 AND c.name = ?
            ORDER BY n.id
            LIMIT ?
            """,
            (category_name.strip(), limit),
        )

    return [dict(row) for row in cursor]
```

这里选择两条固定 SQL 分支，避免把没有必要的动态 SQL 拼接复杂化。

规模更大时可以组合受控条件，但参数值仍应单独绑定。

### 23.9 标记完成，并防止无声覆盖旧版本

假设用户刚看到 `revision=1` 的笔记。

另一位操作已经修改了它，版本变成二。

如果你仍无条件更新，可能覆盖对方基于新状态作出的决定。

可以把期待版本放进 WHERE：

```python

class NoteConflictError(Exception):
    pass


def complete_note(connection, note_id, expected_revision):
    if type(note_id) is not int or note_id <= 0:
        raise ValueError("note_id 必须是正整数")
    if type(expected_revision) is not int or expected_revision <= 0:
        raise ValueError("expected_revision 必须是正整数")

    now = utc_now_text()

    with connection:
        cursor = connection.execute(
            """
            UPDATE notes
            SET completed = 1,
                updated_at = ?,
                revision = revision + 1
            WHERE id = ? AND revision = ? AND completed = 0
            """,
            (now, note_id, expected_revision),
        )

        if cursor.rowcount != 1:
            raise NoteConflictError("笔记不存在、已经完成，或版本已变化")

        connection.execute(
            "INSERT INTO note_events(note_id, message, created_at) VALUES (?, ?, ?)",
            (note_id, "标记完成", now),
        )
```

### 23.10 这叫乐观并发检查，但不是全部并发设计

它让一条更新只有在当前状态符合预期时才成功。

事务隔离、锁等待和业务授权仍需单独处理。

错误中合并了“不存在、已完成、版本变化”几种情况，是本例接口选择。

如果界面需要更精确反馈，可以在失败后再查询，但查询结果也只是相应时刻观察到的状态。

### 23.11 在内存里完整跑一遍

```python
connection = open_learning_database()
try:
    create_notes_schema(connection)
    python_id = create_category(connection, "Python")
    linux_id = create_category(connection, "Linux")

    first_id = create_note(connection, python_id, "open 文件模式")
    second_id = create_note(connection, python_id, "pathlib 路径")
    create_note(connection, linux_id, "进程与文件描述符")

    pending = list_pending_notes(connection, "Python")
    print([(row["id"], row["title"], row["revision"]) for row in pending])

    complete_note(connection, first_id, expected_revision=1)

    pending = list_pending_notes(connection, "Python")
    print([row["title"] for row in pending])

    event_count = connection.execute(
        "SELECT COUNT(*) FROM note_events WHERE note_id = ?",
        (first_id,),
    ).fetchone()[0]
    print(event_count)
finally:
    connection.close()
```

在新建内存库中，第一轮显示两条 Python 笔记，完成第一条后只剩 pathlib，第一条笔记有创建和完成两条事件。

具体时间值每次不同，不要把它们硬编码进测试期望。

### 23.12 如果第二次再次标记第一条完成呢？

当前状态已是完成，或者版本已变化，条件 UPDATE 不再匹配，函数抛 `NoteConflictError`。

因此不会再无条件添加一条重复“完成”事件。

这比先查询、再没有条件地更新，更准确地表达状态转换。

## 第二十四章：数据库迁移——旧文件没有新字段，怎么办？

### 24.1 修改 CREATE TABLE 文本，不会自动修改旧库

你在代码里增加了 `revision`，旧数据库文件仍保存旧结构。

再次 `CREATE TABLE IF NOT EXISTS` 不会替你比较差异并迁移。

应建立明确版本与升级步骤。

### 24.2 一个最小的练习迁移

SQLite 的 `PRAGMA user_version` 可以存放应用自己使用的整数版本标记。

它不是数据库自动维护的业务模式版本。

```python

def migrate_learning_schema(connection):
    version = connection.execute("PRAGMA user_version").fetchone()[0]

    if version == 0:
        with connection:
            if not connection.in_transaction:
                connection.execute("BEGIN")
            connection.execute("""
                CREATE TABLE learning_items (
                    id INTEGER PRIMARY KEY,
                    title TEXT NOT NULL
                )
            """)
            connection.execute("PRAGMA user_version = 1")
        version = 1

    if version == 1:
        with connection:
            if not connection.in_transaction:
                connection.execute("BEGIN")
            connection.execute(
                "ALTER TABLE learning_items ADD COLUMN completed INTEGER NOT NULL DEFAULT 0"
            )
            connection.execute("PRAGMA user_version = 2")
        version = 2

    if version != 2:
        raise ValueError(f"不支持的数据库模式版本：{version}")
```

本例只用于一个专门的新建练习库，不用于任意不明历史数据库。

### 24.3 为什么版本与结构修改放同一事务？

如果先改版本，后改表失败，数据库可能自称已经升级，却没有相应结构。

把适用的结构修改与版本记录放在同一事务内，更容易保持一致。

这里在旧式模式下显式 `BEGIN`，因为 CREATE/ALTER 本身不保证触发 Python 的旧式隐式事务；若现代模式已经处于事务中，就不重复 BEGIN。

此函数只在自己拥有事务边界的入口调用，不能把其他尚未提交的业务修改一起带进来。每一版迁移单独提交，因此第一版成功、第二版失败时会停在版本 1，可按明确步骤重试。

真实迁移还要验证目标 SQLite 对相应 DDL 的事务和 ALTER 支持细节。

### 24.4 数据迁移比加一列更复杂

字段改名、拆表、唯一性收紧、清理坏数据，都可能需要明确转换步骤。

已有数据可能不满足新约束，不能只写一条新建表语句就认定迁移完成。

应先在备份副本或临时样本上验证，保留恢复方案，并确认所有连接和进程的版本配合。

### 24.5 失败后能不能直接删除数据库重来？

练习内存库可以重建。

真实数据不能因为模式不匹配就随意删除，或者把本地空库覆盖线上文件。

迁移的目标是保存并转换已有信息，不是把问题变成一份没有历史的新文件。

## 第二十五章：CSV 导入与导出，怎样把文件和事务连接起来？

### 25.1 先定义输入契约

CSV 三列：

```csv
category,title,completed
Python,open 文件模式,0
Python,pathlib 路径,1
Linux,进程与文件描述符,0
```

分类按名称查找或建立，笔记标题不能为空，完成状态只能是零或一。

一次导入要求全部成功或全部回滚。

### 25.2 导入函数

```python
import csv


def import_notes_csv(connection, path):
    with open(path, encoding="utf-8-sig", newline="") as source:
        reader = csv.DictReader(source)
        if reader.fieldnames != ["category", "title", "completed"]:
            raise ValueError("CSV 标题不符合导入格式")

        records = []
        for number, row in enumerate(reader, start=1):
            if None in row or any(value is None for value in row.values()):
                raise ValueError(f"第 {number} 条记录列数错误")
            category = row["category"].strip()
            title = row["title"].strip()
            completed_text = row["completed"].strip()
            if not category or not title:
                raise ValueError(f"第 {number} 条记录分类和标题不能为空")
            if completed_text not in {"0", "1"}:
                raise ValueError(f"第 {number} 条记录 completed 必须是 0 或 1")
            records.append((category, title, int(completed_text)))

    now = utc_now_text()

    with connection:
        for category, title, completed in records:
            connection.execute(
                "INSERT INTO categories(name) VALUES (?) ON CONFLICT(name) DO NOTHING",
                (category,),
            )
            category_id = connection.execute(
                "SELECT id FROM categories WHERE name = ?",
                (category,),
            ).fetchone()[0]
            cursor = connection.execute(
                """
                INSERT INTO notes(category_id, title, completed, updated_at)
                VALUES (?, ?, ?, ?)
                """,
                (category_id, title, completed, now),
            )
            connection.execute(
                "INSERT INTO note_events(note_id, message, created_at) VALUES (?, ?, ?)",
                (cursor.lastrowid, "CSV 导入", now),
            )

    return len(records)
```

`ON CONFLICT ... DO NOTHING` 属于 SQLite UPSERT 语法家族，需要 SQLite 3.24.0+。

本机验证引擎支持，但部署仍应检查版本。

### 25.3 为什么先读取和验证，再进入写事务？

小规模导入中，这让文件读取和解析不占用长写事务。

格式错误在修改数据库前就暴露。

代价是保存全部待导入记录到内存。

大文件可以分批验证和写入，但如果仍要求整体原子导入，就要设计更适合规模的暂存和事务安排。

### 25.4 为什么不在循环里调用 create_note()？

前面的 `create_note()` 自己用 `with connection` 提交一个业务操作。

如果外层又希望整份导入是一个事务，内部每条函数都提交，就会破坏整体边界。

所以这里把同一组受控 SQL 放在统一外层事务中。

真实项目可以把“不提交的底层数据库操作”与“管理事务的业务入口”分开，减少重复代码，同时保持责任清楚。

### 25.5 导出函数

```python
import csv


def export_notes_csv(connection, path):
    cursor = connection.execute("""
        SELECT c.name AS category, n.title, n.completed
        FROM notes AS n
        JOIN categories AS c ON c.id = n.category_id
        ORDER BY c.name, n.id
    """)

    with open(path, "x", encoding="utf-8", newline="") as target:
        writer = csv.DictWriter(
            target,
            fieldnames=["category", "title", "completed"],
            lineterminator="\n",
        )
        writer.writeheader()
        for row in cursor:
            writer.writerow(dict(row))
```

输出使用 `x` 拒绝覆盖已有文件。

如果导出过程中失败，可能留下部分 CSV，应按发布要求选择临时文件替换或任务目录策略。

数据库读事务不会替你撤销已经写到外部文件的字节。

## 第二十六章：执行失败时，应该向用户解释什么？

### 26.1 先分清“输入不合适”和“数据库无法工作”

假设你准备新增分类 `Python`，而数据库里已经有这个分类。

因为 `categories.name` 有唯一约束，重复插入会触发 `sqlite3.IntegrityError`。

这时向用户提示“分类已经存在”比较有帮助。

但如果数据库文件所在磁盘无法写入，提示同样的话就会误导用户。

不要把所有数据库异常都翻译成同一个业务原因。

```python
connection = open_learning_database()
try:
    create_notes_schema(connection)
    create_category(connection, "Python")

    try:
        create_category(connection, "Python")
    except sqlite3.IntegrityError as error:
        print("分类没有新增，请检查是否重名。")
        print(type(error).__name__)
finally:
    connection.close()
```

在本篇定义的表结构中，输出为：

```text
分类没有新增，请检查是否重名。
IntegrityError
```

这里为什么没有直接写“百分之百是重名”？

因为 `IntegrityError` 包含的不只是 UNIQUE，还可能包括 NOT NULL、CHECK、外键等约束失败。

本例先做了名称验证，SQL 也很简单，所以重复分类是可明确判断的情境。

但函数以后增加新字段和新约束，捕获范围就要一起检查。

### 26.2 几种常见异常，分别让你检查哪里？

`sqlite3.OperationalError` 经常与运行条件有关，例如数据库被锁、某些路径无法打开、SQL 运行时失败。

它也可能包含 SQL 拼写问题，例如表根本不存在。

所以名字里的 Operational 不等于“重试就好”。

`sqlite3.ProgrammingError` 经常表示接口使用不对，例如占位符数量与参数不匹配、使用已经关闭的连接。

`sqlite3.IntegrityError` 表示违反关系完整性约束。

`sqlite3.DatabaseError` 是更广的数据库错误类别。

如果你只想在命令行最外层把数据库错误统一转换成友好提示，可以捕获它；但内层仍应保留真正的异常上下文。

```python
try:
    connection.execute("SELECT ? + ?", (1,))
except sqlite3.ProgrammingError as error:
    print("SQL 占位符和参数数量不一致。")
    print(type(error).__name__)
```

这是故意失败的练习片段，要求 `connection` 还没关闭。

预期输出：

```text
SQL 占位符和参数数量不一致。
ProgrammingError
```

这里的修正方向是把参数补为 `(1, 2)`，而不是休眠一秒再执行同样的错误代码。

### 26.3 能不能依靠异常消息里的英文关键词判断？

不建议把消息字符串当成稳定协议。

不同 SQLite 版本、语句和失败位置，消息可能不同。

Python 3.11 为来自 SQLite 库的错误提供了 `sqlite_errorcode` 与 `sqlite_errorname` 属性。

可以在具体约束实验中查看：

```python
connection = open_learning_database()
try:
    connection.execute("CREATE TABLE tags(name TEXT UNIQUE NOT NULL)")
    connection.commit()

    try:
        with connection:
            connection.execute("INSERT INTO tags VALUES (?)", ("Python",))
            connection.execute("INSERT INTO tags VALUES (?)", ("Python",))
    except sqlite3.IntegrityError as error:
        print(error.sqlite_errorname)

    print(connection.execute("SELECT COUNT(*) FROM tags").fetchone()[0])
finally:
    connection.close()
```

本篇验证环境输出：

```text
SQLITE_CONSTRAINT_UNIQUE
0
```

第一条 INSERT 虽然先执行成功，但第二条失败使异常离开 `with`，事务整体回滚。

所以计数是 `0`。

如果错误由 Python 接口检查直接产生，而不是来自 SQLite 库，不应假设一定存在上述属性。

写通用诊断时可以用 `getattr(error, "sqlite_errorname", None)`。

错误码能帮助分类，但仍要结合语句的业务含义，不要对所有约束错误做同样的自动处理。[Python 的错误说明](https://docs.python.org/3.14/library/sqlite3.html#exceptions)

### 26.4 数据库被锁时，为什么不能无条件重跑最后一条 SQL？

假设操作包含三步：

先读取当前 revision，再更新笔记，最后插入事件。

如果中间遇到并发冲突，你只重新执行最后一条 SQL，可能脱离原来读取和判断的条件。

正确的重试单位通常应是一个有明确边界的业务事务。

重试前还要确认上次事务已经结束、操作是否允许重复、是否会重复外部通知。

不要把数据库忙、唯一约束失败和参数写错统统放进无限重试循环。

本篇工具在冲突时直接给调用者明确异常，让上层重新加载记录，是容易理解的初始策略。

### 26.5 调试时想看到执行过哪些 SQL 呢？

`set_trace_callback()` 可以注册一个跟踪函数。

```python
connection = open_learning_database()
try:
    connection.set_trace_callback(print)
    with connection:
        connection.execute("CREATE TABLE trace_demo(value TEXT)")
        connection.execute(
            "INSERT INTO trace_demo(value) VALUES (?)",
            ("hello",),
        )
    connection.set_trace_callback(None)
finally:
    connection.close()
```

你通常能看到建表、事务语句和插入语句。

具体行的格式及引擎内部触发的语句不要作为测试的固定输出。

特别注意：跟踪内容可能包含实际绑定的数据。

如果记录中有口令、令牌或个人信息，直接把所有 SQL 打进共享日志就不合适。

这个接口便于理解“Python 是否发出了 BEGIN”，不是让你永久打印所有生产数据。

清理跟踪函数用 `None`。

调试开关也不应变成工具默认面向普通用户的输出。

## 第二十七章：`executescript()` 看起来更短，为什么不能随便替代 execute？

### 27.1 一个方法能执行多条语句

前面建表函数把固定 SQL 存在列表中，逐条 `execute()`。

你可能会问：

“把这些 SQL 连成一大段，再调用 `executescript()`，不是更省事吗？”

它确实支持执行脚本中的多条 SQL。

例如下面的独立内存练习：

```python
import sqlite3

connection = sqlite3.connect(":memory:", isolation_level=None)
try:
    connection.executescript("""
        CREATE TABLE colors(name TEXT NOT NULL);
        INSERT INTO colors VALUES ('blue');
        INSERT INTO colors VALUES ('green');
    """)
    rows = connection.execute("SELECT name FROM colors ORDER BY name").fetchall()
    print(rows)
finally:
    connection.close()
```

输出：

```text
[('blue',), ('green',)]
```

但这里没有外部输入，也没有声称两次 INSERT 自动组成一个业务事务。

### 27.2 `executescript()` 不接受单独绑定参数

它不像 `execute(sql, parameters)` 那样接收值参数。

如果需要插入用户给出的标题，仍然用参数化 `execute()`。

不要为了使用脚本接口，把标题转回 f-string 拼进脚本文本。

这里“脚本”指 SQL 文本，不代表它能理解 Python 的变量或自动保护字符串。

### 27.3 旧事务控制下，它可能先提交已有事务

在 Python 的旧事务控制模式中，如果已有待提交事务，`executescript()` 会先隐式提交。

因此，把它放在 `with connection` 里面，不自动保证你想象中的整个块原子性。

下面用一个故意失败的例子看结果：

```python
import sqlite3

connection = sqlite3.connect(":memory:")
try:
    connection.execute("CREATE TABLE script_demo(value INTEGER UNIQUE)")

    try:
        with connection:
            connection.execute("INSERT INTO script_demo VALUES (1)")
            connection.executescript("""
                INSERT INTO script_demo VALUES (2);
                INSERT INTO script_demo VALUES (2);
            """)
    except sqlite3.IntegrityError:
        print("脚本中途失败")

    values = connection.execute(
        "SELECT value FROM script_demo ORDER BY value"
    ).fetchall()
    print(values)
finally:
    connection.close()
```

在这里采用的旧模式下，输出：

```text
脚本中途失败
[(1,), (2,)]
```

你原以为失败后都消失，实际 `1` 在执行脚本前已经提交，脚本里的第一条 `2` 也已单独完成。

这个例子只创建内存数据，可以安心观察失败结果。

### 27.4 如果确实要执行一份受控迁移脚本呢？

需要清楚写出脚本自己的 BEGIN、COMMIT，并在失败时回滚残留事务，同时考虑调用前是否允许已有事务。

不能只是背一句“executescript 会自动处理事务”。

对于本篇这种几十条以内、需要和 Python 校验配合的数据库操作，逐条参数化执行、明确一个事务入口，通常更容易审查。

现代 `autocommit` 模式又有自己的边界，不应把上面旧模式的“先提交”行为当成所有模式的通用规则。[脚本接口与事务控制](https://docs.python.org/3.14/library/sqlite3.html#sqlite3.Cursor.executescript)

## 第二十八章：真实文件实践——提交、重开、备份和只读检查连起来

### 28.1 为什么还要做文件实验？

内存库能讲清 SQL，但看不到“关闭以后数据还在不在”。

你也还没验证备份出来的东西能否被另一个连接读取。

下面创建一个自动清理的临时目录。

目录里面只有本次练习新建的文件，不要求你准备任何业务库。

本节依赖第三章 `open_learning_database()` 和第二十三章的建表、新增函数，请先把那些函数放在同一个脚本中。

```python
from pathlib import Path
from tempfile import TemporaryDirectory
import sqlite3

with TemporaryDirectory(prefix="sqlite-notes-") as directory:
    root = Path(directory)
    database_path = root / "notes.db"
    backup_path = root / "notes-backup.db"

    connection = open_learning_database(database_path)
    try:
        create_notes_schema(connection)
        category_id = create_category(connection, "Python")
        create_note(connection, category_id, "读懂事务")
    finally:
        connection.close()

    source = sqlite3.connect(database_path, isolation_level=None)
    target = sqlite3.connect(backup_path, isolation_level=None)
    try:
        source.backup(target)
    finally:
        target.close()
        source.close()

    readonly_uri = backup_path.resolve().as_uri() + "?mode=ro"
    check = sqlite3.connect(readonly_uri, uri=True, isolation_level=None)
    try:
        title = check.execute("SELECT title FROM notes").fetchone()[0]
        result = check.execute("PRAGMA integrity_check").fetchone()[0]
        print(title)
        print(result)
    finally:
        check.close()
```

输出：

```text
读懂事务
ok
```

### 28.2 第一处关闭为什么不会丢数据？

因为 `create_category()`、`create_note()` 已经在自己的事务边界正常退出并提交。

关闭只结束连接，不负责替它们提交。

如果你删掉业务函数中的提交结构，让 INSERT 留在未提交事务里，关闭后再打开，就可能看不到相应记录。

这也是“终端打印了添加成功”不能单独证明数据持久化的原因。

### 28.3 为什么备份另外打开 source？

这样很容易看出：备份读取的是已提交的文件状态，而不是依赖同一个连接里尚未处理的修改。

`isolation_level=None` 使这里的备份连接不额外维护 Python 的旧式隐式写事务。

并不是说备份只能这样连接，而是本例主动减少待提交事务带来的混淆。

对已经处于写事务的源连接盲目调用备份，可能遇到等待和忙状态；不要把备份当成一次顺手复制函数。

### 28.4 `integrity_check` 返回 ok，是不是已经证明备份完美？

它说明引擎的这项完整性检查没有发现相应存储结构问题。

但它不证明业务记录完整，也不替代外键检查。

你还应核对预期的表、记录数和关键关系。

外键检查可以独立执行：

```python
problems = check.execute("PRAGMA foreign_key_check").fetchall()
print(problems)
```

将这段放到前一个例子的 `check` 仍然打开时执行。

正确练习库预期输出：

```text
[]
```

没有违规行返回。

如果返回非空结果，要按官方定义理解表名、行标识、父表与约束编号，不要仅把消息忽略掉。

### 28.5 为什么最后不能再用 database_path 打开库？

退出 `TemporaryDirectory` 后，临时目录会被清理。

变量 `database_path` 仍是一个路径对象，但它描述的位置已经不保证存在。

路径对象保存的是位置表达，不是数据库内容的快照。

把这种生命周期也想清楚，文件与数据库代码就不容易混乱。

## 第二十九章：两个人同时修改时，亲眼看一次锁

### 29.1 不用线程，也能用两个连接模拟

下面程序只在临时目录运行，超时设为很短的 `0.05` 秒。

第一条连接先写入而不提交。

第二条连接尝试写入，观察到它不能同时成为另一个写者。

```python
from pathlib import Path
from tempfile import TemporaryDirectory
import sqlite3

with TemporaryDirectory(prefix="sqlite-lock-") as directory:
    path = Path(directory) / "lock.db"
    first = sqlite3.connect(path, isolation_level=None, timeout=0.05)
    second = sqlite3.connect(path, isolation_level=None, timeout=0.05)
    try:
        first.execute("PRAGMA journal_mode = WAL")
        first.execute("CREATE TABLE tasks(title TEXT NOT NULL)")
        first.execute("BEGIN IMMEDIATE")
        first.execute("INSERT INTO tasks VALUES (?)", ("first",))

        try:
            second.execute("INSERT INTO tasks VALUES (?)", ("second",))
        except sqlite3.OperationalError as error:
            print(type(error).__name__)

        first.execute("ROLLBACK")
        second.execute("INSERT INTO tasks VALUES (?)", ("second",))
        print(second.execute("SELECT title FROM tasks").fetchall())
    finally:
        second.close()
        first.close()
```

在本篇本地 SQLite 环境中输出：

```text
OperationalError
[('second',)]
```

### 29.2 为什么开了 WAL 还是失败？

因为 WAL 改善的是读写并发组织，并没有允许多个连接同时持有写事务。

第一条连接还没结束写事务，第二条连接就要等待。

等待超过本例很短的忙等待期限后，操作失败。

这和你的 SQL 占位符有没有写对是两回事。

### 29.3 回滚第一条后，为什么 second 能写了？

第一条连接的事务结束，释放相应写入占用。

第二条连接重做这次独立 INSERT，能完成。

本例中第一条 `first` 已经回滚，因此最终只剩 `second`。

注意：这里重试的是已知尚未成功的单条独立插入，不能直接推广到包含网络请求的多步业务。

### 29.4 为什么不直接让每个连接等待一分钟？

较长 timeout 有时合理，但它不会消除事务设计问题。

如果一个连接拿着写事务等待用户输入，另一个连接等一分钟只是让用户更晚收到错误。

更有效的起点是缩短事务，把输入、文件解析和耗时外部操作放到事务之外。

真正需要排队写入的工具，可以让一个专门写入者管理任务队列；但这又涉及队列容量、失败反馈和关闭流程，不是改一个连接参数就结束。

### 29.5 读取快照为什么可能“还看不见新记录”？

一个已经建立读快照的事务，会继续看到它所对应的状态。

别的连接提交之后，它不一定立刻刷新成最新视图。

结束读事务，开始下一次读取，才会进入新的可见性范围。

Python 3.12 的 `autocommit=False` 会保持事务管理，所以长时间读连接也要留意何时 `commit()` 或 `rollback()` 来结束当前快照。

基础旧模式的 SELECT 又不完全按相同方式开启 Python 隐式事务；判断时要看实际事务和语句生命周期。[SQLite 事务与快照](https://www.sqlite.org/lang_transaction.html)

## 第三十章：练习题——让每一个边界都变成你能检查的结果

下面练习仍然只使用本篇内存库或新临时目录。

建议先写自己的代码，观察输出，再看答案。

练习不是背 API 名字，而是回答“这一步失败之后，数据应该处于什么状态”。

### 30.1 练习：标题里有单引号和 SQL 文本

新增标题：

```text
Alice's note; DROP TABLE notes;
```

要求它作为普通标题保存，随后 `notes` 表仍可查询。

参考答案，依赖第二十三章函数：

```python
connection = open_learning_database()
try:
    create_notes_schema(connection)
    category_id = create_category(connection, "Python")
    title = "Alice's note; DROP TABLE notes;"
    note_id = create_note(connection, category_id, title)
    row = connection.execute(
        "SELECT title FROM notes WHERE id = ?", (note_id,)
    ).fetchone()
    print(row["title"] == title)
    print(connection.execute("SELECT COUNT(*) FROM notes").fetchone()[0])
finally:
    connection.close()
```

输出：

```text
True
1
```

关键不是先把分号删掉，而是 SQL 结构和值始终分开。

如果业务不允许某些字符，输入规则可以另行限制，但输入过滤不代替参数化。

### 30.2 练习：没有这一行，和这一行的值是 NULL

创建一张允许空值的练习表。

查询已有的空值，再查询不存在的编号，分别判断结果。

```python
connection = open_learning_database()
try:
    with connection:
        connection.execute("CREATE TABLE optional_values(id INTEGER PRIMARY KEY, value TEXT)")
        connection.execute("INSERT INTO optional_values(id, value) VALUES (?, ?)", (1, None))

    existing = connection.execute(
        "SELECT value FROM optional_values WHERE id = ?", (1,)
    ).fetchone()
    missing = connection.execute(
        "SELECT value FROM optional_values WHERE id = ?", (2,)
    ).fetchone()
    print(existing is None)
    print(existing["value"] is None)
    print(missing is None)
finally:
    connection.close()
```

输出：

```text
False
True
True
```

第一行查询到了一个 Row，行里的 value 是 `None`。

第二行查询根本没有找到记录，`fetchone()` 自身返回 `None`。

如果只写 `row["value"]` 而不判断 row 是否存在，就可能把“没记录”的情况变成 Python 下标错误。

### 30.3 练习：相同编号，读到旧 revision 之后怎么办？

先读取 revision 为 `1` 的笔记。

第一次完成成功，再使用同一个旧 revision 完成一次。

要求第二次抛出 `NoteConflictError`，事件总数仍然是 `2`，不是 `3`。

```python
connection = open_learning_database()
try:
    create_notes_schema(connection)
    category_id = create_category(connection, "Python")
    note_id = create_note(connection, category_id, "测试旧版本")
    complete_note(connection, note_id, 1)

    try:
        complete_note(connection, note_id, 1)
    except NoteConflictError:
        print("请重新加载记录")

    count = connection.execute(
        "SELECT COUNT(*) FROM note_events WHERE note_id = ?", (note_id,)
    ).fetchone()[0]
    print(count)
finally:
    connection.close()
```

输出：

```text
请重新加载记录
2
```

第一条事件是创建笔记，第二条是首次完成。

第二次条件 UPDATE 没有命中，异常使这个事务退出，因而不会新增一次完成事件。

这才是有意义的验收结果：既检查异常，也检查数据没有多出来。

### 30.4 练习：导入第三行失败，前两行能留下吗？

要求整份文件导入要么全部成功，要么全部回滚。

输入前两行格式正确，第三行与第一行的“分类 + 标题”重复。

准备以下 CSV 内容，写到新临时目录再调用 `import_notes_csv()`：

```csv
category,title,completed
Python,第一条,0
Linux,第二条,1
Python,第一条,0
```

参考检查代码，依赖前文导入函数：

```python
from pathlib import Path
from tempfile import TemporaryDirectory
import sqlite3

connection = open_learning_database()
try:
    create_notes_schema(connection)
    with TemporaryDirectory(prefix="sqlite-import-") as directory:
        path = Path(directory) / "notes.csv"
        path.write_text(
            "category,title,completed\nPython,第一条,0\nLinux,第二条,1\nPython,第一条,0\n",
            encoding="utf-8",
        )
        try:
            import_notes_csv(connection, path)
        except sqlite3.IntegrityError:
            print("导入已回滚")

    for table in ("categories", "notes", "note_events"):
        count = connection.execute(f"SELECT COUNT(*) FROM {table}").fetchone()[0]
        print(table, count)
finally:
    connection.close()
```

输出：

```text
导入已回滚
categories 0
notes 0
note_events 0
```

这里 f-string 的表名来自代码内固定元组，不来自外部输入。

分类、笔记和事件放在同一个事务里，所以前两条已经执行过的插入也被撤销。

如果你把 `with connection` 挪到循环里面，就会每行提交一次，第三行失败时前两行可能已经留下。

改动仅有几行，业务语义却完全不同。

### 30.5 练习：外键是否真的打开？

不要只检查建表文本包含 FOREIGN KEY。

先查询每个实际连接的外键开关，再尝试写入不存在的分类编号。

参考答案：

```python
connection = open_learning_database()
try:
    create_notes_schema(connection)
    print(connection.execute("PRAGMA foreign_keys").fetchone()[0])
    try:
        create_note(connection, 999, "不存在的分类")
    except sqlite3.IntegrityError:
        print("无效分类被拒绝")
    print(connection.execute("SELECT COUNT(*) FROM notes").fetchone()[0])
finally:
    connection.close()
```

输出：

```text
1
无效分类被拒绝
0
```

第三章把开关放在事务外设置，就是为了这个结果。

配置是否写过，和当前连接配置是否生效，是两个要分别验证的问题。

### 30.6 练习：为什么 with 结束以后还能查询？

写一个上下文块，里面插入一行。

正常退出后查询，最后再显式关闭。

参考答案：

```python
connection = open_learning_database()
try:
    with connection:
        connection.execute("CREATE TABLE lifecycle(value INTEGER)")
        connection.execute("INSERT INTO lifecycle VALUES (7)")
    print(connection.execute("SELECT value FROM lifecycle").fetchone()[0])
finally:
    connection.close()
```

输出：

```text
7
```

这说明事务上下文已经结束，但连接仍然可用。

如果要让作用域退出时自动关闭，应再结合 `contextlib.closing()`，并把“结束事务”和“关闭资源”看成两个动作。

### 30.7 练习：把成功判断写成真正的断言

不要只运行一次然后说“没有报错，所以工具正确”。

对新增、完成、重复完成这条流程，至少检查：

新笔记的 completed 是 `0`。

首次完成以后 completed 是 `1`，revision 是 `2`。

重复完成失败，事件计数没有增加。

数据库关闭再打开，已经提交的结果仍能读到。

备份能读到相同关键记录，外键检查没有违规。

这些观察共同覆盖业务状态、事务和持久化；任何一个单独的 `print("ok")` 都不能替代它们。

可以把前面练习改为 `assert`，作为个人工具最初的一组可重复验证。

记得断言表达的是你要求的行为，不能只是照抄函数内部实现。

## 第三十一章：继续写自己的工具时，下一步应该怎么安排？

### 31.1 我需要把 SQL 全部封装成类吗？

不必一开始就这么做。

前面的函数已经可以把输入验证、数据库操作和输出分开。

当连接配置、生命周期和依赖注入需要稳定组织时，再考虑一个轻量的仓库对象或上下文管理器。

无论用函数还是类，事务归谁管理、连接由谁关闭，都必须清楚。

复杂类结构不能代替清楚的数据边界。

### 31.2 我能把所有异常都藏在数据库层吗？

数据库层可以转换你真正理解的业务冲突。

例如条件更新没命中，可以抛 `NoteConflictError`。

但磁盘无法访问、模式不兼容、SQL 写错等问题，不应都变成“请换一个标题”。

最外层命令行负责把可预期错误显示成人能理解的话，并设置恰当退出码；调试日志保留诊断信息。

让用户知道可以怎样处理，和让开发者知道具体哪里失败，都有价值。

### 31.3 我需要默认开启 WAL、STRICT 和所有新功能吗？

先根据实际目标决定。

WAL 要考虑文件系统和连接生命周期。

STRICT 要考虑底层引擎版本与历史数据。

较新 SQL 可以让代码简洁，但要写清最低引擎版本。

一个可靠的小工具，首先应有可说明的结构、清楚的输入和可验证的事务；功能开关越多不自动越可靠。

### 31.4 如果将来换 PostgreSQL，还能用这些知识吗？

连接、参数化、事务、约束、索引和并发更新的思考仍然重要。

但占位符写法、事务行为、类型系统、SQL 方言和部署方式会变化。

不能把 SQLite 的 `PRAGMA`、`INTEGER PRIMARY KEY` 和锁行为照搬过去。

能迁移的核心能力，是看到一段业务代码时，主动问出：输入是什么，关系由谁保证，失败后哪些改变必须一起撤销，其他连接会看到什么。

### 31.5 遇到版本疑问，具体查哪里？

Python 接口以 [Python 3.11 sqlite3 文档](https://docs.python.org/3.11/library/sqlite3.html)核对基础接口，以 [Python 3.14 sqlite3 参考](https://docs.python.org/3.14/library/sqlite3.html)核对新增 API 和版本说明。

两种事务控制的差异查 [Python 的 transaction control 解释](https://docs.python.org/3.14/library/sqlite3.html#transaction-control)。

SQL 的事务和保存点查 [SQLite 事务文档](https://www.sqlite.org/lang_transaction.html)与 [SAVEPOINT 文档](https://www.sqlite.org/lang_savepoint.html)。

外键是否启用、约束何时检查，查 [SQLite 外键说明](https://www.sqlite.org/foreignkeys.html)。

WAL 的并发与文件要求，查 [SQLite WAL 文档](https://www.sqlite.org/wal.html)。

查询计划的含义与输出不稳定性，查 [SQLite EXPLAIN QUERY PLAN 说明](https://www.sqlite.org/eqp.html)。

备份的能力和使用边界，查 [SQLite 在线备份接口说明](https://www.sqlite.org/backup.html)。

本篇实际运行环境为 Python 3.12.3、SQLite 3.45.1；基础连接选择 3.11 已提供的接口，并在当前解释器验证旧式控制分支。没有把“语法兼容 3.11”写成“在 3.11 解释器实际运行过”。

现代 autocommit 演示在当前 3.12 环境验证；查询计划、跟踪 SQL、错误消息和临时目录路径不保证在其他版本逐字相同。

回到你最初的笔记脚本，现在可以沿着一个完整链条理解它：打开受控位置的练习库，启用关系规则，校验输入，绑定参数，执行一组必须共同成功的修改，提交，关闭，并在另一次连接中验证数据确实存在。

当一个边界不满足预期时，先定位是哪一层，再修改那一层的规则。
