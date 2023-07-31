---
title: 「研」 以太坊区块结构和数据存储
category_bar: true
date: 2023-03-03 10:25:26
tags:
categories: 理论研究
banner_img:
---

此篇文章用以太坊为例，研究一下一个区块里面到底有什么，以及以太坊中的数据是怎么存储的。

<!-- more -->

## 区块结构

一个区块由两部分组成：

* 区块头（Header）
* 区块体（Body）

![Block](1.png)

轻节点只会存储 Header。

区块中的三棵树：

* **状态树**：状态树是全局的、不断更新的，它的 key 为 keccak256(ethereumAddress)，value 为 rlp(ethereumAccount)。其中 ethereumAddress 表示以太坊账户地址；ethereumAccount 表示以太坊账户，包含四个字段：nonce，balance，storageRoot，codeHash。如果 storageRoot 和 codeHash 不为空，则为合约账户，其中 codeHash 对应合约代码的哈希，storageRoot 对应另一棵树的树根，这棵树我们称为**存储树**。存储树存储了合约的所有状态数据，每个合约有单独的存储树。

* **交易树**：每个区块都有一棵独立的交易树，对应区块头里的交易根。交易树的 key（路径）为 rlp(transactionIndex)，value 为交易序列化后的值。其中 transactionIndex 表示交易在该区块中的下标。

* **回执树**：每个交易对应一个交易回执，交易回执记录了交易执行结果，包括执行状态、Gas 消耗、事件日志等。每个区块有自己的回执树，对应了区块头里的回执根。与交易树类似，key 为 rlp(transactionIndex)，value 为交易回执序列化后的值。

![Trie](2.png)

## 数据结构

以太坊中的账户状态是用什么数据结构存储的呢？

有关这个问题[肖臻老师的公开课](https://www.bilibili.com/video/BV1Vt411X7JF?p=16)有非常详细的解释。这里直接说结论：

**Merkle Patricia Trie（MPT）**

![World State Trie](3.png)

右上角为简化的 key-value 定义。我们可以看到图中有 2 个拓展节点，2 个分支节点，4 个叶子节点。在最下面的两个叶子节点中，prefix 3 的右边有个格子，有个箭头从 7 指向这个格子，表示 3 和 7 两个 nibble 组成一个字节存储。当一个节点被另一个节点在内部指向时（比如上图中的分支节点内部指向了叶子节点），父节点会存储 H(rlp.encode(x))。
其中 H(y) = keccak256(y) if len(y) >= 32 else y，rlp.encode 为 RLP 编码函数。

由于 MPT 树是确定性的，所以如果两棵树存储了完全相同的数据，那么这两棵树的节点将完全相同，包括根节点。假设在某一时刻，当其中一个合约修改了某个变量的数据，使得它与另一个合约的数据不同时，会生成一个新的节点，并从新节点开始由下往上直到根节点，整个路径的节点值都会更新，新生成的节点会存储到硬盘，但旧的节点不会从硬盘删除。

![update data](4.png)

## StateDB

在以太坊中，StateDB 是一个用于管理账户状态数据的内存数据库，它的数据存储在节点的内存中，并且随着区块链的不断增长而不断变化。但是，为了确保数据的可靠性和持久性，StateDB 数据需要被定期持久化到磁盘中。

为了实现数据的持久化，以太坊节点使用了 LevelDB 这样的键值对存储引擎。当 StateDB 中的数据发生变化时，节点会将这些变化记录在 LevelDB 中，以便在下次启动时从磁盘中重新加载数据。

需要注意的是，由于 LevelDB 是一种磁盘存储引擎，相比于内存数据库，它的读写速度要慢很多。因此，在以太坊节点中，StateDB 通常被用作内存数据库，而 LevelDB 则用于实现数据的持久化。这种方式可以同时满足节点的高性能和数据的可靠性要求。

## block.go 源码

研究那么多不如来看一看区块结构的源码：go-ethereum/core/types/block.go

```go
// Header represents a block header in the Ethereum blockchain.
type Header struct {
	ParentHash  common.Hash    // 父区块的哈希
    UncleHash   common.Hash    // 所有叔块的哈希（Block.uncles）
    Coinbase    common.Address // 接受出块奖励的地址，矿工出块时在这个字段填入自己的地址
    Root        common.Hash    // 此区块包含的所有交易完成后，state 对象的哈希值
    TxHash      common.Hash    // 此区块包含的所有交易的哈希（Block.transactions）
    ReceiptHash common.Hash    // 此区块所有交易完成后所产生的所有收据的哈希
    Bloom       Bloom          // 用来快速判断一个参数 Log 对象是否存在于一组已知的 Log 集合中
    Difficulty  *big.Int       // 此区块的难度值
    Number      *big.Int       // 此区块的高度
    GasLimit    uint64         // 此区块所有交易消耗的 gas 值的上限
    GasUsed     uint64         // 此区块所有交易消耗的 gas 的实际值
    Time        *big.Int       // 区块时间戳
    Extra       []byte         // 区块的额外数据
    MixDigest   common.Hash    // 以太坊 hashimoto 算法产生的哈希
    Nonce       BlockNonce     // 通过此字段产生符合 Difficulty 值要求的区块哈希
}

type Body struct {
	Transactions []*Transaction
	Uncles       []*Header
}

// Block represents an entire block in the Ethereum blockchain.
type Block struct {
	header       *Header
	uncles       []*Header
	transactions Transactions

	// caches
	hash atomic.Value
	size atomic.Value

	// 从创世块开始，到当前区块截止，累积的所有区块 Difficulty 之和
	td *big.Int

	// These fields are used by package eth to track
	// inter-peer block relay.
	ReceivedAt   time.Time
	ReceivedFrom interface{}
}
```

可见一个区块携带的数据主要是：**Header 中的信息 + 叔块们的头部信息 + 块中包含的所有交易信息**。