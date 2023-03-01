---
title: "「论」 Hyperledger Fabric: A Distributed Operating System for Permissioned Blockchains"
category_bar: true
date: 2023-02-20 22:20:27
tags:
categories: 论文阅读
banner_img:
---

摘要：Fabric 是一个模块化、可扩展的开源系统，用于部署和操作**许可区块链**，也是 Linux 基金会托管的超级账本项目之一。它支持**模块化共识协议**，允许系统根据特定用例和信任模型进行定制。在某些流行的部署配置中， Fabric 实现了每秒 3500 TPS，具有 sub-second 的延迟，可扩展到 100 多个对等节点。

<!-- more -->

## 1 INTRODUCTION

区块链相较于传统的 SMR（state-machine replication，状态机复制）存在拜占庭式缺陷：
1. 不只一个，而是许多分布式应用程序同时运行
2. 应用程序可以由任何人动态部署
3. 应用程序代码是不可信的，甚至可能是恶意的

许多现有区块链实施的是 **order-execute** 结构，它的局限性在于要求所有对等方执行每个交易，并且所有交易都是确定的。

* 共识在平台内是硬编码的，这与没有一刀切的（BFT）共识协议的公认理解相矛盾
* 交易验证的信任模型由共识协议决定，不能适应智能合约的要求
* 智能合约必须以固定的、非标准的或特定领域的语言编写，这会阻碍广泛的应用，并可能导致编程错误
* 所有对等方对所有交易的顺序执行限制了性能，需要采取复杂的措施防止来自不可信合约的拒绝服务攻击
* 交易必须是确定性的，这在编程上很难保证
* 每个智能合约都在所有对等方上运行，这与保密性不符，并禁止向一部分对等方传播合约代码和状态

Fabric 可以克服这些局限性。

Fabric 使用 **execute-order-validate** 结构，将交易分为三个步骤，可以在系统中不同的实体中运行：

1. **executing** 交易并检查其正确性，从而 **endorsing** 交易（对应于其他区块链中的“交易验证”）
2. 通过共识协议进行 **ordering**，而不考虑交易语义
3. 根据特定于应用程序的信任假设进行 **validation**，这防止了并发导致的竞争

该设计结合了两种复制方案：**passive**（当一个事务提交时，领导者副本将该事务的数据变更广播给所有跟随者副本，跟随者按照提交顺序应用这些变更） 和 **active**（每个副本独立执行相同的确定性交易代码）

* 首先 Fabric 使用 passive or primary-backup replication， 每个事务仅由一个对等方的子集执行（认可），这允许**并行执行**。灵活的背书政策规定了哪些对等方或其中多少对等方需要为正确执行给定智能合约提供担保。
* 其次 Fabric 结合了 active replication，即每个对等方单独执行的确定性验证步骤中，交易对账本状态的影响只有在总顺序达成共识后才能写入。状态更新的排序被委托给一个模块化组件以实现共识（即 atomic broadcast），该组件是无状态的，并且在逻辑上与执行事务和维护账本的对等方分离。由于共识是模块化的，因此它的实现可以根据特定部署的信任假设进行调整。

Fabric 中包含了以下组件：
* **ordering service** 以原子方式向对等方广播状态更新，并就事务顺序建立共识。
* **membership service provider** 负责将对等方与加密身份相关联。它保持了 Fabric 的许可性。
* 可选的 **peer-to-peer gossip service** 通过 ordering service 向所有对等节点传播块的输出。
* Fabric 中的 **smart contracts** 在容器环境中运行以隔离。它们可以用标准编程语言编写，但不能直接访问帐本状态。
* 每个对等方以 append-only blockchain 的形式在本地维护 **ledger**，并将其作为键值存储中最新状态的快照。

## 2 BACKGROUND

### 2.1 Order-Execute Architecture for Blockchains

![Order-execute architecture in replicated services.](1.png)

### 2.2 Limitations of Order-Execute

* Sequential execution：成为性能瓶颈
* Non-deterministic code：导致分叉
* Confidentiality of execution：保密开销巨大

### 2.3 Further Limitations of Existing Architectures

* Fixed trust model：应用程序级别的信任不应固定为协议级别的信任
* Hard-coded consensus：应使用不同共识协议适应不同的环境

### 2.4 Experience with Order-Execute Blockchain

区块链系统的关键属性，即一致性、安全性和性能，不得依赖于其用户的知识和善意，因为区块链在不受信任的环境中运行。

## 3 ARCHITECTURE

### 3.1 Fabric Overview

Execute-order-validate blockchain architecture：
![Execute-order-validate architecture of Fabric](2.png)

Fabric 的分布式应用程序由两部分组成：

* **chaincode**：一个智能合约，在执行阶段运行，可以使用通用编程语言（Golang、Java等）
* **endorsement policy**：一个背书策略，在验证阶段被评估，只有特定的管理员可以修改

客户端向背书策略指定的节点发送交易，然后由节点执行每个交易，并记录其输出，这一步骤也称为 endorsement。执行完成后，交易进入排序阶段，通过可插拔的共识协议对这些交易排序（并非交易输入，而是对交易输出排序，并结合状态依赖），然后向所有节点广播。然后，每个节点根据背书策略验证背书交易的状态变化，并验证交易的一致性。在此期间 Fabric 使用了主动和被动复制的混合策略。

三个角色：

![MSP: membership service provider](3.png)

* **Clients**：提交、广播交易
* **Peers**：执行（并不执行所有交易，只有被指定为 endorsers 时执行）、验证交易，维护区块链
* **Orderers** (OSN，Ordering Service Nodes)：给所有交易排序

交易流：

![Fabric high level transaction flow](4.png)

### 3.2 Execution Phase

在执行阶段，客户端签署交易提案（proposal）并发送给背书策略中指定的背书人。背书人根据本地区块链状态模拟执行提案，此时的执行结果并不会与其他节点同步，也不会写入账本中。链码创建的状态仅限于该链码，不能由其他链码直接访问。给定适当的许可，链码可以调用另一个链码来访问同一通道内的状态。

每一个背书人在模拟执行的时候会生成一个 writeset（模拟执行产生的状态更新，即修改的键和值）和一个 readset（表示提案模拟执行的版本依赖，即模拟过程中所有的键和它的版本号）。模拟结束后，背书人对一条名为背书（endorsement）的消息进行加密签名，该消息包含 writeset 和 readset（以及交易 ID、背书人 ID 和背书人签名等元数据），并在提案响应中将其发送回客户端。客户端收集这些 endorsement 直到满足背书策略的要求，同时所有背书人要产生相同的执行结果（即，相同的 writeset 和 readset）。然后客户端会将交易发送给排序服务。

这种在排序之前的执行可以容忍非确定性链码，执行结果的不一致会导致它无法收集足够数量的背书。同时背书人若怀疑收到 DoS 攻击，可以根据本地协议单方面终止执行交易。

### 3.3 Ordering Phase

当客户端收集到足够的背书时，它组装一个交易并将其提交给排序服务，该交易包含链码操作和参数、交易元数据和一组背书。排序阶段为每个通道（channel，不同的区块链系统可以连接至同一个排序服务，每一个区块链系统都叫做一个通道）提交的交易分别进行总排序。此外，排序服务将多个交易打包成块，并输出块的哈希链序列。

排序服务的节点不维护区块链的状态，也不验证或执行交易。这种架构是 Fabric 的一个关键的特征，使 Fabric 成为第一个将共识与执行和验证完全分离的区块链系统。同时请注意，我们不需要排序服务来防止交易重复，因为在验证阶段节点在读写检查中过滤了重复的交易。

### 3.4 Validation Phase

区块直接由排序服务或者 gossip 发送给节点，然后进入验证阶段：

1. 对区块内的所有交易并行进行背书策略评估。评估是所谓的验证系统链码（VSCC）的任务，如果背书不满足，则交易被标记为无效，其影响将被忽略。
2. 对区块内的所有交易按序进行读写冲突检查。对于每一笔交易，它都会将 readset 字段中的键与节点本地存储的账本当前状态的键进行比较，并确保它们相同。如果版本不匹配，则转换标记为无效，并忽略其影响。
3. 最后对账本进行更新，其中区块被附加到本地存储的账本，区块链状态被更新。特别是，当将块添加到账本时，前两个步骤中的有效性检查结果也将以位掩码的形式持久化，该位掩码表示块内有效的交易。这有助于以后恢复状态。此外，所有状态更新都是将写集中的键值对写入本地状态。

Fabric 帐本中也包括无效的交易，只是因为排序服务在形成区块时与链码状态无关。在某些需要在后续审计期间跟踪无效交易的用例中需要此功能，与以太坊（只包含有效交易）等其他区块链形成对比。

### 3.5 Trust and Fault Model

所有客户端都是潜在的恶意或者拜占庭节点；节点被分为不同的组织，组织内节点相互信任，但不信任组织外的某一节点；排序服务认为所有客户端和节点都可能是拜占庭的。

Fabric 将应用程序的信任模型与共识的信任模型解耦。分布式应用程序可以定义自己的信任假设，这些假设通过背书策略传递，并且独立于由排序服务实现的共识假设。

## 4 FABRIC COMPONENTS

Fabric 是用 Go 语言编写的，并采用 gRPC 框架在节点之间传输信息。

![Fabric 节点的组成](5.png)

### 4.1 Membership Service

The membership service provider (MSP) 维护着系统内所有节点（clients，peers，OSNs）的身份信息，并负责颁发用来认证和授权的凭据。因为 Fabric 是许可链，节点之间的所有交互都是通过经过身份验证的消息进行的，通常会使用数字签名。用于节点密钥管理和注册的工具也是 MSP 的一部分。

### 4.2 Ordering Service

排序服务管理着多个通道，对于每个通道它提供以下服务：

1. 用于交易排序的原子广播，实现 broadcast 和 deliver 调用
2. 在成员广播配置更新交易时，修改通道的配置
3. 访问控制（可选），限制向指定客户端和节点广播交易和接收区块

OSN 直接将新接收的交易注入原子广播（例如，Kafka代理）。OSNs 对原子广播接收的交易批处理并形成块。一旦满足以下三个条件之一，即成块：

1. 该块包含指定的最大交易数
2. 块已达到最大大小（字节）
3. 从接收到新块的第一个交易起已经过去了一段时间

**该批处理过程是确定性的，因此在所有节点上产生相同的块。**（这是由原子广播的特性决定的）

### 4.3 Peer Gossip

将执行、排序和验证阶段分离的一个优点是它们可以独立扩展。由于大多数共识算法（在 CFT 和 BFT 模型中）都有带宽限制，排序服务的吞吐量会受到其节点的网络容量的限制。研究表明共识性能不能通过增加节点数量来增大，这样做吞吐量反而会降低。然而，由于排序和验证是分离的，因此在排序阶段之后，可以高效地将执行结果广播给所有节点进行验证；以及向新加入的节点和长期断开连接的节点的状态同步，正是 gossip 部分的目标。

Gossip 的通信层基于 gRPC，并利用 TLS 相互认证，其组件维护系统中最新的在线成员视图。

Fabric gossip 使用两个阶段进行信息传播：

* push：每个节点从成员视图中随机选取一组活跃的邻居节点，并向它们转发消息
* pull：每个节点周期性地访问一组随机选择的节点并请求丢失的消息。

为了减少从排序节点向网络发送块的负载，该协议还选择一个领导节点从排序服务中 pull 区块并发起 gossip。

### 4.4 Ledger

Ledger 由一个 block store 和一个 peer transaction manager 组成：

* Ledger block store：由一组仅添加的文件组成，同时维护一些索引，用于随机访问块或块中的交易。
* Peer transaction manager（PTM）：在版本化的键值储存中维护最新状态（key，val，ver）

对于交易验证，PTM 顺序验证块中的所有交易。这将检查一个交易是否与之前的任何交易冲突（当前块或更早的块中）。对于 readset 中的任何键，如果 readset 中记录的版本与最新状态下的版本不同（假设所有之前的有效交易都已提交），则 PTM 将该交易标记为无效。PTM 会计算并保存一个 savepoint，表示成功应用的最大块数，用于崩溃中恢复时从持久化块中恢复索引和最新状态。

### 4.5 Chaincode Execution

链码的执行环境与节点低耦合，它支持用于添加新的链码编程语言的插件，目前支持 Go、Java 和 Node。

每个用户级或应用程序链码都在 Docker 容器的一个单独进程中运行，这将链码彼此隔离，并与节点隔离。而系统链码直接在节点进程中运行，可以实现 Fabric 所需的特定功能，并且可以在用户链码之间的隔离过于严格的情况下使用。

### 4.6 Configuration and System Chaincodes

Fabric 可以通过 channel configuration 和 system chaincodes 来定制结构。每个通道形成一个逻辑区块链，通道的配置保存在特殊 configuration blocks（genesis block）的元数据中。通道配置包括：

1. 确定参与节点的 MSPs
2. OSNs 的网络地址
3. 实现共识和排序服务的共享配置（例如 batch size 和 timeouts）
4. 制定访问排序服务的操作（broadcast 和 deliver）
5. 制定修改通道配置的规则

## 5 EVALUATION

此节介绍一些初步的性能数据和影响因素。引入 Fabric Coin（Fabcoin，UTXO cryptocurrencies）用于性能测试和比较。

## 6 APPLICATIONS AND USE CASES

略

## 7 RELATED WORK

略

## 8 CONCLUSION

未来的改进工作：

1. 探索基准测试和优化实现性能
2. 可扩展到大型部署
3. 一致性保证和更通用的数据模型
4. 通过不同的共识协议保证其他弹性
5. 通过加密技术对交易和账本数据进行隐私和保密