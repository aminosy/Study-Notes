# SQL 学习笔记

数据库与 SQL 方向的自学笔记索引：从单表查询到连接、聚合、子查询与 CTE，再到 DDL、约束与索引基础，最后用 10 道练习闭环。

## 这个板块在学什么

我的目标不是背语法，而是三个具体的能力：

1. **能写出正确的查询** —— 尤其是理解 SQL 的**逻辑执行顺序**、`NULL` 的三值逻辑、`WHERE` 与 `HAVING` 的区别这些"不写对就一定出错"的点；
2. **能读懂别人写的查询** —— 子查询嵌套三层、CTE 串成流水线时还能看懂它想干什么；
3. **能设计出合理的表结构** —— 约束怎么定、索引怎么建、什么时候该拆表。

这三个能力里，**第一个靠理解语义，第二个靠熟悉模式，第三个靠踩坑经验**。所以本板块里我会反复强调"为什么"，而不只是"怎么写"。

## 文件说明

| 文件 | 内容 | 难度 |
| --- | --- | --- |
| [01-basic-query.md](01-basic-query.md) | 逻辑执行顺序、SELECT/WHERE/ORDER BY/LIMIT/DISTINCT、三值逻辑与 NULL、`NOT IN` 的 NULL 陷阱 | 入门 |
| [02-joins.md](02-joins.md) | INNER/LEFT/RIGHT/FULL JOIN、自连接、Venn 图解法、`ON` 与 `WHERE` 在外连接中的区别 | 入门-中 |
| [03-aggregation.md](03-aggregation.md) | COUNT/SUM/AVG/MIN/MAX、`GROUP BY`/`HAVING`、聚合与 NULL 的完整规则、窗口函数入门 | 中 |
| [04-subquery-and-cte.md](04-subquery-and-cte.md) | 标量/行/列/表子查询、`IN` vs `EXISTS`、派生表、`WITH` 与递归 CTE | 中-高 |
| [05-ddl-and-constraints.md](05-ddl-and-constraints.md) | 数据类型、`CREATE TABLE`、主键/外键/唯一/非空/默认/CHECK、索引与最左前缀原则、`EXPLAIN` | 中-高 |
| [06-practice-queries.md](06-practice-queries.md) | 10 道由易到难的练习题 + 完整参考答案与易错点分析 | 综合 |

## 学习路线

```text
第 1 步：理解 SQL 的执行模型
  01-basic-query.md
    - 书写顺序 ≠ 逻辑执行顺序：
        FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
    - 由此推出的三个结论：
        · WHERE 里不能用聚合函数（那时还没分组）
        · WHERE 里通常不能用 SELECT 的别名（那时还没算别名）
        · ORDER BY 里可以用别名（它在 SELECT 之后）
    - NULL 是三值逻辑（TRUE/FALSE/UNKNOWN），WHERE 只保留 TRUE
  过关标准：能解释"为什么 `WHERE score > AVG(score)` 会报错"以及该怎么改

第 2 步：把多张表连起来
  02-joins.md
    - 连接的本质是笛卡尔积 + 过滤
    - 四种外连接的语义差别，能用图画出来
    - 【重点】LEFT JOIN 的 ON 条件与 WHERE 条件的区别
        ON 里的条件不满足 → 保留左表的行、右表补 NULL
        WHERE 里的条件不满足 → 整行被过滤掉（外连接退化成内连接）
    - 自连接：同一张表起两个别名
  过关标准：能一眼看出"LEFT JOIN 之后在 WHERE 里过滤右表列"这个经典错误

第 3 步：把行折叠成结论
  03-aggregation.md
    - COUNT(*) vs COUNT(col) vs COUNT(DISTINCT col)，以及它们对 NULL 的处理
    - AVG(col) 的分母是非 NULL 的行数，所以 AVG ≠ SUM/COUNT(*)
    - WHERE（分组前过滤行）与 HAVING（分组后过滤组）的边界
    - 窗口函数：不折叠行的"聚合"，用于组内排名
  过关标准：能说出每个聚合函数遇到 NULL 时的行为

第 4 步：把复杂逻辑拆成可读的步骤
  04-subquery-and-cte.md
    - 子查询的四种返回形态：标量 / 行 / 列 / 表
    - NOT IN 与 NULL 的陷阱，以及为什么 NOT EXISTS 更安全
    - 派生表 vs CTE：可读性与复用性
    - 递归 CTE：anchor + UNION ALL + recursive member
  过关标准：能把一个三层嵌套子查询改写成同等语义的 CTE 流水线

第 5 步：设计表而不是只查表
  05-ddl-and-constraints.md
    - 数据类型选型（金额与成绩必须用 NUMERIC/DECIMAL，不能用 FLOAT）
    - 六类约束的语义与违反时的行为
    - 索引：为什么能加速、最左前缀原则、什么时候索引用不上
    - 代理键 vs 自然键
  过关标准：能解释 `WHERE YEAR(birth_date) = 2005` 为什么用不上索引，以及怎么写才对

第 6 步：用练习检验
  06-practice-queries.md
    - 10 道题覆盖上面全部知识点
    - 先在真实数据库里跑一遍，再对照参考答案
    - 重点看"易错点"，那些才是真正区分会与不会的地方
  过关标准：10 道题里至少 8 道能独立写对，且能说出为什么这么写
```

## 本板块统一使用的示例数据库

所有文件复用同一套结构（学生 / 教师 / 课程 / 选课），这样看到任何一篇文章时都不用重新理解表结构。

```sql
-- 教师表
CREATE TABLE teachers (
    teacher_id   INTEGER PRIMARY KEY,
    teacher_name VARCHAR(50) NOT NULL,
    department   VARCHAR(50)
);

-- 学生表
CREATE TABLE students (
    student_id   INTEGER PRIMARY KEY,
    name         VARCHAR(50)  NOT NULL,
    gender       CHAR(1),                     -- 'M' / 'F'
    birth_date   DATE,
    major        VARCHAR(50),
    city         VARCHAR(50),
    enroll_year  INTEGER
);

-- 课程表
CREATE TABLE courses (
    course_id    INTEGER PRIMARY KEY,
    course_name  VARCHAR(100) NOT NULL,
    credits      INTEGER,
    teacher_id   INTEGER REFERENCES teachers(teacher_id)
);

-- 选课表（成绩）
CREATE TABLE enrollments (
    enrollment_id INTEGER PRIMARY KEY,
    student_id    INTEGER REFERENCES students(student_id),
    course_id     INTEGER REFERENCES courses(course_id),
    score         NUMERIC(5,2),               -- 100 分制，NULL 表示未出成绩
    semester      VARCHAR(20),                -- 例如 '2024-Fall'
    UNIQUE (student_id, course_id, semester)
);
```

表之间的关系：

```text
    teachers 1 ─────< courses          一门课由一位教师讲授
       │                 │
       │                 │
       │            enrollments >───── 1 students
       │                 │
       └─────────────────┴─── 选课记录把学生与课程连起来

    teachers  ──1:N──>  courses
    students  ──1:N──>  enrollments
    courses   ──1:N──>  enrollments
    students  ──N:M──>  courses   （通过 enrollments 实现）
```

**几个结构上的设计说明**（这些在后面会反复用到）：

- `enrollments` 是**关联表 (junction table)**，把 N:M 的"学生—课程"关系拆成两个 1:N 关系。这是关系型设计的标准做法。
- `score` **允许为 NULL**，语义是"成绩还没出"，而不是"考了 0 分"。**这个区分是本套笔记里大量 NULL 陷阱的来源，是有意设计的。**
- `UNIQUE (student_id, course_id, semester)` 保证同一个学生同一学期不会重复选同一门课。
- 没有 `NOT NULL` 约束的列（如 `major`、`city`）在真实业务里通常也应该加约束，这里保留可空是为了制造练习用的边界情况。

## 推荐练习平台与资料

| 资源 | 说明 |
| --- | --- |
| **LeetCode 数据库题库** | 按难度分级的 SQL 题，题目质量整体较好，适合刷手感 |
| **HackerRank SQL** | 分 Basic / Intermediate / Advanced 三级，前面几节很适合入门 |
| **SQLZoo** | 交互式教程，边学边在网页里跑 SQL，适合完全零基础时起步 |
| **PostgreSQL 官方文档** | 公认写得最完整的 SQL 参考之一，语法与语义都能查；其 Tutorial 部分也适合通读 |
| **MySQL 官方文档** | 查方言差异时必需（尤其是 `ONLY_FULL_GROUP_BY`、`GROUP BY` 的宽松行为） |
| **SQLite 官方文档** | 查 SQLite 的类型亲和性、动态类型等特殊行为 |
| 《SQL 基础教程》 | MICK 著，日本作者的入门书，讲解直白，适合作为第一本 |
| 《SQL 学习指南》 | Alan Beaulieu 著，覆盖面比上一本广，讲到窗口函数与性能 |

**关于版本的具体能力**，本板块只在确信的地方标注（例如"MySQL 8.0+ 支持窗口函数"、"MySQL 不支持 FULL OUTER JOIN"）。遇到不确定版本行为的地方，会明确说明"不同数据库/版本行为不同，需要用 `EXPLAIN` 或实际测试确认"，不会替某个版本下结论。

## 说明

- 示例基于**通用标准 SQL**，在 SQLite / PostgreSQL / MySQL 上大体通用。方言差异会在正文里就地标注，不会混在一起讲。
- 文中所有查询都是可执行的；示例输出若为示意会明确写成"（示意）"。
- 本板块是本科生的自学记录，不是官方教材或课程讲义。如果发现错误，欢迎提 Issue。
