---
title: SQL
date: 2020-01-16 13:59:07
tags: [database, SQL]
toc: true
description: "SQL基础知识"
keywords: [database, sql, 题解]
---

SQL基础语句快查

## 一、定义

**结构化查询语言**（**S**tructured **Q**uery **L**anguage，**SQL**）。是一种 ANSI（American National Standards Institute）标准的计算机语言。目的是访问和处理数据库。

## 二、查询 - SELECT

Select 查询某些属性列（specific columns）的语法：

```sql
SELECT column（列名）, another_column, …
FROM mytable（表名）;
```

Select 查询所有列：

```sql
SELECT *
FROM mytable（表名）;
```

## 三、条件查询  - WHERE 

条件查询语法，WHERE：

```SQL
SELECT column, another_column, …
FROM mytable
WHERE condition
    AND/OR another_condition
    AND/OR …;
```

参考下表：

<br>

| Operator（关键字）  | Condition（意思）                                            | **SQL Example(例子）**        |
| ------------------- | ------------------------------------------------------------ | ----------------------------- |
| =, !=, < <=, >, >=  | Standard numerical operators 基础的 大于，等于等比较         | col_name != 4                 |
| BETWEEN … AND …     | Number is within range of two values (inclusive) 在两个数之间 | col_name BETWEEN 1.5 AND 10.5 |
| NOT BETWEEN … AND … | Number is not within range of two values (inclusive) 不在两个数之间 | col_name NOT BETWEEN 1 AND 10 |
| IN (…)              | Number exists in a list 在一个列表                           | col_name IN (2, 4, 6)         |
| NOT IN (…)          | Number does not exist in a list 不在一个列表                 | col_name NOT IN (1, 3, 5)     |

##  四、模糊查询 - LIKE

参考下表：

<br>

| Operator（操作符） | Condition（解释）                                            | Example（例子）                                              |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| =                  | Case sensitive exact string comparison (*notice the single equals*)完全等于 | col_name = "abc"                                             |
| != or <>           | Case sensitive exact string inequality comparison 不等于     | col_name != "abcd"                                           |
| LIKE               | Case insensitive exact string comparison 没有用通配符等价于 = | col_name LIKE "ABC"                                          |
| NOT LIKE           | Case insensitive exact string inequality comparison 没有用通配符等价于 != | col_name NOT LIKE "ABCD"                                     |
| %                  | Used anywhere in a string to match a sequence of zero or more characters (only with LIKE or NOT LIKE) 通配符，代表匹配0个以上的字符 | col_name LIKE "%AT%" (matches "AT", "ATTIC", "CAT" or even "BATS") "%AT%" 代表AT 前后可以有任意字符 |
| _                  | Used anywhere in a string to match a single character (only with LIKE or NOT LIKE) 和% 相似，代表1个字符 | col_name LIKE "AN_" (matches "AND", but not "AN")            |
| IN (…)             | String exists in a list 在列表                               | col_name IN ("A", "B", "C")                                  |
| NOT IN (…)         | String does not exist in a list 不在列表                     | col_name NOT IN ("D", "E", "F")                              |

## 五、过滤 - DISTINCT

选取出唯一的结果的语法：

```sql
SELECT DISTINCT column, another_column, … 
FROM mytable 
WHERE condition(s);
```

## 六、排序 - ORDER

结果排序（ordered results）：

```sql
SELECT column, another_column, … 
FROM mytable 
WHERE condition(s)
ORDER BY column1 ASC/DESC, column2 ASC/DESC;
```

`ASC`：升序；`DESC`：降序。

## 七、选取 - LIMIT & OFFSET

`LIMIT` 和 `OFFSET` 子句通常和 `ORDER BY` 语句一起使用，当我们对整个结果集排序之后，我们可以 `LIMIT` 来指定只返回多少行结果，用 `OFFSET` 来指定从哪一行开始返回：

```sql
SELECT column, another_column, …
FROM table1
WHERE condition(s)
ORDER BY column ASC/DESC, another_column ASC/DESC, …
LIMIT num_limit OFFSET num_offset;
```

## 八、多表联合 - JOIN

主键（primary key），一般关系数据表中，都会有一个属性列设置为 `primary key`。主键是唯一标识一条数据的，不会重复复（想象你的身份证号码)。一个最常见的主键就是 auto-incrementing integer（自增 ID，每写入一行数据 ID + 1, 当然字符串，hash 值等只要是每条数据是唯一的也可以设为主键。借助 `primary key`（当然其他唯一性的属性也可以），我们可以把两个表中具有相同主键 ID 的数据连接起来。

![joins](https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/joins.png)

### INNER JOIN

用 INNER JOIN 连接表的语法。通过 `ON` 条件描述的关联关系；`INNER JOIN` 先将两个表数据连接到一起. 两个表中如果通过 ID 互相找不到的数据将会舍弃：

```sql
SELECT table1.column, table2.another_table_column, … 
FROM table1 （主表） 
INNER JOIN table2 （要连接的表）    
	ON table1.id = table2.matching_id 
WHERE condition(s) 
ORDER BY table1.column, … ASC/DESC 
LIMIT num_limit OFFSET num_offset;
```

### OUTER JOINs

`INNER JOIN` 只会保留两个表都存在的数据（还记得之前的交集吗），这看起来意味着一些数据的丢失，在某些场景下会有问题.

用 LEFT/RIGHT/FULL JOINs 做多表查询：

```sql
SELECT table1.column, table2.another_table_column, … 
FROM table1 （主表） 
LEFT/RIGHT/FULL JOIN table2 （要连接的表）    
	ON table1.id = table2.matching_id 
WHERE condition(s) 
ORDER BY table1.column, … ASC/DESC 
LIMIT num_limit OFFSET num_offset;
```

## 九、表达式 - AS

当我们用表达式对 col 属性计算时，很多事可以在 SQL 内完成，这让 SQL 更加灵活，但表达式如果长了则很难一下子读懂。所以 SQL 提供了`AS`关键字， 来给表达式取一个别名：

```sql
SELECT col_expression AS expr_description
FROM table1 AS newTableName
WHERE condition(s)
ORDER BY expr_description ASC/DESC
LIMIT num_limit OFFSET num_offset;
```

## 十、统计

下面介绍几个常用统计函数：

<br>

| Function                            | Description                                                  |
| ----------------------------------- | ------------------------------------------------------------ |
| **COUNT(*****)**, **COUNT(column)** | 计数！COUNT(*) 统计数据行数，COUNT(column) 统计column非NULL的行数. |
| **MIN(column)**                     | 找column最小的一行.                                          |
| **MAX(column)**                     | 找column最大的一行.                                          |
| **AVG(column)**                     | 对column所有行取平均值.                                      |
| **SUM(column)**                     | 对column所有行求和.                                          |

当不求整组数据时，可以使用 `GROUP BY` 对数据分组：

```sql
SELECT column, count(*)*100/(SELECT count(*) FROM table1) FROM table1 
WHERE column in (1,3,5,7)
GROUP BY column;
```

可以通过嵌套一个 select 来实现百分比的求解。

数据库是先对数据做 `WHERE`，然后对结果做分组，如果我们要对分组完的数据再筛选就需要使用 `HAVING` 语法：

```sql
SELECT group_by_column, AGG_FUNC(column_expression) AS aggregate_result_alias, …
FROM mytable
WHERE condition
GROUP BY column
HAVING group_condition;
```

## 十一、总结

完整的 SELECT 查询：

```sql
SELECT DISTINCT column, AGG_FUNC(column_or_expression), …
FROM mytable
JOIN another_table
    ON mytable.column = another_table.column
WHERE constraint_expression
GROUP BY column
HAVING constraint_expression
ORDER BY column ASC/DESC
LIMIT count OFFSET COUNT;
```

**1. FROM 和 JOINs**

`FROM` 或 `JOIN` 会第一个执行，确定一个整体的数据范围. 如果要 JOIN 不同表，可能会生成一个临时 Table 来用于 下面的过程。总之第一步可以简单理解为确定一个数据源表（含临时表)。

**2. WHERE**

我们确定了数据来源 `WHERE` 语句就将在这个数据源中按要求进行数据筛选，并丢弃不符合要求的数据行，所有的筛选 col 属性 只能来自 `FROM` 圈定的表. AS 别名还不能在这个阶段使用，因为可能别名是一个还没执行的表达式。

**3. GROUP BY**

如果你用了 `GROUP BY` 分组，那 `GROUP BY` 将对之前的数据进行分组，统计等，并将是结果集缩小为分组数.这意味着 其他的数据在分组后丢弃。

**4. HAVING**

如果你用了 `GROUP BY` 分组，`HAVING` 会在分组完成后对结果集再次筛选。AS 别名也不能在这个阶段使用。

**5. SELECT**

确定结果之后，`SELECT` 用来对结果 col 简单筛选或计算，决定输出什么数据。

**6. DISTINCT**

如果数据行有重复 `DISTINCT` 将负责排重。

**7. ORDER BY**

在结果集确定的情况下，`ORDER BY` 对结果做排序。因为 `SELECT` 中的表达式已经执行完了。此时可以用 AS 别名。

**8. LIMIT / OFFSET**

最后 `LIMIT` 和 `OFFSET` 从排序的结果中截取部分数据。

## 十二、题解

[leetcode 176. 第二高的薪水](https://leetcode-cn.com/problems/second-highest-salary/)

```mysql
SELECT (
    SELECT 
    	DISTINCT Salary
    FROM
        Employee
    ORDER BY
        Salary DESC
    LIMIT 1 OFFSET 1
    )  AS SecondHighestSalary
```

或者使用`IFNULL`：

```mysql
SELECT 
    IFNULL (
        (SELECT DISTINCT Salary
        FROM Employee
        ORDER BY Salary DESC
        LIMIT 1 OFFSET 1),
        NULL
    ) AS SecondHighestSalary
```

[leetcode 177. 第N高的薪水](https://leetcode-cn.com/problems/nth-highest-salary/)

```mysql
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
    SET N = N - 1;
    RETURN (
        SELECT (
            SELECT DISTINCT Salary
            FROM Employee
            ORDER BY Salary DESC
            LIMIT 1 OFFSET N
        )
    );
END
```

[leetcode 178. 分数排名](https://leetcode-cn.com/problems/rank-scores/)

自连接：

```mysql
SELECT
    S1.score AS Score,
    COUNT(DISTINCT S2.score) AS `Rank`
FROM
    Scores S1
    INNER JOIN Scores S2
    ON S1.score <= S2.score
GROUP BY
    S1.id
ORDER BY
    S1.score DESC;
```

子查询：

```mysql
SELECT
    S1.Score AS Score,
    (SELECT 
        COUNT(DISTINCT S2.Score)
    FROM Scores S2
    WHERE S1.score <= S2.score) AS `Rank`
FROM
    Scores S1
ORDER BY
    S1.score DESC;
```

[leetcode 181. 超过经理收入的员工](https://leetcode-cn.com/problems/employees-earning-more-than-their-managers/)

```mysql
SELECT E1.Name as Employee
FROM Employee E1 
INNER JOIN Employee E2
ON E1.ManagerID = E2.Id
AND E1.Salary > E2.Salary
```

[leetcode 182. 查找重复的电子邮箱](https://leetcode-cn.com/problems/duplicate-emails/)

```mysql
SELECT Email
FROM Person
GROUP BY Email
HAVING count(*) > 1
```

[leetcode 183. 从不订购的客户](https://leetcode-cn.com/problems/customers-who-never-order/)

```mysql
SELECT C.Name AS Customers
FROM Customers C
LEFT JOIN Orders O
ON C.Id = O.CustomerId
WHERE O.Id is null
```

[leetcode 184. 部门工资最高的员工](https://leetcode-cn.com/problems/department-highest-salary/)

```mysql
SELECT 
    D.Name AS Department, 
    E.Name AS Employee,
    E.Salary
FROM
    Employee E
    JOIN 
    	Department D
    ON 
    	E.DepartmentId = D.Id
WHERE
    (E.Salary, E.DepartmentId) 
    IN (
        SELECT
            MAX(Salary), DepartmentId
        FROM
            Employee
        GROUP BY
            DepartmentId
    )
```

[leetcode 185. 部门工资前三高的所有员工](https://leetcode-cn.com/problems/department-top-three-salaries/)

```mysql
SELECT 
    D.Name AS Department,
    E.Name AS Employee,
    E.Salary
FROM
    Employee E
    JOIN Department D
    ON E.DepartmentId = D.Id
WHERE
    (SELECT COUNT(DISTINCT Salary)
    FROM Employee EE
    WHERE EE.DepartmentId = E.DepartmentId
        AND EE.Salary >= E.Salary) <= 3
```

[leetcode 196. 删除重复的电子邮箱](https://leetcode-cn.com/problems/delete-duplicate-emails/)

```mysql
DELETE P2
FROM Person P1, Person P2
WHERE P1.Id < P2.Id 
    AND P1.Email = P2.Email
```

[leetcode 620. 有趣的电影](https://leetcode-cn.com/problems/not-boring-movies/)

```mysql
SELECT id, movie, description, rating
FROM cinema
WHERE id % 2 = 1 AND description != 'boring'
ORDER BY rating DESC
```

[leetcode 627. 交换工资](https://leetcode-cn.com/problems/swap-salary/)

```mysql
UPDATE salary
SET sex = CHAR(ASCII(sex) ^ ASCII('f') ^ ASCII('m'));
```

[leetcode 626. 换座位](https://leetcode-cn.com/problems/exchange-seats/)

```mysql
SELECT (CASE 
            WHEN MOD(id,2) = 1 AND id = (SELECT COUNT(*) FROM seat) THEN id
            WHEN MOD(id,2) = 1 THEN id+1
            ElSE id-1
        END) AS id, student
FROM seat
ORDER BY id;
```

## 参考

1. [SQL教程](http://www.xuesql.cn/)