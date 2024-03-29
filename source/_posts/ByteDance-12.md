---
title: 「训」 笔记(12)：存储和数据库
category_bar: true
date: 2023-05-03 19:55:34
tags:
categories: 字节青训
banner_img:
---

存储的本质 —— 状态。

<!-- more -->

## 1 存储系统

一个提供了读写、控制类接口，能够安全有效地把数据持久化的软件，就可以称为存储系统。

### 1.1 系统特点

* 作为后端软件的底座，性能敏感
* 存储系统软件架构，容易受硬件影响
* 存储系统代码，既“简单”又“复杂”

### 1.2 RAID 技术

单机存储系统怎么做到高性能/高性价比/高可靠性：R(edundant) A(rray) of I(nexpensive) D(isks)。

* RAID 0
    * 多块磁盘简单组合
    * 数据条带化存储，提高磁盘带宽
    * 没有额外的容错设计
  
* RAID 1
    * 一块磁盘对应一块额外镜像盘
    * 真实空间利用率仅 50%
    * 容错能力强

* RAID 0 + 1
    * 结合了 RAID 0 和 RAID 1
    * 真实空间利用率仅 50%
    * 容错能力强，写入带宽好

## 2 数据库

* **关系型数据库**
    关系型数据库是把数据以表的形式进行储存，然后再各个表之间建立关系，通过这些表之间的关系来操作不同表之间的数据。
    * 结构化数据友好
    * 支持事务（ACID）
    * 支持复杂查询语言

* **非关系型数据库**
    NoSQL 或非关系数据库，支持存储和操作非结构化及半结构化数据。相比于关系型数据库，NoSQL 没有固定的表结构，且数据之间不存在表与表之间的关系，数据之间可以是独立的。NoSQL 的关键是它们放弃了传统关系型数据库的强事务保证和关系模型，通过所谓最终一致性和非关系数据模型（例如键值对，图，文档）来提高 Web 应用所注重的高可用性和可扩展性。
    * 半结构化数据友好
    * **可能**支持事务（ACID）
    * **可能**支持复杂查询语言

### 2.1 结构化数据管理

![](1.png)

### 2.2 事物能力

* A(tomicity)，事务内的操作要么全做，要么不做
* C(onsistency)，事务执行前后，数据状态是一致的（数据完整，逻辑一致）
* I(solation)，可以隔离多个并发事务，避免影响
* D(urability)，事务一旦提交成功，数据保证持久性

### 2.3 复杂查询能力

![](2.png)

## 3 主流产品

### 3.1 单机存储

单个计算机节点上的存储软件系统，一般不涉及网络交互。

* **本地文件系统**：如 Linux 文件系统
    * Index Node 记录文件元数据，如 id、大小、权限、磁盘位置等。inode 是一个文件的唯一标识，会被存储到磁盘上，其总数在格式化文件系统时就固定了。
    * Directory Entry 记录文件名、inode 指针，层级关系（parent）等。dentry 是内存结构，与 inode 的关系是 N:1（hardlink的实现）。

* **key-value 存储**：如 RocksDB

### 3.2 分布式存储

在单机存储基础上实现了分布式协议，涉及大量网络交互。

* **HDFS**：支持海量数据存储，高容错性，弱 POSIX 语义，使用普通 x86 服务器，性价比高
* **Ceph**：一套系统支持对象接口、块接口、文件接口，但是一切皆对象，数据写入采用主备复制模型，数据分布模型采用 CRUSH 算法

### 3.3 单机数据库

单个计算机节点上的数据库系统。

* **关系型数据库**
    * Oracle
    * MySQL
    * PostgreSQL

* **非关系型数据库**
    * MongoDB
    * Redis
    * Elasticsearch

### 3.4 分布式数据库

* **容量**：单点容量有限，受硬件限制；存储节点池化，动态扩缩容
* **弹性**：方便扩缩容
* **性价比**：平衡容量与 CPU 利用率需求

## 4 MySQL

### 4.1 SQL

一种编程语言，目前几乎所有的关系数据库都使用 SQL (Structured Query Language) 编程语言来查询、操作和定义数据，进行数据访问控制。

![](3.png)

#### 4.1.1 SQL 引擎

* **查询解析 Parser**：解析器（Parser）一般分为词法分析（Lexical analysis）、语法分析（Syntax analysis）、语义分析（Semantic analyzer）等步骤。SQL 语言接近自然语言，入门容易。但是各种关键字、操作符组合起来，可以表达丰富的语意。因此想要处理 SQL 命令，首先将文本解析成结构化数据，也就是抽象语法树（AST）

* **查询优化 Optimizer**：SQL 是一门非过程化的语言，只说“做什么”，而不说“怎么做”。所以需要一些复杂的逻辑选择“如何拿数据”，也就是选择一个好的查询计划。优化器的作用根据 AST 优化产生最优执行计划（Plan Tree）

* **查询执行 Executor**：根据查询计划，完成数据读取、处理、写入等操作

#### 4.1.2 事务引擎

处理事务一致性、并发、读写隔离等。

* Atomicity：Undo Log 是逻辑日志，记录的是数据的增量变化。利用 Undo Log 可以进行事务回滚，从而保证事务的原子性。同时也实现了多版本并发控制（MVCC），解决读写冲突和一致性读的问题
* Isolation：锁、MVCC
* Durability：Redo Log 是物理日志，记录的是页面的变化，它的作用是保证事务持久化。如果数据写入磁盘前发生故障，重启 MySQL 后会根据 Redo Log 重做

#### 4.1.3 存储引擎
    
内存中的数据缓存区、数据文件、日志文件。

![](4.png)

* Buffer Pool
    
    ![](5.png)

    MySQL 中每个 chunk 的大小一般为 128M，每个 block 对应一个page，一个 chunk 下面有 8192 个 block。这样可以避免内存碎片化。分成多个 instance，可以有效避免并发冲突。

    Page id % instance num 得到它属于哪个 instance。

* Page
  
    ![](6.png)

* B+ Tree
  
    页目录中使用二分法快速定位到对应的槽，然后再遍历该槽对应分组中的记录即可快速找到指定的记录。
    ![](7.png)

## 5 Redis

Redis 用于解决关系型数据库性能问题，可以将经常访问的热数据放入内存中。

### 5.1 基本原理

![](8.png)

* 内存数据库，数据从内存中读写
* 数据保存到硬盘上防止重启数据丢失
    * 增量数据保存到 AOF 文件（Append Only File）
    * 全量数据保存到 RDB 文件（Redis Database Backup file）
* 单线程处理所有操作命令

### 5.2 数据结构

#### 5.2.1 字符串 - sds

最常规的 set/get 操作，Value 可以是 String 也可以是数字。一般做一些复杂的计数功能的缓存。

![](9.png)

#### 5.2.2 链表 - Quicklist

Quicklist 由一个双向链表和 listpack 实现。可以做简单的消息队列的功能。

![Quicklist](10.png)
![listpack](11.png)

#### 5.2.3 集合 - set

无序的字符串集合，不存在重复的元素。

因为 Set 堆放的是一堆不重复值的集合。所以可以做全局去重的功能。另外，就是利用交集、并集、差集等操作，可以计算共同喜好，全部的喜好，自己独有的喜好等功能。

Zset：有序集合，多了一个权重参数 Score，集合中的元素能够按 Score 进行排列。可以做排行榜应用，取 TOP N 操作。

#### 5.2.4 哈希表 - dict

这个结构体里定义了两个哈希表，在正常服务请求阶段，插入的数据，都会写入到 ht[0]，此时的 ht[1] 并没有被分配空间。随着数据逐步增多，会触发 rehash 操作用于扩大空间。

![](12.png)

rehash：给 ht[1] 分配空间，一般会比 ht[0] 大 2 倍。将 ht[0] 中的数据，全部迁移到 ht[1] 中。迁移完成后，ht[0] 的空间会被释放，并把 ht[1] 设置为 ht[0]，然后在 ht[1] 新创建一个空白的哈希表，为下次 rehash 做准备。数据量小的场景下，直接将数据从 ht[0] 拷贝到 ht[1] 速度是较快的。数据量大的场景，例如存有上百万的 KV 时，迁移过程将会明显阻塞用户请求。

渐进式 rehash：为避免出现这种情况，每次用户访问时都会迁移少量数据。将整个迁移过程，平摊到所有请求过程中。

#### 5.2.5 跳表 - zskiplist

链表在查找元素的时候，因为需要逐一查找，所以查询效率非常低，时间复杂度是 O(N)，于是就出现了跳表。跳表是在链表基础上改进过来的，实现了一种**多层**的有序链表，这样的好处是能快读定位数据。

![](13.png)

### 5.3 注意事项

#### 5.3.1 大 Key

String 类型 value 的字节数大于 10KB 即为大 key；Hash/Set/Zset/list 等复杂数据结构类型元素个数大于 5000 个或总 value 字节数大于 10MB 即为大 key。

* 读取成本高
* 容易导致慢查询（过期、删除）
* 主从复制异常，服务阻塞无法正常响应请求

**解决方法**：
1. 拆分：将大 key 拆分为小 key。例如一个 String 拆分成多个 String
2. 压缩：将 value 压缩后写入 redis，读取时解压后再使用
3. 集合类结构 hash、list、set、set
   1. 拆分：可以用 hash 取余、位掩码的方式决定放在哪个 key 中
   2. 区分冷热：如榜单列表场景使用 zset，只缓存前 10 页数据，后续数据读取 db

#### 5.3.2 热 Key

用户访问一个 Key 的 QPS 特别高，导致 Server 实例出现 CPU 负载突增或者不均的情况。

**解决方法**：
1. 设置 Localcache：在访问 Redis 前，在业务服务侧设置 LocalCache，降低访问 Redis 的 QPS。LocalCache 中缓存过期或未命中，则从 Redis 中将数据更新到 LocalCache
2. 拆分：将 key:value 这一个热 Key 复制写入多份，例如 keyl:value，key2:value，访问的时候访问多个 key，但 value 是同一个，以此将 QPS 分散到不同实例上，降低负载。代价是，更新时需要更新多个 key，存在数据短暂不一致的风险

#### 5.3.3 慢查询

**容易导致 Redis 慢查询的操作**：
1. 批量操作一次性传入过多的 key/value，如 mset/hmset/sadd/zadd 等 O(n) 操作建议单批次不要超过 100，超过 100 之后性能下降明显
2. zset 大部分命令都是 O(log(n))，当大小超过 5k 以上时，简单的 zadd/zrem 也可能导致慢查询
3. 操作的单个 value 过大，超过 10KB。即避免使用大 Key
4. 对大 key 的 delete/expire 操作也可能导致慢查询

#### 5.3.4 缓存穿透

热点数据查询绕过缓存，直接查询数据库。

1. 查询一个一定不存在的数据（穿透）：通常不会缓存不存在的数据，这类查询请求都会直接打到 db，如果有系统 bug 或人为攻击，那么容易导致 db 响应慢甚至宕机
2. 缓存过期时（击穿）：在高并发场景下，一个热 key 如果过期，会有大量请求同时击穿至 db，容易影响 db 性能和稳定。同一时间有大量 key 集中过期时，也会导致大量请求落到 db 上，导致查询变慢，甚至出现 db 无法响应新的查询

**解决方法**：
1. 缓存空值：如一个 id 在缓存和数据库中都不存在。则可以缓存一个空值，下次再查缓存直接反空值
2. 布隆过滤器：通过 bloom filter 算法来存储合法 Key，得益于该算法超高的压缩率，只需占用极小的空间就能存储大量 key 值
3. IP 拉黑，但可能会用不同的 IP 来攻击
4. 接口层增加参数合法性校验，如用户鉴权校验，id <= 0 的直接拦截

#### 5.3.5 缓存雪崩

大量缓存同时过期，导致大量请求全部打到数据库，造成数据库挂掉。

**解决方法**：
1. 设置过期时间：将缓存失效时间分散开，比如在原有的失效时间基础上增加一个随机值，例如不同 Key 过期时间，可以设置为 10 分 1 秒过期，10 分 23 秒过期，10 分 8 秒过期。单位秒部分就是随机时间，这样过期时间就分散了。对于热点数据，过期时间尽量设置得长一些，冷门的数据可以相对设置过期时间短一些
2. 使用缓存集群，避免单机宕机造成的缓存雪崩
3. 设置热点数据永远不过期。
4. 不断的用定时任务去刷新缓存

## 6 ClickHouse

### 6.1 列式存储

![行式存储](14.png)
![列式存储](15.png)
![](16.png)

**列存的优点**：

1. 数据压缩
   * 数据压缩可以使读的数据量更少，在 IO 密集型计算中获得大的性能优势
   * 相同类型压缩效率更高
   * 排序之后压缩效率更高
   * 可以针对不同类型使用不同的压缩算法
   * 几种常见的压缩算法：LZ4、Run-length encoding、Delta encoding

2. 数据选择
   * 可以选择特定的列做计算而不是读所有列，对聚合计算友好 

3. 延迟物化：
   * 尽可能推迟物化操作的发生（物化：将列数据转换为可以被计算或者输出的行数据或者内存数据结果的过程，物化后的数据通常可以用来做数据过滤，聚合计算，Join）
   * 缓存友好
   * CPU / 内存带宽友好
   * 可以利用到执行计划和算子的优化，例如 filter
   * 保留直接在压缩列做计算的机会

4. 向量化
   * SIMD（single instruction multiple data）：对于现代多核 CPU，其都有能力用一条指令执行多条数据。用 SIMD 指令完成的代码设计和执行的逻辑就叫做向量化
   * 数据格式要求：需要处理多个数据，因此数据需要是连续内存。需要明确数据类型
   * 执行模型要求：数据需要按批读取，函数的调用需要明确数据类型

### 6.2 ClickHouse 存储设计

![](17.png)

每个 column 都是一个文件，所有的 column 文件都在自己的 part 文件夹下

一个 part 有一个主键索引，每个 column 都有列索引

![](18.png)