---
title: 「毕」 5. Fabric 的改进
category_bar: true
date: 2023-06-07 09:59:14
tags:
categories: 毕设项目
banner_img:
---

本文总结论文阅读中对于 Fabric 的改进方案，并梳理一下毕设的改进思路。

<!-- more -->

## 1 Fabric

### 1.1 交易流程

![Fabric](1.png)

1. 客户将其交易发送给一定数量的背书节点。
2. 每个背书节点在隔离环境中模拟执行交易，并计算相应的读写集以及访问的每个键的版本号。模拟成功后返回背书响应给客户端。
3. 客户端收集足够数量的背书，然后将这些响应打包发送给排序节点。
4. 排序节点首先对传入交易的顺序达成共识，然后将消息队列划分为块。
5. 块被传递给对等节点，然后由对等节点验证并提交它们。 

### 1.2 阶段展示

![Phases](2.png)

### 1.3 改进思路

有实验使用 Fabric 网络发送大量的转账交易，结果如下：

![TPS](3.png)

可以看出大量的交易因为读写冲突被中止了，而发送空白交易的对比试验总的 TPS 与发送转账交易的 TPS 相差不大，这说明系统的瓶颈并不在于智能合约的计算上。

由此可以得出两种改进思路：

* 增加系统的整体吞吐量

* 将原本会被 Fabric 中止的交易变为成功的交易

## 2 增加吞吐量

### 2.1 Optimizing Fabric [1]

此文对 Fabric 1.0 做了大量测试，发现以下**瓶颈**：

1. 身份的加密操作
2. 区块中的交易串行验证
3. 对 CouchDB 的多个 REST API 调用

**优化**（单通道提升 16 倍）：

1. MSP Cache
2. 并行 VSCC 验证
3. 区块 MVCC 验证时的批量读写

### 2.2 FastFabric [2]

**优化**：

1. 排序阶段仅使用交易 ID，并行处理多个交易，提高共识效率
2. 使用内存中的哈希表代替 LevelDB/CouchDB 来储存世界状态，提高访问速度
3. 分离背书和验证的角色
4. 缓存解析的区块数据
5. 并行验证块和块中交易的有效性

**结果**：

![Orderer](4.png)

* Opt O-I: 只使用交易 ID 
* Opt O-II: 并行处理传入交易

![Peer](5.png)

* Opt P-I: 使用内存哈希表
* Opt P-II: 验证与提交分离
* Opt P-III: 缓存解析数据

## 3 提高成功率

### 3.1 Fabric++ [3]

**优化**：

1. 事务重新排序机制（transaction reordering mechanism）
2. 早期中止（early abort）

**重排算法**：

![Reordering](6.png)

1. 建立所有交易的冲突图
2. 识别冲突图中的所有环
3. 识别每一笔交易出现在哪个环中
4. 依次中止出现在最多环中的交易，直到图无环
5. 使用剩余的交易建立一个可串行化调度

**早期中止**：

![Early Abort](7.png)

**结果**：

![Result](8.png)

### 3.2 FabricSharp [4]

**优化**：
1. Fabric++ 过度中止了跨块读取的交易
2. Fabric++ 没有考虑跨快并行交易之间的依赖，重排效果受限
3. 更好的早期中止和交易重排算法

![](9.png)

六种基本依赖：

![](10.png)

* Fabric/Fabric++ 不允许在两个交易之间的 anti-rw，因为后一个交易将读取前一个更新的旧版本，因此必须中止。

* 如果两个交易 Txn1 和 Txn2 表现出 c-rw 或 anti-rw 依赖性，则改变顺序不会影响它们的依赖顺序。

* 如果两个交易 Txn1 和 Txn2 表现出 c-ww 依赖性，交换它们的顺序将翻转它们的依赖顺序。

![](11.png)

* 由上可知：如果环没有 c-ww 依赖关系，则无法重新排序为可序列化。

![](12.png)

* 所以对于一个新交易，所有待排序交易（包括新交易）中除了 c-ww 的所有依赖项中如果存在依赖环，直接中止新交易。

**Overview**

![](13.png)
![](14.png)
![](15.png)
![](16.png)

## 4 毕设想法

### 4.1 交易合并

这是针对简单的转账交易的改进。

假设有 Tx1(A --10-->B)，Tx2(A --10-->C)，在目前 Fabric 系统中如果将这两笔交易打包到一个区块中，即使 A 的余额充足（即大于 20），排在后面的一笔交易也会因为读写冲突被中止。同理 Tx3(B --10-->A)，Tx4(C --10-->A) 也是如此。

那么我希望可以将由同一个 key 转出的，或者转入到同一个 key 的交易进行合并。

### 4.2 拆分子键

假设有 Tx1(A --10-->B)，Tx2(B --10-->C)，这两笔交易无法合并，那么可以采用子键的形式：

![Sub-K](17.png)

从主键 B 中拆分两个子键 B1 和 B2：B1 用于接收 A 的转账，B2 用于给 C 转账。导师建议我在最后将 B 的子键合并。

### 4.3 交易重排

构建一个转账流图。利用之前的改进进行交易重排序。

### 4.4 并行验证

在重排时生成依赖，对不存在依赖的独立交易链并行验证。

## 5 参考文献

[1] Nathan S , Thakkar P , Vishwanathan B . Performance Benchmarking and Optimizing Hyperledger Fabric Blockchain Platform: IEEE, 10.1109/MASCOTS.2018.00034[P]. 2018.

[2] Gorenflo C ,  Lee S ,  Golab L , et al. FastFabric: Scaling Hyperledger Fabric to 20,000 Transactions per Second[J]. IEEE, 2019.

[3] Sharma A ,  Schuhknecht F M ,  Agrawal D , et al. Blurring the Lines between Blockchains and Database Systems: the Case of Hyperledger Fabric[C]// ACM SIGMOD 2019. ACM, 2019.

[4] Ruan P ,  Loghin D ,  Ta Q T , et al. A Transactional Perspective on Execute-order-validate Blockchains[J].  2020.
