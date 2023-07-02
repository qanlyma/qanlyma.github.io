---
title: 「训」 笔记(11)：RPC 原理
category_bar: true
date: 2023-07-01 16:14:40
tags:
categories: 字节青训
banner_img:
---

RPC —— Remote Procedure Calls。

<!-- more -->

## 1 基本概念

### 1.1 远程函数调用

**RPC 需要解决的问题**：
1. 函数映射
    在本地调用中，函数体是直接通过函数指针来指定的，我们调用哪个方法，编译器就自动帮我们调用它相应的函数指针。但是在远程调用中，函数指针是不行的，因为两个进程的地址空间是完全不一样的。所以函数都有自己的一个 ID，在 RPC 的时候要附上这个 ID，还得有个 ID 和函数的对照关系表，通过 ID 找到对应的函数并执行。
2. 数据转换成字节流
    在本地调用中，我们只需要把参数压到栈里，然后让函数自己去栈里读就行。但是在远程调用时，客户端跟服务端是不同的进程，不能通过内存来传递参数。这时候就需要客户端把参数先转成一个字节流，传给服务端把字节流转成能读取的格式。
3. 网络传输

### 1.2 完整过程

![](1.png)

* IDL (Interface description language) 文件：IDL 是方法与参数的描述文件，它通过一种中立的方式来描述接口，使得在不同平台上运行的对象和用不同语言编写的程序可以相互通信

* 生成代码：通过编译器工具把 IDL 文件转换成语言对应的静态库

* 编解码：从内存中表示到字节序列的转换称为编码，反之为解码，也常叫做序列化和反序列化

* 通信协议：规范了数据在网络中的传输内容和格式。除必须的请求/响应数据外，通常还会包含额外的元数据

* 网络传输：通常基于成熟的网络库 TCP/UDP 传输

### 1.3 RPC 的优势

1. 单一职责，有利于分工协作和运维开发（可以采用不同语言）
2. 可扩展性强，资源使用率更优（底层基础服务可以复用）
3. 故障隔离，服务的整体可靠性更高

### 1.4 RPC 的问题

1. 服务宕机，对方应该如何处理？
2. 在调用过程中发生网络异常，如何保证消息的可达性？
3. 请求量突增导致服务无法及时处理，有哪些应对措施？

用 RPC 框架来解决这些问题。

## 2 分层设计

![](2.png)

### 2.1 编解码层

#### 2.1.1 生成代码

![](3.png)

#### 2.1.2 数据格式

* 语言特定的格式：许多编程语言都内建了将内存对象编码为字节序列的支持，例如 Java 有 java.io.Serializable。这种编码形式好处是非常方便，可以用很少的额外代码实现内存对象的保存与恢复，这类编码通常与特定的编程语言深度绑定，其他语言很难读取这种数据。
  
* 文本格式：JSON、XML、CSV 等文本格式，具有人类可读性。但是数字的编码多有歧义之处，比如 XML 和 CSV 不能区分数字和字符串，JSON 虽然区分字符串和数字，但是不区分整数和浮点数，而且不能指定精度。处理大量数据时，这个问题更严重了，没有强制模型约束，实际操作中往往只能采用文档方式来进行约定，这可能会给调试带来一些不便。由于 JSON 在一些语言中的序列化和反序列化需要采用反射机制，所以性能比较差。
  
* 二进制编码：具备跨语言和高性能等优点，常见有 Thrift 的 BinaryProtocol，Protobuf 等。

#### 2.1.3 二进制编码

**TLV 编码**

* Tag：标签，可以理解为类型
* Lenght：长度
* Value：值，Value 也可以是个 TLV 结构

![](4.png)

TLV 编码结构简单清晰，并且扩展性较好，但是由于增加了 Type 和 Length 两个冗余信息，有额外的内存开销，特别是在大部分字段都是基本类型的情况下有不小的空间浪费。

### 2.2 协议层

#### 2.2.1 概念

特殊结束符：过于简单，对于一个协议单元必须要全部读入才能够进行处理，除此之外必须要防止用户传输的数据不能同结束符相同，否则就会出现紊乱。HTTP 协议头就是以回车（CR）加换行（LF）符号序列结尾。
变长协议：一般都是自定义协议，由 header 和 payload 组成，会以定长加不定长的部分组成，其中定长的部分需要描述不定长的内容长度，使用比较广泛。

#### 2.2.2 构造

![](5.png)

* LENGTH (32bits)：数据包大小，不包含自身
* HEADER MAGIC (16bits)：标识版本信息，协议解析时候快速校验
* SEQUENCE NUMBER (32bits)：表示数据包的 seqID,可用于多路复用，单连接内递增
* HEADERSIZE (16bits)：头部长度，从第 14 个字节开始计算一直到 PAYLOAD 前
* PROTOCOL ID：编解码方式，有 Binary 和 Compact 两种
* TRANSFORM ID：压缩方式，如 zlib 和 snappy
* INFOID：传递一些定制的 meta 信息
* PAYLOAD：消息体

### 2.3 网络通信层

#### 2.3.1 Sockets API

![](6.png)
![](7.png)

套接字编程中的客户端必须知道两个信息：服务器的 IP 地址，端口号。

socket 函数创建一个套接字，bind 将一个套接字绑定到一个地址上，listen 监听进来的连接，accept 函数从队列中取出连接请求并接收它。

connect 客户端向服务器发起连接，accept 接收一个连接请求，如果没有连接则会一直阻塞直到有连接进来。调用 read，write 函数和客户端通讯，读写方式和其他 I/O 类似。

socket 关闭套接字，当另一端 socket 关闭后，这一端读写的情况：尝试去读会得到一个 EOF，并返回 O。

#### 2.3.2 网络库

* 提供易用 API
    - 封装底层 Socket API
    - 连接管理和事件分发
    
* 功能
    - 协议支持：tcp、udp 和 uds 等
    - 优雅退出、异常处理等
    
* 性能
    - 应用层 buffer 减少 copy 
    - 高性能定时器、对象池等

## 3 关键指标

## 3.1 稳定性

* **保障策略**
    * 熔断：保护调用方，防止被调用的服务出现问题而影响到整个链路。一个服务 A 调用服务 B 时，服务 B 的业务逻辑又调用了服务 C，而这时服务 C 响应超时了，由于服务 B 依赖服务 C，C 超时直接导致 B 的业务逻辑一直等待，而这个时候服务 A 继续频繁地调用服务 B，服务 B 就可能会因为堆积大量的请求而导致服务宕机，由此就导致了服务雪崩的问题。
    * 限流：保护被调用方，防止大流量把服务压垮超时。当调用端发送请求过来时，服务端在执行业务逻辑之前先执行检查限流逻辑，如果发现访问量过大并且超出了限流条件，就让服务端直接降级处理或者返回给调用方一个限流异常。
    * 控制：避免浪费资源在不可用节点上。当下游的服务因为某种原因响应过慢，下游服务主动停掉一些不太重要的业务，释放出服务器资源，避免浪费资源。

* **请求成功率**
    * 负载均衡
    * 重试：防止重试风暴，限制单点重试和限制链路重试。

* **长尾请求**
    长尾请求一般是指明显高于均值的那部分占比较小的请求。业界关于延迟有一个常用的 P99 标准，P99 单个请求响应耗时从小到大排列，顺序处于 99% 位置的值即为 P99 值，那后面这 1% 就可以认为是长尾请求。在较复杂的系统中，长尾延时总是会存在。造成的原因非常多，常见的有网络抖动，GC，系统调度。

## 3.2 易用性

## 3.3 扩展性

## 3.4 观测性

## 3.5 高性能

* **高吞吐**
* **低延迟**