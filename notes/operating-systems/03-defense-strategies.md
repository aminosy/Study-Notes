# 内存安全防御策略：从编译期到运行时的纵深防御

本篇整理业界用于防御内存类漏洞的主要机制：ASLR、DEP/NX、栈保护 (canary)、RELRO、CFI，以及安全编码规范与静态/动态检测工具，并说明每种机制分别防住了什么、又防不住什么。

## 0. 核心观念：防御必须叠加

没有任何单一机制能防住所有内存漏洞。业界的做法是**纵深防御 (defense in depth)**，在软件生命周期的不同阶段布防：

```text
   源码 / 编码阶段        编译阶段           链接阶段          加载阶段           运行阶段
  ┌────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐
  │ 安全编码规范    │  │ 编译器警告    │  │  RELRO        │  │  ASLR         │  │  CFI           │
  │ 代码审查        │→ │ 栈保护 canary │→ │  BIND_NOW     │→ │  NX / DEP     │→ │  sanitizer     │
  │ 静态分析        │  │ FORTIFY_SOURCE│  │  符号可见性   │  │  页权限        │  │  运行期加固     │
  │ 类型安全语言    │  │ PIE           │  │              │  │              │  │  监控 / 缓解    │
  └────────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  └────────────────┘
       预防                 预防                预防              缓解              检测 + 缓解
```

注意"预防 (prevention)"与"缓解 (mitigation)"的区别：**预防**让漏洞无法被触发（如边界检查）；**缓解**承认漏洞可能被触发，但让利用变得困难或不稳定（如 ASLR）。ASLR 不修复任何 bug，它只是把"稳定可复现的利用"变成"需要额外信息泄露的赌博"。理解这一点，就不会误以为开了 ASLR 就安全了。

## 1. ASLR (Address Space Layout Randomization)

**防的是什么**：让攻击者无法预先知道代码、库、栈、堆、mmap 区域的地址，从而无法在 payload 里写死目标地址。

**原理**：加载程序和内核在每次进程启动时，对以下区域的基址加上一个随机偏移：

| 区域 | 随机化对象 | 说明 |
| --- | --- | --- |
| 可执行文件 (PIE) | 程序映像基址 | 需要编译为 PIE (Position Independent Executable) 才生效 |
| 共享库 | 每个 `mmap` 的库加载基址 | 由内核决定 |
| 栈 | 栈顶位置 | 偏移范围通常较小 |
| 堆 | 堆起始位置 | |
| `mmap` 区域 | 每次映射的位置 | |

**关键细节**：

- **ASLR 需要 PIE 配合**。如果可执行文件不是 PIE，程序本身的代码段仍然固定在编译时的地址（例如 `0x400000`），攻击者至少知道这一块。现代发行版默认编译 PIE（GCC 的 `-pie -fPIE`，Debian/Ubuntu 与多数发行版的 GCC 默认开启 `--enable-default-pie`），但要**验证而不是假设**。
- **熵 (entropy) 决定强度**。32 位系统的随机化空间有限，暴力猜测有时可行；64 位系统位数大得多，暴力猜测基本不可行。这是一个量级差异。
- **ASLR 防不住信息泄露**。一旦攻击者通过格式化字符串、越界读等手段拿到了一个真实地址，他就能算出基址，ASLR 的随机性被彻底抵消。**这就是为什么"信息泄露漏洞"通常被视为与"任意写漏洞"同等严重**。
- **ASLR 不防住相对偏移攻击**。如果攻击目标是"与某个已知地址距离固定的东西"，随机化无效。

**检查方法**：

```bash
# 查看系统级 ASLR 设置：0=关闭, 1=保守, 2=完全随机化
cat /proc/sys/kernel/randomize_va_space

# 查看某个可执行文件是不是 PIE
file /bin/ls
# 输出含 "pie executable" 则为 PIE；"executable" 则不是

# 或者用 readelf 看符号类型（T 表示固定地址，通常是非 PIE）
readelf -h /bin/ls | grep Type
# Type: EXEC  -> 非 PIE
# Type: DYN   -> PIE / 共享对象
```

## 2. DEP / NX (Data Execution Prevention / No-eXecute)

**防的是什么**：阻止执行位于数据区域（栈、堆、数据段）中的代码。

**原理**：在页表项 (PTE) 中有一个 **NX 位 (No-eXecute bit)**（在 AMD 文档中称为 NX，Intel 称为 XD，即 Execute Disable）。操作系统把栈和堆映射为"可读写但不可执行"，把代码段映射为"可读可执行但不可写"。CPU 在取指时检查该位，若发现目标页不可执行则触发 fault。

回到上一篇的 PTE 表：`NX` 就是"该页是否允许作为指令执行"那一位。所以 DEP/NX 的根基是**页级权限**这一操作系统机制。

**它防住的是什么攻击**：经典的"往栈上注入 shellcode 然后跳到栈上执行"。有了 NX，这段注入的字节即使被成功跳转过去，取指阶段就会 fault。

**它防不住什么 —— 也就是为什么会演化出 ROP**：NX 只限制"执行哪里的代码"，不限制"跳到哪里"。攻击者于是**不注入新代码，而是复用已有的代码片段**：在一段已有的可执行代码（程序本身或 libc）中寻找以 `ret` 结尾的短指令序列，称为 **gadget**，把这些 gadget 地址按顺序摆在栈上，`ret` 一个接一个地串联执行，形成 **ROP (Return-Oriented Programming)** 链。由于执行的全是合法代码，NX 完全无效。

必须诚实地说：**ROP 是 NX 的直接后果之一**，NX 大幅提高了门槛但没有消灭代码执行。真正的补充手段是**代码指针完整性**（见第 5 节 CFI）和 ASLR（让 gadget 地址未知）——这再一次说明为什么必须叠加。

**检查方法**：

```bash
# 查看某个二进制的段权限：GNU_STACK 一行右侧有 E 表示栈可执行（危险）
readelf -l ./app | grep -A1 GNU_STACK
# 期望看到：GNU_STACK ... RW   （没有 E）

# 查看内核对 NX 的支持（nx 出现在 flags 中）
grep -o 'nx' /proc/cpuinfo | head -1
```

## 3. 栈保护 (stack canary / stack protector)

**防的是什么**：栈缓冲区溢出覆盖返回地址。

**原理**：在函数的返回地址与局部变量之间插入一个随机值（"金丝雀" canary，名字来源于矿井里用金丝雀探测毒气）。函数入口写入该值，函数返回前**检查它是否被改变**；若改变则说明发生了栈溢出，立即调用 `__stack_chk_fail()` 终止进程。

```text
        高地址
    +---------------------------+
    |  返回地址                  |
    +---------------------------+
    |  canary（随机值，最低字节  |  <-- 函数返回前必须与保存值比对
    |  通常为 0x00 以防字符串函数）|      不一致就 abort
    +---------------------------+
    |  局部变量 (buf)            |
    +---------------------------+
        低地址
```

**为什么 canary 的最低位常常是 `\0`**：因为典型的溢出是 `strcpy`/`strcat` 这类字符串函数造成的，它们会写入一个结尾空字节。把 canary 最低字节设为 `\0`，字符串溢出在覆盖 canary 时就会**提前停止**（或者说，即使覆盖也要先破坏这个空字节），这让检测更可靠。这是一个实现细节，不是语言规定。

**canary 的类型（GCC 的 `-fstack-protector` 系列）**：

| 选项 | 保护的函数范围 | 说明 |
| --- | --- | --- |
| `-fstack-protector` | 含数组/地址被取的函数 | 覆盖面有限 |
| `-fstack-protector-strong` | 更广：含数组、含局部地址引用、含 `alloca` 等 | 多数发行版默认 |
| `-fstack-protector-all` | 所有函数 | 性能开销最大 |
| `-fno-stack-protector` | 无 | 只应在明确知道后果时使用 |

**它防不住什么**：

- **不覆盖 canary 的溢出**：例如只覆盖远在 canary 之前的其他局部变量（改写一个局部函数指针、一个长度变量、一个 `bool` 标志位），canary 完好无损，检测不触发。**这是 canary 最重要的局限**——它保护的是"返回地址"，不是"所有数据"。
- **能读出 canary 的攻击**：借助格式化字符串或任意读泄露 canary 后，攻击者可以在 payload 中原样写回正确的 canary 值，检测同样不触发。
- **堆溢出与 UAF**：canary 只在栈上，对堆完全无效。
- **覆盖异常处理结构 / 其他控制数据**：如果编译器把其他控制信息放在受保护区域之外，同样绕过。

**检查方法**：

```bash
# 看二进制里有没有 canary 相关符号
nm ./app | grep -i stack_chk
# 期望看到 U __stack_chk_fail（说明开启了栈保护）
```

## 4. RELRO (Relocation Read-Only)

**防的是什么**：改写 GOT (Global Offset Table) 等重定位表，从而劫持函数调用。

**背景**：动态链接的程序里，对共享库函数的调用通常经过 PLT (Procedure Linkage Table) → GOT。GOT 里保存的是函数的真实地址；**延迟绑定 (lazy binding)** 意味着第一次调用某个函数时才由动态链接器解析地址并写入 GOT。因此 GOT 在运行期必须是可写的——这就是攻击面：一个任意写漏洞如果能改写 GOT 项，之后每次调用该函数都会跳到攻击者指定的地址。

**RELRO 的两个级别**：

| 级别 | 编译/链接选项 | 效果 |
| --- | --- | --- |
| Partial RELRO | `-Wl,-z,relro` | 把 `.got` 之外的段（如 `.init_array`、`.fini_array`、`.dynamic`）在重定位完成后标记为只读；`.got.plt` 仍可写 |
| Full RELRO | `-Wl,-z,relro,-z,now` | 加上 `BIND_NOW`，**启动时解析全部符号**，之后把整个 GOT 标记为只读 |

**Full RELRO 的代价**：启动时要解析所有符号，程序启动变慢（对于大量依赖动态库的大程序可能有可感知的延迟）；同时失去了延迟绑定带来的"没用到的函数不解析"的优化。

**它防不住什么**：RELRO 保护的是重定位表，不保护**函数指针变量**、C++ 虚表 (vtable)、以及栈上的返回地址。它对应的更普遍问题是"可写的控制数据"——历史上 `glibc` 的 `__malloc_hook` 之类的钩子就是很受青睐的目标，而这类钩子已在较新的 glibc 版本（2.34 起）中被移除，属于缩小可利用面的改动。**注意：具体库与版本的行为会变，理解"哪些控制数据是可写的"这个分析角度比记住某个符号更有用。** RELRO 本身是一种"缩小可写控制数据范围"的手段，而不是通用防护。

**检查方法**：

```bash
# 检查 RELRO 状态
readelf -l ./app | grep -i relro
# 期望看到：GNU_RELRO

# 更直观：checksec 工具（属于 pwntools 生态，需要单独安装）
checksec --file=./app
```

## 5. CFI (Control-Flow Integrity)

**防的是什么**：控制流劫持本身。前面的所有机制都假设"控制流不会被重定向"，CFI 则直接给控制流加上规则。

**原理**：编译期通过静态分析建立**合法控制流图 (Control Flow Graph, CFG)**，记录每个间接跳转/间接调用（`call *%rax`、虚函数调用、函数指针调用、`ret`）的合法目标集合。在运行时对每个间接控制转移做检查：目标不在合法集合内就终止。

**两个层次**：

- **前向边 (forward-edge) CFI**：约束间接调用与间接跳转的目标（例如"这个函数指针只能指向签名匹配的函数"）。
- **后向边 (backward-edge) CFI**：约束 `ret` 的返回目标。常见实现是 **shadow stack（影子栈）**：在另一块受保护的内存里维护一份返回地址副本，`ret` 时比对两份是否一致，不一致即说明返回地址被改写。**影子栈直接针对 ROP，是与 NX 形成互补的关键机制。**

**现有实现（真实存在的工程实现）**：

- **LLVM 的 `-fsanitize=cfi`**：需要 LTO (Link Time Optimization)，主要针对 C++ 虚表与前向边；属于 sanitizer 家族，可用于测试与加固。
- **Microsoft 的 Control Flow Guard (CFG)**：Windows 上的前向边 CFI，由编译器与操作系统协作实现。
- **Intel CET (Control-flow Enforcement Technology)**：硬件支持，包含 **shadow stack** 与 **indirect branch tracking** 两部分，是后向边与前向边保护的硬件方案。
- **Linux 内核的 `CONFIG_CFI_CLANG`**：用 Clang 的 KCFI 为内核提供前向边保护（较新的内核特性）。

**它防不住什么 / 现实局限**：

- **粒度问题**：CFI 的精度取决于合法目标集合有多大。如果同一签名的函数很多，集合就大，攻击者仍可能在集合内挑选一个"有用"的函数（这类攻击被称为 CFI 的 COOP 类绕过思路）。**粗粒度 CFI 只是提高了难度，不是绝对防护。**
- **兼容性与性能开销**：严格的 CFI 需要 LTO、要求所有代码用同一套规则编译，对第三方预编译库不友好；性能与二进制体积都有代价。这是它目前未被全面默认开启的原因。
- **数据流不在保护范围内**：CFI 只管"跳到哪"，不管"数据被改成了什么"。纯数据攻击（例如只改写一个表示权限的布尔值而不改控制流）完全不受 CFI 影响。

## 6. FORTIFY_SOURCE 与编译期加固

**`_FORTIFY_SOURCE`** 是 glibc 提供的一套机制：编译器在**能确定目标缓冲区大小**的情况下，把 `strcpy`、`memcpy`、`sprintf`、`printf` 等调用替换为带边界检查的 `__*_chk` 版本；检查失败时终止程序而不是继续越界。

```bash
# 推荐组合（优化级别是必要的，因为需要编译期可知的对象大小）
gcc -O2 -D_FORTIFY_SOURCE=3 -fstack-protector-strong -fPIE -pie \
    -Wl,-z,relro,-z,now -Wall -Wextra -o app app.c
```

各选项的作用回顾：`-D_FORTIFY_SOURCE=3` 运行时检查；`-fstack-protector-strong` 栈保护；`-fPIE -pie` 让 ASLR 对主程序生效；`-Wl,-z,relro,-z,now` 启用 Full RELRO；`-Wall -Wextra` 打开警告。

**局限**：FORTIFY 只在编译期能推断出对象大小时生效。如果缓冲区大小来自运行时（例如 `char *p = malloc(n)` 传给 `strcpy`），编译器无法插入检查。所以它是一道**覆盖面不确定的**防线，不能替代人工审计。

## 7. 安全编码规范

工具只能发现"能推理出来的"问题，规范与审查负责剩下的部分。以下是我在实践中认为最有效的几条：

| 规范 | 具体做法 | 针对的漏洞 |
| --- | --- | --- |
| 长度与数据分离 | 所有外部输入都带显式长度，不用隐式终止符作唯一依据 | 溢出、越界读 |
| 先验证后使用 | 长度/索引的检查必须在运算之前，并考虑回绕 | 整数溢出导致越界 |
| 单一所有权 | 一块内存只有一个明确的释放责任人；用 RAII / 智能指针 / 垃圾回收代替手工 `free` | double free、UAF |
| 释放即置空 | `free(p); p = NULL;`，并意识到这只对单个变量有效 | double free |
| 基于 fd 而非路径名 | 用 `open` + `fstat` + `openat` 代替 `access` + `open` | TOCTOU |
| 格式串常量化 | 永远写 `printf("%s", user)`，用 `-Wformat-security` 兜底 | 格式化字符串 |
| 不在信号处理函数里做复杂操作 | 只调用 async-signal-safe 函数，用 `volatile sig_atomic_t` 传递状态 | 竞态、重入破坏 |
| 优先使用内存安全语言 | Rust / Go / Java 等在语言层面消除整类问题 | 上述大部分 |
| 外部输入设业务上界 | 例如"单条消息 ≤ 1 MiB"，而不是只检查算术边界 | 溢出、DoS |
| 默认拒绝 | 权限、白名单、失败即关闭，而不是失败即放行 | 逻辑绕过 |

关于**内存安全语言**，值得单独说一句：Rust 通过所有权与借用检查在编译期消除 UAF 与 double free，并且默认不检查边界（越界会 panic 而非静默破坏）。这也是近年推动"关键组件用内存安全语言重写"这一讨论的核心动机。但要注意：**Rust 中的 `unsafe` 块会把保证交回程序员手中**，内存安全语言降低的是概率，不是把问题变成零。

## 8. 静态与动态检测工具

### 8.1 静态分析 (static analysis)

不运行程序，直接分析源码或二进制。

| 工具 | 类型 | 说明 |
| --- | --- | --- |
| 编译器警告 | 源码 | `-Wall -Wextra -Wformat=2`，成本最低、收益最高，应该作为第一道防线 |
| `clang-tidy` | 源码 | Clang 生态的 linter，支持大量检查项与自定义规则 |
| `cppcheck` | 源码 | 独立的 C/C++ 静态分析工具，易于集成到 CI |
| Coverity / Fortify | 商业 | 企业级静态分析平台 |
| `shellcheck` | Shell | Shell 脚本的静态检查（与本仓库 Linux 板块呼应） |
| 二进制分析 | 二进制 | `checksec` 看加固状态；反汇编/反编译工具用于审计无源码的组件 |

静态分析的**固有局限**：需要近似判断，因此必须接受两类错误——**漏报 (false negative)** 与 **误报 (false positive)**。误报太多会让开发者忽略告警，这是静态分析落地最大的现实障碍。此外，**配置/环境相关的行为静态分析看不到**（这一点是下一篇的主题）。

### 8.2 动态分析 (dynamic analysis)

在真实或受控执行中检测问题，能发现静态分析看不到的运行时行为。

**Sanitizer 家族（Compiler-based，真实存在且广泛使用）**：

```bash
# AddressSanitizer：检测堆/栈/全局越界、use-after-free、double free、内存泄漏
gcc -fsanitize=address -g -O1 -o app_asan app.c
# 运行时会打印详细的报告：出错类型、地址、分配与释放的调用栈

# LeakSanitizer 通常包含在 ASan 中（Linux 上默认开启）
# 也可以用 ASAN_OPTIONS 精细控制
ASAN_OPTIONS=detect_leaks=1:abort_on_error=1 ./app_asan

# UndefinedBehaviorSanitizer：检测有符号溢出、移位越界、类型混淆等 UB
gcc -fsanitize=undefined -g -o app_ubsan app.c

# ThreadSanitizer：检测数据竞争与死锁
gcc -fsanitize=thread -g -o app_tsan app.c

# MemorySanitizer：检测未初始化内存的读取（需要 Clang，且所有依赖需一起插桩）
clang -fsanitize=memory -g -o app_msan app.c
```

ASan 的代价值得说明：它会给程序加约 2 倍左右的 CPU 开销和显著的内存开销（因为它用 shadow memory 跟踪每一字节的可访问性），因此**通常用于测试环境而不是生产环境**。TSan 的开销更大。

**内存错误检测工具**：

| 工具 | 检测能力 | 特点 |
| --- | --- | --- |
| Valgrind (memcheck) | 未初始化读、越界读写、泄漏、double free、`free` 非法指针 | 不需要重新编译；通过动态二进制插桩实现，因此运行速度显著变慢（数量级），适合定位难复现问题 |
| AddressSanitizer | 同上有重叠，但速度快得多 | 需要重新编译；生产级项目常搭配 fuzzing 使用 |
| Electric Fence | 通过页边界捕获越界 | 概念简单，让每次越界立即变成段错误而不是静默破坏 |
| `glibc` 的 `MALLOC_CHECK_` / `mtrace` | 分配器层面的基本检查、泄漏追踪 | 内置于 glibc，设置环境变量即可用，能力有限但零成本 |

**Fuzzing（模糊测试）**：向程序投喂大量（半）随机输入并监控崩溃。现代 fuzzer 使用覆盖率反馈来引导输入变异：

- **AFL / AFL++**：覆盖率引导的 fuzzing，广泛用于 C/C++ 程序。
- **libFuzzer**：与 LLVM 集成的 in-process fuzzer，适合对单个函数做 fuzzing。
- **OSS-Fuzz**：Google 维护的持续 fuzzing 服务，为大量开源项目长期跑 fuzzer。
- **Honggfuzz**、**syzkaller**（针对内核系统调用）等。

Fuzzing 与 sanitizer 是**天然搭配**：ASan/UBSan 把"静默的内存破坏"变成"立即崩溃"，fuzzer 才有可能发现它。这也是为什么"fuzzing + ASan"被视为发现内存漏洞最有效的组合之一。

**必须诚实指出的局限**：**动态分析只能覆盖被执行到的路径**。测试用例没走到的分支里的漏洞，动态工具一无所知。所以静态与动态分析是互补关系，不是替代关系。

## 9. 一张总表：机制 → 防的是什么 → 防不住什么

| 机制 | 生效阶段 | 防住 | 防不住 |
| --- | --- | --- | --- |
| ASLR | 加载期 | 在 payload 里写死固定地址 | 信息泄露后的重算；相对偏移攻击 |
| PIE | 编译/链接期 | 使 ASLR 对主程序生效 | 单独使用无意义（必须配合 ASLR） |
| NX / DEP | 加载期（页权限） | 执行栈/堆上注入的代码 | ROP / JOP（复用已有代码） |
| 栈 canary | 编译 + 运行期 | 覆盖返回地址的栈溢出 | 不触动 canary 的溢出；泄露 canary 后绕过；堆问题 |
| RELRO (Full) | 链接 + 加载期 | 改写 GOT 劫持调用 | 函数指针变量、vtable、栈上返回地址 |
| CFI / 影子栈 | 编译 + 运行期 | 间接转移与 `ret` 目标被篡改 | 粗粒度下集合内挑选；纯数据攻击；兼容性开销 |
| FORTIFY_SOURCE | 编译 + 运行期 | 编译期可确定大小的越界调用 | 运行时决定大小的缓冲区 |
| ASan / Valgrind | 测试期 | 大量内存错误，且带完整调用栈 | 未被执行到的代码路径；性能开销不适合生产 |
| Fuzzing | 测试期 | 意外输入触发的崩溃 | 需要语义才能触发的逻辑漏洞 |
| 安全编码 + 审查 | 编码期 | 设计层面的问题（所有权、检查顺序） | 依赖人的注意力，无法穷尽 |

## 10. 我实际会怎么用这套东西

把上面的机制落到日常开发的一个可执行清单：

```bash
# 1. 调试构建：先让问题暴露出来
gcc -g -O0 -Wall -Wextra -Wformat=2 -fsanitize=address,undefined -o app_dbg app.c

# 2. 测试：ASan + fuzzing（如果程序有可 fuzz 的输入面）
#    对 libFuzzer/AFL 接入后长期运行

# 3. 发布构建：加固选项全开
gcc -O2 -D_FORTIFY_SOURCE=3 -fstack-protector-strong -fPIE -pie \
    -Wl,-z,relro,-z,now -Wall -Wextra -o app app.c

# 4. 发布前验证加固真的生效了（不要只是假定）
checksec --file=./app          # 需要安装
readelf -l ./app | grep -i relro
readelf -l ./app | grep -A1 GNU_STACK
nm ./app | grep -i stack_chk

# 5. 上线后：对关键崩溃做符号化回溯，保持 ASan/UBSan 在测试环境常开
```

## 小结

- 防御分"预防"与"缓解"两类：ASLR/NX/canary 都是缓解，它们不修复 bug，只提高利用门槛。
- ASLR 需要 PIE 才能覆盖主程序；它对信息泄露漏洞完全无效，这是最容易被忽略的短板。
- NX 阻止执行注入的代码，直接催生了 ROP（复用已有代码片段），因此必须与 ASLR、影子栈/CFI 配合。
- canary 保护的是返回地址，不是所有数据；不触动 canary 的溢出可以完全绕过它。
- RELRO 把 GOT 变成只读，缩小了"可写的控制数据"范围，但函数指针与 vtable 不在其保护内。
- CFI（尤其配合影子栈）直接约束控制流，是当前对抗 ROP 的方向；难点在粒度、兼容性与性能。
- 静态分析受限于近似判断（漏报与误报），动态分析受限于代码覆盖率；二者互补，都不能替代安全编码与代码审查。
- 最能改变安全性的往往是语言与 API 的选择（内存安全语言、基于 fd 的 API），而不是事后添加的加固选项。
- 下一篇把视角从"单个漏洞"提升到"系统级检查"：为什么配置文件与静态校验通过，不代表运行时行为符合预期。
