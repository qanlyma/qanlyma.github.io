---
title: 「论」 对于 EOV 区块链的优化
category_bar: true
date: 2023-10-30 13:20:09
tags:
categories: 论文阅读
banner_img:
---

FabircMan 的论文。

<!-- more -->

## Abstract

伴随着区块链技术的成熟与进步，一种新型的 Execute-Order-Validate（EOV）区块链架构被提出，使得交易在执行阶段可以并行。这一架构的代表系统便是 Hyperledger Fabric。但是 Fabric 的并行执行，会在验证阶段产生大量的 MVCC 冲突，导致大量交易被无效化。针对这一问题，我们基于 Fabric v2.4，在排序阶段设计了一个重排序算法来提高交易成功率，并且实现了一个基于缓存的版本验证机制，来提前检测并中止无效交易。同时针对简单的资产转移交易，我们实现了对交易的合并，使其不会触发多次版本检验。最后我们实现了验证阶段的并行验证，以提升系统整体的吞吐量。我们将优化后的系统命名为 FabricMan，并与 Fabric 和 Fabric++ 两个系统进行了实验对比。

## 1 INTRODUCTION

区块链技术的出现使得在缺少可信第三方条件下的点对点交易变为可能，因此大量的研究人员投身其中，希望这一技术能够对金融，游戏，医疗，物联网等等行业产生重大影响。但是目前实现这一美好愿景的最大阻碍就是区块链系统吞吐量过低的问题。例如最早出现的比特币[1]的 TPS 只有 7 左右，而能够执行智能合约的以太坊[2]的 TPS 也只有 15。这显然远远低于各种中心化系统，因此其无法满足实际应用的要求。

与比特币和以太坊等不需要验证节点身份的无许可链形成对比的是许可链，这种区块链系统需要每个参与的节点是已知的，并经过了身份验证。由于无许可链的不可信特性，系统需要花费大量资源去解决拜占庭容错共识问题[3]，而许可链对参与节点进行约束，大大减轻了共识的负担。并且许可链往往只应用于某些特定的场景，所以这类系统的性能总是更好。例如其代表之一的 Hyperledger Fabric[4] 的 TPS 在特定配置下可以达到 3500。

与传统区块链系统的先排序后执行架构不同，Fabric 使用的是新型的 Execute-Order-Validate（EOV）模型。在执行阶段，客户端会把交易提案发送给多个节点进行背书，收集到足够数量的背书后，客户端会将他们打包成一笔交易发送给排序节点进行出块，然后再由排序节点将区块发送给所有节点来同步账本状态。这种执行方式可以允许多笔交易的并行执行，但随之而来的问题是在最后验证阶段可能会出现的 MVCC 版本冲突。我们将区块内的冲突分为两类，一是区块内读写冲突 (within-block conflicts)，即在同一个区块内先执行交易的写集更改了后面某一笔交易读集的版本号，使得后一笔交易在验证阶段无效。二是跨区快读写冲突 (cross-block conflicts)，即某一笔交易在执行阶段读的值，在到达验证阶段之前，被这期间提交的其他区块修改所导致的最终无效化。Ankur 等人提出的 Fabric++[5] 通过对交易重排序来改善区块内读写冲突的问题，使得最终被无效的交易数量减少。但是经我们测试，该算法在交易冲突率较高时效率很低。本文的贡献主要如下：

* 我们设计并实现了一个时间复杂度较稳定的重排序算法来减少区块内的读写冲突，并设计实验与 Fabric++ 的算法相比较，发现在交易冲突率较高的情况下我们的算法更优秀。
* 我们实现了一个基于缓存的版本验证机制，在排序阶段提前检测并中止了无效交易，来减少跨区块的读写冲突。
* 基于重排序时生成的交易冲突图，我们在验证阶段对无关交易进行并行验证，以此发挥出多核 cpu 的优势。
* 在链码参数层面分析交易，我们对简单的资产转移交易进行合并，使得多笔交易在验证阶段只用验证一次读写集，最大化该类型交易的验证通过率。

本文第 2 章会简要介绍 Hyperledger Fabric 和 Fabric++ 的结构以及其他的相关研究，第 3 章会对 Fabric 中存在的问题进行理论分析并提出解决思路，第 4 章会介绍 FabricMan 的设计和优化点，第 5 章对 FabricMan 的优化进行实验测试，并分别与 Fabric 和 Fabric++ 对比，第 6 章对我们的工作进行总结。

## 2 BACKGROUND AND RELATED WORK

### 2.1 EOV Architecture in Hyperledger Fabric

Hyperledger Fabric 是最先进的许可区块链之一，其特点是基于 EOV 架构的模块化设计。Fabric 中的所有节点在任何时候都是已知并经过授权的，主要分为三种类型：(1) client 节点负责提交交易提案（transaction proposals），并收集背书响应；(2) peer 节点负责 executes 和 validates 交易提案，以维护本地的账本，不同的 peer 节点被划分到不同的 organizations，同一组织内的 peer 相互可信；(3) orderer 负责对接收到的交易进行排序，并按照预设的规则出块。区块中的交易顺序是由所有 orderer 节点基于共识协议共同决定的。The workflow of a transaction consists of three phases: execution, ordering, and validation.

#### 2.1.1 Execution Phase

在执行阶段，客户端会按照提前设定好的背书政策（endorsement policy）将交易提案发送给 peer 节点的一个子集，called the endorsers。由于参与交易的多个组织之间是互不信任的，通常背书政策需要每个组织至少有一个 endorser 参与背书。endorsers 会按照本地维护的当前账本状态并行的模拟执行交易，并生成相应的读写集。这里的读集是一个（key，ver）二元组，写集是一个（key，val）二元组，这里的 key is a unique name representing the entry，ver 和 val 分别是实体最新的版本号（由最近更新此实体的区块高度和交易号表示）和值。执行完成后，endorsers 会将读写集与自己的签名返回给 client，当 client 收集到足够数量的来自不同 endorsers 的相同读写集时，便可以将它们打包成一笔交易发送给排序集合，来进入下一个阶段。

#### 2.1.2 Ordering Phase

在排序阶段，不同的 orderers 会不断从不同的 clients 接受交易。ordering service 需要达成两个目标：(a) 对交易顺序达成共识，(b) 将排好序的交易按照规则打包成区块并 deliver 给所有的 peers。Fabric v2.4 在 (a) 中使用的是 Raft[6] 协议，for achieving crash-fault-tolerant consensus。而 (b) 中的出块规则一般由最大出块间隔和区块包含最大交易数量共同形成。注意，系统并不保证所有节点会同时收到区块，但是可以确定所有节点收到的区块中的交易顺序是相同的。

#### 2.1.3 Validation Phase

当节点收到来自 orderers 的一个区块时，首先检测是否有 orderers 的签名以及区块结构的合法性，若通过检测则将区块加入一个验证队列，保证其可以加入区块链，然后依次进入 VSCC 和 MVCC 验证阶段。在 VSCC 阶段，节点会检查区块中每一笔交易的签名是否满足该链码特定的背书策略。若不满足，则将该交易标记为无效但依旧留在块中。在 MVCC 阶段，会按顺序对所有标记为有效的交易进行 Multi-Version Concurrency Control[7] check。如果交易读集中某个 key 的版本号与当前世界状态中的版本号不一致，交易将会被标记无效。最后节点将区块写入本地账本，并根据每一笔交易的有效性修改当前世界状态。

### 2.2 Optimization of Fabric++

The vanilla Fabric 是按照交易抵达排序节点的先后顺序来确定区块内的交易顺序的，这样虽然可以快速进行排序，但同时可能导致潜在的不必要的序列化冲突。如 Table 1 所示，四笔交易按照 T1，T2，T3，T4 的顺序排列出块，T1 首先将 k1 的版本从 v1 更新至 v2。由于 T2、T3、T4 都需要读取 v1 版本的 k1，它们在验证阶段都会因为读取过时的版本被标记为无效，最终区块只会有一笔有效交易。被无效的交易若想被重新执行，则又需要通过新一轮的 simulation, ordering, and validation，这无疑大大降低了系统的效率。

![Table 1](Table1.png)

Ankur 等人对上述问题提出了一个名为 Fabric++ 的优化系统。他们发现之前例子中的四笔交易是存在一个无冲突排序的，即 T4，T2，T3，T1 如 Table 2 所示。导致区块产生冲突的原因是 Fabric 按照交易到达顺序进行排序。所以他们在 Fabric 的基础上增加了一个重排序算法，根据交易间读写集的关系来提前中止少量交易，并为剩余交易构建一个无冲突排序，以此来增加区块内的交易成功率。

![Table 2](Table2.png)

Fabric++ 的重排序算法主要分为五个步骤：(1)首先根据所有待排序交易的读写集构建冲突图。(2)然后应用 Tarjan[8] 的算法来识别所有强连通子图，并使用 Johnson[9] 的算法识别强连通子图中的所有环。(3)识别每一笔交易所在的环，并分别统计出现在多少个环中。(4)依次中止出现在最多环中的交易，直到冲突图无环。(5)最后利用剩余的交易建立一个可序列化的调度方案。

### 2.3 Related Work

对于以 Fabric 为代表的 EOV 区块链的优化，研究主要可以分为两类：(1)提升系统整体的吞吐量[10]，[11]。(2)减少并行执行所带来的读写冲突[5]，[12]，[13]。

Parth Thakkar[10] 等人通过配置 Block Size、Endorsement Policy、Channel、Resource Allocation、Ledger Database 等五个参数，对 Fabric v1.0 系统性能进行了较为全面的测试，并发现了其三个主要的性能瓶颈：(1)背书策略的验证。(2)块中交易的顺序策略验证。(3) CouchDB 的状态验证和提交。同时他们对上述问题进行了简单的优化：(1)使用以序列化形式为键的哈希映射来缓存反序列化的身份，以此减少加密操作的资源消耗。(2)并行验证多个交易的背书，以利用闲置的 CPU 并提高整体性能。(3)对于 CouchDB 的批量读写优化。达到了提升系统整体吞吐量的效果。

Christian Gorenflo[11] 等人重新构建了 Hyperledger Fabric v1.2。通过(1)在排序时仅传入交易 ID 而非整个交易。(2)在 committers 中积极缓存未组装的块，同时并行化尽可能多的验证步骤。(3)重新设计数据管理层，使用内存数据库取代原有数据存储。(4)分离负责背书和提交的节点角色。以此减少交易排序和验证期间的计算和 I/O 开销，将交易吞吐量从每秒 3000 笔提高到 20000 笔。

Pingcheng Ruan[12] 等人研究了 Fabric++[5] 的方案，认为其没有考虑到跨区快交易之间的依赖性，重排序效果受限。他们提出了一种更细粒度并发控制策略下的重排序算法，并验证了其安全性，使得重排序效果得到提升。

Qiucheng Sun[13] 等人分析了 Fabric++ 实现的重排序算法，在信任方面发现问题。并基于贪心算法提出了一个可信的重排序算法。

## 3 PROBLEM ANALYSIS

本章第一节主要介绍 Fabric 中存在的两种 MVCC 冲突，namely within-block conflicts and cross-block conflicts。并分析产生这两种冲突的原因。第二节则分析 Fabric++ 中重排序算法带来的的问题。

### 3.1 MVCC Conflict in Fabric

正如在第 2 章所提到的，Fabric 在执行阶段会生成交易的读写集。其中读集包含该交易读取的所有 key 的列表，以及相应的 version number。写集包含最终用于更新账本的键值对。在验证阶段，节点会根据本地数据库的当前状态对交易的读写集做 MVCC 验证。验证方法就是检测读集中的版本号是否与本地当前状态的版本号一致。若版本不一致，交易会被标记为无效，其写集无法被用于更新账本状态。值得注意的是，这些无效交易依旧会和有效交易一起被记录在链上。在高并发执行环境中，我们将产生 MVCC 冲突的原因分为两种：within-block conflicts and cross-block conflicts。

Within-block conflicts 是发生在同一个区块不同交易之间的读写冲突。在构建区块时，如果将多个对同一个 key 进行读写的交易划分到同一区块中，就可能导致这一冲突。例如在之前的例子 Table 1 中，T1、T2、T3、T4 按顺序被打包到一个区块中。在验证阶段，T1 先对 k1 进行了更新操作，使其版本号变为 v2。接下来轮到验证 T2，它的读集中包含 v1 版本的 k1，在进行 MVCC 验证时发现 v1 ≠ v2，所以 T2 被标记为无效。同理 T3、T4 也会无效。Within-block conflicts 的根本原因是同一区块中两笔不同的交易对某个 key 进行了先写后读的操作，导致后一笔交易在验证读集时发生版本冲突。

Cross-block conflicts 是发生在不同区块之间的读写冲突。由于 Execute-Order-Validate 的结构特性，交易在 simulated execution 和 verifcation submission 之间有一定的时延。从而导致后一个区块中发生脏读，最终引起 MVCC 冲突。如 Figure 2 所示，T1 和 T2 是处在不同区块中的两笔交易。在执行交易时，T1 读取 k1 的当前版本号 v0 并加入其读集。从验证阶段可以看出 T1 的写集对 k1 进行了修改，但由于这一步是模拟执行，执行结果并不会更改数据库状态，所以紧随其后的 T2 执行阶段读取到的依旧是 v0 版本的 k1。之后包含 T1 的区块先进入验证阶段，并更新了节点当前状态中 k1 的版本号到 v1。当验证包含 T2 的区块时，节点就会检测出 T2 的读集中存在 k1 的版本冲突，从而导致交易无效。这一旧读问题会一直等到最后的验证提交阶段才能被发现。

![Figure 2](fig2.png)

综合以上分析，我们决定在排序阶段引入一个重排序算法，针对同一个 key，先执行读集包含该 key 的交易，再执行写集包含该 key 的交易。以此减少 Within-block conflicts 中止的交易。同时在 orderer 中加入一个缓存机制，来储存每个 key 最新的版本号，如果遇到低于缓存版本的交易读集，则直接将其提前中止。以此提前解决 Cross-block conflicts 带来的问题。详细实现方案会在第 4 章介绍。

### 3.2 the Problem of Fabric++

Fabric++ 使用了重排序来解决 Within-block conflicts。在排序阶段，Fabric++ 会记录 orderer 收到交易的读写集，并用它们生成交易冲突图。若想得到无冲突排序，则需要冲突图中无环。如 2.2 节所述，Fabric++ 使用 Tarjan[8] 的算法将冲突图划分为多个强连通子图，然后使用 Johnson[9] 的算法从中检测所有的环。通过不断删除在环中出现频率最高的交易，这个算法可以得到一个包含尽可能多交易的无环图。最后通过无环图构建一个无冲突排序用以出块。

但是，在上述步骤中 Johnson 算法的时间复杂度是 O((n+e)(c+1))，其中 n 是节点数量，e 是边的数量，c 是图中环的数量。冲突图中节点和边的数量受图大小的限制，可以控制在较小的值。但是环的数量却没有限制，可能非常大。在第实验中我们发现，在相同配置条件下使用 Fabric++ 重排序算法，不同区块重排序的时间差异较大。当排序阶段交易冲突率增加时，重排序所需时间增加显著，甚至会无法完成正常出块。以 SmallBank[14] 智能合约为例，攻击者可以在少量账户之间高频率发送交易，从而使得重排序时间过长最终导致系统崩溃。文章[15] 也指出了类似的问题：Fabric++ 在解决环时，每删除一个交易都需要重新计算循环中单个事务的出现次数，这使得整个算法的时间复杂度来到 O(n^3)。

为了让算法执行时间更加稳定，我们在第 4 章基于拓扑排序提出了一种新的重排序算法，使得在交易冲突率较高的环境下系统依旧可以较快的完成重排序。

## 4 DESIGN OF FABRICMAN

我们已经分析过 Fabric 和 Fabric++ 存在的问题及原因，接下来介绍我们是如何解决这些问题的。我们将设计的系统命名为 FabricMan。

### 4.1 Transaction Reordering

在对一个交易集合 S 进行重排序时，首先需要识别交易间的依赖关系，来构建交易冲突图。在冲突图中，如果一笔交易 Ti 指向另一笔交易 Tj，则表示 Ti 的写集与 Tj 的读集有交集，记为 Ti → Tj。这表示在排序出块时，Tj 需要在 Ti 之前，否则 Tj 会因为读取过时的版本而被无效。在构建冲突图时，每一个节点代表一笔交易，由上述定义可知一个节点的出度表示该交易会影响多少个交易的有效性，入度表示受到多少其他交易的影响。此外，构建无冲突排序需要解决的一个重要问题就是图中出现的环。有环的图是无法进行序列化排序的，这需要我们在算法中移除一些交易来使原本的冲突图变为无环图。

我们的算法是基于拓扑排序的，但是由于拓扑排序无法适用于有环的图，所以我们做了一些改进。From a high-level perspective，算法主要分为五个步骤：(1)首先根据交易集合 S 中每一笔交易的读写集构建冲突图，并记录每个节点的入度和出度。(2)选择一个入度最小的节点作为待处理节点，若有多个节点满足条件，则选择一个出度最大的节点，若仍有多个则任选其中一个节点。(3)将指向待处理节点的其他节点从图中移除，直到待处理节点的入度为 0。(4)将待处理节点加入排序队列并从图中移除。(5)重复上述(2)(3)(4)步直到冲突图中无剩余节点。最后将排序队列进行反序，便可以得到一个无冲突的序列化排序。The pseudo-code of Algorithm 1 implements these five steps。注意在步骤 (2) 中，由于需要将待处理节点的入度变为 0，所以我们优先选择入度最小的节点，以留下更多的交易。又因为节点的出度是其在排序后会影响的交易数量，而最终排序是排序队列的反序，即在算法中先进入排序队列的交易会在较后的位置。所以我们优先选择出度最大的交易进行排序。

```pseudo-code
Algorithm 1 Reorder Sorting

func ReorderSort(Transaction[] S) {
    // Step 1: 
    Graph cg = buildConflictGraph(S)
    Graph incg = invert cg
    map[Transaction]int indegree = Calculate in-degrees using cg
    map[Transaction]int outdegree = Calculate out-degrees using cg

    // Step 2: 
    while S is not empty:
        for each Transaction tx in S:
            if indegree[tx] < minIndegree:
                minIndegree = indegree[tx]
                nodeToSort = node
            else if indegree[tx] == minIndegree and outdegree[tx] > outdegree[nodeToSort]:
                nodeToSort = node

        // Step 3: 
        for each nodeToRemove in incg[nodeToSort]:
            if nodeToRemove not in S:
                continue
            remove nodeToRemove from S
            for each tx in cg[nodeToRemove]:
                indegree[tx]--        
            for each tx in incg[nodeToRemove]:
                outdegree[tx]--

        // Step 4: 
        append nodeToSort to result
        for each tx in cg[nodeToSort]:
            indegree[tx]--
        remove nodeToSort from S

    // Step5: 
    return result.invert()
}
```

为了理解算法中的实现细节，我们举一个例子。假设交易集合 S 中有 T0 到 T5 六笔交易，通过 Step(1) 根据交易读写集构建出的冲突图如 Figure 3 所示，每个节点的入度和出度如 Table 3 所示。由 Step(2)，我们选择 T5 作为 nodeToSort，因为它此时是集合 S 中入度最小的节点。由于它的入度为 0 所以跳过 Step(3)。Step(4) 会将 T5 加入 result 队列，并将其从集合 S 中删除，然后将它指向的 T2 节点的入度减一。

![Figure 3](fig3.png)

![Table 3](Table3.png)

进入下一个循环，此时我们发现 T0、T2、T4 的入度都是 1，均为此时入度最小的节点，但是 T4 有最大的出度。根据 Step(2) 我们选择 T4 作为 nodeToSort。然后 Step(3) 从 S 中删除指向 T4 的节点 T2，使得 T4 的入度为 0，此时的冲突图如 Figure 4 所示，点的入度和出度如 Table 4 所示。Step(4) 将 T4 加入 result 队列，并将其从集合 S 中删除，然后将 T1 和 T3 的入度分别减一。

![Figure 4](fig4.png)

![Table 4](Table4.png)

此时图中只剩下一个由 T0、T1、T3 组成的环，由于其入度出度都一样，我们随机选择 T0 作为 nodeToSort，移除 T1，将 T0 加入 result 队列，最后将 T3 加入 result 队列。将 result 队列反序，我们便得到了最终的重排序结果，即 T3、T0、T4、T5。此排序过程主体部分与拓扑排序时间复杂度相当，为 O(n+e)，并且该算法并不受到图中环的数量影响，消除了 Fabric++ 算法中的这一缺陷。值得一提的是，我们的算法并不保证中止图中最少数量的交易使图无环，因为这是一个 NP 难的问题。我们只是提供了一个非常轻量级的方式来中止少量交易，从而生成一个可序列化排序方案。

### 4.2 Caching Mechanism

为了缓解 3.1 节中提到的 Cross-block conflicts 读写冲突问题，我们在 orderer 中使用哈希表实现了一个缓存。哈希表的 key 是智能合约链码中某实体的唯一名称，值是其最新的版本号以及一个是否被更新的标记。该缓存在交易到达 orderer 时，会检查每一笔交易的读集。若读集中的某个 key 小于缓存中的版本号，则直接将交易中止，这可以防止由于网络中某些陈旧的交易占用系统资源。若读集中的某个 key 大于缓存中的版本号，或不存在于缓存，则更新缓存储存最新的版本号。若读集中的某个 key 等于缓存中的版本号，则看其是否有 updated 标记，若有则中止交易。在进行完重排序后，我们得到了一个包含无冲突交易排序的区块，系统正常运行下，这个区块的写集都会被应用于账本的更新。此时 orderer 会检查每一笔交易的读集，若缓存中包含与读集相同的 key，则在缓存对应的值中加上 updated 标记。这表示该 key 已被更新，该版本已不是此 key 的最新版本。

如 Figure 5 所示，T1 和 T2 是处在不同区块中的两笔交易。在执行交易时，T1 读取 k1 的当前版本号 v0 并加入其读集，此时节点的本地状态并未发生变化，T2 也读取了 k1 和其版本号 v0。在排序阶段，当读取到 T1 读集中的 k1 时，orderer 将其与缓存中的值进行比较并发现缓存中不存在 k1，于是将 k1:v0 加入缓存。经过重排序后，orderer 再次检查交易序列，发现 k1 出现在序列交易的读集中，于是将缓存更新为 k1:v0-updated，表示当前节点的帐本中 k1 的版本大于 v0。随后包含交易 T2 的区块到来，orderer 检测到其读集中 k1 的版本为 v0，而缓存中 k1 的版本为 v0-updated，于是中止 T2。

![Figure 5](fig5.png)

使用此缓存机制存在一个问题，假如上述示例中 T1 由于某些意外原因（如签名问题）未能通过最终的验证阶段。此时节点世界状态中 k1 的版本号依旧为 v0，但是 orderer 中 k1 的版本号已经被更新为 v0-updated，这会导致后续读取 k1 的交易都无法通过。针对这一问题，我们在 orderer 的缓存中加入计时器，如果某个 key 在一个出块间隔时间内未被更新，则将其移出缓存。这同时也保证了缓存所占空间不会无限制增加。

### 4.3 Parallel Verification

文章[10]指出 Fabric 的性能瓶颈之一是验证阶段对区块内所有交易的串行 MVCC 验证。由于系统在交易重排序时根据区块内交易的读写集构建了冲突图，这可以很好的体现出交易之间的依赖关系，并区分无关交易。我们使用 Algorithm 1 中构建的冲突图以及深度优先遍历算法，将图划分为尽可能多的不连通子图。处于不同子图的交易读写集之间没有依赖关系，可以并行验证。划分好的子图跟随区块从 orderer 传递给验证节点，系统根据其对区块中的交易进行并行验证，以此充分利用多核 cpu 的优势。

### 4.4 Transaction Merging

最后来介绍我们实现的简单交易合并。由于区块链系统大多涉及加密货币和其他数字资产的转移，假设在进行简单的转账交易时，有两笔交易 T1（A 给 B 转账 10 个数字货币）和 T2（B 给 C 转账 10 个数字货币），并且被打包到 Fabric 的同一区块中。由于 T1 和 T2 都对 B 进行了读写，所以在 Fabric 中只有一笔交易可以通过验证。我们设计的系统通过在背书阶段对交易链码的参数和读写集进行分析，来识别这种简单的资产转移交易，并且在生成交易读集时不仅需要 key 和版本号，也需要将对应的 value 加入其中。通过合并它们读写集的方式，将同一链码下的多笔资产转移交易在排序阶段合并为一笔交易。

假设在此例中，A、B、C 的初始值都为 10，初始版本都是 v0。orderer 读取 T1 的读写集，由于 T1 已经在前一阶段被识别为转账交易，orderer 首先记录读集中 A 和 B 的初始值，然后根据写集 A 和 B 值的变化推测出转账金额，将 A 的值减 10，B 的值加 10，若此时余额不足则将交易中止。同理操作 T2，只不过在记录初始值时忽略已经存在的 B。最终得到 A:0、B:10、C:20，将其作为写集，并将 v0 版本的 A、B、C 作为读集生成合并交易。由于验证阶段只对合并交易的读写集验证一次，因此 T1 和 T2 对账本状态的修改都可以生效。

## 5 EXPERIMENTAL EVALUATION

### 5.1 Setup and Workload

• Hyperledger Fabric 2.4.0
• Fabric++
• FabricMan

节点：1 channel, 4 peers, 1 orderer
工具： caliper-benchmarks


### 5.2 The Impact of Blocksize

测试区块大小对 Fabric 的影响：10,000 个账户随机触发 100,000 笔读写交易
Block Size: 16, 32, 64, 128, 256, 512, 1024

![Figure 6](fig6.png)

• 此实验 FabricMan 仅使用重排算法；
• 重排算法对系统整体性能影响很小；
• 同一区块内交易数量越多，冲突概率越大，重排作用越明显。

后续实验区块大小：256
最大出块时间间隔：5 s
交易发送速率：2048 / s

### 5.3 The Comparison of Reordering Algorithm

比较 Fabric++ 和 FabricMan 算法重排序时间和有效交易数量：
发生读写交易（读四个账户写其中两个）的账户数量：1500, 1400, 1300, 1200, 1100, 1000

![Figure 7](fig7.png)

• Fabric++ 在同一轮次实验中，不同区块排序时间差别巨大，而 FabricMan 重排算法较稳定；
• Fabric++ 在账户数量为 1000 的时候已经无法正常出块，而 FabricMan 不受影响。

![Figure 8](fig8.png)

### 5.4 The Effect of Parallel Verification

分配不同数量 vCPUs 验证阶段时间：
使用一个准备好的区块测试，vCPUs: 1, 2, 4, 8, 16

![Figure 9](fig9.png)

• 增加使用的核心数后， MVCC 验证时间有明显减少；
• 核心数到达 8 以上时间减少不明显，可能是因为与数据库交互成为瓶颈。

### 5.5 The Effect of Transaction Merging

500 个账户发送随机读写交易（读两个账户写其中一个）和转账交易（读写两个）：
转账交易的占比：15%, 25%, 35%, 45%, 55%, 65%, 75%, 85%

![Figure 10](fig10.png)

• 转账交易占比较低时，FabricMan 因为使用重排序，成功交易数量略高于 Fabric；
• 转账交易占比较高时，交易合并作用明显；
• 由于读写交易比转账交易少写一个账户，所以随着转账交易占比增加，在 Fabric 中 MVCC 冲突数量增加。

### 5.6 Combinations of Optimizations


## 6 CONCLUSION








## 参考文献

[3] M. Vukolic, “The quest for scalable blockchain fabric: Proof-of-work vs. bft replication.” in International Workshop on Open Problems inNetwork Security. Springer, 2015, pp. 112–125.

[5] Ankur Sharma, Felix Martin Schuhknecht, Divya Agrawal, and Jens Dittrich. 2019. Blurring the Lines between Blockchains and Database Systems: the Case of Hyperledger Fabric. In Proceedings of the 2019 International Conference on Management of Data. ACM, 105–122.

[6] D. Ongaro and J. Ousterhout. In search of an understandable consensus algorithm. In USENIX Annual Technical Conference (ATC), pages 305–320, 2014

[7] C. H. Papadimitriou and P. C. Kanellakis. On concurrency control by multiple versions. ACM Trans. Database Syst., 9(1):89–99, Mar. 1984.

[8] Robert Endre Tarjan. 1972. Depth-First Search and Linear Graph Algorithms. SIAM J. Comput. 1, 2 (1972), 146–160. https://doi.org/10.1137/0201010

[9] Donald B. Johnson. 1975. Finding All the Elementary Circuits of a Directed Graph. SIAM J. Comput. 4, 1 (1975), 77–84. https://doi.org/10.1137/0204007

[10] Thakkar, Parth, Senthil Nathan, and Balaji Viswanathan. “Performance benchmarking and optimizing hyperledger fabric blockchain platform.” 2018 IEEE 26th International Symposium on Modeling, Analysis, and Simulation of Computer and Telecommunication Systems (MASCOTS). IEEE, 2018.

[11] Gorenflo, Christian, et al. “FastFabric: Scaling hyperledger fabric to 20000 transactions per second.” International Journal of Network Management 30.5 (2020): e2099.

[12] A Transactional Perspective on Execute-order-validate Blockchains

[13] A Trusted Solution to Hyperledger Fabric Reordering Problem

[14] 2019. http://hstore.cs.brown.edu/documentation/deployment/benchmarks/smallbank/

[15] FabricETP: A high‑throughput blockchain optimization solution for resolving concurrent conflicting transactions