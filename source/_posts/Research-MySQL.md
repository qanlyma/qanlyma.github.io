---
title: 「研」 MySQL 理解
category_bar: true
date: 2023-08-07 14:44:23
tags:
categories: 理论研究
banner_img:
---

MySQL 是最流行的关系型数据库管理系统，在 web 应用方面 MySQL 是最好的 RDBMS（Relational Database Management System，关系数据库管理系统）应用软件之一。

<!--more-->

## 1 基本架构

![MySQL 基本架构](1.png)

大体来说，MySQL 可以分为 Server 层和存储引擎层两部分。

* Server 层涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

* 存储引擎层负责数据的存储和提取。支持 InnoDB、MyISAM、Memory 等多个存储引擎，不同的存储引擎共用一个 Server 层。现在最常用的存储引擎是 InnoDB，它从 MySQL 5.5.5 版本开始成为了默认存储引擎。

### 1.1 Server 层

#### 1.1.1 连接器

连接器负责跟客户端建立连接、获取权限、维持和管理连接。连接命令一般是这么写的：

```SQL
mysql -h$ip -P$port -u$user -p
```

连接命令中的 mysql 是客户端工具，用来跟服务端建立连接。在完成经典的 TCP 握手后，连接器就要开始认证你的身份，这个时候用的就是你输入的用户名和密码。

数据库里面，长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。建立连接的过程通常是比较复杂的，所以在使用中要尽量减少建立连接的动作，也就是尽量使用长连接。

#### 1.1.2 查询缓存

MySQL 拿到一个查询请求后，会先到查询缓存看看，之前是不是执行过这条语句。之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。key 是查询的语句，value 是查询的结果。

查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。因此很可能你费劲地把结果存起来，还没使用呢，就被一个更新全清空了。对于更新压力大的数据库来说，查询缓存的命中率会非常低。除非你的业务就是有一张静态表，很长时间才会更新一次。比如一个系统配置表，那这张表上的查询才适合使用查询缓存。

需要注意的是，MySQL 8.0 版本直接将查询缓存的整块功能删掉了，也就是说 8.0 开始彻底没有这个功能了。

#### 1.1.3 分析器

MySQL 需要知道你要做什么，因此需要对 SQL 语句做解析。

* 词法分析：你输入的是由多个字符串和空格组成的一条 SQL 语句，MySQL 需要识别出里面的字符串分别是什么，代表什么。

* 语法分析：根据词法分析的结果，语法分析器会根据语法规则，判断你输入的这个 SQL 语句是否满足 MySQL 语法。

#### 1.1.4 优化器

经过了分析器，MySQL 就知道你要做什么了。在开始执行之前，还要先经过优化器的处理。

优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序，提高执行效率。

#### 1.1.5 执行器

开始执行的时候，要先判断一下你对这个表有没有执行查询的权限，如果有权限，就打开表继续执行。

### 1.2 日志模块

#### 1.2.1 redo log

MySQL 如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录更新，整个过程 IO 成本、查找成本都很高。为了解决这个问题使用了 WAL（Write-Ahead Logging）技术，它的关键点就是先写日志，再写磁盘。

当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log 里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。

有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。

#### 1.2.2 binlog

redo log 是 InnoDB 引擎特有的日志，而 Server 层也有自己的日志，称为 binlog（归档日志）。

**这两种日志有以下三点不同**：

* redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。

* redo log 是物理日志，记录的是"在某个数据页上做了什么修改"；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如"给ID=2这一行的c字段加1"。

* redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

![update T set c=c+1 where ID=2](2.png)

图中浅色框表示是在 InnoDB 内部执行的，深色框表示是在执行器中执行的。redo log 的写入拆成了两个步骤：prepare 和 commit，这就是"两阶段提交"。

**两阶段提交是为了让两份日志之间的逻辑一致**。

binlog 会采用追加写记录所有的逻辑操作，如果 DBA 承诺说半个月内可以恢复，那么备份系统中一定会保存最近半个月的所有 binlog，同时系统会定期做整库备份，可以是一天一备，也可以是一周一备。当需要恢复到指定的某一秒时，你可以这么做：

1. 找到最近的一次全量备份，从这个备份恢复到临时库。
2. 从备份的时间点开始，将备份的 binlog 依次取出来，重放到中午误删表之前的那个时刻。

由于 redo log 和 binlog 是两个独立的逻辑，如果不用两阶段提交，要么就是先写完 redo log 再写 binlog，或者采用反过来的顺序。我们看看这两种方式会有什么问题。

仍然用前面的 update 语句来做例子。假设当前 ID=2 的行，字段 c 的值是 0，再假设执行 update 语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了 crash：

* 先写 redo log 后写 binlog。假设在 redo log 写完，binlog 还没有写完的时候，MySQL 进程异常重启。由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同。

* 先写 binlog 后写 redo log。如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了"把 c 从 0 改成 1"这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。

## 2 事务



## 3 索引



## 4 锁




## 5 SQL

SQL（Structured Query Language）指结构化查询语言，可以访问和处理数据库，包括数据插入、查询、更新和删除。

### 命令

* SELECT - 从数据库中提取数据
* UPDATE - 更新数据库中的数据
* DELETE - 从数据库中删除数据
* INSERT INTO - 向数据库中插入新数据
* CREATE DATABASE - 创建新数据库
* ALTER DATABASE - 修改数据库
* CREATE TABLE - 创建新表
* ALTER TABLE - 变更（改变）数据库表
* DROP TABLE - 删除表
* CREATE INDEX - 创建索引（搜索键）
* DROP INDEX - 删除索引

### 语法

* SELECT 语句用于从数据库中选取数据。结果被存储在一个结果表中，称为结果集。
* SELECT DISTINCT 语句用于返回唯一不同的值。

```SQL
SELECT column1, column2, ...
FROM table_name;
```

* WHERE 子句用于提取那些满足指定条件的记录。
* LIKE 操作符用于在 WHERE 子句中搜索列中的指定模式。
* 通配符可用于替代字符串中的任何其他字符。
* IN 操作符允许您在 WHERE 子句中规定多个值。
* BETWEEN/NOT BETWEEN 操作符用于选取介于/不介于两个值之间的数据范围内的值。
* WHERE 关键字无法与聚合函数一起使用。HAVING 子句可以让我们筛选分组后的各组数据。
* WHERE column IS NULL 在 column 值为空时使用。

```SQL
SELECT column1, column2, ...
FROM table_name
WHERE condition;

SELECT column1, column2, ...
FROM table_name
WHERE column LIKE pattern;

SELECT * FROM Websites
WHERE url LIKE 'https%';

SELECT column1, column2, ...
FROM table_name
WHERE column IN (value1, value2, ...);

SELECT column1, column2, ...
FROM table_name
WHERE column BETWEEN value1 AND value2;

SELECT column_name, aggregate_function(column_name)
FROM table_name
WHERE column_name operator value
GROUP BY column_name
HAVING aggregate_function(column_name) operator value;
```

* AND & OR 运算符用于基于一个以上的条件对记录进行过滤。

```SQL
SELECT * FROM Websites
WHERE alexa > 15
AND (country='CN' OR country='USA');
```

* ORDER BY 关键字用于对结果集进行排序。
* ASC 表示按升序排序；DESC 表示按降序排序。
* GROUP BY 语句用于结合聚合函数，根据一个或多个列对结果集进行分组。

```SQL
SELECT column1, column2, ...
FROM table_name
ORDER BY column1, column2, ... ASC|DESC;

SELECT column_name, aggregate_function(column_name)
FROM table_name
WHERE column_name operator value
GROUP BY column_name;
```

* INSERT INTO 语句用于向表中插入新记录，column 可省略。
* SELECT INTO 语句从一个表复制数据，然后把数据插入到另一个新表中。
* MySQL 数据库不支持 SELECT INTO 语句，但支持 INSERT INTO ... SELECT。

```SQL
INSERT INTO table_name (column1,column2,column3,...)
VALUES (value1,value2,value3,...);

INSERT INTO table2
SELECT * FROM table1;
```

* UPDATE 语句用于更新表中已存在的记录。

```SQL
UPDATE table_name
SET column1 = value1, column2 = value2, ...
WHERE condition;
```

* DELETE 语句用于删除表中的记录。
* DROP 语句可以删除索引、表和数据库。

```SQL
DELETE FROM table_name
WHERE condition;
```

* SQL JOIN 子句用于把来自两个或多个表的行结合起来，基于这些表之间的共同字段。
* INNER JOIN：如果表中有至少一个匹配，则返回行
* LEFT JOIN：即使右表中没有匹配，也从左表返回所有的行
* RIGHT JOIN：即使左表中没有匹配，也从右表返回所有的行
* FULL JOIN：只要其中一个表中存在匹配，则返回行

```SQL
SELECT column1, column2, ...
FROM table1
JOIN table2
ON condition;
```

### 函数

* AVG() - 返回平均值
* COUNT() - 返回行数
* FIRST() - 返回第一个记录的值
* LAST() - 返回最后一个记录的值
* MAX() - 返回最大值
* MIN() - 返回最小值
* SUM() - 返回总和
* LEN() - 返回文本字段中值的长度（MySQL 中使用 LENGTH）