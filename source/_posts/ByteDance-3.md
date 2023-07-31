---
title: 「训」 笔记(3)：内存管理
category_bar: true
date: 2023-03-06 12:13:09
tags:
categories: 字节青训
banner_img:
---

本文是关于自动内存管理和 Go 内存管理及优化的笔记。

<!-- more -->

## 1 自动内存管理

### 1.1 概念

**自动内存管理**（垃圾回收，GC）：由程序语言的运行时系统管理动态内存。避免手动内存管理，专注于实现业务逻辑，并保证内存使用的正确性和安全性: double-free problem, use-after-free problem

**三个任务**：

* 为新对象分配空间
* 找到存活对象
* 回收死亡对象的内存空间

**Mutator**: 业务线程，分配新对象，修改对象指向关系
**Collector**: GC 线程，找到存活对象，回收死亡对象的内存空间
**Serial GC**: 只有一个 collector
**Parallel GC**: 并行 GC，支持多个 collectors 同时回收的 GC 算法
**Concurrent GC**: 并发 GC，支持 mutator(s) 和 collector(s) 同时执行的 GC 算法

![GC](1.png)

### 1.2 追踪垃圾回收

* 被回收的条件：指针指向关系不可达对象

* 过程：

    ![扫描过程](2.png)

    1. 标记根对象 (GC roots)：静态变量、全局变量、常量、线程栈等

    2. 标记：找到所有可达对象

    3. 清理：回收所有不可达对象占据的内存空间
        * **Copying GC**: 将存活对象从一块内存空间复制到**另外一块内存空间**，原先的空间可以直接进行对象分配
        ![Copying GC](3.png)
        * **Mark-sweep GC**: 将死亡对象所在内存块标记为可分配，使用 free list 管理可分配的空间
        ![Mark-sweep GC](4.png)
        * **Mark-compact GC**: 将存活对象复制到同一块内存区域的开头
        ![Mark-compact GC](5.png)

### 1.3 分代 GC

分代假说：most objects die young，很多对象在分配出来后很快就不再使用了

* 年轻代（Young generation）
    * 常规的对象分配
    * 由于存活对象少，可以采用 copying collection
    * GC 吞吐率很高

* 老年代（Old generation）
    * 对象趋于一直活着，反复复制开销大
    * 可以采用 mark-sweep collection

### 1.4 引用计数

每个对象都有一个与之关联的引用数目

对象存活的条件：当且仅当引用数大于 0

* 优点：
    * 内存管理的操作被平摊到程序运行中：指针传递的过程中进行引用计数的增减
    * 不需要了解 runtime 的细节：因为不需要标记 GC roots，因此不需要知道哪里是全局变量、线程栈等

* 缺点：
    * 开销大，因为对象可能会被多线程访问，对引用计数的修改需要**原子操作**保证原子性和可见性
    * 无法回收环形数据结构
    * 每个对象都引入额外存储空间存储引用计数
    * 虽然引用计数的操作被平摊到程序运行过程中，但是回收大的数据结构依然可能引发暂停

## 2 Go 内存管理及优化

### 2.1 分块

为对象在 heap 上分配内存，提前将**内存分块**

* 调用系统调用 mmap() 向 OS 申请一大块内存，例如 4 MB
* 先将内存划分成大块，例如 8 KB，称作 mspan
* 再将大块继续划分成**特定大小**的小块，例如 8 B、16 B、24 B，用于对象分配
* noscan mspan: 分配不包含指针的对象 —— GC 不需要扫描
* scan mspan: 分配包含指针的对象 —— GC 需要扫描

### 2.2 缓存

Go 内存管理构成了多级缓存机制，从 OS 分配得的内存被内存管理回收后，也不会立刻归还给 OS，而是在 Go runtime 内部先缓存起来，从而避免频繁向 OS 申请内存。

![多级缓存](6.png)

* 每个 p 包含一个 mcache 用于快速分配，用于为绑定于 p 上的 g 分配对象
* mcache 管理一组 mspan
* 当 mchache 中的 mspan 分配完毕，向 mcentral 申请带有未分配块的 mspan
* 当 mspan 中没有分配的对象，mspan 会缓存在 mcentral 中，而不是立即释放并归还给 OS

### 2.3 GMP

* G：goroutine
* M：machine（机器线程/内核线程）
* P：processor（处理器）

![](9.png)

* M 负责管理 goroutine 的执行，它与操作系统线程（OS Thread）一一对应。Go 运行时会根据需要创建 M，并在多个 M 之间调度 goroutine
* P 负责执行 goroutine，它维护了一个 goroutine 的队列，当 M 空闲时，会从队列中获取 goroutine 来执行
* 每个 P 和一个 M 绑定，M 是真正的执行 P 中 goroutine 的实体

模型优点：

* 复⽤线程：避免频繁的创建销毁线程
* 多核并⾏

### 2.4 优化方案

**Go 内存管理的问题**：对象分配是非常高频的操作，每秒分配 GB 级别的内存；Go 的内存分配流程很长，占用很多 CPU

**字节跳动的优化方案**：Balanced GC

核心：将 noscan 对象在 per-g allocation buffer (GAB) 上分配，并使用移动对象 GC 管理这部分内存，提高对象分配和回收效率。

![GAB](7.png)

```go
if top + size <= end {
    addr := top
    top += size
    return addr
}
```

* 每个 g 会绑定一个较大的 allocation buffer (例如 1 KB) 用来分配小于 128 B 的 noscan 小对象
* 分配对象时，根据对象大小移动 top 指针并返回，快速完成一次对象分配
* 同原先调用 mallocgc() 进行对象分配的方式相比，balanced GC 缩短了对象分配的路径，减少了对象分配执行的指令数目，降低 CPU 使用

从 Go runtime 内存管理模块的角度看，一个 allocation buffer 其实是一个大对象。本质上 Balanced GC 是将多次小对象的分配合并成一次大对象的分配。因此，当 GAB 中哪怕只有一个小对象存活时，Go runtime 也会认为整个大对象（即 GAB）存活。为此，balanced GC 会根据 GC 策略，将 GAB 中存活的对象移动到另外的 GAB 中，从而压缩并清理 GAB 的内存空间，原先的 GAB 空间由于不再有存活对象，可以全部释放。

![用 copying GC 管理小对象](8.png)

Balanced GC 只负责 noscan 对象的分配和移动，对象的标记和回收依然依赖 Go GC 本身，并和 Go GC 保持兼容。