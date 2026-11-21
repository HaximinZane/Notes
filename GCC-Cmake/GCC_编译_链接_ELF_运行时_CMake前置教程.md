# 从 `g++ main.cpp` 到进程启动：GCC、链接器、ELF、静态/动态库与运行时全链路

> **定位**：现代 C/C++ 工程与 CMake 的前置教材。读者已大致了解 `.cpp`、头文件、编译与链接的区别，但希望真正读懂 `g++` 命令行、CSAPP Linking 课件，以及 CMake 为什么要管理 targets、libraries 和 usage requirements。
>
> **平台基准**：重点讨论 **Linux + GCC/G++ + GNU binutils + ELF + glibc**，默认 x86-64、System V ABI。Windows/MSVC/PE 与 macOS/Clang/Mach-O 只作对照，不把 Linux 的行为生硬套过去。
>
> **课件基准**：CMU CSAPP 15-213，*Linking*，2015-10-13，49 页（你提供的 `13-linking.pdf`）。正文会注明相关页码；并明确区分“课件当年的简化解释”和“现代工具链更准确的行为”。
>
> **使用方式**：先读第 0～9 章建立因果链；第 10～15 章加深对动态链接、运行和 C++ 的理解；之后做实验，最后阅读 CMake 对照表。本教程的目标不是让你背命令，而是让你看到任意构建命令都知道它正在指挥哪个阶段，以及出问题时应该检查哪一层。

---

## 目录

- [0. 先给你整条路线图](#0-先给你整条路线图)
- [1. `gcc` / `g++` 到底是什么](#1-gcc--g-到底是什么)
- [2. 四大阶段：预处理、编译、汇编、链接](#2-四大阶段预处理编译汇编链接)
- [3. 头文件、翻译单元、独立编译与增量构建](#3-头文件翻译单元独立编译与增量构建)
- [4. GCC/G++ 常见参数：不只是记语法](#4-gccg-常见参数不只是记语法)
- [5. 进阶参数：优化、调试、安全、性能和工程化](#5-进阶参数优化调试安全性能和工程化)
- [6. `.o` 究竟是什么：ELF、节、段、符号表](#6-o-究竟是什么elf节段符号表)
- [7. 链接的第一件事：符号解析](#7-链接的第一件事符号解析)
- [8. 链接的第二件事：重定位](#8-链接的第二件事重定位)
- [9. 静态库 `.a`：归档、按需抽取与顺序陷阱](#9-静态库-a归档按需抽取与顺序陷阱)
- [10. 动态库 `.so`：链接、装载与运行时查找](#10-动态库-so链接装载与运行时查找)
- [11. PIC、PIE、GOT、PLT 与 ASLR](#11-picpiegotplt-与-aslr)
- [12. `./app` 后到底发生什么：从 `execve` 到 `main`](#12-app-后到底发生什么从-execve-到-main)
- [13. C++ 比 C 多出来的链接复杂性](#13-c-比-c-多出来的链接复杂性)
- [14. 课件最后的案例：三种函数拦截](#14-课件最后的案例三种函数拦截)
- [15. 把 GCC 与链接知识一一映射到 CMake](#15-把-gcc-与链接知识一一映射到-cmake)
- [16. 值得亲手做的实验](#16-值得亲手做的实验)
- [17. 构建报错的分层排查手册](#17-构建报错的分层排查手册)
- [18. 工具与命令速查表](#18-工具与命令速查表)
- [19. CSAPP 49 页课件的阅读索引和时代差异](#19-csapp-49-页课件的阅读索引和时代差异)
- [20. 阅读 CMake 前的最终检查](#20-阅读-cmake-前的最终检查)
- [21. 选修进阶：真正的大型系统工程还会遇到什么？](#21-选修进阶真正的大型系统工程还会遇到什么)
- [22. 把最后那些最容易绕晕的问题一次问透](#22-把最后那些最容易绕晕的问题一次问透)

---

# 0. 先给你整条路线图

有一天你写了这条命令：

```bash
g++ -std=c++20 -O2 -g -Wall -Iinclude src/main.cpp src/math.cpp -o app
```

你可能以为 `g++` 一下子把源代码“编成了程序”。从**命令使用者**看，这样说可以；从**计算机系统**看，这句话隐藏了好几种本质不同的工作。

```text
开发者编写             工具链 / 系统执行
────────────────────────────────────────────────────────────────
main.cpp, math.cpp, math.h
        │
        ├─ 预处理器：展开 #include、宏、条件编译
        ▼
    预处理后的 C++ 翻译单元（逻辑上 .ii）
        │
        ├─ C++ 编译器前端、中端、后端：语法/类型、优化、代码生成
        ▼
    汇编语言（逻辑上 .s）
        │
        ├─ 汇编器 as：机器指令编码，生成节、符号和重定位记录
        ▼
    main.o / math.o（ELF 可重定位目标文件）
        │
        ├─ 链接器 ld / gold / lld / mold：
        │     ① 决定纳入哪些输入文件和静态库成员
        │     ② 符号解析：sum 这个名字对应哪一个定义？
        │     ③ 布局和重定位：调用位置、数据地址怎么填写？
        │     ④ 写出 ELF 可执行文件，记录动态依赖
        ▼
    app（ELF 可执行文件；常见为 PIE）
        │
        ├─ shell 启动：fork/posix_spawn 等，子进程 execve
        ├─ Linux 内核：读 ELF、创建进程映像、映射 PT_LOAD
        ├─ 动态加载器（若需要）：找 .so、映射、完成动态重定位
        ├─ C/C++ 运行时启动代码：初始化环境/运行构造器
        ▼
    main(argc, argv) 开始执行
        │
        ▼
    返回状态，执行清理、终结函数，最终进程退出
```

这里有五类容易混的“地址/位置”：

1. **源码位置**：例如 `math.cpp:17`，方便人和调试器定位。
2. **目标文件内部偏移**：例如 `math.o` 的 `.text` 第 0x20 字节；独立编译时还不知道最终布局。
3. **链接期虚拟地址或相对值**：链接器分配布局，修补已知引用；对于 PIE/DSO，其中部分值仍留给装载时处理。
4. **运行时虚拟地址**：进程的地址空间中映射的位置；可能受 ASLR 影响。
5. **物理内存位置**：操作系统内存管理的另一层，不是链接器一般需要知道的东西。

**一句话抓住链接**：编译器能在不知道其他源文件具体机器码位置的前提下，分别生成 `.o`；链接器把这些独立产物中的**名称关系**和**地址关系**处理成可以执行或进一步动态装载的形式。这正是 CSAPP 第 5～8 页要回答的问题。

另外请先分清三组术语：

| 名词 | 更准确的意思 | 常见误解 |
|---|---|---|
| **编译**（宽义） | 从源码到最终程序的整体活动 | 以为每个语境都只指 C++→汇编 |
| **编译**（窄义） | 预处理后源代码→汇编/内部机器表示 | 以为这一步必然就能运行 |
| **链接** | 组织对象文件/库，解析符号并完成相应重定位 | 以为只是把文件“拼一起” |
| **装载**（loading） | 内核/动态加载器建立可执行映像 | 以为链接器直接把程序放到 RAM |
| **构建**（build） | 决定执行哪些翻译、链接、生成和测试工作 | 以为 Make/Ninja 就是编译器 |

接下来逐层剥开。

---

# 1. `gcc` / `g++` 到底是什么

## 1.1 编译器驱动程序：它不等于一个“只会编译”的二进制

你敲 `gcc hello.c -o hello` 时调用的 `gcc`，最直观地理解是**驱动程序**（compiler driver）。它会根据输入类型与参数，安排真正的工具执行：预处理、编译器前端/后端、汇编器，以及链接器。CSAPP 第 4 页中的 `cpp / cc1 / as → ld` 就是这种分工的教学示意。

Linux 上常见的组件：

- `gcc` / `g++`：用户面对的 GCC 驱动程序。
- `cc1`：GCC 的 C 语言编译器内部组件。
- `cc1plus`：GCC 的 C++ 编译器内部组件。
- `as`：汇编器，汇编代码变成机器码和 ELF `.o`。
- `collect2`：GCC 某些配置下的链接辅助工具，最终会调用链接器。
- `ld`（GNU ld.bfd）、`gold`、`ld.lld`、`mold`：不同实现的**链接器**，不应把每个系统的默认链接器都假定成 GNU `ld.bfd`。
- `ar`、`ranlib`：归档 `.o` 成静态库、维护索引。
- `readelf`、`nm`、`objdump`：查看结果；它们通常不参与普通的编译过程。

驱动程序好比总包工头，`cc1plus` 是进行 C++ 语义分析和代码生成的工人，`as` 负责把汇编指令编码，`ld` 负责跨模块协调。**总包工头自己不会凭空变成所有专业工人**。

亲自观察：

```bash
g++ --version
g++ -v hello.cpp -o hello       # 真正执行，并打印各阶段及路径
g++ -### hello.cpp -o hello     # 展示拟执行的子命令，通常不实际执行
```

`-v` 和 `-###` 都是理解“`g++` 到底帮我调用了哪些组件”的第一手观察工具。输出会很长，别急着逐项背参数，先找 `cc1plus`、`as`、`collect2` 或 `ld` 的位置。

## 1.2 为什么有 gcc 和 g++ 两个命令？

最实用的区别：

- `gcc hello.c`：按扩展名一般作为 C 处理，驱动链接 C 的默认运行时。
- `g++ hello.cpp`：按 C++ 处理，链接时自动安排 C++ 标准库等默认依赖。
- **`gcc hello.cpp` 也能按 `.cpp` 扩展名调用 C++ 前端**；但链接 C++ 程序时，通常不会像 `g++` 那样自动加入同一套 C++ 库，因而经常出现 `std::cout`、`operator new` 等 undefined reference。
- 可以通过 `-x c++` 强制把没有 `.cpp` 扩展名的输入当 C++ 来解释；这也不会自动把 `gcc` 变成 `g++` 的 C++ 链接行为。

例如：

```cpp
// hello.cpp
#include <iostream>
int main() { std::cout << "hello\n"; }
```

```bash
g++ hello.cpp -o hello            # 常见正确选择
gcc hello.cpp -o hello            # 通常会在链接阶段缺少 C++ 运行库相关符号
```

在 C++ 项目里，**最后一步链接一般交给 `g++` 驱动，而不是手工调用 `ld`**。因为它除了组合你的 `.o`，还负责选对启动文件、C++ 运行库、必要的 GCC 支持库、链接器参数等。手工调用 `ld` 很容易漏掉这些隐式依赖。

## 1.3 文件扩展名是约定，不是魔法

常见约定：

| 扩展名 | 含义 |
|---|---|
| `.c` | C 源代码 |
| `.cc`、`.cpp`、`.cxx` | C++ 源代码 |
| `.h`、`.hpp` | 头文件，通常被其他源码包含 |
| `.i`、`.ii` | 预处理后的 C / C++ 输入 |
| `.s` | 汇编源码 |
| `.S` | 通常是需要先预处理的汇编源码 |
| `.o` | 可重定位目标文件（Linux ELF 常见） |
| `.a` | 静态库归档（archive），装着一个或多个 `.o` |
| `.so`、`.so.N` | ELF 共享对象/动态库 |
| 无扩展名、`a.out` | Linux 可执行文件常见命名；真正格式要看 `file` / `readelf` |

**不要误会**：Linux 判断可执行文件类型不是依赖 `.exe` 扩展名；它可以看文件内容（如 ELF 魔数）、脚本 shebang 等。`a.out` 在现代 Linux 中多数只是默认输出**文件名**，不意味着古老的 a.out 二进制格式。CSAPP 第 9 页中的 `(a.out file)` 用的是文件名示例。

---

# 2. 四大阶段：预处理、编译、汇编、链接

用一个足够小、但又真的有跨文件依赖的例子贯穿：

```text
sample/
├── math.h
├── math.cpp
└── main.cpp
```

```cpp
// math.h
#pragma once
int add(int a, int b);
```

```cpp
// math.cpp
#include "math.h"
int add(int a, int b) { return a + b; }
```

```cpp
// main.cpp
#include <iostream>
#include "math.h"
int main() { std::cout << add(2, 3) << '\n'; }
```

在 `sample/` 目录运行。

## 2.1 阶段一：预处理（preprocessing）

```bash
g++ -E main.cpp -o main.ii
```

`-E` = **只做到预处理，停止**。你会得到一个可能很大的 `main.ii`。打开搜索 `int add`，你会发现来自 `math.h` 的声明已经进入这个翻译单元；标准库的许多声明也被带进来了。

预处理器做的主要事情：

1. 处理 `#include`：近似把被包含文件的内容插到这里（实际还要经过条件预处理、宏等规则）。
2. 展开 `#define` 宏：文本/token 层面的替换，不是 C++ 模板。
3. 处理 `#if`、`#ifdef`、`#ifndef` 等条件编译。
4. 处理一些编译器预定义宏，如 `__cplusplus`、`__FILE__`。
5. 记录源文件位置相关信息以辅助后续诊断。

重要问题：**`#include "math.h"` 为什么不等于把 `math.cpp` 的实现编进来？**

因为 `math.h` 只有声明：

```cpp
int add(int, int);
```

它告诉当前编译器“这个函数存在，参数与返回类型是什么”，没有提供机器码。编译 `main.cpp` 时，编译器通常仍然不知道 `add` 的机器码最终放在哪儿，因此留下一个对 `add` 的待解析引用。等 `math.o` 和 `main.o` 参加链接时，链接器才去解决。

如果你在头文件里直接写非 `inline` 的普通外部函数定义，并且多个 `.cpp` 都 `#include` 它，很可能造成**重复定义**。后文用 ODR 详细解释。

## 2.2 阶段二：窄义编译（C++ → 汇编）

```bash
g++ -S -O0 main.cpp -o main.s
```

`-S` = **做到生成汇编代码，停止，不汇编，也不链接**。

编译器在这里负责：词法和语法分析、名称查找、类型检查、模板实例化、优化、生成目标架构指令等。你看到的 `.s` 已经很接近机器执行的指令，只是用人类可读的汇编助记符表示。

为什么汇编中可能见到一个陌生名字，比如 `_Z3addii` 而不是 `add`？因为 **C++ name mangling（名字修饰）**：链接符号要区分重载、作用域、参数类型。`add(int,int)` 与 `add(double,double)` 不能使用同一个普通 C 式符号名。后文展开。

为什么 `-O0` 下有一堆栈操作、`-O2` 下几乎什么都没了？因为优化会内联、删代码、常量传播、重新分配寄存器等。**源码并不是机器码的逐行翻译稿**。

观察：

```bash
g++ -S -O0 -fverbose-asm math.cpp -o math_O0.s
g++ -S -O2 -fverbose-asm math.cpp -o math_O2.s
```

拿一个源文件、同一编译器、不同优化等级做对比，是学习机器码和性能的极佳方法。

## 2.3 阶段三：汇编（`.s` → `.o`）

```bash
g++ -c main.cpp -o main.o
g++ -c math.cpp -o math.o
```

`-c` = **编译/汇编成 `.o`，但不执行最后的链接**。

也可以直接：

```bash
g++ -S math.cpp -o math.s
as math.s -o math_manual.o
```

但手动 `as` 可能受编译器额外选项、调试格式、架构设置影响；平常用 `g++ -c` 更稳妥。

`.o` 文件里有：

- `.text` 里的一部分机器码。
- `.data` / `.rodata` / `.bss` 等数据相关节。
- 供链接器读取的符号表和重定位记录。
- 可能的调试信息、异常展开信息等。

此时 `main.o` 通常已经有调用 `add` 的机器指令，但**还没有完整决定它应跳向哪里**。汇编器会生成重定位记录，告诉链接器：“这里将来要填与 `add` 有关的地址/位移。”

## 2.4 阶段四：链接（多个 `.o` → 可执行文件）

```bash
g++ main.o math.o -o app
./app
```

输出：

```text
5
```

你现在真的运行了一个跨翻译单元程序。这里的链接器至少需要：

1. 发现 `main.o` 引用了 `add`，找到 `math.o` 对应的定义。
2. 决定目标的布局，确定 `main`、`add`、所需库等位置关系。
3. 根据重定位记录修补相关机器码或数据字段。
4. 加入必要的启动代码/运行库关系，生成可执行 ELF。

**注意：链接不一定把 libc 或 libstdc++ 的所有机器码拷进可执行文件**。常规动态链接模式下，文件记录对相应 `.so` 的依赖，运行时再由动态加载器处理。

## 2.5 一口气记住四条命令

```bash
g++ -E main.cpp -o main.ii    # 预处理结果
g++ -S main.cpp -o main.s     # 汇编文本
g++ -c main.cpp -o main.o     # 目标文件，未链接
g++ main.o math.o -o app      # 链接出可执行文件
```

如果你只有一个源码：

```bash
g++ main.cpp -o app
```

不是“没有链接阶段”，而是驱动程序把前面几个阶段和链接阶段**自动串起来**了。

## 2.6 为什么 `g++ -c a.cpp b.cpp -o x.o` 不行？

`-c` 对两个输入通常会产生两个独立对象文件；`-o x.o` 却只给了一个单一输出文件名，GCC 会报错。应写成：

```bash
g++ -c a.cpp b.cpp         # 输出 a.o、b.o
# 或明确分别指定：
g++ -c a.cpp -o a.o
g++ -c b.cpp -o b.o
```

而：

```bash
g++ a.cpp b.cpp -o app
```

这里最终只有一个**链接输出** `app`，所以 `-o app` 合理。这一点能帮助你理解为什么不同阶段的输出文件数量不同。

---

# 3. 头文件、翻译单元、独立编译与增量构建

## 3.1 Translation Unit（翻译单元）究竟是什么？

在一个典型的 C++ 编译过程中，`main.cpp` 与它预处理展开的头文件内容共同构成一个**翻译单元**（TU）。`math.cpp` 是另一个 TU。编译器分别处理它们，产出相应的对象文件。

```text
main.cpp + #include 展开 ──> TU A ──> main.o
math.cpp + #include 展开 ──> TU B ──> math.o
                                  │
                      linker <────┘
```

这样就有个意义重大的结论：**编译 main.cpp 的时候，编译器通常看不到 math.cpp 的函数体**。它看到的是声明，因此既能在本翻译单元中检查调用参数，也可以让未解析的跨模块引用暂时存在于 `.o` 中。链接才负责找到它的定义。

（补充：LTO 等技术可能让编译器在链接期跨 TU 做更多优化，但这不是普通独立编译的默认工作模型。）

## 3.2 为什么头文件改了也可能引发多个 `.cpp` 重新编译？

因为多个 TU 都 `#include` 它。改 `math.h` 可能改变 `main.cpp` 和 `math.cpp` 实际参与编译的内容；两者都需要重新编译。

这不属于链接器主动“读取源码变更”的职责，而是 **Make/Ninja 等构建工具依据依赖关系决定谁要重新编译**。

```text
math.h ───> main.cpp 编译 ───> main.o ──┐
      └───> math.cpp 编译 ───> math.o ──┴──> app 链接
```

修改 `math.cpp`：通常只重新编译 `math.cpp` 并重新链接 `app`。

修改 `math.h`：两个包含它的 TU 都可能需要重新编译，然后链接。

这解释了 CSAPP 第 5～6 页的 **modularity** 和 **time efficiency**，也解释了为什么你之后需要 CMake + Ninja 来管理“文件之间的依赖图”。

## 3.3 `-I` 的目的是什么？

头文件在独立文件夹：

```text
project/
├── include/math/math.h
└── src/main.cpp
```

源码：

```cpp
#include "math/math.h"
```

命令：

```bash
g++ -Iinclude -c src/main.cpp -o main.o
```

`-Iinclude` 不是“给链接器一个目录”，而是**告诉预处理器/编译器在查找头文件时增加一个搜索路径**。它影响的是从源码到 `.o` 的过程，不是最后挑哪个 `.a/.so`。

相对地，`-L` 是链接期的**库搜索目录**，后文会详细对比。

关于 `#include "file"` 与 `#include <file>`：一般情况下，前者会优先考虑包含该语句的源文件目录，再按照相应规则搜索；后者通常按配置的系统/指定包含目录规则搜索。具体顺序还涉及 `-iquote`、`-I`、`-isystem` 与编译器实现，不要粗暴记成“一个相对路径、一个绝对路径”。

## 3.4 声明、定义、引用：这三个词不能混

```cpp
extern int counter;        // 声明变量：告诉编译器别处存在它
int counter = 42;          // 定义：为变量提供实际实体/存储
int f(int);                // 函数声明
int f(int x) { return x; } // 函数定义
```

而：

```cpp
int y = f(counter);        // 使用/引用已有名字
```

C 和 C++ 关于某些声明/定义（尤其是文件作用域无初始化变量）的细节不同。链接器最终关注的，是对象文件中**定义了哪些链接符号、引用了哪些外部符号**。

## 3.5 头文件保护、防止“一份头文件被包含两次”

```cpp
#pragma once
```

或者传统方式：

```cpp
#ifndef MATH_H
#define MATH_H
int add(int, int);
#endif
```

它们防止的是**单个翻译单元内部**因重复 `#include` 产生同一头文件的重复声明/定义问题。**它们不负责防止两个不同 `.cpp` 各自包含同一个非 inline 函数定义而产生的链接冲突**。这是许多人的第一个 ODR 陷阱。

## 3.6 为什么模板经常放头文件？

```cpp
// add.hpp
template<class T>
T add(T a, T b) { return a + b; }
```

模板通常在使用时需要有可见的定义才能实例化特定类型的代码。把模板**定义**放在头文件中，使每个使用它的 TU 在需要时都能生成对应实例，是最常见的做法。实现可以在 `.cpp` 中配合**显式实例化**，但就需要管理究竟提供了哪些类型的实例。

这与普通非模板函数“头文件声明、`.cpp` 定义”是不同的工程模型。C++20 modules 又引入了另一套接口与编译依赖机制；它不改变理解 TU、符号与链接的必要性。

---
# 4. GCC/G++ 常见参数：不只是记语法

“参数”这个词的核心不是背缩写，而是问：**这个参数对哪一阶段提出什么要求？如果不提供它，默认会怎样？为什么默认不总适合我？**

## 4.1 按作用阶段分类的全景表

| 参数 | 主要作用 | 阶段 | 为什么需要 |
|---|---|---|---|
| `-E` | 只做预处理 | 预处理 | 排查宏、头文件展开 |
| `-S` | 生成汇编后停止 | 编译 | 查看优化产生的汇编 |
| `-c` | 生成 `.o`，不链接 | 编译/汇编 | 独立编译、多模块增量构建 |
| `-o path` | 指定当前阶段的输出名 | 多阶段 | 避免默认 `a.out`，区分构建产物 |
| `-x c++` | 指定后续输入文件的语言 | 输入识别 | 扩展名不能充分反映语言时使用 |
| `-std=c++20` | 选择语言标准 | 编译 | 控制语法和标准库声明/语言规则 |
| `-Wall` | 开启一组常见警告 | 编译 | 找出可疑代码 |
| `-Wextra` | 更多诊断 | 编译 | 发现额外的可疑写法 |
| `-Wpedantic` | 对非标准扩展诊断 | 编译 | 提升可移植性 |
| `-Werror` | 将警告升级成错误 | 编译 | 维持严格质量门槛 |
| `-g` | 生成调试信息 | 编译/汇编/输出 | GDB/CLion 查看源码位置及变量 |
| `-O0/-Og/-O1/-O2/-O3` | 优化等级 | 编译 | 调试体验、运行速度与编译开销平衡 |
| `-I dir` | 增加头文件搜索目录 | 预处理 | `#include` 能找到项目头文件 |
| `-isystem dir` | 增加系统头文件目录 | 预处理 | 把依赖视为系统头文件，影响警告等 |
| `-DNAME[=VALUE]` | 定义预处理宏 | 预处理 | 按构建模式选择代码分支 |
| `-UNAME` | 取消宏定义 | 预处理 | 控制预处理环境 |
| `-MMD -MP` | 生成用户头文件依赖并生成伪目标 | 编译 | 构建工具能追踪头文件变更 |
| `-L dir` | 增加链接时库搜索目录 | 链接 | 找到 `libfoo.so` / `libfoo.a` |
| `-lfoo` | 请求链接名为 foo 的库 | 链接 | 解决引用并记录适当依赖 |
| `-pthread` | 线程支持相关的编译与链接配置 | 编译+链接 | POSIX 线程程序的推荐选项 |
| `-Wl,<arg>` | 把参数传给链接器 | 链接 | 控制 rpath、map、wrap 等 |
| `-fPIC` | 生成适用于共享库的位置无关代码 | 编译 | 同一库可映射到不同位置 |
| `-shared` | 输出共享对象 | 链接 | 生成 `.so` 而非主程序 |
| `-fPIE` / `-pie` | 生成/链接位置无关可执行文件 | 编译/链接 | 配合 ASLR 与安全性 |
| `-static` | 尽量以静态方式链接 | 链接 | 减少常规共享库依赖，但存在部署限制 |
| `-flto` | 链接期优化 | 编译+链接 | 跨翻译单元优化 |
| `-fsanitize=address` | 内存错误检测插桩和运行库 | 编译+链接 | 发现越界、use-after-free 等 |
| `-v`、`-###` | 展示驱动工作 | 驱动 | 确认最终调用哪些工具 |

### `-o` 到底控制哪个文件？

这个问题很常见。它不是永远输出“可执行文件”。

```bash
g++ -E main.cpp -o main.ii      # 预处理输出
g++ -S main.cpp -o main.s       # 汇编输出
g++ -c main.cpp -o main.o       # 目标文件输出
g++ main.o math.o -o app        # 链接输出
```

**同一个 `-o`，是在命令最终允许达到的阶段上指定主输出文件名。**

## 4.2 编译标准：`-std=c++17`、`-std=c++20`、`gnu++20`

```bash
g++ -std=c++20 main.cpp -o app
```

它主要决定编译器按照哪个 C++ 语言标准接受和解释源代码。例如 concepts、ranges、coroutines 等相关支持需要相应的语言模式与编译器/标准库实现。

`-std=c++20` 和 `-std=gnu++20` 不完全一样：后者允许相应的 GNU 语言扩展。你如果希望做可移植、标准化的 C++ 项目，优先使用标准方言；如果依赖 GNU 扩展，应该主动意识到它的存在。

**额外提醒**：指定 `-std=c++20` 不代表目标系统必然有你需要的所有 C++20 标准库实现。语言编译器、标准库版本、操作系统接口是不同维度。

## 4.3 警告不是错误：`-Wall -Wextra -Wpedantic -Werror`

```bash
g++ -std=c++20 -Wall -Wextra -Wpedantic main.cpp -o app
```

- `-Wall` 名字虽叫 *all*，实际**不是开启 GCC 所有警告**。
- `-Wextra` 补充一批诊断。
- `-Wpedantic` 对不符合所选标准的扩展给出诊断（具体行为以 GCC 文档为准）。
- `-Werror` 让本来只是警告的问题造成编译失败。团队/CI 很有用，但第三方源码或新编译器可能因为新警告导致意外构建失败。
- `-Wconversion`、`-Wsign-conversion` 帮助检查隐式数值转换、符号转换，能发现某些 bug，但也可能产生大量需要甄别的警告。
- `-Wshadow` 检查变量遮蔽；`-Wformat=2` 强化格式字符串诊断；`-Wundef` 检查未定义宏参与 `#if`；`-Wnull-dereference`、`-Wduplicated-cond` 等也可按需开启。
- `-Wno-unused-parameter` 能关闭特定警告；不要把 `-w`（关闭所有警告）当成“消除问题”的方案。

警告选项通常属于**某个 target 编译自己的源文件**的性质，CMake 中更适合 `target_compile_options()` 而非全局污染。

## 4.4 头文件路径与宏：`-I/-iquote/-isystem/-D/-U`

```bash
g++ -std=c++20 -Iinclude -DAPP_DEBUG=1 -c src/main.cpp -o main.o
```

相当于编译前提供宏，允许源码写：

```cpp
#if APP_DEBUG
// 调试分支
#endif
```

但 `-DAPP_DEBUG=1` 是**预处理定义**，不是 C++ 运行时变量；改它意味着代码可能需要重新编译。

`-iquote dir` 针对引号形式的 include 搜索；`-isystem dir` 可用于第三方/系统头文件路径，常使编译器对其中的部分警告处理不同。不要用 `-isystem` 隐藏你自己的项目问题。

`-include config.h` 可强制预先包含指定头文件；`-imacros file` 也可影响预处理宏环境。它们在构建系统和平台适配时有用，但初学尽量少使用魔法配置。

## 4.5 依赖文件：`-MMD -MP -MF`

想象你手写 Makefile：改了 `include/math.h`，Make 如何知道需要重新编译 `src/main.cpp`？编译器能够输出**依赖文件**：

```bash
g++ -Iinclude -MMD -MP -MF main.d -c src/main.cpp -o main.o
```

生成的 `main.d` 可能类似：

```makefile
main.o: src/main.cpp include/math.h
include/math.h:
```

- `-MMD`：生成依赖信息，通常排除系统头文件。
- `-MF main.d`：指定依赖文件的输出路径。
- `-MP`：为头文件添加空目标规则，减少头文件删除/重命名带来的某些 Make 报错。
- `-M`、`-MM`、`-MD`、`-MT`、`-MQ` 是相关选项，可以在需要自写构建规则时再研究。

**重要连接点**：Make/Ninja/CMake 构建系统之所以能处理“头文件改动引起重新编译”，很大程度上依赖编译器提供的依赖信息，而不是自己完全实现一套 C++ 预处理器。

## 4.6 链接参数 `-L` 和 `-l`：两个容易混的 L

```bash
g++ main.o -L./lib -lmath -o app
```

读成两句话：

- `-L./lib`：链接器可以到 `./lib` 寻找库。
- `-lmath`：要求链接名为 `math` 的库，常对应 `libmath.so` 或 `libmath.a`。

注意：

1. `-L` 影响**链接时**找库，通常不代表程序运行时也能找到 `.so`。
2. `-lmath` 不是包含 `math.h`，头文件要用 `-I`。
3. 若同时可选 `.so` 与 `.a`，默认动态链接场景通常优先选择 `.so`，但由平台、模式和链接器选项决定。
4. **库的位置/顺序可能重要**，特别是静态库（第 9 章）。
5. 直接写具体库路径（如 `./lib/libmath.a`）与使用 `-L... -lmath` 在搜索和选择规则上有所区别。

## 4.7 `-Wl,...`：为什么不能直接在 `g++` 后面写所有 ld 参数？

`g++` 驱动和底层 `ld` 解析的选项并不完全一致。例如 GNU ld 有 `--wrap=malloc`，但 `g++` 不一定把陌生选项直接正确交给 ld。

```bash
g++ main.o -Wl,-Map=app.map -o app
```

`-Wl,` 后面每个逗号分隔的片段成为一个链接器参数。所以：

```bash
g++ main.o -Wl,--wrap=malloc -o app
```

是让驱动转交链接器 `--wrap=malloc`。

也可以使用：

```bash
g++ main.o -Xlinker --wrap=malloc -o app
```

如果链接器选项有独立的参数，`-Xlinker` 通常要分别传给参数和它的值。最常用的还是 `-Wl,...`。

对汇编器则可使用 `-Wa,...`，对预处理器可以使用相应 GCC 选项。**识别“这到底是给谁的参数”是阅读构建日志的重要技能**。

## 4.8 线程相关：为什么常建议 `-pthread` 而不是单纯 `-lpthread`？

在支持的平台上：

```bash
g++ -pthread thread.cpp -o thread_demo
```

`-pthread` 是 GCC 驱动级的线程支持选项，可能同时影响预处理/编译设置以及链接设置；而 `-lpthread` 更像是在链接时明确要求一个库。

现代 glibc（例如 2.34 以后）的某些 pthread 实现符号已并入 libc，旧平台经验不应机械套用。即使某台 Linux 机器上“没写 `-pthread` 也能编过去”，工程上仍应通过 CMake 的 `find_package(Threads)` + `Threads::Threads` 明确表达线程需求。

## 4.9 一个错误命令，怎么判断错在哪个阶段？

```bash
g++ -Iinclude -c src/main.cpp -o main.o
```

如果报 `fatal error: math.h: No such file or directory`：头文件搜索/预处理层面的错误。

如果报 `invalid conversion`、`no matching function`：编译器语义检查错误。

如果 `.o` 已生成，执行：

```bash
g++ main.o -o app
```

报 `undefined reference to add(int, int)`：通常**链接阶段**缺少相应定义或符号不兼容。

如果链接成功，但是执行 `./app` 报找不到某个 `.so`：通常是**运行时动态装载**问题。

把问题分层，排错难度会大幅下降。

---

# 5. 进阶参数：优化、调试、安全、性能和工程化

这部分覆盖你未来做 HTTP Server、C++ 系统编程、低延时实验、benchmark 时最可能见到的参数。**不是让你现在全部开启，而是知道它们影响哪一层、有什么代价。**

## 5.1 优化等级：为什么不是永远 `-O3`？

| 选项 | 大致目标 | 典型使用 |
|---|---|---|
| `-O0` | 尽量少做优化，编译较快 | 初学、调试、想对照源码 |
| `-Og` | 开启偏向调试体验的优化 | 调试大型 C/C++ 程序 |
| `-O1` | 较轻量优化 | 特殊构建需求 |
| `-O2` | 较全面优化，通常性能/编译时间平衡较好 | 典型 Release 基线 |
| `-O3` | 在 `-O2` 基础上开启更多激进优化 | 需要 benchmark 证明收益的场景 |
| `-Os` | 倾向优化代码体积 | 体积敏感项目 |
| `-Oz` | 若编译器版本/目标支持，进一步偏向体积 | 极致体积优化，先确认版本 |
| `-Ofast` | 更激进且可能放宽部分标准/数值语义 | **必须评估正确性，不是免费加速按钮** |

可能出现的情况：`-O3` 生成更大代码，给 instruction cache 带来压力；某些微基准更快，实际服务端更慢。**低延时优化要以可重复的 benchmark、真实负载、性能分析为依据**，而不是“参数越多越快”。

浮点领域还会看到 `-ffast-math`：它允许不严格遵循一些 IEEE 754 与标准语言语义的变换。数值正确性敏感、要求 NaN/Inf/舍入语义的程序不要随手开启。

## 5.2 `-g`：到底往文件里加了什么？

```bash
g++ -O0 -g main.cpp -o app
```

`-g` 让编译器生成调试信息，Linux ELF 常见为 DWARF 格式的 `.debug_*` 节。调试器可以借这些信息做源码行映射、变量/类型信息还原等。

重要纠正：

- **`-g` 不代表关闭优化**：`-O2 -g` 完全合法，发布排查性能问题时很常见。
- **调试信息不是 CPU 直接执行的普通 `.text` 指令**；它主要供调试工具使用。
- **优化会影响调试体验**：变量可能显示 `<optimized out>`，函数可能被内联，单步执行顺序未必对应源代码顺序。
- `-g0` 可以显式关闭调试信息；`-g3` 会包含更丰富的宏相关调试信息；`-ggdb` 有面向 GDB 的倾向。
- `-fno-omit-frame-pointer` 保留帧指针（在支持平台与情境下），常用于采样剖析/堆栈回溯；它有一定性能和寄存器成本，不是所有环境都必需。
- 调试符号可以单独拆分、通过 build-id 关联；别把 `strip` 后的程序不能源码调试理解成“没有符号就不能执行”。

## 5.3 架构优化：`-march`、`-mtune`

```bash
g++ -O2 -march=native app.cpp -o app
```

- `-march=...`：允许生成特定 ISA/CPU 特性的机器指令。例如使用某些 SIMD 指令；**可能让程序无法在旧 CPU 上运行**（illegal instruction）。
- `-mtune=...`：在不一定更改允许指令集的情况下，面向某种微架构调整调度、成本模型等。
- `-march=native`：针对**构建机器**自动检测，实验方便，分发二进制和 CI 上需谨慎。
- `-mavx2` 等：显式启用 CPU 特性，只有目标 CPU 支持才能安全执行这些指令；最好明确部署基线或做运行时 dispatch。

写低延时系统时，编译器/目标 CPU/内存布局/实际负载共同决定性能。构建参数只是其中一层。

## 5.4 LTO：为什么“独立编译”并不是优化的天花板？

```bash
g++ -O2 -flto -c a.cpp -o a.o
g++ -O2 -flto -c b.cpp -o b.o
g++ -O2 -flto a.o b.o -o app
```

**Link Time Optimization（链接期优化）** 的思路是：正常 `.o` 大多已变为目标代码，链接器通常不能做任意高级 C++ 语义优化；LTO 则保留编译器中间表示或相应信息，在链接阶段配合编译器插件/工具进行跨 TU 优化。

可以带来跨文件内联、死代码消除、全程序分析等收益，但也可能增加链接时间、内存消耗，并引入工具链兼容问题。**用 `-flto` 生成的对象文件不一定和普通 `ar/nm` 的每一种用法无缝兼容**；工具链通常有 `gcc-ar`、`gcc-nm`、`gcc-ranlib` 或相应 linker plugin 支持。

CMake 对应的是 `INTERPROCEDURAL_OPTIMIZATION` target 属性和 `CheckIPOSupported`，后文展开。

## 5.5 Sanitizers：不是把警告等级调高

```bash
g++ -g -O1 -fsanitize=address -fno-omit-frame-pointer \
    main.cpp -o app_asan
```

AddressSanitizer（ASan）在代码中插入额外检查，并链接运行时，用来发现典型的内存越界、use-after-free 等。

```bash
g++ -g -O1 -fsanitize=undefined main.cpp -o app_ubsan
```

UBSan 检测许多未定义行为。

```bash
g++ -g -O1 -fsanitize=thread -pthread thread.cpp -o app_tsan
```

TSan 针对数据竞争等问题，可能与其他 Sanitizer 不能直接组合使用，并有较大的运行开销。

重要：**编译和最终链接都需要相应 sanitizer 选项**，因为不仅要插桩，还要把对应运行时正确链接进来。Sanitizer 构建不是真实性能 benchmark 环境。Linux 平台和编译器版本也会影响兼容性。

## 5.6 安全与二进制防护选项

你在发行版构建日志中可能见到：

- `-fstack-protector-strong`：在适用函数中加入栈保护检查；不是防御所有内存漏洞。
- `-D_FORTIFY_SOURCE=2` 或 `=3`：在满足 libc/优化等条件时，对部分已知长度的库调用加入检查；不是对任意指针访问的通用保护。
- `-fPIE` + `-pie`：创建位置无关可执行文件，可配合 ASLR。
- `-Wl,-z,relro`：使部分动态链接相关内存区域在重定位后变为只读。
- `-Wl,-z,now`：让动态函数绑定等更倾向启动时完成，常用于 full RELRO 配置；可能增加启动成本。
- `-Wl,-z,noexecstack`：请求不可执行栈（具体需要结合架构和输入节标记）。
- `-fvisibility=hidden`：限制共享库默认导出的符号，缩小 ABI 暴露面和潜在的符号抢占影响；要明确给 public API 设置导出可见性。

编译防护、安全机制、OS 权限和程序逻辑是不同层面的东西。不要把某个 flag 视为安全性的保证。

## 5.7 与链接器选型、链接诊断有关的高级参数

```bash
g++ main.o -fuse-ld=lld -o app
```

在工具链支持且已安装时，可选择不同链接器（如 `bfd`、`gold`、`lld`、`mold`，具体支持由 GCC/环境决定）。它们在速度、功能、错误提示、链接脚本兼容性等方面可能有差异。

```bash
g++ main.o math.o -Wl,-Map=app.map -o app
```

产生链接 map 文件，方便观察**输入对象、节布局、符号分配和库成员抽取**。

```bash
g++ main.o math.o -Wl,--trace-symbol=add -o app
```

这会请求 GNU ld 跟踪某个**实际链接符号名**的引用/定义。C++ 名字可能被修饰，要先用 `nm` 找到真正的符号名。

```bash
g++ main.o math.o -Wl,--no-undefined -o app
```

对构建共享库的部分场景尤其有用：禁止留下不符合预期的未定义引用（具体约束和平台相关）。

```bash
g++ -shared -fPIC util.cpp -Wl,-soname,libutil.so.1 -o libutil.so.1.0
```

`SONAME` 是动态库的 ABI 身份/依赖名机制，后面详细解释。

## 5.8 控制产物体积：节级回收与 strip

```bash
g++ -O2 -ffunction-sections -fdata-sections -c main.cpp -o main.o
g++ main.o -Wl,--gc-sections -o app
```

思路：把函数/数据放到更细粒度的节中，链接器按依赖关系回收没有被保留的节。这与静态库按成员 `.o` 抽取是**两层不同粒度**的裁剪。

`strip app` 主要是剥离部分不需要运行的符号/调试信息，减小文件大小。它不会自动像程序优化那样让 CPU 执行更少的指令，也不能保证隐藏所有敏感信息。

## 5.9 PGO 和 benchmark 参数

Profile-Guided Optimization（PGO）：

```bash
# 示意：先生成插桩程序，然后用代表性负载训练，再反馈优化
g++ -O2 -fprofile-generate source.cpp -o app_train
./app_train     # 运行有代表性的输入
g++ -O2 -fprofile-use source.cpp -o app_pgo
```

实际多文件项目需保证训练数据位置/文件名匹配，且按编译器文档处理 profile 收集与合并。PGO 有时比机械上 `-O3` 更有用，但训练负载不代表生产时也可能产生退化。

和低延时开发相关的另一些术语：`-fno-exceptions`、`-fno-rtti`（改变 C++ 语言运行时能力，不能盲目开启）、`-DNDEBUG`（常使 `<cassert>` 的 `assert` 失效）、`-fno-semantic-interposition`（影响可抢占符号假设，需懂 ABI 后再用）、`-fdiagnostics-color`、`-ftime-report` 等。**优化性能时，先保证正确性和可测量性，再调整选项。**

## 5.10 哪些参数必须同时用于编译和链接？

| 参数 | 常见要求 | 为什么 |
|---|---|---|
| `-fsanitize=address` | 编译与链接 | 编译插桩；链接 Sanitizer runtime |
| `-flto` | 通常编译与链接一致配合 | 提供 LTO 数据并在链接期使用 |
| `-pthread` | 建议编译与链接一致 | 可能同时影响宏/编译和线程库链接 |
| `-fPIC` | 主要编译阶段 | 控制对象机器码可否适合 DSO |
| `-shared` | 主要链接阶段 | 决定输出共享对象 |
| `-fPIE` | 主要编译阶段 | 生成适合 PIE 的代码 |
| `-pie` | 链接阶段 | 决定构建 PIE 可执行文件 |
| `-Wl,-z,now` | 链接阶段 | 写入动态链接相关配置 |
| `-I` | 编译阶段 | 源码需要找到头文件 |
| `-L` / `-l` | 链接阶段 | 链接器需要寻找和选择库 |

**一个 CMake 思考题**：`target_compile_options()` 和 `target_link_options()` 为什么要分成两个命令？答案已经藏在这张表里。

---

# 6. `.o` 究竟是什么：ELF、节、段、符号表

这里是从“会敲命令”进入“真正理解链接”的关键门槛。对应 **CSAPP 第 9～13 页**，其中第 11～12 页的 ELF 示意图值得边看边读。

## 6.1 `.o` 不是“半个 exe”，而是有结构的二进制容器

在 Linux ELF 平台上，一个 `.o` 文件不是随手堆砌的一串 CPU 指令。它是符合 ELF（Executable and Linkable Format）规范的结构化文件，包含：

- ELF 文件头：标识格式、架构、文件种类、入口信息等。
- Section Header Table（节头表）：列出各个 section 的属性和位置。
- `.text` / `.rodata` / `.data` / `.bss` 等节。
- `.symtab`、`.strtab`：用于链接的符号与字符串信息。
- `.rela.*`：记录哪些位置在链接时还要修补。
- 可选调试、异常展开、元数据等节。

你马上可以做：

```bash
g++ -g -O0 -c math.cpp -o math.o
file math.o
readelf -h math.o        # ELF 头
readelf -S math.o        # 节头表
readelf -sW math.o       # 符号表
readelf -rW math.o       # 重定位记录
objdump -dr -C math.o   # 反汇编并显示重定位，C++ 符号尽量还原
```

**观察提示**：单独的 `math.o` 可能因没有跨外部对象的复杂数据引用而没有你预想的重定位；请对含 `extern` 函数调用的 `main.o` 使用 `readelf -rW`。

## 6.2 Section 与 Segment：中文都叫“段”很容易误导

建议你先强行区分两个英文词：

- **Section（节）**：主要用于**链接、静态分析和代码组织**。例如 `.text`、`.symtab`、`.rela.text`。
- **Segment（装载段）**：主要用于**装载与运行**，由 Program Header Table（程序头表）描述，多个相关 sections 可以归入一个可映射的 segment。

简化地说：

```text
面向链接器：Section / Section Header Table
    .text, .rodata, .data, .bss, .symtab, .rela.text ...

面向装载器：Segment / Program Header Table
    PT_LOAD (R-X)  ──> 某些代码/只读内容
    PT_LOAD (R--)  ──> 某些只读内容
    PT_LOAD (RW-)  ──> 某些可写数据/.bss
    PT_INTERP      ──> 动态加载器路径
    PT_DYNAMIC     ──> 动态链接信息
```

关系不是“一个 Section 就对应一个 Segment”。许多调试/符号信息 sections 根本不需要映射到运行中的进程。现代二进制在 `-z separate-code` 等配置下，还可能按权限细分更多 PT_LOAD。

**这正是 CSAPP 第 11 页和第 23 页两张图要分别告诉你的事情**：前者看的是文件中的组织，后者看的是如何映射到运行时地址空间。

## 6.3 常见 Section，一项一项讲清楚

### `.text`：代码

一般存放机器指令，可具有执行权限。不是说 `.text` 的字节天然能独立执行，它们依然可能依赖链接器/加载器提供的地址、库和环境。

### `.rodata`：只读数据

常见字符串常量、某些 `const` 数据、跳转表。编译器可能合并常量、直接放入指令立即数、消除未使用常量，因此**不是每个源码里的 `const` 都必然对应 `.rodata` 中一个可见符号**。

### `.data`：有初值、需要可写存储的数据

```cpp
int global_count = 7;
```

在未被优化消除、未使用特殊 section 等常规情况下，通常会产生 `.data` 数据。

### `.bss`：需要内存但不占用同等文件内容的数据

```cpp
int global_zero;
int global_explicit_zero = 0;
static int cache[4096];
```

常见放到 `.bss`（具体依编译器/目标/优化而变）。`.bss` 是 `SHT_NOBITS` 一类节：**在文件里主要是描述需要多少空间，并不需要真把几千个零写满文件**。装载后进程对应内存初始为零。

注意 CSAPP 第 15 页图上的文字“`static int x=0` 在 `.data`”：这只是课件图示中不够严谨的说法。现代 GCC 在常规 ELF 构建下通常会把零初始化的静态存储期变量放进 `.bss`；`static int x=1` 则通常进入 `.data`。**看 `readelf` 的实际结果，不按颜色图生搬硬套。**

### `.symtab` 和 `.strtab`：符号清单与名字存储

符号表包含符号名、绑定属性、类型、所属 section、值、大小等；名字常由字符串表存储，符号表条目持有字符串表索引而不是把完整名字直接塞进去。

重要问题：**有些变量为什么不在 `.symtab`？**

例如：

```cpp
int f() { int local = 3; return local; }
```

编译器可能把 `local` 完全优化掉，或者把它放在寄存器/栈里。它不是链接器需要跨模块解析的外部链接符号。你可能在 DWARF 中看到调试器需要的局部变量信息，但那不是说每个 C++ 自动局部变量都要在 `.symtab` 有一个可供跨对象引用的名字。

CSAPP 第 13～15 页特别强调的“local linker symbol（局部链接符号）”与“local C variable（局部程序变量）”**不是同一个概念**。

### `.rela.text`、`.rela.data`：重定位表

它们记录在 `.text` / `.data` 中哪些**位置**还需要根据最终链接布局修补。这些是理解链接器最直接的证据。

课件第 11～12 页写作 `.rel.text`、`.rel.data`；它是把 relocation section 作为通用概念介绍。**在 x86-64 ELF 上，常见实际名字是 `.rela.text`、`.rela.data`，`RELA` 条目通常显式带 addend（加数）**。不同架构/格式存在 `REL` 与 `RELA` 的差别，不要看到名字不同就以为自己做错实验。

### `.debug_*`：调试信息

如 `.debug_info`、`.debug_line` 等。`-g` 会产生它们。编译后进行 strip 可能移除部分信息，不影响普通加载/运行，但影响调试体验。

### 其他值得认识的节

- `.eh_frame`、`.gcc_except_table`：异常展开、栈回溯或相关表信息（具体依 ABI）。
- `.init_array` / `.fini_array`：初始化/终结函数指针，C++ 全局对象构造、析构与这些机制有关。
- `.got`、`.got.plt`：位置相关地址的间接存储，稍后和 PIC/PLT 一起解释。
- `.plt`、`.plt.sec` 等：部分 ELF 链接实现中的过程链接代码。
- `.dynamic`：共享对象或动态链接 ELF 的动态信息。
- `.dynsym`、`.dynstr`：动态链接器需要的符号和字符串信息。
- `.gnu.hash`、`.hash`、`.gnu.version*`：动态符号查找与版本相关信息。
- `.note.gnu.build-id`：常用来识别构建产物，辅助符号文件匹配。

不必背完所有 section 名字：先知道**它属于代码、数据、符号、重定位、装载还是调试哪一类**。

## 6.4 ELF 文件的三种常见类型

对应 CSAPP 第 9～10 页：

1. **ET_REL**：可重定位对象文件，如普通 `.o`。
2. **ET_EXEC**：传统类型的可执行 ELF（如 `-no-pie` 的常见产物）。
3. **ET_DYN**：共享对象 `.so`；现代 **PIE 可执行文件也常是 ET_DYN**。

这点非常容易疑惑：

```bash
g++ main.cpp -o app
readelf -h app | grep Type
```

可能显示 `DYN (Position-Independent Executable file)`，并不是说你不小心编译成了 `.so`。现代发行版的 GCC 默认可能启用 PIE，**能不能直接作为程序运行，不能只看 ET_DYN 这一个枚举值**；还要结合入口点、解释器、标志和实际构建方式。

## 6.5 ELF header、section header、program header 分别要回答什么？

- **ELF Header**：这是 ELF 吗？64 位还是 32 位？大小端？哪个机器架构？文件类型？入口在哪里？其他表在哪里？
- **Section Header Table**：文件包含哪些 sections？类型、大小、对齐、偏移、属性？链接器怎样找到符号和重定位表？
- **Program Header Table**：装载器应该把哪些**文件字节**映射到**什么虚拟地址和权限**？哪些数据只占内存而不占文件？动态加载器是谁？

```bash
readelf -hW app   # ELF Header
readelf -SW app   # Sections
readelf -lW app   # Program Headers
readelf -dW app   # Dynamic entries（若存在）
readelf -sW app   # Symbols
readelf -rW app   # Relocations
```

最好把三条输出真的打开看过一次。这样你以后读到 `.text`、PT_LOAD、`.symtab` 就不会再把它们想成同一种“段”。

## 6.6 为什么 `.bss` 的大小和文件大小不是一回事？

例如一个真正被使用的巨大数组：

```cpp
// big.cpp
volatile unsigned char data[100'000'000];
int main() { data[0] = 1; return data[0] - 1; }
```

```bash
g++ -O0 big.cpp -o big
ls -lh big
size -A big | grep -E '\.bss|\.data|\.text'
readelf -lW big
```

可能发现：文件远小于 100 MB，但可写装载段在内存中的 `MemSiz` 比 `FileSiz` 大很多。因为 `.bss` 的零填充可以在装载时通过虚拟内存机制提供，而不是把 100 MB 的零放进 ELF 文件。

**更深一步**：内核通常不会在 `execve` 瞬间给你物理分配好全部 100 MB；虚拟内存映射和按需分配页面会影响实际常驻内存（RSS）。所以**文件大小、虚拟地址空间占用、实际物理页占用不是同一个数**。

---
# 7. 链接的第一件事：符号解析

这是整篇教材的第一个真正核心章节，对应 **CSAPP 第 7、13～19 页**。

## 7.1 为什么链接器必须关心“符号”？

想象你编译 `main.cpp`：

```cpp
int square(int);
int main() { return square(7); }
```

编译器知道 `square` 接受一个 `int` 并返回一个 `int`，因此知道怎么生成调用指令、怎样安排参数和返回值。但 `square` 的实现没有在当前 TU 出现，它不知道最后机器码放在哪个地址。它不能凭空猜一个数字。

于是对象文件里会留下两类信息：

- “我**引用**了一个叫 `square(int)` 的外部符号”；
- “我在这个调用位置，需要链接时根据该符号地址做修补”。

而另一个文件：

```cpp
// square.cpp
int square(int n) { return n * n; }
```

会产生一个定义 `square(int)` 的对象文件。

**符号解析（symbol resolution）** 就是把 `main.o` 中对某个符号的引用，与相关输入中提供它的定义建立正确联系。

```text
main.o                         square.o
┌────────────────────────┐     ┌────────────────────────┐
│ .text: call ???        │     │ .text: square 机器码  │
│ symtab: square = UND   │     │ symtab: square = DEF   │
│ relocation: 指向square │     └────────────────────────┘
└────────────────────────┘                 │
              └──────── 链接器 ───────────┘
                         │
                 识别同一链接符号
                         │
                确定布局并修补引用
```

`UND`（undefined）不是说源代码在语法上不合法，而是**当前对象文件没有提供定义**。这是可重定位 `.o` 非常正常的状态；不过当链接成一个需要完整解析该引用的输出时，如果依然找不到定义，就会出现 undefined reference 之类的错误。

## 7.2 什么算“符号”？与普通变量有什么区别？

课件第 13 页分为：

1. **Global symbols（全局链接符号）**：本模块定义、可以被其他模块引用，如具外部链接属性的函数和全局变量。
2. **External symbols（外部符号引用）**：本模块引用，但定义在别处。
3. **Local symbols（局部链接符号）**：本对象文件内部可见的链接符号，例如文件作用域 `static` 函数或内部链接变量。

这些是**以链接器和对象文件为中心的分类**，与 C++ 作用域分类不是完全相同的轴。

例如：

```cpp
// a.cpp
static int private_counter = 10; // 通常内部链接，可对应 STB_LOCAL 符号
int public_counter = 20;         // 通常外部链接，可跨 TU 使用

int f(int x) {
    int temporary = x + 1;        // 普通自动局部变量，不必成为 linker symbol
    return temporary + private_counter;
}
```

- `public_counter` 是典型外部可见定义。
- `private_counter` 可能形成本对象文件局部符号。
- `temporary` 是语言级自动局部变量；即使调试器能显示，也不代表其在链接符号表中有一项。

还有：

```cpp
int f() {
    static int count = 0;
    return ++count;
}
```

这里的 `count` 是**函数作用域的静态存储期对象**；它不是每次调用时新建的栈变量。它有跨调用生命周期，因此通常有持久的存储位置和相应内部符号（也可能被优化处理）。这个例子正对应 CSAPP 第 15 页的主题。

## 7.3 用 `nm` 亲眼看符号解析

```cpp
// main.cpp
extern int shared;
int square(int);
int main() { return square(shared); }
```

```cpp
// square.cpp
int shared = 7;
int square(int x) { return x * x; }
```

```bash
g++ -O0 -c main.cpp -o main.o
g++ -O0 -c square.cpp -o square.o
nm -C main.o
nm -C square.o
```

你可能看到近似结果：

```text
main.o:
  U shared
  U square(int)
  T main

square.o:
  D shared
  T square(int)
```

不同平台、优化、工具链细节可能导致输出变化。主要理解 `nm` 常见字母：

| 标记 | 常见含义 |
|---|---|
| `U` | Undefined：当前文件引用、未定义 |
| `T` / `t` | 代码节中的全局 / 局部符号 |
| `D` / `d` | 已初始化数据节中的全局 / 局部符号 |
| `B` / `b` | BSS 类数据中的全局 / 局部符号 |
| `R` / `r` | 只读数据中的全局 / 局部符号 |
| `W` / `w` | 弱符号（细节与是否定义有关） |
| `C` | Common 符号；旧式 C tentative definition 可能出现 |

`nm -C` 会尝试把 C++ mangled names 恢复为可读的 `square(int)`。如果想看真正链接符号名，去掉 `-C`：

```bash
nm main.o
```

## 7.4 为什么 `undefined reference` 和 `multiple definition` 恰好相反？

### 缺定义：undefined reference

```bash
g++ -c main.cpp -o main.o
g++ main.o -o app
```

如果 `main.o` 需要 `square(int)`，而没有提供该定义，链接器会报找不到：

```text
undefined reference to `square(int)'
```

可造成这种错误的原因不止“忘了写 square.cpp”：

- 忘了将源文件加入构建。
- 忘了将相应 `.o` 或 `.a/.so` 加进链接命令。
- 函数声明/定义签名不一致（C++ 类型不同，mangled name 不同）。
- C/C++ 混用时漏了 `extern "C"`。
- 模板定义不可见，又没有提供显式实例化。
- 第三方库 ABI/版本不兼容。
- 静态库顺序不合适。
- 所需符号被标成 hidden 或没有导出。
- 链接时运行库选择错误（例如用 `gcc` 驱动链接 C++）。

### 多定义：multiple definition

```cpp
// a.cpp
int duplicated() { return 1; }
```

```cpp
// b.cpp
int duplicated() { return 2; }
```

两个具有外部链接的同名同签名函数定义合在一起，通常构成违反单一定义规则的链接冲突：

```text
multiple definition of `duplicated()'
```

这一类错误的修复原则**不是选择一个任意定义**，而是让工程真正只提供该外部符号的一份正确实体：通常函数在 `.h` 声明、在一个 `.cpp` 定义；或者在适合的头文件场景用 `inline` / 模板等规则。

## 7.5 C++ 的 `static`、匿名命名空间、`inline`：究竟如何影响链接？

### 文件作用域 `static`

```cpp
// a.cpp
static int helper() { return 1; }
```

其名字通常具有内部链接，不会与 `b.cpp` 的另一个 `static int helper()` 形成普通外部符号冲突。

### 匿名命名空间

```cpp
namespace {
int helper() { return 1; }
}
```

通常同样用于实现仅在当前翻译单元可见的名字，是现代 C++ 比文件作用域 `static` 函数更通用的组织方式。

### `inline`

```cpp
// math.hpp
inline int twice(int x) { return x * 2; }
```

`inline` 除了与编译器可能内联展开有关，更重要的一层是 **ODR 下允许满足条件的相同定义出现在多个翻译单元中**。实现往往借助 weak/COMDAT 等机制合并或选取定义。

**不要误以为 `inline` = 一定不产生函数调用**。优化器仍可能保留 out-of-line 实现或选择不内联；`-O2` 也可能内联一个没写 `inline` 的函数。

C++17 的 `inline` 变量也可以用于头文件中被多个 TU 包含的共享变量定义：

```cpp
inline int global_config_version = 1;
```

前提是遵守适用的 ODR 条件。

## 7.6 CSAPP 的 strong / weak 规则，现代人应该怎样读？

CSAPP 第 16～18 页的经典模型：

- strong：函数及初始化的全局变量等。
- weak：旧 GCC C 模式下常见的未初始化全局变量（tentative definition 映射成 COMMON）。
- strong+strong：报错。
- strong+weak：通常选 strong。
- weak+weak：可以选一个。

课件想提醒你：**某些同名全局定义并不一定立刻报错，可能悄悄共享存储甚至产生危险覆盖**。这非常有教育意义，但 2015 年的编译器前提必须更新。

### 关键历史变化：GCC 10 默认 `-fno-common`

```c
// a.c
int x;
```

```c
// b.c
int x;
```

在很老的 GCC C 默认配置下，两个 tentative definitions 常被输出为 COMMON 符号，链接器可合并。现代 GCC **自 10 起默认 `-fno-common`**，通常把各自定义放入实际存储节，于是两个对象一起链接会报 multiple definition。

可做实验：

```bash
gcc -fcommon -c a.c -o a_common.o
gcc -fcommon -c b.c -o b_common.o
nm a_common.o           # 可观察 C x（实际结果依环境）

# 现代 GCC 默认
gcc -c a.c -o a.o
gcc -c b.c -o b.o
nm a.o                  # 常见 B x，而非 C x
```

**为什么这个变化重要？** 因为历史上依靠 COMMON 静默合并可能掩盖严重 bug。更合理的写法是：

```c
// shared.h
extern int x;  // 声明

// shared.c
#include "shared.h"
int x = 0;     // 唯一定义
```

多个 `.c` 只包含声明即可。C++ 从语言规则和工具链行为上也**不能照搬旧式 C COMMON 经验**。

### 显式 weak 与旧式 COMMON 不是同一件事

GNU/ELF 还允许通过编译器属性显式产生 weak 符号（如 `__attribute__((weak))`），用于某些覆盖/默认实现机制；这和旧 GCC 把未初始化 C tentative definition 变成 COMMON 是不同机制。**不要把所有弱链接行为归结为“没初始化就是 weak”**。

## 7.7 CSAPP 第 18 页为什么举 `int x` / `double x` 的恐怖例子？

假如两个旧式 C 对象文件使用同一个名字，但开发者错误地把它当作不同类型：

```c
// file1.c
int x;
int y;
```

```c
// file2.c
double x;
```

若错误的外部符号被合并，两个 TU 对 `x` 的数据类型和大小认知不同。一个按 8 字节写入，另一个按 4 字节存储和访问，可能造成破坏邻接数据等严重问题（具体布局、链接规则和行为依实际环境）。

**现在不应把“能成功链接”当作 C 类型兼容检查的保证**。C 链接器按符号名和 ELF 元数据工作，不会替你执行完整的跨 TU 语言类型证明。C++ 虽有 mangling，但变量类型、ABI 布局等问题也不自动都能由链接器发现。

## 7.8 符号的可见性与绑定：local/global/weak 是不同维度

在 `readelf -sW` 你会看到：

```text
Num: Value Size Type Bind Vis Ndx Name
```

- `Type`：`FUNC`、`OBJECT`、`NOTYPE`、`SECTION`、`TLS` 等。
- `Bind`：`LOCAL`、`GLOBAL`、`WEAK` 等。
- `Vis`：`DEFAULT`、`HIDDEN`、`PROTECTED` 等（语义随 ELF 平台）。
- `Ndx`：对应的 section 索引，或者 `UND`、`ABS`、`COM` 等特殊值。

**弱/强是 binding 规则的一部分；隐藏/可见是 visibility 规则；定义在哪个 section 又是第三个维度。** 不能把它们揉成一个“static/public”二元分类。

共享库的“导出 API”需要额外考虑 `.dynsym` 与可见性：仅仅在源码中有函数定义，不保证程序可以通过 `dlsym()` 找到它。

---

# 8. 链接的第二件事：重定位

这是整篇的第二个核心章节，对应 **CSAPP 第 8、20～22 页**。你说当年 CSAPP linking 没怎么听懂，我猜这很可能是最卡人的地方：**明明 main.o 中已经有 call 指令，为什么还需要链接器修改那几个机器码字节？**

## 8.1 从“电话号码还没分配”理解重定位

想象你写了一封信：

> “请拨打 `sum` 的电话。”

但你还不知道 `sum` 未来使用哪个号码。你可以先把信写完，留下一个待填写位置，并另附一条说明：

> “最终分配完号码以后，把这封信第 120 个字节起的 4 字节写成 `sum` 的电话相关数值。”

这就是可重定位目标文件的核心结构：

- `.text` 的大部分机器码可以先生成。
- 一些位置暂时无法最终编码。
- 重定位表记录应该去哪里修补、要引用哪个符号、采用哪种计算方式。
- 链接器统一布局之后，把适当的地址、偏移或索引写进去。

**重定位（relocation）不是“给程序整体加个偏移”这么简单**：它是对一个个使用到符号的位置进行计算和修补，计算公式因重定位类型而异。

## 8.2 为什么链接前不能知道绝对地址？

`main.o` 独立产生时，编译器不知道：

- `math.o` 有多大。
- 其他函数、初始化代码、静态库成员最终会不会被加入。
- 链接器如何对齐和排序 `.text/.data`。
- 可执行文件是否是 PIE。
- 动态库运行时最终会映射到哪里。

因此像 `call add` 或“取全局变量地址”之类操作，可能需要后续步骤决定相对位移或地址信息。

## 8.3 先懂 PC-relative：`call` 为什么常不存完整的 64 位地址？

x86-64 的常见直接调用指令：

```asm
call some_function
```

机器码中常见形式：

```text
e8 xx xx xx xx
```

- `e8` 是该类 `call rel32` 的操作码。
- 后面的 4 个字节是一个**有符号的相对位移**（小端存储）。
- CPU 实际的跳转目标，一般等于**下一条指令地址 + 该位移**。

例如 `call` 从地址 `0x4004de` 开始，占 5 字节，下一条指令地址 `0x4004e3`，目标函数入口是 `0x4004e8`。

```text
call 指令起点       0x4004de
位移字段起点        0x4004df
下一条指令         0x4004e3
目标函数 sum       0x4004e8
所需位移           0x4004e8 - 0x4004e3 = 5
```

因此对应位移编码可以是：

```text
e8 05 00 00 00
```

这正是 **CSAPP 第 22 页**的关键计算：`0x4004e8 = 0x4004e3 + 0x5`。

你不需要死背那个具体十六进制数；真正要记的是：**“相对位移”通常相对于下一条指令，而不是直接存一个绝对函数地址。**

## 8.4 链接器的重定位条目到底记录什么？

在 ELF64/x86-64 常见 RELA 形式中，可以从概念上理解为：

```text
Relocation Entry:
    offset   要修改目标 section 的哪个偏移
    symbol   使用哪个符号（如 sum）
    type     用什么计算规则和编码宽度
    addend   额外加数 A
```

常用表示：

- `S`：被引用符号的最终相应地址或值。
- `A`：重定位条目中的加数（addend）。
- `P`：重定位字段本身所在的地址（place）。

一些典型 x86-64 重定位公式：

```text
R_X86_64_64    : S + A         # 写入 64 位绝对地址/值
R_X86_64_PC32  : S + A - P     # 写入 32 位相对位移
R_X86_64_PLT32 : L + A - P     # 针对 PLT/链接器所选调用落点的相对位移
```

`L` 可理解为相关 PLT 位置（具体语义按 ABI/链接器处理）。共享库、PIE 还会有大量其他重定位类型。你现在掌握 `S+A` 与 `S+A-P`，就可以读懂最典型的静态链接例子。

### 令人困惑的 `-4` 从哪里来？

CSAPP 第 21 页 `objdump -r -d main.o` 中你可能见到类似：

```text
R_X86_64_PC32 sum-0x4
```

为什么是 `sum - 4`？

因为链接器以**位移字段的起始位置 P** 计算，而 x86 `call rel32` 的 CPU 语义是从**位移字段结束后的下一条指令地址**计算，也就是 `P + 4`。

所以：

```text
需要写入的数值 = S - (P + 4)
              = S + (-4) - P
```

因此 addend 就体现为 `-4`。以前看起来神秘的 `sum-0x4`，实际上是在**补偿“重定位字段起点”与“CPU 下一条指令地址”之间的 4 个字节**。

还有一个时代差异：现代汇编器对一些外部函数调用可能使用 `R_X86_64_PLT32`，不一定与你的 2015 年课件第 21 页完全一样。**修补调用位移的本质没有变，但要以实际 ABI/编译器/链接器给出的类型为准。**

## 8.5 绝对地址与 PC 相对地址，为什么两种都要有？

例如：

```cpp
extern int global_number;
int* ptr = &global_number;
```

如果需要让一个存储单元保有指向 `global_number` 的地址，就可能出现绝对地址或依赖装载时基址的重定位。

而执行：

```cpp
extern int f();
int g() { return f(); }
```

常见机器指令使用相对调用，可能只需根据相对距离修补位移。

两者各有价值：

- 绝对地址容易直接表示，但受装载基址变化影响。
- PC-relative 可让同一段代码在不同地址被映射时，**只要引用目标和调用指令整体一起平移，二者距离不变**；这有利于位置无关代码。

注意：相对地址也有编码范围限制（如有符号 32 位），跨越范围时链接器可能报 relocation truncated/overflow 或采用特殊机制。**链接错误不只有“没找到名字”，也可能是地址编码装不下。**

## 8.6 一次真正的重定位观测实验

创建：

```cpp
// main.cpp
extern int global_value;
int square(int);
int main() { return square(global_value); }
```

```cpp
// square.cpp
int global_value = 7;
int square(int x) { return x * x; }
```

先生成未链接的两个对象：

```bash
g++ -O0 -fno-pie -c main.cpp -o main.o
g++ -O0 -fno-pie -c square.cpp -o square.o
```

观察：

```bash
nm -C main.o
readelf -rW main.o
objdump -drC main.o
```

你应该重点找：

1. `main` 的机器码。
2. `global_value` 的加载引用。
3. `square(int)` 的函数调用引用。
4. `.rela.text` 中的 relocation 信息。

链接成一个更接近老式教学例子的非 PIE 可执行文件：

```bash
g++ -no-pie main.o square.o -o app
```

然后：

```bash
objdump -dC app | sed -n '/<main>:/,+24p'
nm -nC app | grep -E 'square|global_value|main'
```

比较链接前后：原来某些机器码字段是不完整/待修补的；现在已经有符合最终布局的位移/地址相关值。

特别注意：`-fno-pie` 是**编译阶段**选项，而 `-no-pie` 是**链接阶段**选项。为了让本实验更稳定地呈现你在课件看到的绝对地址/非 PIE 风格，两者分别使用。真实项目通常应保留系统安全默认值，不需要长期禁用 PIE。

## 8.7 但为什么最终可执行文件里可能仍有重定位？

因为现实世界不总是在链接命令结束时完成所有地址确定：

- 静态链接器可以解决某些已知对象之间的地址关系。
- PIE/共享库可能保留**动态重定位**，交给装载时的动态链接器处理。
- 对动态库外部函数的某些绑定可以推迟到运行时第一次调用。

所以：

```bash
readelf -rW app
```

即使 `app` 已经链接成功，也可能列出 `.rela.dyn`、`.rela.plt` 等动态重定位。这不矛盾：**“链接完成”不等于“运行时所有库地址和符号地址都已在磁盘上永久固定”。**

## 8.8 一条更准确的链接工作流程

CSAPP 第 7～8 页为了教学，把链接主任务分成：

1. Symbol Resolution（符号解析）。
2. Relocation（重定位）。

很好记，但真实链接器还会做：

- 搜索对象文件、静态归档和共享库。
- 决定哪些归档成员参与链接。
- 处理 COMDAT/重复节组、可选的节垃圾回收。
- 决定 sections 合并、排序、对齐与输出布局。
- 计算符号最终值/偏移并执行相应修补。
- 可能生成 PLT/GOT、动态链接元数据、符号版本记录。
- 可能执行 LTO 协调。
- 输出可执行 ELF、共享对象或进一步可重定位对象。

**记两大核心任务，再知道真实链接器的工程细节远比“拼两个文件”复杂。**

---

# 9. 静态库 `.a`：归档、按需抽取与顺序陷阱

对应 **CSAPP 第 24～30 页**。这是你学习 CMake 中 `add_library(... STATIC ...)` 和 `target_link_libraries()` 的直接前置。

## 9.1 为什么不用一个巨大 `utils.o`，偏要发明 `.a`？

假设标准库有 1000 个函数。若把它们都编译成一个巨大的 `liball.o`，简单的程序只用一个函数，却可能在链接阶段处理太多无关代码。如果每个 `.o` 都散放在文件夹，使用者又要维护无数文件名。

静态归档 `libfoo.a` 提供一种折中：**把许多独立可重定位对象文件打包进一个 archive，并保留可用于按需查找成员的索引。**

```
libmath.a
┌──────────────────┐
│ archive header   │
├──────────────────┤
│ math.o           │  定义 add、subtract
├──────────────────┤
│ geometry.o       │  定义 distance
├──────────────────┤
│ format.o         │  定义 print_num
└──────────────────┘
```

链接器通常不需要把归档里的所有成员都加入最终程序，而是根据当前的未解析符号引用**按需抽取必要的成员**。

这就是 CSAPP 第 25 页最核心的想法。请留意：**抽取的基本单位通常是 archive member（一个 `.o`），而不总是单个函数**。一个成员 `.o` 内有多个函数时，它们可能随这个成员整体进入链接（之后还能进一步通过节级回收裁剪）。

## 9.2 自己制造一个 `.a`

```cpp
// math.cpp
int add(int a, int b) { return a + b; }
```

```cpp
// main.cpp
int add(int, int);
int main() { return add(2, 3) - 5; }
```

```bash
g++ -c math.cpp -o math.o
ar rcs libmath.a math.o
ar t libmath.a
nm -C libmath.a
```

`ar rcs` 的直观意思：

- `r`：插入成员或替换同名成员。
- `c`：必要时创建归档；抑制某些提示。
- `s`：写入/更新归档索引。

`ranlib libmath.a` 也可更新索引，现代 `ar s` 常起相应作用。

然后：

```bash
g++ main.cpp -L. -lmath -o app
./app
echo $?    # 应为 0
```

这里 `-lmath` 通过库搜索规则找到了 `libmath.a`，抽取其中 `math.o`，解决了 `add` 引用。

## 9.3 为什么 `-lmath` 的前后顺序能决定链接成功与否？

两个命令对照：

```bash
g++ main.o -L. -lmath -o app        # 通常成功
g++ -L. -lmath main.o -o app        # 静态库常见失败方式
```

传统 GNU 链接器处理静态归档时，通常按命令行顺序处理输入：

1. 先看到 `main.o`：得知 `add` 尚未解析。
2. 再看到 `libmath.a`：发现里面有 `add` 的定义，于是抽取 `math.o`。
3. 输出成功。

换个顺序：

1. 先看到 `libmath.a`：这时可能还没有发现任何需要 `add` 的引用；没有理由抽取 `math.o`。
2. 再看到 `main.o`：它提出对 `add` 的新需求。
3. 若链接器不回头重新扫描之前的归档，就找不到定义，链接失败。

把静态库想象成“按需求取件的仓库”：**先告诉仓库你缺什么，再让仓库检查库存。**

CSAPP 第 30 页重点讲的就是这个规则。现代 GNU ld、lld、mold 在若干边缘情况下的具体回溯/归档处理能力未必完全相同；学习时要掌握**可移植的依赖排序习惯**：通常对象/消费者在前，提供静态实现的库在后；避免依赖某个特定链接器的特殊宽容行为。

## 9.4 循环依赖怎么办？

假设静态库 A 里的某个成员需要 B，B 的另一个成员又需要 A 的另一部分。这时：

```bash
g++ main.o -lA -lB -o app
```

可能不够。你会见到几种技巧：

```bash
g++ main.o -lA -lB -lA -o app
```

或者 GNU ld 的归档组：

```bash
g++ main.o -Wl,--start-group -lA -lB -Wl,--end-group -o app
```

`--start-group/--end-group` 允许反复搜索组内归档，直到不能找到新的可解决引用；但有性能代价，**不是所有项目默认都该套一个大 group**。最根本的工程改进通常是消除不合理的库循环依赖，或调整模块边界。

## 9.5 `-Wl,--whole-archive`：我就想把全部成员纳入

```bash
g++ main.o -Wl,--whole-archive libmath.a \
    -Wl,--no-whole-archive -o app
```

这个选项要求链接器将指定归档的成员整体纳入，而非仅按未解析引用抽取。它在注册表、插件注册器、静态初始化入口等场景偶有必要。

但要注意：

- 容易增加产物大小，或者把不想引入的全局初始化带入程序。
- 仍可能受后续节级垃圾回收等机制影响；“所有成员被加入”不必然等于“所有函数指令必然留存”。
- 必须及时恢复 `--no-whole-archive`，否则后面默认链接的其他归档也可能被全量纳入。

CMake 也有相应机制（例如适用版本中的 `$<LINK_LIBRARY:WHOLE_ARCHIVE,...>`），不需要长期靠散落的原始 `-Wl` 手写。

## 9.6 “静态库是静态链接”，为什么 `g++ app.o libfoo.a` 生成的程序还会依赖 libc.so？

因为**你的 libfoo.a 被静态纳入**，不代表所有依赖都静态纳入。最终程序仍可能动态依赖 libc、libstdc++、libm 等。

```bash
g++ main.o ./libmath.a -o app
readelf -dW app | grep NEEDED
```

如果出现 `libc.so.6`、`libstdc++.so.6` 等，只能说明**最终 ELF 有这些动态依赖**。静态库与动态依赖可以同时存在于一个程序中。

真正请求尽量静态链接通常是：

```bash
g++ -static main.cpp -o app
```

但这需要系统有适当的静态库，也可能在 glibc 的 NSS、DNS、动态插件等场景遇到部署/行为限制。**`-static` 并不保证把 Linux 内核或一切运行时设施都“打包进去”**。

## 9.7 头文件、静态库、动态库究竟分别给我什么？

| 文件 | 主要提供 | 不负责什么 |
|---|---|---|
| `math.h` | C/C++ 声明与源代码级接口 | 不自动提供函数定义的机器码 |
| `libmath.a` | 可供链接器按需抽取的 `.o` 实现 | 不替你提供缺失的头文件声明 |
| `libmath.so` | 共享对象中的实现和动态符号 | 链接期找到 `.so` 不等于运行时一定能定位它 |

只要真正搞懂这张表，CMake 里的 `target_include_directories()` 和 `target_link_libraries()` 就再也不是两条毫无关联的口诀。

---
# 10. 动态库 `.so`：链接、装载与运行时查找

对应 **CSAPP 第 31～36 页**。前面静态库 `.a` 的典型模型是“链接时把需要的对象成员纳入程序”；动态库 `.so` 的模型则是“把可共享的实现以独立 ELF 对象保存，在装载/执行期间由动态链接机制建立关系”。

## 10.1 为什么需要动态库？

静态归档易于链接，但若 100 个进程都使用同一套庞大的 libc，实现被静态纳入各自的可执行文件，可能造成磁盘和内存使用上的重复。系统还需要考虑修复库 bug、发布安全更新、模块更新等问题。

共享对象的价值：

- 多个进程可以映射同一库的**共享、只读代码页**，在合适条件下共用物理页。
- 可执行程序不用总把库机器码静态拷进去。
- 在保持 ABI 兼容的前提下，库可以独立更新，无须每个使用者重新静态链接。
- 支持插件与运行时按需加载。

但它有代价：

- 运行时必须找得到兼容的 `.so`。
- 存在库版本与 ABI 兼容问题。
- 可能需要装载时/运行时重定位。
- 更复杂的符号搜索、可见性、延迟绑定、部署安全策略。

一句话：**静态库偏向“把实现纳入这个输出”，动态库偏向“保留对外部可装载实现的依赖关系”。**

## 10.2 从零编一个 `.so`

```cpp
// math.cpp
extern "C" int add(int a, int b) { return a + b; }
```

```cpp
// main.cpp
#include <cstdio>
extern "C" int add(int, int);
int main() { std::printf("%d\n", add(2, 3)); }
```

构建：

```bash
g++ -fPIC -c math.cpp -o math_pic.o
g++ -shared math_pic.o -o libmath.so

g++ main.cpp -L. -lmath -Wl,-rpath,'$ORIGIN' -o app
./app
```

预期输出：

```text
5
```

这里三行各自做什么：

1. `-fPIC -c`：为共享对象生成适宜位置无关装载的目标代码。
2. `-shared`：链接得到 `.so`。
3. `-L.` 让链接器在构建时找到库；`-lmath` 引用它；`-Wl,-rpath,'$ORIGIN'` 把**运行时**库搜索提示写进可执行文件（`$ORIGIN` 被运行时加载器展开为包含该程序的目录）。

最后这点非常关键：**构建期找到库，不代表运行时找得到库。**

## 10.3 动态链接是分几个时间点进行的？

### 链接期（你执行 g++ 的那一刻）

当你写：

```bash
g++ main.o -L. -lmath -o app
```

静态链接器会检查相关动态库能否满足当前链接需求，并在输出 ELF 中记录恰当的动态依赖和重定位信息。

这个阶段常常没有把整个 `libmath.so` 的 `.text` 代码拷贝到 `app`。

### 装载时（`execve` 创建程序映像时）

Linux 内核按 ELF 解释器信息把控制权交给动态加载器（常见为某种 `ld-linux-x86-64.so.*`）；它寻找依赖的 `.so`，映射它们，完成必要重定位和初始化，然后交给程序的启动过程。

### 运行时（执行过程中）

程序可能调用 `dlopen()` 显式加载新的 `.so`；某些函数符号绑定也可能按**惰性绑定**策略延迟到首次调用时发生。

因此 CSAPP 第 36 页说 linking 可能发生在程序生命周期不同时间点，是为了引导你认识**静态链接、装载期动态链接和运行期动态链接**。但“动态链接”不是说内核每执行一条函数调用都要重新查一次名字。

## 10.4 ELF 中是怎样记录动态库依赖的？

```bash
readelf -dW app
```

你会看到一些 `Dynamic section` 项目：

```text
NEEDED     Shared library: [libmath.so]
NEEDED     Shared library: [libc.so.6]
RUNPATH    Library runpath: [$ORIGIN]
```

实际是否列出某个库受你的源码、链接器选项（如 `--as-needed`）、libc/工具链等影响。

理解三个重点：

- `DT_NEEDED`：程序所依赖的共享对象名字。
- `DT_RPATH` / `DT_RUNPATH`：可执行文件或共享对象携带的库搜索路径提示，二者有重要差别。
- `DT_SONAME`：共享对象对外标识自己的 ABI 名称，供链接时及动态依赖处理使用。

ELF 不只是 `.text` 机器码，还自带了**动态加载说明书**。

## 10.5 `-L`、`LD_LIBRARY_PATH`、RPATH、RUNPATH：四者到底差在哪？

假设你的程序在：

```text
project/
├── bin/app
└── lib/libmath.so
```

### `-L/path`

链接**构建时**寻找 `libmath.so` / `libmath.a` 的位置：

```bash
g++ main.o -Lproject/lib -lmath -o app
```

这不能保证用户机器的动态加载器在运行时到同一路径找到它。

### `LD_LIBRARY_PATH`

运行**装载时**的环境变量：

```bash
LD_LIBRARY_PATH="$PWD/project/lib" ./project/bin/app
```

适合本地测试与排查。它影响某个进程的动态库搜索行为，存在环境依赖和潜在安全/可复现性问题，**不建议把全局 `LD_LIBRARY_PATH` 当作正式软件部署的万能解决方案**。

### RPATH / RUNPATH

这些是写在 ELF 的动态节中的库查找信息。示例：

```bash
g++ main.o -Lproject/lib -lmath \
    -Wl,-rpath,'$ORIGIN/../lib' \
    -o project/bin/app
```

必须注意 shell 引号：

```bash
'$ORIGIN/../lib'
```

使用单引号，避免 shell 抢先把 `$ORIGIN` 当环境变量展开。我们希望**动态加载器以后**替换它，而不是 shell 现在替换。

### 一个很重要但常被忽略的差异

在 glibc 的常规 ELF 依赖搜索规则中，`DT_RPATH`（没有 `DT_RUNPATH` 的情况下）与 `DT_RUNPATH` 不完全一样：**RUNPATH 通常只用于查找该对象的直接 `DT_NEEDED` 依赖，不像某些 RPATH 场景那样传播到更深层的依赖搜索**。此外还会考虑 `LD_LIBRARY_PATH`、系统缓存、默认目录等；安全执行模式可能忽略环境变量。

请不要死背成“永远先查 LD_LIBRARY_PATH，再查 rpath”：动态加载器真正的查找顺序要按对象及 `RPATH/RUNPATH` 存在情况判断。Linux/glibc 的细则见手册 `ld.so(8)`。

## 10.6 为什么有 `libfoo.so`、`libfoo.so.1` 和 `libfoo.so.1.2.3`？

常见发布方式：

```text
libfoo.so       -> libfoo.so.1
libfoo.so.1     -> libfoo.so.1.2.3
libfoo.so.1.2.3  （真正的 ELF 文件）
```

直觉上：

- `libfoo.so`：通常供开发/链接阶段使用的名字。
- `libfoo.so.1`：共享对象 ABI 的 SONAME 大版本标识。
- `libfoo.so.1.2.3`：实际文件版本。

这不是所有平台和所有软件都必须遵守的固定文件名方案，但对 Linux 共享库开发非常典型。

**ABI 兼容**不等于“源码 API 名字没变”。函数签名、结构体布局、调用约定、异常/RTTI、标准库 ABI、符号版本等变化都可能破坏旧二进制。

## 10.7 `ldd`、`readelf`、`LD_DEBUG` 各自回答什么？

```bash
readelf -dW app | grep NEEDED  # 二进制直接声明的动态依赖
ldd ./app                     # 查看某环境下动态依赖的解析结果
LD_DEBUG=libs ./app            # glibc 动态加载器库查找诊断（输出很长）
```

`readelf` 是静态读取 ELF；`ldd` 用来检查动态依赖解析结果，但**不要对不可信二进制随便使用 `ldd`**，因为某些情况/历史实现可能执行目标相关代码或其解释器。分析不可信文件时更稳妥的是 `readelf -d` 等静态工具。

`LD_DEBUG=bindings` 可以进一步观察动态符号绑定（输出非常庞大）。这些诊断主要适用于 glibc 的动态加载器环境，其他 libc 的行为可能不同。

## 10.8 链接成功、运行失败：经典错误 `cannot open shared object file`

想象：

```bash
g++ main.cpp -L. -lmath -o app    # 成功
./app
```

却出现：

```text
error while loading shared libraries: libmath.so: cannot open shared object file
```

根因：**链接期的 `-L.` 与运行时动态加载器的搜索策略是两回事**。修复思路：

1. 检查 `readelf -dW app` 中的 `NEEDED`、`RUNPATH/RPATH`。
2. 检查库实际存在路径、架构、权限和 SONAME。
3. 本地临时测试可以设置 `LD_LIBRARY_PATH=...`。
4. 正式部署可以使用合理的安装布局/`$ORIGIN` RUNPATH/系统包管理器配置，而不是把所有绝对路径硬编码进源码。

## 10.9 `dlopen()`：已经运行的进程还能链接新功能？

可以。对应 CSAPP 第 34～35 页。以 C API 的 `add` 为例：

```cpp
// plugin.cpp
extern "C" int plugin_add(int a, int b) { return a + b; }
```

```cpp
// host.cpp
#include <dlfcn.h>
#include <iostream>

int main() {
    void* handle = dlopen("./libplugin.so", RTLD_NOW | RTLD_LOCAL);
    if (!handle) {
        std::cerr << dlerror() << '\n';
        return 1;
    }

    dlerror(); // 清除旧错误状态
    using AddFn = int (*)(int, int);
    auto fn = reinterpret_cast<AddFn>(dlsym(handle, "plugin_add"));
    const char* error = dlerror();
    if (error) {
        std::cerr << error << '\n';
        dlclose(handle);
        return 1;
    }

    std::cout << fn(2, 3) << '\n';
    dlclose(handle);
}
```

```bash
g++ -fPIC -shared plugin.cpp -o libplugin.so
g++ host.cpp -ldl -o host
./host
```

在 POSIX/glibc 的 `dlsym` 语义下，可按照相应 ABI 使用返回的函数地址；移植到非 POSIX 系统需要考虑函数指针转换的可移植性。

这里 `-ldl` 在部分现代 glibc 版本已经不再总是链接成功的硬需求，但保留在经典教学命令中可以帮助理解历史及旧发行版支持；其他 libc/平台要参照文档。

### `RTLD_LAZY` 与 `RTLD_NOW`

- `RTLD_NOW`：要求在返回前完成相关可解析函数绑定。
- `RTLD_LAZY`：允许某些函数符号延迟解析，但数据等重定位并不必然全都延迟；装载器和环境也可能强制立即绑定。
- `RTLD_LOCAL`：新装载对象的符号一般不进入后续一般性全局查找域；`RTLD_GLOBAL` 则可能让其符号供后续装载对象解析。

**`dlopen` 不等于“把某个文件的机器码直接跳进去”**；它仍要走共享对象装载、重定位、符号与依赖处理。

---

# 11. PIC、PIE、GOT、PLT 与 ASLR

这些缩写一开始很像黑话，但你已经理解重定位，再看它们会容易得多。

## 11.1 核心矛盾：同一个 `.so` 要在不同进程里放在不同地址

假设 `libmath.so` 在进程 A 被映射到 `0x7f...` 某处，在进程 B 被映射到另一处。为了让它的代码页可以被多个进程共享、且不必因为每个进程不同的装载地址而逐一修改大量代码字节，需要尽量生成**位置无关代码（PIC）**。

PIC 不是“代码不需要地址”，而是尽量通过：

- PC-relative 寻址；
- GOT（Global Offset Table）提供需要动态解析的地址槽；
- PLT（Procedure Linkage Table，或其他等效调用方案）；
- 动态重定位更新可写的数据槽；

来减少不可共享的代码修补。

## 11.2 `-fPIC` 和 `-shared` 的关系

```bash
g++ -fPIC -c lib.cpp -o lib.o
g++ -shared lib.o -o libdemo.so
```

- `-fPIC` 控制**编译时生成适合共享库的位置无关机器码**。
- `-shared` 控制**链接输出共享对象**。

二者不是一回事。只给 `-shared` 不代表输入的任意机器码都适合成为 x86-64 共享对象；非 PIC 输入有时会导致 `relocation ... can not be used when making a shared object; recompile with -fPIC`。

**静态库也可能需要 PIC**：如果你把一个 `.a` 的对象成员最终链接进 `.so`，这些被纳入 `.so` 的对象通常也需要满足共享库的 PIC 要求。因此“静态库不需要 PIC”不是普遍正确的结论。

## 11.3 `-fPIE`、`-pie`：共享库之外，可执行程序也能位置无关

- `-fPIE` / `-fpie`：生成适合位置无关**可执行程序**的代码。
- `-pie`：链接成 PIE 可执行文件。
- `-no-pie`：链接时禁用 PIE。

PIE 可以在不同运行中以不同基址映射，配合操作系统的 ASLR（Address Space Layout Randomization，地址空间布局随机化）。**PIC 和 PIE 的共同点是位置无关；共享库与主可执行文件对符号抢占、编译优化等语义不完全相同。**

想验证：

```bash
g++ -fPIE -pie main.cpp -o app_pie
g++ -fno-pie -no-pie main.cpp -o app_exec
readelf -hW app_pie | grep Type
readelf -hW app_exec | grep Type
```

常见 `app_pie` 为 `DYN`，`app_exec` 为 `EXEC`。具体默认和平台由构建环境决定。

## 11.4 GOT 是什么？

**Global Offset Table** 可以想成“地址簿上的若干可更新格子”。

比如一个共享库需要访问某个可动态解析的外部全局变量：

```text
共享库机器指令：
    ① 用相对地址找到 GOT 中的某个格子
    ② 从格子读出 global_var 的运行时地址
    ③ 再访问该地址的数据

GOT:
    [slot #7] ───> global_var 的实际运行时虚拟地址
```

程序装载时，动态加载器可以更新**GOT 条目**，而不必为每个使用地点修改只读代码页。现代编译器在某些条件下也会直接使用 PC-relative 寻址或 GOT 优化变体，并非每个全局引用都必经一个 GOT 间接访问。

## 11.5 PLT 是什么？为什么第一次函数调用可能更慢？

**Procedure Linkage Table** 是传统 ELF 动态链接实现中进行外部函数调用的跳板机制之一。

简化的惰性绑定路径：

```text
主程序 call printf@PLT
           │
           ▼
    PLT 跳板读取相关 GOT 槽
           │
           ├── 尚未解析 ──> 动态解析器查找 printf 地址
           │                      │
           │               将 GOT 槽更新为真正地址
           │                      │
           └──────────────────────┘
           │
           ▼
     后续调用通过已更新的地址跳转
```

这个示意能够解释：**某些调用可能在第一次执行时产生额外的绑定开销**，而不意味着每次都做全局符号名搜索。

注意实际实现复杂得多：

- 可能通过 `-Wl,-z,now`、`LD_BIND_NOW` 或发行版安全策略选择立即绑定。
- `-fno-plt` 可以改变某些外部调用的代码生成方式。
- 不同 ELF ABI、链接器、优化、CET/安全配置会改变 PLT 的具体形态。
- 部分调用可能被链接器优化为直接 PC-relative 调用，不需要传统 PLT 跳板。

所以你需要理解**间接调用与延迟绑定的设计动机**，而不是把所有 ELF 的函数调用都画成同一条固定链路。

## 11.6 PIE / ASLR 与重定位有什么关系？

假设主程序或共享库链接时使用相对基址进行布局，但运行时实际被映射到地址 `B`。一些动态重定位（例如架构相关的 `RELATIVE` 类型）可根据 `B + addend` 确定最终指针值。

这就解释了为什么 `readelf -r app` 仍可能存在重定位：**编译时不知道随机装载基址，链接时也不可能把所有运行时绝对地址永久写死。**

## 11.7 动态链接是否一定比静态链接慢？

不能简单地得出这个结论。差异来自：

- 启动时映射与重定位开销。
- 符号惰性/立即绑定成本。
- 某些跨 DSO 调用的间接跳转和符号抢占语义。
- 代码复用及缓存、TLB/I-cache、页面共享等效果。
- LTO、内联边界、可见性策略和实际工作负载。

对高吞吐/低延时程序，性能瓶颈通常需要具体测量：不要先验地相信“所有共享库都很慢”或“全部静态链接一定最快”。

---

# 12. `./app` 后到底发生什么：从 `execve` 到 `main`

对应 **CSAPP 第 23 页**。前面你学完 ELF、重定位、动态链接，现在可以把硬盘上的 ELF 和正在运行的进程真正接起来。

## 12.1 Shell 并不是“把 app 读进自己的进程然后跳到 main”

当你在 shell 输入：

```bash
./app hello world
```

shell 会按自己的命令执行策略建立子进程或相应进程执行环境，并最终通过 `execve` 等机制启动该程序。一般交互式 shell 对外部命令的常见模式是创建子进程并让它执行 `execve`，同时父 shell 等待或根据作业控制策略继续工作；具体实现细节可涉及 `fork`、`vfork`、`posix_spawn` 等。

`execve(path, argv, envp)` 并不是“在旧程序现有 `main` 函数后再追加一段代码”，而是**用新程序的映像替换当前进程的用户空间程序映像**（在成功时不返回），并按照约定启动它。

## 12.2 Linux 内核装载 ELF 的大概流程

1. 检查文件可执行性和格式，读取 ELF header/program headers。
2. 根据 `PT_LOAD` 等记录建立必要的虚拟内存映射，设置相应权限。
3. 对 BSS 等 `p_memsz > p_filesz` 的范围提供零初始化内存语义。
4. 建立用户栈初始内容：`argc`、`argv`、环境变量 `envp`、辅助向量 `auxv` 等。
5. 如果 ELF 指定 `PT_INTERP`，安排动态加载器参与启动。
6. 设置初始寄存器/入口点，移交控制权。

注意，这只是高层示意。实际内核需要处理地址随机化、内存保护、凭据、文件描述符继承、线程语义、vDSO 等大量细节。

## 12.3 为什么不是直接进入 main？

因为 `main` 之前通常还有启动运行时：

```text
Linux 内核（和必要的动态加载器）
        │
        ▼
ELF 入口点 _start
        │
        ▼
C 运行库启动代码（glibc 常见 __libc_start_main 路径）
        │
        ├── 运行需要的初始化逻辑
        ├── 初始化相关 C/C++ 运行时设施
        ├── 运行符合 ABI/库约定的全局构造器
        ▼
main(argc, argv, ...)
        │
        ▼
正常返回时经运行时完成终结/析构等流程
```

不同 C 库、工具链、链接选项、C++ 环境可能采用不同启动细节，但核心理念相同：**源码写的 `main` 并不必然是 ELF 程序真正的入口地址**。

观察：

```bash
readelf -hW app | grep 'Entry point'
objdump -d app | grep -A15 '<_start>:'
```

为什么驱动程序链接时需要 `crt*.o` 启动对象和必要运行库？它们帮助建立上述“`_start → main`”桥梁。**这也解释了为什么不应该把手工 `ld main.o -o app` 当作通常的 C++ 链接方式**。

## 12.4 进程的虚拟地址空间长什么样？

一个极度简化的 x86-64 Linux 进程示意：

```text
      高虚拟地址
┌────────────────────────────┐
│ 内核空间（用户态不可访问）  │
├────────────────────────────┤
│ 用户栈 stack（常见向低址扩）│  ← 局部自动变量/返回地址等
│ ...                        │
├────────────────────────────┤
│ mmap 区、共享库、匿名映射    │  ← 动态库、mmap 分配等
│ ...                        │
├────────────────────────────┤
│ heap 及其他可写映射         │  ← malloc/new 可能使用这些
├────────────────────────────┤
│ .data / .bss               │  ← 静态存储期可写数据
├────────────────────────────┤
│ .rodata / .text            │  ← 常量、机器码
├────────────────────────────┤
│ 其他映射/未使用区域         │
└────────────────────────────┘
      低虚拟地址
```

这张图**仅说明概念，不代表真实的固定顺序和边界**。现代系统中堆、栈、DSO、PIE 基址及映射区受 ASLR 和地址空间布局策略影响，`malloc` 也可能使用 `mmap` 而不只是传统 `brk` 堆。

### `stack` 与 `.bss` 的关键区别

```cpp
int global_array[1000];  // 静态存储期，通常 .bss

int f() {
    int local_array[1000];  // 自动存储期，常见栈或优化后的其他形式
    return local_array[0]; // 注意此处读取未初始化对象是有问题的
}
```

一个是程序映像中的静态存储期对象，一个是函数调用期间的自动对象。不能因为二者都叫“变量”就觉得它们都必须在 ELF `.data/.bss` 中有各自的 4000 字节。

### `new` / `malloc` 与运行时

- `new` 是 C++ 语言层面的分配/构造表达式，必要时调用 `operator new` 并构造对象。
- `malloc` 是 C 分配接口，通常经过 libc 分配器。
- 分配器可能通过 `brk` 和/或 `mmap` 等机制管理虚拟内存区域。
- `free` / `delete` 是释放对象或内存管理资源的接口，**不代表立刻把相关物理页面全部归还内核**。

因此，链接器的 `.bss/.data`、运行时堆、线程栈是三套互相关联但职责不同的概念。

## 12.5 一段可实际看到映射地址的程序

```cpp
#include <cstdio>
#include <cstdlib>

int global_initialized = 7;
int global_zero;

void function() {}

int main() {
    int local = 3;
    void* heap = std::malloc(64);
    std::printf("function=%p\n", (void*)&function);
    std::printf("global_initialized=%p\n", (void*)&global_initialized);
    std::printf("global_zero=%p\n", (void*)&global_zero);
    std::printf("local=%p\n", (void*)&local);
    std::printf("heap=%p\n", heap);
    std::free(heap);
}
```

此代码中将函数指针转换并用 `%p` 展示的做法偏向 Linux/x86-64 的教学观察，不应误认为所有 C++ 平台的函数指针和 `void*` 都具有同样表示。若追求完全可移植，改用调试器/反汇编工具观察函数地址。

多运行几次观察地址可能变化（取决于 ASLR、PIE 与系统策略）。再运行：

```bash
cat /proc/self/maps       # 查看当前 cat 进程自身的映射，并非你的程序
```

要查看**目标程序**，让它暂停或打印 PID：

```cpp
// 在目标程序适当位置打印 PID 并等待用户输入，再在另一个终端检查 /proc/<pid>/maps
```

`/proc/<pid>/maps` 显示进程当前映射：起止虚拟地址、权限、文件偏移、对应文件等，是连接“ELF 文件”与“运行时 VMA”的直接证据。

## 12.6 `argc/argv`、返回码与可执行文件是什么关系？

```cpp
#include <iostream>
int main(int argc, char* argv[]) {
    std::cout << "argc=" << argc << '\n';
    for (int i = 0; i < argc; ++i)
        std::cout << i << ": " << argv[i] << '\n';
    return 7;
}
```

```bash
g++ args.cpp -o args
./args first second
echo $?   # 7
```

**命令行参数由 shell 组织并通过 exec 系统调用等方式提供给新程序，不是链接器把 `first second` 写进了可执行文件。** 与你此前学过的标准输入、文件描述符、重定向知识也可以接起来：`./args 1 2` 是 argv，而 `./args < input.txt` 是 shell 在启动前改变文件描述符 0 的来源；两个机制完全不同。

## 12.7 文件被执行时，是把整个 ELF 一次拷入 RAM 吗？

不是必须这样。现代 Linux 常通过**内存映射、按需分页**等机制按需要带入代码/数据页。某些部分共享页缓存；某些可写映射按写时复制（COW）等机制处理。

所以：

- ELF 文件占用的磁盘字节数；
- PT_LOAD 声明的虚拟内存大小；
- 进程虚拟地址空间总量；
- 当前 RSS（实际常驻物理页）；

是不同概念。理解这四种大小，对以后阅读性能分析报告、内存占用指标非常重要。

---
# 13. C++ 比 C 多出来的链接复杂性

CSAPP 的原始教学示例多使用 C。它非常适合把符号/重定位讲干净，但你未来主要写 C++，还需要额外理解一层：**C++ 的语言实体比 C 复杂得多，而二进制链接器本质上仍要通过符号、节、重定位和 ABI 约定来协作。**

## 13.1 C++ 函数重载为什么需要 name mangling？

```cpp
int parse(int);
double parse(double);
```

两个函数都叫 `parse`，但形参不同。在源码中，C++ 编译器通过类型和重载决议区分它们。**跨 `.o` 文件交给链接器时不能只留下同一个字符串 `parse`**，否则无法正确配对。

因此 GCC 常采用 Itanium C++ ABI 风格的名字修饰，将函数名、命名空间、某些参数类型等信息编码到链接符号名中。

```cpp
// names.cpp
int parse(int x) { return x; }
double parse(double x) { return x; }
```

```bash
g++ -c names.cpp -o names.o
nm names.o             # 看原始符号名
nm -C names.o          # 尝试 demangle
c++filt _Z5parsei      # 看 mangled name 的可读形式（示例）
```

输出具体符号会受到编译器 ABI 的影响。**C++ 标准本身不规定所有平台必须使用同一套 mangling 方案**；MSVC 和 GCC/Clang 的 C++ ABI 通常不能直接当成二进制兼容。

## 13.2 为什么 `extern "C"` 经常出现在跨语言/插件接口？

```cpp
extern "C" int plugin_add(int a, int b);
```

`extern "C"` 指定该声明使用 C 语言链接方式；在典型 ELF/GCC 环境中可以避免常见的 C++ 函数名修饰，方便从 C 代码或 `dlsym(handle, "plugin_add")` 找到该名字。

注意：

- 它不是“把函数体改成 C 语言编译”。函数体仍由 C++ 编译器编译，仍可使用 C++ 特性。
- 不能借此让任意 C++ 对象布局、异常、`std::string`、模板接口变成跨编译器 ABI 安全。
- C ABI 也依赖目标架构及调用约定；`extern "C"` 不是跨不同 CPU 架构的可执行兼容魔法。

C/C++ 混合项目典型头文件：

```c
// api.h
#ifndef API_H
#define API_H

#ifdef __cplusplus
extern "C" {
#endif

int api_add(int a, int b);

#ifdef __cplusplus
}
#endif

#endif
```

这样一个 `.h` 可以分别被 C 和 C++ 翻译单元包含，使用正确的语言链接约定。

## 13.3 ODR：One Definition Rule（单一定义规则）

你最需要记住的是：**跨 TU 的定义规则不等于“只要名字不重复就没问题”**。

### 错误示例：头文件定义普通函数

```cpp
// helpers.h
int answer() { return 42; }
```

`a.cpp` 和 `b.cpp` 都 `#include "helpers.h"`，它们各自编译出一个外部 `answer()` 定义，链接时常报 multiple definition。

三种常见修法：

1. 头文件仅声明，在**一个** `.cpp` 中定义普通函数。
2. 真正适合放头文件的短实现使用 `inline`，满足 ODR 的一致性规则。
3. 模板通常把定义放头文件，按模板实例化/ODR 规则处理；或采用显式实例化方案。

### 反直觉：违反 ODR 有时不会报链接错误

例如两个 TU 从不同宏状态包含同一个 `inline` 定义，虽然最终产生相同符号名，但函数体语义不同。链接器可能选择/合并某个定义，而不会做完整的“两个 C++ 定义是否逐 token 一致”检查。

结果可能是**未定义行为、无诊断要求（IFNDR）**。所以：**链接成功不是 C++ 程序满足 ODR 的证明**。

## 13.4 模板为什么常报 undefined reference？

```cpp
// math.h
template<typename T>
T add(T a, T b); // 只有声明
```

```cpp
// math.cpp
template<typename T>
T add(T a, T b) { return a + b; }
```

```cpp
// main.cpp
#include "math.h"
int main() { return add<int>(1, 2) - 3; }
```

编译 `main.cpp` 时，它没有看到模板定义；编译 `math.cpp` 时，也不一定实际需要实例化 `add<int>`。最后可能没有任何对象文件提供这个具体的模板函数实例，链接报错。

可选修复：

- 把模板定义放在头文件（最常见）。
- 或在 `.cpp` 中显式实例化：

```cpp
template int add<int>(int, int);
```

并正确管理对外声明和 ABI。

## 13.5 “undefined reference to vtable for X” 到底意味着什么？

例如：

```cpp
// base.h
struct Base {
    virtual ~Base();
    virtual void run();
};
```

如果只声明虚析构或某个必要的非纯虚虚函数，却没有提供其应有的定义：

```cpp
// base.cpp
#include "base.h"
Base::~Base() = default;
void Base::run() {}
```

缺少对应定义时，某些 Itanium C++ ABI 实现会导致链接阶段报告 vtable/typeinfo 相关未定义引用。原因涉及**关键虚函数（key function）及虚表发射位置**等约定。

排查思路：

1. 虚析构是否只声明没定义？即使声明为纯虚析构，也通常需要提供定义。
2. 某个虚函数的实现文件是不是忘了加入 Target？
3. `.cpp` 是否和头文件的命名空间、const/ref 限定、参数列表匹配？
4. 是否跨库隐藏了必要的 RTTI/vtable 符号？

**不是所有 vtable 报错都可以通过“加一个虚析构函数”来解决**；要找出真正缺失的定义/ABI 关系。

## 13.6 ABI（Application Binary Interface）是什么？

源码级 API 告诉你：

```cpp
int calculate(int x);
```

ABI 则规定编译结果如何互相配合，包括：

- 函数参数如何传递，寄存器/栈如何分工。
- 返回值放哪里。
- 调用者与被调用者各自保存哪些寄存器。
- 栈对齐要求。
- 数据类型大小、对齐、结构体布局。
- C++ 名字修饰、异常、RTTI、虚表等对象模型约定。
- 动态库符号和版本的兼容策略。

以典型 **System V x86-64 ABI** 下的简单整数/指针参数为例，前几个通用参数常通过 `rdi`、`rsi`、`rdx`、`rcx`、`r8`、`r9` 等寄存器传递；整数返回值常放在 `rax`。但**这只是分类规则的一部分**：浮点/向量参数、复合对象、变参函数、C++ 特殊返回值等有更复杂的规则。不要把“六个寄存器”当成适用于所有类型的万能口诀。

### 为什么 ABI 与链接紧密相关？

假设函数双方成功按名字链接，但是一方认为某个结构体大小 16 字节，另一方认为 24 字节；或者一方按照一种调用约定传参、另一方按照另一种取参。**即使找到了符号名，运行仍可能出错**。

链接器主要负责二进制符号和重定位，通常不可能在链接时完整检查双方的所有 C++ 类型语义/对象布局。

## 13.7 C++ 标准库 ABI 与 `_GLIBCXX_USE_CXX11_ABI`

在 GNU libstdc++ 生态中，历史上的 dual ABI（双 ABI）机制可能让部分标准库类型（如 `std::string` 的一些接口）在不同编译选项/库环境中使用不同 ABI。

你可能看到：

```text
std::__cxx11::basic_string<...>
```

或者：

```text
[abi:cxx11]
```

出现此类 undefined reference 时，不能简单地认定“标准库没有安装”。它可能是：

- 你编译的目标和第三方预编译库使用不兼容 ABI。
- 不同 GCC/libstdc++ 版本、构建宏、接口签名造成不匹配。
- 链接时混用了不一致的运行库。

`_GLIBCXX_USE_CXX11_ABI` 是 libstdc++ 特有的构建兼容选项，并不是 C++ 标准语言特性，更不是解决所有 ABI 不兼容的万能开关。**先确认第三方库的构建说明与 ABI 约定，不要盲目修改宏直到“能链接”。**

## 13.8 共享库边界上的 C++ 例外、RTTI 与资源管理

如果动态库 A 抛出 C++ 异常，调用者 B 需要跨 DSO 捕获，双方通常必须使用兼容的异常处理 ABI、runtime 和类型信息策略。GCC 还会涉及 `libgcc_s`、`libstdc++` 等运行库安排。

跨插件接口直接传递 `std::string`、STL 容器、C++ 类对象，甚至让 A 分配、B 用不兼容的分配器释放，都会引入 ABI 风险。

**实用建议**：如果你将来为 HTTP Server 或交易系统设计可动态升级的插件，跨 DSO 接口可以优先考虑稳定的 C ABI、显式版本号、opaque handle（不透明句柄）、清晰的分配/释放责任，而不是默认暴露全部内部 C++ 类布局。

## 13.9 全局对象的构造器何时执行？

```cpp
struct Logger {
    Logger();
    ~Logger();
};

Logger global_logger;
```

具有静态存储期的 C++ 对象若需要动态初始化，启动运行时通常会根据链接器输出的初始化元数据（如 `.init_array`）安排构造；退出时根据相关机制安排终结/析构。

这引出“静态初始化顺序问题”：跨翻译单元的动态初始化顺序不应被随意假定。某个全局对象构造时使用另一个尚未初始化的全局对象，可能出错。

在库化工程里，这不是纯语言教科书的问题：**静态库成员是否被抽取、共享库何时被加载、初始化函数什么时候运行**，都可能影响最终行为。不要依赖“没有显式调用就一定不会运行任何代码”；全局构造器便是反例。

---

# 14. 课件最后的案例：三种函数拦截

对应 **CSAPP 第 38～49 页**。这部分最容易让人迷糊的不是宏语法，而是：**为什么同样一个 `malloc()` 调用，可以在不同时间点被改成调用我们自己的函数？**

所谓 **library interposition（库函数拦截/插桩）**：把程序某些原本指向函数 A 的调用，改为经过包装函数 B，以便记录日志、计数、检查、注入测试逻辑等。课件给了三种时间点。

## 14.1 第一种：编译期拦截（preprocessor macro）

```c
// wrapper.h
void *my_malloc(size_t n);
#define malloc(n) my_malloc(n)
```

程序 `#include` 它后：

```c
void *p = malloc(32);
```

预处理阶段就可能变成：

```c
void *p = my_malloc(32);
```

这不是 ELF 动态链接器的黑科技，而是**源码进入编译器之前就被宏替换了函数名**。课件第 42～43 页演示了这种做法。

特点：需要控制源码的头文件/编译方式；宏容易影响名字、表达式求值和作用域；实现包装函数自身时要避免宏递归。通常只适合明确可控的代码库。

## 14.2 第二种：链接期拦截（GNU ld `--wrap`）

GNU ld 提供专门的 `--wrap=foo` 机制：对于符合条件的**未定义外部引用**，把对 `foo` 的引用转到 `__wrap_foo`，并允许 `__wrap_foo` 使用 `__real_foo` 引用真正的实现。

```c
// base.c
int work(int x) { return x + 1; }
```

```c
// main.c
int work(int);
int main(void) { return work(3) - 4; }
```

```c
// wrap.c
#include <stdio.h>
int __real_work(int);

int __wrap_work(int x) {
    fprintf(stderr, "before work(%d)\n", x);
    int answer = __real_work(x);
    fprintf(stderr, "after work -> %d\n", answer);
    return answer;
}
```

构建：

```bash
gcc -c main.c -o main.o
gcc -c base.c -o base.o
gcc -c wrap.c -o wrap.o
gcc main.o base.o wrap.o -Wl,--wrap=work -o app
./app
```

关键逻辑：

```text
main.o 中引用 work
     │
     ▼
链接器 --wrap=work
     │
     ▼
__wrap_work
     │
     └── __real_work ──由链接器重新映射──> work
```

**特别注意**：`--wrap` 针对的是链接器能看到的相应未定义引用。如果函数调用在同一个对象文件中就已经由编译器/汇编器就地绑定，或者编译器进行了内联，预期的 wrap 可能不会发生。这也是实验要求把三个部分分开编译的原因。课件第 44～45 页演示了用 `__wrap_malloc` 和 `__real_malloc` 拦截 malloc。

## 14.3 第三种：装载/运行期拦截（`LD_PRELOAD`）

在常规 glibc 动态链接情形下，可以预先装载一个 `.so`，让其中的某些全局默认可见符号在正常动态符号搜索中被优先找到。

先使用自定义 `greet` 观察，比直接改 `malloc` 更安全：

```c
// base.c
#include <stdio.h>
void greet(void) { puts("base greet"); }
```

```c
// override.c
#include <stdio.h>
void greet(void) { puts("PRELOAD greet"); }
```

```c
// main.c
void greet(void);
int main(void) { greet(); }
```

```bash
gcc -fPIC -shared base.c -o libbase.so
gcc -fPIC -shared override.c -o liboverride.so
gcc main.c -L. -lbase -Wl,-rpath,'$ORIGIN' -o app

./app
LD_PRELOAD="$PWD/liboverride.so" ./app
```

常见输出：

```text
base greet
PRELOAD greet
```

第一次正常动态解析来自 `libbase.so`；第二次预加载对象中的同名符号先被找到，于是改为 `override.c` 的实现。

**它不是修改 `main.c`，也不是重新链接 app**，而是在动态加载/符号解析层改变了外部引用的归属。课件第 46～48 页针对 `malloc/free` 演示了这个思路。

## 14.4 拦截 malloc 为什么特别危险？

课件中的 `malloc` wrapper 以 `dlsym(RTLD_NEXT, "malloc")` 找真正分配器，再打印分配日志。教学上非常好，但现代真实工程必须警惕：

- `printf` / `fprintf` / `dlsym` 等操作可能间接需要分配内存，导致**递归重入**。
- 在分配器尚未完全可用的早期初始化期间，自定义 wrapper 可能被调用。
- 多线程下的重入、锁与初始化次序可能死锁。
- 标准库/优化器的内部调用不一定都走你期望的可抢占动态符号路径。
- `LD_PRELOAD` 会被某些安全执行环境限制或忽略，不是能够干预所有进程的万能开关。
- 观察本身可能极大改变性能；对低延时 benchmark 尤其需要谨慎。

**本教程推荐先做上面的 greet 拦截实验，理解机制后再阅读课件里的 malloc 示例；不要把演示版 malloc wrapper 直接拿去做生产级内存分析器。**

## 14.5 `RTLD_NEXT` 是什么意思？

在 glibc 的 ELF 动态符号查找机制中，`dlsym(RTLD_NEXT, "malloc")` 大致表示：在调用者后续的符号查找范围中寻找下一个匹配的 `malloc` 实现。典型用途是：wrapper 抢占 `malloc`，但自身又要调用原来的实现。

它依赖动态加载器/调用者地址等上下文，**不是 C++ 标准库提供的通用功能**。即使能找到下一个实现，也仍然需要解决初始化、线程安全、递归、性能等工程难题。

## 14.6 三种拦截机制放在同一张表里

| 方法 | 生效时机 | 改变的对象 | 典型限制 |
|---|---|---|---|
| 宏替换 | 预处理/编译 | 源码标识符展开 | 必须影响源码的预处理过程 |
| `--wrap` | 静态链接 | 特定未定义引用的符号解析 | 需要链接器支持，内部已绑定调用未必拦得到 |
| `LD_PRELOAD` | 动态加载/运行 | 共享对象符号搜索的优先关系 | 取决于动态链接、可见性、绑定和安全模式 |

你的收获应当是：**函数调用并非只在“写源码那一刻”被固定。不同阶段都可能影响它最终指向谁。** 这也是研究 profiler、sanitizer、malloc tracing、Linux 调试工具的一个很好的入口。

---

# 15. 把 GCC 与链接知识一一映射到 CMake

现在再看你上一篇 CMake 教程，许多命令就不再需要背。**CMake 描述 target 的构建需求，底层构建系统决定何时执行，GCC/G++ 驱动和链接器完成具体翻译与链接。**

## 15.1 从手写命令回到 CMake

假设手写是：

```bash
g++ -std=c++20 -O2 -Wall -Wextra -Iinclude \
    -c src/math.cpp -o math.o
ar rcs libmath.a math.o

g++ -std=c++20 -O2 -Wall -Wextra -Iinclude \
    -c src/main.cpp -o main.o

g++ main.o libmath.a -o app
```

现代 CMake 可以写：

```cmake
cmake_minimum_required(VERSION 3.20)
project(LinkingDemo LANGUAGES CXX)

add_library(math STATIC src/math.cpp)

target_compile_features(math PUBLIC cxx_std_20)

target_include_directories(math PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE math)

target_compile_options(app PRIVATE
    "$<$<CXX_COMPILER_ID:GNU,Clang>:-Wall;-Wextra>"
)
```

你现在应该能逐行解释：

- `add_library(math STATIC ...)`：定义静态库 Target，底层会安排其 `.o` 编译和归档输出。
- `target_compile_features(math PUBLIC cxx_std_20)`：math 自身与使用者都需要对应 C++ 语言特性要求。
- `target_include_directories(math PUBLIC include)`：编译 math 及其消费者时要能找到对外公开的头文件。
- `add_executable(app ...)`：构建可执行 Target。
- `target_link_libraries(app PRIVATE math)`：建立 app 使用 math 的链接依赖；CMake负责转化为合适的构建关系和链接顺序，而不是简单拼接一个字符串。
- `target_compile_options(app PRIVATE ...)`：只给 app 自身提供警告选项；它不是给链接器的参数。

注：上面生成器表达式在要求的 CMake 版本中对 GNU/Clang 选择生效；实际跨平台项目还需考虑 AppleClang/MSVC 的选项差异。简单场景也可以直接用 `if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang") ...`。

## 15.2 现代 CMake 为什么把 include 和 link 分开？

因为这两个事情发生在不同阶段：

```cmake
target_include_directories(app PRIVATE include)
```

影响编译时 `#include` 搜索，类似 `-I`。

```cmake
target_link_libraries(app PRIVATE math)
```

建立链接依赖，可能让底层链接命令包含 `libmath.a` / 共享库目标路径或其他合适链接形式。

**一个头文件可以声明 API，但不提供其实现；一个库可以提供实现，却不能让编译器凭空猜对声明。** 这就是两类命令分开的原因。

## 15.3 为什么 PUBLIC / PRIVATE / INTERFACE 跟链接有关？

考虑一条依赖链：

```text
app ──> server ──> network
```

如果 `server` 的公开头文件暴露了 `network` 的类型或头文件，`app` 编译它时需要 network 的对应使用要求。于是：

```cmake
target_link_libraries(server PUBLIC network)
```

让 server 消费者继承适当的 network 使用要求。

如果 `server` 的 `.cpp` 内部用了 `network`，但对外头文件完全不需要该依赖，可以倾向：

```cmake
target_link_libraries(server PRIVATE network)
```

**细节别误会**：对静态库而言，`PRIVATE` 的依赖仍可能需要通过某些“仅链接、非编译接口”的关系传递给最终消费者，以确保最终可执行程序能够解析静态库对象的外部引用。因此不能粗暴理解成“PRIVATE 在任何后续链接命令里都绝不会出现”。CMake 的 `INTERFACE_LINK_LIBRARIES` 与 `$<LINK_ONLY:...>` 等机制正是为细致区分使用要求服务的。

判定依据仍是：**当前模块是否需要该配置？使用这个模块的其他模块在编译/链接时又需要什么？**

## 15.4 三种库在 CMake 里如何对应？

```cmake
add_library(foo STATIC foo.cpp)    # 普通静态库归档
add_library(bar SHARED bar.cpp)    # 动态库 / 共享对象
add_library(headers INTERFACE)    # 纯接口 target，不产生一般二进制库
```

`INTERFACE` 不是 `.a` 与 `.so` 的第三种物理二进制格式；它是 **CMake target 模型**中的一种逻辑接口容器，常用来传递头文件目录、宏、语言特性等要求。

## 15.5 `POSITION_INDEPENDENT_CODE` 与 `-fPIC`

```cmake
add_library(plugin SHARED src/plugin.cpp)
```

现代 CMake 对 `SHARED` / `MODULE` 目标通常会自动启用合适的 `POSITION_INDEPENDENT_CODE` 初始属性。对于准备放入共享库的静态库，可显式指定：

```cmake
add_library(core STATIC src/core.cpp)
set_target_properties(core PROPERTIES POSITION_INDEPENDENT_CODE ON)

add_library(plugin SHARED src/plugin.cpp)
target_link_libraries(plugin PRIVATE core)
```

这体现了“`core` 最后是静态归档，但它的对象最终可能被纳入动态库，因此需要 PIC”的知识。

## 15.6 `target_compile_options()` 与 `target_link_options()`

```cmake
target_compile_options(app PRIVATE -Wall -Wextra)
```

针对编译阶段。

```cmake
target_link_options(app PRIVATE
    "LINKER:-Map=app.map"
)
```

针对链接阶段，`LINKER:` 是 CMake 跨编译器驱动传递链接器参数的一种方式。输出路径是否需要绝对化、不同链接器是否支持 map、生成器表达式如何用于多配置，都应按实际项目进一步细化。

如果你看到有人把所有链接选项都塞进 `CMAKE_CXX_FLAGS`，你现在就知道它为什么不够清晰：**编译阶段和链接阶段本来就有不同参数、不同职责**。

## 15.7 如何在 CMake 中指定优化、Debug、Sanitizer？

首先区分两种生成器：

- Ninja / Unix Makefiles（单配置）常用 `-DCMAKE_BUILD_TYPE=Debug/Release`。
- Visual Studio / Ninja Multi-Config（多配置）通常构建时指定 `--config Debug/Release`。

CMake 的内置构建类型一般会提供合理的默认优化/调试参数，**不必再盲目用 `target_compile_options(... -O3 -g)` 把所有配置硬盖掉**。

Sanitizer 示例（只在 GCC/Clang 等支持环境中使用）：

```cmake
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)

if(ENABLE_ASAN)
    target_compile_options(app PRIVATE
        -fsanitize=address
        -fno-omit-frame-pointer
    )
    target_link_options(app PRIVATE
        -fsanitize=address
    )
endif()
```

为什么需要两处？因为 ASan 既要对代码插桩，也需要在最终链接中接入 sanitizer 运行时。这与你第 5 章的知识完全对应。

## 15.8 外部依赖：`find_package()` 不是“神秘下载器”

例如：

```cmake
find_package(Threads REQUIRED)
add_executable(server src/server.cpp)
target_link_libraries(server PRIVATE Threads::Threads)
```

`Threads::Threads` 是 CMake 的导入目标，携带对应平台需要的使用要求。它可能让编译/链接使用 `-pthread`，也可能在某些平台上不需要额外的显式库参数。

另一个例子：

```cmake
find_package(OpenSSL REQUIRED)
target_link_libraries(server PRIVATE OpenSSL::SSL OpenSSL::Crypto)
```

前提是安装并找到兼容的 OpenSSL 开发依赖。**头文件、共享对象、链接目标、运行时库路径是四件不同的事**，`find_package` 只是帮助 CMake 把这些关系表达得更可靠。

## 15.9 动态库运行时路径和安装

CMake 可以通过 `BUILD_RPATH`、`INSTALL_RPATH`、安装规则及适当的 target 属性管理构建树/安装树中的运行时库定位，不必每个项目都硬编码 `-Wl,-rpath,...`。

例如教学用：

```cmake
set_target_properties(app PROPERTIES
    BUILD_RPATH "$ORIGIN/../lib"
    INSTALL_RPATH "$ORIGIN/../lib"
)
```

这段路径是否正确，取决于你的**实际可执行文件和库的输出/安装目录布局**，不能直接复制到任何项目都指望它工作。真正安装库还要进一步学习 `install(TARGETS ...)`、`GNUInstallDirs`、导出 targets 与包配置。

## 15.10 CMake 为什么比一条巨大 `g++` 命令适合大工程？

不是因为 CMake 比链接器更懂如何修机器码，而是它能管理：

- 一个项目有哪些 Target。
- 每个 Target 的 `.cpp` 属于哪些翻译单元。
- 头文件、宏、语言标准、编译选项等使用要求。
- Target 间的依赖、合理的构建顺序与链接关系。
- Debug / Release 与不同工具链配置。
- 哪些文件修改后要重新构建哪些对象。
- 测试、安装、生成文件、第三方包和跨平台构建。

**CMake 管项目的构建关系，Make/Ninja 执行依赖图，编译器制作 `.o`，链接器处理跨对象符号与重定位，内核/动态加载器让 ELF 真正跑起来。** 到这里，它们终于站在了同一条因果链上。

---
# 16. 值得亲手做的实验

下面不是考试题，而是一份**可反复使用的实验手册**。建议在 Linux/WSL 中做，安装 `g++ gcc binutils make cmake ninja-build`；可选 `gdb`、`strace`。如果你使用 CLion，可以打开同一文件夹，并在它的内置终端执行命令。

## 实验前约定

1. 每个实验尽量放在独立目录，避免同名 `.o/.so` 混淆。
2. 如果命令刻意失败，这是教学设计；读懂报错是实验的一部分。
3. 二进制地址、`nm` 输出的精确偏移与 `objdump` 指令排列可能因 GCC 版本、优化、PIE 默认值、CPU 不同而改变。
4. 尽量先**预测**结果，再运行，再解释。
5. 实验中为了观察可能使用 `-O0 -fno-pie -no-pie`，它们不是生产构建默认建议。

---

## 实验 1：亲眼看懂 -E / -S / -c / 链接

**目标**：不再把 `g++` 当黑箱。

新建目录 `lab01/`，创建：

```cpp
// main.cpp
#define SCALE 2
int square(int x) { return x * x; }
int main() { return square(SCALE + 1) - 9; }
```

执行：

```bash
g++ -E main.cpp -o main.ii
g++ -S -O0 main.cpp -o main.s
g++ -c -O0 main.cpp -o main.o
g++ main.o -o app

file main.ii main.s main.o app
ls -lh main.ii main.s main.o app
```

再执行：

```bash
g++ -v main.cpp -o app_verbose 2> build.log
grep -E 'cc1plus|as |collect2|ld' build.log | head -20
```

**观察/解释**：

- `.ii` 可读，能看到宏 `SCALE` 的效果。
- `.s` 是汇编文本。
- `.o` 是 ELF relocatable object。
- `app` 是 ELF 可执行程序（默认可能为 PIE/ET_DYN）。
- 使用 `-v` 可以知道 gcc/g++ 背后调用了哪些工具。

**自问**：如果添加一个 `math.cpp`，为什么不能只把它放在文件夹就指望 `main.o` 自动找到它？

## 实验 2：把头文件、编译和链接错误故意分开

```cpp
// math.h
#pragma once
int add(int, int);
```

```cpp
// math.cpp
#include "math.h"
int add(int a, int b) { return a + b; }
```

```cpp
// main.cpp
#include "math.h"
int main() { return add(2, 3) - 5; }
```

执行：

```bash
g++ -c main.cpp -o main.o
nm -C main.o | grep -E 'add|main'

g++ main.o -o app              # 预期链接失败：缺少 add 定义
g++ -c math.cpp -o math.o
g++ main.o math.o -o app
./app
echo $?
```

现在把 `math.h` 放进 `include/`，修改 `#include` 路径，再尝试不加 `-I` 编译。

**观察/解释**：

- 头文件不存在或搜索目录错误：预处理/编译阶段无法完成。
- `main.o` 能生成，但引用 `add` 尚未解析：这正常。
- 忘记 `math.o`：链接阶段失败。
- 添加 `math.o`：链接器找到定义，完成构建。

## 实验 3：用 ELF 头、节表和符号表看 `.o`

沿用实验 2 的 `main.o`：

```bash
file main.o
readelf -hW main.o
readelf -SW main.o
readelf -sW main.o
nm -C main.o
objdump -drC main.o
```

**观察问题**：

1. `Type` 是什么？`Machine` 是什么？
2. 能找到 `.text` / `.symtab` / `.rela.text` 吗？
3. `add(int,int)` 为什么显示 `UND` 或 `U`？
4. 哪个位置有一条对外部符号的调用重定位？

**自问**：我看到的是“运行中的程序”还是“供链接器继续工作的文件”？

## 实验 4：算出你自己的 `call rel32`

建立 `lab04/`：

```cpp
// main.cpp
extern int global_value;
int square(int);
int main() { return square(global_value); }
```

```cpp
// square.cpp
int global_value = 7;
int square(int x) { return x * x; }
```

```bash
g++ -O0 -fno-pie -c main.cpp -o main.o
g++ -O0 -fno-pie -c square.cpp -o square.o
readelf -rW main.o
objdump -drC main.o

g++ -no-pie main.o square.o -o app
objdump -dC app | sed -n '/<main>:/,+25p'
nm -nC app | grep -E 'main|square|global_value'
```

**你应该能看到**：类似 `R_X86_64_PC32 ... - 4` 和 `R_X86_64_PLT32 ... - 4` 的重定位记录，链接后变成具体可用的位移。

**动手计算**：

1. 找出 `call` 指令起始虚拟地址 `C`。
2. 找出它的下一条指令地址 `C+5`（若为常见 5 字节 rel32 call）。
3. 找出调用目标地址 `S`。
4. 算 `S-(C+5)`。
5. 看 `call` 机器码中 4 字节位移是否符合小端序编码结果。

这一步如果亲手算明白，CSAPP 第 21～22 页会彻底通透。

## 实验 5：strong/weak、COMMON 与现代 GCC

新建 `lab05/`：

```c
// a.c
int x;
int fa(void) { return x; }
```

```c
// b.c
int x;
int fb(void) { return x; }
```

```c
// main.c
int fa(void), fb(void);
int main(void) { return fa() + fb(); }
```

现代 GCC 默认：

```bash
gcc -c a.c -o a.o
gcc -c b.c -o b.o
gcc -c main.c -o main.o
nm a.o | grep ' x'
nm b.o | grep ' x'
gcc main.o a.o b.o -o app      # 预期 multiple definition
```

旧式 C COMMON 模型复现：

```bash
gcc -fcommon -c a.c -o a_common.o
gcc -fcommon -c b.c -o b_common.o
nm a_common.o | grep ' x'    # 常见 C x
nm b_common.o | grep ' x'
gcc main.o a_common.o b_common.o -o app_common
```

**为什么值得做**：你会发现 2015 年课件第 16～18 页不是“数学定理式永远成立的现代 GCC 默认行为”，而是有其 C 编译选项背景。**正式代码用头文件 `extern int x;` 声明、单一 `.c/.cpp` 定义**。

## 实验 6：制造一个头文件 ODR 冲突

```cpp
// bad.h
#pragma once
int value() { return 42; }
```

```cpp
// a.cpp
#include "bad.h"
int a() { return value(); }
```

```cpp
// b.cpp
#include "bad.h"
int b() { return value(); }
```

```cpp
// main.cpp
int a(), b();
int main() { return a() + b() - 84; }
```

```bash
g++ -c a.cpp b.cpp main.cpp
g++ a.o b.o main.o -o app      # 常见 multiple definition
```

把 `bad.h` 改为：

```cpp
#pragma once
inline int value() { return 42; }
```

重新编译链接：

```bash
g++ -c a.cpp b.cpp main.cpp
g++ a.o b.o main.o -o app
```

**自问**：`#pragma once` 不是已经避免重复包含了吗？答案：它只能避免同一翻译单元的重复展开，不能跨多个 TU 让普通外部定义“自动只有一份”。

## 实验 7：自己打包静态库并故意排错顺序

```cpp
// math.cpp
int add(int a, int b) { return a + b; }
```

```cpp
// main.cpp
int add(int, int);
int main() { return add(2, 3) - 5; }
```

```bash
g++ -c math.cpp -o math.o
g++ -c main.cpp -o main.o
ar rcs libmath.a math.o
ar t libmath.a
nm -C libmath.a

g++ main.o -L. -lmath -o app   # 通常成功
g++ -L. -lmath main.o -o bad   # 传统 GNU 静态归档顺序下通常失败
```

进一步：

```bash
g++ main.o -L. -lmath -Wl,-Map=app.map -o app
less app.map
```

试着找到 `libmath.a` 中哪个对象成员被抽取、为什么被抽取。**这是一条直接连接到 CMake `target_link_libraries()` 的实验。**

## 实验 8：`.a` 不是“整个全部塞进去”

```cpp
// used.cpp
int used() { return 7; }
```

```cpp
// unused.cpp
int unused() { return 99; }
```

```cpp
// main.cpp
int used();
int main() { return used() - 7; }
```

```bash
g++ -O0 -c used.cpp unused.cpp main.cpp
ar rcs libboth.a used.o unused.o
g++ main.o libboth.a -o app
nm -C app | grep -E 'used|unused'
```

一般会看到 `used()`，而 `unused()` 未必被抽取。再尝试：

```bash
g++ main.o -Wl,--whole-archive libboth.a \
    -Wl,--no-whole-archive -o app_all
nm -C app_all | grep -E 'used|unused'
```

**自问**：为何 `--whole-archive` 与 `--gc-sections` 不是同一粒度的事情？

## 实验 9：动态库三种定位方式：-L、LD_LIBRARY_PATH、RUNPATH

```cpp
// lib.cpp
extern "C" int triple(int x) { return x * 3; }
```

```cpp
// main.cpp
#include <cstdio>
extern "C" int triple(int);
int main() { std::printf("%d\n", triple(7)); }
```

```bash
g++ -fPIC -shared lib.cpp -o libtriple.so
g++ main.cpp -L. -ltriple -o app
readelf -dW app | grep -E 'NEEDED|RUNPATH|RPATH'
./app                         # 某些环境预期找不到 libtriple.so
LD_LIBRARY_PATH="$PWD" ./app # 通常输出 21

g++ main.cpp -L. -ltriple -Wl,-rpath,'$ORIGIN' -o app_rpath
readelf -dW app_rpath | grep -E 'NEEDED|RUNPATH|RPATH'
./app_rpath                   # 通常输出 21
```

解释为什么系统默认搜索目录一般不含当前工作目录；为什么 `.so` 在链接阶段被找到了，运行时仍然可能找不到。

## 实验 10：PIC、PIE 和进程中的实际地址

新建：

```cpp
// main.cpp
#include <cstdio>
int value = 1;
int main() {
    std::printf("global address = %p\n", (void*)&value);
    return 0;
}
```

```bash
g++ -fPIE -pie main.cpp -o pie_app
g++ -fno-pie -no-pie main.cpp -o nonpie_app
readelf -hW pie_app | grep Type
readelf -hW nonpie_app | grep Type

./pie_app
./pie_app
./nonpie_app
./nonpie_app
```

在启用 ASLR 的常见 Linux 环境下，PIE 中 `value` 地址可能变化；非 PIE 主程序里该静态全局的地址常更稳定。**不要把这个观察上升为“所有 Linux 配置下每次都必变/必不变”的定律**。ASLR 可以被系统配置、调试器、容器环境等影响。

## 实验 11：`dlopen` 的真正意义

使用第 10.9 节中的 `plugin.cpp` 与 `host.cpp`：

```bash
g++ -shared -fPIC plugin.cpp -o libplugin.so
g++ host.cpp -ldl -o host
./host
```

检查：

```bash
readelf -dW host | grep libplugin
```

通常不会在 `DT_NEEDED` 里看到 `libplugin.so`，因为 host 没有在普通链接命令中建立该库的强制依赖，而是**运行时 `dlopen` 再加载**。

再修改 `dlopen("./libplugin.so", ...)` 的路径为不存在的文件，理解为什么这是运行时错误，不是链接时错误。

## 实验 12：观察 C++ name mangling

```cpp
// funcs.cpp
int parse(int x) { return x; }
double parse(double x) { return x; }
extern "C" int c_parse(int x) { return x; }
```

```bash
g++ -c funcs.cpp -o funcs.o
nm funcs.o | grep -E 'parse'
nm -C funcs.o | grep -E 'parse'
```

**应该观察**：重载的两个 C++ 函数在原始符号层面有不同的 mangled names；`c_parse` 通常保留 C 风格链接符号名。把函数名改到命名空间中，再看看符号变化。

## 实验 13：链接期 `--wrap` 与动态库 `LD_PRELOAD` 拦截

第 14.2、14.3 节提供了完整三文件代码和命令，建议两个都做。

为了避免“只是在抄命令”，你需要回答：

1. `--wrap` 时，哪个**输入 `.o`** 包含了对 `work` 的未定义引用？
2. `__real_work` 为什么没有在普通源码里定义却能得到真正的函数实现？
3. `LD_PRELOAD` 实验中 `app` 的磁盘文件是否被重新生成？
4. 如果把 `greet` 静态链接入程序，或在库内部直接绑定/隐藏符号，会有哪些不同？
5. 为什么生产级 malloc 拦截实现容易踩递归初始化的坑？

## 实验 14：调试和优化不是一条坐标轴

```cpp
// main.cpp
#include <cstdio>
int square(int x) { return x * x; }
int main() {
    int y = square(5);
    std::printf("%d\n", y);
}
```

```bash
g++ -O0 -g main.cpp -o debug_app
g++ -O2 -g main.cpp -o opt_debug_app
g++ -O2 main.cpp -o opt_app

objdump -dC debug_app | sed -n '/<main>:/,+25p'
objdump -dC opt_debug_app | sed -n '/<main>:/,+25p'
readelf -SW debug_app | grep '\.debug'
```

**思考**：

- 为什么同样 `-g`，GDB 看到的局部变量有时被优化掉？
- 为什么 `-O2` 下 `square(5)` 可能不再对应一个真正的函数调用？
- 如果研究低延时性能，为什么不能用完全不同的构建配置随意对比数值？

## 实验 15：用 ASan 证明“链接需要运行时”

```cpp
// bad.cpp
int main() {
    int* p = new int[2];
    p[3] = 7;  // 故意越界：只用于 sanitizer 实验
    delete[] p;
}
```

```bash
g++ -g -O1 -fsanitize=address -fno-omit-frame-pointer \
    bad.cpp -o bad_asan
./bad_asan  # 预期报告内存越界，不要把它用于生产代码
```

再试：

```bash
g++ -g -O1 -fsanitize=address -c bad.cpp -o bad.o
g++ bad.o -o bad_missing_runtime     # 通常链接时出现缺少 ASan 运行库符号
```

这个实验极直观地说明：**编译期插桩产生了新的外部符号引用，链接阶段必须提供对应运行时库**。

## 实验 16：利用 map、symtab 和动态节完成一次“二进制体检”

挑自己写的任何一个小型 C++ 项目：

```bash
file ./app
readelf -hW ./app
readelf -lW ./app
readelf -dW ./app
readelf -SW ./app
nm -C ./app | head -40
objdump -dC ./app | head -80
```

尝试回答：

- 这是 ET_EXEC 还是 ET_DYN？是否 PIE？
- 入口点在哪里？
- 哪些 PT_LOAD 可执行，哪些可写？
- `PT_INTERP` 指定了哪个动态加载器？
- 直接声明了哪些 `DT_NEEDED`？
- 是否有 `.dynsym` / `.symtab`？它们用途为什么不同？
- `.rela.dyn` / `.rela.plt` 是否存在？
- `.text`、`.data`、`.bss` 与实际内存映射有怎样的关系？

这个实验就是“你能不能从源码层走进系统层”的分水岭。

---

# 17. 构建报错的分层排查手册

现实工程出现问题时，先不要狂搜完整错误句子。先把它归类到**预处理 → 编译 → 汇编 → 链接 → 装载 → 执行**的哪一层。

## 17.1 预处理/头文件问题

| 典型信息 | 第一反应 | 常用检查 |
|---|---|---|
| `fatal error: xxx.h: No such file or directory` | 找不到头文件 | `-I`、`#include` 路径、文件大小写、CMake include dirs |
| 宏展开导致古怪语法错误 | 宏污染或条件编译错误 | `g++ -E` 查看预处理结果 |
| 改头文件但构建没更新 | 依赖追踪配置出问题 | `-MMD`、Ninja/Make 规则、CMake 重新配置 |

## 17.2 编译语义问题

| 典型信息 | 第一反应 | 常用检查 |
|---|---|---|
| `no matching function for call` | 形参或重载匹配失败 | 源码声明、类型、命名空间 |
| `undefined identifier` / `not declared` | 当前 TU 看不到名字 | `#include`、声明作用域、宏条件 |
| `requires ... C++20` | 语言标准不足 | `-std=c++20`、编译器版本 |
| `file format not recognized`（若发生在汇编/链接） | 输入不是该工具预期格式 | `file`、目标架构、文件是否损坏 |

## 17.3 链接问题：最需要掌握

| 报错或现象 | 常见根因 | 优先动作 |
|---|---|---|
| `undefined reference to foo` | 没有提供定义，或选错库/顺序/ABI | `nm -C *.o`、检查链接命令与库位置 |
| `multiple definition of foo` | 同一外部符号重复定义 | 检查头文件中的普通函数定义、重复编译源文件、C 的 `-fno-common` |
| `undefined reference to vtable for X` | 虚函数/析构定义缺失或 ABI/符号隐藏 | 检查虚函数声明和实现、`nm -C` |
| `undefined reference to std::...` | C++ 运行库没正确链接、ABI 不匹配 | 最终用 `g++` 链接，检查 libstdc++/符号版本 |
| `cannot find -lfoo` | 构建期搜索路径或库名称有误 | `-L`、开发包安装、库架构 |
| `relocation ... can not be used when making a shared object` | 非 PIC 对象用于不允许的共享库链接 | 对相关对象/静态库启用 PIC |
| `relocation truncated to fit` | 符号地址/位移不符合重定位编码范围 | 架构/内存模型/链接脚本/代码生成方式 |
| `DSO missing from command line` | 跨依赖关系未被正确显式链接 | 添加直接依赖，检查 `--as-needed` 等 |
| `undefined reference to __asan_...` | 插桩后未链接对应运行时 | 编译和链接阶段均传 sanitizer 选项 |
| `undefined reference to __gxx_personality_v0` | C++ 异常支持运行库/驱动方式异常 | 用 `g++` 完成 C++ 链接，检查 runtime |

## 17.4 装载/运行时动态链接问题

| 报错或现象 | 常见根因 | 优先动作 |
|---|---|---|
| `error while loading shared libraries` | 动态加载器找不到库 | `readelf -dW`、`ld.so` 搜索规则、RUNPATH |
| `undefined symbol: foo` | 装载/绑定期符号缺失或 ABI 不合 | `readelf -sW`、`nm -D`、版本、可见性 |
| `version GLIBCXX_x.y.z not found` | 运行环境的 libstdc++ 符号版本不足 | ABI 与部署环境匹配，不盲目复制系统库 |
| `version GLIBC_x.y not found` | 构建时要求比目标系统更新的 glibc | 选择兼容部署基线或适合的工具链/容器 |
| `Illegal instruction` | CPU 不支持编译时启用的 ISA | 检查 `-march`、CPU 功能基线 |
| `Segmentation fault` | 运行时非法内存访问等 | `gdb`、ASan、Core dump；并非先怪 CMake |
| 运行输出异常而没有链接报错 | ODR、ABI、UB、数据竞争等 | Sanitisers、调试符号、头文件一致性与测试 |

## 17.5 五步排查模板

**第一步：找到最早的关键报错，而不是最后一行 `make: *** Error 1`。**

**第二步：确定谁报的错**：预处理器、编译器、汇编器、链接器、动态加载器，还是程序本身？

**第三步：如果是链接错误，先看实际链接命令**：CMake/Ninja 可用 verbose 构建，例如：

```bash
cmake --build build --verbose
```

关注输入的 `.o`、`.a`、`.so`、顺序、`-L/-l`、是否用 `g++`、是否有 PIC/PIE/LTO/sanitizer 选项。

**第四步：用二进制工具核实**：

```bash
nm -C some.o
nm -D -C libfoo.so
readelf -rW some.o
readelf -dW app
```

**第五步：只改一层，再重试**。不要同时换编译器、换 CMake 生成器、删一半代码、修改五个库版本。否则即使错误消失，也无法证明你真正找到了原因。

---

# 18. 工具与命令速查表

## 18.1 我最希望你会用的 20 个命令

| 目的 | 命令 |
|---|---|
| 查看 GCC 版本 | `g++ --version` |
| 驱动展开实际工作 | `g++ -v ...` / `g++ -### ...` |
| 预处理 | `g++ -E a.cpp -o a.ii` |
| 只产出汇编 | `g++ -S a.cpp -o a.s` |
| 只产出对象文件 | `g++ -c a.cpp -o a.o` |
| 编译并链接 | `g++ a.cpp b.cpp -o app` |
| 指定库搜索目录与库名 | `g++ a.o -Ldir -lfoo -o app` |
| 查看文件类型 | `file a.o` / `file app` |
| ELF 文件头 | `readelf -hW app` |
| ELF 节头表 | `readelf -SW app` |
| ELF 程序头表 | `readelf -lW app` |
| ELF 符号 | `readelf -sW app` |
| ELF 重定位 | `readelf -rW a.o` |
| 动态 ELF 依赖 | `readelf -dW app` |
| 查看符号及 C++ demangle | `nm -C a.o` |
| 查看动态符号 | `nm -D -C libfoo.so` |
| 反汇编及重定位 | `objdump -drC a.o` |
| 反汇编最终程序 | `objdump -dC app` |
| 生成静态库 | `ar rcs libfoo.a a.o b.o` |
| 看归档成员 | `ar t libfoo.a` |
| 动态依赖解析（可信文件） | `ldd ./app` |
| 链接 map | `g++ a.o -Wl,-Map=app.map -o app` |
| 构建时显示真实编译/链接命令 | `cmake --build build --verbose` |
| 查看进程映射 | `cat /proc/<pid>/maps` |
| 查看 C++ mangled 名字 | `c++filt _Z5parsei` |

## 18.2 重要参数速记（看问题时按阶段查）

```text
输入/预处理：-x  -I  -iquote  -isystem  -D  -U  -include  -E  -MMD  -MF  -MP
语言/检查：  -std=  -Wall  -Wextra  -Wpedantic  -Werror  -Wconversion  -Wshadow
产物/诊断：  -c  -S  -o  -g  -ggdb  -v  -###  -save-temps
优化/性能：  -O0  -Og  -O2  -O3  -Os  -flto  -march  -mtune  -fno-omit-frame-pointer
安全/检测：  -fsanitize=address  -fsanitize=undefined  -fsanitize=thread
             -fstack-protector-strong  -D_FORTIFY_SOURCE=2  -fvisibility=hidden
库/链接：    -L  -l  -pthread  -static  -shared  -fPIC  -fPIE  -pie  -no-pie
             -rdynamic  -Wl,...  -Xlinker  -fuse-ld=...
GNU ld：     --wrap=...  --start-group  --end-group  --whole-archive
             --gc-sections  -Map=...  -z now  -z relro
```

## 18.3 一分钟判断错误类型

```text
看到了 `fatal error: x.h not found`
   → 头文件搜索/预处理；查 -I / target_include_directories

看到了 `no matching function`
   → 编译器的 C++ 语义/类型检查；查源码声明

看到了 `undefined reference`
   → 链接器找不到定义；查 nm、输入库、符号名和顺序

看到了 `multiple definition`
   → 重复外部定义；查 ODR/头文件/重复编译/COMMON

看到了 `shared object file: cannot open`
   → 运行时装载找不到共享库；查 RUNPATH / ld.so

看到了 `undefined symbol`（运行时加载失败）
   → 动态符号或 ABI/版本不匹配；查 dynsym、SONAME、符号版本

程序已经进入 main 然后崩溃
   → 首先研究运行时错误：GDB/ASan/UBSan/TSan；不是先乱改 -l
```

---

# 19. CSAPP 49 页课件的阅读索引和时代差异

你上传的文件是 **CMU 15-213《Linking》，2015-10-13，49 页**。我按你提供的这一份原课件来列目录，不把后来版本的 CSAPP 讲义混进来。

## 19.1 与本文章节的精确对应

| PDF 页码 | 原课件讲什么 | 你应该回本文看什么 |
|---|---|---|
| 1～4 | Linking 主题、main.c/sum.c 示例、gcc driver 的编译链接流程 | 第 0～2 章 |
| 5～6 | 为什么需要 linker：模块化、分别编译、空间效率 | 第 0、3、9 章 |
| 7～8 | 两大链接任务：符号解析、重定位 | 第 7～8 章 |
| 9～10 | Relocatable、Executable、Shared objects；ELF | 第 1、6 章 |
| 11～12 | ELF 文件的节、符号表、重定位节、debug 信息 | 第 6 章 |
| 13～15 | 全局、外部、局部符号；局部静态变量 | 第 7 章 |
| 16～19 | strong/weak、linker puzzles、全局变量设计建议 | 第 7 章的 `-fno-common` 现代补充 |
| 20 | 把多个 `.o` 的节布局合并到输出 | 第 8 章 |
| 21～22 | 具体 `objdump` 重定位条目、`sum-0x4`、PC-relative call | **第 8.3～8.6 节和实验 4** |
| 23 | ELF 装载到进程内存的示意 | 第 12 章 |
| 24～27 | 为什么需要库、`.a` 的创建、`ar` | 第 9 章 |
| 28～29 | `libvector.a` 的按需抽取与链接流程 | 第 9 章与实验 7～8 |
| 30 | 归档搜索顺序、未解析引用集合 | **第 9.3～9.4 节** |
| 31～33 | 共享库的必要性、装载期动态链接 | 第 10～11 章 |
| 34～35 | `dlopen` / `dlsym` 的运行期动态加载 | 第 10.9 节与实验 11 |
| 36～37 | Linking 总结，过渡到 Interposition | 第 10、14 章 |
| 38～40 | 拦截技术及安全/调试/监测应用 | 第 14 章 |
| 41～43 | malloc/free 编译期宏拦截 | 第 14.1 节 |
| 44～45 | GNU `--wrap` 链接期拦截 | 第 14.2 节与实验 13 |
| 46～48 | `LD_PRELOAD` / `RTLD_NEXT` 动态期拦截 | 第 14.3～14.5 节与实验 13 |
| 49 | 三种 Interposition 对比 | 第 14.6 节 |

## 19.2 2015 年课件和现代工具链之间需要校正的点

**差异一：C 的 tentative definition 与 COMMON。**

课件 16～18 页把未初始化全局变量视作 weak/COMMON 处理的故事，反映旧 GCC 默认 `-fcommon` 的常见做法。**GCC 10 起默认 `-fno-common`**。今天同一例子通常会变成 multiple definition。保留课件机制是为了理解历史 bug，不是建议你恢复默认容忍行为。

**差异二：RELA 与 REL 的名字。**

课件 ELF 示意出现 `.rel.text`、`.rel.data`。现代 x86-64 GNU 工具链常见 `.rela.text` 等 RELA 节，记录中有显式 addend。理论框架一致，但具体节名/记录格式要看你实际的 ABI。

**差异三：可执行文件的 ELF 类型。**

课件容易让人形成“可执行 = ET_EXEC，动态库 = ET_DYN”的二分印象。现代 Linux 发行版的 PIE 可执行程序常是 ET_DYN；不能只按 ELF 类型名字判断“能不能运行”。

**差异四：固定地址图和 ASLR。**

课件第 22～23 页为了演示使用可直观理解的固定地址以及简化的内存布局。现代 PIE/ASLR 环境下，绝对虚拟地址常不是运行前就完全固定的，实际 VMA 也比教学图更复杂。

**差异五：`.bss` 与零初始化静态数据。**

课件第 15 页对 `static int x=0` 的 `.data` 文字不够严谨；现代 GCC 常把这类零初始化静态对象放在 `.bss`。本质应理解为“静态存储期对象不等于普通栈变量”，具体节放置以对象文件为准。

**差异六：`-ldl` 与现代 glibc。**

课件 `dlopen` 示例的动态装载接口逻辑仍适用，但 glibc 2.34 以后某些旧分立库已并入 libc，因此是否必须显式 `-ldl` 与版本/平台有关。课程里可以保留旧链接形式理解逻辑。

**差异七：函数调用的 relocation type。**

第 21 页的示例使用某种 `R_X86_64_PC32` 说明 PC-relative。现代汇编器可能显示 `R_X86_64_PLT32` 来表达函数调用相关重定位；不要期待工具输出每个字节都与课件完全一致。

**差异八：旧例子不等于生产可复制方案。**

课件的 `malloc/free` 拦截主要为展示三种 interposition 机制，未完整讨论现代分配器所需的重入防护、初始化安全性、性能影响等。请用教学目的理解，真实系统开发需要专门的成熟实现与测试。

---

# 20. 阅读 CMake 前的最终检查

如果你能独立回答下面的问题，就基本具备深入读现代 CMake 的前置知识了。第 21～22 章是选修补遗，建议在做完这些检查后按需阅读。

**构建过程层**

1. 为什么 `g++ a.cpp b.cpp -o app` 内部不止发生一次编译？
2. 为什么 `-c` 后可能出现 `U foo`，却依然算“编译成功”？
3. `-E`、`-S`、`-c`、`-o` 分别控制什么？
4. 为什么 `gcc` 能编译 `.cpp`，却可能不能正确完成普通 C++ 程序的默认链接？
5. 头文件改动和 `.cpp` 改动为何触发不同的增量构建范围？

**符号和 ELF 层**

6. `.o` 为什么不是一个可以直接 `./main.o` 运行的正常程序？
7. `.text`、`.data`、`.bss`、`.symtab`、`.rela.text` 各自承担什么职责？
8. Section Header 和 Program Header 的区别是什么？
9. 本地 C++ 自动变量为何不一定是 linker symbol？
10. `U`、`T`、`D`、`B`、`W`、`C` 大致代表什么？
11. 为什么 `int x;` 在 2015 年和现代 GCC 下可能得到不同的链接结果？

**重定位层**

12. 为什么 `main.o` 里已经有 `call` 指令，却仍然需要 relocation entry？
13. 什么叫 `S + A - P`？
14. x86-64 的 `call rel32` 为什么以**下一条指令**为参考？
15. 课件中的 `sum-0x4` 为什么出现 `-4`？
16. ELF 链接成功后为什么仍然可能保留 `.rela.dyn`？

**库与运行层**

17. `.a` 是怎么制造出来的？链接器为何通常按需抽取 `.o` 成员？
18. `g++ main.o -lfoo` 和 `g++ -lfoo main.o` 为什么对静态库可能不同？
19. 为什么链接成功的程序仍然可能报告找不到 `libfoo.so`？
20. `-I`、`-L`、`-l`、`LD_LIBRARY_PATH`、`RUNPATH` 是五种不同职责吗？
21. `-fPIC` 与 `-shared` 有什么区别？
22. 为什么 PIE 可执行文件常标为 ET_DYN？
23. GOT、PLT、动态装载器分别要解决什么问题？
24. `execve` 以后 CPU 为什么通常不是立刻进入你写的 `main()`？
25. 为什么“文件大小”“虚拟地址空间占用”“RSS”不是同一个指标？

**C++ 与 CMake 层**

26. C++ overload 为什么让链接符号名比 C 更复杂？
27. `extern "C"` 解决了什么？它为什么不能解决所有 ABI 问题？
28. 为什么头文件中的普通非 inline 函数定义容易导致重复定义？
29. 为什么 `target_include_directories()` 和 `target_link_libraries()` 不能合并理解？
30. 为什么一个静态库 Target 的 `PRIVATE` 链接依赖，仍可能影响最终 executable 的链接命令？
31. 为什么 `target_compile_options()` 和 `target_link_options()` 应该分开？
32. CMake 为什么把编译器、链接器、Target、生成器区分开？

**最重要的三个实践要求**：

- 能看 `nm -C main.o`，解释“引用但未定义”的符号需要谁来提供。
- 能看 `objdump -drC main.o`，指出一条实际的重定位，并解释 `call` 的相对位移。
- 能手写一个静态库、一个共享库，并分别解决“链接时找不到实现”和“运行时找不到 `.so`”这两类本质不同的问题。

如果这三件事能亲手做到，CMake 的 `target`、`link`、`include directories`、`PUBLIC/PRIVATE/INTERFACE` 就不再是孤立的语法。

---

# 21. 选修进阶：真正的大型系统工程还会遇到什么？

前 20 章已经足以建立 CMake 所需的主干知识。本章把一些你可能在 Redis/Nginx 构建、操作系统代码、库部署、性能工具、交叉编译中遇到，但普通 CMake 入门教程常常略过的机制补齐。它们**不要求立刻会写**，但希望你看到这些名字时不再觉得“又来了一套完全无关的东西”。

## 21.1 链接器脚本（linker script）与默认链接布局

前面我们把链接器理解成“合并/布局 section 并修补引用”。问题是：**它凭什么决定 `.text` 放哪里、如何对齐、生成哪些 PT_LOAD、入口点叫什么？**

一部分来自 ELF ABI、输入对象信息和链接器默认策略；另一部分可以由 **linker script** 明确控制。

GNU ld 支持类似：

```bash
gcc ... -Wl,-T,custom.ld -o app
```

脚本中可能涉及：

```ld
/* 仅示意核心语法，不是可以原样用于 glibc 程序的完整脚本 */
ENTRY(_start)
SECTIONS {
    . = 0x10000;
    .text : { *(.text*) }
    .rodata : { *(.rodata*) }
    .data : { *(.data*) }
    .bss : { *(.bss*) *(COMMON) }
}
```

这里 `. = 0x10000` 指定链接位置计数器相关的布局值，`*(.text*)` 从输入文件收集相应 sections。**它没有替你创建 glibc 启动环境，也没有自动安排全部 PT_LOAD/动态链接要求**；它适合用来理解内核、裸机、嵌入式等工程的布局需求，不要直接套到普通用户态程序。

你未来阅读内核、引导程序或者 C++ 底层运行库项目时会遇到链接脚本；做普通 HTTP Server 通常不需要自己写。

## 21.2 为什么直接用 ld 链接一个 C++ main.o 容易失败？

你可能想：既然已经学会了链接器，为什么不这样？

```bash
g++ -c main.cpp -o main.o
ld main.o -o app
```

这条命令对普通 C++ 程序通常会报缺少 `_start`、运行库符号等问题。**原因不是 ld 不会链接，而是普通 `g++` 驱动帮你组织了大量必要输入**：启动对象（CRT）、C++ runtime、libstdc++、libgcc、libc、平台参数、可能的 PIE/动态加载器配置。

GNU 工具链提供若干用于特殊链接场景的选项：

- `-nostartfiles`：不使用标准启动文件，但仍可能链接默认标准库。
- `-nodefaultlibs`：省略常规默认库，但标准启动文件仍可能存在。
- `-nostdlib`：不使用标准启动文件和标准库，适合非常明确地管理运行环境。
- `-nostdlib++`：省略默认 C++ 标准库支持（适用 GCC 版本）。
- `-ffreestanding`：面向不假定完整托管 C 运行时的代码，常用于内核/裸机，和链接选项不是同一个概念。

如果你读操作系统内核源码，这些选项将变得重要；如果你只是写常规服务端程序，**尽量交给 `g++` 驱动提供成熟的启动/运行时链接配置**。

## 21.3 Partial linking：`ld -r` 与复合对象

你知道：

```text
a.o + b.o → app
```

但还存在一种中间组合：

```text
a.o + b.o → combined.o → app
```

在 GNU 链接器中：

```bash
ld -r a.o b.o -o combined.o
```

`-r` 请求输出**仍可重定位**的对象，允许之后继续参加完整链接。此时可以合并 sections 与部分符号关系，但未必已经完成生成可执行文件所需的所有解析。

为何需要？某些内核、嵌入式、特殊模块或大型构建组织可能希望分阶段聚合可重定位对象。这个操作与 `ar rcs libcombined.a a.o b.o` 有本质区别：**`ar` 是打包原对象成员，`ld -r` 是产生经过部分链接的单个复合对象**。

## 21.4 Thin archive 与普通 archive

`ar` 还支持 *thin archives*，大致是归档索引中保留对外部对象文件的引用，而不是把每个对象内容完整复制到归档里。

例如部分 GNU 环境中：

```bash
ar rcT libthin.a a.o b.o
```

它可以减少大型构建中的文件复制和磁盘占用，但要求对象成员文件路径在后续使用时仍有效。**普通 `.a` 更适合作为独立交付的文件；thin archive 常适合受控构建树内部使用**。

## 21.5 `--as-needed`：我写了 `-lfoo`，为什么最终没有 `DT_NEEDED`？

有些链接器/发行版会启用类似 `--as-needed` 的行为。如果处理某个动态库时，没有发现需要由它提供的相关未解析引用，链接器可能不在最终 ELF 的 `DT_NEEDED` 中记录它。

于是：

```bash
g++ main.o -lfoo -o app
readelf -dW app | grep NEEDED
```

并不必然包含 `libfoo.so`。

GNU ld 可通过：

```bash
-Wl,--as-needed
-Wl,--no-as-needed
```

控制相应策略（具体规则还与链接顺序和链接器版本有关）。

这是一个高级但非常重要的提醒：**链接命令里出现某个库名，不必然意味着最终运行时依赖表里有它；编译时使用某个头文件，也不必然意味着最后要加载对应库。**

## 21.6 动态符号版本：为什么会提示 `GLIBCXX_... not found`？

ELF 动态链接可包含版本化符号信息。普通“函数名匹配”并不总够，还可能要求**符号版本**匹配。

```bash
readelf -VW app
readelf -VW libfoo.so
objdump -T libfoo.so | head
```

你可能看到 `GLIBC_2.x`、`GLIBCXX_3.4.x` 等版本标签。它们是相应库 ABI 管理机制的一部分，而不是源代码中的 `#define` 常量。

如果你在较新的 Linux 发行版编译程序，再拿到旧服务器运行，出现 `GLIBC_2.xx not found`，不能简单创建一个同名 `.so` 的符号链接解决：**链接器/加载器需要的是与二进制 ABI 兼容的实际符号实现及相应版本**，不只是文件名长得像。

可行的工程策略包括：选择合适的部署基线、在兼容的 sysroot/container 环境里构建、管理编译器和标准库 ABI 等。

## 21.7 TLS：线程局部存储不是静态全局、也不是普通栈变量

```cpp
thread_local int requests_seen = 0;
```

这类变量具有**线程存储期**，每个线程有自己的实体。它并不是简单放在进程唯一的 `.data/.bss` 里让所有线程共享，也不等同于普通自动局部变量。

ELF/ABI 通常有与 TLS 有关的段、节、符号及重定位模式；例如 `.tdata`、`.tbss`，以及架构相关的 TLS 地址计算机制。不同 TLS model（local-exec、initial-exec、global-dynamic、local-dynamic 等）在效率、动态库装载约束上存在区别。

对服务器/低延时系统，TLS 可以降低某些共享数据同步需求，但线程创建/模块装载、内存占用、缓存布局、错误的生命周期假设都值得关注。**现在先知道 TLS 是独立的存储类别，之后再研究 `thread_local` 性能与 ABI。**

## 21.8 ELF IFUNC：一个外部符号有时会在运行时选择具体实现

在部分 GNU ELF 平台中，`STT_GNU_IFUNC` 表示一种由 resolver 决定实际函数实现的符号机制。它常用于运行时选择针对不同 CPU 特性的实现，例如某些 libc 中的字符串/内存操作函数。

从使用者角度：同一个公开 API 在不同 CPU 上可能绑定到不同的优化实现。它体现了**符号解析不一定永远是简单地“函数名→一个写死的 `.text` 地址”**。此机制牵涉加载器、重定位、安全和早期初始化，普通应用无需自行使用。

## 21.9 Copy relocation：共享库的全局变量访问为什么更复杂？

某些 ELF 动态链接情形下，为了兼容主程序对共享库中外部数据对象的访问，可能出现 COPY relocation（例如某些非 PIE 或 ABI 特定场景）。这会产生“符号看上去属于 DSO，但主程序运行时有复制出的存储位置”一类复杂语义。

你不需要现在去记 `R_X86_64_COPY` 的每个实现细节，但应该记住：**跨 DSO 公开可写全局变量会加深 ABI 与重定位的复杂度**。相比暴露一个全局变量，公开稳定函数 API 通常更容易控制二进制兼容性。

## 21.10 Symbol interposition 和 `-Bsymbolic`

默认 ELF 动态符号可见性和符号搜索规则允许某些跨库符号被抢占（interpose）。但并不是所有库内部调用都必须经过外部可抢占地址解析。

链接选项如：

```bash
-Wl,-Bsymbolic
-Wl,-Bsymbolic-functions
```

可以影响共享库内部对某些全局符号的绑定行为，从而减少或改变可抢占语义。这可能有优化价值，也可能破坏依赖 interposition 的程序设计。`-fvisibility=hidden`、`-fno-semantic-interposition`、LTO、内联都可能和这种机制产生交互。

你看到低延时项目讨论符号可见性时，不应仅把它理解成“隐藏名字让别人看不到”。它同时关乎**ABI 设计、动态符号表、链接器优化空间和跨 DSO 可抢占性**。

## 21.11 编译器优化与未定义行为：为什么“能跑”不能证明代码正确？

编译器的很多优化依赖语言规则。例如：

- 有符号整数溢出在普通 C++ 语义中通常是 UB，编译器可能假定程序不会发生它。
- 违反 strict aliasing 规则可能让优化后的内存访问与你直觉不同。
- 数据竞争在 C++ 内存模型中可能导致 UB。
- 越界访问、对象生命周期错误可能让看似正常的测试在 `-O2` 时突然出错。

相关选项可能包括 `-fno-strict-aliasing`、`-fwrapv` 等；它们能改变编译器对特定行为的处理假设，但**不能把任意错误 C++ 程序变正确**。系统编程/低延时领域尤其应该掌握语言内存模型、对齐、对象生命周期以及 Sanitizer 的局限。

**工程建议**：优先修正源代码 UB；用性能基准评估优化；避免为了让测试通过就随手关掉大量优化行为。

## 21.12 `--sysroot`、交叉编译、目标架构与工具链

假设你在 x86-64 开发机上构建运行于 AArch64 Linux 的程序。仅使用：

```bash
g++ -march=native ...
```

显然不够。`-march` 控制目标指令集/CPU 特性，但**不会凭空把一个 x86-64 GCC 变成 AArch64 交叉编译器**。

通常需要：

- 目标架构的交叉编译器，例如 `aarch64-linux-gnu-g++`。
- 目标平台的头文件、libc、libstdc++ 与其他库。
- 适当的 sysroot/工具链文件。
- 目标架构适用的编译、链接、ABI 选项。

例如：

```bash
aarch64-linux-gnu-g++ --sysroot=/path/to/sysroot \
    -c main.cpp -o main_arm.o
```

这里 `/path/to/sysroot` 必须真的是合适的目标系统根目录，不能当作随便指定的输出文件夹。CMake 中通常配合 `CMAKE_TOOLCHAIN_FILE`、编译器配置和 find-root 规则。

**这些都是构建系统范围的“工程问题”，不是链接器可以通过猜测独立解决的问题。**

---

# 22. 把最后那些最容易绕晕的问题一次问透

### Q1：`#include` 最终不是把头文件文本展开了吗？为什么头文件不负责链接？

因为**声明可以让编译器知道接口，不代表当前 TU 有实现**。如果头文件只有 `int f();`，编译器可以生成调用 `f` 的指令与未解析符号；如果链接时没有任何输入提供符合 ABI 的 `f` 定义，才报链接错误。

### Q2：那我直接 `#include "math.cpp"`，是不是就不需要链接了？

它确实会把 `.cpp` 的内容当作当前翻译单元的一部分，某些极小例子可能能编译。但当 `math.cpp` 又被独立编译时，会产生重复定义；编译依赖与模块封装也会被破坏。**普通项目应包含声明头文件，独立编译实现文件，再链接。** 不要把包含 `.cpp` 作为长期工程习惯。

### Q3：CMake 创建了一个静态库 Target，最终可执行文件里会“嵌入一个 `.a` 文件”吗？

通常不是原样嵌入。`.a` 是一组可重定位成员对象的归档，链接器按需抽取成员，并把相应代码/数据 sections 参与最终布局、符号解析与重定位。最终程序不一定含有一个可原样提取出来的 `libfoo.a` 文件。

### Q4：`main.o` 里为什么会引用一个“没有定义”的函数，却不算非法？

因为独立编译设计就是允许跨 TU 通过声明进行调用。**`.o` 的工作契约是“我已经产出部分机器码，并把待解析关系写给链接器”**，不是“我自身已经能运行”。只要最后链接能找到定义，流程就是正常的。

### Q5：符号是不是只有函数名？

不止。它还可能指向全局/静态存储期对象、TLS、section、特殊链接实体等。链接器的符号表也不等于 C++ 语言里所有变量/类型/作用域的完整镜像。

### Q6：链接器是否会检查两个 TU 中相同名字的数据类型完全一致？

通常不会执行完整 C/C++ 语义类型检查。它主要处理符号名、绑定、可见性、重定位与 ABI 信息。有些跨 TU 类型不匹配或 ODR 违规不会被立即报错，因此**源码级接口一致性和构建系统中的统一头文件非常重要**。

### Q7：两个进程怎么能拥有相同的虚拟地址？

因为虚拟地址属于**各自独立的进程地址空间**。一个进程的 `0x400000` 不天然就是另一个进程物理上相同的内存；页表把每个进程的虚拟地址映射到适当物理页。反过来，两个进程也可以把同一共享库的只读代码页映射到各自不同的虚拟地址，从而共享物理页而不共享同一虚拟地址数值。

### Q8：`.text` 在文件中是只读/可执行的，为什么链接器还能修改指令？

**链接器工作在构建产物的文件字节上**，可以按需要重写输出的代码字段；“运行时映射为 R-X、不能随便写”是加载后的内存权限问题。别把**构建时写二进制文件**和**运行中的进程写只读代码页**混为一谈。

### Q9：为什么 `-g` 不会显著改变程序的业务功能，却能让 GDB 调试？

因为调试信息主要提供“机器码如何对应源码/变量”的元数据，并非为了改变程序业务算法。但不同优化设置与构建布局仍可能影响机器码、性能和未定义行为表现；不应把调试版本与发布版本当成逐字节完全相同的产物。

### Q10：为什么我的程序没有显式使用 malloc，却仍依赖 libc？

你使用的启动运行时、I/O、异常处理、系统函数包装或编译器支持代码都可能间接使用运行库。一个 C/C++ 程序的完整依赖不限于你在 `main()` 中亲手写出的几个函数名。CMake/链接驱动会帮你组织一部分默认运行库。

### Q11：`-static` 是不是就完全没有动态行为、甚至不依赖操作系统？

不是。静态程序仍需要系统调用和内核资源。某些 C 运行库功能还可能涉及运行时配置、插件、NSS 等复杂机制。`-static` 控制的是链接库策略，不是把操作系统整体复制到你的 ELF 文件。

### Q12：同一套函数编成 `.a` 和 `.so`，源码调用写法一定不同吗？

通常可以保持相同的头文件接口和 C++ 调用写法，区别更多在**链接命令、地址解析、动态库查找、ABI 和部署**。如果导出符号的 ABI 不兼容，即使接口名字看起来没变，也可能无法安全替换。

### Q13：`g++` 的 `-DXXX=1` 和 `cmake -DXXX=1` 是同一个操作吗？

不是。`g++ -DXXX=1` 给**预处理器**定义宏，影响 C/C++ 源码编译；`cmake -DXXX=1` 给**CMake 配置/缓存变量**赋值，影响它如何生成构建系统。只有当 `CMakeLists.txt` 明确将该配置映射为 `target_compile_definitions(... XXX=1)` 等操作时，两者才产生关联。

### Q14：`-L` 可以设置运行时库路径吗？那 CMake 的 `CMAKE_PREFIX_PATH` 呢？

通常不可以。`-L` 用于链接器搜索库；CMake 的 `CMAKE_PREFIX_PATH` 用于帮助 CMake 的包/文件等查找逻辑；动态运行时库搜索由系统加载器的机制（RUNPATH/RPATH、`LD_LIBRARY_PATH`、默认目录等）控制。**三个不同主体的“找文件”功能，不能因为名字里有 path 就混成一个。**

### Q15：`PUBLIC` 链接依赖是不是会把整个依赖库复制到任何使用者里？

不是。`PUBLIC` 主要是 Target 使用要求和链接接口的传递语义，不等于实际把全部库字节拷进每个使用者；最终是静态归档按需抽取、动态依赖记录还是其他形式，由库类型、链接器和目标配置决定。

### Q16：构建速度、运行速度和启动速度是一个指标吗？

完全不同。`-flto` 可能提高运行性能却增加构建时间；共享库按需装载可能节省重复存储，但引入装载/重定位成本；`-O3` 可能改善某些热循环，却因为代码膨胀伤害真实负载；PCH/Unity build 更偏构建速度优化。**做系统编程时明确你在优化“哪一个阶段”的成本。**

---


## 参考资料与进一步阅读

**你提供的原始课件（核心阅读依据）**：

- CMU 15-213 / CSAPP，*Linking*，2015-10-13，共 49 页，文件名 `13-linking.pdf`。正文第 19 章已逐组标出页码，并注明现代工具链变化。

**以下为本教材补充的现代 Linux/GCC/CMake 知识，可与课件分开阅读**：

- GCC 官方文档 — Invoking GCC：<https://gcc.gnu.org/onlinedocs/gcc/Invoking-GCC.html>
- GCC 官方文档 — Overall Options：<https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html>
- GCC 官方文档 — Link Options：<https://gcc.gnu.org/onlinedocs/gcc/Link-Options.html>
- GCC 10 Release Notes（`-fno-common` 变化）：<https://gcc.gnu.org/gcc-10/changes.html>
- GNU binutils / ld 选项文档：<https://sourceware.org/binutils/docs/ld/Options.html>
- glibc 动态加载器手册 `ld.so(8)`：<https://man7.org/linux/man-pages/man8/ld.so.8.html>
- 动态库接口手册 `dlopen(3)`：<https://man7.org/linux/man-pages/man3/dlopen.3.html>
- Linux `execve(2)`：<https://man7.org/linux/man-pages/man2/execve.2.html>
- CMake `target_link_libraries` 官方参考：<https://cmake.org/cmake/help/latest/command/target_link_libraries.html>
- CMake 官方学习教程：<https://cmake.org/cmake/help/latest/guide/tutorial/index.html>
- ELF 与 AMD64 ABI 的进一步细节：查 System V ABI / AMD64 psABI 的 ELF relocation、calling convention 章节。

> **最后的心智模型**：C++ 源码首先在翻译单元内被理解和编译，产生包含机器码、符号和“待修补地址说明”的对象文件；链接器解析跨文件名字并按规则修补地址关系；静态库是可以按需抽取的对象文件归档，动态库则让一部分符号与地址工作延续到装载和运行期；ELF 告诉链接器与加载器如何理解这些二进制结构，内核及运行时最终把它变成进程。**CMake 只是在更高一层有组织地表达和调度这一切。**
