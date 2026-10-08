# 阅读清单：操作系统与数据库方向的经典资料

本清单收集操作系统与数据库/数据管理方向的经典书籍与公开课程，均为真实存在的资料，尽可能标注作者与版本信息。

## 怎么用这份清单

我的做法是**一个方向只精读一本主教材 + 一本练习型资料**，剩下的当参考书按需查。原因很实际：

- 操作系统与数据库这两个方向，**不同教材的组织方式差别很大**（有的重概念、有的重实现、有的重数学），同时读三本会不断在"同一个概念的不同讲法"之间切换，效率反而低；
- 但这些方向的**细节非常多**，一本教材不可能覆盖所有你需要的点，所以需要参考书。

所以下面的清单分三类：

| 标记 | 含义 |
| --- | --- |
| **主** | 建议精读一遍（含做题） |
| **参** | 当参考书，遇到具体问题时查对应章节 |
| **课** | 公开课程，可配合教材一起看 |

**一个重要的说明**：下面所有书籍的作者、书名都是真实的。**版本信息我只在确信的地方标注**；如果你要买某一本，请以出版社当前版本为准——教材改版频率比我写这份笔记的频率高得多。

## 一、操作系统

### 1.1 主教材

| 书名 | 作者 | 说明 |
| --- | --- | --- |
| **《操作系统导论》(Operating Systems: Three Easy Pieces)** | Remzi H. Arpaci-Dusseau & Andrea C. Arpaci-Dusseau | **我最推荐的第一本**。作者是威斯康星大学麦迪逊分校的教授，全书围绕三个主题组织：虚拟化 (virtualization)、并发 (concurrency)、持久性 (persistence)。最大的优点是**讲得极其清楚、几乎每个概念都配一个真实可跑的小程序**，而且正文里有大量"作业 (homework)"和模拟器（例如用 Python 写的进程调度模拟器、页表模拟器）。**这本书的电子版由作者免费公开在官网**（搜索书名即可找到），纸版由 Morgan Kaufmann 出版。 |
| **《操作系统概念》(Operating System Concepts)** | Abraham Silberschatz, Peter B. Galvin, Greg Gagne | 俗称"恐龙书"（封面是恐龙）。**覆盖面最广、最像教科书**，术语体系与大多数课程/考试最贴近。优点是全面、结构规范；缺点是**偏概念与综述，实现细节较少**，读起来不如上一本有趣。适合作为课程配套教材或当参考书。 |
| **《现代操作系统》(Modern Operating Systems)** | Andrew S. Tanenbaum | Tanenbaum 的经典教材，**视角更偏"设计与权衡"**，会讲很多"为什么这么设计"以及不同系统的对比。篇幅大、信息密度高。适合已经有一点基础之后读，能建立起"设计选择"的判断力。 |
| **《深入理解计算机系统》(Computer Systems: A Programmer's Perspective, CSAPP)** | Randal E. Bryant & David R. O'Hallaron | 严格说这是**计算机系统**教材而不是操作系统教材，但它把"程序在机器上如何运行"讲透了：数据表示、汇编、内存层次、链接、异常控制流、虚拟内存、并发。**对写代码的人来说，这本书的实用价值极高**。第 9 章（虚拟内存）与本仓库操作系统板块的笔记直接对应。**建议在学操作系统之前或同时读。** |

### 1.2 参考书与进阶

| 书名 | 作者 | 说明 |
| --- | --- | --- |
| 《UNIX 环境高级编程》(Advanced Programming in the UNIX Environment, APUE) | W. Richard Stevens & Stephen A. Rago | UNIX/Linux 系统编程的圣经。文件 I/O、进程、信号、线程、IPC 的权威参考。**不需要通读，当手册查。** |
| 《UNIX 网络编程》(UNIX Network Programming) | W. Richard Stevens | 套接字编程的经典。分卷出版（第一卷讲套接字 API，第二卷讲进程间通信）。 |
| 《Linux 内核设计与实现》(Linux Kernel Development) | Robert Love | 讲 Linux 内核的整体结构与子系统，篇幅不大，适合作为"内核长什么样"的入门。 |
| 《深入理解 Linux 内核》(Understanding the Linux Kernel) | Daniel P. Bovet & Marco Cesati | 更深入、更偏源码细节，阅读门槛高。 |
| 《性能之巅》(Systems Performance: Enterprise and the Cloud) | Brendan Gregg | **排查性能问题的必读**。讲了方法论与 Linux 上的各种观测工具（`perf`、`ftrace`、`bpftrace` 等）。与笔记里的 `top`/`vmstat`/`iostat` 部分直接呼应，但深得多。 |
| 《BPF 之巅》(BPF Performance Tools) | Brendan Gregg | 用 eBPF 做系统可观测性的实战书，工具导向。 |
| 《计算机程序的构造和解释》 | Harold Abelson & Gerald Jay Sussman | 与操作系统关系不大，但对"抽象"的理解有帮助。列在这里是因为我在读并发与状态时回去翻过它。 |

### 1.3 公开课程

| 课程 | 出处 | 说明 |
| --- | --- | --- |
| **MIT 6.S081 / 6.828: Operating System Engineering** | MIT | 基于 **xv6**（一个教学用的类 UNIX 内核）的课程，**有著名的一系列 lab**（system calls、page tables、traps、multithreading、network driver 等）。这是我见过**最能建立"操作系统到底怎么实现"这个直觉**的材料。xv6 的源码注释版（xv6 book）也由 MIT 公开。 |
| **CS 162: Operating Systems and Systems Programming** | UC Berkeley | Berkeley 的操作系统课，讲并发与系统的权衡很扎实。 |
| **Nand2Tetris (Build a Modern Computer from First Principles)** | Coursera / 官网 | 从逻辑门一直搭到操作系统与编译器。**不是操作系统课，但"亲手搭一遍"的经验对理解抽象层次极有价值。** |
| **MIT 6.824: Distributed Systems** | MIT | 分布式系统（Raft、MapReduce 等）。列在这里因为并发与一致性问题在这里被系统性地展开，是操作系统的自然延伸。 |

**关于 xv6 的建议**：如果你只想做一件事来加深对操作系统的理解，**做 xv6 的 lab**（尤其是 page tables 和 traps 那两个）。它们会强迫你真正读懂页表、陷入 (trap) 处理、用户态/内核态切换。这比读十遍教材有用。

### 1.4 内存安全与系统安全方向

操作系统板块的第 2、3 篇笔记涉及内存漏洞与防御。这个方向我推荐的材料：

| 资料 | 作者 / 出处 | 说明 |
| --- | --- | --- |
| 《深入理解计算机系统》第 3 章 | Bryant & O'Hallaron | 讲栈帧布局、缓冲区溢出、ROP 的基础。**这是从系统课自然过渡到安全的最佳入口。** |
| 《C 陷阱与缺陷》(C Traps and Pitfalls) | Andrew Koenig | 篇幅不大，把 C 里那些"看起来对但实际是错的"写法讲得很清楚（包括整数提升、类型转换、`printf` 相关的问题）。 |
| 《C 安全编码规范》(The CERT C Coding Standard) | CERT / SEI（卡内基梅隆大学软件工程研究所） | **官方公开的 C 语言安全编码规则集**，每条规则都有编号、说明和示例。它是"安全编码规范"这一主题最权威、最可操作的来源之一。 |
| 《Rust 程序设计语言》(The Rust Programming Language) | Steve Klabnik & Carol Nichols | 想理解"语言层面如何消除 UAF 与 double free"读这本。**官方免费电子版**（"the book"）。 |
| Valgrind 与 Sanitizer 官方文档 | valgrind.org / LLVM 与 GCC 文档 | 工具的行为细节（ASan 支持哪些错误、各选项的含义）以官方文档为准，不要靠二手资料。 |
| OWASP 相关指南 | OWASP | 应用安全方向的公开资料。注意区分"内存安全"与"Web 安全"是两个不同的问题域。 |

**关于漏洞编号的一点说明**：本仓库的笔记**不引用具体 CVE 编号**。如果你需要联系真实案例，去 **NVD（National Vulnerability Database）** 或厂商安全公告检索。**不要从博客或二手文章里抄编号**——那些资料里的编号经常张冠李戴，而我无法核实。

## 二、数据库与数据管理

### 2.1 主教材

| 书名 | 作者 | 说明 |
| --- | --- | --- |
| **《数据库系统概念》(Database System Concepts)** | Abraham Silberschatz, Henry F. Korth, S. Sudarshan | **数据库课程的标准教材**（常被称作"数据库恐龙书"）。覆盖关系模型、SQL、范式、事务、并发控制、恢复、存储与索引。理论体系完整，与大多数课程的术语最一致。**注意它偏理论**，SQL 的实战技巧需要另找材料。 |
| **《数据库系统实现》(Database System Implementation)** | Hector Garcia-Molina, Jeffrey D. Ullman, Jennifer Widom | 讲**数据库内部怎么实现**：存储管理、索引结构、查询处理与优化、事务管理。**读完《数据库系统概念》之后读这本**，能把"SQL 是怎么被执行的"这件事串起来。 |
| **《数据库管理系统》(Database Management Systems)** | Raghu Ramakrishnan & Johannes Gehrke | 另一本广泛使用的教材，**对查询优化和索引的讲解特别好**。"Cow book"（封面是牛）。 |

### 2.2 SQL 实战

| 书名 | 作者 | 说明 |
| --- | --- | --- |
| **《SQL 基础教程》** | MICK | 日本作者的入门书，**讲解直白、图示多、适合零基础**。缺点是深度有限（不讲窗口函数、CTE 这些）。作为第一本很好。 |
| **《SQL 学习指南》(Learning SQL)** | Alan Beaulieu | **覆盖面比上一本广得多**：子查询、CTE、窗口函数、索引、事务都有涉及，而且实战性强。**如果只能买一本 SQL 书，我推荐这本。** |
| 《SQL 权威指南》(SQL: The Ultimate Guide) / 《SQL 必知必会》(SQL in 10 Minutes, Sams Teach Yourself) | Ben Forta | Forta 的书以"短小、直接、例子驱动"著称，适合快速上手。 |
| 《高性能 MySQL》(High Performance MySQL) | Baron Schwartz, Peter Zaitsev, Vadim Tkachenko 等 | MySQL 运维与优化的经典。讲索引、查询优化、复制、分区的实战细节。**版本更新较快，注意买当前版本。** |
| 《SQL 反模式》(SQL Antipatterns) | Bill Karwin | **用"反模式"的方式讲数据库设计**：Jaywalking（逗号分隔的列表）、Naive Trees（错误的树形存储）、Fear of the Unknown（滥用 NULL）等。**读起来很有趣，而且讲的都是真实会犯的错误。** 与笔记里反复强调的 NULL 陷阱、约束设计高度呼应。 |
| 《设计数据密集型应用》(Designing Data-Intensive Applications, DDIA) | Martin Kleppmann | **严格说不是数据库教材，而是分布式数据系统**。但它是理解"为什么不同存储引擎有不同权衡""一致性与隔离级别到底差在哪""复制与分区怎么工作"的最好的一本书。**强推**，尤其在你学完基础数据库之后。 |

### 2.3 公开课程

| 课程 | 出处 | 说明 |
| --- | --- | --- |
| **CMU 15-445 / 15-721: Database Systems** | Carnegie Mellon University | Andy Pavlo 讲授的数据库系统课，**从零实现一个存储引擎 + 查询执行器**（BusTub 项目）。配套的课程视频与讲义都公开。这是**理解数据库内部最好的公开材料**。15-721 是高级版，讲现代数据库的专题（列存、向量化执行、并发控制等）。 |
| **CS 186: Introduction to Database Systems** | UC Berkeley | Berkeley 的数据库课，配套项目也不错。 |
| **Stanford CS 145 / InfoLab 课程** | Stanford | 有关于查询优化与数据挖掘的公开材料。 |
| **MIT 6.830: Database Systems** | MIT | 实现一个简单的关系数据库（SimpleDB），项目导向。 |

### 2.4 官方文档（比任何书都重要）

我必须强调这一点：**SQL 是方言差异极大的语言，查具体行为一定要看官方文档，不要看二手教程。**

| 文档 | 用它查什么 |
| --- | --- |
| **PostgreSQL 官方文档** | 语法与语义的权威参考。Tutorial 部分适合通读；"The SQL Language" 部分适合精读；"Server Administration" 用于运维。**个人认为它是写得最好的开源文档之一。** |
| **MySQL 官方文档** | 方言差异（`ONLY_FULL_GROUP_BY`、`ON UPDATE CURRENT_TIMESTAMP`、`TIMESTAMP` 的 2038 限制、`utf8mb4`、`BOOLEAN` 是 `TINYINT(1)` 别名）必须查这里 |
| **SQLite 官方文档** | 类型亲和性 (type affinity)、动态类型、`AUTOINCREMENT` 的细节、`PRAGMA` 命令。SQLite 的类型系统与其他数据库很不一样，不能凭经验套 |
| **Oracle Database SQL Language Reference** | Oracle 的方言（`MINUS`、`VARCHAR2` 的字节/字符语义、`ROWNUM`） |
| **Microsoft SQL Server 文档** | T-SQL 的扩展（`TOP`、`NVARCHAR`） |

**一个实用建议**：把你要用的那个数据库的官方文档加到浏览器书签里，遇到不确定的行为先查它，**而不是搜索博客**。博客文章经常是基于某个旧版本的，而且很少标注适用于哪个版本。

## 三、学习路线的建议

基于我自己的经验，给出两个方向的一个可行顺序。**这不是唯一正确的顺序，但它避免了"先啃最难的"这类常见问题。**

### 3.1 操作系统方向

```text
第 1 步：建立"程序怎么跑"的直觉（2-4 周）
  CSAPP 第 1-6 章（数据表示、汇编、内存层次）
  配合：在 Linux 上用 gcc -S 看自己写的 C 代码的汇编

第 2 步：系统编程（3-4 周）
  CSAPP 第 8-10 章（异常控制流、虚拟内存）
  APUE 的对应章节（进程、信号、文件 I/O）—— 不要通读，按需查
  配合：写小程序亲手验证（fork、exec、signal handler、mmap）

第 3 步：操作系统原理（6-10 周）
  《操作系统导论》(OSTEP) 通读 + 做 homework
  或者：《操作系统概念》通读
  配合：把本仓库 notes/operating-systems/01-memory-management.md 当作自己的复习提纲

第 4 步：亲手实现一遍（8-12 周，投入产出比最高）
  MIT 6.S081 的 xv6 labs（至少做完 page tables 与 traps）
  这一步会让你前面学的东西全部"落地"

第 5 步：方向选择
  - 想做系统/内核 → 《Linux 内核设计与实现》→ 读内核源码 → 《性能之巅》
  - 想做安全 → CSAPP 第 3 章 → CERT C 规范 → 工具链（ASan/Valgrind/fuzzing）
  - 想做分布式 → MIT 6.824 → DDIA
```

### 3.2 数据库方向

```text
第 1 步：SQL 会用（3-4 周）
  《SQL 基础教程》或 《SQL 学习指南》
  配合：真的建库、跑查询（SQLite 最省事，PostgreSQL 最接近生产）
  配合：LeetCode 数据库题库刷 50 题左右建立手感

第 2 步：SQL 用对（2-3 周）
  重点攻三个主题：
    - NULL 与三值逻辑（大量 bug 的来源）
    - 连接的语义（尤其 LEFT JOIN 的 ON vs WHERE）
    - 逻辑执行顺序（为什么窗口函数不能在 WHERE 里用）
  配合：本仓库 notes/sql/01-06 的笔记与 10 道练习

第 3 步：数据库理论（8-12 周）
  《数据库系统概念》：关系模型、范式、事务、并发控制、恢复
  这一阶段要动笔：自己推导范式分解、自己画并发调度的依赖图

第 4 步：数据库内部（8-12 周）
  CMU 15-445 的讲义 + BusTub 项目
  或者《数据库系统实现》/《数据库管理系统》
  配合：EXPLAIN 反复观察自己写的查询的实际执行计划

第 5 步：工程与规模（持续）
  索引设计与优化、分区、复制
  《高性能 MySQL》或对应数据库的运维资料
  DDIA：理解分布式数据系统的一致性与权衡
```

### 3.3 两个方向共同的三条经验

1. **一定要动手，而且要"故意破坏"**。看懂"`NOT IN` 遇到 NULL 返回空集"和亲手制造一次看到空结果，是完全不同的理解深度。本仓库的 SQL 练习文件里专门设计了"故意的实验"，就是这个原因。

2. **查官方文档，不要只靠教程**。教程会过时、会简化、会省略版本差异。数据库和操作系统这两个领域的**行为差异极其依赖版本**，一手资料是唯一可靠的来源。

3. **把学到的东西写成自己的笔记**。写不出来通常意味着没理解透。这个仓库本身就是这个习惯的产物——每一篇笔记都是"我能不能把它讲清楚"的一次检验。

## 四、关于这份清单的准确性说明

为了让这份清单可核查，我把能确认的都写明了，也把不确定的标出来：

**我可以确认的**：

- 书名与作者：以上列出的所有书籍名称与作者姓名都是真实的，我不编造书名或作者。
- 《操作系统导论》(Operating Systems: Three Easy Pieces) 的作者是 Remzi H. Arpaci-Dusseau 与 Andrea C. Arpaci-Dusseau，二人均在威斯康星大学麦迪逊分校任教，该书围绕虚拟化、并发、持久性三个主题组织。
- CSAPP 的作者是 Randal E. Bryant 与 David R. O'Hallaron（卡内基梅隆大学）。
- 《数据库系统概念》的作者是 Abraham Silberschatz、Henry F. Korth、S. Sudarshan。
- 《设计数据密集型应用》的作者是 Martin Kleppmann。
- MIT 6.S081/6.828 使用 xv6；CMU 15-445 由 Andy Pavlo 讲授并使用 BusTub。
- CERT C Coding Standard 由卡内基梅隆大学软件工程研究所 (SEI) 维护。

**我不确定的、因此没有写的**：

- **具体版本次号与出版年份**。教材改版频繁，我无法确认"现在是第几版"。买书前请查出版社或书店的当前信息。
- **各书的 ISBN**。我没有核实的渠道，不编造。
- **公开课程的具体周数、作业数量、评分比重**。这些会随学期变化。
- **某些电子版是否仍然免费开放**（例如 OSTEP）。我写"作者公开了电子版"是基于长期情况，但具体当前的获取方式请自行到官网确认。
- **某本书的最新版是否新增了某个章节**。我只描述该书长期以来稳定的整体特点。

**我刻意避开的**：

- 不列任何"下载链接"或网盘资源（版权与链接有效性都无法保证）。
- 不引用任何具体 CVE 编号或统计数字。
- 不推荐具体的商业培训课程（我无法评估其质量）。

如果你发现某个书名、作者或归属写错了，非常欢迎提 Issue 指正——**准确性比篇幅重要。**
