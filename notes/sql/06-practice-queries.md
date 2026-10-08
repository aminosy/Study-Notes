# SQL 练习题：10 道由易到难（含参考答案）

本篇提供 10 道覆盖整个 SQL 板块知识点的练习题、完整的示例数据与逐题参考答案，每题都附"思路"与"易错点"分析。

## 0. 怎么用这篇

1. **先只看题目**，自己在数据库里写一遍；
2. 再对照参考答案，**不要一开始就看**——看答案学不到东西；
3. 重点看每题下面的 **易错点**，那些才是真正区分"会"与"不会"的地方；
4. 如果结果与预期不符，先看是不是 `NULL` 的问题（这套数据里埋了两处 `NULL`）。

## 1. 示例数据库

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

### 1.1 示例数据

```sql
INSERT INTO teachers (teacher_id, teacher_name, department) VALUES
    (1, '黄志强', '计算机系'),
    (2, '林美玲', '计算机系'),
    (3, '吴建国', '数学系'),
    (4, '郑晓月', '软件工程系');

INSERT INTO students (student_id, name, gender, birth_date, major, city, enroll_year) VALUES
    (1, '王小明', 'M', '2004-03-15', '计算机科学与技术', '厦门', 2022),
    (2, '李雨桐', 'F', '2004-07-22', '计算机科学与技术', '北京', 2022),
    (3, '张浩然', 'M', '2003-11-08', '软件工程',         '厦门', 2021),
    (4, '陈思远', 'M', '2005-01-30', '计算机科学与技术', '上海', 2023),
    (5, '刘悦然', 'F', '2004-09-12', '数据科学',         NULL,   2022),
    (6, '赵子涵', 'F', NULL,         '软件工程',         '厦门', 2023),
    (7, '钱景行',  'M', '2004-05-18', '数据科学',         '广州', 2023);   -- 没有任何选课记录

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

### 1.2 刻意埋下的"陷阱数据"

这套数据里有六处刻意设计，它们决定了某些题目的正确答案，**做题前先看一眼这份清单**：

| # | 陷阱数据 | 影响哪些题 | 说明 |
| --- | --- | --- | --- |
| 1 | **学生 7（钱景行）没有任何选课记录** | 题 5、题 8 | 是"找出没选课的学生"的答案。**注意：学生 6（赵子涵）是有选课记录的（101 号课 59.0 分）**，不要把他误当成"没选课的人" |
| 2 | **课程 106（软件工程导论）没有任何人选** | 题 6、题 7 | 用于验证 `LEFT JOIN` 的方向；也用于检查"没人选的课"是否被正确处理 |
| 3 | **教师 4（郑晓月）没有任何课** | （进阶思考） | 三表连接的缺口 |
| 4 | **`enrollments` 第 5 行的 `score` 是 `NULL`**（李雨桐的数据库课成绩未出） | 题 6、题 7、题 8 | 区分 `COUNT(*)` 与 `COUNT(score)`；区分 `AVG` 的分母 |
| 5 | **`enrollments` 第 11 行的 `score` 是 `0`**（刘悦然的操作系统成绩 0 分） | 题 6、题 7、题 8 | **`0` ≠ `NULL`**！0 分是有效成绩，必须被计入平均分且算作不及格 |
| 6 | **学生 5（刘悦然）的 `city` 是 `NULL`；学生 6（赵子涵）的 `birth_date` 是 `NULL`** | 题 3、题 9 | 演示"NULL 不会被任何比较匹配" |
| 7 | **学生 6（赵子涵）的 59.0 分是唯一的 F 档成绩**，且他是**唯一"有成绩但只有一条"的学生** | 题 8、题 9、题 9(c) | 用于验证 F 档计数，以及"有选课但成绩很少"的学生在平均分比较中的位置 |

**第 4、5 条是最关键的一对**：如果题目要求"平均分"，你要明确"NULL 算不算分母里的一个"；如果要求"不及格"，`0` 分必须算不及格而 `NULL` 不算。**这两行数据的区别就是本练习最有价值的地方。**

## 2. 题目

### 题目 1（易）—— 单表筛选 + 多列排序 + `LIMIT`

**要求**：列出所有"计算机科学与技术"专业的学生，输出学号、姓名、入学年份、城市。
按入学年份**降序**、姓名**升序**排列，只取前 3 条。

**提示**：注意多列排序时每列都要写方向（或接受默认的 `ASC`）。

---

### 题目 2（易）—— `DISTINCT`

**要求**：
(a) 列出 `enrollments` 里出现过的所有学期，每个学期只显示一次，按学期降序。
(b) 列出 `students` 里出现过的所有专业，每个专业只显示一次。

**提示**：想清楚 `DISTINCT` 是作用于整行还是作用于单列。

---

### 题目 3（易-中）—— `LIKE` / `BETWEEN` / `IN` / `NULL` 的组合

**要求**：找出满足以下**全部**条件的学生，输出学号、姓名、出生日期、城市：
- 出生日期在 2004-01-01 到 2004-12-31 之间（含端点）；
- 城市是"厦门"或"北京"；
- 姓名里含有"雨"字**或**姓名以"王"开头。

**提示**：混用 `OR` 和 `AND` 时必须加括号。另外想一下：出生日期为 `NULL` 的学生会出现吗？为什么？

---

### 题目 4（中）—— `INNER JOIN`

**要求**：查询每位学生的选课明细，输出学生姓名、课程名称、成绩、学期。
只包含**有选课记录**的学生。按学生姓名升序、成绩降序排列。

**提示**：需要连接三张表。

---

### 题目 5（中）—— `LEFT JOIN` 反连接（找"缺失"）

**要求**：找出**没有选过任何课**的学生，输出学号、姓名、专业。

**提示**：有两种写法（`LEFT JOIN ... IS NULL` 和 `NOT EXISTS`），两种都写一遍，并说明各自的特点。

---

### 题目 6（中）—— `GROUP BY` + `HAVING`

**要求**：统计每门课的选课情况，输出课程号、课程名、选课人数、平均分。
条件：
- **只统计已出成绩的记录**（`score` 不为 `NULL`）；
- 只保留"选课人数 ≥ 2"的课程；
- 按平均分降序排列。

**提示**：过滤人数和平均分是"组级条件"（`HAVING`），过滤"未出成绩"是"行级条件"（`WHERE`）。

---

### 题目 7（中-难）—— 聚合与 `NULL` 的完整处理

**要求**：为**每一门课**（包括没人选的课）输出一行，包含：
- 课程号、课程名；
- 总记录数（`COUNT(*)` 的含义下，选课记录条数）；
- 已出成绩的记录数；
- 未出成绩的记录数；
- 平均分（只按已出成绩的记录算，保留 2 位小数）；
- 最高分。

对于没人选的课程，人数类字段应显示 `0`，平均分与最高分显示 `NULL`。

**提示**：这道题的关键是"没人选的课也要出现"（用 `LEFT JOIN`），以及"用 `COUNT` 的哪个变体"才能把没人选的课正确算成 0。

---

### 题目 8（难）—— 相关子查询 / `EXISTS` / `NOT EXISTS`

**要求**：
(a) 找出**至少有一门课成绩高于 90 分**（含 90）的学生，输出学号、姓名、专业。
(b) 找出**有选课记录但没有任何不及格（< 60）记录**的学生，输出学号、姓名。

**提示**：
- (a) 用 `EXISTS` 或 `IN` 都可以。
- (b) 要注意两件事：**"没有选课记录"的学生应该被排除**（因为题目要求"有选课记录"）；**`score` 为 `NULL` 不算不及格**。

---

### 题目 9（难）—— CTE + 分档统计

**要求**：
(a) 用 `CASE WHEN` 把成绩分档：`>= 90` 为 `A`，`80-89` 为 `B`，`70-79` 为 `C`，`60-69` 为 `D`，`< 60` 为 `F`，`NULL` 为 `未出成绩`。
(b) 统计每个档位有多少条记录，输出档位、人数，按档位字母顺序排列（`未出成绩` 排在最后）。
(c) 再用一个 CTE 先算出"每位学生的平均分"，然后找出**平均分高于全体平均分**的学生，输出学号、姓名、平均分。

**提示**：`CASE WHEN` 必须**显式处理 `NULL`**，否则 `NULL` 会掉进 `ELSE`；档位排序要自定义顺序，不能直接按字母排（因为"未出成绩"不是普通字母）。

---

### 题目 10（难）—— 窗口函数 / 递归 CTE 综合

**从下面两个方向中任选一个完成**（也可以两个都做）：

**(a) 窗口函数**：找出**每门课成绩前 2 名**的学生（含并列，即用 `RANK`）。输出课程名、学生姓名、成绩、该课内排名。

**(b) 递归 CTE**：先建一张课程先修关系表：

```sql
CREATE TABLE course_prereq (
    course_id        INTEGER NOT NULL,
    prereq_course_id INTEGER NOT NULL,
    PRIMARY KEY (course_id, prereq_course_id)
);

INSERT INTO course_prereq VALUES
    (102, 101),      -- 数据结构 需要 程序设计基础
    (103, 102),      -- 数据库系统原理 需要 数据结构
    (104, 102),      -- 操作系统 需要 数据结构
    (105, 101),      -- 离散数学 需要 程序设计基础
    (106, 103);      -- 软件工程导论 需要 数据库系统原理
```

用它回答："要修 106 号课（软件工程导论），需要依次先修完哪些课？"输出层级与课程名。

**提示**：
- (a) 窗口函数的结果不能在 `WHERE` 里直接过滤，需要包一层（CTE 或派生表）。
- (b) 递归 CTE 由锚点、`UNION ALL`、递归成员三部分组成；从 106 往上找先修课。

---

## 3. 参考答案

### 答案 1

```sql
SELECT student_id, name, enroll_year, city
FROM students
WHERE major = '计算机科学与技术'
ORDER BY enroll_year DESC, name ASC
LIMIT 3;
```

**思路**：单表查询，`WHERE` 过滤专业，`ORDER BY` 指定两个排序键（各自带方向），`LIMIT` 截取。

**易错点**：
- `ORDER BY enroll_year DESC, name ASC` 里的第二个方向不能省（省略默认 `ASC`，这里恰好是想要的，但写出来更清楚）。
- **`LIMIT` 必须有 `ORDER BY` 才有确定的意义**。没有 `ORDER BY` 时数据库返回哪 3 行是不确定的。
- 严格说，`enroll_year` 相同的学生在 `name` 上还要能唯一确定顺序；如果姓名也相同（本数据里没有），顺序仍不确定。**分页时应该加主键作为决胜键**，这里因为只是取前 3 条且数据量小，影响不大。

---

### 答案 2

```sql
-- (a) 出现过的所有学期
SELECT DISTINCT semester
FROM enrollments
ORDER BY semester DESC;

-- (b) 出现过的所有专业
SELECT DISTINCT major
FROM students
ORDER BY major;
```

**思路**：`DISTINCT` 对结果集的整行去重；单列查询时就是对该列去重。

**易错点**：
- **(a) 的结果里可能包含 `NULL`**（如果 `semester` 允许为空且有这样的行）。在本数据里 `semester` 都有值，所以不会出现；但实际数据里要考虑"要不要排除 NULL"。
- **(b) 的结果里有 3 个专业**（计算机科学与技术、软件工程、数据科学）。注意"数据科学"有 2 个学生（刘悦然、钱景行），但 `DISTINCT` 只列一次——`DISTINCT` 不关心出现次数。
- 如果写成 `SELECT DISTINCT major, name`，那去重的是 **(专业, 姓名) 组合**，因为姓名各不相同，结果就是 7 行（等于学生总数）。**这是"`DISTINCT` 没起到作用"的典型原因。**

---

### 答案 3

```sql
SELECT student_id, name, birth_date, city
FROM students
WHERE birth_date BETWEEN '2004-01-01' AND '2004-12-31'
  AND city IN ('厦门', '北京')
  AND (name LIKE '%雨%' OR name LIKE '王%')
ORDER BY student_id;
```

**结果**：学生 1（王小明）与学生 2（李雨桐）。

逐个学生验证：

| 学生 | 生日在范围内 | 城市符合 | 姓名符合 | 通过？ |
| --- | --- | --- | --- | --- |
| 1 王小明 | 2004-03-15 ✅ | 厦门 ✅ | `王%` ✅ | ✅ |
| 2 李雨桐 | 2004-07-22 ✅ | 北京 ✅ | `%雨%` ✅ | ✅ |
| 3 张浩然 | 2003-11-08 ❌ | 厦门 ✅ | ❌ | ❌ |
| 4 陈思远 | 2005-01-30 ❌ | 上海 ❌ | ❌ | ❌ |
| 5 刘悦然 | 2004-09-12 ✅ | **NULL** ❌ | ❌ | ❌ |
| 6 赵子涵 | **NULL** ❌ | 厦门 ✅ | ❌ | ❌ |
| 7 钱景行 | 2004-05-18 ✅ | 广州 ❌ | ❌ | ❌ |

**注意张浩然（2003-11-08）与陈思远（2005-01-30）都在 2004 年之外**，所以被 `BETWEEN` 排除；学生 7（钱景行）虽然生日在范围内，但城市"广州"不在列表里；刘悦然与赵子涵则分别因为 `city` 和 `birth_date` 是 `NULL` 而被排除（见下面易错点第二条）。

**易错点**：
- **括号不能省**：如果不写 `(name LIKE '%雨%' OR name LIKE '王%')`，因为 `AND` 优先级高于 `OR`，条件会被解析成
  `(birth AND city AND name LIKE '%雨%') OR (name LIKE '王%')`，于是**所有姓王的学生无论生日城市都会被选中**。这个错误不会报错，只会静默返回多出来的行。
- **`NULL` 不会被任何比较匹配**：
  - 学生 5 的 `city` 是 `NULL`，`NULL IN ('厦门','北京')` 是 `UNKNOWN`，被 `WHERE` 过滤；
  - 学生 6 的 `birth_date` 是 `NULL`，`NULL BETWEEN ...` 是 `UNKNOWN`，同样被过滤。
  **所以这两行都不会出现在结果里**。如果业务上需要包含"生日未知"的学生，必须显式写 `OR birth_date IS NULL`。
- `BETWEEN '2004-01-01' AND '2004-12-31'` 对 `DATE` 类型是安全的（不含时间部分）。**但如果列是 `TIMESTAMP`，这样写就会漏掉 12-31 当天有时间的记录**，那时应该改用左闭右开：`>= '2004-01-01' AND < '2005-01-01'`。

---

### 答案 4

```sql
SELECT s.name          AS student_name,
       c.course_name   AS course_name,
       e.score,
       e.semester
FROM enrollments e
JOIN students s ON s.student_id = e.student_id
JOIN courses  c ON c.course_id  = e.course_id
ORDER BY s.name ASC, e.score DESC;
```

**思路**：`enrollments` 是关联表，从它出发连接 `students` 和 `courses`。用 `INNER JOIN` 因为我们只要"有选课记录"的行。

**结果行数 = 12**（等于 `enrollments` 的总行数）。注意学生 7（钱景行）**不会出现**——他没有选课记录，`INNER JOIN` 会把他排除。这正是下一题要专门找出来的人。

**易错点**：
- **连接列不能搞混**：`s.student_id = e.student_id` 和 `c.course_id = e.course_id`。写反了（例如 `s.student_id = c.course_id`）不会报错但结果完全错误——这是最容易犯的错误之一，因为 `student_id` 和 `course_id` 都是整数，类型检查通不出问题。
- **排序里的 `NULL`**：`ORDER BY e.score DESC` 时，李雨桐的 `NULL` 成绩排在哪里取决于数据库（PostgreSQL 里 `DESC` 时 NULL 在最前，MySQL/SQLite 里在最后）。要确定顺序就显式处理：
  ```sql
  ORDER BY s.name ASC, (e.score IS NULL), e.score DESC   -- 让 NULL 排最后
  ```
- 输出列名用了 `AS` 别名，让结果更可读。

---

### 答案 5

```sql
-- 写法 A：LEFT JOIN 反连接
SELECT s.student_id, s.name, s.major
FROM students s
LEFT JOIN enrollments e ON e.student_id = s.student_id
WHERE e.enrollment_id IS NULL;            -- 用右表的【主键】列判断

-- 写法 B：NOT EXISTS（推荐）
SELECT s.student_id, s.name, s.major
FROM students s
WHERE NOT EXISTS (
    SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id
);
```

**结果**：**只有学生 7（钱景行，数据科学）**。

**这里有一个必须小心的点**：很多人会凭印象把学生 6（赵子涵）也当成"没选课的人"。**但他是有选课记录的**——`enrollments` 第 12 行是他的 101 号课成绩 59.0 分。如果他出现在结果里，说明你的连接条件或 `IS NULL` 判断的列写错了。

**两种写法的对比**：

| | 写法 A（`LEFT JOIN ... IS NULL`） | 写法 B（`NOT EXISTS`） |
| --- | --- | --- |
| 可读性 | 需要读者理解"补 NULL 再过滤" | **直接表达"不存在"** |
| 是否可能重复 | ⚠️ 如果右表对同一左表行有多行匹配，会返回多行（本数据不会） | ✅ 天然只返回一行 |
| `NULL` 安全性 | 需要选对"判断 IS NULL 的列" | ✅ 不受 NULL 影响 |
| 优化器处理 | 通常变成 anti-join | 通常变成 anti-join |

**为什么必须用 `e.enrollment_id`（主键）而不是随便一列**：`WHERE e.some_column IS NULL` 只有在"该列在 `enrollments` 里一定非空"时才等价于"没有匹配的行"。用**主键**（隐含 `NOT NULL`）是最稳妥的。如果误用一个可空列（例如 `e.score`），后果分两种情况：
- 对**学生 7**（完全没有选课记录），那一行 `e.score` 确实是 `NULL`，所以他仍会被正确选出；
- 但对**李雨桐**（有记录、但 103 号课成绩未出）这样"有记录而成绩为 NULL"的行，`e.score IS NULL` 也为真，于是**她会被错误地算进结果**。

在这个数据集里，如果把判断列换成 `e.score`，结果会**多出学生 2（李雨桐）**，本题答案就错了。**这是一个非常隐蔽的 bug，值得记住：反连接要用右表的非空列（最好是主键）判断。**

**易错点**：
- **绝对不能写成 `WHERE s.student_id IS NULL`**（判断左表的列），那样永远没有结果。
- 不能用 `NOT IN` 而不用 `IS NOT NULL` 保护：
  ```sql
  -- ⚠️ 危险
  WHERE s.student_id NOT IN (SELECT student_id FROM enrollments)
  -- 如果 enrollments.student_id 里有一行 NULL，这个查询返回【空集】
  ```
  本数据里 `enrollments.student_id` 都有值，所以恰好安全；但**依赖"数据里恰好没有 NULL"是危险的**。

---

### 答案 6

```sql
SELECT c.course_id,
       c.course_name,
       COUNT(e.enrollment_id)          AS student_count,   -- 只数已出成绩的记录
       ROUND(AVG(e.score), 2)          AS avg_score
FROM enrollments e
JOIN courses c ON c.course_id = e.course_id
WHERE e.score IS NOT NULL                        -- 行级过滤：只用已出成绩的记录
GROUP BY c.course_id, c.course_name
HAVING COUNT(e.enrollment_id) >= 2               -- 组级过滤：人数
ORDER BY avg_score DESC;
```

**思路**：`WHERE` 先丢掉"未出成绩"的行（行级条件），再按课程分组，最后用 `HAVING` 过滤人数。

**逐个课程验证**：

| 课程 | 选课记录 | 已出成绩的记录 | 人数 ≥ 2？ | 平均分（仅已出成绩） |
| --- | --- | --- | --- | --- |
| 101 程序设计基础 | 学生 1,2,3,6 | 4 条 | ✅ | (88.5+95.5+72.0+59.0)/4 = 78.75 |
| 102 数据结构 | 学生 1,4 | 2 条 | ✅ | (92.0+91.0)/2 = 91.50 |
| 103 数据库系统原理 | 学生 1,2,5 | 2 条（学生 2 的为 NULL） | ✅ | (79.0+88.0)/2 = 83.50 |
| 104 操作系统 | 学生 4,5 | 2 条 | ✅ | (67.5+0)/2 = 33.75 |
| 105 离散数学 | 学生 3 | 1 条 | ❌ 被 HAVING 过滤 | — |
| 106 软件工程导论 | 无 | 0 条 | ❌ | — |

**结果**：101、102、103、104 四门课，按平均分降序为 102（91.50）、103（83.50）、101（78.75）、104（33.75）。

**易错点**：
- **`WHERE e.score IS NOT NULL` 与 `HAVING` 的分工不能搞混**。"只用已出成绩的记录"改变的是**参与统计的数据**，属于行级过滤（`WHERE`）；"人数 ≥ 2"是对**统计结果**的过滤（`HAVING`）。
- **`0` 分必须被计入平均分**：104 号课的平均分是 `(67.5 + 0) / 2 = 33.75`，而不是"只有 67.5 一个人所以平均 67.5"。**`0` 是有效成绩，不是 `NULL`。** 这是本套数据里最重要的一个陷阱。
- **`AVG(e.score)` 的分母是"已出成绩的记录数"**，因为 `WHERE` 已经把 NULL 过滤掉了。如果改成 `SUM(e.score) / COUNT(*)`，在过滤后两者恰好相同；但如果不写 `WHERE`，两者就会不同。**要清楚自己的分母是什么。**
- **这里用了 `JOIN` 而不是 `LEFT JOIN`**：因为 `HAVING COUNT >= 2` 会把"没人选的课"过滤掉，用 `JOIN` 更直接。如果要保留没人选的课，就得用 `LEFT JOIN`（见下一题）。
- `GROUP BY` 里同时写了 `c.course_id` 和 `c.course_name`：在 PostgreSQL 上只写主键 `c.course_id` 也合法（函数依赖推断），但显式写更可移植。

---

### 答案 7

```sql
SELECT c.course_id,
       c.course_name,
       COUNT(e.enrollment_id)                                   AS total_records,
       COUNT(e.score)                                            AS scored_count,
       COUNT(e.enrollment_id) - COUNT(e.score)                   AS missing_count,
       ROUND(AVG(e.score), 2)                                    AS avg_score,
       MAX(e.score)                                              AS max_score
FROM courses c
LEFT JOIN enrollments e ON e.course_id = c.course_id       -- LEFT：没人选的课也要出现
GROUP BY c.course_id, c.course_name
ORDER BY c.course_id;
```

**思路**：`LEFT JOIN` 保证"没人选的课"也有一行（`e` 的列全为 NULL），然后用 `COUNT` 指定**右表的列**（这里用 `e.enrollment_id`，因为它是主键、必然非空），把 NULL 行正确地算成 0。

**逐个课程验证**：

| 课程 | total_records<br>`COUNT(e.enrollment_id)` | scored_count<br>`COUNT(e.score)` | missing_count | avg_score | max_score |
| --- | --- | --- | --- | --- | --- |
| 101 程序设计基础 | 4 | 4 | 0 | 78.75 | 95.5 |
| 102 数据结构 | 2 | 2 | 0 | 91.50 | 92.0 |
| 103 数据库系统原理 | 3 | 2 | 1 | 83.50 | 88.0 |
| 104 操作系统 | 2 | 2 | 0 | 33.75 | 67.5 |
| 105 离散数学 | 1 | 1 | 0 | 85.00 | 85.0 |
| 106 软件工程导论 | **0** | **0** | 0 | **NULL** | **NULL** |

**易错点（这是本篇最关键的一题）**：

- **绝对不能用 `COUNT(*)`**：因为用了 `LEFT JOIN`，没人选的课（106）也会产生一行（`e` 的列全为 NULL），`COUNT(*)` 会把它数成 **1**。必须用 `COUNT(e.enrollment_id)` 或其他右表的非空列：
  ```sql
  COUNT(*)                 AS wrong_count,   -- 106 号课会显示 1（错！）
  COUNT(e.enrollment_id)   AS right_count    -- 106 号课显示 0（对）
  ```
  **"`LEFT JOIN` 之后计数必须用右表的列"** 是必须记住的一条。

- **"未出成绩的记录数"要用两个 `COUNT` 相减**：`COUNT(e.enrollment_id) - COUNT(e.score)`。
  - 不能写 `COUNT(e.score IS NULL)`：因为 `score IS NULL` 返回的是 `TRUE`/`FALSE`（永不返回 NULL），所以 `COUNT` 会数**所有行**；
  - 也可以写 `SUM(CASE WHEN e.score IS NULL AND e.enrollment_id IS NOT NULL THEN 1 ELSE 0 END)`，但要额外排除"没人选的课"那一行（否则 106 号课会显示 1 个"未出成绩"，这也是错的）。

- **106 号课的 `avg_score` 与 `max_score` 是 `NULL`**（因为 `AVG`/`MAX` 在无行时返回 NULL），这是符合题目要求的。如果业务上要显示 0，用 `COALESCE(AVG(e.score), 0)`——**但要先想清楚"没人选的课平均分是 0"在业务上是否合理**（通常 `NULL` 更合适，因为"没有成绩"和"平均 0 分"是两回事）。

- **`0` 分被正确计入**：104 号课的 `avg_score` 是 33.75，说明那道 0 分被算进去了。同时 `scored_count` 是 2（不是 1），说明 0 分被当作"已出成绩"。

---

### 答案 8

**(a) 至少有一门课成绩 ≥ 90 的学生**

```sql
SELECT s.student_id, s.name, s.major
FROM students s
WHERE EXISTS (
    SELECT 1 FROM enrollments e
    WHERE e.student_id = s.student_id
      AND e.score >= 90
)
ORDER BY s.student_id;

-- 等价写法（用 IN）
SELECT student_id, name, major
FROM students
WHERE student_id IN (
    SELECT student_id FROM enrollments WHERE score >= 90
)
ORDER BY student_id;
```

**结果**：学生 1（92.0）、2（95.5）、4（91.0）三人。

**为什么这里 `IN` 也是安全的**：子查询里有 `WHERE score >= 90`，而 `NULL >= 90` 是 UNKNOWN 不会满足条件，所以子查询**不可能返回 NULL 的 `student_id`**（前提是 `enrollments.student_id` 本身非空，本数据满足）。**所以用 `IN` 恰好安全——但要能说出"为什么安全"，而不是碰巧对。**

---

**(b) 有选课记录但没有不及格记录的学生**

```sql
SELECT s.student_id, s.name
FROM students s
WHERE EXISTS (                                    -- 条件 1：有选课记录
        SELECT 1 FROM enrollments e
        WHERE e.student_id = s.student_id
      )
  AND NOT EXISTS (                                -- 条件 2：没有不及格记录
        SELECT 1 FROM enrollments e
        WHERE e.student_id = s.student_id
          AND e.score IS NOT NULL                 -- 排除未出成绩
          AND e.score < 60
      )
ORDER BY s.student_id;
```

**结果**：学生 1、2、3、4、5。**学生 6（赵子涵）被他自己的 59.0 分排除了**（0 < 60 之外，59 < 60 也是不及格）；**学生 7（钱景行）被"必须有选课记录"这个条件排除**。

逐个验证：

| 学生 | 有选课？ | 有 < 60 的成绩？ | 结果 |
| --- | --- | --- | --- |
| 1 王小明 | ✅ (88.5, 92.0, 79.0) | ❌ | ✅ 通过 |
| 2 李雨桐 | ✅ (95.5, NULL) | ❌（NULL 不算不及格） | ✅ 通过 |
| 3 张浩然 | ✅ (72.0, 85.0) | ❌ | ✅ 通过 |
| 4 陈思远 | ✅ (91.0, 67.5) | ❌ | ✅ 通过 |
| 5 刘悦然 | ✅ (88.0, **0**) | ✅ **0 < 60** | ❌ **被排除！** |
| 6 赵子涵 | ✅ (59.0) | ✅ **59 < 60** | ❌ **被排除！** |
| 7 钱景行 | ❌ **没有选课** | — | ❌ 被排除 |

**注意学生 5（刘悦然）和学生 6（赵子涵）都被排除了**：
- 刘悦然有 0 分，而 `0 < 60` 为真——**如果误把 0 分当成"没有成绩"（NULL），就会错误地把她算成通过**；
- 赵子涵只有一条 59.0 分，`59 < 60` 也为真，所以同样被排除。

**这两个人是这道题的两个不同方向的考验**：一个考"0 分不是 NULL"，一个考"只有一条不及格记录也算不及格"。

**易错点（这道题的三个坑）**：

1. **必须加 `EXISTS` 要求"有选课记录"**。如果不加，学生 7（钱景行，从没选课）会因为"不存在不及格记录"而**被错误地算作通过**——这是 **"空集为真 (vacuously true)"** 的经典陷阱：
   ```sql
   -- ❌ 少了 EXISTS 这一半，学生 7 会被错误包含
   SELECT s.student_id, s.name FROM students s
   WHERE NOT EXISTS (
       SELECT 1 FROM enrollments e
       WHERE e.student_id = s.student_id AND e.score < 60
   );
   -- 结果会包含学生 7（钱景行）—— 但"从没考过试"怎么能算"都及格了"？
   ```

2. **`e.score IS NOT NULL` 不能省**（虽然在这个具体查询里它不影响结果）。`NULL < 60` 是 `UNKNOWN`，本来就不会满足条件，所以不加也对。**但写出来能让意图清晰**，也让读者不用自己去推 `NULL` 的行为。

3. **绝不能用 `NOT IN` 而不保护 NULL**：
   ```sql
   -- ⚠️ 危险写法
   WHERE s.student_id NOT IN (
       SELECT e.student_id FROM enrollments e WHERE e.score < 60
   )
   -- 这个子查询恰好不会返回 NULL（score < 60 排除了 NULL 行），所以能用；
   -- 但如果条件改成不带 score 过滤，一旦子查询返回 NULL，结果就是空集。
   ```

---

### 答案 9

**(a) 分档**（先用 CTE 算出一列，再分组 —— 推荐写法）

```sql
WITH graded AS (
    SELECT enrollment_id,
           CASE
               WHEN score IS NULL THEN '未出成绩'     -- 【必须放最前】显式处理 NULL
               WHEN score >= 90    THEN 'A'
               WHEN score >= 80    THEN 'B'
               WHEN score >= 70    THEN 'C'
               WHEN score >= 60    THEN 'D'
               ELSE 'F'
           END AS grade_band
    FROM enrollments
)
SELECT grade_band, COUNT(*) AS n
FROM graded
GROUP BY grade_band
ORDER BY CASE grade_band
             WHEN 'A' THEN 1
             WHEN 'B' THEN 2
             WHEN 'C' THEN 3
             WHEN 'D' THEN 4
             WHEN 'F' THEN 5
             ELSE 6                              -- '未出成绩' 排最后
         END;
```

**结果**：

| grade_band | n | 明细 |
| --- | --- | --- |
| A | 3 | 92.0, 95.5, 91.0 |
| B | 3 | 88.5, 85.0, 88.0 |
| C | 2 | 79.0, 72.0 |
| D | 1 | 67.5 |
| F | 2 | **0**, 59.0 |
| 未出成绩 | 1 | NULL |

**合计 3 + 3 + 2 + 1 + 2 + 1 = 12 ✅**（与 `enrollments` 的总行数一致——这是一个很好的自检：分档统计的各档人数之和必须等于总记录数）。

注意**学生 7（钱景行）不会出现在这张表里**，因为他没有任何成绩记录——"未出成绩"这一档统计的是**有记录但成绩为 NULL** 的情况（李雨桐），而不是"没有记录"。**这两者不是一回事**，如果需要统计"没选课的人"，那是另一个查询（见题 5）。

逐条记录核对：

| enrollment_id | score | 档位 |
| --- | --- | --- |
| 1 | 88.5 | B |
| 2 | 92.0 | A |
| 3 | 79.0 | C |
| 4 | 95.5 | A |
| 5 | **NULL** | 未出成绩 |
| 6 | 72.0 | C |
| 7 | 85.0 | B |
| 8 | 91.0 | A |
| 9 | 67.5 | D |
| 10 | 88.0 | B |
| 11 | **0** | F |
| 12 | 59.0 | F |

**易错点**：
- **`WHEN score IS NULL` 必须写在最前面**（或者至少写在所有数值比较之前）。因为 `NULL >= 90` 是 `UNKNOWN` 不是 `TRUE`，所有数值分支都不会命中，`NULL` 会掉进 `ELSE` 被当成 `F`。**如果 `CASE` 里没有 `ELSE`，`NULL` 行的结果是 `NULL`，会单独形成一个 `NULL` 分组——看起来"对"了，但组名是 `NULL` 而不是"未出成绩"。**
- **档位排序不能直接 `ORDER BY grade_band`**：按字母序 `A, B, C, D, F, 未出成绩` 恰好是想要的顺序，但"未出成绩"的中文字符在排序中可能排在字母之前（取决于排序规则）。**用 `CASE` 映射成数字最可靠。**
- 另一种写法是在 `GROUP BY` 里重复整个 `CASE` 表达式（标准 SQL 要求），但**可读性差且容易改漏一处**。用 CTE 先算出一列是更好的做法（也正好演示 CTE 的价值）。

---

**(c) 找出平均分高于全体平均分的学生**

```sql
WITH per_student AS (
    -- 第 1 步：每位学生的平均分（只按已出成绩的记录算）
    SELECT student_id,
           AVG(score) AS avg_score,
           COUNT(score) AS scored_count
    FROM enrollments
    WHERE score IS NOT NULL
    GROUP BY student_id
),
overall AS (
    -- 第 2 步：全体平均分（引用上一个 CTE）
    SELECT AVG(avg_score) AS overall_avg FROM per_student
)
-- 第 3 步：筛选
SELECT s.student_id,
       s.name,
       ROUND(ps.avg_score, 2) AS avg_score,
       ps.scored_count
FROM per_student ps
JOIN overall o ON ps.avg_score > o.overall_avg     -- 用 ON 做过滤也可以
JOIN students s ON s.student_id = ps.student_id
ORDER BY ps.avg_score DESC;
```

**先算每个学生的平均分**：

| 学生 | 已出成绩 | 平均分 |
| --- | --- | --- |
| 1 王小明 | 88.5, 92.0, 79.0 | 86.50 |
| 2 李雨桐 | 95.5（103 号为 NULL） | 95.50 |
| 3 张浩然 | 72.0, 85.0 | 78.50 |
| 4 陈思远 | 91.0, 67.5 | 79.25 |
| 5 刘悦然 | 88.0, **0** | 44.00 |
| 6 赵子涵 | 59.0 | 59.00 |

（**学生 7（钱景行）不在上表里**——他没有成绩，`GROUP BY student_id` 不会为他产生任何分组。这是"没有记录"与"记录里成绩为 NULL"的又一个区别。）

**全体平均分** = (86.50 + 95.50 + 78.50 + 79.25 + 44.00 + 59.00) / 6 = 442.75 / 6 ≈ **73.79**

**结果**：平均分高于 73.79 的学生 —— 王小明（86.50）、李雨桐（95.50）、张浩然（78.50）、陈思远（79.25）。

**注意分母是 6 而不是 7**：学生 7 因为没有成绩记录，根本不参与 `per_student` 这个 CTE，所以"全体平均分"是在**有成绩的 6 个学生**之间算的。如果业务上要求"7 个学生都算，没成绩的按 0 分计"，结果会完全不同。**这是做这类统计时最需要先和需求确认的地方。**

**另外这里有两种"全体平均分"的口径**：

| 口径 | 计算方式 | 值 |
| --- | --- | --- |
| **学生的平均分的均值** | `AVG(per_student.avg_score)` | 442.75 / 6 ≈ 73.79 |
| **所有成绩的算术平均** | `AVG(score)` 直接对 11 条记录 | 817.5 / 11 ≈ 74.32 |

**这两个值不同**（73.79 vs 74.32），因为它们对"学生"的加权方式不同：前者每个学生等权（不管他考了几门），后者每条成绩等权（考得多的学生影响更大）。**这是一道题目表述不够严谨的地方**——"全体平均分"必须明确是哪种口径。我在查询里用的是第一种（每生等权），因为题目是"平均分高于全体平均分的学生"，比较的单位是"学生"。

**易错点**：
- **必须明确"平均分"的分母口径**（这个坑在上面已经具体化了，实际工作中一定要和需求方确认）。
- **不能直接写 `HAVING AVG(score) > AVG(score)`**：这两个 `AVG` 在同一个分组上下文里是同一个值，条件永远为假。**"组内平均"和"全体平均"是两个不同层级的聚合**，必须用子查询/CTE 把全体平均算出来。
- **CTE 的顺序有意义**：`overall` 引用了 `per_student`，所以 `per_student` 必须写在前面。后面的 CTE 可以引用前面的，反之不行。
- **`COUNT(score)` 而不是 `COUNT(*)`**：用来显示"这个学生有几门成绩是有效的"，`COUNT(*)` 会把未出成绩的记录也算进去。
- **刘悦然的 0 分让她的平均分从 88.00 掉到 44.00**，直接影响了她是否"高于全体平均"。**这就是 0 分数据的意义——它必须在分母里，且值就是 0，不是"没有成绩"。**

---

### 答案 10

#### (a) 每门课成绩前 2 名（含并列）

```sql
WITH ranked AS (
    SELECT e.course_id,
           e.student_id,
           e.score,
           RANK() OVER (PARTITION BY e.course_id ORDER BY e.score DESC) AS rnk
    FROM enrollments e
    WHERE e.score IS NOT NULL                 -- 未出成绩的不参与排名
)
SELECT c.course_name,
       s.name  AS student_name,
       r.score,
       r.rnk
FROM ranked r
JOIN courses  c ON c.course_id  = r.course_id
JOIN students s ON s.student_id = r.student_id
WHERE r.rnk <= 2
ORDER BY r.course_id, r.rnk;
```

**逐门课验证**（`WHERE score IS NOT NULL` 之后）：

| 课程 | 参与排名的成绩（降序） | 前 2 名 |
| --- | --- | --- |
| 101 程序设计基础 | 95.5, 88.5, 72.0, 59.0 | 95.5(李雨桐), 88.5(王小明) |
| 102 数据结构 | 92.0, 91.0 | 92.0(王小明), 91.0(陈思远) |
| 103 数据库系统原理 | 88.0, 79.0（李雨桐的 NULL 被排除） | 88.0(刘悦然), 79.0(王小明) |
| 104 操作系统 | 67.5, 0 | 67.5(陈思远), **0(刘悦然)** |
| 105 离散数学 | 85.0 | 85.0(张浩然) |

**`RANK` vs `ROW_NUMBER` 的选择**：

| 用哪个 | 效果 |
| --- | --- |
| `RANK()` | **并列都保留**（如果前两名分数相同，可能返回 3 行） |
| `ROW_NUMBER()` | **严格只返回 2 行**（并列的按某种确定顺序取前 2 个） |

题目说"含并列"，所以用 `RANK` 是对的。本数据里没有并列，所以两者结果相同——**但要知道它们的语义差别，否则在有并列的数据上会得到意外的行数。**

**易错点**：
- **窗口函数的结果不能在 `WHERE` 里直接过滤**。因为逻辑执行顺序上窗口函数是在 `WHERE` 之后、`ORDER BY` 之前计算的，所以必须用 CTE 或派生表包一层：
  ```sql
  -- ❌ 报错：窗口函数不能出现在 WHERE 里
  SELECT ... , RANK() OVER (...) AS rnk
  FROM enrollments
  WHERE RANK() OVER (...) <= 2;
  ```
- **`WHERE e.score IS NOT NULL` 必须在 CTE 内部**（参与排名之前就排除）。如果放在外层，`NULL` 成绩会占用排名位置（在 PostgreSQL 里 `DESC` 时 NULL 最大，会排第 1）。
- **104 号课的"第 2 名"是 0 分**：因为这门课只有 2 条已出成绩的记录，0 分也排进前 2。**这说明"前 N 名"在数据稀疏时可能包含很差的成绩**——业务上可能需要额外条件（例如"成绩 ≥ 60 才能上榜"），这要在题目和查询里明确。
- **窗口函数需要 MySQL 8.0+ / PostgreSQL / SQLite 3.25+**；MySQL 5.7 不支持。

#### (b) 递归 CTE：106 号课的先修链

```sql
WITH RECURSIVE prereq_chain AS (
    -- ① 锚点：从目标课程自身开始（层级 0）
    SELECT cp.course_id,
           cp.prereq_course_id,
           0 AS level
    FROM course_prereq cp
    WHERE cp.course_id = 106

    UNION ALL

    -- ② 递归成员：沿着先修关系继续往上找
    SELECT cp.course_id,
           cp.prereq_course_id,
           pc.level + 1
    FROM course_prereq cp
    JOIN prereq_chain pc ON cp.course_id = pc.prereq_course_id
    WHERE pc.level < 20                       -- ③ 安全上限，防止数据有环时无限递归
)
SELECT pc.level,
       pc.course_id       AS course_id,
       c1.course_name     AS course_name,
       pc.prereq_course_id AS requires,
       c2.course_name      AS requires_name
FROM prereq_chain pc
LEFT JOIN courses c1 ON c1.course_id = pc.course_id
LEFT JOIN courses c2 ON c2.course_id = pc.prereq_course_id
ORDER BY pc.level, pc.course_id;
```

**逐步推演**：

| 层级 | `course_id` | `prereq_course_id` | 含义 |
| --- | --- | --- | --- |
| 0 | 106 | 103 | 软件工程导论 需要 数据库系统原理 |
| 1 | 103 | 102 | 数据库系统原理 需要 数据结构 |
| 2 | 102 | 101 | 数据结构 需要 程序设计基础 |
| 3 | 101 | （无记录） | 递归成员返回空集 → 递归结束 |

**所以修 106 之前要依次修完：101 程序设计基础 → 102 数据结构 → 103 数据库系统原理。**

**如果想输出"一条链"而不是"每层一行"**，可以像这样拼接路径：

```sql
WITH RECURSIVE chain AS (
    SELECT cp.course_id,
           cp.prereq_course_id,
           0 AS level,
           CAST(c.course_name AS VARCHAR(500)) AS path
    FROM course_prereq cp
    JOIN courses c ON c.course_id = cp.course_id
    WHERE cp.course_id = 106

    UNION ALL

    SELECT cp.course_id,
           cp.prereq_course_id,
           ch.level + 1,
           CAST(ch.path || ' <- ' || c.course_name AS VARCHAR(500))
    FROM course_prereq cp
    JOIN chain ch ON cp.course_id = ch.prereq_course_id
    JOIN courses c ON c.course_id = cp.course_id
    WHERE ch.level < 20
)
SELECT level, course_id, prereq_course_id, path
FROM chain
ORDER BY level DESC;                 -- 反过来输出，从最基础的先修课开始
```

**易错点**：
- **`RECURSIVE` 关键字不能省**（PostgreSQL / MySQL 8.0+ / SQLite 都要求）。SQL Server 与 Oracle 则不写这个关键字。
- **`path` 列必须在锚点里 `CAST` 成足够宽的类型**。因为很多数据库用锚点的列类型来决定递归 CTE 的列类型，如果锚点里是 `VARCHAR(50)`，拼几次就会报 "value too long"。
- **必须有终止条件**。这里的终止是"隐式"的：当某一层的 `prereq_course_id` 在 `course_prereq` 里查不到时，递归成员返回空集，递归自然结束。**但如果数据里有环（A 需要 B、B 需要 A），就永远不会返回空集**，所以额外加了 `WHERE ch.level < 20` 作为安全网。**这是一个好习惯——成本极低，但能避免脏数据把生产查询挂死。**
- **注意递归的方向**：`JOIN chain ch ON cp.course_id = ch.prereq_course_id` 表示"找以'当前层的先修课'作为课程的记录"，也就是**继续往上找更基础的先修课**。如果把方向写反（`cp.prereq_course_id = ch.course_id`），就变成了"往下找依赖它的课"。
- **`LEFT JOIN courses`** 而不是 `JOIN`：万一 `course_prereq` 里引用了 `courses` 里不存在的 `course_id`（脏数据），`LEFT JOIN` 至少能显示出那一行，便于发现问题；`INNER JOIN` 会静默丢掉它。

---

## 4. 知识点汇总表

| 题号 | 考点 | 关键语法 / 函数 |
| --- | --- | --- |
| 1 | 单表筛选、多列排序、分页 | `WHERE`、`ORDER BY ... DESC, ... ASC`、`LIMIT` |
| 2 | 去重 | `DISTINCT`（作用于整行） |
| 3 | 多条件组合、`NULL` 的过滤行为 | `BETWEEN`、`IN`、`LIKE`、括号、`AND`/`OR` 优先级 |
| 4 | 三表内连接 | `INNER JOIN ... ON`、表别名 |
| 5 | 反连接（找缺失） | `LEFT JOIN ... IS NULL`、`NOT EXISTS` |
| 6 | 行级过滤 + 组级过滤 | `WHERE` vs `HAVING`、`COUNT`、`AVG`、`ROUND` |
| 7 | 聚合与 `NULL` 的完整处理 | `LEFT JOIN` + `COUNT(右表列)`、`COUNT(*)` vs `COUNT(col)`、`MAX` |
| 8 | 存在性判断、空集语义 | `EXISTS`、`NOT EXISTS`、`IN`、真空为真 |
| 9 | 分档、CTE、两级聚合 | `CASE WHEN`、`WITH`、`CASE` 排序、`AVG` 的口径 |
| 10 | 窗口函数 / 递归 CTE | `RANK() OVER (PARTITION BY ... ORDER BY ...)`、`WITH RECURSIVE`、`UNION ALL` |

**贯穿全篇的三条主线**：

```text
① NULL 的三值逻辑
   题 3、5、7、8、9 都与它有关
   - NULL 不被任何比较匹配（题 3）
   - 反连接要用右表主键判断 IS NULL（题 5）
   - COUNT(*) vs COUNT(col)（题 7）
   - 空集为真 (vacuously true)（题 8）
   - CASE WHEN 必须显式处理 NULL（题 9）

② 逻辑执行顺序
   题 6、7、8、10 都与它有关
   - WHERE 过滤行、HAVING 过滤组（题 6）
   - 窗口函数不能在 WHERE 里用，要包一层（题 10）
   - 派生表/CTE 存在的根本原因就是这个顺序（题 9、10）

③ "0" 与 "NULL" 的区别
   题 6、7、8、9 都涉及
   - 0 是有效成绩，要计入平均分（题 6、7）
   - 0 < 60 是不及格，NULL 不是（题 8）
   - 0 掉进 F 档，NULL 是"未出成绩"（题 9）
```

## 5. 如何自测

1. **真的建表并跑一遍。** 用 SQLite（最省事，`sqlite3 test.db`）、PostgreSQL 或 MySQL 都行。**只在脑子里过一遍不算做过。**

2. **用行数做快速校验。** 几个可以直接对比的预期值：

   | 查询 | 预期行数 |
   | --- | --- |
   | `SELECT COUNT(*) FROM students` | 7 |
   | `SELECT COUNT(*) FROM courses` | 6 |
   | `SELECT COUNT(*) FROM enrollments` | 12 |
   | 题 2(a) 出现过的学期数 | 2（2024-Fall、2025-Spring） |
   | 题 2(b) 出现过的专业数 | 3 |
   | 题 4（选课明细）行数 | 12 |
   | 题 5（没选课的学生）行数 | 1（学生 7） |
   | 题 6（人数 ≥ 2 的课）行数 | 4 |
   | 题 7（所有课程）行数 | **6**（包括没人选的课） |
   | 题 8(a)（有 ≥ 90 分的学生）行数 | 3 |
   | 题 8(b)（有课且无不及格）行数 | 5（学生 1,2,3,4,5；学生 6 有 59 分、学生 7 没选课，都被排除） |
   | 题 9(b) 各档合计 | 12（= `enrollments` 总行数） |
   | 题 10(a)（前 2 名，RANK）行数 | 9 |

   最后一行怎么来的：101 有 4 条有效成绩取前 2 = 2 行，102 有 2 条取前 2 = 2 行，103 有 2 条取前 2 = 2 行，104 有 2 条取前 2 = 2 行，105 有 1 条 = 1 行 —— 合计 **9 行**。

3. **故意破坏数据看会发生什么。** 几个有价值的实验：

   ```sql
   -- 实验 1：先跑一次题 5 的两种写法，确认结果都是学生 7（1 行）
   SELECT s.student_id, s.name FROM students s
   LEFT JOIN enrollments e ON e.student_id = s.student_id
   WHERE e.enrollment_id IS NULL;

   -- 现在插一条 student_id 为 NULL 的记录（模拟脏数据）
   INSERT INTO enrollments (enrollment_id, student_id, course_id, score, semester)
   VALUES (99, NULL, 101, 80, '2025-Fall');

   -- 再跑【不加保护的 NOT IN】写法：
   SELECT * FROM students WHERE student_id NOT IN (SELECT student_id FROM enrollments);
   -- 观察：结果变成【空集】！（这就是 NOT IN 陷阱的实证）

   -- 再跑 NOT EXISTS，它不受影响：
   SELECT s.student_id, s.name FROM students s
   WHERE NOT EXISTS (SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id);
   -- 观察：仍然正确返回学生 7

   -- 记得用完删掉：DELETE FROM enrollments WHERE enrollment_id = 99;

   -- 实验 2：把某条 score 从 0 改成 NULL，或反之，看题 6/7 的平均分怎么变
   -- 这会让你直观理解"0 与 NULL"的区别

   -- 实验 3：在 course_prereq 里插一条形成环的记录，跑题 10(b)
   INSERT INTO course_prereq VALUES (101, 106);    -- 101 需要 106，而 106 又需要 103→102→101，成环！
   -- 如果没有 level 上限，这个查询会一直跑到资源耗尽
   ```

4. **把每题改写成另一种写法**，对比结果是否相同：
   - 题 5 的两种写法（`LEFT JOIN` vs `NOT EXISTS`）；
   - 题 8(a) 的两种写法（`EXISTS` vs `IN`）；
   - 题 9 的 CTE 写法 vs 在 `GROUP BY` 里重复 `CASE` 的写法。

   **如果两种写法结果不同，就说明你找到了一个语义差异**（例如 `NOT IN` 与 `NOT EXISTS` 在含 NULL 时的差别），这比做对题目更有价值。

## 小结

- **这套练习真正的价值在"陷阱数据"**：一个 `NULL` 成绩、一个 `0` 分成绩、一个没选课的学生、一门没人选的课。**能不能正确处理这四个边界情况，就是"会写 SQL"与"背过语法"的分界。**
- **`NULL` 的三值逻辑贯穿一半以上的题目**：`NULL` 不被比较匹配、`COUNT(*) ≠ COUNT(col)`、`NOT IN` 遇到 `NULL` 返回空集、`CASE WHEN` 必须显式处理 `NULL`、空集为真。
- **`0` 和 `NULL` 是完全不同的东西**：0 是有效成绩（计入平均分、算不及格、归入 F 档）；`NULL` 是"未知/未出成绩"（跳过聚合、不满足任何比较）。**混淆这两者是本套练习最想纠正的错误。**
- **`WHERE` 过滤行、`HAVING` 过滤组**。"只用已出成绩的记录"是行级条件，"人数 ≥ 2"是组级条件——放错位置会得到完全不同的结果。
- **`LEFT JOIN` 之后计数必须用 `COUNT(右表列)`**，`COUNT(*)` 会把"没有匹配"的那一行数成 1。
- **反连接要用右表的"主键"列判断 `IS NULL`**，用可空列（如 `score`）判断会把"有记录但该列为空"误判成"没有记录"。
- **窗口函数和聚合结果都不能直接在 `WHERE` 里过滤**（因为逻辑执行顺序），必须用 CTE 或派生表包一层。
- **递归 CTE 一定要加深度上限**，成本极低，但能防止脏数据造成的无限递归。
- **"全体平均分"这类表述必须先明确口径**：是"每个学生等权"还是"每条成绩等权"？两种算法结果不同（本数据里是 73.79 vs 74.32）。**这是需求沟通问题，不是 SQL 问题——但不确认就会算错。**
- **真正学会的标志不是"写对了"，而是"能说出为什么这么写"**。特别是：为什么这里用 `NOT EXISTS` 而不是 `NOT IN`？为什么 `COUNT` 要指定右表的列？为什么 `CASE WHEN` 里 `NULL` 要放最前？**能回答这些问题，才算掌握了。**
