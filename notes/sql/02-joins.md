# SQL 连接查询：INNER / LEFT / RIGHT / FULL JOIN 与自连接

本篇讲清连接的数学本质（笛卡尔积 + 过滤）、四种外连接的语义差别与图解，以及外连接里 `ON` 与 `WHERE` 的关键区别——这是最容易写错的地方。

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

示例数据（关注"有学生没选课""有课没人选"这两种情况，它们是外连接练习题的关键）：

```sql
INSERT INTO teachers (teacher_id, teacher_name, department) VALUES
    (1, '黄志强', '计算机系'),
    (2, '林美玲', '计算机系'),
    (3, '吴建国', '数学系'),
    (4, '郑晓月', '软件工程系');      -- 这位老师暂时没有课

INSERT INTO students (student_id, name, gender, major, city, enroll_year) VALUES
    (1, '王小明', 'M', '计算机科学与技术', '厦门', 2022),
    (2, '李雨桐', 'F', '计算机科学与技术', '北京', 2022),
    (3, '张浩然', 'M', '软件工程',         '厦门', 2021),
    (4, '陈思远', 'M', '计算机科学与技术', '上海', 2023),
    (5, '刘悦然', 'F', '数据科学',         NULL,   2022),
    (6, '赵子涵', 'F', '软件工程',         '厦门', 2023);   -- 这位学生没选课

INSERT INTO courses (course_id, course_name, credits, teacher_id) VALUES
    (101, '程序设计基础',   4, 1),
    (102, '数据结构',       4, 1),
    (103, '数据库系统原理', 3, 2),
    (104, '操作系统',       4, 2),
    (105, '离散数学',       3, 3),
    (106, '软件工程导论',   3, NULL);   -- 还没分配老师

INSERT INTO enrollments (enrollment_id, student_id, course_id, score, semester) VALUES
    (1, 1, 101, 88.5, '2024-Fall'),
    (2, 1, 102, 92.0, '2024-Fall'),
    (3, 1, 103, 79.0, '2025-Spring'),
    (4, 2, 101, 95.5, '2024-Fall'),
    (5, 2, 103, NULL, '2025-Spring'),   -- 成绩未出
    (6, 3, 101, 72.0, '2024-Fall'),
    (7, 3, 105, 85.0, '2024-Fall'),
    (8, 4, 102, 91.0, '2025-Spring'),
    (9, 4, 104, 67.5, '2025-Spring'),
    (10, 5, 103, 88.0, '2025-Spring');
```

**注意三处刻意设计**：
- 学生 6（赵子涵）**没有任何选课记录** → 用来演示 `LEFT JOIN ... IS NULL`；
- 课程 106（软件工程导论）**没有学生选** → 同理；
- 教师 4（郑晓月）**没有课** → 演示三表连接的缺口。

## 1. 为什么需要连接

关系型数据库设计的核心原则是**规范化 (normalization)**：每一份事实只存一处，避免重复与不一致。所以：

- 学生信息在 `students`，课程信息在 `courses`；
- "谁选了哪门课、得了多少分"是**它们之间的关系**，存在 `enrollments`。

要回答"王小明在数据库课上的成绩是多少"，必须把三张表拼起来。这个"拼起来"的动作就是**连接 (join)**。

## 2. 连接的数学本质：笛卡尔积 + 过滤

**连接的基础是笛卡尔积 (Cartesian product / cross product)**：把两张表的每一行两两配对。

```sql
-- 显式的笛卡尔积（CROSS JOIN）
SELECT * FROM students CROSS JOIN courses;
-- 6 个学生 × 6 门课 = 36 行
```

```text
students(6行)  CROSS JOIN  courses(6行)
┌──────────────────────────────────────────────┐
│ 王小明 × 程序设计基础                          │
│ 王小明 × 数据结构                              │
│ ...                                          │
│ 赵子涵 × 软件工程导论                          │
└──────────────────────────────────────────────┘
共 6 × 6 = 36 行
```

**内连接就是"笛卡尔积 + 只保留满足连接条件的行"**：

```sql
-- 显式 INNER JOIN（推荐的现代写法）
-- 建议按"从关系表出发、逐步连到维度表"的顺序写，连接条件里引用的两张表都已出现，最易读
SELECT s.name, c.course_name, e.score
FROM enrollments e
JOIN students s ON s.student_id = e.student_id
JOIN courses  c ON c.course_id  = e.course_id;

-- 隐式写法（老式，不推荐）：在 WHERE 里写连接条件
SELECT s.name, c.course_name, e.score
FROM students s, courses c, enrollments e
WHERE e.student_id = s.student_id
  AND e.course_id  = c.course_id;
```

两种写法在**内连接**下结果相同，但显式 `JOIN ... ON` 有明确的优势：

| | 显式 `JOIN ... ON` | 隐式 `FROM a, b WHERE ...` |
| --- | --- | --- |
| 可读性 | 连接条件紧跟在 `JOIN` 后面，一眼看清 | 连接条件和业务过滤混在 `WHERE` 里，难以区分 |
| 外连接 | ✅ 支持 | ❌ **无法表达外连接** |
| 忘写条件的后果 | 语法上很显眼（`JOIN` 后面直接跟 `WHERE`） | **静默产生笛卡尔积**，可能把 1000 行变成 100 万行 |
| 现代习惯 | 推荐 | 避免 |

> ⚠️ **忘记写连接条件是最危险的一类错误**：它不会报错，只是返回一堆无意义的行。在只有 10 行的小表上你不一定注意得到，在生产环境的大表上会直接拖垮数据库（10 万 × 10 万 = 100 亿行）。**看到结果行数远超预期，第一反应就是检查连接条件。**

## 3. 四种连接的图解

```text
设 A = 左表的行集, B = 右表的行集
匹配的行 = A ∩ B（满足连接条件的配对）

─────────────────────────────────────────────────────────────────────────────

【INNER JOIN】  只保留匹配上的
    ┌─────────┐
    │  A   ██ │
    │    ████ │
    │  ██████ │
    │    ████ │
    │  B   ██ │
    └─────────┘
   结果 = A ∩ B
   图例: ██ = 出现在结果里的行

   例: 选了课的学生 × 他们选的课
       → 赵子涵(没选课) 不在结果里，106号课(没人选) 也不在结果里


【LEFT JOIN (LEFT OUTER JOIN)】  保留左表所有行
    ┌─────────┐
    │  A   ░░ │   ░░ = 结果里有，但右表列全部为 NULL
    │    ████ │   ██ = 真正匹配上的
    │  ██████ │
    │    ████ │
    │  B   ██ │
    └─────────┘
   结果 = A 的全部行
          匹配上的用 B 的值
          没匹配的，B 的列全部补 NULL

   例: 所有学生 × 他们的选课
       → 赵子涵 出现，但 course_id / score 全是 NULL


【RIGHT JOIN (RIGHT OUTER JOIN)】  保留右表所有行（= 交换表顺序的 LEFT JOIN）
    ┌─────────┐
    │  A   ██ │
    │    ████ │
    │  ██████ │
    │    ████ │
    │  B   ░░ │   ░░ = 结果里有，但左表列全部为 NULL
    └─────────┘
   结果 = B 的全部行

   例: 所有课程 × 选它的学生
       → 106号课 出现，但 student_id / name 全是 NULL


【FULL OUTER JOIN】  两边的所有行都保留
    ┌─────────┐
    │  A   ░░ │
    │    ████ │
    │  ██████ │
    │    ████ │
    │  B   ░░ │
    └─────────┘
   结果 = A 的全部行 ∪ B 的全部行
          匹配上的合并
          只在 A 的，B 侧补 NULL
          只在 B 的，A 侧补 NULL

   例: 所有学生 × 所有课程（通过选课关系）
       → 赵子涵(没选课) 和 106号课(没人选) 都出现
```

**用一句话记住三者的差别**：

| 连接 | 保留谁 | 记忆法 |
| --- | --- | --- |
| `INNER JOIN` | 只保留匹配上的 | 交集 |
| `LEFT JOIN` | 左表全部 + 匹配上的 | "以左表为准" |
| `RIGHT JOIN` | 右表全部 + 匹配上的 | "以右表为准" |
| `FULL OUTER JOIN` | 两边全部 | 并集 |

**`LEFT JOIN` 和 `RIGHT JOIN` 是同一个操作的两个方向**：

```sql
-- 下面两个查询结果相同（列顺序可能不同）
SELECT * FROM students s LEFT  JOIN enrollments e ON e.student_id = s.student_id;
SELECT * FROM enrollments e RIGHT JOIN students s ON e.student_id = s.student_id;
```

**因此 `RIGHT JOIN` 几乎没有必要用**——把表顺序换一下写成 `LEFT JOIN` 更易读（读者的眼睛习惯从左往右读，"以第一张表为准"比"以最后一张表为准"更直观）。**建议统一用 `LEFT JOIN`。**

## 4. `ON` 与 `WHERE` —— 外连接里的关键区别

这是本篇最重要的一节。**对于 `INNER JOIN`，把条件写在 `ON` 还是 `WHERE` 里结果相同；但对于外连接，结果完全不同。**

### 4.1 内连接下两者等价

```sql
-- 这两种写法结果相同
SELECT s.name, e.score
FROM students s JOIN enrollments e ON e.student_id = s.student_id
WHERE e.score >= 90;

SELECT s.name, e.score
FROM students s JOIN enrollments e
  ON e.student_id = s.student_id AND e.score >= 90;
```

因为内连接只保留"匹配上且满足过滤"的行，条件放在哪一步都能达到同样效果。

### 4.2 外连接下两者完全不同

**场景**：想列出每个学生及其 90 分以上的成绩，**没考到 90 分的学生也要列出来**（成绩显示为 NULL）。

```sql
-- ✅ 正确：条件写在 ON 里
SELECT s.student_id, s.name, e.course_id, e.score
FROM students s
LEFT JOIN enrollments e
       ON e.student_id = s.student_id
      AND e.score >= 90;
```

执行过程（用"迭代模型"理解）：

```text
对于 students 的每一行 s：
    在 enrollments 里找所有满足 (e.student_id = s.student_id AND e.score >= 90) 的行
    如果找到了 → 每个匹配组合输出一行
    如果没找到 → 输出一行，e 的列全部为 NULL   ← 关键：LEFT JOIN 保证这一步发生

结果：
  王小明 | 102 | 92.0     （匹配上）
  李雨桐 | NULL | NULL    （她最高 95.5 是 101 课... 实际上 95.5 >= 90，
                            这里假设数据后也是匹配；用 score<90 的课程演示时她仍有一行 NULL）
  赵子涵 | NULL | NULL    （完全没选课，也有一行）
```

```sql
-- ❌ 错误：条件写在 WHERE 里，结果变成了 INNER JOIN
SELECT s.student_id, s.name, e.course_id, e.score
FROM students s
LEFT JOIN enrollments e ON e.student_id = s.student_id
WHERE e.score >= 90;
```

执行过程：

```text
第 1 步 LEFT JOIN：先产生所有行，没匹配的学生那行的 e.score 是 NULL
第 2 步 WHERE e.score >= 90：
    NULL >= 90 的结果是 UNKNOWN
    WHERE 只保留 TRUE 的行 → 那些补了 NULL 的行被【全部过滤掉】
最终结果 = 只有真正匹配上的行 = INNER JOIN 的结果
```

**这就是"在外连接之后用 `WHERE` 过滤右表的列，会把外连接退化成内连接"这个经典错误。** 它的隐蔽之处在于：**不报错，只是少了行**。如果你在测试时用的数据恰好每行都匹配，你根本发现不了。

### 4.3 判断规则（记住这一条）

> **`ON` 里的条件影响"怎么匹配"；`WHERE` 里的条件影响"匹配完之后的哪些行保留"。**
>
> 因此在外连接中：
> - 想让"不匹配的行也保留下来" → 条件写 **`ON`**
> - 想过滤掉某些保留的行 → 条件写 **`WHERE`**（但要知道这会破坏外连接的效果）

### 4.4 一个更有迷惑性的例子

```sql
-- 需求：列出所有学生，以及他们在 '2024-Fall' 学期的成绩
SELECT s.name, e.course_id, e.score, e.semester
FROM students s
LEFT JOIN enrollments e ON e.student_id = s.student_id
WHERE e.semester = '2024-Fall';        -- ❌ 又退化成 INNER JOIN 了
```

```sql
-- ✅ 正确写法
SELECT s.name, e.course_id, e.score, e.semester
FROM students s
LEFT JOIN enrollments e
       ON e.student_id = s.student_id
      AND e.semester = '2024-Fall';     -- 条件放进 ON
```

### 4.5 例外：`WHERE` 里过滤左表的列是安全的

```sql
-- ✅ 过滤【左表】的列不会破坏 LEFT JOIN 的效果
SELECT s.name, e.score
FROM students s
LEFT JOIN enrollments e ON e.student_id = s.student_id
WHERE s.enroll_year = 2022;      -- 这是在对"保留哪一行左表记录"做筛选，符合直觉
```

因为左表的所有行本来就会被保留，用 `WHERE` 过滤左表列只是"少考虑一些左表行"，语义清晰。**危险的是"在 `WHERE` 里过滤右表的列"。**

同理，`WHERE e.col IS NULL` 是一个**有意的例外**——它正是"反连接 (anti-join)"模式的写法（见下一节）。

## 5. 反连接：找出"不存在"的记录

这是 `LEFT JOIN` 最有价值的用法。**核心模式**：

```sql
SELECT ...
FROM A
LEFT JOIN B ON <连接条件>
WHERE B.<某个 NOT NULL 的列> IS NULL;
```

原理：`LEFT JOIN` 保留了所有 A 的行；如果某行在 B 里没有匹配，B 的列会全部是 NULL。所以"B 的列全为 NULL"就等价于"A 中不存在对应的 B 行"。

### 5.1 找出没有选任何课的学生

```sql
SELECT s.student_id, s.name, s.major
FROM students s
LEFT JOIN enrollments e ON e.student_id = s.student_id
WHERE e.student_id IS NULL;
-- 输出：赵子涵（学生 6）
```

**为什么用 `e.student_id IS NULL` 而不是 `e.enrollment_id IS NULL`**：两者都可以，但要选一个**在 `enrollments` 里一定非空**的列。`student_id` 在业务上应该非空（虽然表定义里没写 `NOT NULL`，这是一个设计瑕疵）。**更稳妥的做法是用右表的 `PRIMARY KEY` 列**，因为主键隐含 `NOT NULL`：

```sql
-- ✅ 更稳妥：用右表主键判断
SELECT s.student_id, s.name
FROM students s
LEFT JOIN enrollments e ON e.student_id = s.student_id
WHERE e.enrollment_id IS NULL;
```

### 5.2 找出没人选的课程

```sql
SELECT c.course_id, c.course_name, c.credits
FROM courses c
LEFT JOIN enrollments e ON e.course_id = c.course_id
WHERE e.enrollment_id IS NULL;
-- 输出：106 软件工程导论
```

### 5.3 找出没有课的教师

```sql
SELECT t.teacher_id, t.teacher_name, t.department
FROM teachers t
LEFT JOIN courses c ON c.teacher_id = t.teacher_id
WHERE c.course_id IS NULL;
-- 输出：4 郑晓月
```

### 5.4 用 `NOT EXISTS` 做同样的事

```sql
-- 与 5.1 等价的写法
SELECT s.student_id, s.name
FROM students s
WHERE NOT EXISTS (
    SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id
);
```

**两种方式的对比**：

| | `LEFT JOIN ... IS NULL` | `NOT EXISTS` |
| --- | --- | --- |
| 可读性 | 需要读者理解"补 NULL"再过滤 | **直接表达"不存在"，更贴合意图** |
| 是否需要去重 | ⚠️ 如果右表对同一左表行有**多行**匹配，会返回多行，需要 `DISTINCT` | ✅ 天然不会重复（存在性判断只返回一个布尔值） |
| 多数优化器的处理 | 通常能变成 anti-join，性能好 | 通常能变成 anti-join，性能好 |
| 常见犯错的点 | 忘记用右表的非空列判断、忘记去重 | 几乎不会写错 |

**结论**：**"找出不存在"的语义优先用 `NOT EXISTS`**，它更不容易写错，也不需要担心重复行。`LEFT JOIN ... IS NULL` 在性能上有时期望更明确（尤其当优化器对 `NOT EXISTS` 处理不好时），可以作为备选。

**注意 5.1 与 5.2 中的 `LEFT JOIN` 版本是否会重复**：`enrollments` 里一个学生对同一门课在同一学期只能有一条记录（有 `UNIQUE` 约束），但一个学生可以选多门课，所以"没有选任何课"的学生在 `LEFT JOIN` 结果里只有一行（因为没有匹配，只会补一行 NULL）。**这是安全的**。但如果反过来是"找出没有教师的课"，且一门课可以有多条关联记录时，就需要小心重复了。

## 6. `FULL OUTER JOIN`

```sql
SELECT COALESCE(s.name, '（无学生）')       AS student,
       COALESCE(c.course_name, '（无课程）') AS course,
       e.score
FROM students s
FULL OUTER JOIN enrollments e ON e.student_id = s.student_id
FULL OUTER JOIN courses c     ON c.course_id  = e.course_id;
```

`FULL OUTER JOIN` 的用途是"两边的所有行都要看到"。典型场景：

- **数据对账**：两张表应该一致，找出"只在 A 有""只在 B 有""两边都有但值不同"的记录；
- **合并两个来源的数据**：任一来源独有的行都不应丢失。

### 6.1 一个实用的对账模板

```sql
-- 对账：找出两个数据源之间的差异
SELECT COALESCE(a.id, b.id) AS id,
       a.value AS value_a,
       b.value AS value_b,
       CASE
           WHEN a.id IS NULL      THEN '仅在 B 中'
           WHEN b.id IS NULL      THEN '仅在 A 中'
           WHEN a.value <> b.value THEN '值不同'
           ELSE '一致'
       END AS diff_type
FROM source_a a
FULL OUTER JOIN source_b b ON a.id = b.id
WHERE a.id IS NULL OR b.id IS NULL OR a.value <> b.value;
```

`COALESCE(a.id, b.id)` 是关键技巧：因为 `FULL OUTER JOIN` 会让"只在一边存在"的行的另一边列为 NULL，用 `COALESCE` 就能取到那个总是存在的键。

### 6.2 MySQL 不支持 `FULL OUTER JOIN`

**这是一个必须知道的方言差异**：MySQL 没有实现 `FULL OUTER JOIN`。用 `LEFT JOIN` 和 `RIGHT JOIN` 的并集模拟：

```sql
-- MySQL 中等价于 FULL OUTER JOIN 的写法
SELECT s.name, e.course_id, e.score
FROM students s
LEFT JOIN enrollments e ON e.student_id = s.student_id

UNION

SELECT s.name, e.course_id, e.score
FROM students s
RIGHT JOIN enrollments e ON e.student_id = s.student_id;
```

几个要点：

- **用 `UNION` 而不是 `UNION ALL`**：`UNION` 会自动去重，把两个查询结果里重复的"匹配行"合并成一行。用 `UNION ALL` 会让匹配的行出现两次；
- 两个查询的**列数与列类型必须对应**（列名取第一个查询的）；
- 这种写法的代价是**两次扫描**，性能不如原生的 `FULL OUTER JOIN`。

**顺便说明 `UNION` 与 `UNION ALL` 的区别**（这个区别在别处也很重要）：

| | `UNION` | `UNION ALL` |
| --- | --- | --- |
| 去重 | ✅ 自动去重 | ❌ 保留所有行 |
| 性能 | 慢（需要排序或哈希来去重） | 快（直接拼接） |
| 何时用 | 需要去重（如上面的模拟） | 确定没有重复，或**需要保留重复**时 |

**默认应该用 `UNION ALL`**（更快且语义更直白）。只有确实需要去重时才用 `UNION`——而且要注意"`UNION` 会去掉重复行"这件事可能不是你想要的（例如统计两次活动的参与情况时，同一个人参加两次应该出现两次）。

## 7. 多表连接

三张以上表的连接就是把 `JOIN` 链式串起来。

```sql
-- 查询每位学生的选课情况，含课程名与任课教师
SELECT s.name        AS student_name,
       c.course_name AS course_name,
       t.teacher_name AS teacher_name,
       e.score,
       e.semester
FROM enrollments e
JOIN students s ON s.student_id = e.student_id
JOIN courses  c ON c.course_id  = e.course_id
LEFT JOIN teachers t ON t.teacher_id = c.teacher_id    -- 用 LEFT：课程可能还没分配老师
ORDER BY s.name, c.course_name;
```

**关于连接顺序的几点说明**：

1. **逻辑上连接顺序无关**：`A JOIN B JOIN C` 与 `A JOIN C JOIN B` 的内连接结果相同（都是"满足所有连接条件的行的组合"）。外连接则顺序**有影响**，因为"保留谁"变了。
2. **物理上顺序影响性能**：优化器会根据表的统计信息、索引、预估行数来选择连接的物理顺序与算法（嵌套循环 nested loop、哈希连接 hash join、排序合并连接 sort-merge join）。**我们写 SQL 时不必手工指定顺序，但理解这一点有助于解释为什么加个索引就快了。**
3. **混合内外连接时要小心**：如果链中间有一个 `LEFT JOIN`，它之后的所有 `JOIN` 都作用在"已经补过 NULL 的结果"上。这时把过滤条件放在 `WHERE` 里同样会出问题。

```sql
-- ⚠️ 这个查询里 t.department = '计算机系' 放在 WHERE 会破坏 LEFT JOIN
SELECT s.name, c.course_name, t.teacher_name
FROM enrollments e
JOIN students s ON s.student_id = e.student_id
JOIN courses  c ON c.course_id  = e.course_id
LEFT JOIN teachers t ON t.teacher_id = c.teacher_id
WHERE t.department = '计算机系';      -- 没分配老师的课被过滤掉了

-- ✅ 正确：条件放进 ON
... LEFT JOIN teachers t
      ON t.teacher_id = c.teacher_id
     AND t.department = '计算机系';
```

## 8. 连接的写法细节

### 8.1 `ON` 与 `USING`

```sql
-- ON：显式指定条件，两列都保留
SELECT * FROM students s JOIN enrollments e ON e.student_id = s.student_id;
-- 结果列里有两个 student_id（s.student_id 和 e.student_id）

-- USING：当两边的列名相同时，可以简写
SELECT * FROM students s JOIN enrollments e USING (student_id);
-- 结果列里【只有一个】student_id（被合并了）
```

| | `ON a.col = b.col` | `USING (col)` |
| --- | --- | --- |
| 简写 | 稍长 | 更短 |
| 结果中的连接列 | **两列都出现** | **合并成一列** |
| 列必须同名 | 不必 | 必须 |
| `SELECT *` 的可读性 | 有重复列，容易混淆 | 更干净 |
| 可移植性 | 最好 | 好（标准 SQL，主流数据库都支持） |

**建议**：`USING` 在列名相同时更干净，但 `ON` 更通用（可以做非等值连接、多条件连接）。**理解 `USING` 会合并列**这一点很重要——它影响 `SELECT *` 的结果，也影响你是否能用不带表前缀的 `student_id` 引用该列（用 `USING` 后可以，用 `ON` 后必须加前缀，因为存在歧义）。

### 8.2 `NATURAL JOIN` —— 明确建议不要用

```sql
-- ⚠️ 不要用
SELECT * FROM students NATURAL JOIN enrollments;
```

`NATURAL JOIN` 会**自动按所有同名的列**做等值连接。危险在于：

1. **隐式**：连接条件没有写在代码里，读代码的人必须去查两表的所有列名才知道按什么连；
2. **脆弱**：如果以后给表加了一个同名列（例如给两张表都加了 `semester`），连接条件会**悄悄改变**，结果就错了，而且不会报错；
3. **可能产生意外的多列连接**：如果意外有多个同名列，会变成 `AND` 连接，结果可能与预期完全不同。

**相比之下，`USING (student_id)` 至少把连接列写在代码里了。** 所以：用 `ON` 或 `USING`，**永远不要用 `NATURAL JOIN`**。

### 8.3 非等值连接

连接条件不一定是 `=`，可以是任何布尔表达式：

```sql
-- 找出成绩高于该课程平均分的学生
-- （这里为了在一个查询里展示，用了子查询计算均分；实际也可以写成连接）
SELECT e1.student_id, e1.course_id, e1.score
FROM enrollments e1
JOIN (SELECT course_id, AVG(score) AS avg_score
      FROM enrollments GROUP BY course_id) AS a
  ON a.course_id = e1.course_id
WHERE e1.score > a.avg_score;

-- 区间连接：把成绩映射到等级区间表
-- 假设有个 grade_scale 表: (grade, min_score, max_score)
SELECT e.enrollment_id, e.score, g.grade
FROM enrollments e
JOIN grade_scale g
  ON e.score >= g.min_score AND e.score <= g.max_score;
```

**区间连接**是一个有用的模式，但要注意：如果区间表有重叠或覆盖不全，可能出现一行匹配多个等级（产生重复行）或一个都匹配不上（该行消失）。**做这类连接前先用查询验证区间表是完整且不重叠的。**

### 8.4 `CROSS JOIN` 的正当用途

虽然意外产生的笛卡尔积是灾难，但**有意的 `CROSS JOIN` 是有用的**：

```sql
-- 1. 生成"所有学生 × 所有课程"的完整矩阵，用于统计"哪些组合没有选课记录"
SELECT s.student_id, s.name, c.course_id, c.course_name
FROM students s
CROSS JOIN courses c
LEFT JOIN enrollments e
       ON e.student_id = s.student_id AND e.course_id = c.course_id
WHERE e.enrollment_id IS NULL;
-- 输出：每个学生没选的课。这是"缺口分析"的标准模式。

-- 2. 生成日期序列（用于报表补齐空缺日期）
-- 与一个日期维度表做 CROSS JOIN
SELECT d.day, p.product_id, COALESCE(SUM(s.amount), 0) AS total
FROM date_dim d
CROSS JOIN products p
LEFT JOIN sales s ON s.sale_date = d.day AND s.product_id = p.product_id
WHERE d.day BETWEEN '2025-01-01' AND '2025-01-31'
GROUP BY d.day, p.product_id
ORDER BY d.day, p.product_id;
```

第 2 个例子是报表开发的经典技巧：**"不存在的日期/组合也要显示为 0"必须靠 `CROSS JOIN` 制造出完整框架，再用 `LEFT JOIN` 补数据。**

## 9. 自连接 (self join)

自连接就是**把同一张表用两个不同的别名引用**，用来处理"表内行与行之间的关系"。**必须用别名**，否则无法区分引用的是哪一个"角色"。

### 9.1 场景一：找出没有选课的学生（用自连接的形式表达）

其实更常见的第一个自连接例子是"员工与经理"这种**层级关系**。为了演示，先临时建一张员工表：

```sql
CREATE TABLE employees (
    emp_id     INTEGER PRIMARY KEY,
    emp_name   VARCHAR(50) NOT NULL,
    manager_id INTEGER,                       -- 指向同一张表的 emp_id
    department VARCHAR(50),
    salary     NUMERIC(10,2)
);

INSERT INTO employees VALUES
    (1, '总经理-张伟',   NULL, '管理层',   35000),
    (2, '技术总监-李强',   1, '技术部',   25000),
    (3, '财务总监-王芳',   1, '财务部',   24000),
    (4, '后端组长-刘洋',   2, '技术部',   18000),
    (5, '前端组长-陈静',   2, '技术部',   17500),
    (6, '后端工程师-赵磊', 4, '技术部',   12000),
    (7, '前端工程师-孙娜', 5, '技术部',   11500),
    (8, '会计-周涛',       3, '财务部',    9000);
```

**查询 1：列出每位员工及其直接经理的姓名**

```sql
SELECT e.emp_id,
       e.emp_name        AS employee,
       e.department,
       m.emp_name        AS manager
FROM employees e
LEFT JOIN employees m ON m.emp_id = e.manager_id     -- 自连接：同一张表两个别名
ORDER BY e.emp_id;
```

**为什么用 `LEFT JOIN` 而不是 `INNER JOIN`**：总经理的 `manager_id` 是 `NULL`，用 `INNER JOIN` 会因为 `NULL = emp_id` 是 UNKNOWN 而**把他排除掉**。这是一个非常实际的例子：**层级查询的顶层节点必须用左连接才能保住。**

**查询 2：找出员工比其经理薪水还高的情况**

```sql
SELECT e.emp_name AS employee_name, e.salary AS emp_salary,
       m.emp_name AS manager_name,  m.salary AS mgr_salary,
       e.salary - m.salary AS difference
FROM employees e
JOIN employees m ON m.emp_id = e.manager_id
WHERE e.salary > m.salary;
-- 用 INNER JOIN：没有经理的人（总经理）本来就不参与比较
```

**查询 3：找出同一部门的员工两两配对（用于找"同部门同事"）**

```sql
SELECT e1.emp_name AS emp_a,
       e2.emp_name AS emp_b,
       e1.department
FROM employees e1
JOIN employees e2
  ON e1.department = e2.department
 AND e1.emp_id < e2.emp_id        -- 【关键】用 < 而不是 <>
ORDER BY e1.department, e1.emp_name, e2.emp_name;
```

**`e1.emp_id < e2.emp_id` 是关键，这里必须用严格小于**，原因有两个：

| 若用 `<>` | 若用 `<` |
| --- | --- |
| `(1,2)` 和 `(2,1)` 都会出现 → **每对出现两次** | 每对只出现一次 |
| `e1.emp_id <> e1.emp_id` 为 FALSE，所以不会自配对 ✓ | 同样不会自配对 ✓ |
| 结果里会有重复的配对（顺序不同） | 结果干净 |

**用 `<`（或 `>`）是自连接做"配对"时的标准技巧**：它同时排除了自配对（`a < a` 为假）并让每一对只出现一次。

### 9.2 场景二：同专业同城市的学生配对（用示例库的 students 表）

```sql
SELECT s1.student_id AS id_a, s1.name AS name_a,
       s2.student_id AS id_b, s2.name AS name_b,
       s1.major, s1.city
FROM students s1
JOIN students s2
  ON s1.major = s2.major
 AND s1.city  = s2.city
 AND s1.student_id < s2.student_id      -- 避免重复对与自配对
ORDER BY s1.major, s1.city;
```

结果（基于前面的示例数据）：王小明与李雨桐同专业，但城市不同（厦门 vs 北京），所以不配对。**张浩然与赵子涵**（都是软件工程，都在厦门）会配成一对。

**注意 `NULL` 的处理**：刘悦然的 `city` 是 `NULL`，`NULL = NULL` 是 UNKNOWN，所以**她不会和任何其他 city 为 NULL 的学生配对**。如果确实需要让 NULL 也能配对，必须显式写：

```sql
ON s1.major = s2.major
AND (s1.city = s2.city OR (s1.city IS NULL AND s2.city IS NULL))
AND s1.student_id < s2.student_id
```

### 9.3 场景三：找出分数差距在 5 分以内的选课对（自连接做"相似度"分析）

```sql
SELECT e1.student_id AS stu_a, e1.course_id AS course_a, e1.score AS score_a,
       e2.student_id AS stu_b, e2.course_id AS course_b, e2.score AS score_b
FROM enrollments e1
JOIN enrollments e2
  ON e1.course_id = e2.course_id               -- 同一门课
 AND e1.student_id < e2.student_id             -- 不同学生，每对一次
 AND ABS(e1.score - e2.score) <= 5             -- 分差在 5 分内
WHERE e1.score IS NOT NULL
  AND e2.score IS NOT NULL                     -- 排除未出成绩的
ORDER BY e1.course_id, e1.score DESC;
```

**这里显式排除了 `NULL` 分数**：`ABS(NULL - 90)` 是 `NULL`，`NULL <= 5` 是 UNKNOWN，所以那些行本来也会被过滤掉。写成显式的 `IS NOT NULL` 是为了**让意图清晰**（读者不用自己去推 NULL 的行为）。

## 10. 连接性能的基本直觉

这一节只讲最实用的几条，详细的索引与执行计划在 [05-ddl-and-constraints.md](05-ddl-and-constraints.md)。

### 10.1 在连接列上建索引

连接的本质是"对左表的每一行，去右表里找匹配的行"。如果右表的连接列上有索引，这个"找"就从**全表扫描**变成了**索引查找**：

```sql
-- 假设经常按 student_id 连接 enrollments
CREATE INDEX idx_enrollments_student ON enrollments (student_id);
CREATE INDEX idx_enrollments_course  ON enrollments (course_id);
```

**注意一个真实的差异**：**MySQL 的 InnoDB 会自动为外键列创建索引**（这是外键约束的要求），而 **PostgreSQL 不会**——它只创建约束，索引要自己建。所以从 MySQL 转到 PostgreSQL 时，"外键列上有索引"这个假设不再成立，需要显式建索引。

### 10.2 用小表驱动大表

在**嵌套循环连接**下，外层循环的行数越少越好（因为要对每一行执行一次内层查找）。所以"小表驱动大表"通常更快。但**这是优化器的职责**，现代数据库会基于统计信息自己决定顺序。我们能做的：

- **保持统计信息更新**（PostgreSQL 的 `ANALYZE`、MySQL 的 `ANALYZE TABLE`）；
- **不要在连接条件里用函数包裹列**（会让优化器无法用索引，也无法准确估算）：
  ```sql
  -- ❌ 用不上索引
  FROM a JOIN b ON UPPER(a.code) = UPPER(b.code)
  -- ✅ 保持列"裸露"，用索引；如果确实要大小写不敏感，考虑用 CITEXT（PostgreSQL）或在写入时统一大小写
  FROM a JOIN b ON a.code = b.code
  ```

### 10.3 用 `EXPLAIN` 观察执行计划

```sql
EXPLAIN SELECT s.name, e.score
FROM students s JOIN enrollments e ON e.student_id = s.student_id
WHERE s.major = '计算机科学与技术';

-- PostgreSQL / MySQL 8+ 支持 ANALYZE，会真正执行并给出实际耗时与行数
EXPLAIN ANALYZE SELECT ...;
```

**该看什么**（字段名因数据库而异，这里只说通用的关注点）：

| 关注点 | 说明 |
| --- | --- |
| 是否走了索引 | 出现 "Seq Scan" / "ALL" / "full table scan" 说明是全表扫描 |
| 预估行数与实际行数的差异 | 差得太多说明统计信息过期，优化器可能选错计划 |
| 连接算法 | Nested Loop / Hash Join / Merge Join；大表之间的 Nested Loop 通常是警报 |
| 连接顺序 | 谁在前谁在后，是否符合"小表驱动" |
| 是否有排序/临时表 | "Sort"、"Using temporary"、"Using filesort" 在大数据量下是性能瓶颈 |
| 过滤在哪一步生效 | `WHERE` 条件是否被下推 (push down) 到了扫描阶段 |

**一个重要的提醒**：`EXPLAIN` 的输出**因数据库版本差异很大**（MySQL 8 与 5.7 就不一样，PostgreSQL 各版本也有变化）。**不要背某个版本的输出格式，理解上面这些"关注点"更重要。**

## 11. 常见错误清单

| 错误 | 问题 | 原因与修正 |
| --- | --- | --- |
| 忘写连接条件 | 产生笛卡尔积，行数爆炸 | 检查 `JOIN` 后面是否紧跟 `ON`/`USING` |
| `LEFT JOIN` 后在 `WHERE` 里过滤右表列 | 退化成 `INNER JOIN`，少了行 | 把条件移进 `ON` |
| `LEFT JOIN` 后用 `WHERE 右表列 = 某值` | 同上 | 同上 |
| 用 `INNER JOIN` 查层级关系的顶层 | 顶层节点（父键为 NULL）被丢掉 | 改用 `LEFT JOIN` |
| 自连接配对用 `<>` | 每对出现两次 | 用 `<` 或 `>` |
| 自连接忘加别名 | 列名歧义报错 | 必须给同一张表起两个别名 |
| 用 `NATURAL JOIN` | 加一个同名列就悄悄改变语义 | 用 `ON` 或 `USING` |
| 反连接时用右表的可空列判断 | 该列在匹配行里也可能是 NULL，造成误判 | 用右表的**主键**列 |
| 反连接后忘了去重 | 若右表有多行匹配会返回重复 | 用 `NOT EXISTS`（天然不重复）或用 `DISTINCT` |
| MySQL 里写 `FULL OUTER JOIN` | 语法不支持，报错 | 用 `LEFT JOIN UNION RIGHT JOIN` |
| 模拟 FULL JOIN 时用了 `UNION ALL` | 匹配的行出现两次 | 用 `UNION` |
| 在连接条件里用函数包列 | 用不上索引，优化器估算失准 | 让列保持"裸露" |
| 用 `FULL JOIN` 做对账时直接输出 | 无法区分"只在 A""只在 B""值不同" | 用 `COALESCE` + `CASE` 分类（见 6.1 模板） |

## 小结

- 连接的本质是**笛卡尔积 + 过滤**；忘写连接条件会静默产生行数爆炸，这是最危险的一类错误。
- `INNER` 只保留匹配；`LEFT` 保留左表全部（右表缺失处补 NULL）；`RIGHT` 保留右表全部；`FULL OUTER` 两边都保留。**`RIGHT JOIN` 可以总改写成交换顺序的 `LEFT JOIN`，建议统一用 `LEFT JOIN`。**
- **`ON` 决定"怎么匹配"，`WHERE` 决定"匹配完之后保留哪些行"。** 在 `LEFT JOIN` 的 `WHERE` 里过滤**右表**的列会把外连接退化成内连接（不报错，只是少了行）；过滤**左表**的列是安全的。
- **反连接模式**（找出"不存在"的记录）：`LEFT JOIN ... WHERE 右表主键 IS NULL`，或者更推荐的 `NOT EXISTS`。**`NOT EXISTS` 不会因为右表多行匹配而产生重复，更不容易写错。**
- 用反连接时应当用**右表的主键列**判断 `IS NULL`，而不是任意可空列。
- MySQL **不支持 `FULL OUTER JOIN`**，要用 `LEFT JOIN UNION RIGHT JOIN` 模拟，且必须用 `UNION`（不是 `UNION ALL`）才能去掉重复的匹配行。
- `FULL OUTER JOIN` 做对账时用 `COALESCE(a.id, b.id)` 取键，用 `CASE` 区分"只在 A / 只在 B / 值不同"。
- `USING (col)` 会把连接列**合并成一列**，`ON a.col = b.col` 会保留两列；**`NATURAL JOIN` 应当避免使用**，因为它隐式按所有同名列连接，加一列就可能悄悄改变语义。
- **自连接必须用两个别名**；做"配对"时用 `e1.id < e2.id` 而不是 `<>`，可以同时避免重复对和自配对。
- **层级查询（员工-经理、分类树）的顶层节点父键为 NULL，必须用 `LEFT JOIN` 才能保住**，用 `INNER JOIN` 会把它排除。
- 自连接里的 `NULL` 不会互相匹配（`NULL = NULL` 是 UNKNOWN），需要时显式写 `(a IS NULL AND b IS NULL)`。
- `CROSS JOIN` 的正当用途是**制造完整框架**（所有学生 × 所有课程、日期 × 产品），配合 `LEFT JOIN` 实现"不存在的组合也显示为 0"。
- 连接性能的基本功：在连接列上建索引（**注意 PostgreSQL 不会为外键自动建索引，MySQL 会**）、不在连接条件里用函数包列、保持统计信息更新，并用 `EXPLAIN` 观察实际计划。
- `EXPLAIN` 的输出格式因数据库与版本差异很大，**理解"是否走索引、预估与实际行数差异、连接算法、有无排序/临时表"这些关注点，比背输出格式有用。**
