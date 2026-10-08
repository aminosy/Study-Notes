# 课程论文存档

这个目录存放我在本科阶段完成的课程论文与研究报告，用于留存学术写作记录。

## 已有论文

### Exploiting the Gap between Configuration Verification and Runtime Execution: Analysis of Common Vulnerabilities in Operating System Memory Management and Its Defense Strategies

- **课程**：Linux 系统（Linux System）
- **语言**：英文
- **篇幅**：约 2,000 词
- **完成情况**：独立完成，课程评分优秀，获任课教师肯定
- **文件**：`os-memory-management-vulnerabilities.pdf`

**研究内容**：围绕操作系统内存管理中「配置校验（configuration verification）」与
「运行时执行（runtime execution）」之间的脱节展开分析——即静态的配置检查无法覆盖运行时
的实际行为，由此产生的安全缺口如何被利用。论文梳理了常见的内存类漏洞（缓冲区溢出、
use-after-free、double free、整数溢出导致的越界访问等）的成因，并结合防御机制
（ASLR、DEP/NX、栈保护、CFI 等）讨论缓解思路。

> 笔记目录 [`../notes/operating-systems/`](../notes/operating-systems/) 是在这篇论文基础上
> 整理出的知识点沉淀。

---

## 如何添加新论文

1. 把 PDF 放进本目录，文件名用英文小写短横线格式，例如 `database-index-design.pdf`
2. 在上面的「已有论文」下面新建一个 `###` 小节，写清楚课程、语言、篇幅、完成方式
3. 用一段话说明研究问题与主要结论，方便阅读者快速判断内容
