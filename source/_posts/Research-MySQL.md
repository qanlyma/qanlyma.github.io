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

* Server 层负责建立连接、分析和执行 SQL。所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

* 存储引擎层负责数据的存储和提取。支持 InnoDB、MyISAM、Memory 等多个存储引擎，不同的存储引擎共用一个 Server 层。现在最常用的存储引擎是 InnoDB，它从 MySQL 5.5.5 版本开始成为了默认存储引擎。

### 1.1 连接器

连接器负责跟客户端建立连接、获取权限、维持和管理连接。连接命令一般是这么写的：

```SQL
mysql -h$ip -P$port -u$user -p
```

连接命令中的 mysql 是客户端工具，用来跟服务端建立连接。在完成经典的 TCP 握手后，连接器就要开始认证你的身份，这个时候用的就是你输入的用户名和密码。

数据库里面，长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。建立连接的过程通常是比较复杂的，所以在使用中要尽量减少建立连接的动作，也就是尽量使用长连接。但是，使用长连接后可能会占用内存增多，因为 MySQL 在执行查询过程中临时使用内存管理连接对象，这些连接对象资源只有在连接断开时才会释放。可以使用定期断开长连接或者客户端主动重置连接的方式释放内存。

### 1.2 查询缓存

MySQL 拿到一个查询请求（select 语句）后，会先查询缓存。之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。key 是查询的语句，value 是查询的结果。

查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。因此很可能你费劲地把结果存起来，还没使用呢，就被一个更新全清空了。对于更新压力大的数据库来说，查询缓存的命中率会非常低。除非你的业务就是有一张静态表，很长时间才会更新一次。比如一个系统配置表，那这张表上的查询才适合使用查询缓存。

需要注意的是，MySQL 8.0 版本直接将查询缓存的整块功能删掉了。

### 1.3 分析器

MySQL 需要知道你要做什么，因此需要对 SQL 语句做解析。

* 词法分析：MySQL 会根据你输入的字符串识别出关键字，构建 SQL 语法树。

* 语法分析：根据词法分析的结果，语法解析器会根据语法规则，判断你输入的这个 SQL 语句是否满足 MySQL 语法。

### 1.4 优化器

优化器主要负责将 SQL 查询语句的执行方案确定下来，比如在表里面有多个索引的时候，优化器会基于查询成本的考虑，来决定选择使用哪个索引。

### 1.5 执行器

经历完优化器后，就确定了执行方案，接下来 MySQL 就真正开始执行语句了，这个工作是由执行器完成的。在执行的过程中，执行器会和存储引擎交互，交互是以记录为单位的。

### 1.6 存储

一张数据库表的数据是保存在 `表名字.ibd` 的文件里的，这个文件也称为独占表空间文件。表空间由段（segment）、区（extent）、页（page）、行（row）组成，InnoDB 存储引擎的逻辑存储结构大致如下图：

![](3.png)

* **行（row）**
    
    数据库表中的记录都是按行（row）进行存放的，每行记录根据不同的行格式，有不同的存储结构。

* **页（page）**
    
    记录是按照行来存储的，但是数据库的读取并不以「行」为单位，否则一次读取（也就是一次 I/O 操作）只能处理一行数据，效率会非常低。
    
    因此，InnoDB 的数据是按「页」为单位来读写的，当需要读一条记录的时候，并不是将这个行记录从磁盘读出来，而是以页为单位，将其整体读入内存。
    
    默认每个页的大小为 16KB，也就是最多能保证 16KB 的连续存储空间。

* **区（extent）**

    InnoDB 存储引擎是用 B+ 树来组织数据的。

    B+ 树中每一层都是通过双向链表连接起来的，如果是以页为单位来分配存储空间，那么链表中相邻的两个页之间的物理位置并不是连续的，可能离得非常远，那么磁盘查询时就会有大量的随机 I/O，随机 I/O 是非常慢的。

    解决这个问题也很简单，就是让链表中相邻的页的物理位置也相邻，这样就可以使用顺序 I/O 了，那么在范围查询（扫描叶子节点）的时候性能就会很高。

    在表中数据量大的时候，为某个索引分配空间的时候就不再按照页为单位分配了，而是按照区（extent）为单位分配。每个区的大小为 1MB，对于 16KB 的页来说，连续的 64 个页会被划为一个区，这样就使得链表中相邻的页的物理位置也相邻，就能使用顺序 I/O 了。

* **段（segment）**
    
    表空间是由各个段（segment）组成的，段是由多个区（extent）组成的。段一般分为数据段、索引段和回滚段等。

    * 索引段：存放 B + 树的非叶子节点的区的集合
    * 数据段：存放 B + 树的叶子节点的区的集合
    * 回滚段：存放的是回滚数据的区的集合

## 2 日志模块

更新语句的流程会涉及到 undo log、redo log 、binlog 这三种日志：

* undo log（回滚日志）：是 Innodb 存储引擎层生成的日志，实现了原子性，主要用于**事务回滚**和 **MVCC**
* redo log（重做日志）：是 Innodb 存储引擎层生成的日志，实现了持久性，主要用于掉电等**故障恢复**
* binlog（归档日志）：是 Server 层生成的日志，主要用于**数据备份**和**主从复制**

### 2.1 undo log

一个事务在执行过程中，在还没有提交事务之前，如果 MySQL 发生了崩溃，可以通过 undo log 回滚到事务之前的数据。

undo log 是一种用于撤销回退的日志。在事务没提交之前，MySQL 会先记录更新前的数据到 undo log 日志文件里面，当事务回滚时，可以利用 undo log 来进行回滚。发生回滚时，就读取 undo log 里的数据，然后做原先相反操作。比如当 delete 一条记录时，undo log 中会把记录中的内容都记下来，然后执行回滚操作的时候，就读取 undo log 里的数据，然后进行 insert 操作。

另外，undo log 还有一个作用，通过 ReadView + undo log 实现 MVCC（多版本并发控制）。

undo log 为每条记录保存多份历史数据，MySQL 在执行快照读（普通 select 语句）的时候，会根据事务的 Read View 里的信息，顺着 undo log 的版本链找到满足其可见性的记录。

### 2.2 redo log

MySQL 如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录更新，整个过程 IO 成本、查找成本都很高。为了解决这个问题使用了 WAL（Write-Ahead Logging）技术，它的关键点就是先写日志，再写磁盘。

当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log 里面，并更新内存。InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。重做日志文件组是以循环写的方式工作的，从头开始写，写到末尾就又回到开头，相当于一个环形。

redo log 将写操作从「随机写」变成了「顺序写」，提升 MySQL 写入磁盘的性能。

![](4.png)

有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。

**redo log 与 undo log 的区别**：

* redo log 记录了此次事务「完成后」的数据状态，记录的是更新之后的值
* undo log 记录了此次事务「开始前」的数据状态，记录的是更新之前的值
* 事务提交之前发生了崩溃，重启后会通过 undo log 回滚事务，事务提交之后发生了崩溃，重启后会通过 redo log 恢复事务

### 2.3 binlog

MySQL 在完成一条更新操作后，Server 层还会生成一条 binlog，等之后事务提交的时候，会将该事物执行过程中产生的所有 binlog 统一写 入 binlog 文件。

binlog 文件是记录了所有数据库表结构变更和表数据修改的日志，不会记录查询类的操作，比如 SELECT 和 SHOW 操作。

**redo log 与 binlog 的区别**：

* redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
* redo log 是物理日志，记录的是"在某个数据页上做了什么修改"；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如"给ID=2这一行的c字段加1"。
* redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

不可以使用 redo log 文件恢复，只能使用 binlog 文件恢复。因为 redo log 文件是循环写，会边写边擦除日志，只记录未被刷入磁盘的数据的物理日志，已经刷入磁盘的数据都会从 redo log 文件里擦除。binlog 文件保存的是全量的日志，也就是保存了所有数据变更的情况，理论上只要记录在 binlog 上的数据，都可以恢复。

MySQL 的主从复制依赖于 binlog，复制的过程就是将 binlog 中的数据从主库传输到从库上：

1. 写入 Binlog：主库写 binlog 日志，提交事务，并更新本地存储数据。
2. 同步 Binlog：把 binlog 复制到所有从库上，每个从库把 binlog 写到暂存日志中。
3. 回放 Binlog：回放 binlog，并更新存储引擎中的数据。

在完成主从复制之后，就可以在写数据时只写主库，在读数据时只读从库，这样即使写请求会锁表或者锁记录，也不会影响读请求的执行。

### 2.4 执行一条更新

具体更新一条记录 `UPDATE t_user SET name = 'xiaolin' WHERE id = 1;` 的流程如下：

1. 执行器负责具体执行，会调用存储引擎的接口，通过主键索引树搜索获取 id = 1 这一行记录：
    * 如果 id=1 这一行所在的数据页本来就在 buffer pool 中，就直接返回给执行器更新；
    * 如果记录不在 buffer pool，将数据页从磁盘读入到 buffer pool，返回记录给执行器；
2. 执行器得到聚簇索引记录后，会看一下更新前的记录和更新后的记录是否一样：
    * 如果一样的话就不进行后续更新流程；
    * 如果不一样的话就把更新前的记录和更新后的记录都当作参数传给 InnoDB 层，让 InnoDB 真正的执行更新记录的操作；
3. 开启事务，InnoDB 层更新记录前，首先要记录相应的 undo log，因为这是更新操作，需要把被更新的列的旧值记下来。undo log 会写入 Buffer Pool 中的 Undo 页面，不过在内存修改该 Undo 页面后，需要记录对应的 redo log。
4. InnoDB 层开始更新记录，会先更新内存（同时标记为脏页），然后将记录写到 redo log 里面，为了减少磁盘 I/O，不会立即将脏页写入磁盘，后续由后台线程选择一个合适的时机将脏页写入到磁盘。
5. 至此，一条记录更新完了。
6. 在一条更新语句执行完成后，然后开始记录该语句对应的 binlog，此时记录的 binlog 会被保存到 binlog cache，并没有刷新到硬盘上的 binlog 文件，在事务提交时才会统一将该事务运行过程中的所有 binlog 刷新到硬盘。
7. 事务两阶段提交。

### 2.5 两阶段提交

![update T set c=c+1 where ID=2](2.png)

图中浅色框表示是在 InnoDB 内部执行的，深色框表示是在执行器中执行的。redo log 的写入拆成了两个步骤：prepare 和 commit，这就是"两阶段提交"。

**两阶段提交是为了让两份日志之间的逻辑一致**。

binlog 会采用追加写记录所有的逻辑操作，如果 DBA 承诺说半个月内可以恢复，那么备份系统中一定会保存最近半个月的所有 binlog，同时系统会定期做整库备份，可以是一天一备，也可以是一周一备。当需要恢复到指定的某一秒时，你可以这么做：

1. 找到最近的一次全量备份，从这个备份恢复到临时库。
2. 从备份的时间点开始，将备份的 binlog 依次取出来，重放到中午误删表之前的那个时刻。

由于 redo log 和 binlog 是两个独立的逻辑，如果不用两阶段提交，要么就是先写完 redo log 再写 binlog，或者采用反过来的顺序。我们看看这两种方式会有什么问题。

假设当前 ID=2 的行，字段 c 的值是 0，再假设执行 update 语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了 crash：

* 先写 redo log 后写 binlog。假设在 redo log 写完，binlog 还没有写完的时候，MySQL 进程异常重启。由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同。

* 先写 binlog 后写 redo log。如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了"把 c 从 0 改成 1"这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。

## 3 索引

数据库索引是帮助存储引擎快速获取数据的一种数据结构，索引和数据位于存储引擎中。



## 4 事务

事务就是要保证一组数据库操作，要么全部成功，要么全部失败。在 MySQL 中，事务支持是在引擎层实现的，但并不是所有的引擎都支持事务。比如 MySQL 原生的 MyISAM 引擎就不支持事务，这也是其被 InnoDB 取代的重要原因之一。

## 5 锁




## 6 SQL

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