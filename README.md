# Study-Notes

个人计算机专业课学习笔记仓库：Linux、SQL 与操作系统方向的中文笔记整理，技术术语保留英文原文。

## 关于这个仓库

我是厦门大学马来西亚分校（Xiamen University Malaysia）计算机科学与技术专业（Computer Science）的本科生，课程为全英文授课。这个仓库是我在课外自学与复习时整理的笔记，主要目的是：

- 把英文授课的课程内容用中文重新梳理一遍，检验自己是否真的理解了；
- 把零散的终端操作、SQL 语句沉淀成可以随时查阅的速查材料；
- 记录操作系统底层机制（内存管理、内存类漏洞、防御手段）的学习路径。

需要说明的是：**这不是官方教材或课程讲义，而是一名本科生的自学记录**。内容可能仍有理解偏差或遗漏，如果发现错误，欢迎提 Issue 指正。

## 目录结构

```text
Study-Notes/
├── README.md
├── LICENSE
├── .gitignore
├── notes/
│   ├── linux/
│   │   ├── README.md                    # 索引 + 学习路线
│   │   ├── 01-file-system.md            # 目录结构、常用命令、权限与 chmod/chown
│   │   ├── 02-process-and-jobs.md       # 进程、作业控制、信号
│   │   ├── 03-text-processing.md        # grep/sed/awk/sort/uniq/wc/cut 与管道
│   │   ├── 04-permissions-and-users.md  # 用户组、sudo、umask、特殊权限位
│   │   ├── 05-package-and-service.md    # apt/yum 与 systemctl
│   │   ├── 06-shell-scripting.md        # Shell 变量、条件、循环、函数
│   │   └── 07-vm-and-env-setup.md       # VMware 虚拟机、网络模式、快照、开发环境
│   ├── sql/
│   │   ├── README.md                    # 索引 + 学习路线
│   │   ├── 01-basic-query.md            # SELECT/WHERE/ORDER BY/LIMIT/DISTINCT
│   │   ├── 02-joins.md                  # 各类 JOIN、自连接
│   │   ├── 03-aggregation.md            # GROUP BY/HAVING/聚合函数
│   │   ├── 04-subquery-and-cte.md       # 子查询、EXISTS、WITH
│   │   ├── 05-ddl-and-constraints.md    # CREATE TABLE、约束、索引基础
│   │   └── 06-practice-queries.md       # 10 道练习题 + 参考答案
│   └── operating-systems/
│       ├── README.md                    # 索引 + 学习路线
│       ├── 01-memory-management.md      # 分页、分段、虚拟内存、TLB、地址翻译
│       ├── 02-memory-vulnerabilities.md # 缓冲区溢出、UAF、double free、格式化字符串
│       ├── 03-defense-strategies.md     # ASLR、DEP/NX、canary、RELRO、CFI
│       └── 04-config-vs-runtime-gap.md  # 配置校验与运行时行为的脱节：分析框架
├── paper/
│   ├── README.md                        # 课程论文存档说明
│   └── os-memory-management-vulnerabilities.pdf   # Linux 课程论文（英文，约 2,000 词）
└── resources/
    ├── reading-list.md                  # 操作系统 / 数据库经典书目与公开课程
    └── command-cheatsheet.md            # 高频 Linux + SQL 命令速查表
```

## 目录（Table of Contents）

| 板块 | 主题 | 入口 |
| --- | --- | --- |
| Linux | 文件系统、进程、文本处理、权限、包管理、Shell 脚本、虚拟机环境 | [notes/linux/README.md](notes/linux/README.md) |
| SQL | 基础查询、连接、聚合、子查询与 CTE、DDL 与约束、练习 | [notes/sql/README.md](notes/sql/README.md) |
| Operating Systems | 内存管理、内存类漏洞、防御策略、配置与运行时脱节分析 | [notes/operating-systems/README.md](notes/operating-systems/README.md) |
| 课程论文 | 操作系统内存管理漏洞分析与防御策略（英文课程论文存档） | [paper/README.md](paper/README.md) |
| Resources | 书单与公开课程、命令速查表 | [resources/reading-list.md](resources/reading-list.md) · [resources/command-cheatsheet.md](resources/command-cheatsheet.md) |

### Linux 笔记直达

1. [文件系统与常用命令](notes/linux/01-file-system.md)
2. [进程与作业控制](notes/linux/02-process-and-jobs.md)
3. [文本处理三剑客与管道](notes/linux/03-text-processing.md)
4. [权限、用户与组](notes/linux/04-permissions-and-users.md)
5. [软件包与服务管理](notes/linux/05-package-and-service.md)
6. [Shell 脚本基础](notes/linux/06-shell-scripting.md)
7. [虚拟机与开发环境搭建](notes/linux/07-vm-and-env-setup.md)

### SQL 笔记直达

1. [基础查询](notes/sql/01-basic-query.md)
2. [连接查询](notes/sql/02-joins.md)
3. [聚合与分组](notes/sql/03-aggregation.md)
4. [子查询与 CTE](notes/sql/04-subquery-and-cte.md)
5. [DDL 与约束](notes/sql/05-ddl-and-constraints.md)
6. [练习题与参考答案](notes/sql/06-practice-queries.md)

### 操作系统笔记直达

1. [内存管理基础](notes/operating-systems/01-memory-management.md)
2. [常见内存类漏洞](notes/operating-systems/02-memory-vulnerabilities.md)
3. [防御策略](notes/operating-systems/03-defense-strategies.md)
4. [配置校验与运行时执行之间的脱节](notes/operating-systems/04-config-vs-runtime-gap.md)

## 学习进度

- [x] Linux：目录结构与文件操作命令
- [x] Linux：进程、作业控制与信号
- [x] Linux：grep / sed / awk 与管道组合
- [x] Linux：权限、用户与组、特殊权限位
- [x] Linux：apt / systemctl 基础
- [x] Linux：Shell 脚本（变量、条件、循环、函数）
- [x] Linux：VMware 虚拟机与开发环境搭建
- [x] SQL：单表查询与排序、去重
- [x] SQL：多表连接（INNER / LEFT / RIGHT / FULL / 自连接）
- [x] SQL：聚合函数与 GROUP BY / HAVING
- [x] SQL：子查询与 CTE
- [x] SQL：DDL、约束与索引基础
- [x] SQL：练习题 10 道
- [x] OS：内存管理（分页、分段、虚拟内存、TLB、page fault）
- [x] OS：内存类漏洞（栈/堆溢出、UAF、double free、格式化字符串）
- [x] OS：防御策略（ASLR、DEP/NX、canary、RELRO、CFI）
- [x] OS：配置校验与运行时脱节的案例分析框架
- [x] 资源：经典书目与公开课程清单
- [x] 资源：Linux + SQL 命令速查表
- [ ] 补充：文件系统实现（inode、日志、ext4 结构）专题
- [ ] 补充：并发与同步（锁、信号量、条件变量）专题
- [ ] 补充：数据库事务与隔离级别实验记录
- [ ] 待放入：`paper/` 下的课程论文 PDF（本地整理中）

## 阅读建议

- 每个板块的 `README.md` 是索引，包含该方向的学习路线与文件说明，建议先读它。
- 每个主要笔记结尾都有「小结」，复习时可以只看小结快速回忆。
- `resources/command-cheatsheet.md` 适合当工具书用，不需要通读。
- 代码块中的命令与 SQL 均为英文，可直接复制到终端或数据库客户端执行；SQL 示例基于通用的标准 SQL，在 SQLite / PostgreSQL / MySQL 上大体通用，个别方言差异会在文中标出。

## 引用与准确性

笔记中提到的书籍、课程、工具均为真实存在的资料，引用时尽量给出作者与版本信息；正文不编造 CVE 编号、不编造统计数据。对于存在版本差异的行为（例如不同发行版的命令差异），文中会明确说明适用范围。

## License

本项目采用 [MIT License](LICENSE)，Copyright (c) 2026 Yukang Wang。
