# 子查询与公共表表达式 (CTE)

本篇梳理子查询按位置与返回形态的分类、`IN` 与 `EXISTS` 的语义差别（以及 `NULL` 带来的影响）、派生表，以及 `WITH` 与递归 CTE 的用法。

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

```sql
INSERT INTO teachers (teacher_id, teacher_name, department) VALUES
    (1, '黄志强', '计算机系'),
    (2, '林美玲', '计算机系'),
    (3, '吴建国', '数学系'),
    (4, '郑晓月', '软件工程系');

INSERT INTO students (student_id, name, gender, major, city, enroll_year) VALUES
    (1, '王小明', 'M', '计算机科学与技术', '厦门', 2022),
    (2, '李雨桐', 'F', '计算机科学与技术', '北京', 2022),
    (3, '张浩然', 'M', '软件工程',         '厦门', 2021),
    (4, '陈思远', 'M', '计算机科学与技术', '上海', 2023),
    (5, '刘悦然', 'F', '数据科学',         NULL,   2022),
    (6, '赵子涵', 'F', '软件工程',         '厦门', 2023);

INSERT INTO courses (course_id, course_name, credits, teacher_id) VALUES
    (101, '程序设计基础',   4, 1),
    (102, '数据结构',       4, 1),
    (103, '数据库系统原理', 3, 2),
    (104, '操作系统',       4, 2),
    (105, '离散数学',       3, 3),
    (106, '软件工程导论',   3, NULL);

INSERT INTO enrollments (enrollment_id, student_id, course_id, score, semester) VALUES
    (1,  1, 101, 88.5, '2024-Fall'),
    (2,  1, 102, 92.0, '2024-Fall'),
    (3,  1, 103, 79.0, '2025-Spring'),
    (4,  2, 101, 95.5, '2024-Fall'),
    (5,  2, 103, NULL, '2025-Spring'),
    (6,  3, 101, 72.0, '2024-Fall'),
    (7,  3, 105, 85.0, '2024-Fall'),
    (8,  4, 102, 91.0, '2025-Spring'),
    (9,  4, 104, 67.5, '2025-Spring'),
    (10, 5, 103, 88.0, '2025-Spring'),
    (11, 5, 104, 0,    '2025-Spring'),
    (12, 6, 101, 59.0, '2024-Fall');
```

## 1. 子查询的两个分类维度

### 1.1 按位置分

| 位置 | 形式 | 说明 |
| --- | --- | --- |
| `WHERE` / `HAVING` 中 | `... WHERE col IN (SELECT ...)` | 最常见，用于筛选 |
| `FROM` 中（**派生表**） | `... FROM (SELECT ...) AS t` | 把子查询的结果当作一张临时表 |
| `SELECT` 列表中 | `SELECT col, (SELECT ...) AS x` | **标量子查询**，每行算出一个值 |
| `JOIN` 的右侧 | `... JOIN (SELECT ...) AS t ON ...` | 等价于派生表 |

### 1.2 按返回形态分（这个维度更重要）

| 形态 | 返回什么 | 可以用在哪 |
| --- | --- | --- |
| **标量 (scalar)** | 1 行 1 列 | 任何需要"一个值"的地方（`SELECT` 列表、`WHERE col >`、`=`） |
| **行 (row)** | 1 行多列 | 行值比较：`WHERE (a, b) = (SELECT x, y FROM ...)` |
| **列 / 集合 (column)** | 多行 1 列 | `IN`、`ANY`、`ALL`、`EXISTS` |
| **表 (table)** | 多行多列 | `FROM` 中的派生表、`JOIN` |

**标量子查询返回多行会报错**：

```sql
-- ❌ 如果 enrollments 里有多条 student_id = 1 的记录，这里会报错
SELECT name, (SELECT score FROM enrollments WHERE student_id = 1) AS s
FROM students;
-- PostgreSQL: "more than one row returned by a subquery used as an expression"
-- MySQL:      "Subquery returns more than 1 row"

-- ✅ 加上聚合或 LIMIT 让它只返回一行一列
SELECT name, (SELECT AVG(score) FROM enrollments WHERE student_id = s.student_id) AS avg_score
FROM students s;
-- 聚合函数天然只返回一行，所以不会报错

-- 或者显式取一行
SELECT name, (SELECT score FROM enrollments
              WHERE student_id = s.student_id ORDER BY score DESC LIMIT 1) AS best_score
FROM students s;
```

**这个错误在实践中很常见**：写标量子查询时忘了它必须只返回一个值。**加 `LIMIT 1` 是"我知道可能有多个、我只要一个"的显式声明**，比默默地用聚合更清晰（如果业务上确实需要"任意一个"，写 `LIMIT 1` 加上 `ORDER BY` 明确"哪一个"更好）。

## 2. `IN` 与 `NOT IN`

```sql
-- 选了 101 号课的学生
SELECT student_id, name FROM students
WHERE student_id IN (SELECT student_id FROM enrollments WHERE course_id = 101);

-- 没选 101 号课的学生（⚠️ 见下面的 NULL 陷阱）
SELECT student_id, name FROM students
WHERE student_id NOT IN (SELECT student_id FROM enrollments WHERE course_id = 101);
```

### 2.1 `NOT IN` 的 NULL 陷阱

**规则**：如果子查询返回的结果集中**包含任何 `NULL`**，`NOT IN` 会返回**空结果集**。

```sql
-- ⚠️ 这个查询会返回空结果（如果 enrollments.student_id 里有 NULL）
SELECT * FROM students
WHERE student_id NOT IN (SELECT student_id FROM enrollments);
```

**为什么**：`NOT IN` 语义上等价于一串 `<>` 加 `AND`：

```text
x NOT IN (a, b, c)   等价于   x <> a AND x <> b AND x <> c
```

若 `a` 是 `NULL`，则 `x <> NULL` 是 **UNKNOWN**。而 `TRUE AND UNKNOWN` = `UNKNOWN`，`FALSE AND UNKNOWN` = `FALSE`——**无论 `x` 是什么，整个合取都不可能是 `TRUE`**。由于 `WHERE` 只保留 `TRUE` 的行，结果必然为空。

更直观地看真值表：

| `x` | 子查询里的 `a` | `x <> a` | 与其他条件 AND 之后 |
| --- | --- | --- | --- |
| 5 | 1 | TRUE | 取决于其他项 |
| 5 | **NULL** | **UNKNOWN** | **不可能为 TRUE** |

**三种修正**：

```sql
-- 修正 A：在子查询里排除 NULL
SELECT * FROM students
WHERE student_id NOT IN (
    SELECT student_id FROM enrollments WHERE student_id IS NOT NULL
);

-- 修正 B（推荐）：用 NOT EXISTS
SELECT * FROM students s
WHERE NOT EXISTS (SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id);

-- 修正 C：用 LEFT JOIN 反连接
SELECT s.* FROM students s
LEFT JOIN enrollments e ON e.student_id = s.student_id
WHERE e.enrollment_id IS NULL;
```

**为什么 `EXISTS` 不受影响**：`EXISTS` 判断的是"子查询是否返回了至少一行"，这是一个**存在性判断**（真/假二值），不涉及与 `NULL` 的比较，所以不存在 UNKNOWN 问题。

**实践结论：需要"排除"语义时，默认用 `NOT EXISTS`。** 只有当子查询结果确定不含 NULL（例如选了主键列、或者显式加了 `IS NOT NULL`）时，才考虑用 `NOT IN`。

### 2.2 关于 `IN` 遇到 NULL 的行为（作为对比）

`IN` 与 `NOT IN` 不对称：

```sql
SELECT 1 IN (1, 2, NULL);        -- TRUE   （找到 1，立刻为真）
SELECT 3 IN (1, 2, NULL);        -- UNKNOWN（没找到，但因为列表里有 NULL，无法断定"不存在"）
SELECT 3 NOT IN (1, 2, NULL);    -- UNKNOWN（同上，所以 WHERE 把它过滤掉）

-- 在 WHERE 里的实际效果：
SELECT * FROM students WHERE student_id IN (1, 2, NULL);
-- → student_id = 1 和 2 的行会被返回（因为找到即 TRUE）
-- → 其他行返回 UNKNOWN，被过滤
--   所以 IN 里带 NULL 通常【不影响正常结果】，只是多了一个无意义的 NULL 项

SELECT * FROM students WHERE student_id NOT IN (1, 2, NULL);
-- → 空结果集！因为没有任何行能让条件变成 TRUE
```

**这就是不对称的地方**：`IN` 里带 `NULL` 通常无害（因为匹配到就返回 TRUE），但 `NOT IN` 里带 `NULL` 是**灾难性的**（结果恒为空）。**这个不对称正是 `NOT IN` 陷阱难以察觉的原因。**

### 2.3 集合运算的替代方案

"在 A 中但不在 B 中"这个需求也可以用集合运算表达：

```sql
-- PostgreSQL / SQLite / 多数数据库支持 EXCEPT（标准叫法）
SELECT student_id FROM students
EXCEPT
SELECT student_id FROM enrollments
WHERE student_id IS NOT NULL;

-- MySQL 用 NOT IN / NOT EXISTS（MySQL 8.0.31+ 才开始支持 EXCEPT）
-- Oracle 用 MINUS
```

| 运算 | 标准名 | MySQL | Oracle | 说明 |
| --- | --- | --- | --- | --- |
| 差集 | `EXCEPT` | 8.0.31+ 支持 | `MINUS` | A 中有而 B 中没有（**会自动去重**） |
| 交集 | `INTERSECT` | 8.0.31+ 支持 | 支持 | 两边都有（自动去重） |
| 并集 | `UNION` | 支持 | 支持 | `UNION` 去重，`UNION ALL` 不去重 |

**注意 `EXCEPT` 会自动去重**，所以它和 `NOT EXISTS` 的语义不完全一样（如果 A 有重复行，`EXCEPT` 只保留一份）。需要保留重复时用 `NOT EXISTS`。

## 3. `EXISTS` 与 `NOT EXISTS`

### 3.1 语义

`EXISTS (子查询)` 返回一个布尔值：子查询是否返回了**至少一行**。

关键特性：**一旦找到第一行就短路 (short-circuit)，不再继续找。** 因此：

```sql
-- ✅ 惯例是写 SELECT 1，因为返回的内容根本不重要
SELECT * FROM students s
WHERE EXISTS (SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id);

-- 下面这些写法结果完全相同（性能通常也相同）：
--   EXISTS (SELECT 1 ...)
--   EXISTS (SELECT * ...)
--   EXISTS (SELECT NULL ...)
```

`EXISTS (SELECT *)` 不会真的去取所有列——优化器知道只需要判断存在性。所以 `SELECT 1` 是**表达意图**的选择，不是性能优化。

### 3.2 相关子查询 (correlated subquery)

上面例子里的 `WHERE e.student_id = s.student_id` 引用了外层的 `s.student_id`，这类子查询叫**相关子查询**。它的特点是：

- **对外层的每一行执行一次**（逻辑上如此；优化器通常会改写成连接来避免重复执行）；
- 不能独立运行（必须先有外层的行）。

```sql
-- 相关子查询：每个学生的选课数量
SELECT s.student_id, s.name,
       (SELECT COUNT(*) FROM enrollments e WHERE e.student_id = s.student_id) AS course_count
FROM students s;
```

**注意这里用了 `COUNT(*)`**（在标量子查询里是合法的），它会返回 `0` 而不是 `NULL`——所以"没选课的学生"这一行会显示 `0` 而不是 `NULL`。**这是 `COUNT` 相对 `SUM` 的一个便利之处。**

### 3.3 `EXISTS` 与 `IN` 的对比

| 维度 | `IN` | `EXISTS` |
| --- | --- | --- |
| 子查询返回 | 一列多行 | 任意（只看有没有行） |
| `NULL` 安全 | `IN` 基本安全（匹配到就 TRUE）；**`NOT IN` 不安全** | ✅ 两个方向都安全 |
| 短路 | 取决于优化器 | 语义上"找到即止"，通常更快 |
| 是否相关 | 通常不相关 | 通常是相关子查询 |
| 可读性 | "值在集合里"更直观 | "是否存在"更直观 |
| 优化器处理 | 现代优化器常改写成半连接 (semi-join) | 常改写成半连接 |

**实践建议**：
- `IN` 的子查询返回**主键或其他非空列**时，两者等价，选可读性好的；
- 子查询可能返回 `NULL` 时，**永远用 `EXISTS` / `NOT EXISTS`**；
- `NOT IN` 几乎总是应该被 `NOT EXISTS` 替代。

### 3.4 一个完整的对比示例

```sql
-- 需求：找出所有成绩都及格（>= 60）的学生

-- 写法 A：NOT EXISTS —— 找"存在不及格记录"的反面
SELECT s.student_id, s.name
FROM students s
WHERE NOT EXISTS (
    SELECT 1 FROM enrollments e
    WHERE e.student_id = s.student_id
      AND e.score IS NOT NULL
      AND e.score < 60
);
-- ✅ 正确：学生 6（59.0 分）被排除，其他人都留下
--    注意"没选课的学生"也会被保留（因为没有不及格记录）——这是否符合业务需要要想清楚

-- 写法 B：NOT IN —— 危险
SELECT s.student_id, s.name
FROM students s
WHERE s.student_id NOT IN (
    SELECT e.student_id FROM enrollments e WHERE e.score < 60
);
-- 这里恰好安全，因为 score < 60 的条件已经排除了 score 为 NULL 的行，
-- 但如果去掉那个条件、直接 SELECT student_id FROM enrollments，
-- 一旦有 student_id 为 NULL 的行，结果就是空集。
```

**"成绩都及格"的语义陷阱**：写法 A 会把**没选任何课的学生**也算进"成绩都及格"（因为对"所有成绩"这个空集，命题是真空为真 (vacuously true)）。**这在业务上是否正确取决于需求**——"没有不及格记录"和"选过课且都及格"是两件事。

```sql
-- 如果要求"选过课、且都及格"
SELECT s.student_id, s.name
FROM students s
WHERE EXISTS (SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id)   -- 选过课
  AND NOT EXISTS (
      SELECT 1 FROM enrollments e
      WHERE e.student_id = s.student_id AND e.score IS NOT NULL AND e.score < 60
  );
```

**这类"空集为真"的边界是写业务查询时最容易忽略的地方**，值得养成习惯去问一句"如果这组数据一行都没有，我的查询会返回什么"。

## 4. 比较运算符 + 子查询：`ALL` 与 `ANY`

```sql
-- 标量比较
SELECT * FROM enrollments WHERE score > (SELECT AVG(score) FROM enrollments);

-- ANY / SOME：只要满足【任意一个】即可（= ANY 等价于 IN）
SELECT * FROM enrollments WHERE score > ANY (SELECT score FROM enrollments WHERE course_id = 101);
-- 含义：分数大于 101 号课的【至少一个】成绩

-- ALL：必须满足【所有】才行
SELECT * FROM enrollments WHERE score > ALL (SELECT score FROM enrollments WHERE course_id = 101);
-- 含义：分数大于 101 号课的【每一个】成绩（即大于该课最高分）
```

| 写法 | 等价于 | 记忆 |
| --- | --- | --- |
| `= ANY (子查询)` | `IN (子查询)` | 有一个相等 |
| `<> ALL (子查询)` | `NOT IN (子查询)` | 与所有都不等 |
| `> ALL (子查询)` | `> MAX(子查询)` | 比最大的还大 |
| `> ANY (子查询)` | `> MIN(子查询)` | 比最小的大就行 |

**`SOME` 是 `ANY` 的同义词**（标准 SQL 里两者等价，`ANY` 更常用）。

**方言提醒**：`ANY` / `ALL` 支持较好，但可读性普遍不如等价的聚合写法：

```sql
-- 更清楚的写法（推荐）
WHERE score > (SELECT MAX(score) FROM enrollments WHERE course_id = 101)
-- 而不是
WHERE score > ALL (SELECT score FROM enrollments WHERE course_id = 101)
```

**关键点**：`> ALL` 与 `> MAX` 在**子查询为空时行为不同**。`> ALL (空集)` 恒为 `TRUE`（真空为真），而 `> MAX(空集)` 是 `> NULL`，结果是 `UNKNOWN`，被过滤掉。**这是又一个"空集"语义差别的例子**，用聚合写法需要额外注意：

```sql
-- 用聚合写法时要显式处理空集
WHERE score > (SELECT COALESCE(MAX(score), -1) FROM enrollments WHERE course_id = 999)
```

## 5. 派生表 (derived table)

`FROM` 里的子查询叫派生表，**必须起别名**。

```sql
SELECT t.major, t.student_count
FROM (
    SELECT major, COUNT(*) AS student_count
    FROM students
    GROUP BY major
) AS t                                 -- ← 别名必须写
WHERE t.student_count >= 2
ORDER BY t.student_count DESC;
```

**关于别名是否强制**：

| 数据库 | 是否要求别名 |
| --- | --- |
| PostgreSQL | **要求**（`subquery in FROM must have an alias`） |
| MySQL | **要求**（`Every derived table must have its own alias`） |
| SQLite | 不强制，但强烈建议写 |
| Oracle | 不强制（历史上如此） |

**结论：永远写别名**，这样到处都能跑。

### 5.1 为什么要用派生表

派生表的用途是**"分两步算"**：第一步算出中间结果，第二步对它筛选/排序/再聚合。

```sql
-- 需求：找出平均分高于 80 的课程，并显示课程名
-- 不能写 HAVING AVG(score) > 80 之后又去 JOIN courses 吗？可以，但派生表在下面这种场景更必要：

-- 需求：每门课的平均分排名，取前 3 名
SELECT course_id, avg_score
FROM (
    SELECT course_id, AVG(score) AS avg_score
    FROM enrollments
    WHERE score IS NOT NULL
    GROUP BY course_id
) AS course_avg
ORDER BY avg_score DESC
LIMIT 3;
-- 这里也可以不用派生表（直接 GROUP BY + ORDER BY + LIMIT），
-- 但一旦需要"对聚合结果再做聚合"或"对窗口函数结果过滤"，派生表就不可避免。
```

**派生表不可替代的场景**：

```sql
-- 对聚合结果再聚合：平均的"各课平均分"的均值
SELECT AVG(avg_score) AS avg_of_avgs
FROM (
    SELECT course_id, AVG(score) AS avg_score
    FROM enrollments WHERE score IS NOT NULL
    GROUP BY course_id
) AS t;

-- 对窗口函数结果过滤（取每组前 N）
SELECT * FROM (
    SELECT enrollment_id, course_id, score,
           ROW_NUMBER() OVER (PARTITION BY course_id ORDER BY score DESC) AS rn
    FROM enrollments WHERE score IS NOT NULL
) AS ranked
WHERE rn <= 2;
```

### 5.2 派生表的局限

**同一个派生表在同一层不能被引用两次**：

```sql
-- ❌ 想用 t 两次（比较 t 自己的行），做不到
SELECT a.x - b.x
FROM (SELECT ... ) AS t AS a
JOIN (SELECT ... ) AS t AS b ON ...;     -- 必须把子查询写两遍！
```

```sql
-- ✅ 用 CTE 解决：定义一次，可以引用多次
WITH t AS (SELECT ...)
SELECT a.x - b.x
FROM t AS a JOIN t AS b ON ...;
```

**这正是 CTE 相对于派生表最重要的优势**：**可复用**（以及可读性）。见下一节。

## 6. CTE：`WITH` 公共表表达式

**CTE (Common Table Expression)** 用 `WITH` 子句定义一个命名的临时结果集，在主查询里可以像表一样引用。

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT ... FROM cte_name WHERE ...;
```

### 6.1 基本用法

```sql
-- 每门课的平均分，只看平均分 >= 80 的
WITH course_stats AS (
    SELECT course_id,
           COUNT(*)             AS student_count,
           AVG(score)           AS avg_score
    FROM enrollments
    WHERE score IS NOT NULL
    GROUP BY course_id
)
SELECT c.course_name,
       cs.student_count,
       ROUND(cs.avg_score, 2) AS avg_score
FROM course_stats cs
JOIN courses c ON c.course_id = cs.course_id
WHERE cs.avg_score >= 80
ORDER BY cs.avg_score DESC;
```

### 6.2 多个 CTE

用逗号分隔，**后面的 CTE 可以引用前面的 CTE**（顺序有意义）：

```sql
WITH
-- 第 1 步：每个学生的平均分
student_avg AS (
    SELECT student_id, AVG(score) AS avg_score
    FROM enrollments
    WHERE score IS NOT NULL
    GROUP BY student_id
),
-- 第 2 步：全体平均分（引用不到 student_avg，但它自己是个独立聚合）
overall AS (
    SELECT AVG(avg_score) AS overall_avg FROM student_avg    -- ← 引用前面的 CTE
),
-- 第 3 步：找出高于全体平均的学生
above_avg AS (
    SELECT sa.student_id, sa.avg_score
    FROM student_avg sa
    CROSS JOIN overall o
    WHERE sa.avg_score > o.overall_avg
)
SELECT s.student_id, s.name, s.major, ROUND(aa.avg_score, 2) AS avg_score
FROM above_avg aa
JOIN students s ON s.student_id = aa.student_id
ORDER BY aa.avg_score DESC;
```

**这个查询的结构就是 CTE 的核心价值**：它把"分三步算"的过程**从上往下**写出来，每一步都有名字。读者不用像读嵌套子查询那样"从最里面往外读"。

### 6.3 嵌套子查询 vs CTE 的并排对比

**需求**：找出"平均分高于全体平均分"的学生，显示姓名、专业、平均分，按平均分降序。

**写法 A：嵌套子查询**（从内往外读，很累）

```sql
SELECT s.student_id, s.name, s.major, ROUND(t.avg_score, 2) AS avg_score
FROM students s
JOIN (
    SELECT student_id, avg_score
    FROM (
        SELECT student_id, AVG(score) AS avg_score
        FROM enrollments
        WHERE score IS NOT NULL
        GROUP BY student_id
    ) AS per_student
    WHERE avg_score > (
        SELECT AVG(avg_score)
        FROM (
            SELECT student_id, AVG(score) AS avg_score
            FROM enrollments
            WHERE score IS NOT NULL
            GROUP BY student_id
        ) AS inner_avg
    )
) AS t ON t.student_id = s.student_id
ORDER BY t.avg_score DESC;
```

注意这里最糟糕的地方：**计算"每个学生平均分"的那段逻辑被写了两遍**。一旦要改口径（例如把 NULL 按 0 算），两处都得改，很容易漏一处。

**写法 B：CTE**（从上往下读，逻辑只写一次）

```sql
WITH per_student AS (
    SELECT student_id, AVG(score) AS avg_score
    FROM enrollments
    WHERE score IS NOT NULL
    GROUP BY student_id
),
overall AS (
    SELECT AVG(avg_score) AS overall_avg FROM per_student
)
SELECT s.student_id, s.name, s.major, ROUND(ps.avg_score, 2) AS avg_score
FROM per_student ps
JOIN overall o
  ON ps.avg_score > o.overall_avg         -- 用 ON 做过滤（也可以放 WHERE）
JOIN students s ON s.student_id = ps.student_id
ORDER BY ps.avg_score DESC;
```

**对比结论**：

| | 嵌套子查询 | CTE |
| --- | --- | --- |
| 阅读方向 | 从内往外（反直觉） | 从上往下（符合"分步思考"） |
| 逻辑复用 | ❌ 要重复写 | ✅ 定义一次，可引用多次 |
| 命名 | 只有 `AS t` 这种无意义别名 | 每步都有业务含义的名字 |
| 调试 | 难以单独运行中间步骤 | **可以把中间 CTE 单独拿出来跑** |
| 适用场景 | 简单的一层嵌套 | 三步以上的逻辑 |

**调试这一点是 CTE 的隐性优势**：排查一个复杂 CTE 时，你可以把 `WITH x AS (...)` 里的某一段单独复制出来 `SELECT * FROM x LIMIT 10`，立刻看到中间结果。嵌套子查询做不到这么方便。

### 6.4 CTE 与派生表的对比

| 维度 | 派生表 `FROM (SELECT ...) AS t` | CTE `WITH t AS (...)` |
| --- | --- | --- |
| 位置 | 嵌在查询里 | 定义在查询**前面** |
| 能否多次引用 | ❌ 不能 | ✅ 能 |
| 可读性 | 复杂时层层缩进 | 扁平、从上往下 |
| 能否被多个语句共用 | ❌ | ⚠️ 单个语句内可以；跨语句需要临时表 |
| 语法支持 | 极广 | 主流数据库都支持（MySQL 8.0+、PostgreSQL、SQLite、SQL Server、Oracle） |

**`WITH` 的方言提醒**：**MySQL 8.0 之前不支持 CTE**（8.0 才引入）。这是从旧 MySQL 迁移时的另一个常见不兼容点（窗口函数也是 8.0 才有的）。PostgreSQL 从 8.4 起支持，SQLite 从 3.8.3 起支持。

### 6.5 物化 (materialization) 与内联 (inlining)

一个会影响性能的细节：**数据库可能把 CTE 的结果"物化"（算出来存成一个临时结果集），也可能把它"内联"（把定义展开到使用处）**。

| 策略 | 什么情况下好 |
| --- | --- |
| 物化 | CTE 被引用多次时（只算一次），或作为优化边界阻止优化器做出错误决策 |
| 内联 | CTE 只被引用一次时（可以和其他条件下推合并，减少中间结果） |

**不同数据库、不同版本对这个策略的选择不同**，而且可能受查询结构影响。**PostgreSQL 12 改变了 CTE 的默认处理方式**（在此之前的版本倾向于物化，12 起对"只引用一次且无副作用"的 CTE 倾向内联），这也说明了版本差异是真实存在的。

**操作建议**：如果一个 CTE 查询很慢，不要猜，用 `EXPLAIN ANALYZE` 看实际执行计划，确认 CTE 是被物化还是被内联了、实际行数是否符合预期。部分数据库也提供显式的提示（例如 PostgreSQL 的 `MATERIALIZED` / `NOT MATERIALIZED` 关键字）：

```sql
-- PostgreSQL：显式要求物化 / 内联（这是方言扩展）
WITH t AS MATERIALIZED (SELECT ...) SELECT ...;
WITH t AS NOT MATERIALIZED (SELECT ...) SELECT ...;
```

## 7. 递归 CTE

递归 CTE 用于处理**层级 (hierarchical) 或图 (graph)** 结构的数据：组织架构、分类树、物料清单 (BOM)、路径查找。

### 7.1 语法结构

```sql
WITH RECURSIVE cte_name AS (
    -- ① 锚点成员 (anchor member)：不引用自己的起始查询
    SELECT ...

    UNION ALL        -- 或 UNION（去重）

    -- ② 递归成员 (recursive member)：引用 cte_name 自身
    SELECT ... FROM cte_name WHERE <终止条件>
)
SELECT * FROM cte_name;
```

**三个组成部分缺一不可**：
1. **锚点 (anchor)**：定义起始行（递归的"第 0 层"）；
2. **`UNION ALL`**：把锚点结果与递归结果拼起来；
3. **递归成员**：引用 CTE 自己，并且**必须有能让递归终止的条件**。

**注意 `RECURSIVE` 关键字**：标准 SQL 要求写 `WITH RECURSIVE`。PostgreSQL、SQLite、MySQL 8.0+ 都要求这个关键字；**SQL Server 与 Oracle 不写 `RECURSIVE`**（直接用 `WITH`）。**这是方言差异，需要注意。**

### 7.2 例 1：生成 1 到 10 的数字序列

```sql
WITH RECURSIVE nums AS (
    -- 锚点：从 1 开始
    SELECT 1 AS n

    UNION ALL

    -- 递归成员：每次加 1，直到 n = 10
    SELECT n + 1 FROM nums WHERE n < 10
)
SELECT n FROM nums ORDER BY n;
-- 输出：1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

**逐步执行过程**：

| 迭代 | 产生什么 | 说明 |
| --- | --- | --- |
| 锚点 | `{1}` | 起始 |
| 第 1 次递归 | `{2}` | 对工作集 `{1}` 应用递归成员 |
| 第 2 次递归 | `{3}` | 对 `{2}` 应用 |
| … | … | |
| 第 9 次递归 | `{10}` | 对 `{9}` 应用 |
| 第 10 次递归 | `{}`（空） | `WHERE n < 10` 不满足，递归结束 |

**终止条件是 `WHERE n < 10`**。如果写成 `WHERE n < 10` 之外的条件（或忘了写），递归永远不会结束。

**数字序列的实际用途**：生成日期序列补齐报表空缺、生成测试数据、按序号拆分字符串等。

```sql
-- 生成最近 7 天的日期序列（PostgreSQL 语法）
WITH RECURSIVE dates AS (
    SELECT CURRENT_DATE - INTERVAL '6 days' AS d
    UNION ALL
    SELECT d + INTERVAL '1 day' FROM dates WHERE d < CURRENT_DATE
)
SELECT d::date FROM dates ORDER BY d;
```

### 7.3 例 2：组织架构的自顶向下遍历

```sql
CREATE TABLE employees (
    emp_id     INTEGER PRIMARY KEY,
    emp_name   VARCHAR(50) NOT NULL,
    manager_id INTEGER,                       -- 指向同一张表
    department VARCHAR(50)
);

INSERT INTO employees VALUES
    (1, '张伟',   NULL, '管理层'),
    (2, '李强',   1,    '技术部'),
    (3, '王芳',   1,    '财务部'),
    (4, '刘洋',   2,    '技术部'),
    (5, '陈静',   2,    '技术部'),
    (6, '赵磊',   4,    '技术部'),
    (7, '孙娜',   5,    '技术部'),
    (8, '周涛',   3,    '财务部');
```

```sql
WITH RECURSIVE org AS (
    -- ① 锚点：从根节点（没有上级的人）开始
    SELECT emp_id,
           emp_name,
           manager_id,
           0            AS level,          -- 层级从 0 开始
           CAST(emp_name AS VARCHAR(500)) AS path   -- 从根到当前的路径
    FROM employees
    WHERE manager_id IS NULL               -- 这就是"顶层"的定义

    UNION ALL

    -- ② 递归成员：找出"上级已经出现在 org 里的人"
    SELECT e.emp_id,
           e.emp_name,
           e.manager_id,
           o.level + 1,
           CAST(o.path || ' > ' || e.emp_name AS VARCHAR(500))
    FROM employees e
    JOIN org o ON e.manager_id = o.emp_id   -- ← 关键：把 e 接到已展开的层级上
)
SELECT level,
       LPAD('', level * 2) || emp_name AS tree_view,   -- 用缩进可视化层级
       path
FROM org
ORDER BY path;
```

输出（示意）：

```text
level | tree_view          | path
------+--------------------+-----------------------------
    0 | 张伟                | 张伟
    1 |   王芳              | 张伟 > 王芳
    2 |     周涛            | 张伟 > 王芳 > 周涛
    1 |   李强              | 张伟 > 李强
    2 |     刘洋            | 张伟 > 李强 > 刘洋
    3 |       赵磊          | 张伟 > 李强 > 刘洋 > 赵磊
    2 |     陈静            | 张伟 > 李强 > 陈静
    3 |       孙娜          | 张伟 > 李强 > 陈静 > 孙娜
```

**几个技术要点**：

1. **`path` 列的宽度必须显式声明**（`CAST(... AS VARCHAR(500))`）。因为在很多数据库里，递归 CTE 的列类型是由**锚点成员**决定的，而锚点里的 `emp_name` 只有 `VARCHAR(50)`，拼几次就会溢出。**这是写递归 CTE 时最容易踩的坑之一**，报错信息通常是 "value too long for type character varying(50)"。
2. **`ORDER BY path`** 让输出按树的深度优先顺序排列，看起来就是一棵树。
3. **终止条件在这里是隐式的**：`JOIN org o ON e.manager_id = o.emp_id` 只在"存在下一层"时才有匹配。当某一层没有任何人把它作为上级时，递归成员返回空集，递归自然结束。**这种"靠 JOIN 自然终止"的写法是层级遍历的常态。**

### 7.4 例 3：自底向上遍历（找出某人的所有上级链）

```sql
WITH RECURSIVE chain AS (
    -- 锚点：从某个具体员工开始
    SELECT emp_id, emp_name, manager_id, 0 AS steps
    FROM employees
    WHERE emp_id = 6                          -- 赵磊

    UNION ALL

    -- 递归成员：往上找上级
    SELECT m.emp_id, m.emp_name, m.manager_id, c.steps + 1
    FROM employees m
    JOIN chain c ON m.emp_id = c.manager_id   -- 注意方向与自顶向下相反
)
SELECT steps, emp_name FROM chain ORDER BY steps;
-- 输出：0 赵磊 -> 1 刘洋 -> 2 李强 -> 3 张伟
```

**自底向上与自顶向下的唯一区别就是 `JOIN` 的方向**（`m.emp_id = c.manager_id` 对比 `e.manager_id = o.emp_id`）。理解这一点之后，两种遍历都很容易写。

### 7.5 无限递归的风险与保护

**如果数据里有环 (cycle)（例如 A 的上级是 B、B 的上级是 A），递归永远不会终止，查询会一直跑到资源耗尽。**

```text
  A (manager = B)  ←→  B (manager = A)      ← 典型的脏数据造成的环

  递归过程：A → B → A → B → A → ...
  永远不返回空集，永不终止
```

**防护手段**：

| 手段 | 说明 |
| --- | --- |
| **数据层面加约束** | 最根本的方式：禁止环（应用层校验，或触发器）。层级结构的数据维护要有"不能形成环"的检查 |
| **数据库的递归深度限制** | 部分数据库有内建上限。**MySQL 有 `cte_max_recursion_depth` 系统变量（默认 1000）**；超过会报错而不是无限跑下去 |
| **自己在查询里加深度限制** | `WHERE level < 100` 放进递归成员，最可移植 |
| **检测环** | 在 `path` 里记录已访问的节点，发现重复就停止（PostgreSQL 的 `CYCLE` 子句是标准扩展） |

```sql
-- ✅ 可移植的深度保护：加一个层级上限
WITH RECURSIVE org AS (
    SELECT emp_id, emp_name, manager_id, 0 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.emp_name, e.manager_id, o.level + 1
    FROM employees e
    JOIN org o ON e.manager_id = o.emp_id
    WHERE o.level < 100                    -- ← 硬上限，防止无限递归
)
SELECT * FROM org;
```

```sql
-- ✅ 检测环：把访问过的 id 放进 path，重复出现就停止
WITH RECURSIVE org AS (
    SELECT emp_id, emp_name, manager_id,
           CAST(',' || emp_id || ',' AS VARCHAR(1000)) AS visited
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.emp_name, e.manager_id,
           CAST(o.visited || e.emp_id || ',' AS VARCHAR(1000))
    FROM employees e
    JOIN org o ON e.manager_id = o.emp_id
    WHERE o.visited NOT LIKE '%,' || e.emp_id || ',%'    -- ← 已经访问过就不再展开
)
SELECT * FROM org;
```

**关于 PostgreSQL 的说明**：PostgreSQL **没有**内建的递归深度上限。如果写了一个可能无限递归的查询，它不会自动停下来，需要自己加 `LIMIT`、深度条件或超时（`statement_timeout`）。**这是必须自己负责的一件事。**

**关于 SQLite 的说明**：SQLite **也没有**内建上限，同样需要自己加保护条件。

**实践建议：写任何递归 CTE 时都习惯性地加一个 `WHERE level < N`。** 成本极低，但能避免"某个脏数据导致生产查询挂住"。

## 8. 什么问题适合用子查询、CTE 还是连接

这张表把本篇与前面的连接篇串起来：

| 问题类型 | 推荐写法 | 理由 |
| --- | --- | --- |
| 拿一个值去比较 | **标量子查询** | 最直白：`WHERE score > (SELECT AVG(score) FROM ...)` |
| "存在/不存在" | **`EXISTS` / `NOT EXISTS`** | `NULL` 安全、语义直接、通常性能好 |
| "值在集合里" | `IN`（子查询非空时） | 可读；**避免 `NOT IN`** |
| 排除某个集合 | **`NOT EXISTS`** | `NOT IN` 遇到 NULL 会返回空集 |
| 需要多表字段的明细 | **`JOIN`** | 子查询做不到"拼字段" |
| 分多步计算 | **CTE** | 可读、可复用、可单独调试 |
| 一次性的中间结果 | **派生表** | 简单、不用额外命名 |
| 同一中间结果用多次 | **CTE** | 派生表必须重复写 |
| 对聚合/窗口函数结果再过滤 | **派生表或 CTE** | 因为这些结果在 `WHERE` 阶段还不存在 |
| 层级/树/图遍历 | **递归 CTE** | 唯一能表达"不定层数"的方式 |
| 取每组前 N 名 | **CTE + 窗口函数** | 窗口函数结果需要包一层才能过滤 |
| 性能调优 | **先 `EXPLAIN`，再决定** | 不要凭直觉猜哪种写法更快；现代优化器会把多种写法归一到同一个计划 |

**最后一条值得强调**：`IN` 与 `EXISTS` 的"性能差异"在现代优化器上**往往不存在**（都会被改写成半连接）。所以**选择的首要依据应该是"哪个更能表达我的意图、更不容易写错"**，而不是性能传言。真正的性能问题要靠 `EXPLAIN` 去看，而不是靠换写法去猜。

## 9. 常见错误清单

| 错误 | 问题 | 修正 |
| --- | --- | --- |
| 标量子查询返回多行 | 运行时报错 | 加 `LIMIT 1`，或改用聚合函数 |
| `NOT IN` 的子查询含 NULL | **结果恒为空集** | 用 `NOT EXISTS`，或在子查询里 `IS NOT NULL` |
| 派生表忘写别名 | PostgreSQL/MySQL 报错 | 永远写 `AS t` |
| 想多次引用同一个派生表 | 只能把子查询重复写 | 改用 CTE |
| `WITH` 用在 MySQL 5.7 | 不支持 | 升级到 8.0+，或改写为派生表 |
| 递归 CTE 忘写 `RECURSIVE` | 报错（PostgreSQL/MySQL/SQLite 要求） | 写 `WITH RECURSIVE` |
| 递归 CTE 没有终止条件 | 无限递归，耗尽资源 | 加 `WHERE level < N` 或检测环 |
| 递归 CTE 的 `path` 列类型太窄 | "value too long" 报错 | 在锚点里 `CAST` 成足够宽的 `VARCHAR` |
| 递归 CTE 遇到数据里的环 | 永不终止 | 加深度上限，或在数据层禁止环 |
| `> ALL (空子查询)` 与 `> MAX(空子查询)` 混用 | 前者为 TRUE，后者为 UNKNOWN，结果不同 | 意识到空集语义差异，必要时用 `COALESCE` |
| "都存在" 的查询把"没有记录"也算进来 | 真空为真 (vacuously true) | 额外加 `EXISTS` 要求"至少有一条记录" |
| 想用窗口函数结果做 `WHERE` | 窗口函数在 `WHERE` 阶段不存在 | 包一层 CTE/派生表 |

## 小结

- 子查询按**返回形态**分四类：标量（1 行 1 列）、行、列/集合、表。**标量子查询返回多行会报错**，需要 `LIMIT 1` 或聚合。
- **`NOT IN` 的子查询里只要有一个 `NULL`，结果就是空集**（因为 `x <> NULL` 是 UNKNOWN，`WHERE` 只保留 TRUE）。`IN` 则基本不受影响——**这个不对称是陷阱难以察觉的原因**。
- **需要"排除"语义时默认用 `NOT EXISTS`**：它是存在性判断，`NULL` 安全，而且不会因为子查询返回多行而产生重复。
- `EXISTS` 找到第一行就短路，所以惯例写 `SELECT 1`（写 `SELECT *` 效果相同，优化器不会真去取列）。
- **相关子查询**引用外层的列，逻辑上对外层每一行执行一次；优化器通常改写成连接。在标量子查询里用 `COUNT(*)` 会返回 `0` 而不是 `NULL`，这比 `SUM` 方便。
- `= ANY` 等价于 `IN`，`<> ALL` 等价于 `NOT IN`；`> ALL` 等价于 `> MAX`，`> ANY` 等价于 `> MIN`。**但空集语义不同**：`> ALL (空集)` 恒为 TRUE，`> MAX(空集)` 是 UNKNOWN。**等价写法在边界条件下不等价，这是聚合替代 `ALL`/`ANY` 时的隐患。**
- **派生表必须起别名**（PostgreSQL 与 MySQL 强制要求），但它**不能被同一层引用两次**。
- **CTE 的核心价值是"从上往下的可读性 + 可复用 + 可单独调试"**：把嵌套三层的子查询改写成 CTE 流水线，逻辑只需写一遍，改口径时不会漏。**推荐把每一个中间步骤的 CTE 单独拿出来 `SELECT * LIMIT 10` 验证，这是排查复杂查询最有效的方法。**
- **`WITH` 需要 MySQL 8.0+**（PostgreSQL 8.4+、SQLite 3.8.3+ 支持）。
- **CTE 可能被物化也可能被内联**，不同数据库与版本策略不同（PostgreSQL 12 改变了默认行为），会影响性能。**不要猜，用 `EXPLAIN ANALYZE` 看。**
- **递归 CTE 由三部分组成**：锚点成员（不引用自己）、`UNION ALL`、递归成员（引用自己 + 有终止条件）。
- **`RECURSIVE` 关键字**：PostgreSQL / MySQL 8.0+ / SQLite 要求写；SQL Server 与 Oracle 不写。
- **递归 CTE 的列类型由锚点决定**，拼接出来的 `path` 列要在锚点里 `CAST` 成足够宽的 `VARCHAR`，否则会 "value too long"。
- **递归 CTE 遇到数据里的环会永不终止**。MySQL 有 `cte_max_recursion_depth`（默认 1000）保护，**PostgreSQL 与 SQLite 没有内建上限**——必须自己加 `WHERE level < N` 或环检测。**养成写递归就加上限的习惯。**
- 自顶向下与自底向上遍历的唯一区别是 `JOIN` 的方向。
- **"所有 X 都满足 Y"在 X 为空集时真空为真**——这往往是业务上的错误。要表达"选过课且都及格"，需要额外加一个 `EXISTS`。**写业务查询时养成问"这组数据一行都没有时返回什么"的习惯。**
- `IN` 与 `EXISTS` 在现代优化器上**性能通常相同**（都改写成半连接）。选择依据应当是**可读性与正确性**，真正的性能问题交给 `EXPLAIN`。
