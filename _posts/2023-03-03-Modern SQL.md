---
title: Modern SQL
tags: 数据库 SQL
sidebar:
  nav: layouts
---

CMU15-445 [Lecture #02: Modern SQL](https://15445.courses.cs.cmu.edu/spring2023/notes/02-modernsql.pdf)[译]

<!--more-->

## 关系语言（Relational Languages）

&emsp;&emsp;在 1970 年代初，埃德加·科德（Edgar Codd）发表了一篇关于关系模型的重要论文。最初，他仅为 DBMS 定义了在关系模型 DBMS 上执行查询的数学符号表示法。
用户只需使用声明性语言（Example SQL）指定他们想要的结果。DBMS 负责确定产生该答案的最有效计划。
关系代数基于集合（无序，无重复项）。SQL 基于多重集合（无序，允许重复项）。

## SQL 历史(SQL History)

&emsp;&emsp;**<font color = red>SQL（Structured Query Language）</font>**是一种面向关系数据库的声明性查询语言。它最早在 20 世纪 70 年代作为 IBM System R 项目的一部分开发出来。IBM 最初称其为“SEQUEL”（Structured English Query Language）。在 20 世纪 80 年代，它的名称改为了“SQL”（Structured Query Language）。

这种语言由不同类别的命令组成：

1. **数据操作语言（DML）**：包括 SELECT、INSERT、UPDATE 和 DELETE 语句。
2. **数据定义语言（DDL）**：用于定义表、索引、视图和其他对象的模式。
3. **数据控制语言（DCL）**：用于安全性和访问控制。

&emsp;&emsp;SQL 并不是一门过时的语言。它每隔几年都会更新新功能。SQL-92 是一个 DBMS 必须支持的最低标准，以宣称他们支持 SQL。每个供应商在一定程度上遵循这个标准，但也有许多专有扩展。

&emsp;&emsp;每个新版 SQL 标准发布时的一些主要更新如下：

- SQL:1999 正则表达式、触发器
- SQL:2003 XML、窗口函数、序列
- SQL:2008 截断、高级排序
- SQL:2011 时态数据库、流水线式 DML
- SQL:2016 JSON、多态表

## 连接操作(Joins)

&emsp;&emsp;将来自一个或多个表的列组合在一起，生成一个新的表。用于表达涉及跨多个表的数据的查询。

Example：哪些学生在 15-721 课程中获得了 A 分？

```sql
CREATE TABLE student (
	sid INT PRIMARY KEY,
	name VARCHAR(16),
	login VARCHAR(32) UNIQUE,
	age SMALLINT,
	gpa FLOAT
);
CREATE TABLE course (
	cid VARCHAR(32) PRIMARY KEY,
	name VARCHAR(32) NOT NULL
);
CREATE TABLE enrolled (
	sid INT REFERENCES student (sid),
	cid VARCHAR(32) REFERENCES course (cid),
	grade CHAR(1)
);
```

---

```sql
SELECT s.name
FROM enrolled AS e, student AS s
WHERE e.grade = 'A' AND e.cid = '15-721'
AND e.sid = s.sid;
```

## 聚合函数(Aggregates)

&emsp;&emsp;聚合函数以一组元组的包作为输入，然后生成一个标量值作为输出。聚合函数几乎只能在 SELECT 输出列表中使用。

- AVG(COL)：COL 中值的平均值
- MIN(COL)：COL 中的最小值
- MAX(COL)：COL 中的最大值
- COUNT(COL)：关系中的元组数

Example：获取具有'@cs'登录的学生人数。

以下三个查询是等价的：

```sql
SELECT COUNT(*) FROM student WHERE login LIKE '%@cs';
SELECT COUNT(login) FROM student WHERE login LIKE '%@cs';
SELECT COUNT(1) FROM student WHERE login LIKE '%@cs';
```

&emsp;&emsp;一个单独的 SELECT 语句可以包含多个聚合操作：

Example：获取使用‘@cs’登录的学生人数和他们的平均 GPA。

```sql
SELECT AVG(gpa), COUNT(sid)
FROM student WHERE login LIKE '%@cs';
```

&emsp;&emsp;一些聚合函数（Example COUNT、SUM、AVG）支持 DISTINCT 关键字：

Example：使用'@cs'登录获取独特学生人数和其平均 GPA。

```sql
SELECT COUNT(DISTINCT login)
FROM student WHERE login LIKE '%@cs';
```

&emsp;&emsp;聚合操作之外的其他列的输出是未定义的（下面的 e.cid 未定义）。

Example：获取每门课程中学生的平均 GPA。

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid;
```

&emsp;&emsp;在 SELECT 输出子句中未进行聚合的值必须出现在 GROUP BY 子句中。

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid;
```

&emsp;&emsp;HAVING 子句基于聚合计算来过滤输出结果。这使得 HAVING 的行为类似于 GROUP BY 的 WHERE 子句。

Example：获取平均学生 GPA 大于 3.9 的课程集合。

```sql
SELECT AVG(s.gpa) AS avg_gpa, e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid
HAVING avg_gpa > 3.9;
```

&emsp;&emsp;许多主要的数据库系统都支持上述查询语法，但它不符合 SQL 标准。为了使查询符合标准，我们必须在 HAVING 子句的主体中重复使用 AVG(S.GPA)。

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid
HAVING AVG(s.gpa) > 3.9;
```

## 字符串操作(String Operations)

&emsp;&emsp;SQL 标准指出，字符串区分大小写，并且只能使用单引号。有用于在查询的任何部分中使用的操作字符串的函数。

**<font color=red>模式匹配</font>**：使用 LIKE 关键字进行谓词中的字符串匹配。

- “%” 匹配任何子字符串（包括空字符串）。
- “ ” 匹配任何一个字符。

&emsp;&emsp;**<font color=red>字符串函数</font>** SQL-92 定义了字符串函数。许多数据库系统除了标准中的函数之外，还实现了其他函数。标准字符串函数 Example 包括 SUBSTRING(S, B, E) 和 UPPER(S)。

&emsp;&emsp;**<font color=red>连接</font>**：两个竖线将两个或多个字符串连接在一起，形成一个单一的字符串。

## 日期和时间(Date and Time)

&emsp;&emsp;用于操作日期和时间属性的操作。可以用于输出或谓词中。关于日期和时间操作的具体语法在不同系统之间变化很大。

## 输出重定向(Output Redirection)

&emsp;&emsp;与将查询结果返回给客户端（Example 终端）不同，您可以指示 DBMS 将结果存储到另一个表中。然后，您可以在后续的查询中访问这些数据。

- 新表：将查询的输出存储到一个新的（永久的）表中。

```sql
SELECT DISTINCT cid INTO CourseIds FROM enrolled;
```

- 现有表：将查询的输出存储到数据库中已存在的表中。目标表必须与目标表具有相同数量的列以及相同的数据类型，但是输出查询中列的名称不必匹配。

```sql
INSERT INTO CourseIds (SELECT DISTINCT cid FROM enrolled);
```

## 输出控制(Output Control)

&emsp;&emsp;由于 SQL 的结果是无序的，我们必须使用 ORDER BY 子句对元组进行排序：

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
ORDER BY grade;
```

&emsp;&emsp;默认的排序顺序是升序（ASC）。我们可以手动指定 DESC 来颠倒顺序：

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
ORDER BY grade DESC;
```

&emsp;&emsp;我们可以使用多个 ORDER BY 子句来解决并列情况或进行更复杂的排序：

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
ORDER BY grade DESC, sid ASC;
```

&emsp;&emsp;我们还可以在 ORDER BY 子句中使用任意的表达式：

```sql
SELECT sid FROM enrolled WHERE cid = '15-721'
ORDER BY UPPER(grade) DESC, sid + 1 ASC;
```

&emsp;&emsp;默认情况下，DBMS 将返回查询产生的所有元组。我们可以使用 LIMIT 子句来限制结果元组的数量：

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
LIMIT 10;
```

&emsp;&emsp;我们还可以提供一个偏移量来返回结果中的一个范围：

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
LIMIT 20 OFFSET 10;
```

&emsp;&emsp;除非我们在 LIMIT 中使用 ORDER BY 子句，否则 DBMS 可能会在每次查询调用时生成不同的元组，因为关系模型不强制排序。

## 嵌套查询(Nested Queries)

&emsp;&emsp;在其他查询内部调用查询，以在单个查询中执行更复杂的逻辑。嵌套查询通常难以优化。

&emsp;&emsp;外部查询的范围包含在内部查询中（即内部查询可以访问外部查询的属性），但反之不成立。

内部查询可以出现在查询的几乎任何部分：

1. SELECT 输出目标：

```sql
SELECT (SELECT 1) AS one FROM student;
```

2. FROM 子句：

```sql
SELECT name
FROM student AS s, (SELECT sid FROM enrolled) AS e
WHERE s.sid = e.sid;
```

3. WHERE 子句：

```sql
SELECT name FROM student
WHERE sid IN ( SELECT sid FROM enrolled );
```

Example：获取在‘15-445’课程中注册的学生的姓名。

```sql
SELECT name FROM student
WHERE sid IN (
SELECT sid FROM enrolled
WHERE cid = '15-445'
);
```

请注意，sid 的范围取决于它在查询中的出现位置。

Example：找到至少在一个课程中注册的具有最高 id 的学生记录。

```sql
SELECT student.sid, name
FROM student
JOIN (SELECT MAX(sid) AS sid
FROM enrolled) AS max*e
ON student.sid = max_e.sid;
```

**<font color=red>嵌套查询结果表达式</font>**：

- ALL：必须满足子查询中所有行的表达式。
- ANY：必须满足子查询中至少一行的表达式。
- IN：等同于=ANY()。
- EXISTS：至少返回一行。

Example：找到所有没有学生注册的课程。

```sql
SELECT \* FROM course
WHERE NOT EXISTS(
SELECT \_ FROM enrolled
WHERE course.cid = enrolled.cid
);
```

## 窗口函数(Window Functions)

&emsp;&emsp;窗口函数在一组相关的元组上执行“滑动”计算。类似于聚合，但元组不会被分组为单个输出元组。

**函数**：窗口函数可以是我们上面讨论过的任何聚合函数。还有特殊的窗口函数：

1. ROW NUMBER：当前行的编号。
2. RANK：当前行的排序位置。

**分组**：通过 OVER 子句指定在计算窗口函数时如何将元组分组在一起。使用 PARTITION BY 来指定分组。

Example：

```sql
SELECT cid, sid, ROW_NUMBER() OVER (PARTITION BY cid)
FROM enrolled ORDER BY cid;
```

&emsp;&emsp;我们还可以在 OVER 内部放置 ORDER BY，以确保结果具有确定的顺序，即使数据库在内部发生变化。

Example：

```sql
SELECT \*, ROW_NUMBER() OVER (ORDER BY cid) FROM enrolled ORDER BY cid;
```

重要提示：DBMS 在窗口函数排序后计算 RANK，而在排序前计算 ROW NUMBER。

Example：为每门课程找到第二高成绩的学生。

```sql
SELECT _ FROM (
SELECT _, RANK() OVER (PARTITION BY cid ORDER BY grade ASC) AS rank
FROM enrolled) AS ranking
WHERE ranking.rank = 2;
```

## 公用表达式（Common Table Expressions)

&emsp;&emsp;公用表达式（Common Table Expressions，CTEs）是在编写更复杂查询时，与窗口函数或嵌套查询相比的一种替代方案。它们提供了一种在更大查询中为用户编写辅助语句的方式。CTEs 可以被视为一个临时表，其作用范围限定为单个查询。

WITH 子句将内部查询的输出绑定到一个具有该名称的临时结果上。

Example：生成一个名为 cteName 的 CTE，其中包含一个单属性元组，属性设置为“1”。

```sql
WITH cteName AS (
SELECT 1
)
SELECT \* FROM cteName;
```

我们可以在 AS 之前将输出列绑定到名称上：

```sql
WITH cteName (col1, col2) AS (
SELECT 1, 2
)
SELECT col1 + col2 FROM cteName;
```

单个查询可以包含多个 CTE 声明：

```sql
WITH cte1 (col1) AS (SELECT 1), cte2 (col2) AS (SELECT 2)
SELECT \* FROM cte1, cte2;
```

&emsp;&emsp;在 WITH 之后添加 RECURSIVE 关键字允许 CTE 引用自身。这使得在 SQL 查询中实现递归成为可能。使用递归 CTE，SQL 可以被证明是图灵完备的，这意味着它在计算表达能力上与更通用的编程语言一样（尽管可能稍显繁琐）。

Example：打印从 1 到 10 的数字序列。

```sql
WITH RECURSIVE cteSource (counter) AS (
( SELECT 1 )
UNION
( SELECT counter + 1 FROM cteSource
WHERE counter < 10 )
)
SELECT \* FROM cteSource;
```
