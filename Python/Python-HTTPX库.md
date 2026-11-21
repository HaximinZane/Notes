# Python HTTPX 库超详细教程：从一次请求，到同步客户端、异步并发与可靠的数据处理

## 零基础预备课：先彻底搞懂 HTTPX 的位置，再开始学函数

> 如果你现在连“为什么 Python 要发送 HTTP 请求”都还不确定，请从这里读起。**第一次阅读完全不需要先懂协程、`await`、连接池、`MockTransport` 或 HTTP/2。** 它们是为更复杂的需求准备的工具，而不是使用 HTTPX 的入场券。

### 0.1 假设你要写一个“自动查询图书信息”的程序

一开始，你把图书列表写在自己电脑的 `books.json` 中，用 `open()` 和 `json.load()` 读取，就足够了。

但你发现图书信息经常更新，而最新数据保存在图书服务提供方的服务器上。如果每天打开浏览器、查找、复制、粘贴，那还算什么自动化？

你真正想做的是：

```text
本地 Python 脚本（客户端）
    │ 说：“请给我编号 3 的图书信息”
    ▼
网络 / HTTP 请求
    ▼
图书服务（服务器）
    │ 检查路径、权限、参数，然后返回数据或错误
    ▼
HTTP 响应
    ▼
本地 Python 脚本：读出状态码、解析内容、保存结果
```

**HTTPX 就站在“本地 Python 脚本”和“远端 HTTP 服务”之间，替你完成发送请求、接收响应等客户端工作。**

它不负责设计图书网站，也不会自动知道“哪一本书打折”；它只是让你可以按照服务端公开的 API 规则去请求和处理数据。

### 0.2 HTTPX 到底是什么名字？与 HTTP、HTTPS、API 是什么关系？

- **HTTP**：Hypertext Transfer Protocol，超文本传输协议。可以理解成客户端和服务器如何组织“请求/响应”的通用规则。
- **HTTPS**：通常表示 HTTP 在 TLS 加密通道上传输，保护传输中的数据；它不保证接口内容一定可靠。
- **API**：Application Programming Interface，应用程序编程接口。这里通常指某个服务规定好的网络调用方式，比如“GET /books/3 会返回编号 3 的图书”。
- **HTTPX**：一个 Python **第三方 HTTP 客户端库**的项目名称；名字里的 HTTP 表明处理 HTTP 请求，`X` 是名称的一部分，**不代表一个新的、叫 X 的网络协议**。它不是 Python 自带的标准库，需安装。

你可以把它与 `os` 对照：

| 你想做的事 | 应该先想到 |
|---|---|
| 列出本地目录里的文件 | `os` / `pathlib` |
| 打开电脑里的 JSON 文件 | `open()` + `json` |
| 去一个网站或 API 请求最新数据 | `httpx`（或者 `requests`） |
| 将收到的合法 JSON 数据处理为 Python 字典 | `response.json()`；必要时也可以结合 `json` |
| 让别人访问你写的 Web 服务 | 服务端框架/服务器；**不是 HTTPX 的主要用途** |

**为什么已经有 Requests，还出现了 HTTPX？** Python 有不同的网络程序：有些按顺序请求，一个完成后再请求下一个；有些本来就使用 `asyncio`，需要同时管理很多等待中的网络任务。HTTPX 提供**同步和异步两种接口**，也有连接池、可配置超时、流式处理、可选 HTTP/2 等特性。它不是“Requests 已经不能用”的证据；小型同步脚本两者通常都可以。

### 0.3 先看懂一次 HTTP 请求和回应

比如：

```text
请求：
方法：GET
URL：https://example.com/books/3
请求头：Accept: application/json
请求体：无

响应：
状态码：200
响应头：Content-Type: application/json
响应体：{"id": 3, "title": "Python"}
```

`GET` 常用于读取，`POST` 常用于提交；`URL` 负责告诉程序访问哪里；`Headers` 是额外的协议元信息；`Body` 是传输的数据；`Response` 是响应对象。

你现在只要明白：**“发请求”和“读响应”是两步**。它并不意味着你拿到的所有正文都是 JSON——服务器也可能返回 HTML、图片、空正文或错误页面。

### 0.4 真正的第一个 HTTPX 程序，先只写四行

安装：

```bash
python -m pip install httpx
```

然后创建 `first_httpx.py`：

```python
import httpx

url = "https://example.com/"           # ① 字符串形式的目标地址
response = httpx.get(url, timeout=10)   # ② 此时才发起真正的 HTTP 请求
print(response.status_code)             # ③ 看服务器返回的状态码
print(response.text[:100])              # ④ 看正文前 100 个字符
```

逐句看：

1. `import httpx` 只是导入库，不是联网。它是第三方库，所以要在运行脚本的 Python 环境里安装。
2. `url = ...` 只是保存 URL。域名通常需要先经解析，程序才能连接到目标网络服务。
3. `httpx.get(...)` 才是发送 GET 请求的那一步。`timeout` 为网络等待设置限制，`response` 得到的是 `httpx.Response` 对象，而不是已经解析好的字典。
4. `response.status_code` 是**整数属性**，所以没有 `()`；`response.text` 是**字符串属性**。因为演示站点返回 HTML，所以这里只截取一小段文本，不要盲目调用 `.json()`。

此示例需要联网。网站不可用、DNS 失败、证书校验失败、代理配置等均可能导致请求异常。**没有拿到响应对象**与“拿到 404 响应”是两种不同失败：前者通常发生在连接/传输阶段，后者是服务器确实给出了 HTTP 响应。

### 0.5 我想要的不是网页 HTML，而是字典，怎么办？

先确认对方有一个返回 JSON 的接口，且你知道它约定的 URL、方法、参数、权限。设想接口返回：

```json
{"id": 3, "title": "Python 入门", "price": 39}
```

JSON 是网络交换用的一种文本格式；Python 字典是你程序里的数据对象。这两个不能混为一谈。

```python
# 以下仅示意接口写法；example.invalid 不是可连接的真实 API
response = httpx.get("https://books.example.invalid/books/3", timeout=10)
response.raise_for_status()  # HTTP 错误状态时抛异常
book = response.json()       # 如果正文是合法 JSON，解析为 Python 对象
print(book["title"])         # 取字典中的 title 字段
```

请注意，上面用的是 `.invalid` **不可联网域名**，目的只是解释“请求 → 状态检查 → JSON 解析 → 读取字段”。**不能直接运行来获取图书**。真正能离线运行的模拟例子在 `0.9`。

不要混淆这些接口：

| 表达式 | 返回/作用 | 使用场景 |
|---|---|---|
| `response.status_code` | HTTP 状态码整数 | 看服务器返回了什么状态 |
| `response.text` | 按字符编码解码的正文字符串 | HTML、纯文本 |
| `response.content` | 正文字节 `bytes` | 图片、任意二进制 |
| `response.json()` | 尝试解析 JSON | 文档明确说返回 JSON 的 API |
| `response.raise_for_status()` | 对 HTTP 错误状态抛异常 | 别悄悄忽略 404/500 |

`response.json()` 不能替代 `raise_for_status()`。即使数据格式是合法 JSON，也可能是一条错误消息。反过来，HTTP 200 也不代表 JSON 里的业务字段一定满足你的要求。

### 0.6 `params=`、`json=`、`headers=`，不是一回事

你查询第二页图书，应该告诉服务器“page=2”。这通常是 URL 查询参数：

```python
response = httpx.get(
    "https://books.example.invalid/books",
    params={"page": 2, "page_size": 5},  # URL 查询条件
    timeout=10,
)
```

请求 URL 形如 `.../books?page=2&page_size=5`。库负责按照规则编码参数，通常比你手工拼字符串安全可靠。

如果你是**提交新图书记录**，服务可能要求 POST + JSON 请求体：

```python
book = {"title": "Python 入门", "price": 39}
response = httpx.post(
    "https://books.example.invalid/books",
    json=book,                # 把 Python 字典序列化为 JSON 请求体
    timeout=10,
)
```

如果接口要求特定请求头：

```python
headers = {"Accept": "application/json"}
response = httpx.get(
    "https://books.example.invalid/books",
    headers=headers,          # 附加 HTTP 请求头
    timeout=10,
)
```

这三个代码块都为**语法示意**，域名不提供真实 API。

- `params=`：请求 URL 查询字符串；经常是搜索、筛选、分页。
- `json=`：JSON 请求正文；用于服务端约定接受 JSON 的接口。
- `headers=`：请求头；可包含期望格式、认证方式等。
- 文件上传通常用 `files=`；普通表单则可能用 `data=`。

HTTPX 不负责规定“必须用什么 HTTP 方法提交哪一种数据”。对方 API 文档才决定请求是否合法。

### 0.7 同步和异步：先搞懂“为什么要有两个写法”

**同步**：发出请求，当前执行流程等结果返回后再往下走：

```python
response = httpx.get("https://example.com/", timeout=10)
print(response.status_code)
```

**异步**：如果你的程序已经使用 `asyncio` 组织许多任务，你可以在某个网络请求等待时，允许事件循环安排其他协程：

```python
import asyncio
import httpx

async def main():  # 定义协程函数：还没真正运行它
    async with httpx.AsyncClient(timeout=10) as client:  # 异步客户端
        response = await client.get("https://example.com/")  # 等待请求并允许调度其他任务
        print(response.status_code)  # 拿到结果后，继续执行

asyncio.run(main())  # 启动这段异步程序
```

逐句看：

- `async def` 定义协程函数；只写定义不会发请求。
- `asyncio.run(main())` 在普通脚本里创建并运行事件循环，驱动 `main()`。
- `async with` 用异步上下文管理方式创建/释放 `AsyncClient`，避免客户端资源泄露。
- `await` 不是“给函数加速的魔法”。它表示此处等待可等待对象，在等待过程中可让事件循环安排其他任务。
- **只写一个网络请求时，异步写法往往并不能明显更快**。异步主要帮助组织多个可能等待的网络任务；它不会替服务器处理任务，也不自动把 CPU 密集工作变成多核并行。
- 如果你在已经运行事件循环的 Notebook/异步框架内，不应再随手嵌套 `asyncio.run()`；应按该环境的协程调用方式使用。

如果现在看 `async def` 还头痛：**完全可以只学习同步 `httpx.get` 和 `httpx.Client`。**

### 0.8 `Client` 又是干什么的？为什么不用一直 `httpx.get`？

单次请求时，`httpx.get` 很方便。连续请求同一个站点时，每次都通过顶层函数建立独立调用会损失共享配置、连接复用等优势。因此 HTTPX 提供 `Client`：

```python
import httpx

with httpx.Client(base_url="https://example.com", timeout=10) as client:
    first = client.get("/")   # 自动与 base_url 合成目标 URL
    second = client.get("/")  # 同一个客户端上下文
    print(first.status_code, second.status_code)
```

这里 `client` 是一个对象，`get()` 是这个对象的方法；`with` 负责退出时关闭客户端资源。客户端可以维护连接池、共享 Cookie、默认请求头等；**但不保证某次连接一定被重用**，这还取决于服务器、连接状态等。

异步版名称是 `httpx.AsyncClient`，与 `Client` 不能仅靠把 `get` 前面随便加一个 `await` 就互换。必须选择与代码运行方式一致的接口。

### 0.9 不联网，也能练习真实的 HTTPX 调用流程吗？

可以。这里才引出 `MockTransport`（模拟传输层）。它**不是新的 HTTP 协议，也不代表你真的连到了服务器**。在测试里，它把“请求发给网络”换成“请求交给我们写的 Python 函数，再返回模拟响应”。

先分三个小步骤理解，不要一上来就看文后的一百行图书服务。

**第一小步：定义收到请求后怎么回答。**

```python
import httpx

def handler(request):
    # 不管谁来请求，先固定返回这份 JSON
    return httpx.Response(200, json={"message": "你好，HTTPX"})
```

为什么有 `request` 参数？因为处理函数将接收 HTTPX 准备好的请求对象。这个例子暂时不需要检查它的属性。

**第二小步：把这个处理函数交给模拟传输层。**

```python
transport = httpx.MockTransport(handler)
```

这行相当于告诉 HTTPX：“不要真的连互联网。以后这个客户端发送请求时，请把请求交给 `handler`。”

**第三小步：使用客户端，并读取结果。**

```python
with httpx.Client(transport=transport) as client:
    response = client.get("https://books.example.invalid/hello")
    print(response.status_code)       # 200
    print(response.json()["message"]) # 你好，HTTPX
```

这三段按照顺序放在**同一个 Python 文件**就能运行。地址中的 `example.invalid` 是刻意选择的不可联网域名，但此时不会发生真实网络连接，因为 `MockTransport` 拦截了传输流程。

你学到了三件事：① 构造请求；② 获得响应；③ 把依赖真实网络的部分替换成可控的测试对象。**这是测试方式，不是所有 HTTPX 项目必须使用的配置。**

### 0.10 为什么原文后面有一百多行的图书模拟服务？

因为那段程序一次性模拟了很多**服务端规则**：按编号查书、分页、重定向、文本响应、Cookie、下载等。它的复杂性主要来自“要模仿多少种服务器行为”，不是因为发一次 HTTPX 请求真的需要一百行。

你可以把它看作一套**练习用的假服务器**。初学 HTTPX 时，先学会本章 `0.4` 和 `0.9`。只有开始学后文的多个功能时，才需要在附录中复制那份完整代码。

理解模拟服务的顺序：

1. `BOOKS = [...]`：一份普通 Python 列表，里面装字典。
2. `demo_handler(request)`：模拟服务端收到某个路径时，决定返回什么。
3. `httpx.Response(200, json=...)`：创建模拟响应；此时没有真正访问远端图书系统。
4. `make_sync_client()`：把 `base_url`、`MockTransport`、超时配置组装成一个 `Client`。
5. 之后你调用 `client.get("/books/3")`，才真正从客户端角度练习“发请求、看结果”。

### 0.11 对 Requests 和 HTTPX 的选择，不要被新名词带偏

| 你的需求 | 比较自然的选择 |
|---|---|
| 同步脚本，读取少量公开 API，现有项目已用 Requests | 继续用 Requests 完全合理 |
| 新项目想使用统一的同步与异步接口 | 可以选择 HTTPX |
| 已有 `asyncio` 应用，需要异步发多个网络请求 | HTTPX 的 `AsyncClient` 很合适 |
| 需要连接池、Cookie、认证、超时、流式下载 | 两者都有相应功能，写法和默认行为有差异 |
| 只是为了让一个请求“更快”，想换成 `async` | 通常不是可靠的理由 |

几个常见区别要特别留意：HTTPX 通常有自己的**默认超时**，Requests 不显式传超时可能长期等待；HTTPX 默认**不自动跟随重定向**（需按需求使用 `follow_redirects=True`），Requests 的多数常见 GET 请求通常会自动跟随重定向。涉及版本的具体细节以所安装版本的官方文档为准，别假定两套库的所有默认值都相同。

### 0.12 入门后最常见的坑

- **误以为 `response.json()` 能处理任何响应**：HTML、空正文、格式不合法的 JSON 都可能让它失败。
- **请求出错却直接访问 `response`**：连接失败可能直接抛 `httpx.RequestError` 相关异常，那次调用根本没产生响应。
- **忽略状态码**：HTTPX 中通常主动调用 `response.raise_for_status()`，让非成功状态更容易被发现。
- **忘记关闭客户端或流式响应**：优先使用 `with Client(...)`、`async with AsyncClient(...)` 及文档建议的流式上下文。
- **把 `await` 当作多线程**：它是异步协作机制的一部分，不代表每个计算都在独立线程执行。
- **把模拟服务当真实联网**：`MockTransport` 练习客户端 API 很方便，却不能用来验证真实 DNS、TLS、网络超时、连接池实际性能。
- **认为 403、404 都是 Python 代码语法错**：它们通常是你收到了服务器返回的 HTTP 状态，需要检查请求路径、参数、身份或服务端策略。
- **随手关闭 TLS 证书验证**：不要因为 `verify=False` 能暂时跳过某个错误就把它当成通用修复办法，这会削弱 HTTPS 身份校验。

**建议的学习路径**：先学本节 `0.1—0.6` → 自己解释一次 `get` → 学同步 `Client` → 学超时、参数、状态码和 JSON → 用小型 `MockTransport` 练习 → 有明确并发需求再学 `asyncio` 与 `AsyncClient` → 最后阅读文末模拟服务完整代码和高阶案例。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

假设你已经能用 Python 读取本地文件。

现在，程序还要向一个服务查询资料。

查询一条数据时，等待几百毫秒似乎没有问题；一次查询几十条时，你开始觉得程序大部分时间都在等网络。

这时，你会遇到 `httpx.Client`、`httpx.AsyncClient`、`await`、连接池、流式响应和超时等概念。

这一篇从最普通的请求开始，逐步把这些问题连接起来。

常规示例面向 Python 3.11 及以上，HTTPX 接口以 **0.28.1** 为验证基线。

它是第三方库，需要安装；这里不引入大型 Web 框架。

为了便于实际练习，前半篇主要使用 `MockTransport` 构造离线图书服务。

你可以真实执行 HTTPX 的请求准备、客户端调用和响应处理，而不依赖公共网站当时是否可用。

涉及真实连接和超时的地方，再明确切换到本机回环 HTTP 服务。

## 第一章：HTTPX 是什么？为什么会需要另一种 HTTP 客户端？

### 1.1 先从你已经熟悉的等待看起

读取一个本地小文件，通常很快。

访问远程服务，则可能经历域名解析、建立连接、TLS 握手、服务器处理和响应传输。

这些步骤里有相当一部分时间，CPU 并没有在不停计算你的业务结果，而是在等外部事件。

如果代码按顺序写：

```text
查询第一本书 → 等结果
查询第二本书 → 等结果
查询第三本书 → 等结果
```

三个等待会依次发生。

如果任务允许并发，则可以组织成：

```text
发出第一项查询 ── 等待 ── 取得结果
发出第二项查询 ── 等待 ── 取得结果
发出第三项查询 ── 等待 ── 取得结果
```

这些等待有机会相互重叠。

HTTPX 同时提供同步和异步客户端接口，方便应用按自己的运行方式选择。[官方介绍](https://www.python-httpx.org/)

### 1.2 同步到底是什么意思？

同步调用中，当前执行流程会等待这个调用取得结果或失败，再继续后面的语句。

```python
response = client.get("/books/1")
print(response.status_code)
```

`get()` 尚未返回时，当前这条流程不会自动跑到下一行打印。

这很直观，也很适合一般脚本。

同步并不等于“整个操作系统不能做其他事情”，也不意味着程序绝对不能使用线程。

它描述的是这个调用接口与当前流程的组织方式。

### 1.3 异步又改变了什么？

异步接口允许协程在等待可让出的操作时，把执行机会交给事件循环安排的其他任务。

```python
response = await client.get("/books/1")
```

`await` 表达当前协程在这里等待结果，并允许异步运行机制在合适的暂停点调度其他协程。

它不会把远端服务器的工作变快，也不会让一个 CPU 密集计算凭空并行到多个核心。

它主要帮助你组织大量等待型任务。

### 1.4 我只是写一个小脚本，也必须学异步吗？

不必须。

HTTPX 的同步 `Client` 已经能完成请求、认证、分页和下载等常见工作。

当你确实有多个独立网络操作、应用本身已经在异步环境里运行，或者要理解现代 Python 网络代码时，再使用 `AsyncClient`。

学会选择比把每个函数都改成 `async def` 更重要。

### 1.5 Requests 还能继续用吗？

可以。

已有项目稳定使用 Requests，不会因为存在 HTTPX 就自动变成错误方案。

这两套接口有相似概念，但超时、重定向、异常和参数细节并不完全相同。

如果需要比较，可结合 [Requests 独立教材](Python-Requests库.md) 阅读。

本篇会在容易迁移出错的地方单独解释，而不是只把导入名称替换掉。

### 1.6 先把 HTTP 请求里的几个词对上号

HTTP 来自 Hypertext Transfer Protocol，通常译为超文本传输协议。

名称保留了网页发展的历史背景；今天它也大量传递 JSON、图片、文件和其他内容。

在本篇里，你的 Python 程序是客户端，提供图书数据的一端是服务端。

一次请求通常可以分成：

```text
方法：我想执行哪类操作，例如 GET 读取。
URL：我访问哪个服务与资源。
请求头：描述格式、身份和其他协议资料。
请求体：需要提交的内容，某些请求可以没有。
```

一次响应通常可以分成：

```text
状态码：服务端如何描述这次处理结果。
响应头：内容类型、长度、缓存等资料。
响应体：真正返回的数据。
```

所以，`client.get()` 返回响应对象之后，你还需要选择看状态、看头，还是处理正文。

HTTPX 不会因为某个接口名字叫 `books`，就自动知道书价必须是非负整数。

协议传输与业务结构需要分别处理。

你还可能听到 API，也就是 Application Programming Interface，应用程序编程接口。

网络 API 通常用约定好的 URL、方法和数据结构，让不同程序互相调用。

HTTPX 是客户端工具，帮助你按这些约定发请求，不负责替服务端发明接口规则。

## 第二章：安装与版本，先确认代码到底在哪个 Python 里运行

### 2.1 安装

在你的项目环境中执行：

```bash
python -m pip install httpx
```

`python -m pip` 表示使用当前这个 Python 对应的 pip 模块。

如果你在 WSL 写项目，就使用该 Linux 环境里的 Python 和虚拟环境；不要把 Windows 的环境目录当成同一份 Linux 环境。

### 2.2 查看实际版本

```python
import sys
import httpx

print(sys.executable)
print(httpx.__version__)
print(httpx.__file__)
```

第一项帮助确认解释器。

第二项确认库版本。

第三项确认导入来源。

如果你的脚本也叫 `httpx.py`，可能遮蔽真正的包，导致导入结果与你预期不同。

### 2.3 为什么本篇写明 0.28.1？

第三方库会演进。

例如，你可能在旧教程里见到 `proxies=` 参数，而当前版本使用的代理配置入口已经变化。

与其看到 `TypeError` 就盲目降级，不如先核对当前版本的正式接口和项目依赖约定。

本篇普通用法采用当前验证基线，涉及可能变化的接口会给出官方文档入口。

### 2.4 需要 HTTP/2 吗？

HTTPX 可以通过可选依赖和配置支持 HTTP/2。

这不意味着每个请求默认使用 HTTP/2，也不意味着开启后所有服务都会变快。

客户端、服务端、TLS 协商和网络环境都要支持相应协议。

初学阶段先用 HTTP/1.1 语义理解请求和响应，后面再根据服务需求开启可选能力。

不要为了跟随一个名词，就给所有项目添加无用配置。

## 第三章：先搭一个离线图书服务，后面每个例子都有明确输入

### 3.1 为什么采用 MockTransport？

如果每个示例都依赖一个公共 API，可能遇到限流、登录变化、接口改版和网络故障。

这样很难判断你究竟是代码写错，还是外部服务发生变化。

`MockTransport` 允许你提供一个处理函数：接收准备好的 HTTPX 请求，返回你控制的响应。

请求仍然经过 HTTPX 客户端的许多上层步骤，但不会真的建立 TCP/TLS 连接。

### 3.2 先运行一个十余行的完整离线请求

下面代码可以直接在独立脚本中运行，不需要先启动服务器。

```python
import httpx


def hello_handler(request):
    return httpx.Response(200, json={"message": "你好，HTTPX"})


transport = httpx.MockTransport(hello_handler)

with httpx.Client(transport=transport) as client:
    response = client.get("https://books.example.invalid/hello")
    response.raise_for_status()
    print(response.json()["message"])
```

输出：

```text
你好，HTTPX
```

先确认这一小段可以运行，再继续增加图书、分页和异步处理。

### 3.3 `.invalid` 为什么没有导致联网失败？

因为这个客户端配置了 `MockTransport`。

在刚才这个最小例子中，请求交给 `hello_handler()`，没有去解析并连接这个域名。

`.invalid` 只用于表达一个清楚的练习地址，避免误以为它属于真实可用公共服务。

如果你去掉 `transport=`，客户端就会尝试按正常网络流程访问，那时这个地址不应被当作可用服务。

### 入门导读：为什么长代码不应挡在你前面？

先学 get、Response、Client 的基础。完整模拟服务供后文分页、错误状态、Cookie 和下载等练习使用，无需理解它才能使用 HTTPX。它的结构是：图书数据 → 请求处理函数 → 路径分支 → 模拟响应 → 客户端工厂。

使用方法：需要运行练习时，再把文末附录代码保存为 `httpx_demo.py`，先观察各个小块，然后运行客户端。后文原有逐项讲解仍然保留。

### 3.4 后面反复使用的图书接口放在哪里？

后续章节会使用图书、分页、重定向和下载等接口。

先让它们共用一份确定的数据。

下面模块保存五本书，再按请求的路径返回相应响应。

把完整代码保存为 `httpx_demo.py`，与后续练习脚本放在同一目录。

导入模块只会定义数据和函数；真正调用客户端时，HTTPX 才会把相应请求交给处理函数。

**这一段完整配套代码已经移至文末附录 A，初学时请先跳过。**


### 3.5 `demo_handler()` 里面为什么有这么多判断？

因为后面要观察不同请求。

`/hello` 返回一句问候。

`/echo` 返回参数或正文信息，让你看到自己发了什么。

`/books` 根据页码取出一部分书。

`/books/3` 查找编号为三的书。

这些判断相当于把“收到哪种练习请求，就给哪种确定响应”的规则写出来。

它们没有去访问真实公共图书网站。

你可以把书名改成自己的笔记名，但后面核对输出时，也应使用修改后的数据。

### 3.6 分页那几行怎样理解？

例如，每页两条，查询第二页。

起点按下面规则计算：

```text
start = (2 - 1) × 2 = 2
```

然后切片 `BOOKS[2:4]`。

Python 下标从零开始，右边界又不包含在结果中，因此取到的是第三、第四本书。

页码从一开始是本例的 API 约定，不是所有网络接口都遵守的通用规则。

`total` 返回五，表示完整数据集的条目数，而不是本页返回两条。

这两个数在后面的“什么时候停止请求下一页”中各有用途。

### 3.7 响应对象是我自己创建的，还算使用了 HTTPX 吗？

算。

客户端仍然准备 URL、合并查询参数、编码请求体，处理响应与 Cookie。

只是网络收发被替换为你提供的回调。

例如，`request.url.params.multi_items()` 可以观察重复查询键。

`request.content` 可以观察这次已经准备好的请求体字节。

不过，离线正文已经在内存中的情况，不能用来推断真实大文件下载的网络缓冲与性能。

本篇因此分别安排离线行为验证和真实回环连接实验。

### 3.8 `make_sync_client()` 为什么返回一个客户端？

它把后面反复使用的基本地址和传输层配置放在一处。

每次调用都会创建一个新的 `Client`。

调用者通常这样使用：

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.get("/books/1")
    response.raise_for_status()
    print(response.json()["title"])
```

使用本章原始书目时，输出：

```text
Python 文件处理
```

函数名里的 sync 表示这里创建同步客户端。

异步客户端会在后文显式使用 `AsyncClient`，不会因为把变量改名就自动获得异步行为。

### 3.9 这个处理函数是不是 Web 服务器？

它没有监听端口，没有独立进程，也没有实现完整 HTTP 服务。

它是测试传输层回调。

因此，适合验证参数准备、客户端生命周期、响应解析和业务逻辑。

不能用它证明真实 DNS、证书、连接池竞争或套接字超时已经验证成功。

这条边界会贯穿整篇教材。

## 第四章：同步客户端的第一次请求，逐句读懂

新建另一个脚本，与 `httpx_demo.py` 放在同一练习目录。

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.get("/hello")
    response.raise_for_status()
    data = response.json()

print(data["message"])
```

通常输出：

```text
你好，HTTPX
```

### 4.1 `with make_sync_client() as client` 做什么？

工厂函数创建一个 `httpx.Client`。

`with` 在明确范围内管理它的资源。

`client` 是你给这个对象取的变量名，完全可以改成 `api_client`。

它不是模块，也不是服务器。

### 4.2 为什么 `get()` 里只写 `/hello`？

因为客户端已经配置了：

```text
https://books.example.invalid
```

作为基础 URL。

方法调用提供资源路径，客户端按自己的 URL 合并规则构造最终请求。

可以观察：

```python
print(response.url)
```

### 4.3 `response` 是字符串吗？

不是。

它是 HTTPX 的响应对象，包含状态、头、URL 和响应体等信息。

想取得文本可以访问 `.text`；想取得字节可以访问 `.content`；想解析 JSON 可以调用 `.json()`。

把整个响应对象 `print()` 出来，通常只看到概括描述，不是完整业务内容。

### 4.4 `raise_for_status()` 为什么放在解析前面？

先确认状态符合正常成功路径，再按成功响应的结构处理。

错误响应也可能是 JSON，但其字段往往不同。

如果不看状态直接取 `data["message"]`，可能把“资源不存在”变成一个看起来莫名其妙的缺键错误。

### 4.5 离开 Client 后，data 还能用吗？

这里默认请求已经取得并读取响应体，`data` 是解析后独立的 Python 字典。

关闭客户端，不会让已经读入内存的数据消失。

但流式响应还没有读完时，情况不同，后面会单独解释。

## 第五章：基础 URL、默认配置与连接池

### 5.1 为什么不每次调用顶层 `httpx.get()`？

一次性脚本可以使用顶层便捷函数。

同一服务要反复请求时，Client 能集中管理配置、Cookie 和连接池。

```python
with httpx.Client(base_url="https://service.example.invalid") as client:
    first = client.get("/items", timeout=5.0)
    second = client.get("/profile", timeout=5.0)
```

这段使用占位地址，结构演示应替换成你有权访问的服务，或者配置本篇的 MockTransport 后运行。

不要把这个无传输替换的示例当成离线代码。

### 5.2 连接池复用的是什么？

它主要复用适合继续使用的底层连接。

这样可以减少重复建立 TCP/TLS 的成本。

连接能否复用，还取决于服务端、连接状态、响应是否消费完成以及协议条件。

Client 不是“缓存所有请求结果”。

第二次请求仍然可能真正访问服务，只是有机会使用已有连接。

### 5.3 默认头与单次覆盖

```python
import httpx
from httpx_demo import demo_handler

with httpx.Client(
    base_url="https://books.example.invalid",
    transport=httpx.MockTransport(demo_handler),
    headers={"X-Demo": "default"},
) as client:
    first = client.get("/echo", headers={"X-Demo": "temporary"})
    second = client.get("/echo")

    print(first.json()["x_demo"])
    print(second.json()["x_demo"])
```

通常输出：

```text
temporary
default
```

单次请求参数覆盖对应默认项，不等于永久修改客户端状态。

### 5.4 基础 URL 带有 `/api`，前导斜杠会怎样？

这个地方不能直接照搬你对其他 URL 合并函数的印象。

在本篇验证的 HTTPX 0.28.1 中：

```python
import httpx


def show_url(request):
    return httpx.Response(200, json={"url": str(request.url)})


with httpx.Client(
    base_url="https://demo.example.invalid/api",
    transport=httpx.MockTransport(show_url),
) as client:
    response = client.get("/items")
    print(response.json()["url"])
```

输出：

```text
https://demo.example.invalid/api/items
```

HTTPX 客户端会规范化基础路径并按其规则合并。

不要假设 `/items` 一定把 `/api` 丢掉，也不要只凭磁盘路径的绝对片段规则推断 URL 行为。

迁移代码时，打印准备后的请求 URL，是很有效的验证方式。

### 5.5 Client 可以随便打开、关闭再打开吗？

不应把同一个已关闭实例当成可以无限重新进入的可重用上下文。

一个客户端有明确生命周期。

关闭后需要新的一轮独立客户端，就创建新对象。

更常见的设计是让同一实例覆盖适当的应用工作范围，而不是在每次循环里不停创建和销毁。

## 第六章：查询参数、Headers 和 URL 对象

### 6.1 让库编码查询值

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.get(
        "/echo",
        params=[("keyword", "Python 文件"), ("tag", "python"), ("tag", "backend")],
    )
    response.raise_for_status()
    print(response.json()["query"])
```

本地回调会显示准备后解析的多值参数。

JSON 把元组形式键值对转成数组，所以响应中的每一项表现为二元素列表。

### 6.2 `QueryParams` 是什么？

它是 HTTPX 用来表示查询参数的对象。

```python
import httpx

params = httpx.QueryParams([
    ("tag", "python"),
    ("tag", "backend"),
    ("page", 2),
])

print(params.get_list("tag"))
print(list(params.multi_items()))
```

`get_list()` 取得同名参数的所有值。

`multi_items()` 保留多值键值对，而普通字典不能独立保存两个完全相同的键。

### 6.3 `None` 和布尔值怎么编码？

在本篇版本中：

```python
import httpx

params = httpx.QueryParams({"flag": True, "empty": None})
print(str(params))
```

得到类似：

```text
flag=true&empty=
```

`None` 变成空值，不是自动省略。

布尔值使用对应小写形式。

如果业务要求不发送某项，应在构造参数时主动排除。

不要把另一个 HTTP 库的 `None` 省略规则直接带进来。

### 6.4 头字段名区分大小写吗？

HTTP 头字段名通常按不区分大小写语义处理。

```python
import httpx

headers = httpx.Headers({"Content-Type": "application/json"})
print(headers["content-type"])
```

但字段值不因此自动变成无大小写字符串，也不意味着所有重复字段都能简单用逗号合并。

例如 Cookie 相关字段和其他特殊头，应使用库提供的容器与协议规则处理。

### 6.5 URL 是字符串还是对象？

HTTPX 的请求和响应 URL 通常是 `httpx.URL` 对象。

```python
import httpx

url = httpx.URL("https://demo.example.invalid/books?page=2")

print(url.scheme)
print(url.host)
print(url.path)
print(url.params["page"])
print(str(url))
```

对象属性便于解析，不需要自己用字符串下标猜测主机和路径边界。

需要显示或交给明确要求字符串的接口时，再使用 `str(url)`。

URL 仍然不等于本地文件路径，不应该交给 `Path` 作为网络地址构造器。

## 第七章：发送内容时，`params`、`json`、`data`、`content` 与 `files` 怎么分？

学会 GET 以后，常见下一问是：“如果我要把自己的数据交给服务器呢？”

先确认数据应放在请求的哪一部分，再选择参数。

### 7.1 URL 查询参数

```python
response = client.get("/books", params={"page": 2, "page_size": 2})
```

参数进入 URL 查询字符串。

它们不是 JSON 请求体。

一些服务可能会记录 URL，因此不要随意把秘密令牌放进查询参数中。

### 7.2 JSON 请求体

使用完整练习模块：

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.post(
        "/echo",
        json={"title": "学习记录", "finished": True},
    )
    response.raise_for_status()
    print(response.json()["data"])
```

通常得到：

```text
{'title': '学习记录', 'finished': True}
```

你提供 Python 对象，HTTPX 负责序列化和相应内容类型。

不必先 `json.dumps()` 再传给 `json=`，否则可能把对象再次包成 JSON 字符串。

### 7.3 表单字段用 `data`

```python
response = client.post("/echo", data={"name": "小林", "course": "Python"})
```

字典形式通常编码成表单数据。

完整练习回调主要记录媒体类型与原始长度，没有替你实现全部表单业务逻辑。

可以通过准备后的请求观察：

```python
request = client.build_request("POST", "/echo", data={"name": "小林"})
print(request.headers["Content-Type"])
print(request.content)
```

### 7.4 原始字节或文本请求体用 `content`

```python
response = client.post(
    "/echo",
    content=b"raw payload",
    headers={"Content-Type": "application/octet-stream"},
)
```

这表达“整个请求体就是这些内容”。

HTTPX 中建议用 `content=` 传原始内容，避免沿用某些旧代码把任意字节放入 `data=` 的习惯。

参数名字相近，实际编码意图却不同。

### 7.5 上传文件用 `files`

```python
from pathlib import Path
from httpx_demo import make_sync_client

path = Path("httpx_practice") / "note.txt"
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text("一份练习资料\n", encoding="utf-8")

with make_sync_client() as client:
    with path.open("rb") as source:
        response = client.post(
            "/echo",
            files={"document": (path.name, source, "text/plain")},
            data={"description": "练习上传"},
        )
        response.raise_for_status()
        print(response.json()["content_type"])
```

`files` 会参与 multipart 编码。

其中的边界由编码器生成，不要手工覆盖成一个缺少边界的 `multipart/form-data` 头。

### 7.6 文件为什么必须保持打开？

请求发送过程中还可能读取它。

如果离开文件上下文后才开始发送，流已经关闭，后续就无法正常读取。

这与 `open()` 的资源生命周期完全相通。

上传不是把“路径”交给服务器让它访问你的磁盘，而是客户端实际读出内容并发送。

### 7.7 这些参数能全部一起传吗？

不要用“全部给上，让库自己猜”的方式构造请求。

查询参数可以与请求体共存，但 JSON、表单、原始内容和文件容器需要遵守接口格式与库的组合规则。

最清楚的方式是先查看服务契约：它到底接受哪种媒体类型，字段放在哪里，是否允许文件部分。

选择确定之后，再写相应参数。

## 第八章：响应的状态、内容与错误，是三个不同问题

### 8.1 `.content`、`.text`、`.json()`

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.get("/hello")
    print(type(response.content).__name__)
    print(type(response.text).__name__)
    print(type(response.json()).__name__)
```

通常依次看到：

```text
bytes
str
dict
```

字节内容、文本解码和 JSON 结构解析分属不同层次。

`.json()` 得到字典，是因为这条接口返回对象；别的合法 JSON 顶层也可以是列表、字符串或空值。

### 8.2 HTTPX 的 `raise_for_status()` 会拒绝 302

本篇默认不跟随重定向。

```python
import httpx
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.get("/redirect")
    print(response.status_code)

    try:
        response.raise_for_status()
    except httpx.HTTPStatusError:
        print("这不是 2xx 成功响应")
```

这与某些其他 HTTP 库只对 4xx、5xx 抛状态错误的习惯不同。

HTTPX 的成功判断以 2xx 为核心，重定向响应也可能需要你主动处理。

### 8.3 判断响应状态的属性

```python
print(response.is_success)
print(response.is_redirect)
print(response.is_client_error)
print(response.is_server_error)
```

它们便于按状态类别检查。

但是，“HTTP 成功”仍不代表业务字段合法，也不代表当前用户有权执行后续本地动作。

### 8.4 404 响应的 `bool` 是什么？

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.get("/books/999")
    print(response.status_code)
    print(bool(response))
```

在本篇版本中，通常得到：

```text
404
True
```

因此，不要把 `if response:` 当成成功状态判断。

明确看状态或调用状态检查方法，代码意图更清楚。

### 8.5 状态异常与传输异常

```python
import httpx
from httpx_demo import make_sync_client

try:
    with make_sync_client() as client:
        response = client.get("/books/999")
        response.raise_for_status()
except httpx.HTTPStatusError as error:
    print("服务返回状态：", error.response.status_code)
except httpx.RequestError as error:
    print("请求传输失败：", type(error).__name__)
```

`HTTPStatusError` 说明已经有一个响应，但状态不符合正常成功条件。

`RequestError` 分支主要覆盖发送请求与传输相关错误。

没有拿到响应的连接失败，不应假装一定能读取 `error.response.status_code`。

### 8.6 JSON 解析错误应单独处理

```python
import json
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.get("/text")
    response.raise_for_status()

    try:
        data = response.json()
    except json.JSONDecodeError:
        print("这是成功的普通文本响应，不是 JSON")
```

不要只捕获 HTTPX 的传输异常，就认为所有内容解析错误也会被一起处理。

格式解析属于后续步骤。

## 第九章：把同步例子改成异步，为什么多了这些关键字？

你可能见过：

```python
async with httpx.AsyncClient() as client:
    response = await client.get(url)
```

如果只把它当成固定模板，很容易在别的位置乱加 `await`。

我们从一个完整可运行例子拆开。

### 9.1 第一次异步离线请求

```python
import asyncio
import httpx
from httpx_demo import demo_handler


async def main():
    transport = httpx.MockTransport(demo_handler)

    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=transport,
    ) as client:
        response = await client.get("/hello")
        response.raise_for_status()
        data = response.json()
        print(data["message"])


if __name__ == "__main__":
    asyncio.run(main())
```

输出仍然是：

```text
你好，HTTPX
```

### 9.2 `async def main()` 做什么？

它定义协程函数。

调用 `main()` 时得到协程对象，并不等同于普通函数立即把函数体全部执行完。

需要异步运行机制安排这个协程。

普通脚本可以用 `asyncio.run(main())` 创建并运行一次顶层事件循环。

### 9.3 为什么用 `async with`？

异步客户端的进入和退出可能需要异步资源处理。

`async with` 按异步上下文管理协议工作，退出时等待相应关闭过程。

它不是在普通 `with` 前随意加一个好看的单词。

`Client` 用普通 `with`；`AsyncClient` 用 `async with`，分别对应它们提供的接口。

### 9.4 为什么 `client.get()` 前面要 `await`？

异步客户端的请求方法返回需要等待的异步操作。

`await` 让当前协程等待响应，并在合适的 I/O 等待点让出执行机会。

如果你只是：

```python
result = client.get("/hello")
```

拿到的还不是完成后的响应。

继续访问 `result.status_code`，通常会因为对象类型不符合预期而失败，还可能看到协程没有被等待的警告。

### 9.5 为什么 `.json()` 前面没有 `await`？

在这里，普通异步 `get()` 返回前已经读取了响应体。

`.json()` 在现有内容上执行同步解析，返回 Python 对象，不是一个需要等待的协程。

所以应写：

```python
data = response.json()
```

而不是：

```python
# 错误示意：普通 json() 的返回值不是可等待对象。
data = await response.json()
```

看到异步函数，不代表里面每一行都要加 `await`。

先问当前调用的接口到底返回什么。

### 9.6 Notebook 里为什么 `asyncio.run()` 报错？

某些交互式环境已经有正在运行的事件循环。

此时再用 `asyncio.run()` 创建嵌套运行，会受到限制。

环境支持顶层 await 时，可以直接：

```python
await main()
```

这是一种依赖交互环境的调用方式，不能直接放进普通不允许顶层 await 的 `.py` 文件。

普通脚本与 Notebook 的入口不同，应按运行环境选择。

更完整的异步基础可以结合 [异步、协程与事件循环笔记](Python-异步、协程、事件循环与线程.md) 理解。

## 第十章：已经用了 `async`，为什么三个请求还是依次等待？

### 10.1 顺序 await 仍然是顺序流程

```python
first = await client.get("/books/1")
second = await client.get("/books/2")
third = await client.get("/books/3")
```

当前协程先等第一项完成，再开始第二项，然后才开始第三项。

异步接口提供可调度能力，但没有自动把所有后续语句同时启动。

### 10.2 用一个可控延迟观察

```python
import asyncio
import time
import httpx


async def delayed_handler(request):
    await asyncio.sleep(0.1)
    return httpx.Response(200, json={"path": request.url.path})


async def main():
    async with httpx.AsyncClient(
        base_url="https://demo.example.invalid",
        transport=httpx.MockTransport(delayed_handler),
    ) as client:
        started = time.perf_counter()
        responses = []

        for index in range(3):
            responses.append(await client.get(f"/items/{index}"))

        elapsed = time.perf_counter() - started
        print("结果数：", len(responses))
        print("顺序耗时约：", round(elapsed, 2))


if __name__ == "__main__":
    asyncio.run(main())
```

通常会看到接近三个延迟之和的耗时。

具体数值受调度和机器状态影响，不应该断言必须精确等于某个小数。

### 10.3 如果这些请求互不依赖，想让等待重叠呢？

```python
responses = await asyncio.gather(
    client.get("/items/0"),
    client.get("/items/1"),
    client.get("/items/2"),
)
```

这些异步操作会由运行机制并发推进。

在前面的可控异步延迟模型里，总耗时通常更接近一次延迟，而不是三次相加。

但真实服务的连接限制、限流、带宽和负载也会影响结果。

不能把这个演示中的倍率当成所有网络任务的固定加速倍数。

### 10.4 为什么 MockTransport 里要用 `await asyncio.sleep()`？

因为我们想模拟一个会让出执行机会的等待。

如果你在异步处理函数里直接：

```python
import time

time.sleep(0.1)
```

会阻塞当前事件循环线程，其他协程没有因此获得运行机会。

这正好说明：函数被写成 `async def`，不代表内部所有操作自动变成非阻塞。

### 10.5 有先后依赖的请求还能并发吗？

假设第二个请求需要第一个返回的用户 ID。

那么这个依赖关系本身要求先取得前一步结果。

可以并发的是已经具备输入、彼此独立的工作。

不要为了追求“并发”，让尚未取得的结果被错误地当成已存在变量使用。

## 第十一章：`gather()` 的结果顺序与失败传播

### 11.1 哪个先完成，结果就排在哪个位置吗？

`asyncio.gather()` 的结果按你传入操作的顺序组织，不是按实际完成先后排序。

```python
import asyncio


async def work(name, delay):
    await asyncio.sleep(delay)
    return name


async def main():
    results = await asyncio.gather(
        work("慢一点的第一项", 0.1),
        work("快一点的第二项", 0.01),
    )
    print(results)


if __name__ == "__main__":
    asyncio.run(main())
```

通常输出：

```text
['慢一点的第一项', '快一点的第二项']
```

如果要按完成顺序处理，需要考虑 `as_completed()` 等相应组织方式。

### 11.2 一个请求失败，其他请求会怎样？

默认 `gather()` 会把相应异常传播给等待它的调用方。

但不能简单断言它会自动取消所有其他操作。

其他任务可能仍在运行，具体还受到外层事件循环关闭、调用者取消和你的清理逻辑影响。

因此，网络客户端生命周期必须覆盖这些任务，不能某一项抛错后立即关闭客户端，却让其他任务还在使用它。

### 11.3 想收集每一项的成功或失败呢？

可以在每个工作函数内部捕获预期错误，返回明确结果；也可以使用 `return_exceptions=True`，再逐项检查。

```python
results = await asyncio.gather(*operations, return_exceptions=True)

for result in results:
    if isinstance(result, BaseException):
        print("这一项失败：", type(result).__name__)
    else:
        print("这一项成功")
```

这段依赖你已经构造的 `operations`，是结果处理片段。

不要把异常对象误当成正常响应继续访问 `.json()`。

### 11.4 Python 3.11 的 TaskGroup 有什么用？

TaskGroup 提供结构化管理一组异步任务的方式。

```python
import asyncio
import httpx
from httpx_demo import demo_handler


async def fetch_book(client, book_id):
    response = await client.get(f"/books/{book_id}")
    response.raise_for_status()
    return response.json()


async def main():
    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(demo_handler),
    ) as client:
        async with asyncio.TaskGroup() as group:
            tasks = [
                group.create_task(fetch_book(client, book_id))
                for book_id in [1, 2, 3]
            ]

        print([task.result()["id"] for task in tasks])


if __name__ == "__main__":
    asyncio.run(main())
```

离开任务组时，会等待相应任务结束。

组内某项出现非取消异常时，它会按 TaskGroup 的规则取消其他任务并汇集异常。

这与随意创建任务后不保存引用、不等待结束的方式不同。

### 11.5 取消是不是错误，需要全部吞掉？

取消通常是异步控制流的一部分。

在 Python 3.11+ 的常见 asyncio 语境中，`CancelledError` 不属于普通 `Exception` 分支。

不要为了“程序不要报错”就捕获 `BaseException` 并无条件继续，这可能阻止正常停止和资源清理。

需要清理时使用 `finally` 或上下文管理；清理后通常让取消继续传播。

## 第十二章：可以并发以后，为什么还要限制并发？

如果十个请求能同时等待，你可能会想到一次启动一万个。

但任务数量、连接数量、服务承载能力和本机内存都有限。

异步让等待更容易重叠，没有取消资源限制。

### 12.1 用 Semaphore 表达应用并发上限

```python
import asyncio
import httpx
from httpx_demo import demo_handler


async def fetch_limited(client, semaphore, book_id):
    async with semaphore:
        response = await client.get(f"/books/{book_id}")
        response.raise_for_status()
        return response.json()


async def main():
    semaphore = asyncio.Semaphore(2)

    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(demo_handler),
    ) as client:
        results = await asyncio.gather(*[
            fetch_limited(client, semaphore, book_id)
            for book_id in [1, 2, 3, 4, 5]
        ])

    print([book["id"] for book in results])


if __name__ == "__main__":
    asyncio.run(main())
```

信号量初始值为二，表示最多允许两个任务同时进入被包围的工作范围。

其他协程等待许可，不会因为等待信号量就占用一个执行线程不停空转。

### 12.2 `async with semaphore` 为什么有帮助？

进入时取得许可，离开时释放。

即使请求抛异常，正常上下文退出过程也会安排释放，避免因为漏写 `release()` 导致后面任务永远等待。

它和资源上下文的思路相同，只是管理的是并发许可。

### 12.3 应该包围哪一段？

如果你希望限制整个下载动作，包括读取响应体，就应把实际流式读取也放在范围内。

只限制“发出请求到拿到响应头”，之后在外面继续保持大量流式连接，可能超出你原先想限制的资源数量。

并发范围应该对应真正占用资源的工作，而不是随便选一行包起来。

### 12.4 连接池的 `Limits` 又是什么？

```python
import httpx

limits = httpx.Limits(
    max_connections=10,
    max_keepalive_connections=5,
    keepalive_expiry=10.0,
)
```

`max_connections` 限制相应客户端传输层可同时保持的连接数量。

`max_keepalive_connections` 限制可保留用于后续复用的空闲连接数量。

`keepalive_expiry` 表示空闲连接保留时间相关设置。

这几个数不是“最多十个协程”“每秒十次请求”的同义词。

### 12.5 Semaphore 和连接池限制会不会重复？

它们控制不同层次。

信号量可以围住一整个业务工作单元，甚至包含多个请求与后续处理。

连接池限制控制底层连接资源。

HTTP/2 等协议还可能在一个连接上承载多个流，因此“连接数”等于“业务并发数”的假设更不可靠。

应用可能同时需要合理的工作并发和传输资源上限。

### 12.6 MockTransport 能测试连接池上限吗？

本篇的 MockTransport 不建立真实 TCP 连接，也不使用正常网络连接池获取过程。

所以，即使给客户端配置 `Limits`，也不能用模拟回调并发数证明真实连接池限制生效。

可以用它验证应用信号量是否限制工作范围。

要验证 PoolTimeout 或真实连接复用，需要回环服务器或其他受控真实传输。

### 12.7 信号量限制了运行数，为什么内存还是很高？

假设你先为一百万项创建一百万个任务，再让它们等一个容量为十的信号量。

同时运行的工作确实不多，但这一百万个任务对象仍然存在。

因此，大型输入还需要限制“已经创建但尚未处理”的任务数量。

可以采用有界队列和固定数量工作协程，而不是一次建出全部任务。

### 12.8 固定工作协程示例

```python
import asyncio
import httpx
from httpx_demo import demo_handler


async def collect_books(book_ids):
    queue = asyncio.Queue(maxsize=4)
    results = []

    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(demo_handler),
    ) as client:
        async def worker():
            while True:
                book_id = await queue.get()
                try:
                    if book_id is None:
                        return
                    response = await client.get(f"/books/{book_id}")
                    response.raise_for_status()
                    results.append(response.json())
                finally:
                    queue.task_done()

        async with asyncio.TaskGroup() as group:
            for _ in range(2):
                group.create_task(worker())

            for book_id in book_ids:
                await queue.put(book_id)

            for _ in range(2):
                await queue.put(None)

    return results


async def main():
    results = await collect_books([1, 2, 3, 4, 5])
    print(sorted(book["id"] for book in results))


if __name__ == "__main__":
    asyncio.run(main())
```

队列满时，生产者会等待，这形成明确背压。

工作协程数量固定，不随着输入无限增长。

但 `results` 仍保存全部结果，若结果巨大，还应边处理边存储，或者把结果写入数据库等受控位置。

这里使用 `None` 作为停止标记，所以正常任务 ID 不能与它混淆。

## 第十三章：HTTPX 的四类超时，分别在等什么？

### 13.1 默认五秒，不是整个任务五秒

HTTPX 默认具有相应网络等待超时，而不是像某些库那样默认无限等待。

但“默认五秒”应按网络不活动和不同阶段来理解，不是整个业务动作的绝对总截止时间。[官方说明](https://www.python-httpx.org/advanced/timeouts/)

### 13.2 先看明确配置

```python
import httpx

timeout = httpx.Timeout(
    connect=3.0,
    read=10.0,
    write=10.0,
    pool=2.0,
)
```

四项分别控制不同等待。

连接超时，等待建立相应网络连接。

读取超时，等待接收数据。

写入超时，等待发送数据。

连接池超时，等待从池中取得可用连接资源。

### 13.3 `pool` 为什么也会超时？

假设客户端只允许一个连接。

第一项下载长期占用它，第二项请求就可能等待可用连接。

此时远端服务未必慢，DNS 也未必有问题；请求甚至可能还没真正发出去。

因此，需要把“排队等客户端资源”和“已经发出后等服务响应”分开诊断。

### 13.4 设置一个默认值，再覆盖部分阶段

```python
import httpx

timeout = httpx.Timeout(5.0, connect=2.0, pool=1.0)
```

默认五秒用于相应未单独覆盖的阶段，再把连接和池等待设置得更短。

如果你只提供部分阶段而没有默认值，其余必需字段没有定义，可能触发配置错误。

看清构造参数，比把某段代码原样复制过来更可靠。

### 13.5 `timeout=None` 是什么？

它可以关闭相应请求的超时限制。

这不是“自动使用更聪明的默认策略”。

如果没有外部取消和任务期限，调用可能等待很久。

长轮询、长期事件订阅等有特殊需求时可以主动设计，但普通接口调用通常应保留明确边界。

### 13.6 怎样给整个异步动作设置期限？

Python 3.11+ 可以使用：

```python
import asyncio

async with asyncio.timeout(3.0):
    response = await client.get("/books/1")
    response.raise_for_status()
```

这段依赖已经创建的异步 `client`，展示一个总范围控制。

期限通过取消等异步机制生效；它不是强制终止任意阻塞 C 调用的操作系统开关。

如果事件循环被同步阻塞代码占住，取消也无法在你期待的瞬间被调度。

### 13.7 为什么 MockTransport 的慢回调未必触发 ReadTimeout？

MockTransport 不是正常网络传输层。

一个回调 `await asyncio.sleep(2)`，并不自动等同于底层 socket 两秒没数据，因此客户端网络读取超时设置未必在那里生效。

可以让回调显式抛 `httpx.ReadTimeout` 来测试错误分支。

也可以用外层 `asyncio.timeout()` 测试协程期限。

但这两种测试的结论，与真实网络读取超时不同。

### 13.8 真实读取超时怎么练？

这次需要一个真正监听本机端口的程序。

先把下面代码保存为 `httpx_loopback_server.py`。

```python
import json
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlsplit


class LoopbackHandler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"

    def reply(self, status, body, content_type):
        self.send_response(status)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()

        try:
            self.wfile.write(body)
        except (BrokenPipeError, ConnectionResetError):
            pass

    def do_GET(self):
        path = urlsplit(self.path).path

        if path == "/slow":
            time.sleep(0.5)
            body = json.dumps({"message": "稍后返回"}, ensure_ascii=False).encode("utf-8")
            self.reply(200, body, "application/json; charset=utf-8")
        elif path == "/hello":
            body = json.dumps({"message": "你好，真实 HTTP"}, ensure_ascii=False).encode("utf-8")
            self.reply(200, body, "application/json; charset=utf-8")
        elif path == "/download":
            self.reply(200, bytes(range(256)) * 64, "application/octet-stream")
        else:
            self.reply(404, b'{"error":"not found"}', "application/json")


if __name__ == "__main__":
    server = ThreadingHTTPServer(("127.0.0.1", 8766), LoopbackHandler)
    print("本机练习服务：http://127.0.0.1:8766")

    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass
    finally:
        server.server_close()
```

在一个终端启动：

```bash
python httpx_loopback_server.py
```

通常显示：

```text
本机练习服务：http://127.0.0.1:8766
```

这个终端保持运行，再在另一个终端执行：

```python
import httpx

try:
    httpx.get(
        "http://127.0.0.1:8766/slow",
        timeout=httpx.Timeout(1.0, read=0.1),
    )
except httpx.ReadTimeout:
    print("真实回环读取等待按预期超时")
```

这一次确实经过本地网络连接与正常传输层。

如果端口占用，应换一个未占用端口并同步修改两端地址，不要随意终止不认识的进程。

### 13.9 这次为什么真的能产生读取超时？

客户端连接的是本机实际监听的端口。

服务收到 `/slow` 后，先执行 `time.sleep(0.5)`，等待约半秒。

客户端的读取等待限制是零点一秒。

因此，在通常的本地执行条件下，客户端先结束读取等待，抛出 `httpx.ReadTimeout`。

正常请求与失败请求应该成对观察。

先把路径换成 `/hello`，确认连接成功，并能解析返回内容。

再换回 `/slow`，观察特定超时。

如果连正常路径也访问不到，先检查服务是否成功启动、地址与端口是否一致。

### 13.10 `Content-Length` 在这个实验里负责什么？

服务采用 `HTTP/1.1`，普通响应明确写出正文的字节长度。

客户端据此知道这份响应的正文在哪里结束，不需要靠关闭连接来猜测边界。

这与连接复用有关，但这个教学服务并不覆盖真实 HTTP 服务的全部协议能力。

正文先被编码成 UTF-8 字节，所以 `len(body)` 描述的是字节数。

如果直接对中文字符串求长度，把字符数当成字节数发送，响应边界就可能错误。

### 13.11 客户端已经超时，服务端还会继续吗？

本例会。

服务端的等待没有因为客户端的异常自动取消。

等待结束后，它仍会尝试写出响应。

如果连接已经关闭，写入时可能触发断连异常；这里在写响应的局部处理这两种预期情况。

因此，客户端超时不能解释成“服务端刚才的操作已经撤回”。

对会修改数据的接口，是否重试、怎样查执行结果，仍然要遵守服务的业务约定。

### 13.12 服务放 Windows，客户端放 WSL，可以直接照抄吗？

首先要确认两端实际使用的网络环境。

同一个 `127.0.0.1` 字符串在跨环境访问时，是否到达预期服务，与当前 WSL 网络模式及相关设置有关。

最初练习建议把两端放在同一个 WSL 环境，先跑通本章的请求流向。

需要跨系统时，再确认监听地址、转发和防火墙等当前条件。

不要为了这个练习自动更改路由，也不必因为 Windows 可能启用了 TUN 就叠加应用代理。

练习结束后，在服务端终端按 `Ctrl+C` 停止程序。

## 第十四章：流式响应，怎样避免一次把整个文件放进内存？

### 14.1 普通 `.get()` 默认会读完响应体

即使使用 AsyncClient，普通：

```python
response = await client.get("/download")
```

也通常会把响应体读完，再返回给你。

异步改变等待组织，不自动改变整个文件的内存策略。

大型内容需要流式接口。

### 14.2 同步流式读取

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    with client.stream("GET", "/download") as response:
        response.raise_for_status()
        total = 0
        for chunk in response.iter_bytes():
            total += len(chunk)

print(total)
```

练习响应大小为：

```text
262144
```

这里按内容逐块消费。

不过本篇基础 Mock 响应本身用完整字节构造，因此这个练习验证流式 API 用法，不证明发送端也按固定内存生成巨大文件。

### 14.3 异步流式读取

```python
import asyncio
import httpx
from httpx_demo import demo_handler


async def main():
    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(demo_handler),
    ) as client:
        async with client.stream("GET", "/download") as response:
            response.raise_for_status()
            total = 0

            async for chunk in response.aiter_bytes():
                total += len(chunk)

    print(total)


if __name__ == "__main__":
    asyncio.run(main())
```

`async with` 管理流式响应范围。

`async for` 等待异步迭代器的下一块。

`aiter_bytes()` 的 `a` 提示异步接口，与同步 `iter_bytes()` 对应。

### 14.4 为什么 `.content` 可能报“尚未读取”？

流式模式下，响应体还没有被整体读入。

此时直接访问完整内容属性，可能触发 `ResponseNotRead` 等相关错误。

你可以选择按块消费，或者明确调用同步 `read()`／异步 `aread()` 取得全部内容。

一旦选择整体读取，就要接受相应内存开销。

### 14.5 `iter_bytes()` 和 `iter_raw()` 的区别

`iter_bytes()` 面向内容解码后的字节，例如处理常见 HTTP 内容压缩。

`iter_raw()` 让你在相应层次取得未执行该内容解码的原始字节。

它也不是完整的网络抓包，不会把所有 TCP/TLS 报文原样交给你。

计算摘要和核对长度时，要先明确协议里的摘要或大小描述的是哪一层。

### 14.6 文本流与逐行流

同步可用 `iter_text()`、`iter_lines()`。

异步可用 `aiter_text()`、`aiter_lines()`。

文本迭代管理解码边界；行迭代管理行边界。

网络块边界不一定与 UTF-8 字符或记录边界一致，所以不要直接对每个任意字节块独立 `.decode()` 再假设不会切断字符。

### 14.7 只读前面一点就退出，会不会泄露连接？

如果流式响应放在正确的上下文中，离开时会安排关闭。

```python
with client.stream("GET", "/download") as response:
    response.raise_for_status()
    for chunk in response.iter_bytes():
        print("第一块大小：", len(chunk))
        break
```

不要把未消费的响应随意返回出去，然后忘记由谁关闭。

如果确实要转交资源，接口应明确所有权和关闭责任。

## 第十五章：完整实践——同步流式下载与大小上限

这次把内容保存到练习目录。

我们先验证状态，再写临时文件；只有完整消费结束后才替换正式结果。

```python
import hashlib
import tempfile
from pathlib import Path


def download_sync(client, path, destination, max_bytes=10 * 1024 * 1024):
    if type(max_bytes) is not int or max_bytes < 0:
        raise ValueError("max_bytes 必须是非负整数")

    destination = Path(destination).absolute()
    destination.parent.mkdir(parents=True, exist_ok=True)
    temporary = None
    total = 0
    digest = hashlib.sha256()

    try:
        with client.stream("GET", path) as response:
            response.raise_for_status()

            with tempfile.NamedTemporaryFile(
                mode="wb",
                dir=destination.parent,
                prefix=destination.name + ".",
                suffix=".part",
                delete=False,
            ) as target:
                temporary = Path(target.name)

                for chunk in response.iter_bytes():
                    total += len(chunk)
                    if total > max_bytes:
                        raise ValueError("下载超过大小上限")
                    target.write(chunk)
                    digest.update(chunk)

        temporary.replace(destination)
        return {"bytes": total, "sha256": digest.hexdigest()}
    finally:
        if temporary is not None:
            temporary.unlink(missing_ok=True)
```

使用：

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    info = download_sync(client, "/download", "httpx_practice/result.bin")

print(info["bytes"])
```

### 15.1 为什么先检查状态，再打开目标？

服务可能返回 404 JSON 或 500 HTML。

如果你不检查就保存，最后可能得到一个扩展名叫 `.bin` 的错误页。

检查在前，可以让很多明确失败发生在创建下载结果之前。

### 15.2 为什么累计实际接收大小？

响应头可能缺失或对应不同编码层。

只有读取步骤本身实施上限，才能避免“先检查了一下头，后面仍然无限读取”。

这个上限限制的是交给应用的内容字节，不等于完整的网络资源隔离。

### 15.3 为什么只在结束后替换？

下载中途失败时，旧正式结果可以继续保留。

临时文件在 `finally` 中清理。

这不提供多文件事务，也不解决多个写入者互相覆盖；它明确采用“新下载成功后替换旧结果”的政策。

### 15.4 需要更强的持久性怎么办？

还要考虑刷新、文件同步、目录元数据与底层存储行为。

这个教学函数主要改善完整结果发布边界，没有宣称在任意断电故障下都不会丢失数据。

重要资料需要根据实际可靠性要求设计，并保留可恢复来源。

## 第十六章：异步下载，网络异步了，本地文件也自动异步吗？

### 16.1 一个便于理解的实现

```python
import tempfile
from pathlib import Path


async def download_async(client, path, destination, max_bytes=10 * 1024 * 1024):
    if type(max_bytes) is not int or max_bytes < 0:
        raise ValueError("max_bytes 必须是非负整数")

    destination = Path(destination).absolute()
    destination.parent.mkdir(parents=True, exist_ok=True)
    temporary = None
    total = 0

    try:
        async with client.stream("GET", path) as response:
            response.raise_for_status()

            with tempfile.NamedTemporaryFile(
                mode="wb",
                dir=destination.parent,
                prefix=destination.name + ".",
                suffix=".part",
                delete=False,
            ) as target:
                temporary = Path(target.name)

                async for chunk in response.aiter_bytes():
                    total += len(chunk)
                    if total > max_bytes:
                        raise ValueError("下载超过大小上限")
                    target.write(chunk)

        temporary.replace(destination)
        return total
    finally:
        if temporary is not None:
            temporary.unlink(missing_ok=True)
```

使用时，把它放在前面已经创建的 AsyncClient 范围里：

```python
size = await download_async(client, "/download", "httpx_practice/async.bin")
```

### 16.2 为什么文件这里仍然是普通 `with`？

标准库普通磁盘文件对象提供同步读写接口。

`target.write(chunk)` 没有因为外层函数是 `async def` 就变成非阻塞磁盘 I/O。

这个例子适合规模可控的练习文件，用来清楚展示异步网络流与文件发布的连接。

在慢磁盘、网络挂载、大量并发文件操作等情况下，同步写入可能阻塞事件循环。

### 16.3 那我给 `write()` 加一个 await 可以吗？

不可以凭空这样改。

普通文件的 `write()` 返回写入数量等同步结果，不是一个可等待操作。

```python
# 错误示意：普通文件接口不会因为加 await 自动异步化。
await target.write(chunk)
```

应选择明确支持异步文件操作的接口，或把合适的同步工作放到线程执行。

### 16.4 把整段同步下载放到线程也是一种选择

如果你的项目主要是同步逻辑，只需要避免阻塞一个已有事件循环，可以封装整个同步工作：

```python
import asyncio
from httpx_demo import make_sync_client


def download_job(destination):
    with make_sync_client() as client:
        return download_sync(client, "/download", destination)


async def main():
    info = await asyncio.to_thread(download_job, "httpx_practice/threaded.bin")
    print(info["bytes"])


if __name__ == "__main__":
    asyncio.run(main())
```

这里复用前面定义的 `download_sync()`。

线程函数内部创建和关闭同步客户端，资源归属比较明确。

### 16.5 取消等待线程，线程就一定停了吗？

不是。

`asyncio.to_thread()` 不提供强制终止任意正在运行线程函数的能力。

调用方取消等待后，底层同步工作可能还在继续。

因此，下载函数本身仍需要超时、大小限制和合适的停止政策。

不要把异步取消误当成可以随时杀掉任何同步 I/O 的保证。

### 16.6 为什么不随便每块启动一个后台写任务？

如果不等待写完就关闭文件，后台任务可能还在访问已关闭对象。

如果同时写多个块，还可能打乱顺序。

如果把写入无限排队，内存也可能不断增加。

正确异步化需要同时考虑顺序、背压、取消和关闭，不只是把一个函数包成任务。

## 第十七章：构造真正逐段产生的异步响应

基础练习回调把完整下载内容放在内存中。

如果我们想观察每一块异步产生、提前停止和关闭行为，可以定义受控字节流。

### 17.1 自定义一个异步字节流

```python
import asyncio
import httpx


class PracticeAsyncStream(httpx.AsyncByteStream):
    def __init__(self):
        self.closed = False

    async def __aiter__(self):
        for chunk in [b"first\n", b"second\n", b"third\n"]:
            await asyncio.sleep(0.01)
            yield chunk

    async def aclose(self):
        self.closed = True
```

`__aiter__()` 提供异步迭代过程。

`yield` 交出一个字节块。

`await asyncio.sleep()` 在块之间提供受控等待。

`aclose()` 记录关闭状态，便于观察资源管理。

### 17.2 交给 MockTransport

```python
import asyncio
import httpx


async def main():
    stream = PracticeAsyncStream()

    def handler(request):
        return httpx.Response(200, stream=stream)

    async with httpx.AsyncClient(
        transport=httpx.MockTransport(handler),
    ) as client:
        async with client.stream("GET", "https://demo.example.invalid/events") as response:
            async for line in response.aiter_lines():
                print(line)
                if line == "second":
                    break

        print("响应流已关闭：", stream.closed)


if __name__ == "__main__":
    asyncio.run(main())
```

先定义上一节的 `PracticeAsyncStream`，再运行本节。

通常输出：

```text
first
second
响应流已关闭： True
```

### 17.3 为什么没有打印第三行？

因为应用主动停止消费。

退出响应上下文后，流关闭，后面没有继续要求下一块。

这正是“读取全部数据”与“正确管理未读完资源”的区别。

### 17.4 这个异步流能直接交给同步 Client 吗？

不能假设可以。

同步客户端需要同步传输与同步字节流接口。

异步客户端使用异步传输与异步字节流接口。

MockTransport 可以接受不同处理方式，但回调返回的响应流仍要与调用端要求相匹配。

### 17.5 按行处理是否可以忽略协议？

仍然不可以。

如果内容是 JSON Lines，每行可以按它的记录约定解析。

如果内容是 SSE，事件可能跨多行，还包含字段名与空行规则。

如果内容是任意二进制文件，换行字节甚至可能只是数据的一部分。

先知道格式，再选择行迭代或字节迭代。

## 第十八章：异步上传与请求体流

### 18.1 普通 JSON 上传仍然很简单

```python
response = await client.post("/echo", json={"title": "异步学习", "count": 3})
response.raise_for_status()
data = response.json()
```

只有请求等待是异步接口。

JSON 对象准备、解析和后续普通 Python 计算仍然是同步执行。

特别大的 JSON 编码或解码也可能占用事件循环线程，不能因为用上 AsyncClient 就忽略 CPU 工作量。

### 18.2 用异步生成器提供原始请求体

```python
import asyncio
import httpx
from httpx_demo import demo_handler


async def generate_body():
    for piece in [b"hello", b"-", b"httpx"]:
        await asyncio.sleep(0.01)
        yield piece


async def main():
    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(demo_handler),
    ) as client:
        response = await client.post(
            "/echo",
            content=generate_body(),
            headers={"Content-Type": "application/octet-stream"},
        )
        response.raise_for_status()
        print(response.json()["data"])


if __name__ == "__main__":
    asyncio.run(main())
```

通常得到总字节大小十一的记录：

```text
{'raw_size': 11}
```

### 18.3 为什么选择 `content=`？

我们在生成整个原始请求体，而不是表单字段映射。

异步生成器每次交出字节块，客户端按对应方式发送。

真实网络服务还需要支持相应请求体传输方式，不应假设所有简化服务器都能接收任意分块上传。

### 18.4 MockTransport 是否证明上传逐块经过网络？

没有。

它会在自身流程中消费请求内容，再交给受控处理函数，因此主要验证你提供的字节和接口组织是否正确。

真实网络上的背压、写入超时和服务端分块解析，需要真实传输测试。

### 18.5 文件上传中的本地读取仍可能同步

即使外层是 AsyncClient，普通磁盘文件与 multipart 编码链路也可能进行同步文件读取。

对小型文件练习通常容易理解；大量并发巨型文件则应评估读取方式和内存策略。

不要把“HTTP 网络接口异步”扩大为“所有内部操作都不阻塞”。

## 第十九章：完整实践——异步分页读取图书

### 19.1 为什么不能一次 gather 所有页？

你可能还不知道一共有多少页。

接口可能采用游标，下一页信息需要从上一页取得。

即使知道总页数，过度并发也可能造成限流或数据一致性问题。

先实现清楚的顺序分页，再按契约决定是否优化。

### 19.2 一个异步生成器

```python
import httpx


async def iter_books_async(client, page_size=2, max_pages=100):
    if type(page_size) is not int or not 1 <= page_size <= 100:
        raise ValueError("page_size 超出范围")
    if type(max_pages) is not int or max_pages < 1:
        raise ValueError("max_pages 必须是正整数")

    seen_ids = set()

    for page in range(1, max_pages + 1):
        response = await client.get(
            "/books",
            params={"page": page, "page_size": page_size},
        )
        response.raise_for_status()
        data = response.json()

        if not isinstance(data, dict):
            raise ValueError("分页顶层必须是对象")
        if type(data.get("page")) is not int or data["page"] != page:
            raise ValueError("返回页号与请求不一致")

        items = data.get("items")
        if not isinstance(items, list):
            raise ValueError("items 必须是数组")

        if not items:
            return

        for book in items:
            if not isinstance(book, dict) or type(book.get("id")) is not int:
                raise ValueError("图书记录结构不正确")
            if book["id"] in seen_ids:
                raise ValueError("分页出现重复 ID")
            seen_ids.add(book["id"])
            yield book

        if len(items) < page_size:
            return

    raise RuntimeError("达到分页上限，未确认读取完成")
```

消费它：

```python
async for book in iter_books_async(client):
    print(book["id"], book["title"])
```

### 19.3 `async for` 在等什么？

每次需要下一条记录时，异步生成器可能需要请求下一页。

它可以在内部 `await client.get(...)`，等待后继续产生图书。

普通生成器不能直接在普通函数体中使用 `await`。

所以这里的函数和消费语法都需要相应异步协议。

### 19.4 为什么仍然检查重复 ID？

为了发现参数没生效、服务重复返回某页，或数据在分页中发生变化等情况。

但去重不等于完整快照。

记录在分页期间插入或删除，仍然可能遗漏，真正一致性需要服务端协议支持。

### 19.5 提前停止迭代怎么办？

普通分页函数每次非流式请求已完成时，单条响应通常已经读完。

但异步生成器自身仍有生命周期；如果内部以后改为持有长期流式响应，应明确关闭责任。

必要时可用 `contextlib.aclosing()` 管理异步生成器的提前退出。

```python
from contextlib import aclosing

async with aclosing(iter_books_async(client)) as books:
    async for book in books:
        print(book["title"])
        break
```

这个模式表达即使提前停止，也安排异步生成器退出和清理。

## 第二十章：Cookie、认证和重定向

### 20.1 Cookie 放在客户端范围管理

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    client.get("/cookie").raise_for_status()
    response = client.get("/echo")
    response.raise_for_status()
    print(response.json()["cookie"])
```

Cookie 有域和路径范围，不是保存后就发送到任意站点。

处理身份状态时，应该让相应 Cookie 容器管理匹配，而不随意把所有 Cookie 拼进每一个请求。

### 20.2 BasicAuth

```python
import httpx

auth = httpx.BasicAuth("demo", "practice")
```

可以在客户端默认配置或单次请求中使用。

真实远端 Basic 认证通常应配合正确验证的 HTTPS。

Basic 的 Base64 表示不是保密加密，不能因为头看起来不像明文就忽略传输保护。

### 20.3 Bearer Token

```python
headers = {"Authorization": f"Bearer {token}"}
```

这段依赖你已经从受控配置取得 `token`。

不要把真实令牌硬编码到公开源码，也不要在日志里打印完整认证头。

HTTPX 负责发送协议字段，令牌权限和失效政策仍然由服务决定。

### 20.4 默认为什么不自动跟随重定向？

HTTPX 的默认设计要求你明确选择跟随。

```python
response = client.get("/redirect", follow_redirects=True)
```

这时最终响应通常来自 `/hello`。

可以检查：

```python
print(response.url)
print([item.status_code for item in response.history])
```

不跟随时，302 本身就是你取得的响应，需要按状态和 Location 处理。

### 20.5 重定向的目标也需要允许

特别是服务器端下载或代理请求功能，初始 URL 合法不代表后续跳转目标也可以访问。

URL 白名单、网络范围和身份信息传播等，都需要应用层规则。

库能处理协议跳转，不会代替你决定哪些目标被授权。

## 第二十一章：准备请求、手动发送与流式关闭

### 21.1 `build_request()` 不会立刻联网

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    request = client.build_request(
        "POST",
        "/echo",
        json={"title": "准备请求"},
    )

    print(request.method)
    print(request.url)
    print(request.headers.get("Content-Type"))
```

它把客户端配置与请求参数组织成请求对象。

适合调试编码、构造签名所需的受控资料或设计测试。

不要把请求对象包含秘密的头和正文无条件输出。

### 21.2 `send()` 才进入发送过程

```python
with make_sync_client() as client:
    request = client.build_request("GET", "/hello")
    response = client.send(request)
    response.raise_for_status()
    print(response.json())
```

异步版本对应：

```python
request = client.build_request("GET", "/hello")
response = await client.send(request)
```

`build_request()` 本身仍是普通同步构造，通常不需要 await。

### 21.3 手动流式发送时，谁关闭？

```python
response = client.send(request, stream=True)
try:
    response.raise_for_status()
    for chunk in response.iter_bytes():
        print(len(chunk))
finally:
    response.close()
```

异步版本则使用 `await response.aclose()`。

如果你离开高层 `client.stream(...)` 上下文管理形式，就要自己明确资源释放责任。

普通需求优先采用容易看出范围的上下文写法。

### 21.4 请求体能不能发送两遍？

如果请求体来自一次性流或生成器，第一次发送后可能已经消费。

再次发送同一个对象，可能无法重新读取，或者触发流已消费错误。

重试和重定向若要求重放正文，就需要正文来源可重放，或按服务协议重新构造。

这与“请求方法是否允许重复”又是不同问题：语义允许，不代表流对象还能原样重用。

## 第二十二章：事件钩子，怎样观察请求而不改变业务代码？

### 22.1 同步钩子

```python
import httpx
from httpx_demo import demo_handler


def log_request(request):
    print("准备发送：", request.method, request.url.path)


def log_response(response):
    print("收到状态：", response.status_code)


with httpx.Client(
    base_url="https://books.example.invalid",
    transport=httpx.MockTransport(demo_handler),
    event_hooks={"request": [log_request], "response": [log_response]},
) as client:
    response = client.get("/hello")
    response.raise_for_status()
```

钩子接收请求或响应对象，适合集中记录必要信息。

HTTPX 钩子的用法不应套用为“返回另一个对象来替换原响应”的通用机制。

普通观察回调直接完成记录即可。

### 22.2 AsyncClient 的钩子也要异步定义

```python
async def log_request(request):
    print("准备发送：", request.method, request.url.path)


async def log_response(response):
    print("收到状态：", response.status_code)
```

把这些异步函数放入 AsyncClient 的 `event_hooks`。

不要把同步函数原样传进一个要求等待回调的异步钩子位置，再在运行错误后随意加装饰器掩盖类型差异。

### 22.3 为什么响应钩子里不直接读取整个正文？

钩子执行时，响应体可能尚未按调用者计划消费。

如果你为了打印一条日志而执行 `response.read()` 或 `await response.aread()`，可能提前把巨大内容读入内存。

这会改变流式调用的性能和资源行为。

状态、路径、媒体类型和请求标识通常已经足够完成很多诊断。

### 22.4 日志不要泄露完整 URL 中的秘密

有些服务把令牌或个人标识放在查询中。

只记录 `request.url.path` 可以减少暴露查询值的风险，但路径本身也可能含敏感资料。

日志字段应按应用数据政策选择，不能把“框架提供一个可打印对象”当成“可以公开全部内容”。

## 第二十三章：重试应该放在哪里？

### 23.1 先问请求是否可以重复执行

读取图书通常容易重试。

创建订单、发送通知、扣减库存等操作可能具有副作用。

客户端读取超时，不能证明服务器没有完成操作。

所以重试策略必须同时考虑传输错误和业务幂等约定。

### 23.2 底层传输的连接重试

```python
import httpx

transport = httpx.HTTPTransport(retries=2)

with httpx.Client(transport=transport, timeout=5.0) as client:
    pass
```

这展示传输配置，不会因为创建对象就访问网络。

底层这个重试入口主要处理相应连接建立错误，不是对所有读取失败、HTTP 503 和业务失败自动重试的万能策略。

异步传输有相应 `AsyncHTTPTransport`。

### 23.3 受控 GET 的异步有限重试

```python
import asyncio
import httpx


async def get_with_retry(client, path, attempts=3):
    if type(attempts) is not int or attempts < 1:
        raise ValueError("attempts 必须是正整数")

    for index in range(attempts):
        try:
            response = await client.get(path)
            response.raise_for_status()
            return response
        except (httpx.ConnectError, httpx.ConnectTimeout, httpx.ReadTimeout):
            if index + 1 == attempts:
                raise
            await asyncio.sleep(0.1 * (2 ** index))
```

这个函数主动限定为 GET，并只对列出的异常重试。

HTTP 状态错误没有被这个分支捕获，所以不会自动反复请求 401 或 404。

### 23.4 为什么不是 `except Exception` 全部再试？

JSON 结构错误、编程错误、错误的参数类型和服务明确拒绝，都不一定靠等待后再发一次解决。

宽泛捕获会掩盖本应修复的问题，还可能让服务承受额外压力。

先列出允许重试的失败类别，再制定次数、退避和总期限，政策更容易检查。

### 23.5 重试次数与任务期限要一起考虑

每一次尝试都有连接、读取和排队等待。

再叠加退避后，总耗时可能明显超过一次请求的超时设置。

异步场景可以用外层总期限限制整段尝试，但仍需允许取消正常传播并安排资源清理。

### 23.6 状态码重试也需要读取协议

429 可能带 `Retry-After`，503 也可能只是短期不可用。

但服务建议的等待可能超过你的任务期限。

如果你实现状态重试，应限制等待和总预算，关闭不再使用的响应，并记录最终失败。

不要在持有未读完流式响应时睡很久，把池中连接一直占着。

## 第二十四章：TLS、代理与环境设置

### 24.1 验证服务器身份是正常流程的一部分

HTTPS 客户端通常应保留证书验证。

证书失败可能涉及目标地址、系统时间、信任链或企业 CA 配置。

不应把长期设置 `verify=False` 当成普通修复。

本篇离线 MockTransport 不经历真实 TLS，所以也不会验证实际证书配置。

### 24.2 使用明确的 SSLContext

```python
import ssl
import httpx

context = ssl.create_default_context(cafile="/path/to/trusted-ca.pem")

with httpx.Client(verify=context) as client:
    pass
```

路径是示意，需要换成你确认可信且真实存在的 CA 文件。

使用 SSLContext 可以明确组织信任配置。

0.28 系列中一些旧式字符串证书参数已有弃用变化，迁移时应看目标版本官方说明。

### 24.3 客户端证书

服务要求双向 TLS 时，可以在受控上下文中加载客户端证书和私钥：

```python
context.load_cert_chain(
    certfile="client-cert.pem",
    keyfile="client-key.pem",
)
```

这与验证服务器身份是两个方向。

不要把私钥放进文档、日志或提交历史。

### 24.4 当前版本用 `proxy=`

```python
import httpx

with httpx.Client(proxy="http://127.0.0.1:7890") as client:
    pass
```

这里只演示显式配置形式，端口是占位示例，不表示你的电脑一定提供这个代理。

0.28.1 不应照抄旧教程的 `proxies=` 关键字。

多目标传输路由还可以通过 mounts 等机制设计，但普通应用应先保持配置清楚，不必为了一个请求建立复杂路由表。

### 24.5 Windows TUN 与应用代理不要自动叠加

TUN 主要影响系统网络路由。

HTTP/SOCKS 代理是应用连接到转发服务的另一种方式。

宿主系统使用 TUN，不意味着 WSL 内 Python 必须再设置一个硬编码代理。

先检查当前连通性与继承环境，再按用户明确选择使用代理。

### 24.6 `trust_env` 影响什么？

客户端默认可以读取相关环境设置，例如代理和证书环境配置。

受控离线或回环测试时，可以主动选择隔离继承环境：

```python
with httpx.Client(trust_env=False) as client:
    pass
```

但这是测试客户端的明确选择，不应未经分析就修改真实项目全部请求。

它不是一个只影响单一代理变量的全局“修网络”按钮。

### 24.7 为什么同一段程序在 IDE 和终端不同？

启动器可能继承不同环境变量，选择不同解释器，甚至运行在不同系统环境。

先打印解释器路径，检查当前网络设置是否存在，再核对服务地址与监听位置。

不要把这些运行环境差异都归咎于某个请求头。

## 第二十五章：完整实践——业务客户端与资源所有权

我们来组织一个很小的异步图书客户端。

它不自行隐藏创建一个永久存在的网络对象，而是由调用者传入已经管理生命周期的 AsyncClient。

### 25.1 这样调用

```python
async with httpx.AsyncClient(...) as http_client:
    books = BookApi(http_client)
    book = await books.get_book(1)
```

外层负责网络客户端的关闭。

业务对象只负责调用图书接口和验证结构。

### 25.2 完整实现

```python
import httpx


class BookApiError(RuntimeError):
    pass


class BookApi:
    def __init__(self, client):
        self.client = client

    async def get_book(self, book_id):
        if type(book_id) is not int or book_id < 1:
            raise ValueError("book_id 必须是正整数")

        try:
            response = await self.client.get(f"/books/{book_id}")
            response.raise_for_status()
        except httpx.HTTPError as error:
            raise BookApiError(f"读取图书 {book_id} 失败") from error

        try:
            data = response.json()
        except ValueError as error:
            raise BookApiError("图书接口未返回合法 JSON") from error

        if not isinstance(data, dict):
            raise BookApiError("图书响应顶层必须是对象")
        if type(data.get("id")) is not int or data["id"] != book_id:
            raise BookApiError("图书 ID 与请求不一致")
        if not isinstance(data.get("title"), str) or not data["title"].strip():
            raise BookApiError("图书标题不合法")
        if type(data.get("price")) is not int or data["price"] < 0:
            raise BookApiError("图书价格必须是非负整数")

        return {"id": data["id"], "title": data["title"], "price": data["price"]}
```

运行：

```python
import asyncio
import httpx
from httpx_demo import demo_handler


async def main():
    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(demo_handler),
    ) as http_client:
        books = BookApi(http_client)
        first, second = await asyncio.gather(
            books.get_book(1),
            books.get_book(2),
        )
        print(first["title"])
        print(second["title"])


if __name__ == "__main__":
    asyncio.run(main())
```

### 25.3 为什么不在每个 `get_book()` 里新建 AsyncClient？

那样会反复创建和关闭连接池，失去适当复用机会。

也让超时、认证和其他配置散落到每个函数里。

统一传入一个管理好的客户端，调用者能决定它覆盖一次批量工作还是整个应用生命周期。

### 25.4 业务对象要不要自己 `close()`？

本例不拥有传入客户端，因此不在业务方法里擅自关闭它。

否则另外一个业务对象还在使用同一客户端时，资源可能突然失效。

所有权不是靠变量名字猜的，应在类设计和说明中明确。

### 25.5 解析成功为什么还要复制字段？

返回一份明确的结果字典，可以控制后续代码依赖的字段范围。

如果服务新增内部字段，不会自动进入你的公开业务结果。

复杂模型可以使用 Pydantic 等工具组织验证，但工具也需要你定义允许的类型、范围和未知字段政策。

### 25.6 为什么价格用整数？

这个练习把价格定义为某个约定单位的非负整数。

真实金额协议应说明单位与精度，不能只凭字段叫 `price` 就猜是元、分或任意浮点数。

选择整数只是本例模型规则，不是 HTTPX 自动提供的金额系统。

## 第二十六章：客户端生命周期，几个经常写错的位置

### 26.1 在循环里反复创建客户端

```python
# 这种结构会为每项反复创建客户端，通常不利于连接复用。
for item in items:
    async with httpx.AsyncClient() as client:
        response = await client.get(item["url"])
```

如果这些工作属于同一受控批次，通常把客户端上下文放在循环外更清楚。

```python
async with httpx.AsyncClient() as client:
    for item in items:
        response = await client.get(item["url"])
```

两段都是依赖已有输入的异步代码片段，应放在协程中。

### 26.2 把任务返回后，客户端已经关闭

```python
# 反面结构：不要让返回的后台任务继续依赖即将关闭的客户端。
async def start_work():
    async with httpx.AsyncClient() as client:
        return asyncio.create_task(client.get("https://example.invalid"))
```

函数返回时退出客户端上下文，但任务可能还没完成。

应在同一资源范围内等待任务结束，或把客户端生命周期交给更高一层管理。

### 26.3 跨多个 `asyncio.run()` 复用同一 AsyncClient

异步网络资源往往与相应运行环境和事件循环生命周期有关。

不要随意把一个已在某次运行中使用的客户端，保留到另一套独立事件循环继续调用。

更清楚的方式是在一个顶层异步主函数中管理整个工作，并在退出前关闭客户端。

### 26.4 全局对象就一定不好吗？

关键不在于变量写在哪一行，而在于初始化、使用范围和关闭由谁管理。

应用可以有一个长生命周期客户端，但应有明确启动和结束流程。

小脚本通常不需要把客户端放进复杂全局管理器；一个清楚的 `main()` 与上下文已经足够。

### 26.5 异常或取消时，清理范围仍要覆盖资源

网络请求、流式响应、临时文件和工作任务各自有生命周期。

把它们合理嵌套，可以让正常退出和异常展开更容易推理。

如果把资源对象四处返回、任务随意创建、关闭责任隐藏在不相关函数中，程序即使能运行，也很难说明失败后究竟留下什么。

## 第二十七章：离线测试和真实回环测试，各自能证明什么？

### 27.1 MockTransport 适合验证请求与业务逻辑

例如检查查询参数是否正确：

```python
import httpx

seen = []


def handler(request):
    seen.append(request)
    return httpx.Response(200, json={"ok": True})


with httpx.Client(
    base_url="https://demo.example.invalid",
    transport=httpx.MockTransport(handler),
) as client:
    response = client.get("/items", params={"page": 2})
    response.raise_for_status()

assert seen[0].url.path == "/items"
assert seen[0].url.params["page"] == "2"
```

没有网络依赖，输入输出都可控。

### 27.2 显式模拟传输失败

```python
import httpx


def failing_handler(request):
    raise httpx.ConnectError("练习连接失败", request=request)


try:
    with httpx.Client(transport=httpx.MockTransport(failing_handler)) as client:
        client.get("https://demo.example.invalid")
except httpx.ConnectError:
    print("错误处理分支按预期执行")
```

这验证调用者遇到该异常时的行为。

它没有证明真实 DNS 或套接字已经失败。

测试报告应按照实际发生的机制描述结果。

### 27.3 用状态序列测试有限重试

可以让回调第一项抛连接错误，第二项返回成功，再检查请求次数。

这样的测试比故意断开用户网络更稳定，也不会干扰其他程序。

如果策略只重试连接错误，还应测试 404 不重试、业务结构错误不重试，以及达到次数上限后正常失败。

### 27.4 回环服务适合验证真实传输

使用第十三章已经搭建的本机服务，可以验证真实读取超时、HTTP 请求体和普通响应传输。

服务与客户端都在同一个受控环境中，变量相对少。

但它仍不能证明公网 DNS、企业代理和真实远端证书链在你的部署环境里正常。

### 27.5 性能测试为什么不能只看 MockTransport？

MockTransport 跳过很多网络步骤。

你测到的可能主要是 Python 回调、编码和业务处理开销。

真实网络还有握手、拥塞、连接保持、服务处理和带宽等影响。

可以用模拟延迟理解并发原理，但不能把它直接当成生产吞吐基准。

### 27.6 为什么要验证失败后的资源状态？

成功案例容易看到“拿到了结果”。

失败路径才会暴露未关闭响应、部分文件、遗留任务和错误吞掉等问题。

因此，下载测试应检查超限后旧文件是否保留；异步测试应检查提前退出后流是否关闭；分页测试应检查达到上限时有没有冒称完整成功。

这类验证比只断言某个函数返回对象更有实际价值。

## 第二十八章：读者经常会继续问的十个问题

### 28.1 “我已经 async 了，为什么 CPU 还是很高？”

JSON 编解码、压缩解压、图像处理或你自己的循环，仍然可能是 CPU 工作。

异步 I/O 不会自动把这些工作分配到多个核心。

先测量究竟哪一段耗时，再决定是否使用线程、进程或算法改进。

### 28.2 “一次开一千个协程，为什么反而更慢？”

连接池可能排队，服务器可能限流，磁盘写入可能成为瓶颈，任务对象和结果也可能增加内存开销。

并发上限需要根据服务能力和任务大小选择，不是越大越好。

### 28.3 “MockTransport 用 HTTPS 地址，是否测试了证书？”

没有。

地址对象可以保存 `https` 方案，但模拟传输没有建立真实 TLS 连接。

验证地址准备和验证证书，是两项不同工作。

### 28.4 “为什么设置 `timeout=1`，请求两秒才结束？”

先确认等待发生在哪一层。

模拟回调、同步阻塞代码、重试退避、读取持续有数据等情况，都可能与你想象的“一秒总期限”不同。

需要整体异步期限时，明确设置范围；需要真实网络阶段限制时，检查对应 Timeout 配置。

### 28.5 “能不能把普通 requests.get 放在 async def 里面？”

语法上可以调用普通同步函数，但它会同步阻塞当前事件循环线程。

如果要保持异步网络等待，使用 AsyncClient。

如果已有同步逻辑必须复用，可以考虑受控线程调用，并理解取消边界。

### 28.6 “客户端关闭后，为什么已经得到的字典还能读？”

字典是解析后保存在内存的数据。

它不需要保持网络连接才能访问。

未读完的流式响应则依赖仍然可用的流资源，这两种状态要区分。

### 28.7 “为什么 `await response.json()` 报类型错误？”

普通 `.json()` 已经返回 Python 对象。

它不是异步接口。

`await` 不能把任意值变成等待操作，应按函数本身的返回约定使用。

### 28.8 “为什么服务返回错误，我的 except RequestError 没捕获？”

如果服务正常返回 404，而你没有调用状态检查，传输本身可能已经成功。

调用 `raise_for_status()` 后产生的是状态异常类别，应按相应层次捕获。

不要把服务错误响应与完全没有响应的网络失败混在一起。

### 28.9 “限了连接数，还要信号量吗？”

如果你还要限制业务工作数量、解码和落盘并发，应用信号量仍然可能有意义。

连接数只控制传输资源，不能代替所有工作调度。

### 28.10 “是不是每个项目都应该从 Requests 换成 HTTPX？”

先看当前项目需求和运行模型。

同步脚本已有稳定客户端，不一定需要迁移。

异步应用、统一同步异步接口或特定协议需求可能使 HTTPX 更合适。

选择工具应服务于具体任务，也要评估迁移中的重定向、异常、超时和参数差异。

## 第二十九章：带参考答案的练习

下面练习默认已经按第三章保存完整 `httpx_demo.py`。

异步片段需要放进协程中；完整程序会明确给出入口。

### 29.1 练习一：最小同步请求

要求：读取第一本书，打印标题，检查状态。

```python
from httpx_demo import make_sync_client

with make_sync_client() as client:
    response = client.get("/books/1")
    response.raise_for_status()
    book = response.json()

assert book["id"] == 1
print(book["title"])
```

预期标题是：

```text
Python 文件处理
```

先预测 `book` 类型，再运行确认它与 response 对象不同。

### 29.2 练习二：重复参数与空值

```python
import httpx

params = httpx.QueryParams([
    ("tag", "python"),
    ("tag", "backend"),
    ("empty", None),
])

assert params.get_list("tag") == ["python", "backend"]
assert params["empty"] == ""
```

请解释：如果接口要求不发送 `empty`，应该在构造时排除它，而不是依赖 None 自动省略。

### 29.3 练习三：重定向默认行为

```python
import httpx
from httpx_demo import make_sync_client

with make_sync_client() as client:
    first = client.get("/redirect")
    second = client.get("/redirect", follow_redirects=True)

assert first.status_code == 302
assert second.status_code == 200
assert [item.status_code for item in second.history] == [302]
```

再调用 `first.raise_for_status()`，观察 HTTPX 对 3xx 的处理。

### 29.4 练习四：异步请求不是每一行都 await

```python
import asyncio
import httpx
from httpx_demo import demo_handler


async def main():
    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(demo_handler),
    ) as client:
        response = await client.get("/hello")
        response.raise_for_status()
        data = response.json()
        assert data["message"] == "你好，HTTPX"


if __name__ == "__main__":
    asyncio.run(main())
```

把真正需要等待的调用和已经在内存中执行的普通方法分别圈出来。

### 29.5 练习五：验证并发上限

构造一个异步回调，进入时增加 `active`，退出时减少，保存见过的最大值。

```python
import asyncio
import httpx


async def main():
    state = {"active": 0, "peak": 0}
    semaphore = asyncio.Semaphore(2)

    async def handler(request):
        state["active"] += 1
        state["peak"] = max(state["peak"], state["active"])
        try:
            await asyncio.sleep(0.01)
            return httpx.Response(200, json={"ok": True})
        finally:
            state["active"] -= 1

    async with httpx.AsyncClient(transport=httpx.MockTransport(handler)) as client:
        async def work(index):
            async with semaphore:
                response = await client.get(f"https://demo.example.invalid/{index}")
                response.raise_for_status()

        await asyncio.gather(*(work(index) for index in range(8)))

    assert state["active"] == 0
    assert state["peak"] <= 2
    print(state)


if __name__ == "__main__":
    asyncio.run(main())
```

这个测试验证应用信号量，不验证真实连接池。

### 29.6 练习六：分页完整性

使用前面定义的 `iter_books_async()`：

```python
books = [book async for book in iter_books_async(client, page_size=2)]
assert [book["id"] for book in books] == [1, 2, 3, 4, 5]
```

再把 `max_pages` 设为一，确认程序报告未完成。

不要把提前达到预算的部分结果写成完整成功。

### 29.7 练习七：超限下载保留旧文件

使用 `download_sync()`：

```python
from pathlib import Path
from httpx_demo import make_sync_client

path = Path("httpx_practice/limited.bin")
path.parent.mkdir(parents=True, exist_ok=True)
path.write_bytes(b"old")

with make_sync_client() as client:
    try:
        download_sync(client, "/download", path, max_bytes=100)
    except ValueError:
        pass
    else:
        raise AssertionError("应该拒绝超限下载")

assert path.read_bytes() == b"old"
assert not list(path.parent.glob(path.name + ".*.part"))
```

同时检查正式结果和临时结果，才能验证失败边界。

### 29.8 练习八：验证业务结构错误

让模拟服务返回合法 JSON，但价格是字符串。

```python
import asyncio
import httpx


async def main():
    def handler(request):
        return httpx.Response(200, json={"id": 1, "title": "demo", "price": "free"})

    async with httpx.AsyncClient(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(handler),
    ) as client:
        api = BookApi(client)
        try:
            await api.get_book(1)
        except BookApiError:
            print("结构校验按预期拒绝")
        else:
            raise AssertionError("不应接受字符串价格")


if __name__ == "__main__":
    asyncio.run(main())
```

先定义前面的 `BookApi` 和 `BookApiError`。

这个练习说明 HTTP 200 与 JSON 合法都不足以证明业务正确。

### 29.9 练习九：钩子要与客户端类型匹配

为同步 Client 写普通函数钩子，为 AsyncClient 写异步函数钩子。

只记录方法、路径和状态，不消费响应体。

然后用自定义异步字节流观察：钩子没有提前读取正文，逐行消费仍然按应用顺序发生。

### 29.10 练习十：准备请求不联网

```python
import httpx
from httpx_demo import demo_handler

with httpx.Client(
    base_url="https://books.example.invalid",
    transport=httpx.MockTransport(demo_handler),
) as client:
    request = client.build_request("POST", "/echo", json={"count": 3})

assert request.method == "POST"
assert request.url.path == "/echo"
assert request.headers["Content-Type"].startswith("application/json")
```

如果想验证真正发送了几次，可以在模拟回调里累计调用次数；这里仅构造对象，回调不会因此自动执行。

### 29.11 练习十一：总期限与模拟回调

```python
import asyncio
import httpx


async def main():
    async def handler(request):
        await asyncio.sleep(1)
        return httpx.Response(200)

    async with httpx.AsyncClient(transport=httpx.MockTransport(handler)) as client:
        try:
            async with asyncio.timeout(0.05):
                await client.get("https://demo.example.invalid/slow")
        except TimeoutError:
            print("外层协程期限按预期结束等待")


if __name__ == "__main__":
    asyncio.run(main())
```

请准确说明：这验证外层 asyncio 期限，不是证明网络 ReadTimeout 已生效。

### 29.12 练习十二：真实回环读取超时

启动第十三章的 `httpx_loopback_server.py`，再请求 `/slow`，读取超时设为 0.1 秒。

确认捕获的是 `httpx.ReadTimeout`。

然后改为普通 `/hello`，确认服务可达且正常请求成功。

先证明正常路径能运行，再制造特定失败，更容易排除“服务根本没启动”等不相关原因。

## 第三十章：按任务选择接口的速查

| 任务 | 常见选择 | 要同时确认的事 |
| --- | --- | --- |
| 单次同步请求 | `httpx.get/post/...` | 默认超时与重定向政策 |
| 重复同步交互 | `httpx.Client` | 生命周期与连接复用 |
| 异步交互 | `httpx.AsyncClient` | 在协程中等待并关闭 |
| 查询参数 | `params=`、`QueryParams` | 多值、None与布尔编码 |
| JSON 正文 | `json=` | 传Python对象，避免双重编码 |
| 表单字段 | `data=` | 服务器格式契约 |
| 原始正文 | `content=` | 字节或相应同步/异步流 |
| 文件表单 | `files=` | 文件生命周期和本地读取 |
| 状态检查 | `raise_for_status()` | 非2xx状态包括重定向 |
| 同步流 | `client.stream()` | 用with，读完或关闭 |
| 异步流 | `client.stream()` | 用async with与aiter方法 |
| 字节迭代 | `iter_bytes/aiter_bytes` | 内容解码层次 |
| 原始迭代 | `iter_raw/aiter_raw` | 不等于完整网络抓包 |
| 逐行文本 | `iter_lines/aiter_lines` | 仍要按真实记录协议解析 |
| 整体异步读取 | `await response.aread()` | 接受内存开销 |
| 手动关闭 | `close/aclose` | 异步关闭需要await |
| 阶段超时 | `httpx.Timeout` | connect/read/write/pool |
| 总异步期限 | `asyncio.timeout` | 取消与阻塞代码边界 |
| 连接资源 | `httpx.Limits` | 与任务数、速率不同 |
| 应用并发 | `asyncio.Semaphore` | 范围应覆盖真实资源占用 |
| 大型任务输入 | 有界队列和固定worker | 输出也可能占内存 |
| 离线响应测试 | `MockTransport` | 不验证真实网络层 |
| 配置可信TLS | `ssl.SSLContext` | 正确CA与身份材料 |
| 显式代理 | `proxy=` | 0.28接口，避免自动叠加TUN |

建议从同步最小请求开始，接着理解客户端状态、请求体和响应；再单独掌握协程等待、并发调度和资源关闭；最后组合分页、下载、限并发和失败恢复。

你不必为了使用一个 HTTP 库先掌握所有异步底层原理，但应能说明当前调用会不会等待、返回什么、资源由谁关闭，以及失败之后还剩什么状态。

官方参考：[HTTPX 快速入门](https://www.python-httpx.org/quickstart/)、[Client](https://www.python-httpx.org/advanced/clients/)、[异步支持](https://www.python-httpx.org/async/)、[Timeout](https://www.python-httpx.org/advanced/timeouts/)、[Limits](https://www.python-httpx.org/advanced/resource-limits/)、[Transport与MockTransport](https://www.python-httpx.org/advanced/transports/)、[与Requests的差异](https://www.python-httpx.org/compatibility/)。本篇示例模型和客户端是为教学编写，真实服务仍需按它自己的协议和版本要求使用。


## 附录 A：配套完整代码（按小块阅读；第一次阅读可跳过）

**第一块：** 导入库与准备样例数据；**第二块：** 怎样构造响应；**第三块：** 根据 URL 路径决定返回什么；**第四块：** 处理参数与错误状态；**第五块：** 启动服务/配置模拟客户端。先看每块的输入和输出，不必一口气记住所有代码。

将下方代码复制保存为 `httpx_demo.py`：

```python
import json
import httpx


BOOKS = [
    {"id": 1, "title": "Python 文件处理", "price": 39},
    {"id": 2, "title": "路径对象入门", "price": 29},
    {"id": 3, "title": "HTTP 客户端实践", "price": 49},
    {"id": 4, "title": "日志与排错", "price": 35},
    {"id": 5, "title": "数据校验", "price": 45},
]


def demo_handler(request):
    path = request.url.path

    if path == "/hello":
        return httpx.Response(200, json={"message": "你好，HTTPX"})

    if path == "/echo":
        if request.method == "GET":
            return httpx.Response(200, json={
                "method": request.method,
                "query": list(request.url.params.multi_items()),
                "x_demo": request.headers.get("X-Demo"),
                "cookie": request.headers.get("Cookie"),
            })

        content_type = request.headers.get("Content-Type", "")
        if content_type.startswith("application/json"):
            try:
                data = json.loads(request.content)
            except (ValueError, UnicodeDecodeError):
                return httpx.Response(400, json={"error": "JSON 不合法"})
        else:
            data = {"raw_size": len(request.content)}

        return httpx.Response(200, json={
            "method": request.method,
            "content_type": content_type,
            "data": data,
        })

    if path == "/books":
        try:
            page = int(request.url.params.get("page", "1"))
            page_size = int(request.url.params.get("page_size", "2"))
            if page < 1 or not 1 <= page_size <= 100:
                raise ValueError
        except ValueError:
            return httpx.Response(400, json={"error": "分页参数不合法"})

        start = (page - 1) * page_size
        return httpx.Response(200, json={
            "items": BOOKS[start:start + page_size],
            "page": page,
            "page_size": page_size,
            "total": len(BOOKS),
        })

    if path.startswith("/books/"):
        try:
            book_id = int(path.rsplit("/", 1)[1])
        except ValueError:
            return httpx.Response(400, json={"error": "编号不是整数"})

        for book in BOOKS:
            if book["id"] == book_id:
                return httpx.Response(200, json=book)
        return httpx.Response(404, json={"error": "没有这本书"})

    if path == "/redirect":
        return httpx.Response(302, headers={"Location": "/hello"})

    if path == "/empty":
        return httpx.Response(204)

    if path == "/text":
        return httpx.Response(200, text="第一行\n第二行\n")

    if path == "/download":
        return httpx.Response(
            200,
            content=bytes(range(256)) * 1024,
            headers={"Content-Type": "application/octet-stream"},
        )

    if path == "/cookie":
        return httpx.Response(
            200,
            json={"message": "已设置 Cookie"},
            headers={"Set-Cookie": "demo=learn; Path=/; HttpOnly"},
        )

    return httpx.Response(404, json={"error": "接口不存在"})


def make_sync_client():
    return httpx.Client(
        base_url="https://books.example.invalid",
        transport=httpx.MockTransport(demo_handler),
        timeout=httpx.Timeout(5.0, connect=2.0),
    )
```
