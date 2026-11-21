# Python Requests 库超详细教程：从 HTTP 请求与响应，到参数、超时、会话和文件传输

## 阅读前先补上这一课：Requests 到底用来做什么？（真正从零开始）

> **本节是阅读路线，不要求你先懂网络、HTTP、API、服务器。** 如果你此前只学过 `print()`、`open()`、`json`、`os` 等，也可以从这里开始。原文后面仍保留大量进阶知识和完整实验；先把这一节真正弄懂，再往后学，不要一上来就读 100 多行的服务器程序。

### 0.1 先不写代码：你为什么会需要 Requests？

想象你正在写一个“天气信息整理”程序。你已经会从电脑硬盘上读取 `weather.json`：

```python
import json

with open("weather.json", "r", encoding="utf-8") as f:
    weather = json.load(f)

print(weather)
```

这里的 `open()` 读取的是**你电脑上已经存在的文件**。如果数据根本不在你的硬盘，而在一台提供天气信息的远程服务器上，你怎么办？

一种低效的方法是：自己打开浏览器 → 找到页面 → 复制内容 → 粘贴到文件 → Python 再读取。可是你希望每天自动运行，不想每次都手动操作。

另一种方法是：让 Python 程序**自己向远程服务器请求数据**。这就是 Requests 的主要工作：**帮 Python 主动发送 HTTP/HTTPS 请求，并把服务器的回应交给你**。它相当于“程序中的网络客户端”，不是操作系统文件管理器，也不是用来创建网站的服务器。

你可能用它做这些事：

- 从公开的天气、汇率、交通、图书等 API 获取数据（具体接口通常有自己的使用规则）。
- 下载网页、图片、压缩包等资料。
- 向你有权限的服务提交表单、JSON 数据、文件。
- 访问自己写的后端接口，测试接口是否按预期工作。
- 以脚本方式替代重复的手动网络操作。

反过来，**整理本地文件**用 `pathlib`、`os`、`open`；**解析已经拿到的 JSON**用 `json`；**发送 HTTP 请求、拿到远程响应**才是 Requests 的主业。它不会自动绕过登录、验证码、付费限制，也不会像浏览器那样自动执行网页上的 JavaScript。

### 0.2 这个名字从哪里来？什么叫“请求”？

`request` 是英语的“请求”，`response` 是“响应 / 回答”。`requests` 库把“发请求、接收响应”这件事封装成方便使用的 Python 函数。

试着把上网看作一场对话：

```text
你（客户端）：请把 /news 这个地址的内容给我。
服务器：好的，这是内容；状态码 200 表示这次处理成功。

你（客户端）：请把编号 999999 的文章给我。
服务器：没有找到；状态码 404。

你（客户端）：请保存我提交的资料。
服务器：我检查后接受 / 拒绝了这次提交。
```

注意：客户端“发送成功”与服务端“业务处理成功”不是同一件事。你成功联系到服务器，也可能得到 `404`、`401` 或 `500`。Requests 负责传输和结果封装，**不会替你决定业务规则**。

### 0.3 先理解四个词：服务器、URL、HTTP、API

**服务器（server）**：在网络另一端接收请求并按照规则回答的程序/计算机。**客户端（client）**：提出请求的一方。你的 Python 脚本就是客户端。

**URL**（Uniform Resource Locator，统一资源定位符）：你要访问的资源地址。以 `https://example.com/books?page=2` 为例：

```text
https://example.com/books?page=2
└─协议─┘└───域名───┘└路径┘└查询参数┘
```

- `https://`：使用 HTTPS，HTTP 的传输受到 TLS 保护；不是“保证数据真实可信”的万能证明。
- `example.com`：目标主机的域名。程序通常需要通过 DNS 等机制找到可连接的网络地址。
- `/books`：路径，由目标服务自己规定。
- `?page=2`：查询字符串，常用于搜索、筛选、分页；`page` 是参数名，`2` 是参数值。

**HTTP**（Hypertext Transfer Protocol，超文本传输协议）：客户端和服务端交换请求/响应时遵循的一套规则。你暂时可以把它理解成“约定如何问和如何答”，不需要现在学习 TCP 报文的所有细节。

**API**（Application Programming Interface，应用程序编程接口）：广义上是程序之间约定的调用接口；在这里常指通过 HTTP 提供数据或操作的网络接口。不是所有 URL 都是“返回 JSON 的 API”；一些 URL 返回的是 HTML 网页、图片或文件。

### 0.4 先看清一次请求究竟带什么

```text
客户端 → 服务器：请求
    方法 Method       GET / POST / PUT / DELETE ...
    URL              去哪里
    请求头 Headers    提示内容格式、凭据、客户端信息等
    请求体 Body       可选：要提交的数据

服务器 → 客户端：响应
    状态码 Status     200、404、500 等
    响应头 Headers    服务端声明的类型、缓存规则等
    响应体 Body       网页、JSON、图片字节等
```

最常见的两个方法：`GET` 用来读取资源；`POST` 经常用来提交数据、触发服务端操作。**方法本身不保证幂等，也不能仅凭名字认定某个 API 一定安全**，具体以服务端接口文档为准。

“请求头”和“响应头”是两套不同的字段。比如你发送 `Accept: application/json`，只是告诉服务器你希望接收 JSON；**并不意味着服务器必然返回合法 JSON**。最终要看状态码、响应头和响应体。

### 0.5 第一个请求：只用四行，知道每一行为什么存在

安装（在运行脚本的同一个 Python 环境中）：

```bash
python -m pip install requests
```

假设电脑能联网，我们先向公开的演示域名读取一个普通 HTML 页面，不涉及登录、不要求你搭建服务器。创建 `first_request.py`：

```python
import requests  # ① 导入第三方 Requests 包

url = "https://example.com/"            # ② 要访问的网页地址
response = requests.get(url, timeout=10) # ③ 发 GET 请求，等待服务器响应
print(response.status_code)              # ④ 输出响应状态码（正常情况下常见 200）
```

逐句解释：

**① `import requests`**：让 Python 能使用这个包提供的函数。它与 `import os` 不同，Requests 通常要先安装，因为它是第三方库。

**② `url = ...`**：只是把地址保存为字符串；**这一行不会联网**。

**③ `requests.get(url, timeout=10)`**：这一步才发起网络操作。`get` 对应 HTTP 的 GET 方法；`timeout=10` 用来避免某次网络等待无期限拖住程序。它并不是“整个程序或整个大文件下载绝对只能运行十秒”的总时限。返回的 `response` 是 `requests.Response` 对象。

**④ `response.status_code`**：读取对象的属性，它是整数，不是函数，所以不加括号。`200` 通常表示成功；`404` 通常表示服务端没有找到目标资源；`500` 表示服务端内部错误。网络不通时，第三行可能直接抛异常，根本不会产生 `response`。

如果你现在无法联网，可以**先理解这四行，不必因此卡住**。后文的本地练习服务可以在不依赖公共网站的情况下演示同样的过程。注意：本地服务仍然需要监听端口，不等于程序没有进行任何网络通信。

### 0.6 我看到的是 Response 对象；它与内容本身有什么区别？

假设你从网络拿到一个纸箱。纸箱外面贴着物流状态、物品类型，打开后才是正文。`Response` 就像把状态、头信息和正文都装在一起的容器。

```python
response = requests.get("https://example.com/", timeout=10)

print(response.status_code)      # 整数：例如 200
print(response.headers.get("Content-Type"))  # 服务端声称的正文类型
print(response.text[:100])       # 正文按文本解码后，只看前 100 个字符
```

理解四个入口：

| 表达式 | 是什么 | 常用于 |
|---|---|---|
| `response.status_code` | HTTP 状态码，整数 | 判断服务器怎样回答 |
| `response.headers` | 响应头映射 | 看 `Content-Type`、缓存等 |
| `response.text` | 按字符编码得到的字符串 | HTML、纯文本 |
| `response.content` | 正文字节 `bytes` | 图片、压缩包、任意二进制 |
| `response.json()` | **尝试**把正文作为 JSON 解析 | 返回合法 JSON 的 API |

最重要的区别：`.text` 是**字符串属性**，`.content` 是**字节属性**，`.json()` 是**方法**，因为解析 JSON 是一个动作，要写括号。`.json()` 不会自动证明请求成功，也不保证解析一定成功；如果服务器返回的是 HTML 错误页，解析通常会失败。

你已经学过 `json.loads(...)` 的话，可以把 `response.json()` 近似理解成“从响应正文解析 JSON 的便捷方法”，但这只是帮助理解的类比，实际还涉及编码与解析异常处理。

### 0.7 常见的 GET、POST、params、json 到底是什么关系？

先理解需求，再看代码。

**任务 A：读取第二页的图书。** 服务器约定接口为 `/books`，用 `page` 指定页码。无需你自己手工拼接问号和转义字符：

```python
params = {"page": 2, "page_size": 10}
response = requests.get(
    "https://api.example.invalid/books",
    params=params,             # 自动组织 URL 查询参数
    timeout=10,
)
```

这里的 `api.example.invalid` 是**示意地址，不会真实提供图书数据；以上代码只用于解释语法，不能直接运行得到成功响应**。要练习成功响应，请改用有文档、你获准访问的真实 API 或后文的本地服务。

`params` 负责 **URL 的查询参数**，与 Python 函数的形参虽然名字相似，但含义不同。最终请求 URL 形如 `.../books?page=2&page_size=10`。不要把所有用户输入都用字符串 `+` 拼到 URL 上，库会帮你处理多数必要的编码细节。

**任务 B：向服务端提交一个 Python 字典作为 JSON。**

```python
payload = {"name": "小林", "score": 95}
response = requests.post(
    "https://api.example.invalid/records",
    json=payload,              # 让 Requests 将数据序列化成 JSON 请求体
    timeout=10,
)
```

这个地址同样只是说明语法。`json=payload` 中的 `json` 是 **`requests.post` 的关键字参数**，不是 `import json` 之后的模块对象。它会生成 JSON 请求体并通常设置合适的 `Content-Type`。对比：

- `params=...`：常出现在 URL 中，用于筛选、分页、搜索。
- `json=...`：用 JSON 格式发送请求正文。
- `data=...`：常用于表单字段或你自行准备的数据。
- `files=...`：常用于构造文件上传请求。

并非“GET 不能有请求体”这种绝对化判断才是本质；具体 API 支持哪些方法、参数位置和格式，必须看对方文档。

### 0.8 不要一次写一大段：把一个有用的小程序分成三步

我们的目标：访问一个返回 JSON 的 API；如果网络错误、HTTP 失败或正文不是 JSON，给出明确处理。这里的完整调用需要你换成有权限且返回 JSON 的真实 URL；本地练习时可使用后文服务的 `/hello` 端点。

**第一步：只负责发请求，并给等待设置边界。**

```python
import requests

url = "http://127.0.0.1:8765/hello"  # 本地服务启动后才可连接
try:
    response = requests.get(url, timeout=(3, 10))
except requests.exceptions.RequestException as exc:
    print("网络连接或传输失败：", exc)
else:
    print("收到 HTTP 响应：", response.status_code)
```

这里的 `(3, 10)` 分别给连接等待和读取等待设定超时；并非整次请求的总秒数。`RequestException` 是 Requests 这一类异常的常见父类。`else` 表示 `try` 里面没有抛出异常才执行。**即使打印出状态码 404，说明的是收到服务器响应，不等于业务成功**。

**第二步：在收到响应后，检查服务端状态。** 在前一段 `else` 分支中，可以继续写：

```python
try:
    response.raise_for_status()  # 4xx / 5xx 通常会抛 HTTPError
except requests.exceptions.HTTPError as exc:
    print("服务器返回错误状态：", exc)
else:
    print("HTTP 状态检查通过")
```

`raise_for_status()` 不是 Python 自动执行的；**需要你主动调用**。它主要针对 HTTP 错误状态，不会因为返回的数据里有 `{"ok": false}` 就自动发现业务失败。

**第三步：确认这个接口约定返回 JSON，才尝试解析。**

```python
try:
    result = response.json()   # 解析响应正文；可能失败
except requests.exceptions.JSONDecodeError:
    print("响应正文不是合法 JSON")
else:
    print("解析后 Python 对象：", result)
```

三步写清楚，分别负责：**网络传输 → HTTP 状态 → 数据格式**。在正式程序中，你通常要把第二、三步放在第一步成功后的分支内，避免第一步抛异常时使用尚不存在的 `response`。不要把上面三个独立代码块不加修改地上下拼接成一个文件；它们是在解释同一条处理流程中的不同阶段。你可以在后文完整例子中看到如何衔接。

### 0.9 为什么还会出现 Session、Cookie 和连接池？

最早的 `requests.get(...)` 就能发请求。等你要连续访问同一个站点很多次时，会遇到两个新问题：

第一，服务器可能通过 Cookie 记住前一次交互的状态。第二，重复创建连接常常带来额外开销。

于是就有 `requests.Session()`：

```python
import requests

with requests.Session() as session:  # ① 创建一个会话，结束时释放资源
    session.headers.update({"User-Agent": "study-demo/1.0"})  # ② 会话默认头
    first = session.get("https://example.com/", timeout=10)   # ③ 发第一次请求
    second = session.get("https://example.com/", timeout=10)  # ④ 再发一次
    print(first.status_code, second.status_code)
```

- **①** `Session` 可保存 Cookie、复用底层连接池等；`with` 退出时调用清理流程。
- **②** `headers.update(...)` 修改这个会话后续请求的默认请求头；不要把密码、令牌直接写入准备公开的代码。
- **③④** 两次请求使用同一会话。连接是否真正复用还取决于服务端、响应读取方式等条件，**不能把“同一 Session”理解成“保证同一 TCP 连接一直不变”**。

一个脚本只发一次请求时，不必强行上 `Session`；大量有共同配置的请求时再考虑它。

### 0.10 哪些知识暂时不用学？尤其不要被 150 行服务端示例劝退

后文长服务端代码用标准库 `http.server` **模拟“别人提供给你的接口”**，其价值是让你能固定重现状态码、请求参数、文件传输等现象。

你要学的是**Requests 客户端**，不必先把服务端类、`do_GET`、`do_POST`、`send_header` 全部背会。建议按这个顺序阅读：

1. 先理解本节 `0.1—0.9` 的请求/响应关系，自己解释四行 `requests.get`。
2. 再读原文“第一章”，巩固 HTTP 概念；“第二章”先只看安装和本地服务的用途，不必深挖服务端实现。
3. 优先进入后文客户端基础例子，练习 `get`、`params`、`Response`、`json`、`raise_for_status`。
4. 需要做完整离线实验时，再按第二章说明启动服务；**服务端全代码放在文末附录**，并按功能拆解。
5. 最后学认证、Cookie、Session、下载、上传、重试、超时与安全。不要把一篇教程的所有高级功能当成第一次学习的先决条件。

### 0.11 最容易混淆的地方，先一次说清楚

| 你可能会问 | 关键答案 |
|---|---|
| Requests 是浏览器吗？ | 不是。它是 Python 的 HTTP 客户端，通常不会替你运行页面 JavaScript。 |
| Requests 是拿来做爬虫的吗？ | 它能请求网页，但爬取还涉及 HTML 解析、站点许可、限流和合规；并不自带完整爬虫能力。 |
| 有了 `requests.get` 还要 `json` 吗？ | 请求库负责拿到响应；JSON 是数据交换格式。`response.json()` 可以直接解析合法 JSON，但并非所有响应都是 JSON。 |
| `response.status_code == 200` 是不是所有事情都没问题？ | 只能说明一个 HTTP 状态；数据结构、业务含义还要自己检查。 |
| `timeout=10` 是整个下载最多十秒吗？ | 不是；Requests 的超时参数主要约束连接/读等待。 |
| `params` 和 `json` 有什么区别？ | 前者通常写进 URL 查询字符串，后者写入 JSON 请求体。 |
| 为什么访问地址有时显示 404？ | 可能成功连接到服务器，但服务器没有你请求的资源。 |
| 什么情况下该用 Requests，不用 HTTPX？ | 同步脚本、已有 Requests 项目往往继续用它就好；若确实要原生异步等，再考虑 HTTPX。 |

**一个学习检查问题**：如果你写了 `response = requests.get(url)`，请分别说清楚 `url` 是什么、网络请求发生在哪一行、`response` 是什么类型的对象、为什么 `.text` 不等于 `.json()`、以及 404 与连接失败有什么区别。能用自己的话回答，就具备进入下文详细 API 的基础了。

---

本篇终端命令里的 `python` 表示你选用的 Python 解释器。WSL 或 Linux 如果只有 `python3` 命令，请把对应命令开头改成 `python3`；激活项目虚拟环境后，则使用该环境提供的解释器。

这一篇讲 Python 中常见的同步 HTTP 客户端 Requests。它用于让程序主动访问 HTTP 或 HTTPS 服务，例如读取 API、下载文件、提交表单和上传资料。

本文围绕一套本地练习服务展开。你可以在自己的电脑上观察请求究竟发了什么、服务器返回什么，避免只复制一段依赖外部网站状态的代码。

常规客户端示例使用 Requests 2.x 的稳定接口，示例语法面向 Python 3.11 及以上。涉及第三方依赖的接口应以项目实际安装版本为准。网络示例运行时会真实访问其 URL；文中的默认练习地址是本机回环地址。

## 第一章：Requests 解决什么问题？

### 1.1 浏览器访问网站，是一种客户端行为

你在浏览器输入地址，浏览器向服务器发送请求，服务器返回网页、图片或结构化数据。

Python 程序也可以成为客户端。区别是它通常不需要图形界面，而是关心状态码、头字段和响应体。

```text
你的 Python 程序
    │ 调用 Requests
    ▼
HTTP 请求：方法、URL、头、请求体
    │ 网络或本机连接
    ▼
服务器处理
    │
    ▼
HTTP 响应：状态码、头、响应体
    │ Requests 封装
    ▼
Response 对象
```

Requests 为常见 HTTP 工作提供较高层的 Python 接口。你不必自己拼接每个请求报文、管理所有底层套接字细节，就能发送参数、处理 Cookie 和读取响应。[官方入口：Requests Quickstart](https://requests.readthedocs.io/en/latest/user/quickstart/)

### 1.2 名字是什么含义？

request 是“请求”。程序向服务提出一个动作或数据需求，得到相应响应。

安装包与导入名称通常都是小写 `requests`：

```python
import requests
```

文档和项目名经常写作 Requests。它不是 Python 标准库，不会因为你安装了 Python 就必然已经存在。

### 1.3 标准库与第三方库的区别

`json`、`pathlib`、`urllib.request` 属于 Python 标准库范围。Requests 是第三方包，通常需要装进当前项目环境。

使用它的原因是接口方便、常见功能组织清楚，并不是说标准库完全无法发送 HTTP 请求。

选择工具还应考虑项目已经使用的库、同步或异步需求和依赖管理。不要只因为某个教程使用了 Requests，就在已有 HTTP 客户端的项目中再无必要地引入另一套。

## 第二章：安装、解释器与可重复练习环境

### 2.1 使用当前解释器的 pip

```bash
python -m pip install requests
```

`python -m pip` 表达“用当前这个 Python 运行 pip”，能减少多个解释器共存时装错环境的问题。

验证：

```python
import sys
import requests

print(sys.executable)
print(requests.__version__)
print(requests.__file__)
```

这分别显示当前解释器、导入版本和模块来源。如果目录里有你自己命名的 `requests.py`，可能遮蔽真正的包，应更改本地文件名。

### 2.2 推荐在项目虚拟环境中安装

创建环境：

```bash
python -m venv .venv
```

WSL 或 Linux Bash 激活：

```bash
source .venv/bin/activate
python -m pip install requests
```

Windows PowerShell 常见激活：

```powershell
.\.venv\Scripts\Activate.ps1
python -m pip install requests
```

两套路径对应不同系统中的环境，不要把 Windows 虚拟环境当成可以直接给 WSL Linux Python 使用的同一份安装。

如果你已经使用项目规定的 uv、Poetry 或其他管理工具，按项目约定添加依赖即可，不必为了本篇重新建立第二套并行环境。

### 2.3 本地练习服务的作用

后面的例子使用本机地址 `http://127.0.0.1:8765`。

本章接下来会一步一步解释练习服务，再给出可直接运行的完整程序。把它保存为 `requests_demo_server.py`，启动后就能运行第三章及后面的客户端。

如果现在只想先理解客户端，也可以继续往下读，再回头准备这个服务。

服务只绑定本机回环地址，用于观察请求和响应，不实现生产级访问控制，也不应直接对外部署。

服务程序的代码会涉及类和标准库 HTTP 接口。你可以先理解本章给出的请求流向，运行程序，再在后面的客户端例子中逐个观察它的分支。

### 2.4 启动、验证和停止

终端运行：

```bash
python requests_demo_server.py
```

看到监听地址后，在另一个终端执行下一章客户端。结束练习时回到服务器终端按 `Ctrl+C`。

如果端口已被占用，应选择一个未占用端口，并同时修改服务端和客户端地址，不要为了练习随意终止不认识的其他进程。

这套服务使用简化 HTTP 处理，主要帮助观察请求与响应，不用于证明生产性能、连接保持或协议完整兼容性。

### 2.5 我只是学习发请求，为什么还要准备一个服务？

假设你要练习“发送姓名和分数，再读取返回的 JSON”。

如果随便选择一个公共网站，你并不知道它接受哪些字段，也不知道它今天是否需要登录。

即使请求失败，也很难知道失败来自自己的代码，还是来自对方的规则。

这里自己准备一个服务，就是让这段交互的另一端变得确定。

例如，我们约定请求 `/hello` 时，总返回：

```json
{"message": "你好，Requests"}
```

请求 `/echo` 时，则把收到的参数返回给客户端。

echo 是“回声”的意思。

你发送什么，就能在响应中观察相应内容，特别适合验证编码、查询参数和请求体。

这一步不会要求你学习一个 Web 框架。

它只是为本篇客户端提供一个可以看清输入与输出的练习对象。

### 2.6 一个请求进入服务后，会经过哪些步骤？

先把流程画出来：

```text
客户端请求 /echo?name=Alice
    ↓
本机服务接收连接，读到请求路径和请求头
    ↓
do_GET() 判断路径是 /echo
    ↓
parse_qs() 解析 name=Alice
    ↓
send_json() 把 Python 字典变成 UTF-8 字节
    ↓
客户端收到状态码、响应头和 JSON 正文
```

这里的 `do_GET()` 名字由 `BaseHTTPRequestHandler` 约定。

处理器收到 GET 请求时会调用它，不需要我们在主程序里手动逐次调用。

`do_POST()` 对应 POST，`do_HEAD()` 对应 HEAD。

路径末尾有没有 `.json` 并不决定返回格式；本例由处理函数及其设置的响应头决定。

### 入门导读：为什么长代码不应挡在你前面？

先把 HTTP 请求和响应、get、Response、params、json 学会。完整练习服务器只用于本地复现实验，无需理解它才能使用 Requests。它的结构是：数据表 → 响应辅助方法 → GET 路由 → POST 路由 → 本机启动入口。

使用方法：需要运行练习时，再把文末附录代码保存为 `requests_demo_server.py`，先观察各个小块，然后运行客户端。后文原有逐项讲解仍然保留。

### 2.7 完整练习程序

将下面代码保存为 `requests_demo_server.py`。

**这一段完整配套代码已经移至文末附录 A，初学时请先跳过。**


### 2.8 代码里的三种数据，为什么要来回转换？

先看 `send_json()`。

它接收的是 Python 字典、列表等对象。

`json.dumps(...)` 生成 JSON 字符串。

`.encode("utf-8")` 再把字符串编码成可以发送的字节。

所以这里的顺序是：

```text
Python 对象 → JSON 文本 → UTF-8 字节
```

客户端拿到字节以后，`response.json()` 按 JSON 规则解析，再得到 Python 对象。

请求与响应并不会让两端共享同一个字典；两端是通过约定的文本格式交换内容。

`Content-Length` 使用 `len(body)`。

此时 `body` 已经是字节，所以得到的是字节数。

如果先对中文字符串求长度，再把这个字符数量写成响应的字节长度，就可能描述错误。

### 2.9 同一个名字出现两次，服务端为什么返回列表？

假设客户端发送：

```text
/echo?tag=python&tag=files
```

`parse_qs()` 会把同名参数收集起来，形成类似：

```python
{"tag": ["python", "files"]}
```

即使某个参数只有一个值，也采用列表表达。

因此 `/items` 的分页分支使用：

```python
page = int(query.get("page", ["1"])[0])
```

先取这个参数对应的列表，再取第一项，最后转换为整数。

这里“重复分页参数只使用第一项”是练习服务自己选择的规则。

真实 API 可能选择最后一项、拒绝重复，或者采用其他解释，不能从这个例子推断所有网站的行为。

### 2.10 如果我提交的不是 JSON 呢？

`do_POST()` 先读取请求头里的内容类型，再选择解释方式。

`application/json` 进入 JSON 解析分支。

`application/x-www-form-urlencoded` 进入普通表单分支。

其他格式在本例中只记录字节大小。

`/upload` 也只返回收到的正文长度和内容类型，不解析并保存每一个 multipart 文件。

所以你可以用它观察“文件表单请求确实发送了正文”，却不能据此声称已经验证了真实服务的文件命名、内容检查和持久保存。

后面讲 `json=`、`data=`、`files=` 时，会把这些返回值与客户端写法对应起来。

### 2.11 我怎么确认服务已经能用了？

先在一个终端启动服务：

```bash
python requests_demo_server.py
```

终端通常显示：

```text
练习服务：http://127.0.0.1:8765
```

再在另一个终端执行第三章的 GET 请求。

你应该看到状态码二百，以及包含问候语的字典。

如果服务端终端立即报“地址已被使用”，说明启动步骤就失败了，客户端连接问题还没有开始排查。

如果服务正常运行，而客户端返回四百零四，先检查路径是否写对。

这样把启动、连接和请求处理分开观察，比一次修改多个设置更容易知道原因。

### 2.12 客户端超时了，服务端会自动停止吗？

不一定。

`/delay` 分支会等待两秒。

客户端可以在更短时间后结束等待，但服务端的 `sleep()` 不会因此自动取消。

服务随后写响应时，可能发现客户端已经断开，于是触发 `BrokenPipeError` 或 `ConnectionResetError`。

本例只在发送练习响应的这一小段忽略这两类断连错误。

它没有把所有服务端异常都吞掉。

这个现象也提醒我们：请求方不再等待，不等于处理方已经撤销操作。

真实的付费、创建记录或提交任务接口，要按它自己的幂等与取消约定处理。

## 第三章：第一个 GET 请求，逐字拆开

### 3.1 最小完整代码

```python
import requests

response = requests.get(
    "http://127.0.0.1:8765/hello",
    timeout=(3, 5),
)
response.raise_for_status()

print(response.status_code)
print(response.headers.get("Content-Type"))
print(response.json())
```

如果第二章的练习服务已经在同一环境启动，通常输出：

```text
200
application/json; charset=utf-8
{'message': '你好，Requests'}
```

第一行是状态码，第二行是服务声明的内容类型，第三行是把 JSON 正文解析后得到的 Python 字典。

如果换成别的真实服务，响应头和业务内容当然可能不同；这里的固定输出来自我们自己刚刚定义的练习接口。

### 3.2 `requests.get()` 做什么？

它构造并发送 GET 请求，等待响应过程到达相应阶段，返回 `Response` 对象。

GET 通常用于取得资源。但方法名称本身不能保证一个错误设计的服务器绝不修改状态，调用接口仍应遵循服务提供方契约。

### 3.3 `timeout=(3, 5)` 是什么意思？

元组两项分别是连接超时和读取超时，单位秒。

它不表示整个函数最多运行八秒，也不表示整个下载一定在五秒内完成。读取超时主要限制等待套接字数据的时间间隔，后面会专章说明。

### 3.4 为什么单独调用 `raise_for_status()`？

服务器返回 404 或 500 时，Requests 通常仍能成功收到一个 HTTP 响应对象。网络请求“收到响应”和业务“请求成功”不是同一个条件。

`raise_for_status()` 对相应 4xx、5xx 状态抛出 HTTP 错误，让你可以在失败状态时停止正常解析路径。

它不会检查 JSON 中的业务 `code` 字段，也不会把所有 3xx 都当错误。

### 3.5 返回值与内容访问

`status_code` 是整数状态码；`headers` 提供响应头映射；`json()` 尝试把响应体解析为 Python 对象。

如果响应体不是合法 JSON，最后一步仍然会失败。`Content-Type` 声明和解析结果都应结合接口约定理解，不能只看某个属性就断定业务数据正确。

## 第四章：URL、路径和查询参数

### 4.1 URL 的几个部分

```text
http://127.0.0.1:8765/items?page=2&page_size=2
│      │              │     │
│      │              │     └── 查询字符串
│      │              └──────── 资源路径
│      └─────────────────────── 主机和端口
└────────────────────────────── 协议方案
```

URL 不是本地磁盘路径。不要用 `Path(base_url) / "items"` 构造网络地址，路径库按文件系统规则处理，可能破坏 URL 的双斜杠和编码语义。

### 4.2 使用 `params`，避免手工拼查询字符串

```python
import requests

response = requests.get(
    "http://127.0.0.1:8765/echo",
    params={"keyword": "Python 文件", "page": 2},
    timeout=(3, 5),
)
response.raise_for_status()

print(response.url)
print(response.json()["query"])
```

Requests 会把参数按 URL 查询规则编码，空格、中文、`&` 等字符不会简单原样拼入导致结构混乱。

### 4.3 重复参数

```python
params = [("tag", "python"), ("tag", "backend")]
```

传列表键值对可以明确表达重复键。字典值使用列表也可以表达常见多值参数。

本地服务的 `parse_qs()` 会把值解析成列表，所以你可能看到：

```json
{"tag": ["python", "backend"]}
```

这种结果来自服务端解析规则，不是所有 API 都采用相同列表形式。

### 4.4 `None`、空字符串和布尔值

查询参数值为 `None` 时，Requests 通常会省略相应项；空字符串则可能发送一个空值参数。

布尔值的字符串表现不一定符合服务端要求。如果接口要求 `true`、`false`，应主动构造对应字符串，不能假设它理解 Python 风格 `True`、`False`。

### 4.5 路径参数与查询参数不同

`/users/123` 中的 `123` 属于资源路径；`?page=2` 属于查询参数。应该放哪里，由服务器路由和接口设计决定。

Requests 不会因为你传 `params={"id":123}` 就自动把它变成 `/users/123`。客户端负责按照既定接口契约构造请求，不会猜测服务端路由。

## 第五章：请求方法与语义，为什么不能只学 `get()`？

### 5.1 常用方法对照

| 方法 | 常见意图 | Requests 入口 | 实际行为由谁决定 |
| --- | --- | --- | --- |
| GET | 取得资源 | `requests.get()` | 服务端路由与实现 |
| POST | 提交处理、新建等 | `requests.post()` | 服务端契约 |
| PUT | 替换目标资源状态 | `requests.put()` | 服务端契约 |
| PATCH | 部分修改 | `requests.patch()` | 补丁格式与契约 |
| DELETE | 删除目标资源关联 | `requests.delete()` | 服务端契约 |
| HEAD | 获取类似 GET 的头信息而不取响应体 | `requests.head()` | 服务端支持程度 |
| OPTIONS | 查询支持方式等 | `requests.options()` | 服务端实现 |

这些是常见 HTTP 语义，不是让客户端根据自己想法任意替换方法。同一个 URL 是否支持 POST，要看服务器。

本地练习服务主要实现 GET、HEAD 和 POST，没有实现的 PUT、PATCH、DELETE 可能返回未支持状态。这是练习服务的边界，不是 Requests 缺少这些功能。

### 5.2 通用入口 `request()`

```python
import requests

response = requests.request(
    method="GET",
    url="http://127.0.0.1:8765/hello",
    timeout=(3, 5),
)
response.raise_for_status()
```

适合方法由受控程序配置决定的封装。普通固定 GET 调用使用 `get()` 更直观。

不要把来自不可信输入的方法和 URL 原样交给服务器内部客户端执行。是否允许访问某个目标属于调用方的授权规则，HTTP 库不会代替你设计。

### 5.3 安全方法与幂等方法

HTTP 中“安全”通常表示预期不请求修改服务器状态，不是说绝对没有日志、计费或其他副作用。

“幂等”表示相同操作重复执行，在目标资源状态上的预期效果与执行一次一致。它不表示每次响应完全相同，也不表示网络失败时客户端能确定服务端是否执行过。

这些概念直接影响重试。读取列表的 GET 通常容易重试；创建订单的 POST 若没有服务端幂等键，盲目重试可能创建两份订单。

## 第六章：三种常见请求体——`json`、`data`、`files`

### 6.1 发送 JSON

```python
import requests

payload = {"name": "小林", "active": True, "tags": ["python", "backend"]}

response = requests.post(
    "http://127.0.0.1:8765/echo",
    json=payload,
    timeout=(3, 5),
)
response.raise_for_status()
print(response.json())
```

`json=payload` 表示把 Python 对象按 JSON 请求体编码。常规情况下库还会设置相应内容类型。

不要先 `json.dumps(payload)` 再把字符串传给 `json=`，除非接口确实要求一个 JSON 字符串值。否则会出现双重编码，服务器读到字符串而不是对象。

### 6.2 发送表单字段

```python
response = requests.post(
    "http://127.0.0.1:8765/echo",
    data={"name": "小林", "course": "Python"},
    timeout=(3, 5),
)
response.raise_for_status()
print(response.json())
```

字典形式的 `data` 通常编码为表单格式，内容类型类似 `application/x-www-form-urlencoded`。

服务器必须按这种格式解析。你不能只把参数名从 `json` 改成 `data`，就假设服务端收到的结构与原来完全一样。

### 6.3 `data` 也可以是原始字节或流

```python
response = requests.post(
    "http://127.0.0.1:8765/echo",
    data=b"raw payload",
    headers={"Content-Type": "application/octet-stream"},
    timeout=(3, 5),
)
```

这时不是字典表单编码，而是按给定原始内容发送。

同一个参数接受多种类型，因此你必须同时看参数名称和传入对象类型。`data={...}` 与 `data=b"..."` 的作用并不相同。

### 6.4 上传文件使用 `files`

```python
from pathlib import Path
import requests

path = Path("requests_practice") / "upload.txt"
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text("一份练习上传内容\n", encoding="utf-8")

with path.open("rb") as source:
    response = requests.post(
        "http://127.0.0.1:8765/upload",
        files={"document": (path.name, source, "text/plain")},
        data={"description": "练习文件"},
        timeout=(3, 5),
    )

response.raise_for_status()
print(response.json())
```

`document` 是表单字段名，`path.name` 是上传文件名，`source` 是文件流，`text/plain` 是该部分内容的媒体类型。

`files` 与表单字段可以组合，库会生成 multipart 请求体和对应边界。不要手写一个缺少 boundary 的 `Content-Type: multipart/form-data` 覆盖库的正确设置。

### 6.5 为什么文件用 `rb` 打开？

上传传递的是文件字节。二进制模式能避免文本换行转换和字符编码带来的差异，也有利于准确处理长度。

如果你读取文本并改成另一编码，那就已经改变上传内容，应在业务上明确这是转换行为。

### 6.6 三种参数不要随意混合

`json`、表单 `data` 和 `files` 具有不同编码意图。Requests 对同时提供时有具体优先和忽略规则，不能把它们当成“自动合并成一个万能请求”。

最清楚的原则是：JSON 接口传 `json`；普通表单传 `data`；带文件的 multipart 传 `files` 加需要的表单字段。接口约定其他格式时，明确传原始内容和对应头。

## 第七章：响应对象中的常用属性与方法

### 7.1 核心对照

| 成员 | 含义 | 常见误区 |
| --- | --- | --- |
| `status_code` | 整数状态码 | 不等于业务码字段 |
| `headers` | 响应头映射 | 字段值仍需按协议解释 |
| `url` | 最终响应 URL | 可能经过重定向 |
| `history` | 之前的重定向响应列表 | 不包含任意请求历史 |
| `content` | 响应内容字节 | 通常已经过内容解压处理 |
| `text` | 根据编码解释后的文本 | 编码选择可能需要检查 |
| `json()` | 尝试解析 JSON | HTTP 错误也可能有合法 JSON |
| `encoding` | 文本解码使用的编码信息 | 不一定是你期待的实际编码 |
| `cookies` | 响应关联 Cookie 容器 | 不同于会话全部 Cookie 状态 |
| `elapsed` | 请求阶段耗时信息 | 不等于读取整个下载文件的完整总耗时 |
| `request` | 实际准备好的请求信息 | 调试时可能含敏感内容 |
| `close()` | 释放响应相关资源 | 流式请求尤其需要管理 |

### 7.2 `.content` 是 bytes

```python
response = requests.get("http://127.0.0.1:8765/download", timeout=(3, 5))
response.raise_for_status()

print(type(response.content))
print(len(response.content))
```

适合保存图片、压缩文件和其他原始响应内容。这里的“内容”通常经过 Requests 支持的传输内容解码，例如 gzip 解压；它不一定等于线路上逐字节的原始报文。

如果你要做协议级原始抓包，Requests 的普通响应属性不是完整抓包工具。

### 7.3 `.text` 是 str

```python
response = requests.get("http://127.0.0.1:8765/text", timeout=(3, 5))
response.raise_for_status()

print(response.encoding)
print(response.text)
```

Requests 会根据响应头等信息确定文本解码方式。服务器声明不准确时，文本可能看起来乱码。

确认真实格式后，可以在访问 `.text` 前设置：

```python
response.encoding = "utf-8"
```

不要把“强行设 UTF-8”当成所有乱码的万能解法。来源实际不是 UTF-8 时，仍会产生错误解释。

### 7.4 `.json()` 不保证响应成功

```python
response = requests.get("http://127.0.0.1:8765/status/500", timeout=(3, 5))
print(response.json())
```

本地服务会返回合法 JSON 错误体，因此解析可以成功，但状态仍是 500。

正常成功路径通常先 `raise_for_status()`，再解析和验证。需要读取错误体提供更具体提示时，也可以在错误处理分支中尝试解析，但必须允许错误体不是 JSON。

### 7.5 空响应不能直接当 JSON

```python
response = requests.get("http://127.0.0.1:8765/empty", timeout=(3, 5))
response.raise_for_status()

if response.status_code == 204:
    result = None
else:
    result = response.json()
```

204 通常表示没有响应体。`json()` 不会因为头里声明 JSON 就把空字节自动变成 `{}`。

## 第八章：状态码、业务码与异常

### 8.1 状态类别

| 范围 | 常见含义 | 示例 |
| --- | --- | --- |
| 2xx | 请求在 HTTP 层成功处理 | 200、201、204 |
| 3xx | 重定向或缓存相关行为 | 301、302、304 |
| 4xx | 请求或权限等客户端侧问题 | 400、401、403、404、429 |
| 5xx | 服务端处理或上游失败 | 500、502、503、504 |

实际含义以端点协议为准。比如 404 可以表示资源确实不存在，也可能被服务用于隐藏未授权资源信息，客户端不应无依据推断内部实现。

### 8.2 `raise_for_status()` 的返回值

它成功时通常返回 `None`，失败时抛异常。因此不要写：

```python
# 错误用法：成功时 response 会被赋值成 None。
response = response.raise_for_status()
```

应单独调用，保留原响应对象。

### 8.3 常见异常层次

```python
import requests

try:
    response = requests.get("http://127.0.0.1:8765/hello", timeout=(3, 5))
    response.raise_for_status()
    data = response.json()
except requests.exceptions.Timeout:
    print("连接或读取等待超时")
except requests.exceptions.ConnectionError:
    print("无法建立连接，或连接过程失败")
except requests.exceptions.HTTPError as error:
    print("HTTP 状态失败：", error.response.status_code)
except requests.exceptions.JSONDecodeError:
    print("响应内容不是合法 JSON")
except requests.exceptions.RequestException as error:
    print("其他请求错误：", error)
```

应把更具体异常放前面。`ConnectTimeout` 同时与连接错误和超时类有关，分类顺序影响它被哪个分支处理。

### 8.4 不要用响应对象布尔值判断“是否拿到响应”

Requests 的 `Response` 布尔值会结合状态判断。一个 404 响应对象可能在 `if response:` 中表现为假，但它依然是你收到的真实响应。

```python
if response is not None:
    print(response.status_code)
```

用身份判断区分“没有对象”；用状态码或 `raise_for_status()` 判断 HTTP 状态。这是一个容易导致错误处理分支丢失响应信息的细节。

### 8.5 HTTP 200 里的业务失败

有些接口返回：

```json
{"success": false, "error": "余额不足"}
```

即使 HTTP 状态为 200，业务仍然失败。`raise_for_status()` 不知道这个字段的含义。

你应按接口契约验证 `success`、`code` 或具体结果结构。不要在所有项目中假设业务成功码都叫 `code` 或都等于零。

## 第九章：超时，必须把含义讲准确

### 9.1 Requests 默认不替你设置业务超时

普通调用不传 `timeout`，可能等待很久。对脚本和服务调用，应该根据任务明确传入合适超时。

```python
requests.get(url, timeout=10)
```

单一数字会应用于相应连接和读取等待设置，不是严格的整个任务十秒截止时间。

### 9.2 连接超时与读取超时

连接超时主要约束建立连接相关等待；读取超时约束等候响应数据的等待间隔。

```python
requests.get(url, timeout=(3.05, 10))
```

这个例子表达较短连接等待和较长读取等待。数值不是通用最佳值，应根据网络环境和服务时延目标确定。

域名解析、多个地址尝试、代理和重试都可能影响完整耗时，不应把这两个参数相加就当成严格总上限。[官方解释：Timeouts](https://requests.readthedocs.io/en/latest/user/advanced/#timeouts)

### 9.3 本地实验：读取超时

```python
import requests

try:
    requests.get("http://127.0.0.1:8765/delay", timeout=(1, 0.2))
except requests.exceptions.ReadTimeout:
    print("服务尚未在读取等待时间内返回数据")
```

本地 `/delay` 会先等待两秒，所以通常触发读取超时。这个实验不需要依赖某个外网服务器恰好变慢。

超时只说明客户端没有在要求时间内取得相应数据，不能证明服务器没有收到请求或没有执行过动作。

### 9.4 流式下载时的超时

如果服务器不断在读取超时以内送来少量字节，一个很长的下载仍可能持续远超参数中的数值。

要限制整体下载时长，还需要跟踪单调时钟截止点、限制累计大小，并设计取消机制。仅在每轮循环检查时间，也不能中断正在阻塞的底层读取，所以读取超时和总期限应一起考虑。

### 9.5 超时之后是否重试？

读取数据的 GET 往往可以按策略重试。创建订单、提交支付或发送消息的 POST，则可能已经在服务器完成，只是响应在途中丢失。

此时盲目重试可能重复执行。服务端幂等键、请求 ID 和查询操作状态的协议，比客户端简单 `while True` 更重要。

## 第十章：同一个网站要访问很多次，为什么需要 `Session`？

前面每次都直接写：

```python
requests.get(url, timeout=(3, 5))
```

对于一两次独立请求，这样很方便。

假设你现在要访问同一服务一百次，而且每次都要带相同的身份信息、相同的请求头，还要保留服务器设置的 Cookie。

如果每次都重新写一遍参数，既重复，也容易漏掉某个设置。

这时可以创建一个会话对象。

### 10.1 最小写法

```python
import requests

with requests.Session() as session:
    first = session.get("http://127.0.0.1:8765/hello", timeout=(3, 5))
    second = session.get("http://127.0.0.1:8765/echo", timeout=(3, 5))

    first.raise_for_status()
    second.raise_for_status()
    print(first.json())
    print(second.json()["method"])
```

`Session` 可以理解为一次持续交互所共享的客户端状态。

它不等于服务器进程，也不会因为创建了对象就自动登录网站。

你仍然需要发送请求，服务端仍然按自己的规则识别用户。

### 10.2 为什么把它放进 `with`？

会话维护底层连接池等资源。

离开 `with` 后，库会关闭会话所管理的相关连接资源。

这和 `with open(...)` 的思路相似：在一个明确范围内使用，离开时安排清理。

如果会话要贯穿应用生命周期，也可以在初始化时创建，在应用结束时显式调用 `close()`。

关键是明确谁创建、谁复用、谁关闭，不要把每次请求临时创建的新对象误当成长期连接复用。

### 10.3 设置默认请求头

```python
import requests

with requests.Session() as session:
    session.headers.update({
        "User-Agent": "python-learning-client/1.0",
        "X-Demo": "session-default",
    })

    response = session.get(
        "http://127.0.0.1:8765/echo",
        timeout=(3, 5),
    )
    response.raise_for_status()
    data = response.json()

    print(data["user_agent"])
    print(data["x_demo"])
```

本地服务通常返回：

```text
python-learning-client/1.0
session-default
```

`headers.update(...)` 是更新默认头映射，不是发送请求。

真正访问服务器发生在后面的 `session.get()`。

### 10.4 某一次请求想覆盖默认值呢？

```python
response = session.get(
    "http://127.0.0.1:8765/echo",
    headers={"X-Demo": "only-this-request"},
    timeout=(3, 5),
)
```

方法级设置可以覆盖相应默认项。

它通常不会因此永久修改会话默认头。

所以，下一次不带覆盖参数的请求，仍然可以使用 `session-default`。

这和你主动写 `session.headers["X-Demo"] = ...` 是不同的状态修改方式。

### 10.5 Session 还能复用连接，是什么意思？

HTTP 请求经常依赖 TCP 和 TLS 连接。

重新建立连接可能涉及网络往返和握手成本。

会话通过底层连接池，在条件允许时复用已有连接，减少重复建立的开销。

但能否复用还取决于服务器是否允许保持连接、响应是否已消费或关闭、目标地址与连接配置等因素。

本篇简化练习服务器主要用于观察内容，没有把所有生产连接保持行为都实现出来。因此，不能只计时本地两个请求，就据此声称获得某个固定倍数的连接池性能提升。

### 10.6 可以让多个线程随意修改一个 Session 吗？

会话包含 Cookie、默认头和认证等可变状态。

即使底层连接池具有相应并发机制，也不能直接推出整个会话的所有状态修改都可以不加约束地跨线程共享。

个人脚本通常每个受控工作单元使用自己的会话；确实共享时，要避免并发修改状态，并根据项目使用方式验证。

不要把“连接池可复用”翻译成“任何对象在任何线程里都完全安全”。

## 第十一章：Cookie 是怎么保存和再次发送的？

### 11.1 先观察一次服务端设置

本地练习服务有 `/cookie` 接口。

它会在响应中设置一个名为 `demo` 的练习 Cookie。

```python
import requests

response = requests.get(
    "http://127.0.0.1:8765/cookie",
    timeout=(3, 5),
)
response.raise_for_status()

print(response.cookies.get("demo"))
```

通常输出：

```text
learn
```

Cookie 不是 JSON 字段。

它通过响应头中的 `Set-Cookie` 等协议机制传递。

### 11.2 再请求一次，为什么可能没有带上？

如果你每次都使用独立顶层调用，就不应假设前一次响应 Cookie 会自动出现在下一次独立调用里。

希望保持会话状态，可以这样做：

```python
import requests

with requests.Session() as session:
    response = session.get("http://127.0.0.1:8765/cookie", timeout=(3, 5))
    response.raise_for_status()

    response = session.get("http://127.0.0.1:8765/echo", timeout=(3, 5))
    response.raise_for_status()
    print(response.json()["cookie"])
```

你通常会看到包含：

```text
demo=learn
```

的 Cookie 头内容。

### 11.3 Cookie 会发送给任意网站吗？

不会按“只要存在就发给所有请求”的方式理解。

Cookie 有域、路径、Secure 等属性和匹配规则。

例如，一个站点给出的 Cookie，不应无条件被发送到毫不相关的另一个域。

调试时应检查目标 URL、Cookie 所属域与路径，以及是否要求 HTTPS。

不要为了“登录没保持”就把所有 Cookie 手工拼成一行到处发送，这可能破坏范围限制并泄露身份信息。

### 11.4 Cookie 和登录有什么关系？

很多网站在认证成功后，用 Cookie 保存一个会话标识。

客户端后续带这个标识，服务器才能识别对应会话。

但 Cookie 本身只是协议中的状态传递机制，不保证每个 Cookie 都代表登录，也不保证服务器一定采用 Cookie 认证。

有的 API 使用 Bearer Token，有的使用 Basic Auth，有的采用其他签名或身份方案。

应该按接口文档实现，不能看到 Cookie 就猜测它的业务含义。

## 第十二章：认证信息怎样发送？

### 12.1 Basic Auth 的最小练习

本地 `/auth` 接受练习账号 `demo` 和密码 `practice`。

```python
import requests

response = requests.get(
    "http://127.0.0.1:8765/auth",
    auth=("demo", "practice"),
    timeout=(3, 5),
)
response.raise_for_status()
print(response.json())
```

通常得到：

```text
{'user': 'demo'}
```

`auth=(username, password)` 让库按相应 Basic 认证规则生成请求头。

### 12.2 Basic 里面有编码，是否就加密了？

Basic 凭证常使用 Base64 表示。

Base64 是编码，不是保密加密，能够被还原。

真实远程认证通常应在正确验证证书的 HTTPS 连接上使用，避免凭证在不安全链路暴露。

本地示例使用 HTTP，是为了在本机回环练习，账号密码也是专用虚构内容。

### 12.3 Bearer Token

很多接口要求：

```text
Authorization: Bearer <token>
```

客户端常写成：

```python
import os
import requests

token = os.environ["DEMO_API_TOKEN"]

response = requests.get(
    "https://api.example.invalid/profile",
    headers={"Authorization": f"Bearer {token}"},
    timeout=(3, 10),
)
```

这里的 `.invalid` 地址用于说明结构，不能当成真实可调用服务。

应把 URL 换成你实际有权访问的服务地址，再按它的认证契约调用。

### 12.4 为什么示例从环境变量取 Token？

这样可以避免把真实令牌直接写进源代码和提交历史。

但环境变量不是绝对保密容器，仍然可能通过日志、进程环境访问或错误报告泄露。

不要把完整认证头打印出来，也不要在排错截图里公开令牌。

### 12.5 认证失败与没有权限不同

401 常与需要认证或认证失败有关。

403 常与请求被理解、但当前权限不允许有关。

具体服务可能采用自己的约定，甚至为了隐藏资源而返回 404。

客户端应保留状态和安全的错误信息，按文档解释，不应把所有失败都归为“网络不好”。

## 第十三章：重定向——为什么返回的地址和请求时不同？

### 13.1 本地观察

```python
import requests

response = requests.get(
    "http://127.0.0.1:8765/redirect",
    timeout=(3, 5),
)

print(response.status_code)
print(response.url)
print([item.status_code for item in response.history])
print(response.json())
```

通常最后得到 200，最终 URL 指向 `/hello`，历史中包含 302。

因为默认 GET 会跟随相应重定向。

### 13.2 如果只想看服务器第一次返回什么呢？

```python
response = requests.get(
    "http://127.0.0.1:8765/redirect",
    allow_redirects=False,
    timeout=(3, 5),
)

print(response.status_code)
print(response.headers.get("Location"))
```

你通常看到：

```text
302
/hello
```

这次没有继续访问跳转目标。

### 13.3 `Location` 可能是相对地址

不能假设它总是一个完整的 `https://...`。

需要根据当前响应 URL 和 URL 规则解析。

客户端库在自动重定向过程中会处理常见情况；自己实现时应使用 URL 工具，不要拿文件系统 `Path` 去连接网络地址。

### 13.4 重定向过程中方法会不会变化？

不同状态码对方法保留有不同语义，历史兼容行为也存在差异。

例如 307、308 的设计强调保留原方法，某些其他重定向可能把原 POST 转为 GET。

对于具有副作用的请求，不要只看“最后返回成功”就忽略中间跳转过程。

如果服务契约要求严格处理，应检查具体状态和库行为，必要时禁用自动跳转，自己按允许规则处理。

### 13.5 身份信息会不会跟着跳到另一个站点？

库会对跨主机认证等进行相应处理，但你仍然不能把自动跳转当成对任意目标的授权验证。

特别是代理服务、URL 抓取器和服务器端下载功能，应约束允许访问的目标以及重定向后的目标。

用户提供的初始 URL 看起来允许，不表示后面所有跳转地址也应该被允许。

## 第十四章：下载文件，为什么 `stream=True` 很重要？

### 14.1 小文件直接取 `.content`

```python
from pathlib import Path
import requests

response = requests.get("http://127.0.0.1:8765/download", timeout=(3, 5))
response.raise_for_status()

path = Path("requests_practice/download-small.bin")
path.parent.mkdir(parents=True, exist_ok=True)
path.write_bytes(response.content)
```

这会把整个响应内容放入内存。

本地示例文件不大，容易观察。

真实文件达到几 GiB 时，整份缓存就可能不合适。

### 14.2 分块读取响应

```python
from pathlib import Path
import requests

path = Path("requests_practice/download-stream.bin")
path.parent.mkdir(parents=True, exist_ok=True)

with requests.get(
    "http://127.0.0.1:8765/download",
    stream=True,
    timeout=(3, 5),
) as response:
    response.raise_for_status()

    with path.open("wb") as target:
        for chunk in response.iter_content(chunk_size=64 * 1024):
            if chunk:
                target.write(chunk)
```

`stream=True` 表示不要在返回响应前就把整个响应体下载完交给 `.content`。

`iter_content()` 在迭代过程中逐步取得内容。

### 14.3 `chunk_size` 是固定每次返回这么多吗？

它表示请求的分块大小意图，不保证每一块都恰好等长。

最后一块可能更短，底层解码和传输行为也会影响实际分块。

代码应处理任意非空块，而不把固定块长度当成数据格式边界。

### 14.4 为什么响应本身也放进 `with`？

流式响应可能没有被完整消费。

例如写文件时磁盘出错，循环中途抛异常；或者你只想看文件开头就提前退出。

退出响应上下文可以安排释放相关连接资源。

如果既不读完也不关闭，连接池中的资源就可能长期被占用。

### 14.5 设置 `stream=True`，再访问 `.content` 会怎样？

`.content` 仍然需要完整内容，所以会消费并缓存剩余响应体。

这会抵消你为了控制内存而采用的逐块处理策略。

不要先写“流式下载”，然后为了打印调试又把整个 `.content` 读出来。

### 14.6 下载文件名应该直接信任响应头吗？

响应头可能提供 `Content-Disposition` 文件名，但它来自远端输入。

不应把含有路径分隔、`..` 或绝对路径的名字直接拼进本地输出目录。

最清楚的个人脚本可以让用户明确指定输出路径。

如果自动生成文件名，需要按应用约定处理名称、冲突和路径范围，并考虑符号链接和并发修改等边界。

### 14.7 只成功创建了文件，不等于下载完整

中途断网或读取超时，可能留下部分目标。

因此，更可靠的方式是先写临时文件，成功读取到结束后，再发布到正式名称。

还可以按服务契约核对大小或摘要，但要清楚验证对象是解压后的内容还是原始传输字节。

`Content-Length` 在压缩、代理和分块传输等场景下也不能不加理解地与落盘内容长度直接比较。

## 第十五章：完整实践——有大小限制的下载与临时文件发布

我们把前一章的下载程序改成一个可以复用的函数。

先明确它的规则：

目标 URL 由调用者提供。

输出路径由调用者明确指定。

允许覆盖已有正式结果，但只有新下载完整结束后才替换。

对实际接收内容设置大小上限。

出错后清理本次临时文件。

### 15.1 完整代码

```python
import hashlib
import os
import tempfile
from pathlib import Path

import requests


def download_file(session, url, destination, max_bytes=20 * 1024 * 1024):
    if type(max_bytes) is not int or max_bytes < 0:
        raise ValueError("max_bytes 必须是非负整数")

    destination = Path(destination).absolute()
    destination.parent.mkdir(parents=True, exist_ok=True)
    temp_path = None
    total = 0
    digest = hashlib.sha256()

    try:
        with session.get(url, stream=True, timeout=(3, 10)) as response:
            response.raise_for_status()

            with tempfile.NamedTemporaryFile(
                mode="wb",
                dir=destination.parent,
                prefix=destination.name + ".",
                suffix=".part",
                delete=False,
            ) as target:
                temp_path = Path(target.name)

                for chunk in response.iter_content(chunk_size=64 * 1024):
                    if not chunk:
                        continue

                    total += len(chunk)
                    if total > max_bytes:
                        raise ValueError("下载内容超过允许大小")

                    target.write(chunk)
                    digest.update(chunk)

                target.flush()
                os.fsync(target.fileno())

        temp_path.replace(destination)
        return {
            "path": str(destination),
            "size_bytes": total,
            "sha256": digest.hexdigest(),
        }
    finally:
        if temp_path is not None:
            temp_path.unlink(missing_ok=True)


with requests.Session() as session:
    result = download_file(
        session,
        "http://127.0.0.1:8765/download",
        "requests_practice/complete.bin",
    )
    print(result)
```

### 15.2 为什么没有只检查 `Content-Length`？

响应头由远端提供，可能缺失，也可能描述的是与你最终消费内容不同的编码层。

而且检查头之后，真实读取仍然需要遵守限制。

这里在每次取得内容时累计实际字节数量，超过上限就终止。

它控制的是 `iter_content()` 交给程序的内容，不是所有底层网络和解码资源的完整沙箱。

### 15.3 为什么临时文件与目标放在同一目录？

这样有利于让最后的替换发生在同一文件系统中。

跨文件系统不能简单依赖原子重命名语义。

临时文件用随机名字创建，也比固定写 `download.part` 更能减少不同任务之间的冲突。

### 15.4 为什么先关闭，再替换？

文件写入结束后先离开内部 `with`，确保我们持有的临时文件流关闭。

随后离开响应上下文，释放网络资源。

最后才把临时结果发布到正式路径。

这个顺序也更容易兼容 Windows 对打开文件的共享限制。

### 15.5 计算 SHA-256 是不是已经验证文件可信？

这里只计算摘要，方便后续比较。

如果没有来自可信来源的预期摘要、签名或认证，单独计算一个哈希不能证明文件作者或内容来源。

假设攻击者同时替换文件和一个不可信网页上的摘要，两者仍然可以匹配。

摘要解决的是内容标识与比较问题，来源认证还需要额外机制。

### 15.6 这个函数的边界在哪里？

它明确允许替换已有目标。

不保留原文件全部元数据。

不提供多个下载结果同时成功的事务。

不阻止多个写入者相互覆盖。

`fsync()` 之后的断电持久性仍与文件系统、目录元数据和存储设备有关。

这些边界应该随着函数交给使用者，而不是只把它命名为“安全下载”就省略说明。

## 第十六章：完整实践——分页读取，不丢页、不无限请求

### 16.1 一个分页接口通常怎么设计？

本地 `/items` 接受：

```text
page=1
page_size=2
```

返回：

```json
{
  "items": [
    {"id": 1, "name": "Python 文件处理"},
    {"id": 2, "name": "路径对象"}
  ],
  "page": 1,
  "page_size": 2,
  "total": 5
}
```

你需要继续请求第二页、第三页，才能取得全部记录。

Requests 不会根据 `total` 自动帮你翻页，因为不同服务的分页协议不同。

### 16.2 实现一个逐条产生结果的生成器

```python
import requests


def iter_items(session, base_url, page_size=2, max_pages=100):
    if type(page_size) is not int or not 1 <= page_size <= 100:
        raise ValueError("page_size 必须是 1 到 100 的整数")
    if type(max_pages) is not int or max_pages < 1:
        raise ValueError("max_pages 必须是正整数")

    seen_ids = set()

    for page in range(1, max_pages + 1):
        response = session.get(
            base_url + "/items",
            params={"page": page, "page_size": page_size},
            timeout=(3, 5),
        )
        response.raise_for_status()
        data = response.json()

        if not isinstance(data, dict):
            raise ValueError("分页响应顶层必须是对象")
        if type(data.get("page")) is not int or data["page"] != page:
            raise ValueError("服务端返回的页号与请求不一致")

        items = data.get("items")
        if not isinstance(items, list):
            raise ValueError("items 必须是数组")

        if not items:
            return

        for item in items:
            if not isinstance(item, dict) or type(item.get("id")) is not int:
                raise ValueError("记录缺少整数 id")

            if item["id"] in seen_ids:
                raise ValueError("分页结果出现重复 id")

            seen_ids.add(item["id"])
            yield item

        if len(items) < page_size:
            return

    raise RuntimeError("达到最大页数，未确认数据已经结束")


with requests.Session() as session:
    for item in iter_items(session, "http://127.0.0.1:8765"):
        print(item["id"], item["name"])
```

### 16.3 为什么设置 `max_pages`？

如果服务器错误地反复返回满页，或者客户端没有正确更新参数，简单 `while True` 可能一直请求。

最大页数让程序有一个明确边界。

达到上限后抛错误，比静默停止并把部分结果当作完整结果更清楚。

### 16.4 为什么检查服务端返回页号？

请求了第二页，却收到第一页数据，可能是参数名写错、服务器忽略参数或缓存配置异常。

不检查时，你可能把第一页重复保存许多次。

接口如果没有返回页号，可以按其协议采用其他一致性判断，例如游标是否变化、是否出现重复业务 ID。

### 16.5 记录数不足一页，就一定结束吗？

这是本地练习接口采用的规则。

其他 API 可能即使一页很短仍提供下一游标，或者允许动态过滤后返回不足页数据。

所以分页结束条件必须来自接口契约。

不要把某个服务上的经验写成所有分页接口都通用的规律。

### 16.6 分页期间数据变化怎么办？

如果第一页读取后，有记录插入或删除，按页号和偏移量翻页可能出现重复或遗漏。

客户端的去重只能发现或缓解部分问题，不能凭空恢复没读到的记录。

需要稳定快照时，应使用服务端提供的快照、游标、版本或时间边界等机制。

这属于数据一致性问题，不能只靠 HTTP 库解决。

### 16.7 生成器就一定省内存吗？

本例逐条 `yield`，没有保存全部记录。

但 `seen_ids` 会随着记录数增加而增长。

它为了检查重复而保留 ID，因此内存不完全固定。

如果数据量特别大，要在去重能力、完整性要求和存储方式之间做选择，可以借助数据库或外部存储，而不是一句“生成器节省内存”就结束分析。

## 第十七章：请求失败后应该重试吗？

### 17.1 先分清失败发生在哪一层

连接建立失败，和已经收到 400 响应不同。

读取超时，和服务明确返回业务拒绝也不同。

格式损坏，和暂时的 503 过载仍然不同。

如果你把所有异常都放进一个无限重试循环，可能让本来应该修正的请求持续打扰服务器。

### 17.2 哪些情况可能值得有限重试？

临时连接失败、某些网关故障、服务短暂过载等，可能通过稍后重试恢复。

但是否可以重复发送，取决于请求语义。

GET 常见读取操作通常容易制定重试政策。

POST 创建订单可能已经成功，只是响应丢失；没有幂等机制就重复发，可能产生重复订单。

### 17.3 先写一个只用于受控 GET 的简单重试

```python
import time
import requests


def get_with_retry(session, url, attempts=3):
    if type(attempts) is not int or attempts < 1:
        raise ValueError("attempts 必须是正整数")

    for attempt in range(attempts):
        try:
            response = session.get(url, timeout=(3, 5))
            response.raise_for_status()
            return response
        except (requests.exceptions.ConnectionError, requests.exceptions.Timeout):
            if attempt + 1 == attempts:
                raise
            time.sleep(0.2 * (2 ** attempt))
```

这个版本只对列出的传输异常重试。

HTTP 状态错误不会被这个 `except` 捕获，因此不会无条件重试 400 或 401。

返回的普通响应已按默认方式消费内容，调用者仍应按需要管理它。

### 17.4 为什么有退避等待？

如果服务已经过载，所有客户端立即连续重试，会增加压力。

指数退避让后续尝试之间的间隔逐渐增加。

真实系统还常加随机抖动，减少许多客户端在同一时刻再次发送。

重试还会扩大总耗时，因此要与整体任务期限、最大尝试次数和调用方预期一起设计。

### 17.5 使用 `HTTPAdapter` 与 `Retry`

Requests 通过适配器对接底层传输。

可以为会话配置有限的重试策略：

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(
    total=2,
    connect=2,
    read=0,
    status=2,
    allowed_methods={"GET", "HEAD"},
    status_forcelist={429, 502, 503, 504},
    backoff_factor=0.5,
    respect_retry_after_header=True,
)

adapter = HTTPAdapter(max_retries=retry)

with requests.Session() as session:
    session.mount("http://", adapter)
    session.mount("https://", adapter)

    response = session.get("http://127.0.0.1:8765/hello", timeout=(3, 5))
    response.raise_for_status()
    print(response.json())
```

这里的 `total=2` 是重试预算，通常意味着初次尝试之外最多再做相应次数的重试，不是“总共只发两次”的简单同义词。

具体分类预算、重试耗尽后的异常包装和返回政策，还要按 `Retry` 配置理解。

### 17.6 `allowed_methods` 为什么只放 GET 和 HEAD？

这是本例主动限制的政策。

它没有证明其他方法绝对不能重试，而是避免未经业务设计就重发可能具有副作用的操作。

如果服务对创建请求提供幂等键，并明确说明重复请求的处理方式，可以在理解契约后制定其他策略。

### 17.7 `Retry-After` 能不能完全信任？

服务器可以建议等待秒数或日期。

客户端仍需要考虑自己的任务期限、最大等待和可接受负载。

配置了尊重头字段，不代表整个程序就自动获得无限等待的许可。

对于长期运行任务，还应记录尝试次数和最终状态，避免静默卡住。

官方接口：[Requests 适配器](https://requests.readthedocs.io/en/latest/api/#requests.adapters.HTTPAdapter)、[urllib3 Retry](https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html#urllib3.util.Retry)。

## 第十八章：HTTPS、证书和 `verify`

### 18.1 HTTPS 不只是 URL 多了一个字母

HTTPS 使用 TLS 保护通信，涉及加密、完整性和服务器身份验证等机制。

客户端验证证书，是为了确认自己连接的是允许信任的服务器身份。

如果证书验证失败，可能是服务配置、系统时间、信任链、企业证书或网络中间设备等问题。

不能只根据“把验证关掉后能连上”，就认为问题已经正确解决。

### 18.2 默认验证应保留

```python
response = requests.get(
    "https://api.example.invalid/data",
    timeout=(3, 10),
)
```

这里的说明地址需要替换成真实授权服务。

普通调用默认验证服务器证书。

不建议把 `verify=False` 当成常规排错配置长期留在代码里。

### 18.3 如果公司使用自己的 CA 呢？

在你已经取得可信 CA 文件、并确认服务契约的情况下，可以明确指定证书信任文件：

```python
response = requests.get(
    "https://internal.example.invalid/data",
    verify="/path/to/company-ca.pem",
    timeout=(3, 10),
)
```

这表示使用指定 CA 材料完成验证，不是关闭验证。

文件路径必须对应你环境里的真实可信证书集合，不能照抄占位路径。

### 18.4 客户端证书与服务器证书验证不同

某些服务要求双向 TLS，客户端还要提供自己的证书。

```python
response = requests.get(
    "https://internal.example.invalid/data",
    cert=("client-cert.pem", "client-key.pem"),
    timeout=(3, 10),
)
```

`cert` 用于客户端身份材料。

`verify` 用于是否及如何验证服务器证书。

两个参数处理不同方向的问题，不能因为都与证书有关就互相替代。

### 18.5 证书错误的诊断顺序

先检查请求地址是否正确、系统时间是否合理。

再检查证书链和信任来源，以及是否经过了需要额外 CA 的企业网络环境。

然后根据当前服务文档配置正确的 CA 或客户端材料。

涉及真实凭证时，不要把私钥内容打印到日志，也不要为排错把它上传到不相关服务。

## 第十九章：代理、系统路由与 WSL 环境

### 19.1 代理和路由是不同层次

Windows 的 TUN 虚拟网卡可能让流量通过系统路由转发。

HTTP 或 SOCKS 代理则是应用显式连接到代理服务器，再由代理转发请求。

主机使用了 TUN，不表示 Python 请求一定还需要再手动设置一个应用代理。

叠加不匹配配置，反而可能造成请求失败或路径绕行。

### 19.2 先观察现状，不要直接改

排错时可以检查环境里是否存在代理变量，但注意它们的值可能含认证资料。

```python
import os

names = ["HTTP_PROXY", "HTTPS_PROXY", "ALL_PROXY", "NO_PROXY"]
for name in names:
    print(name, "已设置" if name in os.environ else "未设置")
```

这段代码只显示是否存在，不把完整代理地址和凭证输出出来。

### 19.3 Requests 会读取环境配置

默认会话的环境信任机制可能使用代理变量等设置。

因此，同一段代码换一个终端、虚拟环境或启动器，网络行为可能受继承环境影响。

但不应因此直接清空所有环境变量。先确认当前配置、服务目标以及用户期望的路由方式。

### 19.4 显式代理是有意选择

如果你明确需要应用代理，且已确认代理地址和类型，可以传入相应配置。

```python
proxies = {
    "http": "http://127.0.0.1:7890",
    "https": "http://127.0.0.1:7890",
}
```

这里仅展示映射结构，端口是占位示例，不代表你的机器真的在这个位置提供代理。

不要因为网络暂时失败，就自动把教程中的端口写进项目。

### 19.5 `trust_env=False` 是否只是关闭代理？

它会影响会话对环境配置的信任，不只局限于一个代理开关，还可能影响其他相关环境和认证行为。

因此，应先理解影响范围，再在受控测试客户端中使用。

本地离线验证如果为了隔离测试环境，可以明确说明禁用继承环境；日常真实请求仍按用户当前系统/TUN配置和显式代理选择运行。

### 19.6 WSL 的 localhost 也要看服务运行在哪里

如果客户端和练习服务器都运行在同一个 WSL Linux 环境，`127.0.0.1` 指向同一网络命名空间中的回环服务，最容易练习。

如果服务运行在 Windows，客户端在 WSL，连通方式可能受到 WSL 网络模式、监听地址、防火墙和转发设置影响。

不要把 Windows 服务没监听在可达位置的问题，误认为 Requests 的参数编码错误。

先确认服务实际监听在哪里，再确认请求目标地址。

## 第二十章：请求发送之前，能不能先看看究竟准备了什么？

有时你怀疑参数没有正确编码，或者 JSON 传成了表单。

如果每次都真实发送，再从服务器日志猜测，排错会比较慢。

Requests 提供请求对象和准备后的请求对象，让你观察实际将要发送的内容。

### 20.1 `Request` 与 `PreparedRequest`

```python
import requests

request = requests.Request(
    method="GET",
    url="http://127.0.0.1:8765/echo",
    params=[("keyword", "Python 文件"), ("tag", "backend")],
    headers={"X-Demo": "prepared"},
)

prepared = request.prepare()

print(prepared.method)
print(prepared.url)
print(prepared.headers["X-Demo"])
print(prepared.body)
```

这段代码只准备请求，不发送网络访问。

`Request` 保存你提供的较高层意图。

`PreparedRequest` 表示经过相应编码和整理后的请求资料。

### 20.2 为什么准备后 URL 变长了？

中文、空格等字符被转换成适合 URL 的编码表示。

这是为了在传输中明确参数边界，不表示文本内容被随意改变。

服务端按对应规则解析后，可以恢复原先的参数值。

不要为了让 URL 看起来好看，就把编码后的 `%` 序列手工删掉。

### 20.3 观察 JSON 请求体

```python
import requests

prepared = requests.Request(
    "POST",
    "http://127.0.0.1:8765/echo",
    json={"name": "小林", "active": True},
).prepare()

print(prepared.headers.get("Content-Type"))
print(type(prepared.body).__name__)
print(prepared.body)
```

你可以确认请求体确实按 JSON 准备，而不是仅仅在 URL 后追加参数。

具体转义和字节表现可能随库版本和实现细节不同，判断协议含义时应关注解析后的数据与声明的内容类型。

### 20.4 如果我要保留 Session 默认设置呢？

直接 `request.prepare()` 不会自动套用某个你另外创建的 Session 状态。

可以通过会话准备：

```python
import requests

with requests.Session() as session:
    session.headers.update({"X-Demo": "session-value"})

    request = requests.Request(
        "GET",
        "http://127.0.0.1:8765/echo",
        params={"page": 1},
    )

    prepared = session.prepare_request(request)
    print(prepared.headers["X-Demo"])
```

这时默认头、Cookie 等会话相关资料可以参与准备过程。

### 20.5 真正发送准备后的请求

```python
import requests

with requests.Session() as session:
    request = requests.Request("GET", "http://127.0.0.1:8765/hello")
    prepared = session.prepare_request(request)

    settings = session.merge_environment_settings(
        prepared.url,
        proxies={},
        stream=None,
        verify=None,
        cert=None,
    )

    response = session.send(prepared, timeout=(3, 5), **settings)
    response.raise_for_status()
    print(response.json())
```

这里明确合并环境设置，是因为手动准备和发送的流程不应想当然地与所有高层调用步骤完全一致。

普通需求仍然优先使用 `session.get()` 或 `session.post()`，减少自己管理传输细节的负担。

### 20.6 能不能在准备后随意改 body？

理论上某些字段可以修改，但你可能同时需要更新长度、内容类型、签名和其他关联头。

如果只改正文，旧 `Content-Length` 仍然描述旧长度，就可能造成协议错误。

所以，准备对象主要用于理解与受控扩展，不应成为绕开正常编码流程的万能补丁入口。

### 20.7 调试时哪些内容不应直接打印？

`prepared.headers` 可能包含认证头和 Cookie。

`prepared.body` 可能包含密码、个人资料和上传内容。

可以只显示方法、脱敏后的 URL、媒体类型和大小等必要信息。

排错工具的便利，不应让敏感数据自动进入终端历史和共享日志。

## 第二十一章：上传文件时，文件流和请求体是什么关系？

### 21.1 文件对象要保持到请求结束

```python
from pathlib import Path
import requests

path = Path("requests_practice/upload.txt")

with path.open("rb") as source:
    response = requests.post(
        "http://127.0.0.1:8765/upload",
        files={"document": (path.name, source, "text/plain")},
        timeout=(3, 10),
    )

response.raise_for_status()
```

不要先离开文件的 `with`，再把已经关闭的流交给请求发送。

资源范围要覆盖真正使用它的时间。

### 21.2 同一个文件对象能直接重传第二次吗？

要看读取位置。

一次请求可能已经消费文件内容，把位置推进到末尾。

如果你要再次发送同一份内容，应在流支持定位的情况下回到开头，或者重新打开文件。

```python
source.seek(0)
```

这和普通 `open()` 的读取位置规则完全相通。

文件上传没有让这个流自动获得“每次都重新读全文”的特殊能力。

### 21.3 原始请求体上传

某些服务要求整个请求体就是文件内容，而不是 multipart 表单。

可以把文件流交给 `data`：

```python
from pathlib import Path
import requests

path = Path("requests_practice/upload.txt")

with path.open("rb") as source:
    response = requests.post(
        "http://127.0.0.1:8765/upload",
        data=source,
        headers={"Content-Type": "application/octet-stream"},
        timeout=(3, 10),
    )

response.raise_for_status()
print(response.json()["received_bytes"])
```

这与 `files=` 生成带字段名的 multipart 容器不同。

服务器必须支持你选择的格式。

### 21.4 `files=` 就一定完全流式、内存固定吗？

不能这样承诺。

Requests 的普通 multipart 编码流程可能读取并构造请求体，巨型上传需要特别考虑内存和专门的流式 multipart 方案。

原始流上传与 multipart 流式上传也是不同问题。

不要看到参数值是文件对象，就默认整条上传链路没有额外缓存。

### 21.5 生成器请求体与分块传输

某些客户端可以把可迭代字节片段作为请求体，采用相应传输方式。

但服务端也必须支持，例如 chunked 请求体解析。

本篇简化服务器主要按 `Content-Length` 读取，不用于演示任意 chunked 上传。

如果你随手把生成器传进去，服务端可能无法按预期读取，这不一定是生成器本身错误，而是双方协议能力没有对齐。

### 21.6 上传失败后，服务器有没有保存一部分？

客户端超时不能回答这个问题。

服务器可能没收到、收到一半、完整收到但处理失败，或者处理成功但响应没有回来。

正式上传服务应提供上传 ID、完整性检查、幂等或状态查询机制，客户端按契约恢复。

简单重发文件并不能保证不会产生重复记录或孤立临时对象。

## 第二十二章：把 HTTP 成功、JSON 解析与业务校验分开

### 22.1 三个步骤分别解决什么？

看这段代码：

```python
response.raise_for_status()
data = response.json()
name = data["name"]
```

第一步确认 HTTP 状态没有落入对应错误范围。

第二步尝试把响应体解析为 JSON。

第三步假设解析结果具有某个字段。

其中任何一步成功，都不能自动保证下一步成功。

### 22.2 先验证顶层类型

```python
if not isinstance(data, dict):
    raise ValueError("接口响应顶层应为对象")
```

JSON 顶层可以是数组、数字、字符串甚至 `null`。

因此，`.json()` 返回不一定是字典。

### 22.3 字段存在和字段类型也要分别检查

```python
name = data.get("name")

if not isinstance(name, str) or not name.strip():
    raise ValueError("name 必须是非空白字符串")
```

`get()` 缺失时返回默认值，但如果远端明确返回 `null`，你仍会得到 `None`。

这是否可接受，由接口契约决定。

### 22.4 布尔值不是所有地方都能当整数

```python
count = data.get("count")

if type(count) is not int or count < 0:
    raise ValueError("count 必须是非负整数")
```

这里用精确类型，是因为 Python 中 `bool` 是 `int` 子类。

如果你只写 `isinstance(True, int)`，会得到真。

对刚解析的普通 JSON 数据，精确区分布尔与整数有时非常必要。

### 22.5 Content-Type 是提示，不是结构证明

服务器声明 `application/json`，仍可能因为错误代理、服务故障或程序 bug 返回空内容或不合法文本。

客户端应该处理解析异常。

反过来，响应体碰巧能解析成 JSON，也不表示你可以忽略错误状态或授权信息。

### 22.6 不要把服务端错误页当成 JSON 错误本身

假如响应状态是 502，正文是网关生成的 HTML。

直接 `.json()` 可能报解码错误。

如果你只打印“JSON 不合法”，就丢失了更接近根因的网关状态。

所以通常先处理状态，再按协议解析；需要读取错误体时，在错误分支做有边界的尝试。

## 第二十三章：完整实践——一个小型 API 客户端类

重复调用同一个服务时，可以把连接会话、基础 URL、超时和错误解释集中起来。

这个练习不依赖大型框架。

### 23.1 我们希望怎么调用？

```python
with DemoClient("http://127.0.0.1:8765") as client:
    greeting = client.get_greeting()
    first_page = client.get_items(page=1, page_size=2)
```

调用者关心业务动作，底层公共请求步骤由客户端类组织。

### 23.2 完整实现

```python
import requests


class ApiError(RuntimeError):
    pass


class DemoClient:
    def __init__(self, base_url, timeout=(3, 5)):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        self.session = requests.Session()
        self.session.headers.update({"Accept": "application/json"})

    def close(self):
        self.session.close()

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        self.close()
        return False

    def request_json(self, method, path, **kwargs):
        if not path.startswith("/") or path.startswith("//"):
            raise ValueError("path 必须是本客户端的绝对资源路径")

        url = self.base_url + path
        timeout = kwargs.pop("timeout", self.timeout)

        try:
            with self.session.request(
                method,
                url,
                timeout=timeout,
                **kwargs,
            ) as response:
                response.raise_for_status()

                if response.status_code == 204:
                    return None

                try:
                    return response.json()
                except requests.exceptions.JSONDecodeError as error:
                    raise ApiError("服务器未返回合法 JSON") from error
        except requests.exceptions.RequestException as error:
            raise ApiError(f"请求 {method} {path} 失败") from error

    def get_greeting(self):
        data = self.request_json("GET", "/hello")
        if not isinstance(data, dict) or not isinstance(data.get("message"), str):
            raise ApiError("问候响应结构不符合约定")
        return data["message"]

    def get_items(self, page=1, page_size=2):
        if type(page) is not int or page < 1:
            raise ValueError("page 必须是正整数")
        if type(page_size) is not int or not 1 <= page_size <= 100:
            raise ValueError("page_size 超出范围")

        data = self.request_json(
            "GET",
            "/items",
            params={"page": page, "page_size": page_size},
        )
        if not isinstance(data, dict) or not isinstance(data.get("items"), list):
            raise ApiError("分页响应结构不符合约定")
        return data


with DemoClient("http://127.0.0.1:8765") as client:
    print(client.get_greeting())
    print(client.get_items(page=1, page_size=2)["items"])
```

### 23.3 为什么没有继承 Requests 的 Session？

这里采用组合：客户端对象内部持有一个会话。

这样公共业务接口与底层传输对象的职责比较清楚，也便于测试时替换内部会话行为。

这里还有一个容易漏掉的资源责任。

`request_json()` 返回的是解析后的 Python 数据，不会把响应对象交给调用者。

所以响应由这个方法自己通过 `with` 关闭。

即使调用者通过 `kwargs` 提供了 `stream=True`，遇到二百零四状态而提前返回，退出上下文时也仍然会关闭响应。

正常 JSON 则会在返回前读取并解析正文。

因此，这个封装始终是“取得完整 JSON 结果”的接口；它不会因为接收了 `stream=True` 就变成逐块输出 JSON 的生成器。

如果你需要流式下载，应使用前面单独设计的下载方法，并让响应关闭责任与迭代生命周期对应起来。

继承不是必须步骤。小型客户端通常不需要为了“面向对象”就扩展库内部类层次。

### 23.4 `kwargs.pop("timeout", self.timeout)` 做什么？

允许某一次请求显式覆盖默认超时。

取出之后，从 `kwargs` 中删除这个键，避免后面又显式传 `timeout=`、又在 `**kwargs` 里传同名参数。

这是 Python 参数解包与字典操作的组合，不是 Requests 特有语法。

### 23.5 为什么新异常写 `from error`？

对调用者提供简洁的业务错误，例如“服务器未返回合法 JSON”。

同时保留原始异常原因，便于开发者排查连接失败、状态失败或解码失败的细节。

不要把底层错误全部转换成一个没有原因的字符串，否则调试时难以知道真正发生了什么。

### 23.6 `__exit__` 为什么返回 False？

表示不主动吞掉代码块中的异常。

它负责关闭资源，但让原异常继续传播。

如果随手返回真，可能导致调用者没有发现任务已经失败。

### 23.7 为什么说这是教学客户端？

它没有实现完整 URL 授权策略、动态认证刷新、所有模式校验、请求总期限或完整重试政策。

这里只限制了资源路径的基本形状，不能据此宣称能防御所有 URL 绕过或不可信服务器行为。

真实项目应根据服务契约补足恰当能力，而不是把一个通用封装无限扩展成难以理解的万能客户端。

## 第二十四章：如何为请求添加可用的日志？

### 24.1 最简单的记录应该包括什么？

方法、受控的资源路径、状态码和耗时，通常比打印全部响应体更有用。

认证头、Cookie 和完整个人数据通常不应无条件输出。

```python
import logging
import time
import requests

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
logger = logging.getLogger(__name__)

started = time.perf_counter()
response = requests.get("http://127.0.0.1:8765/hello", timeout=(3, 5))
elapsed = time.perf_counter() - started

logger.info("GET /hello status=%s elapsed=%.3fs", response.status_code, elapsed)
response.raise_for_status()
```

### 24.2 为什么用 `perf_counter()`？

这里测量经过时长，不需要把时钟值转换为日历日期。

单调性能时钟更适合比较两个时间点之间的间隔。

如果系统墙上时钟在请求期间调整，使用普通日历时间相减可能不符合耗时测量目的。

### 24.3 `response.elapsed` 能替代所有总耗时吗？

它描述 Requests 记录的相应请求阶段耗时，通常截至响应头取得等阶段，不应直接当成完整流式下载、后续解码、JSON 校验和落盘的总耗时。

如果你关心整个业务动作，应在实际开始和结束处测量。

例如流式下载，应把迭代和文件关闭也包含在计时范围内。

### 24.4 响应钩子

```python
import requests


def report_status(response, *args, **kwargs):
    print("收到状态：", response.status_code)
    return response


with requests.Session() as session:
    session.hooks["response"].append(report_status)
    response = session.get("http://127.0.0.1:8765/hello", timeout=(3, 5))
    response.raise_for_status()
```

钩子让你集中观察响应事件。

回调返回对象时，会参与后续响应处理，因此不要无意返回一个字符串或整数，把原响应替换掉。

### 24.5 钩子里能不能直接读取完整响应体？

可以发生，但要考虑副作用。

对流式请求，如果钩子主动访问 `.content` 或 `.text`，可能提前消费并缓存整个响应，破坏调用者原本的流式处理计划。

日志通常只需要头和状态，不必为记录一条状态就读完整下载内容。

## 第二十五章：离线测试，怎样验证编码而不访问网络？

### 25.1 准备对象本身可以验证很多内容

```python
import json
import requests

prepared = requests.Request(
    "POST",
    "http://example.invalid/items",
    json={"name": "小林", "count": 3},
).prepare()

assert prepared.method == "POST"
assert prepared.headers["Content-Type"].startswith("application/json")
assert json.loads(prepared.body) == {"name": "小林", "count": 3}
```

这段代码不发送请求。

它验证的是客户端编码结果，不验证真实服务器是否接收和处理。

### 25.2 用标准库 Mock 替换一次请求结果

```python
from unittest.mock import Mock
import requests

response = requests.Response()
response.status_code = 200
response.headers["Content-Type"] = "application/json"
response._content = b'{"message":"hello"}'

session = Mock(spec=requests.Session)
session.get.return_value = response

result = session.get("http://example.invalid/hello", timeout=(3, 5))
result.raise_for_status()
assert result.json() == {"message": "hello"}

session.get.assert_called_once_with(
    "http://example.invalid/hello",
    timeout=(3, 5),
)
```

这里 `_content` 是为了构造受控测试响应使用的内部属性，不建议在普通业务代码里修改真实响应内部状态。

### 25.3 Mock 成功，是否说明网络一定通？

没有。

Mock 验证的是你如何组织调用，以及收到指定结果时业务如何反应。

真实 DNS、TLS、代理和服务状态需要不同层面的集成验证。

把这两种测试的结论分清楚，才能知道失败应该去哪里排查。

### 25.4 本地 HTTP 服务可以验证什么？

本篇练习服务器能让你检查真实请求方法、查询编码、请求体、Cookie、重定向、超时和下载流程。

它不依赖外部公共站点，结果比较容易控制。

但它也没有模拟真实服务的全部身份认证、限流、分布式一致性和故障模式。

测试应该与它真正覆盖的行为对应，而不是得到一个 200 就宣称所有网络环境都已验证。

## 第二十六章：流式文本怎样一行一行处理？

前面用 `iter_content()` 处理任意字节块。

如果服务返回的是按行组织的文本，应用通常更关心一条完整行，而不是这一块恰好有多少字节。

这时可以使用 `iter_lines()`。

### 26.1 先读取本地普通文本

```python
import requests

with requests.get(
    "http://127.0.0.1:8765/text",
    stream=True,
    timeout=(3, 5),
) as response:
    response.raise_for_status()
    response.encoding = "utf-8"

    for line in response.iter_lines(decode_unicode=True):
        print(repr(line))
```

对于本篇练习服务，通常得到：

```text
'这是一份普通文本。'
```

返回的行已经去掉行分隔符，所以不再包含原来的末尾换行。

### 26.2 `decode_unicode=True` 是什么意思？

它请求将内容解码为文本行。

文本解码需要编码依据，例子明确约定服务内容是 UTF-8，所以设置 `response.encoding`。

如果编码资料缺失或设置不符合实际，不应假设这个参数会神奇识别所有编码。

需要原始字节行时，可以使用默认二进制行处理，再按协议自己解码。

### 26.3 网络每次送来的块，不一定刚好是一行

假设服务想发送：

```text
first message\n
second message\n
```

实际网络读取可能先拿到：

```text
first mes
```

再拿到：

```text
sage\nsecond
```

最后才是：

```text
 message\n
```

这些块边界由网络和缓冲决定，不是应用消息边界。

`iter_lines()` 会为常见按行消费组织相应缓冲，让你不用把每个 `iter_content()` 块都误当成一条完整记录。

### 26.4 `chunk_size` 小一点，消息就一定立刻显示吗？

客户端分块设置只是整个链路的一部分。

服务器可能没有刷新输出，反向代理可能缓冲，压缩层也可能推迟交付。

所以看到延迟时，需要检查发送方、代理和接收方的完整过程。

不能仅凭客户端把块大小改成一，就承诺所有上游事件实时到达。

### 26.5 可以同时创建两个 `iter_lines()` 交替读吗？

不应这样组织同一个响应流。

流具有消费位置和内部缓冲，多个消费者交替读取可能相互影响。

如果两段业务都需要同一条记录，通常由一个读取循环取得记录，再分发给两个处理函数。

如果要保存后反复分析，可以明确写入受控缓存或文件，但那又需要考虑存储规模。

### 26.6 每行看起来是 JSON，能否直接 `.json()`？

`response.json()` 预期处理整个响应体中的一个 JSON 文档。

JSON Lines 则通常每行是一个独立 JSON 值，应逐行调用 `json.loads()`。

概念上的流程是：

```text
响应字节流
    ↓ 解码与行分隔
一条文本记录
    ↓ json.loads
Python 对象
    ↓ 结构与业务检查
应用处理
```

具体服务还可能使用 SSE 等事件协议，包含 `data:`、事件名称、空行分隔、多行数据等规则。

此时不能把每一条原始文本行都直接当 JSON。

先确认事件协议，再选择或实现符合协议的解析器。

### 26.7 一直不结束的事件流，什么时候关闭？

有些接口会持续发送事件，不打算很快到 EOF。

你的程序需要自己的停止条件，例如接收指定条数、用户取消、任务期限或应用退出。

离开上下文时关闭响应，才能释放连接。

读取超时主要限制等待数据间隔，也不等于这条事件订阅的总生命期限。

### 26.8 如果处理某一行特别慢呢？

同步循环里，处理函数不结束，下一次读取就不会继续进行。

这可能形成自然背压，也可能让服务端积压、连接超时或客户端落后。

是否把读取与处理分开，需要看事件量、可接受延迟和内存边界。

如果使用队列，队列应有明确容量和溢出政策，不要为了“异步处理”无限保存所有未处理事件。


## 第二十七章：常见排错情境，按现象一步一步判断

### 27.1 “浏览器能访问，Python 为什么连不上？”

先确认浏览器和 Python 访问的是不是同一个地址。

浏览器可能自动补 HTTPS、经过登录跳转、使用自己的代理或证书配置。

Python 可能运行在 WSL，继承的环境变量与 Windows 浏览器不同。

所以，第一步应该记录当前解释器、完整目标地址和异常类型。

```python
import sys

print("Python：", sys.executable)
print("平台：", sys.platform)
```

然后检查服务器监听位置、DNS、证书和当前网络配置。

不要直接把所有连接问题都归因于库版本，也不要未经确认就叠加代理。

### 27.2 “返回 200，为什么 `.json()` 报错？”

先检查媒体类型和内容开头，但注意只显示安全且有限的片段。

```python
print(response.status_code)
print(response.headers.get("Content-Type"))
print(repr(response.text[:200]))
```

可能拿到的是 HTML 登录页、空页面或普通文本。

200 只说明 HTTP 层按服务的规则返回成功状态，不保证内容格式是你期待的 JSON。

如果正文含敏感资料，不应把整个响应贴到公共聊天或日志中。

### 27.3 “明明设置了 `json=...`，服务器说字段不存在？”

先确认服务是否真的接受 JSON。

有些端点要求表单字段，有些要求 multipart 文件，有些把参数放在 URL 查询中。

再确认自己有没有把已经序列化的字符串又传给 `json=`。

可以用准备对象观察正文，并在本地 `/echo` 对照请求格式。

### 27.4 “Session 设置了头，为什么某一次没有出现？”

检查是否真的通过同一个会话发送。

下面两个对象不是同一个：

```python
first = requests.Session()
second = requests.Session()
```

给 `first.headers` 设置值，不会自动修改 `second`。

也要检查方法级参数是否覆盖、重定向是否改变目标，以及你是不是只调用了 `Request.prepare()` 而没有通过会话准备。

### 27.5 “下载速度很慢，换更大的 chunk 就一定好吗？”

分块大小只影响客户端处理缓冲的一个方面。

真实速度还取决于服务端限速、网络吞吐、延迟、压缩、磁盘写入和并发数量。

把 64 KiB 改成 64 MiB，可能只增加内存占用，没有改善瓶颈。

先测量完整过程，再确认瓶颈在哪里。

### 27.6 “超时后重发，为什么服务器多了一条记录？”

客户端可能在等待响应时超时，而服务端已经完成创建。

第二次请求又创建一条，这就是典型的副作用重试问题。

需要服务端提供幂等键、请求标识或状态查询，客户端按契约恢复。

仅仅把超时从五秒改成五十秒，可能降低发生概率，却没有建立正确的一次性业务语义。

### 27.7 “我只读取一部分响应，后面请求怎么越来越卡？”

检查流式响应是否关闭。

如果创建许多 `stream=True` 响应，却不读完、不调用 `close()`，资源可能持续被占用。

优先把流式响应放进明确的 `with`。

对连接池问题，也应区分“等待空闲连接”“服务器响应慢”和“DNS/TLS 建立慢”，不要把所有等待都叫网络卡。

### 27.8 “为什么本机端口是开着的，WSL 仍然访问不到？”

先确定服务在哪个系统启动。

服务监听 `127.0.0.1` 还是其他地址，客户端所在网络命名空间是否能到达，都很关键。

Windows 防火墙和 WSL 网络模式也可能影响结果。

本篇推荐服务端和客户端都先放在同一个 WSL 环境练习，减少变量；等 HTTP 基础清楚后，再研究跨系统连接。

### 27.9 “为什么开了 `verify=False` 就好了？”

它可能绕过了原本失败的服务器身份验证步骤。

这说明问题与信任验证相关，不能据此证明远端身份正确。

应回到证书链、地址、系统时间和可信 CA 配置检查，而不是把绕过验证当作最终修复。

### 27.10 “错误对象里明明有 response，`if error.response` 为什么没进去？”

因为 Response 的布尔规则与错误状态有关。

应写：

```python
if error.response is not None:
    print(error.response.status_code)
```

这区分对象存在与否，不会把 404 响应误当成没有响应。

这个问题看起来很小，却会让错误处理丢失最重要的状态信息。

## 第二十八章：断点续传与缓存，为什么不能只加一个请求头？

这两类需求很常见，但需要服务端与客户端一起遵守协议。

### 28.1 Range 请求的意图

例如：

```text
Range: bytes=1000-
```

表达希望从某个字节位置继续取得内容。

服务器可能支持并返回 206，也可能忽略 Range 并返回完整的 200。

因此，客户端不能无条件以追加模式把响应接在本地旧文件后面。

如果服务返回完整文件，你再追加，就得到“旧前缀加完整新文件”的损坏结果。

### 28.2 为什么还要检查资源版本？

假设你昨天下载了前一半，今天服务器上的文件已经更新。

即使继续从相同偏移读取，前半部分和后半部分也可能来自不同版本。

需要结合 ETag、Last-Modified、If-Range 等协议字段及服务契约确认一致性。

断点续传不是单纯的本地 `seek()`，它涉及远端资源身份和字节范围。

### 28.3 本地练习服务是否支持 Range？

本篇的简化 `/download` 没有实现完整范围请求处理。

它用于练习普通下载。

如果你给它加 Range 头，不应假设它就会按范围返回。

这也是学习网络库时必须保持的判断：客户端能发送一个头，不代表服务器已经实现对应功能。

### 28.4 缓存请求中的 ETag

服务器可以给资源一个实体标识，客户端后续带条件请求头，询问资源是否变化。

若返回 304，通常没有新的完整响应体，客户端需要使用自己已经保存且正确对应的缓存内容。

Requests 不会替你自动建立一个完整持久化 HTTP 缓存数据库。

你需要管理缓存键、版本、有效期和存储策略。

### 28.5 为什么 URL 相同也可能不能共用缓存？

响应可能受认证身份、请求头、语言、压缩和服务端 `Vary` 等规则影响。

如果只按 URL 保存某个用户的私人响应，再交给另一个用户，就会发生数据泄露或错误结果。

个人学习脚本可以先掌握普通无缓存请求；需要缓存时，按标准和服务契约引入经过验证的方案。

## 第二十九章：带参考答案的动手练习

练习不要求你第一次就写出完整答案。

建议先预测结果，再运行，最后比较“预期”和“实际”哪里不同。

### 29.1 练习一：检查查询参数编码

要求：发送关键字 `Python 文件` 和两个 `tag`，服务器应分别收到原字符串和两个标签。

```python
import requests

params = [
    ("keyword", "Python 文件"),
    ("tag", "python"),
    ("tag", "backend"),
]

response = requests.get(
    "http://127.0.0.1:8765/echo",
    params=params,
    timeout=(3, 5),
)
response.raise_for_status()
query = response.json()["query"]

assert query["keyword"] == ["Python 文件"]
assert query["tag"] == ["python", "backend"]
print("参数编码与多值参数验证通过")
```

不要直接断言完整 URL 的所有转义表现与某个手写字符串完全相同，因为不同等价编码形式和参数顺序可能不影响协议含义。

这里验证的是服务器实际解析到的值。

### 29.2 练习二：区分 JSON 和表单

要求：向同一个 `/echo` 分别发送两种请求，观察内容类型与解析结果。

```python
import requests

url = "http://127.0.0.1:8765/echo"

json_response = requests.post(url, json={"count": 3}, timeout=(3, 5))
form_response = requests.post(url, data={"count": 3}, timeout=(3, 5))

json_response.raise_for_status()
form_response.raise_for_status()

print(json_response.json()["data"])
print(form_response.json()["data"])
```

本地服务通常分别得到：

```text
{'count': 3}
{'count': ['3']}
```

JSON 保留数值类型，表单解析得到字符串列表。

这不是谁更正确，而是两种格式具有不同表示规则。

### 29.3 练习三：404 也是真实响应

```python
import requests

response = requests.get(
    "http://127.0.0.1:8765/status/404",
    timeout=(3, 5),
)

print(response is not None)
print(bool(response))
print(response.status_code)
```

通常输出：

```text
True
False
404
```

第一项检查对象是否存在。

第二项使用 Requests 的响应布尔规则。

第三项显示真实状态。

请自己解释：为什么错误处理里判断是否有 response，应优先使用 `is not None`。

### 29.4 练习四：显式拒绝错误状态

```python
import requests

try:
    response = requests.get(
        "http://127.0.0.1:8765/status/503",
        timeout=(3, 5),
    )
    response.raise_for_status()
except requests.exceptions.HTTPError as error:
    assert error.response is not None
    print("服务返回：", error.response.status_code)
```

这次请求并不是连接失败。

客户端确实接收到一个 HTTP 响应，只是状态表示服务未能正常处理。

这一区别会影响日志分类和重试政策。

### 29.5 练习五：让某一次覆盖默认头

```python
import requests

with requests.Session() as session:
    session.headers["X-Demo"] = "default"

    first = session.get(
        "http://127.0.0.1:8765/echo",
        headers={"X-Demo": "temporary"},
        timeout=(3, 5),
    )
    second = session.get(
        "http://127.0.0.1:8765/echo",
        timeout=(3, 5),
    )

    first.raise_for_status()
    second.raise_for_status()
    assert first.json()["x_demo"] == "temporary"
    assert second.json()["x_demo"] == "default"
```

这个练习验证“本次参数覆盖”和“永久修改会话默认值”不同。

### 29.6 练习六：不跟随重定向

```python
import requests

response = requests.get(
    "http://127.0.0.1:8765/redirect",
    allow_redirects=False,
    timeout=(3, 5),
)

assert response.status_code == 302
assert response.headers["Location"] == "/hello"
assert response.history == []
```

这里没有真正继续请求 `/hello`。

`history` 是已经发生的自动重定向响应链，不是“服务器给了 Location 就自动包含一项”。

### 29.7 练习七：观察读取超时

```python
import requests

try:
    requests.get(
        "http://127.0.0.1:8765/delay",
        timeout=(1, 0.1),
    )
except requests.exceptions.ReadTimeout:
    print("按预期触发读取超时")
else:
    raise AssertionError("本地延迟接口没有按预期触发超时")
```

服务器会在客户端放弃后继续结束自己的处理，练习服务已处理常见断开连接异常。

超时不会向服务器发送一个具有业务回滚含义的神奇命令。

### 29.8 练习八：先设置 Cookie，再验证发送

```python
import requests

with requests.Session() as session:
    first = session.get("http://127.0.0.1:8765/cookie", timeout=(3, 5))
    first.raise_for_status()

    second = session.get("http://127.0.0.1:8765/echo", timeout=(3, 5))
    second.raise_for_status()

    assert "demo=learn" in second.json()["cookie"]
    print("会话 Cookie 已按练习范围发送")
```

然后把第二次请求改成独立的 `requests.get()`，比较结果，并解释为什么状态不一定自动保持。

### 29.9 练习九：下载超限时，旧文件应该保留

使用前面定义的 `download_file()`。

先准备一份旧正式文件，再把上限设得很小：

```python
from pathlib import Path
import requests

path = Path("requests_practice/limited.bin")
path.parent.mkdir(parents=True, exist_ok=True)
path.write_bytes(b"old-result")

with requests.Session() as session:
    try:
        download_file(
            session,
            "http://127.0.0.1:8765/download",
            path,
            max_bytes=100,
        )
    except ValueError:
        pass
    else:
        raise AssertionError("应该拒绝超限内容")

assert path.read_bytes() == b"old-result"
```

这个测试验证失败时的正式结果边界，比只验证“成功时文件存在”更有意义。

也可以检查目录中是否遗留本次 `.part`，确认清理过程。

### 29.10 练习十：分页必须取得全部五条

使用前面的 `iter_items()`：

```python
import requests

with requests.Session() as session:
    items = list(iter_items(session, "http://127.0.0.1:8765", page_size=2))

assert [item["id"] for item in items] == [1, 2, 3, 4, 5]
```

然后把 `max_pages` 改成一，确认程序报告未完成，而不是把第一页当成完整数据。

### 29.11 练习十一：准备请求不能自动继承任意会话

```python
import requests

with requests.Session() as session:
    session.headers["X-Demo"] = "from-session"
    request = requests.Request("GET", "http://example.invalid/hello")

    plain = request.prepare()
    merged = session.prepare_request(request)

    assert "X-Demo" not in plain.headers
    assert merged.headers["X-Demo"] == "from-session"
```

这段代码完全不访问网络。

它验证的是准备流程和会话状态的关系。

### 29.12 练习十二：客户端封装保留原异常原因

使用前面的 `DemoClient`。

请求一个不存在的端点：

```python
try:
    with DemoClient("http://127.0.0.1:8765") as client:
        client.request_json("GET", "/not-found")
except ApiError as error:
    print(type(error.__cause__).__name__)
```

预期原因属于 Requests 的 HTTP 错误，而不只是一个失去上下文的字符串。

这里观察的是异常实例的 `__cause__`，也就是通过 `raise ... from ...` 保留的直接原因。

### 29.13 练习十三：空响应不要强行解析

```python
import requests

response = requests.get("http://127.0.0.1:8765/empty", timeout=(3, 5))
response.raise_for_status()

assert response.status_code == 204
assert response.content == b""
```

再有意调用 `response.json()`，观察解析错误。

然后解释：为什么 204 可以属于成功状态，却没有一个可供解析的 JSON 对象。

### 29.14 练习十四：原始上传大小与 multipart 大小不同

先创建一个短文件。

分别用 `data=source` 和 `files={...}` 上传到 `/upload`。

原始请求体通常就是文件字节长度；multipart 请求体还包含边界、字段头等结构，因此 `received_bytes` 会更大。

不要把服务端收到的整个 multipart 容器大小误认为文件内容本身大小。

### 29.15 练习十五：构造一次失败测试，而不关闭真实网络

使用 Mock，让 `session.get()` 抛出 `requests.exceptions.Timeout`。

验证你的业务函数是否把它转为恰当错误、是否保留原因，以及是否遵守有限重试次数。

不要为了制造失败去修改系统路由、关闭用户 TUN 或清空全局代理配置。

可重复的测试应该尽量把影响限制在测试对象范围内。

## 第三十章：选接口时，先把问题问清楚

你要取得一份小型 JSON 响应？

先考虑普通 GET、明确超时、状态检查、JSON 解析和结构校验。

你要连续访问同一个服务？

考虑 Session，把生命周期、默认参数和状态管理集中起来。

你要传字段？

先问服务要求查询参数、JSON、表单还是 multipart，再选择 `params`、`json`、`data` 或 `files`。

你要下载大型文件？

考虑流式响应、累计大小上限、临时文件和完整发布，不要先整体读取 `.content`。

你要重试？

先确认请求能否重复执行，再决定哪些错误值得重试、等多久、最多几次，以及总任务期限是什么。

你要解决 HTTPS 失败？

检查信任与地址配置，保留正确身份验证，而不是长期绕过。

你要排查 WSL 网络？

先确认服务位置、监听地址和现有系统/TUN与代理状态，再改变应用配置。

这些选择来自具体需求。Requests 提供实现这些选择的接口，但不会替应用决定业务政策。

## 第三十一章：常用接口速查

| 需求 | 常见入口 | 需要一起理解的条件 |
| --- | --- | --- |
| GET 读取 | `requests.get(url, ...)` | 超时、状态和格式 |
| POST 提交 | `requests.post(url, ...)` | 请求体格式与副作用 |
| 通用方法 | `requests.request(method, url, ...)` | 方法契约 |
| 查询参数 | `params=...` | URL 编码与重复参数 |
| JSON 请求体 | `json=python_object` | 避免双重编码 |
| 表单字段 | `data=dict` | 服务端表单解析 |
| 原始内容 | `data=bytes_or_stream` | 内容类型与长度 |
| 文件表单 | `files=...` | multipart边界和内存策略 |
| 自定义头 | `headers=...` | 秘密字段与覆盖规则 |
| 连接/读取超时 | `timeout=(connect, read)` | 不等于总时限 |
| 状态检查 | `raise_for_status()` | 4xx/5xx与业务码区别 |
| 文本响应 | `response.text` | 编码依据 |
| 二进制内容 | `response.content` | 整体内存与内容解压 |
| JSON 解析 | `response.json()` | 解析失败和结构校验 |
| 流式内容 | `stream=True`、`iter_content()` | 读完或关闭 |
| 逐行内容 | `iter_lines()` | 行边界、编码、单次消费 |
| 会话复用 | `requests.Session()` | 状态与连接生命周期 |
| Cookie | `session.cookies` | 域、路径和安全属性 |
| Basic Auth | `auth=(user, password)` | 真实远程应结合HTTPS |
| 禁止自动跳转 | `allow_redirects=False` | Location和目标授权 |
| 服务器证书 | `verify=True`或可信CA路径 | 不应常规关闭验证 |
| 客户端证书 | `cert=...` | 与服务器验证方向不同 |
| 受控重试 | `HTTPAdapter`与`Retry` | 幂等、预算、退避 |
| 准备请求 | `Request`、`prepare_request()` | 编码与会话合并 |

建议学习顺序是：先跑通本地 GET，再依次观察参数、响应、请求体、状态与超时，随后学习会话和文件传输，最后再设计封装、重试与更复杂协议。

遇到错误时，先判断它发生在地址与连接、HTTP状态、内容解码、JSON结构，还是业务规则层。定位层次清楚，通常就能避免在不相关的参数上反复试错。

官方参考：[Requests 快速入门](https://requests.readthedocs.io/en/latest/user/quickstart/)、[高级用法](https://requests.readthedocs.io/en/latest/user/advanced/)、[接口参考](https://requests.readthedocs.io/en/latest/api/)、[urllib3 重试配置](https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html#urllib3.util.Retry)。本篇本地服务与业务客户端为教学编写，不能替代真实服务的协议文档和生产设计。


## 附录 A：配套完整代码（按小块阅读；第一次阅读可跳过）

**第一块：** 导入库与准备样例数据；**第二块：** 怎样构造响应；**第三块：** 根据 URL 路径决定返回什么；**第四块：** 处理参数与错误状态；**第五块：** 启动服务/配置模拟客户端。先看每块的输入和输出，不必一口气记住所有代码。

将下方代码复制保存为 `requests_demo_server.py`：

```python
import base64
import json
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import parse_qs, urlsplit


ITEMS = [
    {"id": 1, "name": "Python 文件处理"},
    {"id": 2, "name": "路径对象"},
    {"id": 3, "name": "HTTP 请求"},
    {"id": 4, "name": "JSON 数据"},
    {"id": 5, "name": "日志记录"},
]


class DemoHandler(BaseHTTPRequestHandler):
    def send_bytes(self, status, body, content_type, extra_headers=None):
        self.send_response(status)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(body)))
        for name, value in (extra_headers or {}).items():
            self.send_header(name, value)
        self.end_headers()
        if self.command != "HEAD":
            try:
                self.wfile.write(body)
            except (BrokenPipeError, ConnectionResetError):
                pass

    def send_json(self, status, data, extra_headers=None):
        body = json.dumps(data, ensure_ascii=False, allow_nan=False).encode("utf-8")
        self.send_bytes(status, body, "application/json; charset=utf-8", extra_headers)

    def do_GET(self):
        parsed = urlsplit(self.path)
        query = parse_qs(parsed.query, keep_blank_values=True)

        if parsed.path == "/hello":
            self.send_json(200, {"message": "你好，Requests"})
        elif parsed.path == "/echo":
            self.send_json(200, {
                "method": "GET",
                "query": query,
                "user_agent": self.headers.get("User-Agent"),
                "x_demo": self.headers.get("X-Demo"),
                "cookie": self.headers.get("Cookie"),
            })
        elif parsed.path == "/items":
            try:
                page = int(query.get("page", ["1"])[0])
                size = int(query.get("page_size", ["2"])[0])
                if page < 1 or not 1 <= size <= 100:
                    raise ValueError
            except ValueError:
                self.send_json(400, {"error": "分页参数不合法"})
                return
            start = (page - 1) * size
            self.send_json(200, {
                "items": ITEMS[start:start + size],
                "page": page,
                "page_size": size,
                "total": len(ITEMS),
            })
        elif parsed.path.startswith("/status/"):
            try:
                status = int(parsed.path.rsplit("/", 1)[1])
                if status not in {200, 400, 401, 403, 404, 429, 500, 503}:
                    raise ValueError
            except ValueError:
                self.send_json(400, {"error": "不支持的练习状态码"})
                return
            self.send_json(status, {"status": status})
        elif parsed.path == "/download":
            body = bytes(range(256)) * 1024
            self.send_bytes(200, body, "application/octet-stream")
        elif parsed.path == "/delay":
            time.sleep(2)
            self.send_json(200, {"message": "稍后返回"})
        elif parsed.path == "/redirect":
            self.send_bytes(302, b"", "text/plain", {"Location": "/hello"})
        elif parsed.path == "/cookie":
            self.send_json(200, {"message": "已设置练习 Cookie"}, {
                "Set-Cookie": "demo=learn; Path=/; HttpOnly",
            })
        elif parsed.path == "/auth":
            expected = "Basic " + base64.b64encode(b"demo:practice").decode("ascii")
            if self.headers.get("Authorization") != expected:
                self.send_json(401, {"error": "需要练习认证"}, {
                    "WWW-Authenticate": 'Basic realm="demo"',
                })
            else:
                self.send_json(200, {"user": "demo"})
        elif parsed.path == "/empty":
            self.send_bytes(204, b"", "application/json")
        elif parsed.path == "/text":
            self.send_bytes(200, "这是一份普通文本。\n".encode("utf-8"), "text/plain; charset=utf-8")
        else:
            self.send_json(404, {"error": "没有这个练习接口"})

    def do_HEAD(self):
        self.do_GET()

    def do_POST(self):
        length = int(self.headers.get("Content-Length", "0"))
        if not 0 <= length <= 2_000_000:
            self.send_json(413, {"error": "练习请求体过大"})
            return

        body = self.rfile.read(length)
        content_type = self.headers.get("Content-Type", "")
        parsed = urlsplit(self.path)

        if parsed.path == "/upload":
            self.send_json(200, {
                "received_bytes": len(body),
                "content_type": content_type,
            })
            return

        if parsed.path != "/echo":
            self.send_json(404, {"error": "没有这个练习接口"})
            return

        if content_type.startswith("application/json"):
            try:
                data = json.loads(body)
            except (json.JSONDecodeError, UnicodeDecodeError):
                self.send_json(400, {"error": "JSON 解析失败"})
                return
        elif content_type.startswith("application/x-www-form-urlencoded"):
            data = parse_qs(body.decode("utf-8"), keep_blank_values=True)
        else:
            data = {"raw_size": len(body)}

        self.send_json(200, {
            "method": "POST",
            "content_type": content_type,
            "data": data,
        })


if __name__ == "__main__":
    server = ThreadingHTTPServer(("127.0.0.1", 8765), DemoHandler)
    print("练习服务：http://127.0.0.1:8765")
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass
    finally:
        server.server_close()
```
