我觉得你这次的学习思路是对的，而且我非常赞同你提出的一个观点：

**学习 CMake，不能一上来就背 `add_executable()`、`target_link_libraries()`、`include_directories()` 这些函数。你必须先理解：为什么我们需要 CMake？它究竟想解决什么问题？它和编译器、链接器、Makefile、IDE 分别是什么关系？**

否则，你看到的就只是一堆互不相关的命令。今天照着教程写一个 `CMakeLists.txt`，明天项目多了几个文件，又不知道该怎么改；后天需要链接第三方库，就开始到处搜索复制代码。

我打算把这件事从底层逻辑讲起。

你目前已经知道：

- C、C++ 源文件（`.c`、`.cpp`）和头文件（`.h`、`.hpp`）的基本关系。
- 编译、链接、运行时之间的大致关系。
- 可以使用 `g++` 编译单文件，也可以手动编译几个简单的源文件。
- 对 C/C++ 的构建过程已经有了一定认识。

所以我不会重新给你讲一遍 C++ 编译入门，而是以你现有的知识为起点，把它们串成一个完整的体系。

# 第一部分：先建立正确的认知——CMake 究竟是什么？

## 1. 一个问题：为什么我们需要 CMake？

我们先不着急讲 CMake。

假设你现在写了一个非常简单的 C++ 项目。

项目有三个文件：

```text
MyProject/
├── main.cpp
├── math.cpp
└── math.h
```

其中：

`math.h`：

```cpp
#pragma once

int add(int a, int b);
```

`math.cpp`：

```cpp
#include "math.h"

int add(int a, int b) {
    return a + b;
}
```

`main.cpp`：

```cpp
#include <iostream>
#include "math.h"

int main() {
    std::cout << add(1, 2) << '\n';
    return 0;
}
```

你已经知道，可以直接这样编译：

```bash
g++ main.cpp math.cpp -o my_program
```

这没有任何问题。

那么，我们增加一点复杂度。

假设你的项目逐渐发展成这样：

```text
MyProject/
├── src/
│   ├── main.cpp
│   ├── math.cpp
│   ├── network.cpp
│   ├── database.cpp
│   ├── logger.cpp
│   └── ...
│
├── include/
│   ├── math.h
│   ├── network.h
│   ├── database.h
│   ├── logger.h
│   └── ...
│
├── third_party/
│   ├── some_library/
│   └── another_library/
│
└── ...
```

现在，你可能需要：

1. 编译十几个甚至几百个 `.cpp` 文件。
2. 指定头文件的搜索路径。
3. 指定 C++ 标准，例如 C++17、C++20。
4. 指定编译选项，例如 `-O2`、`-Wall`、`-Wextra`。
5. 链接第三方库，例如 OpenSSL、pthread 等。
6. 区分 Debug 和 Release 两种构建模式。
7. 在 Linux、Windows、macOS 上分别构建。
8. 把部分代码编译成一个独立的库，供其他程序使用。
9. 管理多个子项目之间的依赖关系。

这时候，你会发现：

**真正麻烦的事情已经不再是编写 C++ 代码，而是管理如何把这些 C++ 代码正确地编译、链接起来。**

你当然可以继续手写 g++ 命令。

例如：

```bash
g++ -std=c++20 -O2 -Wall \
    -Iinclude \
    src/main.cpp \
    src/math.cpp \
    src/network.cpp \
    src/database.cpp \
    src/logger.cpp \
    -lssl -lcrypto \
    -pthread \
    -o my_program
```

但如果项目有 100 个源文件、10 个第三方库、多个子模块呢？

你需要维护的命令就会变得越来越复杂。

更重要的是：如果只修改了 `math.cpp`，你通常不希望把 100 个源文件全部重新编译一遍。

你希望构建系统能够知道：

- 哪些文件发生了变化？
- 哪些目标需要重新编译？
- 哪些文件依赖于哪些头文件？
- 哪些目标依赖于哪些库？
- 应该按照什么顺序进行构建？
- 最终需要生成什么？

这就是构建系统（Build System）要解决的问题。

## 2. 什么是构建系统（Build System）？

先记住一个定义：

**构建系统负责描述、组织和执行一个软件项目从源代码到最终产物的构建过程。**

这里的构建（Build）不只是编译。

通常包含：

- 编译（Compile）：将 `.cpp` 编译成目标文件 `.o`。
- 链接（Link）：将目标文件和库组合成可执行文件或其他库。
- 生成（Generate）：例如生成代码、配置文件、资源文件。
- 依赖管理（Dependency Management）：确定哪些东西依赖哪些东西。
- 增量构建（Incremental Build）：尽可能只重新构建发生变化的部分。

我们可以把它理解成一个工程调度系统。

编译器负责具体的编译工作，链接器负责链接工作，而构建系统负责组织这些工作。

举个形象的例子：

假设你要建一栋房子。

| 角色 | 类比 |
|---|---|
| `.cpp` 文件 | 建筑材料 |
| 编译器 | 将原材料加工成构件的机器 |
| 链接器 | 将构件组装起来的设备 |
| 构建系统 | 负责安排整个施工流程的工程调度系统 |
| IDE | 你与整个工程系统交互的工作台 |

你不应该让编译器承担整个工程的调度工作。

同样，你也不应该让 IDE 承担整个项目的构建逻辑。

这就是我们需要构建系统的原因。

## 3. Makefile、Make、CMake，究竟是什么关系？

这里是第一个非常容易混淆的地方。

先把三个名字分清楚。

### 3.1 Make

**Make 是一个构建工具（Build Tool）。**

它负责读取 Makefile 中描述的规则，然后按照依赖关系执行相应的命令。

例如：

```makefile
main.o: main.cpp math.h
	g++ -c main.cpp -o main.o

math.o: math.cpp math.h
	g++ -c math.cpp -o math.o

my_program: main.o math.o
	g++ main.o math.o -o my_program
```

这就是一个简单的 Makefile。

你可以在项目目录下执行：

```bash
make
```

Make 就会读取 Makefile，分析目标之间的依赖关系，决定需要执行哪些命令。

注意，这里有一个重要概念：

**Make 并不知道什么是 C++，也不真正理解什么是 `.cpp`、什么是 `.o`。**

它主要关心的是：

- 目标（Target）。
- 依赖（Prerequisite）。
- 命令（Recipe）。

例如：

```makefile
my_program: main.o math.o
	g++ main.o math.o -o my_program
```

这句话对 Make 来说，核心意思是：

> 我要生成 my_program，它依赖 main.o 和 math.o。如果依赖发生变化，就执行后面的命令重新生成它。

Make 只是一个通用的构建工具。

你甚至可以用它处理图片转换、文档生成等工作，不一定非得编译 C++。

### 3.2 Makefile

Makefile 是 Make 使用的构建规则文件。

你可以把它理解成：

- Make：执行者。
- Makefile：告诉 Make 应该怎么做的规则说明书。

它们不是同一个东西。

### 3.3 CMake

**核心概念**

CMake 是一个跨平台的构建系统生成器（Build System Generator）。

这句话非常重要。

**CMake 本身通常不负责真正执行 C++ 编译和链接。它主要负责根据你的项目描述，生成底层构建工具能够理解的构建文件。**

例如：

```text
                 你的 C/C++ 项目
                        │
                        ▼
                  CMakeLists.txt
                        │
                        ▼
                     CMake
                        │
             配置项目、分析依赖关系
                        │
                        ▼
                  生成构建系统
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          Makefile    build.ninja  Visual Studio
             │          │          │
             ▼          ▼          ▼
           Make        Ninja      MSBuild
             │          │          │
             └──────────┼──────────┘
                        ▼
                    编译器
                        │
                        ▼
                     链接器
                        │
                        ▼
                   最终程序
```

这就是 CMake 最核心的设计思路。

你只需要用 CMake 描述项目，而不需要自己分别为 Make、Ninja、Visual Studio 等工具编写不同的构建规则。

例如：

你可以使用同一份 `CMakeLists.txt`：

- 在 Linux 上生成 Makefile。
- 在 Linux 上生成 Ninja 构建文件。
- 在 Windows 上生成 Visual Studio 项目。
- 在 macOS 上生成 Xcode 项目。

具体支持哪些生成器，取决于 CMake 和对应平台的环境。

这就是 CMake 的跨平台价值。

---

## 4. CMake 为什么会诞生？它解决了什么历史问题？

现在我们知道 CMake 是什么了，但还可以进一步理解：为什么当年有人要专门开发这么一个工具？

这其实涉及 C/C++ 开发中一个很重要的问题：**不同操作系统、不同编译器、不同开发环境之间的构建方式不统一。**

假设你写了一个 C++ 项目，希望同时支持：

- Linux + GCC
- Linux + Clang
- Windows + MSVC
- macOS + Clang

在 CMake 出现之前，项目经常需要维护多套构建配置。

例如：

```text
MyProject/
├── Makefile
├── configure
├── MyProject.sln
├── MyProject.vcxproj
├── XcodeProject.xcodeproj
└── src/
```

你在 Linux 上可能使用 Makefile，在 Windows 上使用 Visual Studio 工程文件，在 macOS 上使用 Xcode 工程文件。

问题来了。

假设你今天新增了一个 `network.cpp`。

你可能需要：

1. 把它添加到 Makefile。
2. 把它添加到 Visual Studio 工程。
3. 把它添加到 Xcode 工程。
4. 修改其他相关构建配置。

如果忘记修改其中一份配置，就可能出现：

- Linux 能编译，Windows 不能编译。
- GCC 能编译，MSVC 不能编译。
- 某个平台遗漏了一个源文件。
- 不同平台使用了不同的依赖版本。

这显然很不方便。

CMake 的诞生就是为了解决这类问题。

CMake 最初由 Kitware 开发，2000 年正式发布，起因是美国国家医学图书馆资助的 ITK（医学图像处理工具包）项目需要一个跨 Unix、Windows 和 macOS 的构建环境。（[CMake 官方历史介绍](https://cmake.org/history/)；[Happy Birthday CMake!](https://www.kitware.com/happy-birthday-cmake/)）

当时开发者提出了几个很有意思的目标：

- 用一份构建描述支持多个平台。
- 尽量使用平台原生的构建工具。
- 不要为了构建项目而引入大量额外依赖。
- 能够支持大型、复杂的 C++ 项目。

你可以把它理解成：

**与其给每个平台分别写一份构建说明书，不如写一份通用说明书，再让一个工具将它翻译成各个平台能够执行的形式。**

这就是 CMake 的核心价值。

顺便说一句，CMake 中的 C 是 Cross Platform（跨平台）的意思，并不是说它只用于 C 语言。它当然也支持 C++。

---

# 第二部分：CMake 的设计哲学与设计范式

这部分是整篇内容最重要的部分。

我希望你在看完之后，能够真正理解为什么现代 CMake 喜欢使用 `target_xxx()`，为什么有 `PUBLIC`、`PRIVATE`、`INTERFACE`，为什么不建议到处使用全局变量，以及为什么一个项目会有很多个 `CMakeLists.txt`。

## 5. 设计哲学一：描述项目，而不是手写编译命令

先对比两种思路。

### 思路 A：命令式地告诉编译器应该怎么做

例如：

```bash
g++ -c main.cpp -o main.o
g++ -c math.cpp -o math.o
g++ main.o math.o -o app
```

这里你告诉工具：

1. 执行第一条命令。
2. 执行第二条命令。
3. 执行第三条命令。

你关注的是具体的操作步骤。

这叫偏向命令式（Imperative）的构建方式。

### 思路 B：描述项目里有什么，以及它们之间是什么关系

例如：

```cmake
add_executable(app
    main.cpp
    math.cpp
)
```

你没有告诉 CMake：

- 应该先执行哪个 g++ 命令。
- 应该生成什么 `.o` 文件。
- 最终链接命令应该怎么写。
- 应该使用 Make 还是 Ninja。

你只是告诉它：

> 我有一个叫 app 的可执行目标，它由 main.cpp 和 math.cpp 构成。

剩下的事情交给 CMake。

这就是一种更偏向声明式（Declarative）的使用方式。

注意一个细节：CMake 并不是一门纯声明式语言。它有变量、条件、循环、函数等，也有明显的命令式执行特征。

但是，**现代 CMake 最推荐的项目组织方式是基于 Target 描述项目，而不是手动拼接编译命令。**

这就是你以后要重点掌握的设计范式。

## 6. 设计哲学二：Target 是整个 CMake 的核心

**最重要的概念**

先问你一个问题：

一个 C++ 项目最终会产生什么？

可能产生：

- 一个可执行文件。
- 一个静态库。
- 一个动态库。
- 多个可执行文件。
- 多个静态库和动态库。
- 一些测试程序。

这些东西在 CMake 中有一个统一的概念：

**Target（目标）。**

我们先认识三个最常见的 Target 类型。

### 6.1 Executable Target：可执行目标

例如：

```cmake
add_executable(my_app
    main.cpp
    math.cpp
)
```

它的含义是：

创建一个叫 `my_app` 的可执行目标，使用两个源文件。

在 Linux 上，最终可能得到：

```text
my_app
```

在 Windows 上，可能得到：

```text
my_app.exe
```

CMake 会根据平台决定具体的产物形式。

### 6.2 Static Library Target：静态库目标

假设你的项目有一部分数学运算代码。

你不想把所有代码都直接塞进 `main.cpp` 对应的程序里，而是希望把数学运算单独组织成一个库。

你可以这样写：

```cmake
add_library(math STATIC
    math.cpp
)
```

这代表创建一个叫 `math` 的静态库。

Linux 下通常生成：

```text
libmath.a
```

Windows 下则会生成对应的静态库文件，例如 `.lib`。

之后其他程序就可以使用这个库。

### 6.3 Shared Library Target：动态库目标

```cmake
add_library(math SHARED
    math.cpp
)
```

这代表创建一个动态库。

常见产物：

| 平台 | 动态库形式 |
|---|---|
| Linux | `.so` |
| Windows | `.dll`，通常还会有导入 `.lib` |
| macOS | `.dylib` |

当然，CMake 还有 OBJECT、INTERFACE 等库类型，后面会讲到。

### 6.4 为什么 Target 如此重要？

现在我们稍微改变一下项目。

```text
MyProject/
├── main.cpp
├── math.cpp
├── math.h
├── logger.cpp
└── logger.h
```

我们希望：

- `math.cpp` 和 `math.h` 组成一个数学库。
- `logger.cpp` 和 `logger.h` 组成一个日志库。
- `main.cpp` 编译成主程序。
- 主程序使用数学库和日志库。

那么项目关系是：

```text
                       my_app
                  Executable Target
                          |
              +-----------+-----------+
              |                       |
              v                       v
             math                   logger
         Static Library         Static Library
            math.cpp               logger.cpp
```

my_app 依赖 math 和 logger 两个库

CMake 可以这样描述：

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject LANGUAGES CXX)

add_library(math STATIC math.cpp)
add_library(logger STATIC logger.cpp)

add_executable(my_app main.cpp)

target_link_libraries(my_app PRIVATE
    math
    logger
)
```

这里最重要的一句：

```cmake
target_link_libraries(my_app PRIVATE math logger)
```

它表示：

`my_app` 依赖 `math` 和 `logger`。

CMake 知道了这层依赖关系，就可以自动处理构建顺序、链接参数以及相应的构建关系。

你不需要自己写：

```bash
g++ -c math.cpp -o math.o
g++ -c logger.cpp -o logger.o
g++ -c main.cpp -o main.o

g++ main.o math.o logger.o -o my_app
```

当然，实际生成的命令会根据编译器、平台和构建配置有所不同。

### 6.5 一个非常重要的思想：Target 是一个带有属性的对象

你可以把一个 Target 想象成一个工程对象。

它不只有名字和源文件，还可以具有很多属性：

| 属性 | 意义 |
|---|---|
| Sources | 它由哪些源文件组成 |
| Include Directories | 编译时去哪里搜索头文件 |
| Compile Features | 需要什么 C++ 语言特性 |
| Compile Options | 使用什么编译选项 |
| Compile Definitions | 定义哪些宏 |
| Link Libraries | 需要链接哪些库 |
| Output Name | 最终生成文件叫什么 |

例如：

```cmake
add_executable(my_app main.cpp)

target_compile_features(my_app PRIVATE cxx_std_20)

target_compile_options(my_app PRIVATE
    -Wall
    -Wextra
)
```

这里你是在给 `my_app` 这个 Target 添加属性。

而不是直接告诉 GCC 应该执行什么命令。

这就是现代 CMake 的核心思维：

**先创建 Target，再通过各种 `target_xxx()` 命令描述它的构建需求。**

以后你看到：

```cmake
target_include_directories(...)
target_link_libraries(...)
target_compile_options(...)
target_compile_definitions(...)
target_compile_features(...)
```

不要把它们看成毫无关联的独立命令。

它们都是在操作 Target。

---

## 7. 设计哲学三：依赖关系应该由 Target 自己管理

这是一个稍微难一点、但极其重要的概念。

我们从一个具体问题开始。

假设你有一个数学库：

```text
MyProject/
├── main.cpp
├── math.cpp
└── math.h
```

`main.cpp`：

```cpp
#include <iostream>
#include "math.h"

int main() {
    std::cout << add(1, 2) << '\n';
}
```

你可能会想：

既然 `math.h` 和 `math.cpp` 在项目根目录，那么编译器应该能够找到它们吧？

不一定。

这里要区分两件事：

- CMake 知道哪些文件属于哪个 Target。
- 编译器知道去哪里搜索 `#include` 指定的头文件。

它们不是一回事。

CMake 把 `main.cpp` 交给编译器，不代表编译器就自动知道所有头文件的搜索路径。

我们需要告诉 CMake：

```cmake
target_include_directories(my_app PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
)
```

这里出现了一个新的概念：

`target_include_directories()`。

它的作用是为指定 Target 添加头文件搜索目录。

那么问题来了。

如果我们有很多 Target，它们都需要使用 `math.h` 呢？

是不是每一个都要写一遍？

例如：

```cmake
target_include_directories(app1 PRIVATE include)
target_include_directories(app2 PRIVATE include)
target_include_directories(app3 PRIVATE include)
target_include_directories(app4 PRIVATE include)
```

这显然有点笨。

我们希望：

> 如果一个库要求使用者搜索某个头文件目录，那么只要其他 Target 依赖这个库，就应该自动获得这个目录。

这就引出了现代 CMake 中最重要的三个关键字。

## 8. PUBLIC、PRIVATE、INTERFACE：到底是什么？

这三个关键字是你学习 CMake 时最应该理解透彻的内容之一。

先记住一个原则：

**它们描述的是一个 Target 的使用要求，究竟是只对自己生效，还是也传递给使用它的其他 Target。**

### 8.1 PRIVATE：只对自己生效

例如：

```cmake
target_include_directories(math PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/internal
)
```

意思是：

`math` 自己编译时需要搜索 `internal` 目录。

但是，其他依赖 `math` 的 Target 不会自动获得这个搜索目录。

适合什么情况？

例如：

```text
math/
├── include/
│   └── math.h
└── src/
    ├── math.cpp
    └── internal/
        └── math_internal.h
```

`math.cpp` 需要使用 `math_internal.h`。

但外部使用者只需要使用公开的 `math.h`，不需要知道内部实现。

所以 `internal` 应该是 PRIVATE。

### 8.2 PUBLIC：自己需要，使用者也需要

例如：

```cmake
target_include_directories(math PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

意思是：

- `math` 自己编译时需要这个目录。
- 任何依赖 `math` 的 Target，也会继承这个头文件搜索目录。

为什么会需要这种设计？

假设 `math.h` 是这样的：

```cpp
#pragma once

int add(int a, int b);
```

外部的 `main.cpp` 需要：

```cpp
#include "math.h"
```

那么使用者也必须能够找到 `math.h`。

因此，公开的头文件目录应该作为库的使用要求传递给消费者。

### 8.3 INTERFACE：自己不需要，使用者需要

例如：

```cmake
add_library(math INTERFACE)

target_include_directories(math INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

意思是：

`math` 这个 Target 自己不需要编译任何源文件，但它的使用者需要搜索这个目录。

这通常和 INTERFACE Library 一起使用。

例如，你有一个纯头文件库：

```text
MyProject/
├── include/
│   └── math.hpp
└── main.cpp
```

里面只有模板代码：

```cpp
#pragma once

template<typename T>
T add(T a, T b) {
    return a + b;
}
```

模板的实现就在头文件里，使用者包含头文件即可，不需要单独编译一个 `math.cpp`。

你可以这样组织：

```cmake
add_library(math INTERFACE)

target_include_directories(math INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

add_executable(my_app main.cpp)

target_link_libraries(my_app PRIVATE math)
```

这里的 `math` 不产生 `.a` 或 `.so` 文件。

它只是一个用于传递使用要求的逻辑 Target。

### 8.4 用一张表彻底记住

| 关键字 | 当前 Target 自己使用 | 依赖它的 Target 使用 |
|---|---|---|
| PRIVATE | 是 | 否 |
| PUBLIC | 是 | 是 |
| INTERFACE | 否 | 是 |

这张表非常重要。

以后遇到这三个关键字，你就不要死记硬背了，而是问自己：

1. 这个库自己是否需要这个配置？
2. 使用这个库的其他 Target 是否也需要这个配置？

答案就出来了。

### 8.5 用一个完整的例子理解依赖传递

假设：

```text
app
 │
 ▼
 library_A
 │
 ▼
 library_B
```

`library_A` 的公开头文件里包含了 `library_B` 的头文件：

```cpp
// library_A/include/a.h

#pragma once
#include "b.h"

void do_something();
```

`app` 使用 `a.h`：

```cpp
#include "a.h"
```

那么 `app` 间接也需要找到 `b.h`。

如果 `library_A` 这样声明：

```cmake
target_link_libraries(library_A PUBLIC library_B)
```

那么 `library_B` 的接口使用要求就会传递给 `app`。

因此，`app` 不需要再手动重复声明 `library_B`。

这叫做依赖传递（Transitive Dependencies）。

它是现代 CMake 很重要的能力。

你以后开发 HTTP Server、交易系统时，一个 Target 可能依赖很多其他 Target。如果每个模块都需要手动重复声明所有传递依赖，项目很快就会变得难以维护。

**正确做法是让每个 Target 负责声明自己的依赖和对外使用要求。**

官方 CMake 教程也专门使用 usage requirements（使用要求）来讲解这套设计。（[CMake 官方 Getting Started Tutorial](https://cmake.org/cmake/help/git-stage/guide/tutorial/Getting%20Started%20with%20CMake.html)；[CMake 官方教程：Mastering CMake](https://cmake.org/cmake/help/book/mastering-cmake/cmake/Help/guide/tutorial/)）

---

## 9. 设计哲学四：构建配置与源代码分离

这是另一个很重要的设计理念。

假设你直接在源代码目录执行：

```bash
g++ main.cpp -o main
```

那么你的项目目录会逐渐出现：

```text
MyProject/
├── main.cpp
├── math.cpp
├── main
├── main.o
├── math.o
├── CMakeCache.txt
├── Makefile
└── ...
```

源代码、构建中间文件、最终产物混在一起。

这会带来很多问题：

- 项目目录越来越乱。
- 容易把构建产物提交到 Git。
- Debug 和 Release 产物容易混淆。
- 使用不同编译器构建时容易互相干扰。
- 更换构建工具可能需要清理大量文件。

CMake 强烈推荐一种叫作：

**Out-of-source Build（源码外构建）** 的方式。

例如：

```text
MyProject/
│
├── CMakeLists.txt
├── src/
│   ├── main.cpp
│   └── math.cpp
│
├── include/
│   └── math.h
│
└── build/
    ├── CMakeCache.txt
    ├── CMakeFiles/
    ├── build.ninja
    ├── src/
    └── my_app
```

源代码放在：

```text
MyProject/
```

构建过程中生成的文件放在：

```text
MyProject/build/
```

这样就把两件事情分离了：

- Source Tree：源代码目录。
- Build Tree：构建目录。

以后你可以有多个 Build Tree：

```text
MyProject/
├── CMakeLists.txt
├── src/
├── include/
│
├── build-debug/
├── build-release/
├── build-clang/
└── build-msvc/
```

它们可以分别对应：

| 目录 | 配置 |
|---|---|
| build-debug | GCC + Debug |
| build-release | GCC + Release |
| build-clang | Clang + Release |
| build-msvc | MSVC + Debug |

这样一份源代码，就能对应多个独立的构建环境。

这也是为什么 CMake 命令经常会出现：

```bash
cmake -S . -B build
```

其中：

- `-S` 指定 Source Directory。
- `-B` 指定 Build Directory。

后面我们会完整讲解。

---

## 10. 设计哲学五：配置（Configure）、生成（Generate）、构建（Build）是三个不同阶段

**必须理解的工作流程**

这是很多初学者最容易混淆的地方。

当你第一次使用 CMake 时，可能会觉得：

```bash
cmake -S . -B build
```

这条命令是不是已经把 C++ 编译完了？

不是。

它主要完成的是配置和生成阶段。

我们把完整过程拆开。

| 阶段 | 工作 | 说明 |
|---|---|---|
| 阶段 1 | **Configure：配置阶段** | CMake 读取 `CMakeLists.txt`，检查项目结构、编译器、系统环境、依赖和用户提供的配置选项。 |
| 阶段 2 | **Generate：生成阶段** | CMake 根据项目配置和目标依赖关系，生成 Makefile、Ninja 文件、Visual Studio 工程等底层构建文件。 |
| 阶段 3 | **Build：构建阶段** | Make、Ninja、MSBuild 等底层构建工具，根据生成的构建规则调用编译器和链接器，生成最终产物。 |

```text
Configure -> Generate -> Build -> 最终产物
```

**最终产物**

Executable / Static Library / Shared Library

### 10.1 Configure 阶段到底做了什么？

假设你写了：

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject LANGUAGES CXX)

add_executable(my_app main.cpp)
```

你执行：

```bash
cmake -S . -B build
```

CMake 需要了解：

- 当前操作系统是什么？
- 当前使用什么编译器？
- 编译器能不能正常工作？
- 当前项目使用什么语言？
- 项目里有哪些 Target？
- 需要哪些外部依赖？
- 用户有没有传入额外配置？

比如你在 Linux 上使用 GCC：

```text
Operating System: Linux
C++ Compiler: GCC
Generator: Ninja
```

换成 Windows + MSVC：

```text
Operating System: Windows
C++ Compiler: MSVC
Generator: Visual Studio
```

同一份 `CMakeLists.txt`，最终得到的构建配置可以完全不同。

这里还涉及一个很重要的概念：

**Toolchain（工具链）。**

对于一个普通的 C++ 项目，工具链涉及的核心工具包括：

- C/C++ 编译器。
- 链接器。
- 汇编器等相关工具。
- 必要的系统工具和 SDK。

CMake 本身并不会替你提供 GCC、Clang 或 MSVC。它需要找到并使用你已经安装、配置好的工具链。

### 10.2 Generate 阶段做什么？

Configure 阶段分析完项目后，CMake 已经知道：

```text
Target: my_app
Sources:
    main.cpp

Compiler:
    GCC

C++ Standard:
    C++20

Include Directories:
    include/

Dependencies:
    math
```

现在它需要把这些信息转换成底层构建工具能够理解的规则。

例如生成：

```text
build/
├── Makefile
├── CMakeCache.txt
└── CMakeFiles/
```

如果使用 Ninja：

```text
build/
├── build.ninja
├── CMakeCache.txt
└── CMakeFiles/
```

如果使用 Visual Studio 生成器，则会生成对应的 Visual Studio 项目文件。

### 10.3 Build 阶段做什么？

现在才真正开始编译。

执行：

```bash
cmake --build build
```

CMake 会调用之前选定的底层构建工具。

例如：

```text
cmake --build build
          │
          ▼
       Ninja
          │
          ▼
        GCC
          │
          ▼
       Linker
          │
          ▼
       my_app
```

注意：

`cmake --build` 是一个很有用的跨平台接口。

你不必关心当前底层到底是 Make 还是 Ninja。

同一条命令可以让 CMake 调用相应的构建工具。

### 10.4 为什么一定要分成三个阶段？

因为这三件事情的性质不同。

- Configure：确定这个项目应该如何构建。
- Generate：把项目描述转换成具体构建规则。
- Build：执行这些规则。

例如：

你修改了 `CMakeLists.txt`，增加一个 Target。

那么需要重新配置、生成构建规则。

但你只修改了一个普通 `.cpp` 文件，通常不需要重新完整配置整个项目，只需要让底层构建工具根据依赖关系重新编译必要的部分。

这也是 CMake 能够很好地支持增量构建的原因。

---

## 11. 设计哲学六：CMake 不负责替代底层构建工具，而是利用它们

这点再强调一下。

假设你使用：

```bash
cmake -G Ninja -S . -B build
```

这里的 `-G Ninja` 就是指定生成器（Generator）。

CMake 根据你的项目生成 Ninja 能理解的构建文件。

然后：

```bash
cmake --build build
```

实际执行构建时，调用 Ninja。

你也可以选择 Makefiles：

```bash
cmake -G "Unix Makefiles" -S . -B build
```

然后执行：

```bash
cmake --build build
```

调用的就是 Make。

因此：

| 名称 | 职责 |
|---|---|
| CMake | 解析项目描述、配置项目、生成构建规则 |
| Generator | 决定生成哪一种底层构建系统 |
| Make / Ninja | 根据构建规则调度构建任务 |
| GCC / Clang / MSVC | 编译 C/C++ 代码 |
| Linker | 链接目标文件和库 |

这里还有一个重要的细节：

**一个 Build Directory 通常对应一个固定的 Generator。**

例如你第一次：

```bash
cmake -G Ninja -S . -B build
```

之后不能直接在同一个 `build` 目录中切换成：

```bash
cmake -G "Unix Makefiles" -S . -B build
```

因为生成器信息已经保存在构建目录的缓存中。

正确做法是：

```bash
cmake -G "Unix Makefiles" -S . -B build-make
```

或者删除原来的 `build` 目录，再重新配置。

---

# 第三部分：正式学习 CMake 的语言和常用语法

前面讲的是设计思想。现在开始落地。

你会发现，只要真正理解了前面的内容，很多 CMake 语法其实非常自然。

## 12. CMakeLists.txt 是什么？

CMake 项目最重要的文件就是：

```text
CMakeLists.txt
```

注意大小写。

它是 CMake 的项目描述文件。

当你执行：

```bash
cmake -S . -B build
```

CMake 就会从指定的源码目录中寻找顶层 `CMakeLists.txt`，并读取其中的配置。

一个项目可以有多个 `CMakeLists.txt`。

例如：

```text
MyProject/
│
├── CMakeLists.txt
│
├── src/
│   ├── CMakeLists.txt
│   ├── main.cpp
│   └── math.cpp
│
├── include/
│   └── math.h
│
└── tests/
    ├── CMakeLists.txt
    └── test_math.cpp
```

为什么会有多个？

因为大型项目往往有多个子模块。

每个子模块可以管理自己的 Target、源文件和依赖关系。

顶层 `CMakeLists.txt` 负责组织整个项目，子目录的 `CMakeLists.txt` 负责组织各自的模块。

这和 C++ 中把代码拆分成多个模块的思想非常相似。

## 13. CMake 的基本语法长什么样？

先认识一个最简单的例子：

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject)

add_executable(my_app main.cpp)
```

这里有几个特点。

### 13.1 CMake 的命令形式

基本形式是：

```cmake
命令名称(参数1 参数2 参数3)
```

例如：

```cmake
project(MyProject)
```

又例如：

```cmake
add_executable(my_app main.cpp math.cpp)
```

CMake 的命令名称不区分大小写：

```cmake
project(MyProject)
PROJECT(MyProject)
Project(MyProject)
```

都可以。

不过建议统一使用小写，符合现在大多数项目的习惯。

### 13.2 注释

使用 `#`：

```cmake
# 这是一个注释

project(MyProject)
```

### 13.3 变量

CMake 中变量的定义和使用是这样的：

```cmake
set(MY_NAME "hello")

message(${MY_NAME})
```

其中：

```cmake
set(MY_NAME "hello")
```

创建一个变量。

而：

```cmake
${MY_NAME}
```

表示读取这个变量的值。

你可能会疑惑：为什么不用 `$MY_NAME`？

这是 CMake 自己的语法设计。

以后看到：

```cmake
${CMAKE_CURRENT_SOURCE_DIR}
```

你就应该知道，这是在读取一个名叫 `CMAKE_CURRENT_SOURCE_DIR` 的变量。

### 13.4 列表

CMake 的列表非常常见。

例如：

```cmake
set(SOURCES
    main.cpp
    math.cpp
    logger.cpp
)
```

这里的 `SOURCES` 是一个列表变量。

可以这样使用：

```cmake
add_executable(my_app ${SOURCES})
```

相当于：

```cmake
add_executable(my_app
    main.cpp
    math.cpp
    logger.cpp
)
```

CMake 列表内部使用分号分隔元素。

你也可以这样写：

```cmake
set(SOURCES "main.cpp;math.cpp;logger.cpp")
```

不过通常建议使用换行排版，阅读体验更好。

---

## 14. 最重要的几个基础命令

下面正式认识常用命令。

### 14.1 cmake_minimum_required()

```cmake
cmake_minimum_required(VERSION 3.20)
```

作用：

声明项目要求的最低 CMake 版本。

为什么需要它？

因为 CMake 不断发展，新版本会引入新语法、新命令、新特性。

如果你使用了 CMake 3.20 才有的功能，却允许别人使用 CMake 3.10 来构建项目，就可能出现问题。

另外，这条命令还会设置相应的 CMake Policy 版本行为，帮助项目采用预期的兼容规则。

初学阶段，你可以先把它理解为：

> 我这个项目至少需要某个版本的 CMake。

我建议你学习现代 CMake 时，从 3.20 或更高版本开始。

如果你的开发环境和目标平台都能使用较新的 CMake，也可以提高最低版本要求，但不要为了追求新版本而盲目设置过高的门槛。

### 14.2 project()

```cmake
project(MyProject)
```

作用：

声明一个 CMake 项目。

你可以进一步指定版本和语言：

```cmake
project(
    MyProject
    VERSION 1.0.0
    LANGUAGES C CXX
)
```

解释：

| 参数 | 含义 |
|---|---|
| MyProject | 项目名称 |
| VERSION | 项目版本 |
| LANGUAGES | 项目使用的编程语言 |

注意：

```cmake
LANGUAGES C CXX
```

表示这个项目可能同时包含 C 和 C++ 源文件。

例如：

```text
MyProject/
├── main.cpp
├── network.cpp
├── c_library.c
└── c_library.h
```

CMake 会根据文件类型，选择相应的编译器处理。

如果你的项目只有 C++：

```cmake
project(MyProject LANGUAGES CXX)
```

如果只有 C：

```cmake
project(MyProject LANGUAGES C)
```

我建议你以后写 C/C++ 混合项目时，明确写出 `LANGUAGES C CXX`，不要依赖隐式语言启用。

### 14.3 set()

```cmake
set(MY_VAR hello)
```

定义变量。

实际使用中，最常见的场景之一就是管理源文件：

```cmake
set(SOURCES
    main.cpp
    math.cpp
    logger.cpp
)

add_executable(my_app ${SOURCES})
```

但我要提醒你：

**不要因为有 `set()`，就把所有东西都塞进全局变量。**

现代 CMake 更推荐把配置放到对应 Target 上。

例如：

```cmake
target_include_directories(my_app PRIVATE include)
```

通常比到处维护一个全局 `INCLUDE_DIRS` 变量更清晰。

### 14.4 add_executable()

```cmake
add_executable(my_app
    main.cpp
    math.cpp
)
```

创建可执行 Target。

基本语法：

```cmake
add_executable(目标名称 源文件...)
```

你可以把多个 `.cpp` 放在里面。

还可以使用变量：

```cmake
set(APP_SOURCES
    main.cpp
    math.cpp
)

add_executable(my_app ${APP_SOURCES})
```

特别注意：

头文件不一定要放进 `add_executable()`。

例如：

```cpp
#include "math.h"
```

编译器能够通过头文件搜索路径找到它即可。

不过，为了让 IDE 更好地展示项目结构、方便管理，有些项目也会把头文件列入 Target 的源文件列表。

### 14.5 add_library()

创建库 Target。

最常见的形式：

```cmake
add_library(math STATIC math.cpp)
```

静态库。

```cmake
add_library(math SHARED math.cpp)
```

动态库。

```cmake
add_library(math OBJECT math.cpp)
```

对象库。

```cmake
add_library(math INTERFACE)
```

接口库。

这四种要有所了解，但你目前主要掌握前三种就可以了。

我再解释一下 OBJECT 和 INTERFACE。

**OBJECT Library**

```cmake
add_library(math_obj OBJECT
    math.cpp
)
```

它会编译出目标文件，但不会像 STATIC 或 SHARED 那样生成一个普通的独立库文件。

适合某些需要复用编译结果的项目。

**INTERFACE Library**

```cmake
add_library(math INTERFACE)
```

它本身不编译源文件，也不产生普通的二进制库。

它主要用于传递头文件目录、编译选项、宏定义等使用要求。

模板库、header-only 库经常使用它。

### 14.6 target_sources()

你已经创建了 Target：

```cmake
add_executable(my_app main.cpp)
```

现在发现还需要增加一个文件：

```cmake
network.cpp
```

可以这样写：

```cmake
target_sources(my_app PRIVATE
    network.cpp
)
```

这相当于给已经存在的 Target 添加源文件。

对于小项目，直接把文件放在 `add_executable()` 里就很好。

对于大型项目，`target_sources()` 可以帮助你将一个 Target 的源文件组织在多个位置。

---

## 15. 头文件搜索路径：target_include_directories()

这是你以后最常用的命令之一。

先看一个项目：

```text
MyProject/
├── CMakeLists.txt
├── src/
│   └── main.cpp
└── include/
    └── math.h
```

`main.cpp`：

```cpp
#include "math.h"

int main() {
    return 0;
}
```

我们希望编译器能找到：

```text
include/math.h
```

那么：

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject LANGUAGES CXX)

add_executable(my_app
    src/main.cpp
)

target_include_directories(my_app PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

解释：

```cmake
target_include_directories(
    my_app
    PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

- `my_app`：要配置的 Target。
- `PRIVATE`：只对当前 Target 生效。
- 后面的路径：需要添加的头文件搜索目录。

### CMAKE_CURRENT_SOURCE_DIR 是什么？

它是 CMake 提供的内置变量。

表示：

**当前正在处理的 CMakeLists.txt 所在的源码目录。**

例如：

```text
MyProject/
├── CMakeLists.txt
└── src/
    └── CMakeLists.txt
```

如果当前正在处理根目录的 `CMakeLists.txt`：

```cmake
${CMAKE_CURRENT_SOURCE_DIR}
```

就是：

```text
MyProject/
```

如果当前正在处理 `src/CMakeLists.txt`：

```cmake
${CMAKE_CURRENT_SOURCE_DIR}
```

就是：

```text
MyProject/src/
```

还有几个你以后会经常看到的变量：

| 变量 | 意义 |
|---|---|
| `CMAKE_SOURCE_DIR` | 最顶层源码目录 |
| `CMAKE_BINARY_DIR` | 最顶层构建目录 |
| `CMAKE_CURRENT_SOURCE_DIR` | 当前 CMakeLists 所在的源码目录 |
| `CMAKE_CURRENT_BINARY_DIR` | 当前 CMakeLists 对应的构建目录 |
| `PROJECT_SOURCE_DIR` | 当前 project() 对应的源码目录 |
| `PROJECT_BINARY_DIR` | 当前 project() 对应的构建目录 |

初学时最常用的就是：

```cmake
${CMAKE_CURRENT_SOURCE_DIR}
```

和：

```cmake
${PROJECT_SOURCE_DIR}
```

如果你在根目录写配置，两者经常指向同一个位置；进入子目录后，它们就可能不同了。

---

## 16. 链接库：target_link_libraries()

这也是 CMake 的核心命令。

假设：

```text
MyProject/
├── CMakeLists.txt
├── main.cpp
├── math.cpp
└── math.h
```

我们希望把数学代码编译成静态库，再链接到主程序。

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject LANGUAGES CXX)

add_library(math STATIC
    math.cpp
)

target_include_directories(math PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
)

add_executable(my_app
    main.cpp
)

target_link_libraries(my_app PRIVATE
    math
)
```

这段代码发生了什么？

第一步：

```cmake
add_library(math STATIC math.cpp)
```

创建静态库 Target。

第二步：

```cmake
target_include_directories(math PUBLIC ...)
```

声明 math 的头文件使用要求。

第三步：

```cmake
add_executable(my_app main.cpp)
```

创建可执行程序 Target。

第四步：

```cmake
target_link_libraries(my_app PRIVATE math)
```

声明 my_app 依赖 math。

这样 CMake 就知道了它们之间的关系。

### 一个容易产生误解的地方

`target_link_libraries()` 并不只是简单地把某个 `.a` 文件塞给链接器。

它还负责建立 Target 之间的依赖关系，并传递相应的使用要求。

因此，当你链接一个由 CMake 管理的库 Target 时，CMake 可以知道：

- 这个库的构建产物在哪里。
- 它依赖哪些其他 Target。
- 它公开了哪些头文件搜索目录。
- 它要求什么编译特性。
- 它有哪些传递链接依赖。

这就是为什么现代 CMake 非常强调链接 Target，而不是手动写死 `.a`、`.so` 的路径。

---

## 17. 编译选项、C++ 标准和宏定义

这三个东西在你后续做高性能 C++ 项目时会经常出现。

### 17.1 设置 C++ 标准

最直观的写法：

```cmake
set(CMAKE_CXX_STANDARD 20)

set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_CXX_EXTENSIONS OFF)
```

解释：

`CMAKE_CXX_STANDARD`：

指定默认 C++ 标准。

`CMAKE_CXX_STANDARD_REQUIRED ON`：

要求使用指定标准，而不是在编译器不支持时悄悄降级。

`CMAKE_CXX_EXTENSIONS OFF`：

尽量使用标准 C++ 方言，而不是 GCC 的 GNU 扩展方言。

例如 GCC 下常见的区别：

```text
-std=c++20
-std=gnu++20
```

前者更接近标准 C++，后者包含 GNU 扩展。

还有一种现代 CMake 写法：

```cmake
target_compile_features(my_app PRIVATE cxx_std_20)
```

它表示这个 Target 需要 C++20 特性。

它有一个优点：CMake 会根据编译器支持情况处理相应的语言标准要求。

我的建议：

- 学习和普通项目：使用 `CMAKE_CXX_STANDARD` 设置项目默认标准很方便。
- 模块化、大型项目：逐渐学会使用 `target_compile_features()`，让每个 Target 声明自己的语言特性需求。

### 17.2 设置编译选项

你可能已经熟悉：

```bash
g++ -Wall -Wextra -O2
```

在 CMake 中：

```cmake
target_compile_options(my_app PRIVATE
    -Wall
    -Wextra
    -Wpedantic
)
```

这些选项会应用到对应 Target 的编译过程中。

但有一个问题：

GCC、Clang 和 MSVC 的编译选项并不完全相同。

例如：

| 意义 | GCC / Clang | MSVC |
|---|---|---|
| 常用警告等级 | `-Wall -Wextra` | `/W4` |
| 优化 | `-O2` | `/O2` |
| 调试信息 | `-g` | `/Zi` |

所以，跨平台项目不要直接无条件写 GCC 专用参数。

可以使用条件判断：

```cmake
if(MSVC)

    target_compile_options(my_app PRIVATE
        /W4
    )

else()

    target_compile_options(my_app PRIVATE
        -Wall
        -Wextra
        -Wpedantic
    )

endif()
```

这样：

- MSVC 使用 `/W4`。
- 其他编译器使用 GCC/Clang 风格的警告参数。

更复杂的项目还会使用生成器表达式来区分编译器和配置，后面会提到。

### 17.3 设置宏定义

例如你平时可能会写：

```cpp
#define DEBUG_MODE 1
```

或者通过编译器：

```bash
g++ -DDEBUG_MODE=1 main.cpp
```

CMake：

```cmake
target_compile_definitions(my_app PRIVATE
    DEBUG_MODE=1
)
```

如果使用者也需要这个宏：

```cmake
target_compile_definitions(my_library PUBLIC
    MY_LIBRARY_VERSION=1
)
```

同样遵循：

- PRIVATE：自己使用。
- PUBLIC：自己和使用者使用。
- INTERFACE：只有使用者使用。

---

## 18. 多目录项目：add_subdirectory()

现在你已经会了：

- 创建 Target。
- 创建静态库。
- 创建可执行程序。
- 设置头文件目录。
- 链接库。

但如果一个项目越来越大，难道所有东西都写在根目录的 `CMakeLists.txt` 里吗？

当然不是。

假设：

```text
MyProject/
│
├── CMakeLists.txt
│
├── src/
│   ├── CMakeLists.txt
│   ├── main.cpp
│   └── network.cpp
│
├── math/
│   ├── CMakeLists.txt
│   ├── math.cpp
│   └── math.h
│
└── logger/
    ├── CMakeLists.txt
    ├── logger.cpp
    └── logger.h
```

根目录：

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject LANGUAGES CXX)

add_subdirectory(math)

add_subdirectory(logger)

add_subdirectory(src)
```

这几行的意思是：

让 CMake 进入对应的子目录，继续处理其中的 `CMakeLists.txt`。

`math/CMakeLists.txt`：

```cmake
add_library(math STATIC
    math.cpp
)

target_include_directories(math PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
)
```

`logger/CMakeLists.txt`：

```cmake
add_library(logger STATIC
    logger.cpp
)

target_include_directories(logger PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
)
```

`src/CMakeLists.txt`：

```cmake
add_executable(my_app
    main.cpp
    network.cpp
)

target_link_libraries(my_app PRIVATE
    math
    logger
)
```

现在整个项目的结构就清晰多了。

每个目录只负责自己的模块。

根目录负责组织它们。

这也是大型 C++ 项目经常使用的组织方式。

### 注意：add_subdirectory 不等于创建一个新的独立项目

这是一个容易搞错的点。

```cmake
add_subdirectory(math)
```

不是说 `math` 变成一个完全独立的 CMake 项目。

它仍然处于父项目的构建体系中。

子目录可以创建 Target，而其他目录的 Target 可以根据依赖关系使用它们。

通常不需要在每一个子目录里重新写：

```cmake
project(...)
```

只有当你确实需要独立定义子项目的项目名称、版本或语言范围时，才考虑额外调用。

---

## 19. 第三方库：find_package()

现在进入一个你以后肯定会经常遇到的场景。

假设你要写 HTTP Server。

你可能需要：

- OpenSSL：TLS 加密。
- Threads：多线程。
- SQLite：数据库。
- Boost：一些通用 C++ 库。

这些库不一定是你自己写的。

你怎么告诉 CMake：

> 我要使用这个已经安装在系统里的第三方库？

最常见的方法就是：

```cmake
find_package()
```

### 19.1 一个简单的例子：Threads

假设你写了：

```cpp
#include <thread>
#include <iostream>

int main() {
    std::thread t([] {
        std::cout << "hello\n";
    });

    t.join();
}
```

在 CMake 中：

```cmake
cmake_minimum_required(VERSION 3.20)

project(ThreadDemo LANGUAGES CXX)

find_package(Threads REQUIRED)

add_executable(thread_demo main.cpp)

target_link_libraries(thread_demo PRIVATE
    Threads::Threads
)
```

解释：

```cmake
find_package(Threads REQUIRED)
```

告诉 CMake：

寻找线程支持。

`REQUIRED` 表示找不到就报错，不允许继续配置。

然后：

```cmake
target_link_libraries(thread_demo PRIVATE
    Threads::Threads
)
```

把线程依赖链接到你的程序。

这里的：

```cmake
Threads::Threads
```

叫作 Imported Target（导入目标）。

它不是你自己创建的 Target，而是 CMake 找到相应依赖后提供给你使用的 Target。

它可能携带必要的编译选项、链接选项等信息。

你不需要自己判断当前平台应该添加哪个参数。

### 19.2 再看一个例子：OpenSSL

假设你需要 OpenSSL。

```cmake
find_package(OpenSSL REQUIRED)

add_executable(http_server main.cpp)

target_link_libraries(http_server PRIVATE
    OpenSSL::SSL
    OpenSSL::Crypto
)
```

其中：

- `OpenSSL::SSL`：SSL/TLS 相关库。
- `OpenSSL::Crypto`：加密相关库。

CMake 会根据找到的 OpenSSL 安装信息，提供对应的依赖目标。

注意：

`find_package()` 并不意味着它一定会自动从网上下载库。

它主要是查找依赖。

依赖可以来自：

- 系统包管理器安装的库。
- 你自己编译安装的库。
- vcpkg、Conan 等包管理器提供的库。
- 某些项目自带的依赖。

找不到依赖时，通常需要先安装它，或者告诉 CMake 去哪里寻找。

### 19.3 REQUIRED 和 CONFIG 是什么？

你可能还会看到：

```cmake
find_package(fmt CONFIG REQUIRED)
```

这里：

- `fmt`：要寻找的包名。
- `CONFIG`：优先通过包提供的 CMake Config 文件查找。
- `REQUIRED`：找不到就报错。

与之相对的还有 CMake 自己提供的 Find Module 查找方式。

你现在只需要知道：

`find_package()` 有不同的查找机制，不同第三方库提供的 CMake 集成方式也不一样。

先学会最基本的：

```cmake
find_package(包名 REQUIRED)
```

然后看对应第三方库的官方文档，确定使用哪个 Imported Target。

### 19.4 现代 CMake 为什么强调 Imported Target？

因为它和前面学习的 Target 思想是一致的。

假设你自己写了一个库：

```cmake
add_library(math STATIC math.cpp)
```

它是一个 Target。

假设你找到了 OpenSSL：

```cmake
find_package(OpenSSL REQUIRED)
```

CMake 给你提供：

```cmake
OpenSSL::SSL
```

它也是一个 Target。

于是你只需要：

```cmake
target_link_libraries(my_app PRIVATE
    math
    OpenSSL::SSL
)
```

不需要自己去拼：

```text
/usr/lib/libssl.so
/usr/include/openssl
```

更不需要根据不同操作系统手写不同的绝对路径。

这就是 Target-based CMake 的优势。

---

## 20. option()：让使用者决定是否启用某项功能

大型项目往往存在可选功能。

例如：

- 是否构建测试。
- 是否开启性能测试。
- 是否开启 TLS。
- 是否编译示例程序。

你可以使用：

```cmake
option(BUILD_TESTS "Build tests" ON)
```

这表示创建一个布尔选项，默认开启。

然后：

```cmake
if(BUILD_TESTS)

    add_subdirectory(tests)

endif()
```

用户可以在配置时关闭：

```bash
cmake -S . -B build -DBUILD_TESTS=OFF
```

这里的 `-D` 就是给 CMake 传入变量。

以后你看 Redis、数据库、网络服务器等大型项目的构建配置，可能会看到很多类似的选项。

---

# 第四部分：从零构建一个真正的 C++ 项目

前面学了很多语法，现在我们把它们串起来。

我建议你认真跟着这个例子走一遍。因为你真正需要掌握的不是孤立的 CMake 命令，而是：

**拿到一个项目，知道如何组织目录、编写 CMakeLists、配置依赖、生成构建文件、编译、链接，最后运行程序。**

我们做一个简单的数学库 + 主程序。

## 21. 实战：构建一个多文件 C++ 项目

### 21.1 创建项目结构

```text
MyProject/
│
├── CMakeLists.txt
│
├── include/
│   └── math/
│       └── math.h
│
├── src/
│   ├── CMakeLists.txt
│   ├── main.cpp
│   └── math.cpp
│
└── build/
```

其中：

- `include/`：对外公开的头文件。
- `src/`：源代码。
- 根目录 `CMakeLists.txt`：管理整个项目。
- `src/CMakeLists.txt`：管理源代码模块。
- `build/`：构建目录。

### 21.2 编写 C++ 代码

`include/math/math.h`：

```cpp
#pragma once

int add(int a, int b);

int multiply(int a, int b);
```

`src/math.cpp`：

```cpp
#include "math/math.h"

int add(int a, int b) {
    return a + b;
}

int multiply(int a, int b) {
    return a * b;
}
```

`src/main.cpp`：

```cpp
#include <iostream>
#include "math/math.h"

int main() {
    std::cout << add(10, 20) << '\n';

    std::cout << multiply(10, 20) << '\n';

    return 0;
}
```

### 21.3 编写根目录 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject
    VERSION 1.0.0
    LANGUAGES CXX
)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_subdirectory(src)
```

逐行解释：

```cmake
cmake_minimum_required(VERSION 3.20)
```

要求 CMake 至少为 3.20。

```cmake
project(MyProject
    VERSION 1.0.0
    LANGUAGES CXX
)
```

声明项目名称、版本和使用的语言。

```cmake
set(CMAKE_CXX_STANDARD 20)
```

设置默认 C++ 标准为 C++20。

这里的顺序有意义：这类标准变量通常用于初始化之后创建的 Target 的对应属性，所以应当在创建 Target 之前设置。

```cmake
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

要求编译器支持指定标准。

```cmake
set(CMAKE_CXX_EXTENSIONS OFF)
```

关闭 GNU 扩展方言。

最后：

```cmake
add_subdirectory(src)
```

让 CMake 继续处理 `src/` 下的 `CMakeLists.txt`。

### 21.4 编写 src/CMakeLists.txt

```cmake
add_library(math STATIC
    math.cpp
)

target_include_directories(math PUBLIC
    ${PROJECT_SOURCE_DIR}/include
)

add_executable(my_app
    main.cpp
)

target_link_libraries(my_app PRIVATE
    math
)
```

我们逐行分析。

首先：

```cmake
add_library(math STATIC math.cpp)
```

将 `math.cpp` 编译为静态库。

注意，这个库的公开头文件目录和源文件目录不是一个概念。

头文件放在：

```text
MyProject/include/
```

因此我们需要：

```cmake
target_include_directories(math PUBLIC
    ${PROJECT_SOURCE_DIR}/include
)
```

这里：

```cmake
${PROJECT_SOURCE_DIR}
```

表示当前项目的根源码目录。

由于 `math.h` 是 math 库对外提供的接口，所以我们使用 `PUBLIC`。

接下来：

```cmake
add_executable(my_app main.cpp)
```

创建可执行程序。

最后：

```cmake
target_link_libraries(my_app PRIVATE math)
```

让主程序依赖数学库。

这里非常值得你注意：

**我们没有给 my_app 再写一遍 include 目录！**

为什么？

因为 `math` 的头文件目录通过 `PUBLIC` 声明了使用要求。

`my_app` 链接 `math` 后，就可以继承这个使用要求。

这就是前面讲过的依赖传递。

### 21.5 现在项目是什么样的？

**CMake Target 关系**

```text
                my_app
          Executable · main.cpp
                   |
          target_link_libraries
                   |
                   v
                 math
        Static Library · math.cpp
```

**math 的公开使用要求**

include/ 目录，通过 PUBLIC 传递给消费者

### 21.6 开始配置项目

在项目根目录打开终端：

```bash
cmake -S . -B build
```

我们拆开：

```bash
cmake
```

调用 CMake。

```bash
-S .
```

指定源码目录为当前目录。

```bash
-B build
```

指定构建目录为 `build/`。

这一步主要进行 Configure 和 Generate。

如果你想指定 Ninja：

```bash
cmake -G Ninja -S . -B build
```

前提是环境中有可用的 Ninja。

如果你想使用 Make：

```bash
cmake -G "Unix Makefiles" -S . -B build
```

### 21.7 编译项目

配置成功后：

```bash
cmake --build build
```

你可以加上并行编译：

```bash
cmake --build build --parallel
```

也可以指定并行任务数：

```bash
cmake --build build --parallel 8
```

CMake 会让底层构建工具尽可能并行执行独立的构建任务。

最终可能得到：

```text
build/
├── CMakeCache.txt
├── CMakeFiles/
├── build.ninja
└── src/
    ├── libmath.a
    └── my_app
```

具体目录和文件名会随生成器、平台而变化。

### 21.8 运行程序

Linux / macOS，如果使用 Ninja 或 Unix Makefiles：

```bash
./build/src/my_app
```

预期输出：

```text
30
200
```

至此，你就完成了：

1. 编写 C++ 源代码。

2. 编写 CMakeLists.txt，定义 Target 和依赖关系。

3. 调用 CMake 进行配置和生成。

4. 调用底层构建工具完成编译和链接。

5. 运行最终生成的可执行文件。

这个项目虽然简单，但是已经包含了你未来大多数 C++ 项目都会使用的基础结构。

---

## 22. Debug 和 Release：你一定会遇到的构建模式

你可能已经知道：

```bash
g++ -g main.cpp
```

可以生成调试信息。

而：

```bash
g++ -O2 main.cpp
```

可以开启优化。

那么 CMake 怎么管理这些东西？

它有常见的几种构建类型：

| 构建类型 | 主要用途 |
|---|---|
| Debug | 调试，通常保留调试信息，优化较少 |
| Release | 发布，通常开启优化 |
| RelWithDebInfo | 优化，同时保留调试信息 |
| MinSizeRel | 尽量减小程序体积 |

### 22.1 单配置生成器

例如 Ninja、Unix Makefiles。

配置 Debug：

```bash
cmake -S . -B build-debug \
    -DCMAKE_BUILD_TYPE=Debug
```

构建：

```bash
cmake --build build-debug --parallel
```

配置 Release：

```bash
cmake -S . -B build-release \
    -DCMAKE_BUILD_TYPE=Release
```

构建：

```bash
cmake --build build-release --parallel
```

这里的：

```bash
-DCMAKE_BUILD_TYPE=Debug
```

就是在配置阶段给 CMake 的缓存变量传值。

`-D` 的含义是定义或设置一个 CMake 变量。

### 22.2 多配置生成器

例如 Visual Studio、Xcode、Ninja Multi-Config。

它们可以在同一个构建目录中支持多种构建配置。

这时候通常不在 Configure 阶段通过 `CMAKE_BUILD_TYPE` 选择 Debug 或 Release，而是在 Build 阶段指定：

```bash
cmake --build build --config Debug
```

或者：

```bash
cmake --build build --config Release
```

这是一个特别重要的区别。

很多教程只讲：

```bash
-DCMAKE_BUILD_TYPE=Release
```

但这并不是所有生成器都适用的配置方式。

### 22.3 对你来说应该怎么使用？

我建议：

```text
MyProject/
├── CMakeLists.txt
├── src/
├── include/
├── build-debug/
└── build-release/
```

初期使用两个独立构建目录，简单、直观，不容易把配置搞混。

---

## 23. CMake 的缓存机制：CMakeCache.txt 是什么？

前面你已经看到：

```text
build/
└── CMakeCache.txt
```

这个文件很重要。

它记录了 CMake 配置过程中使用的一些缓存变量。

例如：

```text
CMAKE_CXX_COMPILER
CMAKE_BUILD_TYPE
CMAKE_GENERATOR
```

还有你通过 `-D` 传入的选项：

```bash
cmake -S . -B build \
    -DCMAKE_BUILD_TYPE=Debug \
    -DBUILD_TESTS=OFF
```

这些配置会被保存到构建目录的缓存中。

下一次你再次执行：

```bash
cmake -S . -B build
```

CMake 就可以沿用之前的配置，而不需要你每次重新输入所有参数。

### 一个特别重要的注意事项：更换编译器

假设你第一次：

```bash
cmake -S . -B build \
    -DCMAKE_CXX_COMPILER=g++
```

然后想切换为 Clang：

```bash
cmake -S . -B build \
    -DCMAKE_CXX_COMPILER=clang++
```

这通常不是正确的操作方式。

因为编译器信息已经被缓存，CMake 构建目录也已经按照原来的工具链完成配置。

**更换编译器、生成器等基础配置时，最稳妥的办法是使用新的 Build Directory。**

例如：

```bash
cmake -S . -B build-gcc \
    -DCMAKE_CXX_COMPILER=g++

cmake -S . -B build-clang \
    -DCMAKE_CXX_COMPILER=clang++
```

如果你确实想复用原目录，可以删除整个构建目录，重新配置。

记住：

不要随意修改 `CMakeCache.txt` 来解决所有问题。它是缓存，不是你应该手动维护的项目配置文件。

---

# 第五部分：再补充一些非常常用的 CMake 功能

你已经能够创建项目、管理 Target、链接库、组织多目录了。

下面这些是你在真正做项目时很容易遇到的功能。

## 24. 添加测试：CTest

CMake 生态中还有一个工具叫 CTest。

它负责测试执行和测试结果管理。

假设你有：

```text
MyProject/
├── CMakeLists.txt
├── src/
│   ├── math.cpp
│   └── main.cpp
└── tests/
    └── test_math.cpp
```

根目录可以这样写：

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)

add_library(math STATIC src/math.cpp)

target_include_directories(math PUBLIC include)

add_executable(my_app src/main.cpp)

target_link_libraries(my_app PRIVATE math)

include(CTest)

if(BUILD_TESTING)

    add_executable(test_math tests/test_math.cpp)

    target_link_libraries(test_math PRIVATE math)

    add_test(
        NAME math_basic
        COMMAND test_math
    )

endif()
```

其中：

```cmake
include(CTest)
```

会引入 CTest 的测试支持，并提供 `BUILD_TESTING` 选项。

```cmake
add_test(
    NAME math_basic
    COMMAND test_math
)
```

注册一个测试。

构建完成之后：

```bash
ctest --test-dir build
```

就可以运行测试。

这对你后续写 HTTP Server、交易系统尤其有用。

例如：

- 测试网络协议解析是否正确。
- 测试订单簿的撮合逻辑。
- 测试并发访问是否正确。
- 测试边界条件。
- 做回归测试，避免修改代码后破坏旧功能。

注意，CTest 负责组织和运行测试，不是测试框架本身。你可以结合 GoogleTest、Catch2 等框架使用。

## 25. 自己安装的第三方库：CMAKE_PREFIX_PATH

假设你下载了一个第三方库，自己编译安装到了：

```text
/home/user/mylibs/
```

里面有：

```text
/home/user/mylibs/
├── include/
└── lib/
```

如果这个库提供了 CMake 的包配置文件，你可以通过：

```bash
cmake -S . -B build \
    -DCMAKE_PREFIX_PATH=/home/user/mylibs
```

告诉 CMake：

寻找第三方依赖时，也到这个安装前缀中查找。

它并不代表所有库都会因此自动被找到。

具体还取决于第三方库有没有提供相应的 CMake Package Config，以及安装目录结构是否符合查找规则。

这是你以后使用 OpenSSL、fmt 等库时可能碰到的情况。

## 26. FetchContent：自动获取第三方源码

有时候你不想让使用者自己先安装第三方库。

你希望：

> 构建我的项目时，CMake 自动下载某个第三方库，并把它加入构建体系。

可以了解：

```cmake
include(FetchContent)

FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 11.1.4
)

FetchContent_MakeAvailable(fmt)
```

之后：

```cmake
target_link_libraries(my_app PRIVATE
    fmt::fmt
)
```

这样就可以使用 fmt 库。

这里我只是演示语法，版本号应当根据你实际项目选择，并且正式项目最好固定可靠的版本或提交。

注意 FetchContent 的行为和 `find_package()` 不一样：

- `find_package()`：主要是查找已有的包。
- `FetchContent`：获取第三方项目源码，并让它参与当前构建。

你以后做自己的 HTTP Server 或交易 Demo 时，这个功能很有用。

但不建议你一开始就把所有第三方库都交给 FetchContent 管理。你还需要理解版本固定、网络失败、依赖冲突等问题。

## 27. install()：安装你的库和程序

假设你开发了一个静态库，希望其他项目也能使用。

你不仅要生成 `.a` 文件，还要提供：

- 库文件。
- 公开头文件。
- 其他必要的使用信息。

可以学习：

```cmake
install(TARGETS my_app
    RUNTIME DESTINATION bin
)

install(TARGETS math
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)

install(DIRECTORY include/
    DESTINATION include
)
```

然后：

```bash
cmake --install build --prefix install
```

就可以将构建产物安装到指定目录。

这就涉及软件库的分发、安装和复用。

如果你以后要把自己的 C++ 网络库发布出来，甚至让其他项目通过：

```cmake
find_package(MyLibrary REQUIRED)
```

找到它，那么还要继续学习：

- `install(EXPORT ...)`
- `install(TARGETS ... EXPORT ...)`
- `configure_package_config_file()`
- `write_basic_package_version_file()`

这些属于更深入的 CMake 包管理知识，初期不用全部掌握。

---

# 第六部分：从 VSCode 迁移到 CLion，你应该怎么做？

现在终于可以聊 CLion 了。

我先说结论：

**你完全可以现在就开始使用 CLion，不需要等到 CMake 学完才迁移。实际上，边使用 CLion、边学习 CMake，是很好的学习方式。**

而且，你不需要为了使用 CLion 而先把 CMake 学到很深。

## 28. CLion、CMake、编译器之间究竟是什么关系？

先把三者分开：

| 工具 | 职责 |
|---|---|
| CLion | IDE，提供代码编辑、代码分析、跳转、调试、运行等能力 |
| CMake | 项目构建描述和构建系统生成工具 |
| GCC / Clang / MSVC | 真正编译 C/C++ 代码的工具链 |

例如：

你在 CLion 里点击运行按钮。

实际上发生的是：

```text
你点击 CLion 的运行按钮
          │
          ▼
CLion 检查当前项目的构建配置
          │
          ▼
调用 CMake 完成必要的配置
          │
          ▼
CMake 生成或更新底层构建文件
          │
          ▼
CLion 触发构建
          │
          ▼
Ninja / Make / MSBuild
          │
          ▼
GCC / Clang / MSVC
          │
          ▼
生成可执行文件
          │
          ▼
CLion 启动程序
```

所以：

- CLion 不会取代 CMake。
- CMake 不会取代 GCC。
- GCC 不会取代 CLion。

它们各自负责不同的事情。

## 29. CLion 是不是必须自己安装 CMake？

不一定。

CLion 自带 CMake，你可以直接使用它提供的版本，也可以配置其他 CMake 安装。

但有个容易忽略的地方：

**CLion 自带 CMake，不代表它会自动为你准备好所有 C/C++ 编译器和工具链。**

你仍然需要有可用的编译环境。

例如：

| 环境 | 可以使用的编译器 |
|---|---|
| Windows | MinGW GCC、MSVC、Clang |
| Linux | GCC、Clang |
| macOS | Apple Clang |

Windows 下还可以考虑 WSL。

如果你未来主要学习 Linux 网络编程、Redis、Nginx、Linux 系统编程，我个人建议你逐渐熟悉 Linux 环境，特别是 GCC、GDB、CMake、Ninja 这一套。

你现在已经学过 CSAPP 和一些进程、文件描述符、网络相关内容，这些知识放在 Linux 环境下实践会更自然。

CLion 官方文档也明确支持 CMake、Makefile 和 Compilation Database 等项目格式，不是所有项目都必须使用 CMake。（[CLion Quick start guide](https://www.jetbrains.com/help/clion/clion-quick-start-guide.html)；[CLion Projects](https://www.jetbrains.com/help/clion/project.html)）

## 30. CLion 中的 CMake Profile 是什么？

**非常重要**

CLion 有一个概念叫：

**CMake Profile（CMake 配置文件）。**

它是你在 CLion 中配置项目构建环境的地方。

你可以在：

```text
Settings
    ↓
Build, Execution, Deployment
    ↓
CMake
```

找到它。

一个 Profile 可以配置：

- Toolchain：使用哪个编译器工具链。
- Build Type：Debug、Release 等。
- Generator：Ninja、Makefiles 等。
- CMake Options：传给 CMake 的参数。
- Environment：构建环境变量。
- Build Directory：构建目录。

官方文档中也明确将这些设置归纳到 CMake Profile 中。（[CLion CMake Profiles](https://www.jetbrains.com/help/clion/cmake-profile.html)；[CLion CMake 配置文件](https://www.jetbrains.com/zh-cn/help/clion/cmake-profile.html)）

例如，你可以配置两个 Profile：

**Debug Profile**

开发调试

| Toolchain | GCC |
|---|---|
| Generator | Ninja |
| Build Type | Debug |
| Build Directory | `cmake-build-debug` |

---

**Release Profile**

性能测试

| Toolchain | GCC |
|---|---|
| Generator | Ninja |
| Build Type | Release |
| Build Directory | `cmake-build-release` |

这样你就能很方便地切换开发和发布配置。

CLion 也会根据项目的 CMake Target，为你提供对应的运行和调试配置。

## 31. 你实际迁移时应该怎么操作？

假设你有一个已经写好的 C++ 项目：

```text
MyProject/
├── main.cpp
├── math.cpp
└── math.h
```

### 第一步：给项目增加 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)

project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(my_app
    main.cpp
    math.cpp
)
```

就这么简单。

### 第二步：用 CLion 打开项目

直接：

```text
File → Open
```

选择 `MyProject` 目录。

CLion 会识别顶层的 `CMakeLists.txt`，然后加载项目。

### 第三步：配置 Toolchain 和 Profile

确认：

- CMake 版本可用。
- GCC 或 Clang 可用。
- Ninja 或 Make 可用。
- 选择 Debug Profile。

### 第四步：运行

CLion 会根据 CMake 提供的 Target 信息建立项目模型。

你可以选择 `my_app`，然后运行、调试。

如果你增加了一个新的 `.cpp` 文件：

```text
network.cpp
```

不要忘记把它加入对应 Target：

```cmake
target_sources(my_app PRIVATE
    network.cpp
)
```

或者直接修改：

```cmake
add_executable(my_app
    main.cpp
    math.cpp
    network.cpp
)
```

这是一个很重要的区别：

**文件出现在 CLion 的项目目录树里，不代表它已经加入了某个 CMake Target。**

如果没有加入 Target，它就不会自动参与该程序的构建。

CLion 也提供了将文件加入 CMake Target 的相关操作。（[CLion Manage CMake project files](https://www.jetbrains.com/help/clion/managing-cmake-project-files.html)；[CLion CMakeLists.txt](https://www.jetbrains.com/help/clion/cmakelists-txt-file.html)）

---

# 第七部分：你以后学习大型 C++ 项目时，还需要知道什么？

这部分我想结合你提到的 Redis、Nginx、低延时交易 Demo、HTTP Server 来讲。

因为你现在的目标并不只是学会 CMake，而是希望能够真正阅读、构建、开发大型 C/C++ 项目。

## 32. 一个非常重要的事实：不是所有 C/C++ 项目都使用 CMake

这点千万不要误解。

CMake 虽然非常普及，但它不是 C/C++ 项目的强制标准。

例如：

- 有些项目使用 CMake。
- 有些项目使用 Makefile。
- 有些项目使用 Autotools。
- 有些项目有自己定制的构建脚本。
- 有些项目使用 Meson。

因此：

**你学会 CMake，不意味着你以后看到任何 C++ 项目，都能直接运行 `cmake -S . -B build`。**

比如你未来阅读 Redis、Nginx 源码，就应该先看它们各自仓库的 README 和构建文档。

如果项目没有 CMakeLists.txt，就不能假设它是 CMake 项目。

但这并不意味着 CLion 不能使用。

CLion 也支持 Makefile 和 Compilation Database 等项目格式。（[CLion Quick start guide](https://www.jetbrains.com/help/clion/clion-quick-start-guide.html)；[CLion Projects](https://www.jetbrains.com/help/clion/project.html)）

所以我建议你：

- 自己开发的新项目：优先使用 CMake。
- 阅读别人的项目：尊重原项目的构建系统。
- 不要为了使用 CMake，强行把所有项目改造成 CMake 项目。

## 33. 你的低延时交易 Demo，CMake 可能会是什么样子？

假设你未来做一个简化的交易系统：

```text
TradingSystem/
│
├── CMakeLists.txt
│
├── src/
│   ├── main.cpp
│   │
│   ├── network/
│   │   ├── tcp_server.cpp
│   │   └── protocol.cpp
│   │
│   ├── matching/
│   │   ├── order_book.cpp
│   │   └── matching_engine.cpp
│   │
│   └── utils/
│       ├── logger.cpp
│       └── timer.cpp
│
├── include/
│   ├── network/
│   ├── matching/
│   └── utils/
│
├── tests/
│
└── benchmarks/
```

你可能会创建这些 Target：

```cmake
add_library(network STATIC
    src/network/tcp_server.cpp
    src/network/protocol.cpp
)

add_library(matching STATIC
    src/matching/order_book.cpp
    src/matching/matching_engine.cpp
)

add_library(utils STATIC
    src/utils/logger.cpp
    src/utils/timer.cpp
)

add_executable(trading_server
    src/main.cpp
)

target_link_libraries(trading_server PRIVATE
    network
    matching
    utils
)
```

实际项目还需要进一步设计模块之间的接口和依赖方向。

例如：

- 网络模块负责收发数据。
- 协议模块负责解析请求。
- Matching Engine 负责订单撮合。
- Order Book 负责维护订单簿。
- Utils 提供日志和计时功能。

每个模块都可以成为独立的 Target。

这样做有什么好处？

假设你只修改了订单簿：

```text
matching/order_book.cpp
```

构建系统就可以根据依赖关系，尽可能只重新构建必要的部分。

而不是每次都手动把所有 `.cpp` 文件重新编译一遍。

当然，实际是否需要重新链接最终程序，还要看 Target 之间的依赖关系。

### 33.1 低延时项目还需要学习哪些构建相关知识？

这时就不是简单的：

```cmake
set(CMAKE_CXX_STANDARD 20)
```

了。

你还会遇到：

**第一，编译优化。**

例如：

```text
-O2
-O3
-flto
```

LTO（Link Time Optimization）会涉及编译和链接阶段的配合。

**第二，CPU 架构优化。**

例如：

```text
-march=native
```

但这个参数有可移植性问题：在当前 CPU 上编译出来的程序，未必能在其他 CPU 上运行。

**第三，调试与性能分析。**

你可能需要：

- Debug。
- Release。
- RelWithDebInfo。
- AddressSanitizer。
- UndefinedBehaviorSanitizer。
- ThreadSanitizer。
- Benchmark。

**第四，不同模块使用不同编译选项。**

例如：

你可能不希望测试程序和正式交易程序使用完全相同的编译选项。

这时候就需要熟悉 Target 级别的编译配置，以及 CMake 的生成器表达式。

不过，我不建议你现在就深入研究这些。

先掌握基本的 Target 模型，之后自然会遇到。

---

# 第八部分：你必须知道的常见误区

我把初学者最容易犯的错误整理在这里。建议你以后遇到问题时回来对照。

## 34. 十个容易踩的坑

**误区 1：CMake 就是编译器。**

错。

CMake 主要负责配置项目、生成构建规则。真正编译代码的是 GCC、Clang、MSVC 等。

**误区 2：CMakeLists.txt 和 Makefile 是一回事。**

错。

CMakeLists.txt 是 CMake 的输入文件，Makefile 是 Make 使用的构建规则文件。

CMake 可以生成 Makefile。

**误区 3：把文件放进项目目录，CMake 就会自动编译它。**

错。

你必须将源文件加入某个 Target，例如：

```cmake
add_executable(app main.cpp network.cpp)
```

或者：

```cmake
target_sources(app PRIVATE network.cpp)
```

**误区 4：头文件搜索路径和链接库是同一回事。**

错。

```cmake
target_include_directories(...)
```

负责头文件搜索目录。

```cmake
target_link_libraries(...)
```

负责声明链接依赖以及相关使用要求。

两者不是一回事。

**误区 5：头文件必须放在 add_executable() 里。**

错。

只要编译器能通过 `#include` 找到头文件，就可以使用它。

把头文件列入 Target 通常是为了项目组织和 IDE 展示，不代表它需要单独编译。

**误区 6：PUBLIC 就是给其他人公开源代码。**

错。

它描述的是 Target 的使用要求是否向消费者传递。

和源代码是否公开没有直接关系。

**误区 7：每次执行 cmake 都会重新编译整个项目。**

错。

配置和生成，与真正的编译构建是不同阶段。

CMake 和底层构建工具会利用依赖关系支持增量构建。

**误区 8：所有生成器都能用 CMAKE_BUILD_TYPE。**

错。

它主要用于单配置生成器。Visual Studio、Xcode 等多配置生成器通常在构建阶段通过 `--config` 选择配置。

**误区 9：所有 CMake 项目都应该把编译参数写到 CMAKE_CXX_FLAGS。**

不推荐。

更好的方式通常是：

```cmake
target_compile_options(my_app PRIVATE
    -Wall
    -Wextra
)
```

让配置作用于具体 Target，而不是污染整个项目。

**误区 10：CMakeLists.txt 应该自动搜索所有 cpp 文件。**

初学时经常有人写：

```cmake
file(GLOB SOURCES "*.cpp")
```

然后：

```cmake
add_executable(app ${SOURCES})
```

看起来很方便。

但我建议你初期使用显式源文件列表：

```cmake
add_executable(app
    main.cpp
    math.cpp
    network.cpp
)
```

原因是，普通的 `GLOB` 不一定能在新增文件时自动触发 CMake 重新配置。

虽然 CMake 提供了 `CONFIGURE_DEPENDS` 选项解决部分问题，但对于一般项目，显式列出源文件更清楚、更可靠。

---

# 第九部分：什么时候这些基础语法不够用了？

你最后特别提到，希望我告诉你：

> 在什么情况下，常见 CMake 语法不够用，我还需要学习什么？

这个问题非常好。

我给你整理一个分级学习路线。

## 35. CMake 知识体系地图

**Level 1 · 现在必须掌握**

**基础构建与 Target 管理**

- `cmake_minimum_required()`
- `project()`
- `set()`
- `add_executable()`
- `add_library()`
- `target_sources()`
- `target_include_directories()`
- `target_link_libraries()`
- `target_compile_options()`
- `target_compile_definitions()`
- `add_subdirectory()`
- Debug / Release
- Configure / Generate / Build
- Source Tree / Build Tree

---

**Level 2 · 开始做真正的项目**

**依赖管理与工程组织**

- `find_package()`
- Imported Targets
- `PUBLIC` / `PRIVATE` / `INTERFACE`
- `option()`
- `FetchContent`
- CTest
- GoogleTest / Catch2 集成
- `install()`
- CMakePresets
- `CMAKE_PREFIX_PATH`

---

**Level 3 · 中大型项目**

**高级构建配置**

- Generator Expressions
- Toolchain Files
- Cross Compilation
- `configure_file()`
- `add_custom_command()`
- `add_custom_target()`
- `ExternalProject`
- `install(EXPORT ...)`
- Package Config
- PCH（预编译头）
- Unity Build

---

**Level 4 · 特定领域**

**复杂工程与性能构建**

- C++20 Modules
- CUDA / HIP
- 多平台交叉编译
- 复杂的第三方依赖管理
- LTO
- Sanitizers
- 编译缓存
- 大型项目的增量构建优化

我逐个解释几个你未来很可能会遇到的东西。

### 35.1 Generator Expressions（生成器表达式）

你可能会遇到：

```cmake
target_compile_definitions(my_app PRIVATE
    $<$<CONFIG:Debug>:DEBUG_MODE>
)
```

这是什么意思？

意思是：

如果当前构建配置为 Debug，就给 `my_app` 定义 `DEBUG_MODE`。

如果是 Release，就不定义。

为什么需要这种东西？

因为有些配置不是在 CMake Configure 阶段就能完全确定的，而是需要在生成构建规则或实际构建时根据条件决定。

生成器表达式就是为这种情况设计的。

它的语法看起来很奇怪：

```cmake
$<...>
```

你现在不用深入学习，但当你开始管理多个构建配置、不同编译器、复杂 Target 使用要求时，就应该学它。

### 35.2 Toolchain File（工具链文件）

假设：

你在 x86 Linux 机器上开发，但是希望编译出 ARM Linux 上运行的程序。

这叫交叉编译（Cross Compilation）。

这时就不能简单地让 CMake 使用当前系统默认的 GCC。

你需要告诉它：

- 使用哪个交叉编译器。
- 使用哪个目标系统。
- 使用哪个 SDK。
- 去哪里寻找目标平台的库和头文件。

例如：

```cmake
set(CMAKE_SYSTEM_NAME Linux)

set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)

set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
```

实际项目一般把这些设置放进独立的 Toolchain File，再通过：

```bash
cmake -S . -B build \
    -DCMAKE_TOOLCHAIN_FILE=toolchain.cmake
```

指定。

这在嵌入式开发、ARM 服务器、跨平台软件开发中很常见。

### 35.3 add_custom_command()

假设你的项目需要在编译前自动生成某些代码。

例如：

- 根据 `.proto` 文件生成 C++ 代码。
- 根据接口描述文件生成代码。
- 根据资源文件生成 C++ 头文件。
- 使用脚本自动生成版本信息。

你可能会使用：

```cmake
add_custom_command()
```

它可以定义自定义构建命令，并且声明输入输出依赖。

这时你已经不只是管理普通的 C++ 编译了，而是在管理更加复杂的构建流水线。

### 35.4 CMakePresets.json

这个我非常建议你后续学习。

假设你和朋友共同开发一个项目。

你使用：

```bash
cmake -S . -B build-debug \
    -DCMAKE_BUILD_TYPE=Debug
```

他使用：

```bash
cmake -S . -B build \
    -DCMAKE_BUILD_TYPE=Debug
```

另一个人使用不同的编译器。

每个人都要自己记忆一堆配置参数。

这不太方便。

于是 CMake 提供了：

```text
CMakePresets.json
```

可以把常用配置保存到项目中。

例如：

```json
{
  "version": 2,
  "configurePresets": [
    {
      "name": "debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug"
      }
    },
    {
      "name": "release",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/release",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release"
      }
    }
  ]
}
```

然后：

```bash
cmake --preset debug
```

或者：

```bash
cmake --preset release
```

就可以使用预先定义好的配置。

这样不同开发者、CI 环境就能更容易复用统一的构建配置。

注意，Preset 还有 Build Preset、Test Preset 等概念，以后再学。

---

# 第十部分：给你的实际学习建议

最后我想说说，你究竟应该把多少时间花在 CMake 上。

我不建议你把 CMake 学成一个独立的研究方向。

你真正的目标是 C/C++ 项目开发，CMake 是为这个目标服务的工具。

尤其是你后续想做：

- HTTP Server。
- Redis / Nginx 源码阅读。
- 低延时交易 Demo。
- 多线程网络服务。
- 高吞吐 C++ 项目。

这些项目会让你自然接触到更复杂的构建需求。

## 36. 我建议你的学习顺序

1. **第一阶段：花 1～2 天建立 CMake 基础认知**

   你现在读完这篇后，应该重点掌握：

   - CMake 和 Make 的关系。
   - Configure、Generate、Build。
   - Target。
   - `add_executable()`、`add_library()`。
   - `target_include_directories()`。
   - `target_link_libraries()`。
   - `PUBLIC`、`PRIVATE`、`INTERFACE`。
   - 多目录项目。

2. **第二阶段：花 1～2 天，迁移到 CLion**

   - 配置 GCC 或 Clang。
   - 配置 CMake。
   - 配置 Ninja。
   - 创建一个 C++20 项目。
   - 创建一个静态库。
   - 创建一个可执行程序。
   - 配置 Debug 和 Release。
   - 使用 CLion 断点调试。

   不需要一开始就学高级 CMake。

3. **第三阶段：结合真正的项目学习依赖管理**

   自己写一个小型 HTTP Server。

   尝试集成：

   - Threads。
   - OpenSSL。
   - fmt。
   - GoogleTest。

   练习 `find_package()`、Imported Targets、FetchContent、CTest。

4. **第四阶段：阅读大型开源项目**

   阅读 Redis、Nginx 等源码时，先理解它们原本的构建方式。

   尝试：

   - 配置构建环境。
   - 编译源码。
   - 使用 CLion 阅读和跳转。
   - 运行调试。
   - 理解第三方依赖。

5. **第五阶段：在低延时 C++ 项目中深入构建优化**

   这时候再研究：

   - Release 构建。
   - 编译器优化选项。
   - LTO。
   - Sanitizers。
   - Benchmark。
   - 编译缓存。
   - CMakePresets。
   - Generator Expressions。

## 37. 最后，给你一份可以直接实践的检查清单

我建议你先别继续找其他 CMake 教程了，先把下面这些操作亲自完成。

**CMake 入门完成度**

**0%**

```text
[---------------] 0%
```

0 / 15 项已掌握

---

- [ ] 理解 CMake、Make、Makefile 的区别
- [ ] 能够独立编写最简单的 CMakeLists.txt
- [ ] 能够创建 Executable Target
- [ ] 能够创建 Static Library Target
- [ ] 理解 PRIVATE / PUBLIC / INTERFACE
- [ ] 能够配置头文件搜索目录
- [ ] 能够使用 target_link_libraries 链接库
- [ ] 能够使用 add_subdirectory 组织多目录项目
- [ ] 能够区分 Configure / Generate / Build
- [ ] 能够使用 cmake -S . -B build
- [ ] 能够使用 cmake --build build
- [ ] 能够切换 Debug 和 Release
- [ ] 能够使用 find_package 查找 Threads
- [ ] 能够在 CLion 中打开并调试 CMake 项目
- [ ] 能够解释 CMakeCache.txt 和 Build Directory 的作用

**重置进度**

> Markdown 版操作说明：勾选上述清单记录掌握情况；完成度按“已勾选项数 ÷ 15 × 100%”四舍五入计算。普通 Markdown 不会自动更新进度；重置时取消所有勾选，并将完成度和已掌握项数改回 0。

---

## 最后总结：你真正应该记住的 CMake 思想

如果这篇文章只能让你记住几个东西，我希望是下面这些：

**第一，CMake 是构建系统生成器，不是编译器。**

它把平台无关的项目描述转换成 Make、Ninja、Visual Studio 等工具能够理解的构建规则。

**第二，现代 CMake 的核心是 Target。**

不要总想着怎么拼接 g++ 命令，而要思考：

> 我的项目有哪些 Target？它们之间有什么依赖？每个 Target 需要什么编译配置和使用要求？

**第三，PUBLIC、PRIVATE、INTERFACE 是依赖传递机制的关键。**

理解这三个关键字，你就理解了现代 CMake 很重要的一部分设计。

**第四，Configure、Generate、Build 是三个不同阶段。**

`cmake -S . -B build` 和 `cmake --build build` 不是一回事。

**第五，源码目录和构建目录应该分离。**

一个 Source Tree 可以对应多个 Build Tree，分别使用不同编译器、不同构建配置。

**第六，CMake 是工程工具，不是学习目标本身。**

你不需要在开始写 HTTP Server 之前就学完所有 CMake。

先掌握基础，遇到实际需求再深入。

我认为你现在最值得做的一件事情是：**使用 CLion，亲手从零创建一个包含一个静态库、一个可执行程序、多个源文件和多个目录的 C++20 项目。**

当你能不看教程，独立完成它的 `CMakeLists.txt`，并且能够解释每一条命令为什么存在时，你就已经真正跨过 CMake 入门这道坎了。

官方学习资料我也给你留几个，后续遇到问题可以直接查：

- [CMake 官方 Getting Started Tutorial](https://cmake.org/cmake/help/git-stage/guide/tutorial/Getting%20Started%20with%20CMake.html)：循序渐进的官方教程。
- [CMake 官方历史介绍](https://cmake.org/history/)：了解设计背景。
- [About CMake](https://cmake.org/about/)：了解 CMake 的整体能力。
- [CLion Quick CMake Tutorial](https://www.jetbrains.com/help/clion/quick-cmake-tutorial.html)：CLion 实际操作教程。
- [CLion CMake Profiles](https://www.jetbrains.com/help/clion/cmake-profile.html)：了解 CLion 的构建配置。

**我的建议是：先把这篇文章里的第 21 节实战完整做出来，然后就可以正式开始你的 C++ 项目开发了。**
