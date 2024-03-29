---
title: 「研」 区块链扩容调研
category_bar: true
date: 2022-12-07 23:06:54
tags:
categories: 理论研究
banner_img:
---

研一时写的一篇关于区块链 Layer 1 和 Layer 2 的扩容调研报告。

<!-- more -->

## 一、区块链扩容 

### 1.1 为什么需要扩容

在比特币诞生之初，创始人中本聪并没有特意限制区块的大小，区块最大可以达到 32 MB。有人认为区块链上限过高容易造成计算资源的浪费，还容易发生 DDOS 攻击（分布式拒绝服务攻击）。因此，为了保证比特币系统的安全和稳定，中本聪决定临时将区块大小限制在 1 MB。

随着越来越多的人关注和使用比特币，链上最高时有上万笔交易积压，很多用户为了尽快让自己的交易被打包，不得不增加手续费，有时候比特币转账交易费高达几十美元。网络拥堵时，一个比特币交易甚至需要花费好几天才能被打包，同理也使得以太坊上的 gas 费用居高不下。比特币和以太坊作为区块链 1.0 和 2.0 的代表，比特币每秒大约只能处理 7 个交易，性能只有 7 TPS；以太坊每秒只能处理 15 个交易，性能只有 15 TPS。而作为中心化的代表，淘宝在 17 年双十一超过了 200,000 TPS，可见区块链的 TPS 跟中心化应用还有很大的差距。

与很多分布式系统一样，区块链技术中也有一个“不可能三角形”，比特币和以太坊在最早诞生时最关注的是去中心化和安全，牺牲了可扩展性。后来，市场上不断有一些新的区块链项目牺牲掉去中心化或者安全性来为可扩展性保驾护航，试图部署出一个吞吐量较高的区块链网络。迄今为止，还没有人找到去中心化、可扩展性和安全性三全其美的策略，打造出一个充分运行、基于加密货币的大规模区块链网络。但是，如果想把区块链的应用拓展到虚拟货币投资之外，必须得有支持其吞吐量扩容的解决方案。

![不可能三角](1.png)

### 1.2 如何扩容

所谓的扩容方案是指“为了改善区块链交易速度使其达到规模化所提出的解决方案”，各层所提出的扩容方案，其最终目的都是为了解决区块链交易速度的问题。

![区块链分层](2.png)

目前主要可以分为链上（Layer 1）和链下（Layer 2）的扩容方案。链上扩容是指为了提高区块链的吞吐量而对其进行的任何直接修改，比如比特币修改共识机制来改变区块结构或直接增加区块大小，而以太坊的策略则是改变网络结构进行分片，不同的分片并行处理不同的交易，增加整体吞吐量。而链下扩容是指在不改变公链本身规则的前提下，将数据计算过程等信息移到链下进行，而主链仅记录结果，比如比特币的闪电网络，以太坊的侧链，状态通道和 Rollup 等。下文将对具体方案进行详细介绍。

![扩容方案](3.png)

## 二、Layer 1 扩容

### 2.1 扩块和隔离见证

想让一个区块打包更多的交易，最直观的有两种解决办法：一是增加区块大小；二是缩小交易数据的尺寸。这就分别产生了扩块和隔离见证两个扩容方案，当然这两个方案也可以一起使用，可是不论选择何种方案都避免不了区块链的分叉。

比特币社区中提及比较多的是 2 MB 区块，一部分人希望通过硬分叉直接把区块大小限制从 1 M 改到 2 M，提升单个区块内的交易数，每秒打包的交易就会增加，从而提升 TPS。可是扩块有一些弊端：
* (1) 大区块传播速度变慢，验证速度变慢，导致频繁的重组，双花攻击概率提高；
* (2) 大区块导致储存量大幅度增加，成本提高，节点可能减少，趋向中心化从而影响安全。

比特币社区有一部分人不接受硬分叉，坚持使用隔离见证（Segwit）方案去优化主链结构，并且结合闪电网络等二层网络结构来改善支付体验。隔离见证的原理是压缩每笔交易的大小，从而增加每个区块可以记录的交易数量。此方案将比特币交易数据分为交易信息和签名信息，对于普通用户来说他们只关心每个账户有多少资产，不需要验证信息，隔离见证就是把区块内的数字签名信息拿出去，让每个区块可以承载更多交易，从而达到扩容的目的。可是这样做的扩容能力有限，依然难以满足大量转账的需求。

两种方案的支持者方争论不休，随着加密货币市场迎来了前所未有的关注度，比特币价格连创新高，这也让比特币的交易更加拥堵，最终在 2017 年 8 月 1 日导致了比特币历史上第一次重大硬分叉的出现，比特币区块大小由 1 M 扩大到 8 M，同时也诞生出新币种 BCH（比特币现金）。

### 2.2 分片

分片技术（sharding）来自中心化数据库技术，将大型数据库数据进行切分，并分布在特定的服务器中，以提高数据库性能。如果将分片技术运用到区块链中，就相当于将区块链网络里的所有待处理任务进行分解，全网的节点也进行分组，每一组同时处理一个分解后的任务，这样就从原先单一节点处理全网的所有任务变成了多组节点同时处理，如此以来，自然就能大大提升这条链的处理效率。但是分片技术的问题也是显而易见的：分片多用于以太坊网络中，那么跨片区智能合约交易如何处理？片区如何划分？各片区如何同步？

![分片](4.png)

分片技术根据不同的分片机制可以划分为三种：
* 网络分片（network sharding）
* 交易分片（transaction sharding）
* 状态分片（state sharding）

#### 2.2.1 网络分片

网络分片是最基础的一种分片方式，就是将整个区块链网络划分成多个子网络（也就是一个分片），网络中的所有分片并行处理网络中不同的交易。

但是这个方案会使网络的安全性和去中心化性会下降，比如原来 A 想要在某交易中作恶，因为共识机制的原因，A 需要控制全网的大部分节点或算力才行，但现在因为分片把节点分散到一个个小的区域中，A 只用控制包含这个交易的小区域的大部分节点算力就行。

幸好分片技术另外一个非常重要的机制就是随机分配，在区块链领域建立随机性的方式主要是利用可验证随机函数（VRF, Verifiable Random Function）。利用随机性，网络可以随机抽取节点形成分片。这样一种随机抽样的方式可以防止恶意节点过度填充单个分片，这样想要作恶的人，就很难知道一个小区域中的节点都有谁，作恶成本会大幅提高，从而分片技术能在保证安全与去中心化的同时，解决效率与可扩展性问题。

#### 2.2.2 交易分片

网络分片是其他所有分片的基础，交易分片的前提是先进行网路分片。交易分片主要涉及的问题是哪些交易应该按照特定的属性被分配到哪些分片当中。

* 基于 UTXO 的账本系统。在基于 UTXO 的账本系统中，一笔交易可能由多个输入和多个输出构成，我们没有办法按照地址进行交易分片来有效地避免双花问题。比较直观的交易分片方式是按照交易的 hash 值最后几位进行分片。但是这样也有可能导致双花交易，所以不同分片之间不得不进行通信。
* 基于账户系统。在基于账户系统中，一笔交易只有一个输入，而输入的地址将被记录在账户系统中。该账户系统在交易分片的每个分片中是全局可见的，因此我们只需要将交易按照发送者的地址进行分片，即可保证同一个账户发出的多笔交易将被在同一个分片当中被处理，这样该分片可以有效的检测双花交易而不需要复杂的跨分片的通信。

#### 2.2.3 状态分片

状态分片的关键是将整个存储区分开，让不同的分片存储不同的部分，每个节点只负责托管自己的分片数据，而不是存储完整的区块链状态。状态分片可以减少状态的冗余存储，使得整个区块链网络具有存储的可扩展性。

在账户型系统中，状态分片是按照账户的地址进行分片的，并且一个特定的分片只会保留一部分状态，而不像是交易分片那样每个节点都保存整个网络中的所有状态。这导致可能需要进行频繁的跨分片通信和状态交换。跨分片通信可能又会降低状态分片的性能。

在状态分片的情况下，重新分配节点是非常困难的。由于每个分片只保留了网络状态的一部分，所以在一次重新调整网络的过程中，必须要防止调整过大而导致在同步完成前可能会出现的整个系统失效的问题。为了防止系统的中断，必须对网络进行逐步调整，以确保每个分片在所有节点被清空前仍有足够多的旧节点。而新节点在加入分片之前，需要等待同步完该分片中的状态信息之后才可以正式加入分片并提供算力。

### 2.3 Casper

比特币和以太坊当前都采用工作量证明（PoW）共识机制，这是个低效的系统，因为它消耗会大量的电力和能量。而且这个机制可以通过购买更快更强的 ASIC 设备比其他人拥有更高的概率挖到区块，这导致比特币并没有像它希望的那样分散化。如果采取其他共识机制（PoS），改变区块形成的规则，提高系统效率即可增加每秒处理交易。Csaper 就是以太坊选择实行的 PoS 协议，值得一提的是，Casper 并非专为扩容而设计，但它会对以太坊网络容量产生积极影响。

![PoS](5.png)

在介绍 Casper 之前我们首先要了解无利害关系问题（Nothing at stake），这一问题是由于 PoS 机制不会消耗节点的算力，所以在共识系统出现分叉情况时，出块节点可以在“不受任何损失”的前提下，同时为多条链出块，从而有可能获得“所有收益”。Casper 是一种基于保证金的经济激励共识协议。协议中的节点作为“锁定保证金的验证人”，必须先缴纳保证金才可以参与出块和共识形成。Casper 共识协议通过对这些保证金的直接控制来约束验证人的行为。具体来说就是，如果一个验证人作出了任何 Casper 认为“无效”的事情，他的保证金将被罚没，出块和参与共识的权利也会被取消，从而解决了无利害关系问题。

从扩容的角度来说，修改后，可以从根本上改变区块形成的规则，往有利于交易量增加的方向修改。但是无论如何改变，共识算法仍然是分布式算法，需要多个节点达成一致，处理的上限是一台机器处理能力的上限。并且共识机制是加密货币的核心，涉及加密货币的整体逻辑，需要全盘考虑对旧区块的承接、安全性、后续发展等问题，还需要做大量的技术尝试。

## 三、Layer 2 扩容

第二层扩容也称链下扩容，是不改变公链基础协议的一种应用层上的扩展方案，即不改动区块链本身的规则（区块大小，共识机制等）。由 Layer 2 协议，区块链事务的“状态生成”可以独立于 Layer 1 之外进行。换句话说 Layer 2 扩容方案是尽可能在不牺牲区块链网络安全性的情况下实现高吞吐量的状态生成。

### 3.1 侧链协议

侧链是与公链并排运行并与之通信的独立区块链，它使用另一个代币与公链代币相互锚定，从而创建了双向桥。侧链是完全独立的，具有自己的共识机制和安全性保证。通过这种解决方案，可以实现数字资产从第一个区块链到第二个区块链的转移，又可以在稍后的时间点从第二个区块链安全返回到第一个区块链。其中第一个区块链通常被称为主区块链或者主链，第二个区块链则被称为侧链。

![侧链协议](7.png)

由于没有第一层设计的负担，侧链可以支持超出其基础层能力的某些特性，包括但不限于可扩展性和互操作性，同时不依赖于第一层的存储，可以建设多条侧链提供非常高的 TPS。而且得益于其独立性，如果侧链上出现了代码漏洞和大量资金被盗等问题，主链的安全性和稳定性都不会受到影响。缺点是它是一个无信任的环境，用户需要将资金托管转移到侧链，从侧链取回资产时的安全性问题需要被考虑，侧链目前不那么成熟，去中心化也更差。

### 3.2 状态通道

状态通道是固定一组参与者（通常是两名参与者）之间的协议，用以实现安全的链下交易，其中支付通道专门用来支付。支付通道协议具体情况是两名参与者各自通过链上交易在链上锁定保证金，一旦锁定完成，参与者双方即可互相发送形式为轮次、金额、签名的状态更新来实现转账，无需与主链进行交互，只要双方的余额都还为正值即可。一旦参与者中有一方想要停止使用支付通道，可以执行退出操作：将最后的状态更新提交至主链，结算下来的余额会退给发起支付通道的两方。主链可以通过核实签名和最后结余来验证状态更新的有效性，从而防止参与者使用无效状态来退出支付通道。

该方案会用到一个协议叫 HTLC（Hashed Timelock Contract），其实就是限时转账。A 给 B 转账，A 先冻结一笔钱，并提供一个哈希值，如果在一定时间内 B 能提出一个字符串匹配，则这笔钱转给 B。过了一定时间没有
提交这个字符串的话，A 就可以拿回这笔钱。

![HTLC](12.png)

状态通道带来的优点是交互延迟在毫秒级别，是唯一能够逼近当今互联网用户体验的区块链扩容技术；交易手续费极低，从根本上比所有其他 Layer 2 技术的交易手续费低；水平扩展性强，加节点就能增加总系统容量，TPS 无上限，且互相之间不隔离，不需要有跨分片或者跨链之类的复杂操作。但它的退出模式存在一个问题，即主链无法验证支付通道是否提交了全部交易，也就是说，在提交了状态更新之后是否不再出现新的状态更新。而且它的使用场景较为局限：长期合作关系的双方的支付，偶发性交易难以适用，并且通道不能用于将资金在链外发送给尚未参与的人。此外，状态通道只能在两个参与者之间开设。闪电网络就是比特币使用状态通道的例子。

![闪电网络](6.png)

状态通道相较于侧链协议有更强的隐私性，并且有即时的最终确定性。但是状态通道需要所有参与者 100% 的在线，在侧链中，你就不需要一直在线。

### 3.3 Plasma

Plasma 由 Vitalik 和 Joseph Poon 在 2017 年共同提出，Plasma 是一种链下交易的技术，从一个新的方向实现了状态通道。它允许创建附加在以太坊主链上的子链，这些子链反过来可以产生他们自己的子链。其结果就是，我们可以在子链级别执行许多复杂的操作，运行拥有数千名用户的整个应用程序，并且只需与以太坊主链进行尽可能少的交互。子链可以更快地操作，且交易费用更低，因为它的操作不需要在整个以太坊区块链存留副本。

区别于状态通道，Plasma 中能够运行智能合约，如果说状态通道是对交易吞吐量的扩容，那么 Plasma 是对计算能力的扩容。Plasma 是将计算和数据存储都迁移到 Layer 2 进行，由 Layer 2 的执行者周期性地向主链递交 Merkle 根形式的状态承诺。如果执行者递交无效的状态，用户可以向主链上的智能合约提供错误性证明（fraud proof），一旦确认执行者出现欺诈行为，智能合约会没收他的保证金。

![Plasma](8.png)

虽然说我们可以通过错误性证明，使得提供无效承诺的执行者在主链上遭到惩罚，但 Plasma 的数据并没有提交到链上，如果 Plasma 的执行者拒绝在主链上公开数据，那么用户则无法提供错误性证明，所以 Plasma 面临的最大问题是交易数据可用性。针对这个问题，Plasma 衍生出一些相应的方案，如延长资产从 Layer 2 退出的时间：当出现作恶行为，就能允许资产从 Plasma 链转移回主链。所以在 Plasma 上退出一笔资产的周期会长达一周左右，如果在争议期间没有人提交欺诈证明，那么资产才可以安全退出到主链。相比较而言，普通的侧链就没有这个安全特性。

### 3.4 Rollup

之前介绍的几种链下扩容方案虽然诞生时间很早，但是发展的却比较缓慢，其背后的原因归根结底是数据的可用性（Data Availability）问题。无论是状态通道还是 Plasma 侧链，完整的交易记录和见证数据都只保存在链下，出现争端时如果参与者没有及时提供正确的交易和见证数据，交易的安全性就无法保证。这时一种名为 Rollup 的方案被提了出来。

![Rollup](9.png)

Rollup 可以被认为是一种压缩技术，多笔交易可以压缩在一起（几千笔交易可以被打包到一个 Rollup 区块中），既能减少交易数据规模，又能降低交易验证负担，因此使得以太坊区块链能处理更多交易。它将所有 Layer 2 上的交易数据快照发送到主链上某个智能合约内，用主链上的单个合约来保管所有的资金，通过在主链上为每一笔交易公开一些数据，让任何人都能通过观察区块链上的 calldata（交易输入数据）来获得 Layer 2 的所有数据。Rollup 区块的状态是由用户以及链下运营者来维护的，因此不会占用主链的存储空间。所有交易的收据都存储在以太坊区块链上，这就提升了 Layer 2 交易的安全性。具体的实现方案目前主要分为 ZK Rollup 和 Optimistic Rollup 两种。

#### 3.4.1 ZK Rollup

ZK Rollup 是靠着在主链完成零知识证明，链上无需包含签名数据，因为零知识证明就足以证明交易的有效与否，交易有效性就立刻确认，也即数据可用性放在链上，所以 ZK Rollup 对数据存储方面带来了一定程度上的扩展性提升。它的缺点是验证链路的构造没有一个通用的解决方案，所以目前没有很好的办法做到很广义的虚拟机逻辑，简单来说，ZK Rollup 必须对每一个用例定制，程序正确性的验证相对复杂，二层打包节点负担重，成本高，计算零知识证明所需时间长，用户延迟的体验角度仍然比较差，目前只适合简单的转账。

#### 3.4.2 Optimistic Rollup

ZK Rollup 包含一个 SNARK 零知识证明，合约用它来验证在老的用户状态上施加这批交易，但是生成 SNARK 的成本非常高，所以 Optimistic Rollup 采用了欺诈证明来验证交易有效性。Optimistic Rollup 的理念是由 John Adler 首先构想出来的，它保留了 calldata，主链可以获得所有 Layer 2 的数据，但那些刷新 Layer 2 状态的交易不会在链上被验证，只让主链存储一系列的历史状态根，添加了一个新的状态的一段时间后才将新状态最终敲定，也就是数据可用性放在链下。采用欺诈证明，对提交无效状态的执行者进行惩罚。其链下 OVM 虚拟机可以支持任意智能合约逻辑的实现，与以太坊 EVM 虚拟机搭配使用，开发者就可以用 Solidity 来写代码，实现 DApp 和智能合约之间的无缝互操作性。它的缺点是安全问题，只有使用一到两周的欺诈证明挑战期才足够安全。在挑战期过去以前，没有交易能被认为是确定的。

#### 3.4.3 比较

![比较](10.png)

* 1. Optimistic rollup 和 ZK rollup 的主要区别在于采用了不同的数据证明方式。Optimistic rollup 使用欺诈证明：子链上的交易结果并不直接生成相关证明接入主链，子链仅向主链报告结果。如果有人发现结果错误，他们可以向链上发布一个证明，证明处理计算错误。合约将验证证明，并对结果进行更正。ZK rollup 使用零知识证明: 子链将自身交易采用 ZK-SNARK 技术生成加密证明，证明状态根是执行子链上交易的正确结果。无论计算量有多大，证明都可以在链上快速验证。
* 2. Optimistic Rollup 基于加密经济学有效性博弈，只有过了挑战期才能确认交易生效。ZK Rollup 的延迟相对较小，如果一个打包区块中有 1000 笔交易，在普通的服务器上大概需要 20 分钟就可以构造出一个证明。
* 3. 通用性方面，Optimistic Rollup 明显好于 ZK Rollup，当然它的设计目标就是支持任意智能合约。而 ZK Rollup 目前仅适用于支付之类的特定交易，对于通用智能合约，由于创建零知识证明的成本非常高，部署起来困难较大。

短期看来 Optimistic Rollup 由于较好的通用性会受到开发者的青睐；但从长期来看，随着零知识证明虚拟机的演进，ZK Rollup 会在通用性上不断提高。

## 四、总结

状态通道有一些独特的性质，让它在扩容领域有着独特的地位，它的诸多属性在很多应用中都非常重要。比如游戏、IoT 设备网络、去中心的互联网服务提供商等。Plasma 和状态通道相比，Plasma 中能够运行智能合约，而状态通道则不被允许。分片系统要比 Plasma 链更不易于遭受拒绝服务攻击，分片链提供的防御也更易于普及。但 Plasma 链可以被迭代，新的设计可以更快地被实现，因为每条 Plasma 链都可以在无需与该生态系统中的其他链进行协调的情况下单独地进行部署，而且由单个运营商运行的 Plasma 链还可以提供比分片系统更多的隐私保护。而在分片系统中，所有的数据都是公开的。

相比于 Plasma 和 ZK Rollup，Optimistic Rollup 做了一些权衡，所以带来的扩展性提升幅度最小，但 Optimistic Rollup 不依赖于什么过于前沿的技术或悬而未决的问题，所以实际推广中 Optimistic Rollup 更好落地。而 ZK Rollup 可以解决 Optimistic Rollup 上的几个根本问题，消除了令人厌恶的尾部风险（通过复杂但可行的攻击方法从 Optimistic Rollup 中盗取资金），将提取资金的时间从几周缩减到几分钟，支持快速的交易确认和退出，并且默认保护隐私。对于需要提高流动性的项目而言，资本运作效率 ZK Rollup 高于 Optimistic Rollup。

不同的扩容技术有它不同的优缺点，导致适应不同的应用场景，未来不同的扩容技术之间也会是相互合作关系，某一场景下同时使用多种扩容技术。以太坊基金会在今年1月25日宣布淘汰“以太坊 2.0”的说法，改称为“共识层”，设计人员在其中加入当下的一些先进技术，如分片技术、Casper 协议等等。相信伴随着以太坊的全面升级，其 TPS 必将有很大改善，但其中技术上的一些问题还有待大家共同攻坚克难。

![Eth 2.0](11.png)

## 参考资料

[1] 登链社区. 突破不可能三角. <https://learnblockchain.cn/column/12>.
[2] Croman, K., et al. On Scaling Decentralized Blockchains (A Position Paper).
[3] Gangwal A, Gangavalli H R, Thirupathi A. A Survey of Layer-Two Blockchain Protocols.