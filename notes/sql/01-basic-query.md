# SQL 基础查询：SELECT / WHERE / ORDER BY / LIMIT / DISTINCT

本篇梳理 SQL 的逻辑执行顺序、单表筛选与排序的完整语法，重点讲清三值逻辑与 `NULL` 带来的陷阱，以及 `NOT IN` 为什么可能返回空结果。

## 0. 关系模型与 SQL 分类

一张表 (table) 就是关系的具体形态：**行 (row / tuple) 是记录，列 (column / attribute) 是属性**。表有名字，列有名字和类型，行没有固有顺序（**这一点很重要**——"结果的顺序"必须靠 `ORDER BY` 明确指定）。

SQL 语句按用途分类：

| 类别 | 全称 | 包含 | 例子 |
| --- | --- | --- | --- |
| DDL | Data Definition Language | 定义结构 | `CREATE`、`ALTER`、`DROP`、`TRUNCATE` |
| DML | Data Manipulation Language | 增删改数据 | `INSERT`、`UPDATE`、`DELETE` |
| DQL | Data Query Language | 查询 | `SELECT` |
| DCL | Data Control Language | 权限 | `GRANT`、`REVOKE` |
| TCL | Transaction Control Language | 事务 | `BEGIN`、`COMMIT`、`ROLLBACK` |

**术语上的一个诚实说明**：`SELECT` 是否单独算 DQL 并没有统一共识。很多教材把 DQL 并入 DML（因为都是"操作数据"），也有资料把它单独列出。看到不同分类不要觉得矛盾。

## 1. 示例数据库

本篇的所有示例都基于这套结构（整个 SQL 板块复用同一套）：

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

一点示例数据，后面的例子都基于它可以得到可预期的结果：

```sql
INSERT INTO students (student_id, name, gender, birth_date, major, city, enroll_year) VALUES
    (1, '王小明', 'M', '2004-03-15', '计算机科学与技术', '厦门',   2022),
    (2, '李雨桐', 'F', '2004-07-22', '计算机科学与技术', '北京',   2022),
    (3, '张浩然', 'M', '2003-11-08', '软件工程',         '厦门',   2021),
    (4, '陈思远', 'M', '2005-01-30', '计算机科学与技术', '上海',   2023),
    (5, '刘悦然', 'F', '2004-09-12', '数据科学',         NULL,     2022),
    (6, '赵子涵', 'F', NULL,         '软件工程',         '厦门',   2023);
```

注意第 5 行 `city` 为 `NULL`、第 6 行 `birth_date` 为 `NULL`——这两行是后面演示 `NULL` 语义时必需的。

## 2. 逻辑执行顺序 —— 本篇最重要的概念

**SQL 的书写顺序与它的逻辑执行顺序不同**，这一点是所有"为什么这里不能用那个"问题的根源。

```text
书写顺序 (你打字的顺序)：
  SELECT  ->  FROM  ->  WHERE  ->  GROUP BY  ->  HAVING  ->  ORDER BY  ->  LIMIT

逻辑执行顺序 (数据库思考的顺序)：
  FROM  ->  WHERE  ->  GROUP BY  ->  HAVING  ->  SELECT  ->  DISTINCT  ->  ORDER BY  ->  LIMIT
```

分步解释每一步在做什么：

| 步骤 | 做什么 | 此时数据是什么形态 |
| --- | --- | --- |
| `FROM` | 确定数据源，执行连接，得到"所有候选行" | 行集合（可能很大） |
| `WHERE` | 逐行判断条件，**只保留结果为 TRUE 的行** | 过滤后的行集合 |
| `GROUP BY` | 按分组键把行**分桶**，每桶折叠成一行 | 行数 = 分组数 |
| `HAVING` | 对**每个分组**判断条件，过滤掉整组 | 分组数减少 |
| `SELECT` | 计算要输出的列（表达式、别名在这里产生） | 结果集的列已确定 |
| `DISTINCT` | 对 `SELECT` 出的整行去重 | 行数可能减少 |
| `ORDER BY` | 排序 | 有序结果集 |
| `LIMIT` | 截取前 N 行 | 最终结果 |

### 2.1 由执行顺序推出的四个结论

**结论 1：`WHERE` 里不能用聚合函数。**

```sql
-- ❌ 报错：aggregate functions are not allowed in WHERE
SELECT * FROM enrollments WHERE score > AVG(score);

-- 原因：WHERE 在 GROUP BY 之前执行，此时还没有"分组"，也就没有"平均值"这个东西。
-- 报错信息在不同数据库里措辞不同（PostgreSQL: "aggregate functions are not allowed in WHERE"；
-- MySQL: "Invalid use of group function"），但原因相同。

-- ✅ 正确做法 A：需要的是"全体的平均分"，用标量子查询
SELECT * FROM enrollments WHERE score > (SELECT AVG(score) FROM enrollments);

-- ✅ 正确做法 B：需要的是"每个分组的平均分"，那就先分组再用 HAVING
SELECT course_id, AVG(score) AS avg_score
FROM enrollments
GROUP BY course_id
HAVING AVG(score) > 75;
```

**结论 2：`WHERE` 里通常不能用 `SELECT` 起的别名。**

```sql
-- ❌ 多数数据库会报错：column "grade" does not exist
SELECT score * 1.1 AS adjusted FROM enrollments WHERE adjusted > 90;

-- 原因：WHERE 在 SELECT 之前执行，别名还没被计算出来。

-- ✅ 写法 A：重复表达式
SELECT score * 1.1 AS adjusted FROM enrollments WHERE score * 1.1 > 90;

-- ✅ 写法 B：包一层（用派生表或 CTE）
SELECT * FROM (
    SELECT enrollment_id, score * 1.1 AS adjusted FROM enrollments
) AS t
WHERE adjusted > 90;
```

> **诚实说明**：不同数据库对"别名在各子句中的可用性"宽容度不同。MySQL 允许在 `GROUP BY` / `HAVING` / `ORDER BY` 里用 `SELECT` 别名，但不允许在 `WHERE` 里用；PostgreSQL 也不允许在 `WHERE` 里用。**不要依赖这种宽容性**——它不可移植，而且依赖它会让你对执行顺序的理解变得模糊。

**结论 3：`ORDER BY` 里可以用别名。**

```sql
-- ✅ 合法，因为 ORDER BY 在 SELECT 之后执行
SELECT name, score * 0.6 + 40 AS final_score
FROM students s
JOIN enrollments e ON e.student_id = s.student_id
ORDER BY final_score DESC;

-- ✅ 也可以用序号（按 SELECT 里第几列排序），但不推荐
SELECT name, score FROM ... ORDER BY 2 DESC;
-- 序号排序的问题是：一旦改了 SELECT 的列顺序，排序就悄悄变了，很难发现。
```

**结论 4：`LIMIT` 在最后，所以它是在"排序之后"截取的。**

```sql
-- 取分数最高的 3 条
SELECT * FROM enrollments ORDER BY score DESC LIMIT 3;
-- 若去掉 ORDER BY，就是"任意 3 条"，因为表的行没有固有顺序！
```

## 3. `SELECT`

### 3.1 `SELECT *` 的问题

```sql
SELECT * FROM students;
```

能用，但在正式代码里应当避免。三个理由：

1. **不明确**：读代码的人不知道你实际需要哪些列；
2. **耦合列顺序**：调用方如果按位置取列（很多语言的驱动支持按序号取），加一列就可能错位；
3. **可能带来不必要的列**：大字段（`TEXT`、`BLOB`）被无意拉出来会浪费网络与内存。

```sql
-- ✅ 明确列出需要的列
SELECT student_id, name, major FROM students;

-- 例外：交互式探索数据时用 SELECT * 是合理的
```

### 3.2 列别名与表达式

```sql
SELECT
    name                            AS 姓名,          -- 别名可以是中文（需要看数据库是否允许不引号）
    score,                                           -- 原始列
    score * 0.6 + 40                AS final_score,  -- 表达式
    CASE WHEN score >= 60 THEN '及格' ELSE '不及格' END AS 结果,
    COALESCE(score, 0)              AS score_or_zero -- NULL 处理
FROM students s
JOIN enrollments e ON e.student_id = s.student_id;
```

几点说明：

- **`AS` 可以省略**（`SELECT name 姓名` 也行），但写上更清晰；
- **别名含空格或特殊字符、或是关键字时必须加引号**。标准 SQL 用双引号（`AS "final score"`），MySQL 默认也接受反引号（`` AS `final score` ``）。**注意标准 SQL 里单引号是字符串字面量，不是标识符引用**——用单引号包别名在某些数据库上会被当字符串而报错或产生意外的列名；
- **中文别名**在大多数现代数据库上可以直接用，不加引号通常也行（因为多数数据库允许 Unicode 标识符），但为了可移植性建议加双引号。

## 4. `WHERE`：条件筛选

### 4.1 比较运算符

| 运算符 | 含义 | 备注 |
| --- | --- | --- |
| `=` | 等于 | 注意：SQL 里判等是 `=`，不是 `==` |
| `<>` | 不等于 | 标准写法 |
| `!=` | 不等于 | 被广泛支持的同义词，但不是所有数据库都认 |
| `<` `<=` `>` `>=` | 大小比较 | |

```sql
SELECT * FROM students WHERE gender = 'M';
SELECT * FROM students WHERE major <> '软件工程';
SELECT * FROM enrollments WHERE score >= 90;
```

**`!=` 与 `<>` 的关系**：两者等价。`<>` 是 SQL 标准写法，`!=` 是历史遗留但几乎所有数据库都支持。**写 `<>` 可移植性更好。**

### 4.2 逻辑运算符与优先级

```sql
-- AND 的优先级【高于】OR
-- 所以下面这句的含义可能不是你想的那样：
SELECT * FROM students WHERE major = '计算机科学与技术' OR major = '软件工程' AND city = '厦门';
-- 实际被解析为：major='计算机科学与技术' OR (major='软件工程' AND city='厦门')
-- 也就是说，"计算机科学与技术"的行【不管城市是哪】都会被选中

-- ✅ 要表达"两个专业之一，且都在厦门"，必须加括号
SELECT * FROM students
WHERE (major = '计算机科学与技术' OR major = '软件工程') AND city = '厦门';
```

`NOT` 的优先级比 `AND` 和 `OR` 都高。

**结论**：**在混用 `AND` 和 `OR` 时一律加括号。** 这不只是风格问题——不加括号的 bug 不会报错，只会静默返回错误的结果。

### 4.3 `BETWEEN ... AND ...`

```sql
-- BETWEEN 是【闭区间】：包含两个端点
SELECT * FROM enrollments WHERE score BETWEEN 60 AND 90;
-- 等价于：score >= 60 AND score <= 90

-- 日期范围
SELECT * FROM students WHERE birth_date BETWEEN '2004-01-01' AND '2004-12-31';

-- 取反（注意写法）
SELECT * FROM enrollments WHERE score NOT BETWEEN 60 AND 90;
-- 等价于：score < 60 OR score > 90
```

**`BETWEEN` 的两个注意点**：

1. **对 `NULL` 的行为**：如果 `score` 是 `NULL`，`NULL BETWEEN 60 AND 90` 的结果是 `UNKNOWN`，因此该行被 `WHERE` 过滤掉。**`BETWEEN` 不会匹配 `NULL`**。
2. **用 `BETWEEN` 处理时间戳时容易漏掉最后一天**。如果 `created_at` 是 `TIMESTAMP`（带时分秒），`BETWEEN '2025-03-01' AND '2025-03-31'` 会**漏掉 3 月 31 日当天的所有记录**，因为 `'2025-03-31'` 被解释为 `2025-03-31 00:00:00`。这是一个极其常见的生产事故。

```sql
-- ❌ 漏掉 3 月 31 日的数据
WHERE created_at BETWEEN '2025-03-01' AND '2025-03-31'

-- ✅ 用左闭右开区间（推荐写法）
WHERE created_at >= '2025-03-01' AND created_at < '2025-04-01'

-- ✅ 或者显式补上时间
WHERE created_at >= '2025-03-01 00:00:00' AND created_at <= '2025-03-31 23:59:59'
```

**左闭右开 `>= start AND < next_start` 是处理时间范围的通用最佳实践**，它不依赖列的时间精度，也不会因为闰秒或纳秒精度而漏数据。

### 4.4 `IN` 与 `NOT IN`

```sql
SELECT * FROM students WHERE major IN ('计算机科学与技术', '软件工程', '数据科学');
SELECT * FROM students WHERE major NOT IN ('软件工程');
SELECT * FROM students WHERE enroll_year IN (2022, 2023);
```

`IN` 接受一个列表，也可以接受子查询：

```sql
SELECT * FROM students
WHERE student_id IN (SELECT student_id FROM enrollments WHERE score > 90);
```

### 4.5 `IS NULL` 与 `IS NOT NULL`（不能用 `= NULL`）

```sql
-- ❌ 永远不返回任何行
SELECT * FROM students WHERE city = NULL;
-- 原因：NULL 表示"未知"，与"未知"比较的结果是 UNKNOWN，不是 TRUE。

-- ✅ 正确写法
SELECT * FROM students WHERE city IS NULL;
SELECT * FROM students WHERE city IS NOT NULL;
```

这是初学者最常犯的错误之一。**记住：`NULL` 只能用 `IS NULL` / `IS NOT NULL` 判断，任何 `= NULL`、`<> NULL`、`!= NULL` 都是错的。**

### 4.6 `NOT IN` 的 NULL 陷阱 —— 必须讲清楚

这是 SQL 里最经典的陷阱之一，也是初学者最难自己发现的一个。

```sql
-- 假设 enrollments 里有 score 为 NULL 的行（成绩未出）
-- 下面这个查询想找出"没有不及格记录的学生"
SELECT * FROM students
WHERE student_id NOT IN (SELECT student_id FROM enrollments WHERE score < 60);

-- 这个查询本身是安全的，因为子查询里加了 score < 60 的过滤条件，
-- NULL 的 score 不会满足 score < 60，所以子查询不会返回 NULL。

-- ❌ 但下面这个就危险了：子查询返回的列里可能含 NULL
SELECT * FROM students
WHERE student_id NOT IN (SELECT student_id FROM enrollments);

-- 如果 enrollments 里【有任何一行的 student_id 是 NULL】，
-- 整个查询会返回【空结果集】！
```

**为什么会这样**：`NOT IN` 在语义上等价于一连串的 `<>` 加 `AND`：

```text
x NOT IN (a, b, c)
  等价于  x <> a AND x <> b AND x <> c
```

如果 `a` 是 `NULL`，那么 `x <> NULL` 的结果是 **UNKNOWN**（不是 FALSE）。而 `TRUE AND UNKNOWN` 是 `UNKNOWN`，`UNKNOWN AND anything` 不可能是 `TRUE`。由于 `WHERE` 只保留结果为 `TRUE` 的行，**整个条件永远无法为真，结果必然为空**。

用三值逻辑表看更清楚：

| `x` | `y` | `x <> y` | `TRUE AND (x <> y)` |
| --- | --- | --- | --- |
| 1 | 2 | TRUE | TRUE |
| 1 | 1 | FALSE | FALSE |
| 1 | **NULL** | **UNKNOWN** | **UNKNOWN**（被过滤） |
| **NULL** | 2 | **UNKNOWN** | **UNKNOWN**（被过滤） |

**两种修正方式**：

```sql
-- 修正 A：排除子查询中的 NULL
SELECT * FROM students
WHERE student_id NOT IN (
    SELECT student_id FROM enrollments WHERE student_id IS NOT NULL
);

-- 修正 B（推荐）：改用 NOT EXISTS，它按"存在性"判断，不受 NULL 影响
SELECT * FROM students s
WHERE NOT EXISTS (
    SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id
);
```

**`EXISTS` 为什么不受影响**：`EXISTS` 判断的是"子查询是否返回了至少一行"，这是**布尔存在性**，不涉及与 `NULL` 的比较。所以只要内层没有匹配的行，`NOT EXISTS` 就为真。

**实践结论：需要"排除"语义时，优先用 `NOT EXISTS` 而不是 `NOT IN`。** 详见 [04-subquery-and-cte.md](04-subquery-and-cte.md)。

### 4.7 `LIKE` 与通配符

| 通配符 | 含义 |
| --- | --- |
| `%` | 匹配任意长度（含 0）的任意字符序列 |
| `_` | 匹配**恰好一个**任意字符 |
| `[charlist]` | 匹配字符列表中的任一字符（**不是所有数据库都支持**；SQL Server 支持，PostgreSQL/MySQL/SQLite 的标准 `LIKE` 不支持） |

```sql
-- 姓"王"的学生
SELECT * FROM students WHERE name LIKE '王%';

-- 名字第二个字是"雨"
SELECT * FROM students WHERE name LIKE '_雨%';

-- 名字里含"子"
SELECT * FROM students WHERE name LIKE '%子%';

-- 名字以"远"结尾
SELECT * FROM students WHERE name LIKE '%远';

-- 不区分大小写：用 LOWER/UPPER（可移植），或 ILIKE（PostgreSQL 特有）
SELECT * FROM courses WHERE LOWER(course_name) LIKE '%database%';
-- PostgreSQL 特有：
-- SELECT * FROM courses WHERE course_name ILIKE '%database%';

-- 转义字面量的 % 或 _：用 ESCAPE 子句
SELECT * FROM courses WHERE course_name LIKE '%100\%%' ESCAPE '\';
-- 匹配"含 100% 字样"的课程名
```

**性能提示**：`LIKE '%keyword%'`（**以通配符开头**）无法利用普通的 B-tree 索引，通常会导致全表扫描。如果这类搜索很频繁，需要考虑全文索引 (full-text index) 或专门的搜索引擎。以固定前缀开头的 `LIKE '王%'` 则是可以用上索引的。详见 [05-ddl-and-constraints.md](05-ddl-and-constraints.md) 的索引部分。

## 5. 三值逻辑 (three-valued logic)

这一节是理解 `NULL` 所有行为的总纲。

**`NULL` 的语义是"未知 (unknown)"，而不是"空"或"零"。** 因此它参与任何比较的结果都是"未知"，而 SQL 用三个真值来表示：

| 值 | 含义 |
| --- | --- |
| `TRUE` | 确定为真 |
| `FALSE` | 确定为假 |
| `UNKNOWN` | 无法确定（涉及 NULL 时产生） |

### 5.1 三值逻辑的真值表

**`AND`**：

| AND | TRUE | FALSE | UNKNOWN |
| --- | --- | --- | --- |
| **TRUE** | TRUE | FALSE | UNKNOWN |
| **FALSE** | FALSE | FALSE | FALSE |
| **UNKNOWN** | UNKNOWN | FALSE | UNKNOWN |

**`OR`**：

| OR | TRUE | FALSE | UNKNOWN |
| --- | --- | --- | --- |
| **TRUE** | TRUE | TRUE | TRUE |
| **FALSE** | TRUE | FALSE | UNKNOWN |
| **UNKNOWN** | TRUE | UNKNOWN | UNKNOWN |

**`NOT`**：

| NOT | 结果 |
| --- | --- |
| `NOT TRUE` | FALSE |
| `NOT FALSE` | TRUE |
| `NOT UNKNOWN` | **UNKNOWN** |

### 5.2 最重要的一条规则

> **`WHERE` 只保留条件结果为 `TRUE` 的行。结果为 `FALSE` 或 `UNKNOWN` 的行都会被丢弃。**

这条规则解释了一切：

```sql
SELECT * FROM students WHERE city = '厦门';       -- 城市不是厦门的被丢掉
SELECT * FROM students WHERE city <> '厦门';      -- ⚠️ city 为 NULL 的行【也被丢掉】！
```

第二句尤其重要：`city` 为 `NULL` 的学生**既不在 `= '厦门'` 的结果里，也不在 `<> '厦门'` 的结果里**。它被两个查询都排除了。想把它包括进来必须显式处理：

```sql
SELECT * FROM students WHERE city <> '厦门' OR city IS NULL;
```

### 5.3 `NULL` 参与运算与函数

```sql
SELECT 1 + NULL;                 -- NULL（任何算术运算遇到 NULL 都得 NULL）
SELECT 'abc' || NULL;            -- NULL（字符串拼接；'||' 是标准写法，MySQL 里要用 CONCAT）
SELECT NULL = NULL;              -- UNKNOWN（不是 TRUE！）
SELECT NULL IS NULL;             -- TRUE（IS 是专门用来判断 NULL 的）
SELECT COUNT(*), COUNT(NULL);    -- 行数, 0（聚合函数忽略 NULL，见 03 篇）
```

### 5.4 处理 `NULL` 的函数

```sql
-- COALESCE：返回第一个非 NULL 的参数（标准 SQL，所有主流数据库都支持）
SELECT name, COALESCE(city, '未填写') AS city_display FROM students;
SELECT COALESCE(score, 0) FROM enrollments;

-- NULLIF：两值相等则返回 NULL，否则返回第一个（常用于避免除零）
SELECT 10 / NULLIF(0, 0);        -- NULL，而不是报除零错误
SELECT NULLIF(score, 0);         -- score 为 0 时返回 NULL

-- CASE WHEN：最灵活的方式，可移植性最好
SELECT name,
       CASE
           WHEN city IS NULL THEN '未填写'
           WHEN city = '厦门'  THEN '本地'
           ELSE city
       END AS city_category
FROM students;
```

`COALESCE` 与 `CASE WHEN` 的关系：`COALESCE(a, b)` 基本等价于 `CASE WHEN a IS NOT NULL THEN a ELSE b END`（但 `COALESCE` 可以接受任意多个参数，且是标准函数）。

**`NULLIF` 配合除法的经典用法**：

```sql
-- 计算"及格率"时避免除零
SELECT
    course_id,
    COUNT(*) AS total,
    SUM(CASE WHEN score >= 60 THEN 1 ELSE 0 END) AS passed,
    ROUND(
        SUM(CASE WHEN score >= 60 THEN 1 ELSE 0 END) * 100.0 / NULLIF(COUNT(*), 0),
        2
    ) AS pass_rate
FROM enrollments
GROUP BY course_id;
```

`NULLIF(COUNT(*), 0)` 在没有任何行时把分母变成 `NULL`，于是结果是 `NULL` 而不是报错。**这是"聚合计算 + 除零保护"的标准写法。**

### 5.5 `NULL` 与唯一约束、`DISTINCT`、`GROUP BY`

这些行为在不同子句里不完全一致，值得单独列出（后面几篇会展开）：

| 场景 | `NULL` 的行为 |
| --- | --- |
| `UNIQUE` 约束 | **多个 `NULL` 互不冲突**（标准如此，MySQL/PostgreSQL/SQLite 都遵守）。因为 `NULL <> NULL` 是 UNKNOWN，不构成重复 |
| `PRIMARY KEY` | **不允许 `NULL`**（主键隐含 `NOT NULL`） |
| `DISTINCT` | 多个 `NULL` 被当作**相同**，只保留一个 |
| `GROUP BY` | 所有 `NULL` 被分到**同一个组** |
| `ORDER BY` | `NULL` 排在最前还是最后**因数据库而异**（见下节） |
| `COUNT(*)` | 数所有行，**包括含 NULL 的行** |
| `COUNT(col)` | **跳过** `col` 为 `NULL` 的行 |
| `SUM` / `AVG` / `MIN` / `MAX` | **忽略** `NULL` |
| 字符串拼接 | 任一操作数为 `NULL`，结果就是 `NULL`（用 `COALESCE` 处理） |

**`DISTINCT` 与 `GROUP BY` 把 NULL 归为一组，而 `UNIQUE` 视为不冲突**——这两点看似矛盾，其实一致：`UNIQUE` 关注的是"两个值是否相等"，而 `NULL = NULL` 是 UNKNOWN，所以无法判定为重复；`GROUP BY` / `DISTINCT` 关注的是"如何分桶"，需要一个确定的归组方式，所以把 NULL 放一起。

## 6. `ORDER BY`

```sql
-- 单列
SELECT * FROM enrollments ORDER BY score DESC;      -- 降序（默认是 ASC 升序）

-- 多列：先按第一列排，第一列相同时按第二列
SELECT * FROM students ORDER BY major ASC, enroll_year DESC;

-- 按表达式
SELECT * FROM enrollments ORDER BY score * 1.1 DESC;

-- 按别名（见第 2.1 节结论 3）
SELECT name, score * 0.6 + 40 AS final_score FROM ... ORDER BY final_score DESC;

-- 按序号（不推荐）
SELECT name, score FROM ... ORDER BY 2 DESC;
```

### 6.1 `NULL` 在排序中的位置

**标准 SQL 没有规定 `NULL` 排在最前还是最后，各数据库的实现不同。**

| 数据库 | `ASC` 时 NULL 的位置 | `DESC` 时 NULL 的位置 |
| --- | --- | --- |
| PostgreSQL | 最后 | 最前 |
| MySQL | 最前 | 最后 |
| SQLite | 最前 | 最后 |
| Oracle | 最后 | 最前 |

（这里的"最前/最后"是把 `NULL` 当作最大还是最小值的体现。PostgreSQL 与 Oracle 把 `NULL` 视为最大，MySQL 与 SQLite 视为最小。**这正是"标准未规定、实现各异"的典型例子**，写跨数据库的代码时不能假设。）

### 6.2 显式控制 `NULL` 的位置

```sql
-- PostgreSQL / Oracle：支持标准语法
SELECT * FROM students ORDER BY birth_date ASC NULLS LAST;
SELECT * FROM students ORDER BY birth_date DESC NULLS FIRST;

-- MySQL / SQLite：没有 NULLS FIRST/LAST，用"是否为空"作为第一排序键
SELECT * FROM students
ORDER BY (birth_date IS NULL), birth_date ASC;
-- 原理：(birth_date IS NULL) 在 MySQL/SQLite 里返回 0 或 1，
--       所以非 NULL 的行（0）排前面，NULL 的行（1）排后面。
--       反过来写 (birth_date IS NOT NULL) 就能让 NULL 排最前面。

-- 通用写法（所有数据库都能跑）：用 CASE
SELECT * FROM students
ORDER BY CASE WHEN birth_date IS NULL THEN 1 ELSE 0 END, birth_date ASC;
```

**最后那种 `CASE` 写法是最可移植的**，虽然啰嗦一点，但在任何数据库上语义都相同。需要跨数据库时推荐用它。

### 6.3 排序与"结果顺序不确定"

```sql
-- ⚠️ 这个查询的结果顺序是【不确定的】
SELECT * FROM students LIMIT 5;

-- 因为表的行没有固有顺序：数据库可以按任意顺序返回（取决于存储引擎、
-- 索引选择、并行度、甚至统计信息的更新）。
-- "这次跑出来是这 5 行"不代表"下次也是"。
```

**任何依赖顺序的场景（分页、取 Top N）都必须显式 `ORDER BY`。** 这里有一个更隐蔽的推论：**用于分页的 `ORDER BY` 必须能唯一确定顺序**。

```sql
-- ⚠️ 危险的分页：按 score 排序，但 score 有大量重复值
SELECT * FROM enrollments ORDER BY score DESC LIMIT 10 OFFSET 0;
-- 下一页
SELECT * FROM enrollments ORDER BY score DESC LIMIT 10 OFFSET 10;

-- 问题：分数相同的那些行之间的相对顺序是任意的，两次查询可能给出不同的顺序，
-- 于是翻页时【可能出现重复的行，也可能漏掉行】。

-- ✅ 加上唯一的决胜键，保证顺序完全确定
SELECT * FROM enrollments ORDER BY score DESC, enrollment_id ASC LIMIT 10 OFFSET 0;
```

**"排序键 + 主键"是稳定分页的标准做法。**

## 7. `LIMIT` 与分页

```sql
-- 取前 10 条
SELECT * FROM enrollments ORDER BY score DESC LIMIT 10;

-- 跳过前 20 条，取 10 条（第 3 页，每页 10 条）
SELECT * FROM enrollments ORDER BY score DESC, enrollment_id LIMIT 10 OFFSET 20;

-- 有些数据库支持简写：LIMIT offset, count（MySQL）
SELECT * FROM enrollments ORDER BY score DESC LIMIT 20, 10;   -- 等价于 OFFSET 20 LIMIT 10
-- ⚠️ 注意这两个数字的顺序（先 offset 后 count），很容易搞反。推荐用 LIMIT ... OFFSET ... 的写法。
```

**标准 SQL 的写法**：

```sql
-- SQL:2008 标准
SELECT * FROM enrollments ORDER BY score DESC
OFFSET 20 ROWS FETCH FIRST 10 ROWS ONLY;

-- 或者
... FETCH NEXT 10 ROWS ONLY
```

**方言支持情况**：

| 写法 | 支持情况 |
| --- | --- |
| `LIMIT n` | MySQL、PostgreSQL、SQLite；**SQL Server 不支持** |
| `LIMIT n OFFSET m` | 同上 |
| `OFFSET m ROWS FETCH FIRST n ROWS ONLY` | 标准 SQL；PostgreSQL、Oracle、SQL Server 2012+ 支持 |
| `TOP n` | SQL Server 特有 |
| `ROWNUM` | Oracle 传统方式 |

`LIMIT` 的可移植性实际上很好（除 SQL Server），日常用它就够了。

### 7.1 `OFFSET` 大的时候为什么慢

```text
LIMIT 10 OFFSET 1000000
```

数据库**无法直接跳到第 1000001 行**，因为行没有固有顺序——它必须先产生前 1000010 行（包括排序），然后丢掉前 1000000 行。所以 `OFFSET` 很大时，代价随 offset 线性增长。

**这就是"深分页"问题**。常见的缓解手段（了解思路即可）：

- **基于游标 (keyset pagination)**：不用 `OFFSET`，而是记住上一页最后一行的排序键，下次用 `WHERE (score, enrollment_id) < (:last_score, :last_id)` 直接定位。要求在排序键上有合适的索引。

```sql
-- keyset 分页：假设上一页最后一行是 (score=85.5, enrollment_id=42)
SELECT * FROM enrollments
WHERE (score, enrollment_id) < (85.5, 42)   -- 行值比较，PostgreSQL / MySQL 支持
ORDER BY score DESC, enrollment_id DESC
LIMIT 10;
```

这种写法每次查询的代价与页码无关，是深分页的正解。**缺点是只能顺序翻页，不能直接跳到第 N 页。**

## 8. `DISTINCT`

```sql
-- 单列去重
SELECT DISTINCT major FROM students;

-- 多列：对【整行的组合】去重，不是对每一列分别去重！
SELECT DISTINCT major, city FROM students;
-- 这会返回 (专业, 城市) 的所有不同组合，
-- 而不是"所有专业"或"所有城市"。

-- 计数唯一值（非常常用）
SELECT COUNT(DISTINCT major) AS major_count FROM students;
SELECT COUNT(DISTINCT course_id) AS course_count FROM enrollments;

-- 多列的 distinct 计数：多数数据库支持，但部分数据库（如 MySQL 5.7 之前的某些版本）行为受限
SELECT COUNT(DISTINCT major, city) FROM students;     -- PostgreSQL 支持；MySQL 也支持
```

### 8.1 `DISTINCT` 与 `GROUP BY` 的关系

```sql
-- 这两个查询的【结果相同】
SELECT DISTINCT major, city FROM students;
SELECT major, city FROM students GROUP BY major, city;
```

从逻辑执行顺序看，`DISTINCT` 等价于"对所有输出列做 `GROUP BY`"。区别在于：

| | `DISTINCT` | `GROUP BY` |
| --- | --- | --- |
| 意图 | 去重 | 分组并**计算聚合** |
| 能否用聚合函数 | ❌（`SELECT DISTINCT COUNT(*)` 没有意义） | ✅ |
| 可读性 | 更直白 | 当需要聚合时更自然 |

**选择原则**：只是想去重就用 `DISTINCT`；需要按组算东西就用 `GROUP BY`。**用 `GROUP BY` 来实现"去重"虽然结果对，但会让读者以为你要做聚合**，语义不清晰。

### 8.2 `DISTINCT` 的一个常见误解

```sql
-- ❌ 这段代码的意图可能是"每个专业取一行学生"，但它不会这样工作
SELECT DISTINCT major, name FROM students;
-- 结果：所有 (专业, 姓名) 的不同组合。因为姓名各不相同，
--       所以结果基本就是整张表，DISTINCT 没起到"每个专业一行"的作用。

-- ✅ 想要"每个专业一行"，必须明确指定取哪一行（用窗口函数或聚合）
SELECT major, MIN(name) AS example_name, COUNT(*) AS student_count
FROM students
GROUP BY major;
```

**`DISTINCT` 作用于结果集的整行**，所以只要结果里有任何一列区分度很高（如 `name`、`id`），`DISTINCT` 就几乎不起作用。**这是"DISTINCT 没生效"这类困惑的根本原因。**

## 9. 综合示例

结合上面的知识，写几个完整可用的查询。

```sql
-- 示例 1：找出 2004 年出生、且专业为计算机或软件工程的厦门学生
SELECT student_id, name, major, birth_date
FROM students
WHERE (major = '计算机科学与技术' OR major = '软件工程')
  AND city = '厦门'
  AND birth_date >= '2004-01-01'
  AND birth_date <  '2005-01-01'
  AND birth_date IS NOT NULL              -- 显式排除 NULL（其实上面的比较已经排除了，
ORDER BY birth_date ASC, name ASC;        -- 写出来是为了让意图明确）

-- 示例 2：找出所有"有成绩记录但成绩缺失"的选课，按学期和课程号排序
SELECT enrollment_id, student_id, course_id, semester
FROM enrollments
WHERE score IS NULL
ORDER BY semester DESC, course_id ASC;

-- 示例 3：找出分数在 80-90 之间（含端点）的前 5 条记录，用稳定的排序
SELECT enrollment_id, course_id, score
FROM enrollments
WHERE score BETWEEN 80 AND 90
ORDER BY score DESC, enrollment_id ASC
LIMIT 5;

-- 示例 4：找出所有不同的 (学期, 学年) 组合，按学期降序
SELECT DISTINCT semester
FROM enrollments
WHERE semester IS NOT NULL
ORDER BY semester DESC;

-- 示例 5：列出所有学生，把 NULL 的城市显示成"未填写"，并按"是否有城市"排序
SELECT student_id,
       name,
       COALESCE(city, '未填写') AS city_display
FROM students
ORDER BY (city IS NULL), student_id;
```

**示例 5 的两个技术点**：`COALESCE` 处理显示层的 NULL；`ORDER BY (city IS NULL), student_id` 让有城市的学生排前面（`(city IS NULL)` 为 0 的排在为 1 的前面），且用 `student_id` 作为决胜键保证顺序确定。

## 10. 常见错误清单

| 错误写法 | 问题 | 正确写法 |
| --- | --- | --- |
| `WHERE city = NULL` | 结果永远为空 | `WHERE city IS NULL` |
| `WHERE score <> 60` 却期望包含 NULL 行 | NULL 被过滤掉了 | `WHERE score <> 60 OR score IS NULL` |
| `WHERE a = 1 OR b = 2 AND c = 3` | `AND` 优先级更高，含义与预期不同 | 加括号明确意图 |
| `WHERE score > AVG(score)` | 聚合函数不能出现在 `WHERE` | 用标量子查询或 `HAVING` |
| `WHERE 别名 > 90` | 别名在 `WHERE` 阶段还不存在 | 重复表达式，或包一层子查询 |
| `x NOT IN (子查询含 NULL)` | 结果为空 | `NOT EXISTS`，或给子查询加 `IS NOT NULL` |
| `SELECT * ... LIMIT 5` 无 `ORDER BY` | 返回哪 5 行不确定 | 加 `ORDER BY` |
| 分页只按非唯一列 `ORDER BY` | 翻页可能重复或漏行 | 加唯一决胜键 |
| `BETWEEN '2025-03-01' AND '2025-03-31'` 查时间戳 | 漏掉最后一天 | 左闭右开：`>= 起 AND < 次日` |
| `SELECT DISTINCT major, name` 期望"每个专业一行" | `DISTINCT` 作用于整行 | 用 `GROUP BY` + 聚合 |
| `!=` 用于所有数据库 | 不是标准写法 | 用 `<>` |
| 依赖 `ORDER BY` 里 NULL 的位置 | 标准未规定，各库不同 | 显式 `NULLS FIRST/LAST` 或 `CASE` |

## 小结

- **逻辑执行顺序是 `FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`**，与书写顺序不同。由此推出：`WHERE` 里不能用聚合函数、通常不能用 `SELECT` 别名；`ORDER BY` 里可以用别名。
- `NULL` 表示"未知"，用**三值逻辑**（TRUE / FALSE / UNKNOWN）。**`WHERE` 只保留结果为 TRUE 的行**，所以 `city <> '厦门'` 也会丢掉 `city IS NULL` 的行。
- 判断 `NULL` 只能用 `IS NULL` / `IS NOT NULL`；`= NULL` 永远为 UNKNOWN。
- **`NOT IN` 的子查询里若含 `NULL`，整个查询返回空结果**（因为 `x <> NULL` 是 UNKNOWN）。需要"排除"语义时用 `NOT EXISTS` 更安全。
- **混用 `AND` 与 `OR` 必须加括号**——`AND` 优先级更高，不加括号的错误不会报错，只会静默返回错误结果。
- `BETWEEN` 是闭区间且不匹配 `NULL`；**处理时间戳范围要警惕漏掉最后一天**，用左闭右开 `>= 起 AND < 次日` 最稳。
- `LIKE '%x%'` 以通配符开头时用不上普通索引；`LIKE 'x%'` 可以。
- `COALESCE` 用来提供默认值，`NULLIF` 常用来防止除零（`NULLIF(COUNT(*), 0)`）。
- 表与结果集**没有固有顺序**。任何依赖顺序的场景都要 `ORDER BY`；**分页的排序键必须唯一**（加主键决胜），否则会重复或漏行。
- `NULL` 在 `ORDER BY` 中的位置**标准未规定**（PostgreSQL 视为最大、MySQL 与 SQLite 视为最小）。跨数据库时用 `CASE` 显式控制。
- 深分页用 `OFFSET` 会随偏移量线性变慢；**keyset 分页**（记住上一页的排序键）是正解，但只能顺序翻页。
- `DISTINCT` 作用于**结果集的整行**，不是对每列分别去重；所以列里有高区分度字段时 `DISTINCT` 几乎不起作用。需要"每个分组取一行"要用 `GROUP BY` + 聚合或窗口函数。
- 聚合函数对 `NULL` 的行为是本篇与下一篇的衔接点：**`COUNT(*)` 数所有行，`COUNT(col)` 跳过 NULL，`AVG(col)` 的分母是非 NULL 行数** —— 详见 [03-aggregation.md](03-aggregation.md)。
