# SQL 聚合与分组：GROUP BY / HAVING 与聚合函数

本篇梳理聚合函数的语义（尤其是它们对 NULL 的处理）、`GROUP BY` 与 `HAVING` 的边界、按表达式分组，以及"不折叠行"的窗口函数入门。

## 0. 示例数据库

```sql
CREATE TABLE teachers (
    teacher_id   INTEGER PRIMARY KEY,
    teacher_name VARCHAR(50) NOT NULL,
    department   VARCHAR(50)
);

CREATE TABLE students (
    student_id   INTEGER PRIMARY KEY,
    name         VARCHAR(50)  NOT NULL,
    gender       CHAR(1),                     -- 'M' / 'F'
    birth_date   DATE,
    major        VARCHAR(50),
    city         VARCHAR(50),
    enroll_year  INTEGER
);

CREATE TABLE courses (
    course_id    INTEGER PRIMARY KEY,
    course_name  VARCHAR(100) NOT NULL,
    credits      INTEGER,
    teacher_id   INTEGER REFERENCES teachers(teacher_id)
);

CREATE TABLE enrollments (
    enrollment_id INTEGER PRIMARY KEY,
    student_id    INTEGER REFERENCES students(student_id),
    course_id     INTEGER REFERENCES courses(course_id),
    score         NUMERIC(5,2),               -- 100 分制，NULL 表示未出成绩
    semester      VARCHAR(20),                -- 例如 '2024-Fall'
    UNIQUE (student_id, course_id, semester)
);
```

示例数据（**注意第 5 行 score 是 NULL，第 6 行的分数是 0，这两行的区别在本篇里反复用到**）：

```sql
INSERT INTO students (student_id, name, gender, major, city, enroll_year) VALUES
    (1, '王小明', 'M', '计算机科学与技术', '厦门', 2022),
    (2, '李雨桐', 'F', '计算机科学与技术', '北京', 2022),
    (3, '张浩然', 'M', '软件工程',         '厦门', 2021),
    (4, '陈思远', 'M', '计算机科学与技术', '上海', 2023),
    (5, '刘悦然', 'F', '数据科学',         NULL,   2022),
    (6, '赵子涵', 'F', '软件工程',         '厦门', 2023);

INSERT INTO courses (course_id, course_name, credits) VALUES
    (101, '程序设计基础',   4),
    (102, '数据结构',       4),
    (103, '数据库系统原理', 3),
    (104, '操作系统',       4),
    (105, '离散数学',       3),
    (106, '软件工程导论',   3);

INSERT INTO enrollments (enrollment_id, student_id, course_id, score, semester) VALUES
    (1,  1, 101, 88.5, '2024-Fall'),
    (2,  1, 102, 92.0, '2024-Fall'),
    (3,  1, 103, 79.0, '2025-Spring'),
    (4,  2, 101, 95.5, '2024-Fall'),
    (5,  2, 103, NULL, '2025-Spring'),   -- 成绩未出
    (6,  3, 101, 72.0, '2024-Fall'),
    (7,  3, 105, 85.0, '2024-Fall'),
    (8,  4, 102, 91.0, '2025-Spring'),
    (9,  4, 104, 67.5, '2025-Spring'),
    (10, 5, 103, 88.0, '2025-Spring'),
    (11, 5, 104, 0,    '2025-Spring'),   -- 0 分，与 NULL 完全不同！
    (12, 6, 101, 59.0, '2024-Fall');
```

**`(5, 104, 0)` 这一行是刻意加的**：它让你能直观看到 `AVG` 与 `SUM/COUNT(*)` 的差别，以及"0 分"和"未出成绩"在聚合里的不同命运。

## 1. 聚合函数

聚合函数把**多行折叠成一行**（或者折成一个值）。

### 1.1 函数总表（含对 NULL 的行为）

| 函数 | 作用 | 遇到 NULL | 无行时返回 |
| --- | --- | --- | --- |
| `COUNT(*)` | 数**行数** | **不忽略**——它数的是行，不检查任何列 | `0` |
| `COUNT(col)` | 数 `col` **非 NULL** 的行数 | **跳过** `col` 为 NULL 的行 | `0` |
| `COUNT(DISTINCT col)` | 数 `col` 的**不同非 NULL 值**个数 | **跳过** NULL | `0` |
| `SUM(col)` | 求和 | **跳过** NULL | **`NULL`** |
| `AVG(col)` | 求平均 | **跳过** NULL（分母是非 NULL 行数） | **`NULL`** |
| `MIN(col)` | 最小值 | **跳过** NULL | **`NULL`** |
| `MAX(col)` | 最大值 | **跳过** NULL | **`NULL`** |
| `GROUP_CONCAT` / `STRING_AGG` / `ARRAY_AGG` | 把值拼成字符串或数组 | 行为因实现而异，通常跳过 NULL | `NULL` |
| `STDDEV` / `VARIANCE` | 标准差 / 方差 | 跳过 NULL | `NULL` |

**这张表值得背下来**，因为它是大量 SQL bug 的根源。

### 1.2 `COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`

```sql
-- 三种计数的对比（基于上面的数据，enrollments 共 12 行，其中 1 行 score 为 NULL）
SELECT
    COUNT(*)                    AS all_rows,        -- 12
    COUNT(score)                AS scored_rows,     -- 11（跳过了 1 个 NULL）
    COUNT(*) - COUNT(score)     AS missing_scores,  -- 1
    COUNT(DISTINCT course_id)   AS distinct_courses,-- 5（101,102,103,104,105）
    COUNT(DISTINCT student_id)  AS distinct_students-- 6
FROM enrollments;
```

**`COUNT(*)` 与 `COUNT(1)` 的关系**：两者语义相同（都数行数），性能在几乎所有现代数据库上也相同（优化器会把它们当作同一件事）。所以**不要纠结"哪个更快"，选 `COUNT(*)` 因为它的意图最清楚**。

**"数 NULL 的个数"的正确写法**：

```sql
-- ✅ 用 COUNT(*) 减去 COUNT(col)，或者用 SUM + CASE
SELECT COUNT(*) - COUNT(score) AS null_count FROM enrollments;

SELECT SUM(CASE WHEN score IS NULL THEN 1 ELSE 0 END) AS null_count FROM enrollments;

-- ❌ 这两种都是错的
-- SELECT COUNT(score IS NULL) ...     → score IS NULL 返回 TRUE/FALSE，
--                                        两者都非 NULL，所以 COUNT 会数【所有行】
-- SELECT SUM(score IS NULL) ...       → 这个在部分数据库上其实可以（TRUE=1, FALSE=0），
--                                        但不是标准写法，可移植性差
```

**`COUNT(条件)` 为什么不对**：`COUNT` 的参数是一个**值**，它数的是"这个值非 NULL 的行数"。而 `score IS NULL` 这个表达式的值只可能是 `TRUE` 或 `FALSE`，**永远不是 NULL**，所以 `COUNT(score IS NULL)` 等于 `COUNT(*)`。这是一个非常常见且难以察觉的错误。

### 1.3 `AVG` 的分母是什么 —— 一个必须讲清的点

```sql
-- 基于数据：score 的值是 88.5, 92.0, 79.0, 95.5, NULL, 72.0, 85.0, 91.0, 67.5, 88.0, 0, 59.0
SELECT
    SUM(score)                    AS total,       -- 817.5（NULL 被跳过，0 被计入）
    COUNT(*)                      AS all_rows,    -- 12
    COUNT(score)                  AS scored_rows, -- 11
    AVG(score)                    AS avg_score,   -- 817.5 / 11 = 74.318...
    SUM(score) / COUNT(*)         AS avg_if_null_zero,  -- 817.5 / 12 = 68.125
    SUM(COALESCE(score, 0)) / COUNT(*) AS avg_alt -- 817.5 / 12 = 68.125
FROM enrollments;
```

**关键结论**：**`AVG(score)` 的分母是 `COUNT(score)`（非 NULL 行数），不是 `COUNT(*)`。** 所以：

```text
AVG(col) = SUM(col) / COUNT(col)        ← 不等于 SUM(col) / COUNT(*)
```

**这两者的选择是一个业务决策，不是技术决策**：

| 你的意图 | 应该用 |
| --- | --- |
| "已出成绩的学生的平均分是多少" | `AVG(score)`（分母只算有成绩的） |
| "把这门课当成所有人都参加考试，未出成绩按 0 分算" | `SUM(COALESCE(score,0)) / COUNT(*)` |
| "未出成绩的不参与统计" | `AVG(score)` |

**在做成绩统计之前必须明确这一点**，否则算出来的"平均分"可能完全不是想要的东西。

### 1.4 `SUM`/`AVG`/`MIN`/`MAX` 在无行时返回 `NULL`

这是一个实际会影响程序的地方：

```sql
-- 假设有一个不存在学生的学号
SELECT SUM(score)   FROM enrollments WHERE student_id = 999;   -- NULL（不是 0！）
SELECT COUNT(*)     FROM enrollments WHERE student_id = 999;   -- 0
SELECT AVG(score)   FROM enrollments WHERE student_id = 999;   -- NULL
SELECT MAX(score)   FROM enrollments WHERE student_id = 999;   -- NULL
```

**`SUM` 返回 NULL 而不是 0** 是 SQL 标准的行为。这会带来两类问题：

1. **程序里直接使用会出错**（很多语言把 NULL 映射成 `null`/`None`，做加法会抛异常或者变成 `null`）；
2. **与 `COUNT` 的行为不一致**，让人意外。

```sql
-- ✅ 总是用 COALESCE 包一层
SELECT COALESCE(SUM(score), 0) AS total_score FROM enrollments WHERE student_id = 999;

-- 或者用 CASE 处理（可移植性相同，但更啰嗦）
SELECT CASE WHEN COUNT(*) = 0 THEN 0 ELSE SUM(score) END FROM enrollments WHERE student_id = 999;
```

### 1.5 `MIN`/`MAX` 也可以用于非数值列

```sql
SELECT MIN(name) AS first_by_name, MAX(name) AS last_by_name FROM students;
-- 按排序规则（collation）取字典序最小/最大的字符串

SELECT MIN(birth_date) AS earliest, MAX(birth_date) AS latest FROM students;
-- 日期同样适用
```

**`MIN`/`MAX` 对字符串的行为取决于排序规则 (collation)**，在不同字符集与 `LC_COLLATE` 设置下结果可能不同（尤其涉及中文、大小写混合时）。`MIN`/`MAX` 也**不会做"最长/最短"**——那需要用 `LENGTH()` 配合 `ORDER BY` 或窗口函数。

## 2. `GROUP BY`

`GROUP BY` 把行**按分组键分桶**，每个桶折叠成一行。

```sql
-- 每门课的选课人数与平均分
SELECT course_id,
       COUNT(*)    AS student_count,
       AVG(score)  AS avg_score
FROM enrollments
GROUP BY course_id
ORDER BY course_id;
```

### 2.1 多列分组

```sql
-- 每个学期的每门课的统计
SELECT semester, course_id,
       COUNT(*) AS n,
       ROUND(AVG(score), 2) AS avg_score
FROM enrollments
GROUP BY semester, course_id
ORDER BY semester, course_id;
```

**分组键的顺序会影响输出的顺序（在某些数据库上），但不影响分组结果本身。** 想控制输出顺序就显式 `ORDER BY`，不要依赖 `GROUP BY` 的隐含排序（现代优化器常常会打乱它）。

**分组键是"组合"**：`GROUP BY semester, course_id` 会把 `('2024-Fall', 101)` 和 `('2024-Fall', 102)` 分到不同的组，把 `('2024-Fall', 101)` 和 `('2025-Spring', 101)` 也分到不同的组。

### 2.2 分组后 `SELECT` 能写什么

**规则**：`SELECT` 列表里只能出现

1. **分组键**（在 `GROUP BY` 里出现过的列或表达式），以及
2. **聚合函数**

```sql
-- ✅ 合法：course_id 是分组键，其余是聚合
SELECT course_id, COUNT(*), AVG(score)
FROM enrollments GROUP BY course_id;

-- ❌ 非法（在大多数数据库上）：student_id 既不是分组键也不是聚合
SELECT course_id, student_id, COUNT(*)
FROM enrollments GROUP BY course_id;
-- PostgreSQL 报错：column "enrollments.student_id" must appear in the
--                  GROUP BY clause or be used in an aggregate function
```

**为什么这条规则存在**：一个组里有多个 `student_id` 值（例如 101 号课有 3 个学生选），数据库**无法决定该显示哪一个**。这不是语法限制，而是语义上无解。

**想要"组里的某一行的值"怎么办**：

```sql
-- 需要"每个组里分数最高的那一行的学生是谁" → 用聚合把那个值取出来
SELECT course_id,
       MAX(score) AS top_score,
       COUNT(*)   AS n
FROM enrollments
GROUP BY course_id;

-- 如果需要"取最高分那一行的【其他列】"，GROUP BY 做不到，要用窗口函数（见第 5 节）
-- 或者用一个自连接/子查询去"找回"那一行
```

**关于 MySQL 的宽松行为（需要准确表述）**：MySQL 长期允许 `SELECT` 里出现非分组列，此时它从组里**任意取一行**的值（结果不确定）。`ONLY_FULL_GROUP_BY` 这个 SQL 模式可以强制检查这条规则，而且**在 MySQL 5.7 及之后的版本里它默认是开启的**。所以：

- 在较新的 MySQL 上，上面那个"非法"的查询**会报错**（因为 `ONLY_FULL_GROUP_BY` 默认开启）；
- 你可能会看到"MySQL 允许这么写"的说法——那是更早版本或关闭了该模式的情况；
- **正确的心态是：不要依赖这种宽松性，永远按标准写。** 这样换数据库时不会出问题，也不会因为"取了组里任意一行"而得到随机结果。

### 2.3 按表达式或 `CASE WHEN` 分组

分组键不一定是列，可以是任意表达式。**按分数段分组**是最实用的例子：

```sql
SELECT
    CASE
        WHEN score >= 90 THEN 'A (90-100)'
        WHEN score >= 80 THEN 'B (80-89)'
        WHEN score >= 70 THEN 'C (70-79)'
        WHEN score >= 60 THEN 'D (60-69)'
        WHEN score IS NULL THEN '未出成绩'
        ELSE 'F (0-59)'
    END AS grade_band,
    COUNT(*) AS n
FROM enrollments
GROUP BY
    CASE
        WHEN score >= 90 THEN 'A (90-100)'
        WHEN score >= 80 THEN 'B (80-89)'
        WHEN score >= 70 THEN 'C (70-79)'
        WHEN score >= 60 THEN 'D (60-69)'
        WHEN score IS NULL THEN '未出成绩'
        ELSE 'F (0-59)'
    END
ORDER BY grade_band;
```

**两个要点**：

1. **`GROUP BY` 里必须重复整个 `CASE` 表达式**（标准 SQL 不允许用 `SELECT` 的别名分组；MySQL 允许用别名，PostgreSQL 也允许，但不要依赖）。写两遍很啰嗦，**更好的做法是用派生表或 CTE 先算出一列，再分组**：

```sql
-- ✅ 更可读的写法：先算出 grade_band 这一列，再分组
WITH graded AS (
    SELECT enrollment_id,
           CASE
               WHEN score >= 90 THEN 'A (90-100)'
               WHEN score >= 80 THEN 'B (80-89)'
               WHEN score >= 70 THEN 'C (70-79)'
               WHEN score >= 60 THEN 'D (60-69)'
               WHEN score IS NULL THEN '未出成绩'
               ELSE 'F (0-59)'
           END AS grade_band
    FROM enrollments
)
SELECT grade_band, COUNT(*) AS n
FROM graded
GROUP BY grade_band
ORDER BY grade_band;
```

2. **`CASE WHEN score IS NULL` 这一支必须显式写**，否则 NULL 会落到 `ELSE` 分支被当成 F。因为 `NULL >= 90` 是 UNKNOWN，不是 TRUE，所有 `WHEN` 都不匹配，于是走 `ELSE`。**这是一个很容易忽略的 NULL 陷阱**（而 `ELSE` 不写的话，NULL 行的结果是 NULL，又会落到另一个"未出成绩"的组里，行为更隐蔽）。

### 2.4 `GROUP BY` 与 `NULL`

**所有 `NULL` 会被分到同一个组**：

```sql
SELECT city, COUNT(*) AS n
FROM students
GROUP BY city
ORDER BY city;
-- 结果里会有一行 city = NULL，n = 1（刘悦然）
```

这与 `UNIQUE` 约束对 NULL 的处理不同（`UNIQUE` 认为多个 NULL 不冲突），但两者并不矛盾：`GROUP BY` 需要一个确定的归组方式，而 `UNIQUE` 检查的是"两个值是否相等"（`NULL = NULL` 是 UNKNOWN，无法判定为重复）。详见 [01-basic-query.md](01-basic-query.md) 第 5.5 节。

## 3. `HAVING` 与 `WHERE` 的区别

**这是本篇最重要的概念。**

| | `WHERE` | `HAVING` |
| --- | --- | --- |
| 执行时机 | 在 `GROUP BY` **之前** | 在 `GROUP BY` **之后** |
| 过滤对象 | **行** | **分组（组）** |
| 能否用聚合函数 | ❌ 不能 | ✅ 能 |
| 能否用 `SELECT` 别名 | 通常不能 | **多数实现允许**（标准不允许） |
| 是否必须配合 `GROUP BY` | 不必 | 严格说需要（但有些数据库允许 `HAVING` 单独用，此时整个结果是一个组） |

**一句话记住**：**`WHERE` 过滤"哪些行参与分组"，`HAVING` 过滤"哪些分组留下来"。**

### 3.1 一个业务需求，两种写法，不同结果

**需求**：统计"2025-Spring 学期"里平均分高于 80 的课程。

```sql
-- ✅ 正确：WHERE 先筛学期（行级过滤），HAVING 再筛平均分（组级过滤）
SELECT course_id,
       COUNT(*)            AS n,
       ROUND(AVG(score), 2) AS avg_score
FROM enrollments
WHERE semester = '2025-Spring'          -- 只让这个学期的行参与分组
GROUP BY course_id
HAVING AVG(score) > 80                  -- 只保留平均分 > 80 的组
ORDER BY avg_score DESC;
```

```sql
-- ❌ 错误：把学期条件放到 HAVING 里
SELECT course_id,
       COUNT(*)            AS n,
       ROUND(AVG(score), 2) AS avg_score
FROM enrollments
GROUP BY course_id
HAVING AVG(score) > 80 AND semester = '2025-Spring';
-- 问题：semester 不是分组键，在 HAVING 里引用它违反了"只能引用分组键或聚合"的规则，
--       多数数据库会直接报错。
```

```sql
-- ⚠️ 更隐蔽的错误：把平均分的条件放到 WHERE 里
SELECT course_id, COUNT(*), ROUND(AVG(score), 2) AS avg_score
FROM enrollments
WHERE AVG(score) > 80                    -- ❌ 报错：聚合函数不能出现在 WHERE
GROUP BY course_id;
-- 这个会直接报错，所以不危险。真正危险的是"用行级条件代替组级条件"：
```

```sql
-- ❌ 看似合理但语义完全不同：筛出"单个成绩 > 80 的行"再求平均
SELECT course_id, COUNT(*), ROUND(AVG(score), 2) AS avg_score
FROM enrollments
WHERE score > 80                          -- 先丢掉所有 <= 80 的成绩！
GROUP BY course_id
HAVING AVG(score) > 80;
-- 结果：平均分一定 > 80（因为参与计算的行都 > 80），HAVING 成了废话。
-- 而且 COUNT(*) 变小了（只数了 > 80 的记录），完全不是"这门课的人数"。
```

**这个对比就是"WHERE 与 HAVING 不能互换"的实质**：`WHERE score > 80` 改变的是**参与统计的数据**，`HAVING AVG(score) > 80` 改变的是**哪些统计结果被输出**。**要问的是"我要过滤的是原始数据还是统计结论"。**

### 3.2 什么时候该用哪个

```text
问自己：这个条件是在说"哪些记录不算数"，还是"哪些结论不要显示"？

  "哪些记录不算数"  → WHERE   （例如：只看某个学期、只看有成绩的、排除测试数据）
  "哪些结论不要显示" → HAVING  （例如：平均分不够高的组不要、人数太少的组不要）

两者可以同时用，WHERE 先执行，HAVING 后执行。
```

```sql
-- 完整例：找出"2025-Spring 学期、至少 2 人选、平均分 ≥ 80"的课程
SELECT c.course_name,
       COUNT(*)             AS student_count,
       ROUND(AVG(e.score),2) AS avg_score
FROM enrollments e
JOIN courses c ON c.course_id = e.course_id
WHERE e.semester = '2025-Spring'        -- 行级：只算这个学期
  AND e.score IS NOT NULL               -- 行级：未出成绩的不算
GROUP BY c.course_id, c.course_name
HAVING COUNT(*) >= 2                    -- 组级：至少 2 人
   AND AVG(e.score) >= 80               -- 组级：平均分达标
ORDER BY avg_score DESC;
```

**注意这里 `WHERE e.score IS NOT NULL` 有意排除了未出成绩的记录**，让"平均分"只反映已出成绩的。如果业务上应该把未出成绩按 0 算，那就要改成 `SUM(COALESCE(e.score,0)) / COUNT(*)`（见 1.3 节）。**这是个业务决策，写查询前要想清楚。**

### 3.3 `HAVING` 里用 `SELECT` 别名

```sql
-- 多数数据库允许（PostgreSQL、MySQL 都可以）
SELECT course_id, AVG(score) AS avg_score
FROM enrollments
GROUP BY course_id
HAVING avg_score > 80;
```

**必须诚实说明**：**标准 SQL 不允许在 `HAVING` 里引用 `SELECT` 的别名**（因为逻辑上 `HAVING` 在 `SELECT` 之前执行）。但 PostgreSQL 和 MySQL 都做了扩展，允许这么写。Oracle 则**不允许**（会报 ORA-00904 invalid identifier），必须重复整个表达式。

**建议**：为了可移植性与清晰性，**在 `HAVING` 里重复表达式**：

```sql
SELECT course_id, AVG(score) AS avg_score
FROM enrollments
GROUP BY course_id
HAVING AVG(score) > 80;              -- ✅ 到处都能跑
```

## 4. 聚合与连接的组合

聚合常常需要先连接（因为要统计的字段和分组的字段在不同的表里）。

```sql
-- 每门课的选课人数与平均分，显示课程名与教师名
SELECT c.course_id,
       c.course_name,
       t.teacher_name,
       COUNT(e.enrollment_id)        AS student_count,   -- 注意：用 e 的列！
       ROUND(AVG(e.score), 2)        AS avg_score,
       MIN(e.score)                  AS min_score,
       MAX(e.score)                  AS max_score
FROM courses c
LEFT JOIN enrollments e ON e.course_id = c.course_id      -- LEFT：没选的课也要列出
LEFT JOIN teachers t    ON t.teacher_id = c.teacher_id
GROUP BY c.course_id, c.course_name, t.teacher_name
ORDER BY avg_score DESC NULLS LAST;
```

**这个查询里有三个值得注意的技巧**：

**技巧 1：`COUNT(e.enrollment_id)` 而不是 `COUNT(*)`**

这是一个关键的区分：

| 写法 | 结果 |
| --- | --- |
| `COUNT(*)` | 数**结果集的行数**。因为用了 `LEFT JOIN`，没人选的课（106）也会有一行（补的 NULL），所以 `COUNT(*)` 会算成 **1**——**这是错的！** |
| `COUNT(e.enrollment_id)` | 数 `enrollment_id` 非 NULL 的行数。没选的课那一行 `enrollment_id` 是 NULL，所以不算 → **0**，这是对的 |

```sql
-- 验证这个差别（106 号课没人选）
SELECT c.course_id,
       COUNT(*)                 AS count_star,       -- 1（错）
       COUNT(e.enrollment_id)   AS count_col         -- 0（对）
FROM courses c
LEFT JOIN enrollments e ON e.course_id = c.course_id
GROUP BY c.course_id
ORDER BY c.course_id;
```

**在 `LEFT JOIN` 之后做计数时，必须用右表的列做 `COUNT`，不能用 `COUNT(*)`。** 这是本篇最实用的一条经验。

**技巧 2：`GROUP BY` 里要包含所有非聚合的输出列**

```sql
GROUP BY c.course_id, c.course_name, t.teacher_name
```

因为 `course_name` 和 `teacher_name` 出现在 `SELECT` 里但不是聚合。在 PostgreSQL 上，如果 `course_id` 是 `courses` 的主键，只写 `GROUP BY c.course_id` 也是合法的（因为主键确定后其他列就函数依赖于它，PostgreSQL 支持这种"函数依赖"推断）。**但为了可移植性（以及让优化器更容易处理），显式列出所有非聚合列更稳妥。**

**技巧 3：`ORDER BY ... NULLS LAST`**

没人选的课 `avg_score` 是 `NULL`。`NULLS LAST` 让它们排到最后。**注意这是 PostgreSQL / Oracle 支持的语法，MySQL 与 SQLite 不支持**，跨数据库时要用 `ORDER BY avg_score IS NULL, avg_score DESC`（见 [01-basic-query.md](01-basic-query.md) 第 6.2 节）。

### 4.1 与聚合结果比较

**需求**：找出平均分高于"全体平均分"的课程。

```sql
-- ✅ 用标量子查询（推荐）
SELECT course_id,
       ROUND(AVG(score), 2) AS avg_score
FROM enrollments
WHERE score IS NOT NULL
GROUP BY course_id
HAVING AVG(score) > (SELECT AVG(score) FROM enrollments WHERE score IS NOT NULL)
ORDER BY avg_score DESC;
```

**为什么不能直接写 `HAVING AVG(score) > AVG(score)`**：这两个 `AVG` 在同一个分组上下文里，是同一个值，条件永远为假。**"分组内的平均"与"全体的平均"是两个不同层级的聚合**，必须用标量子查询（或窗口函数）把全体平均算出来。

```sql
-- ✅ 用窗口函数的写法（更现代，能同时显示两个值）
SELECT DISTINCT
       course_id,
       AVG(score) OVER (PARTITION BY course_id) AS course_avg,
       AVG(score) OVER ()                       AS overall_avg
FROM enrollments
WHERE score IS NOT NULL;
-- OVER () 不带 PARTITION BY 表示"整个结果集"，即全体平均
```

## 5. 窗口函数 (window function) —— 聚合的进阶

**窗口函数是"不折叠行的聚合"**。它按某种方式把行划分成窗口 (window)，对每个窗口计算，但**保留每一行**，把计算结果附在行上。

```sql
函数名(...) OVER (
    [PARTITION BY 分组列]
    [ORDER BY 排序列]
    [ROWS/RANGE 窗口范围]
)
```

### 5.1 与 `GROUP BY` 的根本区别

| | `GROUP BY` + 聚合 | 窗口函数 |
| --- | --- | --- |
| 输出行数 | **分组数**（行被折叠） | **原始行数**（行不折叠） |
| 能否同时看到明细与汇总 | ❌ 不能（要么分组要么不分） | ✅ **能**（明细行旁边就是汇总值） |
| 能否在 `WHERE` 里用 | 分组键可以 | ❌ 窗口函数在 `WHERE` 阶段还不可用 |

```sql
-- GROUP BY：每个课程一行
SELECT course_id, AVG(score) AS avg_score
FROM enrollments GROUP BY course_id;

-- 窗口函数：每个【选课记录】一行，旁边带上该课程的平均分
SELECT enrollment_id, student_id, course_id, score,
       ROUND(AVG(score) OVER (PARTITION BY course_id), 2) AS course_avg,
       ROUND(score - AVG(score) OVER (PARTITION BY course_id), 2) AS diff_from_avg
FROM enrollments
WHERE score IS NOT NULL;
```

**"行不折叠"带来的能力**：可以直接算"每条记录与它所在组的平均值的差距""每条记录在组内的排名""累计和"——这些用 `GROUP BY` 都做不到（除非再连回去）。

### 5.2 排名函数

| 函数 | 行为 | 示例（分数 95, 95, 88, 80） |
| --- | --- | --- |
| `ROW_NUMBER()` | **唯一**序号，同值也按出现顺序编号 | 1, 2, 3, 4 |
| `RANK()` | 同值同名次，**跳跃** | 1, 1, 3, 4 |
| `DENSE_RANK()` | 同值同名次，**不跳跃** | 1, 1, 2, 3 |
| `NTILE(n)` | 分成 n 个桶 | 按桶编号 |
| `PERCENT_RANK()` | 百分位排名 | 0, 0, 0.667, 1 |
| `CUME_DIST()` | 累积分布 | 0.5, 0.5, 0.75, 1 |

```sql
-- 每门课内按成绩排名
SELECT course_id, student_id, score,
       ROW_NUMBER() OVER (PARTITION BY course_id ORDER BY score DESC) AS rn,
       RANK()       OVER (PARTITION BY course_id ORDER BY score DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY course_id ORDER BY score DESC) AS dense_rnk
FROM enrollments
WHERE score IS NOT NULL
ORDER BY course_id, score DESC;
```

**`ROW_NUMBER` 与 `RANK` 的实用差别**：要"每门课取前 3 名，且第 3 名并列时都要"用 `RANK`/`DENSE_RANK`；要"每门课严格取 3 条"用 `ROW_NUMBER`。**这是"Top-N per group"问题的核心选择。**

### 5.3 取每组的前 N 名（经典模式）

**为什么需要包一层**：窗口函数在 `WHERE` 阶段还不存在（逻辑执行顺序上，窗口函数在 `SELECT` 与 `ORDER BY` 之间计算），所以**不能直接 `WHERE rn <= 3`**。必须先用子查询/CTE 算出 `rn`，在外层过滤：

```sql
-- 取每门课成绩前 3 的学生
WITH ranked AS (
    SELECT course_id, student_id, score,
           ROW_NUMBER() OVER (PARTITION BY course_id ORDER BY score DESC) AS rn
    FROM enrollments
    WHERE score IS NOT NULL
)
SELECT r.course_id, c.course_name, r.student_id, s.name, r.score, r.rn
FROM ranked r
JOIN courses  c ON c.course_id  = r.course_id
JOIN students s ON s.student_id = r.student_id
WHERE r.rn <= 3                       -- ✅ 在这里过滤是合法的
ORDER BY r.course_id, r.rn;
```

```sql
-- 用 RANK 则并列都会保留（可能超过 3 行）
WITH ranked AS (
    SELECT course_id, student_id, score,
           RANK() OVER (PARTITION BY course_id ORDER BY score DESC) AS rnk
    FROM enrollments
    WHERE score IS NOT NULL
)
SELECT * FROM ranked WHERE rnk <= 3;
```

### 5.4 其他常用窗口函数

```sql
-- 累计和 (running total)：注意 ORDER BY 会把窗口变成"从开头到当前行"
SELECT enrollment_id, course_id, score,
       SUM(score) OVER (PARTITION BY course_id ORDER BY enrollment_id) AS running_total
FROM enrollments
WHERE score IS NOT NULL;

-- 组内前后行的值 (LAG / LEAD)
SELECT enrollment_id, student_id, course_id, semester, score,
       LAG(score)  OVER (PARTITION BY student_id ORDER BY semester) AS prev_score,
       LEAD(score) OVER (PARTITION BY student_id ORDER BY semester) AS next_score,
       score - LAG(score) OVER (PARTITION BY student_id ORDER BY semester) AS improvement
FROM enrollments
WHERE score IS NOT NULL
ORDER BY student_id, semester;

-- 组内首尾值
SELECT DISTINCT course_id,
       FIRST_VALUE(score) OVER (PARTITION BY course_id ORDER BY score DESC) AS highest,
       LAST_VALUE(score)  OVER (PARTITION BY course_id ORDER BY score DESC
                                ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS lowest
FROM enrollments
WHERE score IS NOT NULL;
```

**`LAST_VALUE` 有一个著名的陷阱**：默认窗口框架是 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`，所以 `LAST_VALUE` 返回的是"当前行"而不是"组内最后一行"。**必须显式写 `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` 才能取到组内最后一个值。** 这是窗口函数里最容易踩的坑之一。

**方言说明**：**窗口函数需要 MySQL 8.0+ 或 PostgreSQL（或 SQLite 3.25+）**。MySQL 5.7 及更早版本**不支持窗口函数**——这是从旧 MySQL 迁移时最常见的不兼容点之一。

### 5.5 `ROLLUP` / `CUBE` 简介（生成小计与总计）

```sql
-- 按学期和课程统计，并自动加小计（每个学期一行）与总计（最后一行）
SELECT semester, course_id, COUNT(*) AS n, ROUND(AVG(score), 2) AS avg_score
FROM enrollments
GROUP BY ROLLUP (semester, course_id)
ORDER BY semester NULLS LAST, course_id NULLS LAST;
-- 结果里会出现 semester 为 NULL 的"总计行"，以及 course_id 为 NULL 的"学期小计行"
```

| 语法 | 生成 |
| --- | --- |
| `GROUP BY ROLLUP (a, b)` | 按 (a,b)、(a)、() 三个层次的小计 |
| `GROUP BY CUBE (a, b)` | 所有 2^2 = 4 种组合的小计（(a,b)、(a)、(b)、()） |
| `GROUP BY GROUPING SETS ((a,b), (a), ())` | 显式指定要哪些层次 |

**方言**：`ROLLUP` / `CUBE` / `GROUPING SETS` 在 PostgreSQL、Oracle、SQL Server、MySQL 8.0+ 上支持；**SQLite 不支持**。

**一个使用上的注意**：`ROLLUP` 用 `NULL` 表示"小计行"，这与数据里真实的 `NULL` 值**无法区分**。标准提供了 `GROUPING()` 函数来判断某个 `NULL` 是"小计"还是"真实值"：

```sql
SELECT semester,
       CASE WHEN GROUPING(semester) = 1 THEN '【全部学期】' ELSE semester END AS semester_label,
       COUNT(*) AS n
FROM enrollments
GROUP BY ROLLUP (semester);
```

## 6. 常见错误清单

| 错误 | 问题 | 修正 |
| --- | --- | --- |
| `COUNT(col)` 却期望数所有行 | `COUNT(col)` 跳过 NULL | 用 `COUNT(*)` |
| `COUNT(col IS NULL)` 想数 NULL 个数 | 表达式永不返回 NULL，等于数所有行 | `COUNT(*) - COUNT(col)` 或 `SUM(CASE WHEN col IS NULL THEN 1 ELSE 0 END)` |
| `AVG(col)` 却期望把 NULL 当 0 | 分母是非 NULL 行数 | `SUM(COALESCE(col,0)) / COUNT(*)` |
| 直接使用 `SUM`/`AVG`/`MAX` 的结果 | 无行时返回 `NULL` 而不是 0 | `COALESCE(SUM(x), 0)` |
| `WHERE AVG(col) > x` | 聚合函数不能出现在 `WHERE`（会报错） | 用 `HAVING` |
| `WHERE score > 80` 代替 `HAVING AVG(score) > 80` | 改变了参与统计的数据，语义完全不同 | 分清"过滤行"与"过滤组" |
| `SELECT` 里出现非分组、非聚合的列 | 数据库无法决定显示哪一个值 | 加进 `GROUP BY`，或改用聚合/窗口函数 |
| `LEFT JOIN` 后用 `COUNT(*)` 计数 | 没匹配的行也被数成 1 | 用 `COUNT(右表列)` |
| `CASE WHEN` 忘了处理 `NULL` | NULL 落到 `ELSE` 分支 | 显式加 `WHEN col IS NULL` 分支 |
| 在 `WHERE` 里用窗口函数结果 | 窗口函数在 `WHERE` 阶段还不存在 | 用 CTE/子查询包一层 |
| `LAST_VALUE` 直接取组内最后一个值 | 默认窗口框架只到当前行 | 显式写 `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| 依赖 `GROUP BY` 的隐含排序 | 优化器可能改变输出顺序 | 显式 `ORDER BY` |
| 在 MySQL 5.7 上用窗口函数 | 不支持 | 升级到 8.0+，或用相关子查询/自连接模拟 |
| 依赖 MySQL 允许 `SELECT` 非分组列 | 结果不确定，且新版本默认禁止 | 按标准写，把所有非聚合列放进 `GROUP BY` |

## 小结

- **聚合函数对 NULL 的行为必须记牢**：`COUNT(*)` 数所有行；`COUNT(col)` / `COUNT(DISTINCT col)` / `SUM` / `AVG` / `MIN` / `MAX` 都**跳过 NULL**。
- **`SUM`/`AVG`/`MIN`/`MAX` 在没有任何行时返回 `NULL`**（而 `COUNT` 返回 `0`），所以要习惯用 `COALESCE(SUM(x), 0)`。
- `COUNT(col IS NULL)` 是错的（表达式永不返回 NULL，等于数所有行）；数 NULL 用 `COUNT(*) - COUNT(col)` 或 `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`。
- **`AVG(col) = SUM(col) / COUNT(col)`，不等于 `SUM(col) / COUNT(*)`**。把 NULL 当作 0 参与计算是**业务决策**，要显式写成 `SUM(COALESCE(col,0)) / COUNT(*)`。
- **`WHERE` 过滤行（分组前），`HAVING` 过滤组（分组后）**。`WHERE` 里不能用聚合函数；`HAVING` 里能用。**判断标准是"我要过滤的是原始数据还是统计结论"。**
- `SELECT` 里只能出现**分组键**和**聚合函数**。想要"组内某一行的其他列"，`GROUP BY` 做不到，要用窗口函数或再连接回去。
- `CASE WHEN` 分档时必须**显式处理 `NULL`**，因为 `NULL >= 90` 是 UNKNOWN，会掉进 `ELSE`。
- `GROUP BY` 把所有 NULL 分到**同一个组**（与 `UNIQUE` 认为多个 NULL 不冲突并不矛盾）。
- **`LEFT JOIN` 之后计数必须用 `COUNT(右表列)` 而不是 `COUNT(*)`**，否则"没有关联记录"的行会被数成 1。
- 按表达式分组时，标准 SQL 要求在 `GROUP BY` 里重复整个表达式；**更可读的做法是用 CTE 先算出那一列再分组**。
- **窗口函数是"不折叠行的聚合"**，能同时看到明细与汇总，用于组内排名、累计和、前后行比较。
- **窗口函数不能在 `WHERE` 里用**（它在逻辑顺序上晚于 `WHERE`），取"每组前 N"必须用 CTE/子查询包一层再过滤。
- `ROW_NUMBER` 给唯一序号、`RANK` 并列跳跃、`DENSE_RANK` 并列不跳跃——**"严格取 N 条"用 `ROW_NUMBER`，"并列都算"用 `RANK`**。
- **`LAST_VALUE` 的默认窗口框架只到当前行**，要取组内最后一行必须显式 `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`。
- 窗口函数需要 **MySQL 8.0+ / PostgreSQL / SQLite 3.25+**；MySQL 5.7 不支持。
- `ROLLUP` / `CUBE` 生成小计与总计行，用 `NULL` 表示小计，需用 `GROUPING()` 与真实 NULL 区分；**SQLite 不支持**。
- `COALESCE` 与 `NULLIF` 是处理 NULL 的两个主力：前者提供默认值，后者常用来防除零（`NULLIF(COUNT(*), 0)`）。
