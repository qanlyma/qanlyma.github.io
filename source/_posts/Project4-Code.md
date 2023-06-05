---
title: 「毕」 4. Fabric 2.4.0 源码解析
category_bar: true
date: 2023-05-25 15:26:30
tags:
categories: 毕设项目
banner_img:
---

学习一下 Fabric 2.4 的源代码，本文只关注较为**核心**的部分。

<!-- more -->

## 交互流程

![流程](1.png)

## 代码结构

* build：编译相关

* ci：持续集成（CI）相关配置和脚本 

* cmd：命令行操作相关入口代码

* **common**：一些公共库（错误处理、日志处理、账本存储、策略以及各种工具等等）

* **core**：核心库，组件的核心逻辑，针对每一个组件都有一个子目录（chaincode：与智能合约相关，comm：与网络通信相关，endorser：与背书节点相关）

* **discovery**：服务发现模块相关代码

* docs：文档相关

* **gossip**：组织内部节点数据同步的通信协议，最终一致性算法，用于组织内部数据同步

* **images**：Docker 镜像打包，Docker 镜像都是通过这个目录下的配置文件生成的

* integration：待迁移代码

* **internal**：内部代码，被 cmd 包等调用

* **msp**：成员服务管理，Fabric 网络中会为每一个成员提供相应的证书，msp 模块就是读取这些证书并做一些相应的处理

* **orderer**：排序节点的入口，用于消息的订阅与分发处理

* **pkg**：重写或实现了一些 golang 原生接口

* **protoutil**：Proposal 提案相关的工具包

* release_notes：发布笔记

* **sampleconfig**：配置示例

* **scripts**：脚本，包含启动脚本、检查脚本等

* swagger：swagger 文档生成配置

* **tools**：工具包，目前还是空的（release-2.4 分支）

* vagrant：包含了用 Vagrant 建立一个简单的开发环境所必需的脚本

* vender：存放 Go 中使用的第三方包

## 1 背书

### 1.1 ProcessProposal 函数

`core/endorser/endorser.go`

Client 与 Peer 交互的入口，用于处理提案并生成提案响应。

```go
func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error) {

	// 对提案进行拆包
	up, err := UnpackProposal(signedProp)

	// 进行预处理，检查提案的头部、唯一性和访问控制列表（ACL）
	err = e.preProcess(up, channel)

	// 处理提案并生成提案响应
	pResp, err := e.ProcessProposalSuccessfullyOrError(up)

}
```

```go
func (e *Endorser) ProcessProposalSuccessfullyOrError(up *UnpackedProposal) (*pb.ProposalResponse, error) {

	// 模拟执行
	res, simulationResult, ccevent, ccInterest, err := e.simulateProposal(txParams, up.ChaincodeName, up.Input)

	// 获取链码的背书策略插件
	escc := cdLedger.EndorsementPlugin

	// 使用背书策略插件进行背书
	endorsement, mPrpBytes, err := e.Support.EndorseWithPlugin(escc, up.ChannelID(), prpBytes, up.SignedProposal)

}
```

#### 1.1.1 simulateProposal 函数

执行提案，调用链码。

```go
func (e *Endorser) simulateProposal(txParams *ccprovider.TransactionParams, chaincodeName string, chaincodeInput *pb.ChaincodeInput) (*pb.Response, []byte, *pb.ChaincodeEvent, *pb.ChaincodeInterest, error) {

	// 执行提案，并获得结果
	res, ccevent, err := e.callChaincode(txParams, chaincodeInput, chaincodeName)

	// 调用接口获取事务模拟的结果（读写集）
	simResult, err := txParams.TXSimulator.GetTxSimulationResults()

}
```

```go
func (e *Endorser) callChaincode(txParams *ccprovider.TransactionParams, input *pb.ChaincodeInput, chaincodeName string) (*pb.Response, *pb.ChaincodeEvent, error) {

	// 调用链码执行的接口
	res, ccevent, err := e.Support.Execute(txParams, chaincodeName, input)

}
```

#### 1.1.2 EndorseWithPlugin 函数

`core/endorser/plugin_endorser.go`

给执行结果背书。

```go
// EndorseWithPlugin endorses the response with a plugin
func (pe *PluginEndorser) EndorseWithPlugin(pluginName, channelID string, prpBytes []byte, signedProposal *pb.SignedProposal) (*pb.Endorsement, []byte, error) {
	plugin, err := pe.getOrCreatePlugin(PluginName(pluginName), channelID)

	return plugin.Endorse(prpBytes, signedProposal)
}
```

## 2 排序

Orderer 在 Fabric 网络中的作用主要是原子广播（Atomic Broadcast）和全排序（Total Order）：

* 通过 `broadcast` 接口，接受 client 发送的交易，然后将这些 Tx 进行排序；排序的的原则为 FIFS，但是区块内交易的顺序不一定与实际顺序一样，而是由到达 Orderer 的时间来决定的。
* 排序后的交易根据一定的规则打包成区块，通过 `deliver` 接口将区块发送给 Peer 或 client；保证所有 Orderer 节点分发出来的 block 是一致的。

配置：

* 常量配置：`internal/peer/common/ordererenv.go`
* 变量配置：通过 configtx.yaml 保存在区块中，可以通过 Config 交易进行修改

### 2.1 Broadcast 函数

`orderer/common/server/server.go`

```go
// Broadcast receives a stream of messages from a client for ordering
func (s *server) Broadcast(srv ab.AtomicBroadcast_BroadcastServer) error {

	return s.bh.Handle(&broadcastMsgTracer{
		AtomicBroadcast_BroadcastServer: srv,
		msgTracer: msgTracer{
			debug:    s.debug,
			function: "Broadcast",
		},
	})
}
```

### 2.2 Handle 函数

`orderer/common/broadcast/broadcast.go`

Handle 函数负责原子广播服务端的处理，Handle 从 Broadcast 流里面读取请求，然后调用 ProcessMessage 函数进行处理，最后还会将处理结果返回给数据流。

客户端调用 Broadcast 发起服务请求时，会调用 s.bh.Handle 方法处理请求，调用 Handle 方法，通过服务器端 srv 调用 srv.Recv()，监听并接收 Send 接口发送的交易消息请求。

```go
// Handle reads requests from a Broadcast stream, processes them, and returns the responses to the stream
func (bh *Handler) Handle(srv ab.AtomicBroadcast_BroadcastServer) error {

	for {
		// 从流中接收一个消息，如果接收到的消息是 io.EOF，表示流已结束
		msg, err := srv.Recv()
		
		// 处理收到的消息
		resp := bh.ProcessMessage(msg, addr)

		// 将响应发送回流中
		err = srv.Send(resp)
	}

}
```

### 2.3 ProcessMessage 函数

排序节点接收的交易类型分为 Normal 交易和 Config 交易：

* Normal 交易的内容包含 ProposalResponse 以及其他内容
* Config 交易主要是 channel 的创建或配置修改

因为需要让后面的 Normal 交易尽快在最新配置的 channel 上运行，Config 交易都是单独成块的。

```go
func (bh *Handler) ProcessMessage(msg *cb.Envelope, addr string) (resp *ab.BroadcastResponse) {

	// 解析出消息的通道头部 chdr，配置交易消息标志位 isConfig、链支持对象 processor
	chdr, isConfig, processor, err := bh.SupportRegistrar.BroadcastChannelSupport(msg)

	if !isConfig {

		// 处理普通交易消息
		configSeq, err := processor.ProcessNormalMsg(msg)

		// 检查当前通道共识组件链是否已经准备好接收新消息
		err = processor.WaitReady()

		// 重新配置普通交易消息，交给共识请求排序出块
		err = processor.Order(msg, configSeq)

	} else {
		// 处理配置交易消息
	}

}
```

#### 2.3.1 ProcessNormalMsg 函数

`orderer/common/msgprocessor/standardchannel.go`

```go
func (s *StandardChannel) ProcessNormalMsg(env *cb.Envelope) (configSeq uint64, err error) {

	// 获取通道配置序号
	configSeq = s.support.Sequence()

	// 利用自带的通道信息过滤器过滤该消息，检查是否满足应用通道上的消息处理请求
	err = s.filters.Apply(env)
	
}
```

#### 2.3.2 Order 函数

`orderer/consensus/etcdraft/chain.go`

提交 Normal 交易。

```go
// Order submits normal type transactions for ordering.
func (c *Chain) Order(env *common.Envelope, configSeq uint64) error {

	return c.Submit(&orderer.SubmitRequest{LastValidationSeq: configSeq, Payload: env, Channel: c.channelID}, 0)
}
```

#### 2.3.3 Submit 函数

Submit 首先将请求消息封装为 submit 结构，通过通道 c.submitC 传递给后端处理，同时获取当前时刻 raft 集群的 leader 信息。

若当前节点是 leader 则将待提交的请求发送到本地 goroutine 处理；如果不是 leader，需要将请求发送给 leader 节点进行处理。

```go
func (c *Chain) Submit(req *orderer.SubmitRequest, sender uint64) error {

	select {
	// 将 req 和 leadC 封装为一个 submit 结构体，并将其发送到 c.submitC 通道中。
	case c.submitC <- &submit{req, leadC}:

		// 从 leadC 通道中接收 leader 节点的标识
		lead := <-leadC

		// 若当前集群中还没有选出一个 leader，共识功能暂时不可用
		if lead == raft.None {...}

		// 若当前节点不是 leader，将消息转发给 leader
		if lead != c.raftID {
			if err := c.forwardToLeader(lead, req); err != nil {...}
		}
		
	}

}
```

### 2.4 run 函数

`orderer/consensus/etcdraft/chain.go`

Order 节点启动的时候会执行 `go c.run()`，该方法是一个无限循环，负责处理提交请求、处理应用层消息和定时任务等操作。

```go
func (c *Chain) run() {

	// 进入主循环，在循环中通过 select 语句监听多个通道的事件
	for {
		select {
		
		// submitC 通道：接收提交请求，根据当前节点的角色进行处理：
		// 如果节点是 leader，将请求进行排序和处理；如果节点不是 leader，将请求转发给 leader 节点处理。
		case s := <-submitC:

			s.leader <- soft.Lead
			// 如果当前节点的领导者不是自身节点，则继续下一次循环。
			if soft.Lead != c.raftID {
				continue
			}

			// 如果当前节点是领导者，对提交请求进行排序和处理
			batches, pending, err := c.ordered(s.req)

			// 提交排序后的请求并出块
			c.propose(propC, bc, batches...)

		}
	}

}
```

#### 2.4.1 ordered 函数

如果是 leader 节点，那么将会执行 ordered 函数：

* 如果是 Config 消息，调用 BlockCutter.Cut() 直接切割区块
* 如果是 Normal 消息，调用 BlockCutter.Ordered() 进入缓存排序，根据当前交易量决定增加交易或切割

ordered 处理之后，会得到由 BlockCutter 返回的数据包 bathches 和缓存是否还有数据的信息。如果缓存还有余留数据未出块，则启动计时器，否则重置计时器 timer.C。

```go
// Orders the envelope in the `msg` content. SubmitRequest.
func (c *Chain) ordered(msg *orderer.SubmitRequest) (batches [][]*common.Envelope, pending bool, err error) {

	if isconfig {
		// ConfigMsg
	}
	
	// it is a normal message
	// 对交易进行排序 orderer/common/blockcutter/blockcutter.go
	batches, pending = c.support.BlockCutter().Ordered(msg.Payload)
	
}
```

#### 2.4.2 propose 函数

propose 会根据上一步的 batches 数据包调用 createNextBlock 打包出 block，并将 block 传递给 c.ch 通道。

becomeLeader 时会启动一个线程处理该消息，只有 leader 具有 propose 的权限。

```go
func (c *Chain) propose(ch chan<- *common.Block, bc *blockCreator, batches ...[]*common.Envelope) {

	for _, batch := range batches {

		// 创建下一个区块
		b := bc.createNextBlock(batch)

		select {
		// 将创建的区块发送到通道中
		case ch <- b:
		}

	}

}
```

通过调用 c.Node.Propose 将数据传递给底层 raft 状态机。

**从这一段的逻辑可以看出，所有客户端发送给 Orderer 的请求，都会被发送给 raft 的 leader 节点，由 leader 节点排序并生成区块，生成好的区块发送给底层 raft 状态机进行应用同步。**

## 3 验证

### 3.1 StoreBlock 函数

`gossip/privdata/coordinator.go`

coordinator 是 Fabric 中的一个关键组件，负责协调和处理与区块链交互相关的操作，它是 ledger 的主要控制器：

1. **接收和验证区块**：当一个新的区块到达节点时，coordinator 负责验证区块的有效性，包括验证区块的签名、交易的正确性以及相关的权限和策略。

2. **处理私有数据**：在存储区块时，coordinator 会处理区块中的私有数据，包括从分类账中检索私有数据、处理缺失的私有数据以及将私有数据与区块关联起来。

3. **提交区块**：一旦区块和私有数据都成功存储，coordinator 将提交区块到分类账中，使其成为区块链的一部分。

```go
// StoreBlock stores block with private data into the ledger
// 该函数在 gossip/state/state.go 调用
func (c *coordinator) StoreBlock(block *common.Block, privateDataSets util.PvtDataCollections) error {

	// 校验区块信息和交易签名 VSCC
	err := c.Validator.Validate(block)

	// 提交区块，在此过程中执行 MVCC 验证
	err = c.CommitLegacy(blockAndPvtData, &ledger.CommitOptions{})

}
```

### 3.2 Validate 函数

`core/committer/txvalidator/v20/validator.go`

Validate 执行区块的 VSCC 验证。每个交易的验证并行进行。

```go
func (v *TxValidator) Validate(block *common.Block) error {

	go func() {
		for tIdx, d := range block.Data.Data {

			go func(index int, data []byte) {

				// 将每个交易封装成结构体并调用 validateTx，将验证结果发送到 results 通道中
				v.validateTx(&blockValidationRequest{
					d:     data,
					block: block,
					tIdx:  index,
				}, results)
			}(tIdx, d)
		}
	}()

	// 主线程从 results 通道中接收验证结果，并根据结果进行处理
	for i := 0; i < len(block.Data.Data); i++ {
		res := <-results

		if res.err != nil {
			// 如果某个交易验证失败，则会记录第一个失败交易的错误
		} else {
			// 如果结果中没有错误，则将验证结果设置到 txsfltr 中，并将交易 ID 存储到 txidArray 中
		}
	}

}
```

### 3.3 CommitLegacy 函数

`core/ledger/kvledger/kv_ledger.go`

```go
func (l *kvLedger) CommitLegacy(pvtdataAndBlock *ledger.BlockAndPvtData, commitOpts *ledger.CommitOptions) error {

	if err := l.commit(pvtdataAndBlock, commitOpts); err != nil {...}

}
```

```go
// commit commits the block and the corresponding pvt data in an atomic operation.
func (l *kvLedger) commit(pvtdataAndBlock *ledger.BlockAndPvtData, commitOpts *ledger.CommitOptions) error {

	// 读写集的 MVCC 检查
	txstatsInfo, updateBatchBytes, err := l.txmgr.ValidateAndPrepare(pvtdataAndBlock, true)

	// 添加区块到文件系统
	if err = l.commitToPvtAndBlockStore(pvtdataAndBlock); err != nil {...}

	// 提交有效数据到状态数据库
	if err = l.txmgr.Commit(); err != nil {...}

}
```

#### 3.3.1 ValidateAndPrepare 函数

`core/ledger/kvledger/txmgmt/txmgr/lockbased_txmgr.go`

```go
func (txmgr *LockBasedTxMgr) ValidateAndPrepare(blockAndPvtdata *ledger.BlockAndPvtData, doMVCCValidation bool) (
	[]*validation.TxStatInfo, []byte, error,) {

	batch, txstatsInfo, err := txmgr.commitBatchPreparer.ValidateAndPrepareBatch(blockAndPvtdata, doMVCCValidation)

}
```

`core/ledger/kvledger/txmgmt/validation/batch_preparer.go`

```go
// ValidateAndPrepareBatch performs validation of transactions in the block and prepares the batch of final writes
func (p *CommitBatchPreparer) ValidateAndPrepareBatch(blockAndPvtdata *ledger.BlockAndPvtData,
	doMVCCValidation bool) (*privacyenabledstate.UpdateBatch, []*TxStatInfo, error) {

	// 预处理，过滤无效交易
	if internalBlock, txsStatInfo, err = preprocessProtoBlock(...) {...}

	// 读写集检验
	if pubAndHashUpdates, err = p.validator.validateAndPrepareBatch(internalBlock, doMVCCValidation); err != nil {...}

}
```

`core/ledger/kvledger/txmgmt/validation/validator.go`

```go
// validateAndPrepareBatch performs validation and prepares the batch for final writes
func (v *validator) validateAndPrepareBatch(blk *block, doMVCCValidation bool) (*publicAndHashUpdates, error) {

	// 遍历每一笔交易，进行 MVCC 验证，若交易有效，将交易中的写集放入 updates。
	for _, tx := range blk.txs {

		if validationCode, err = v.validateEndorserTX(tx.rwset, doMVCCValidation, updates); err != nil {...}

		if validationCode == peer.TxValidationCode_VALID {
			if err := updates.applyWriteSet(tx.rwset, committingTxHeight, v.db, tx.containsPostOrderWrites); err != nil {...}
		}
	}

}
```

#### 3.3.2 validateEndorserTX 函数

```go
// validateEndorserTX validates endorser transaction
func (v *validator) validateEndorserTX(...) (peer.TxValidationCode, error) {

	// MVCC 验证，不通过则使交易失效
	if doMVCCValidation {
		validationCode, err = v.validateTx(txRWSet, updates)
	}

}
```

```go
func (v *validator) validateTx(txRWSet *rwsetutil.TxRwSet, updates *publicAndHashUpdates) (peer.TxValidationCode, error) {

	for _, nsRWSet := range txRWSet.NsRwSets {
		ns := nsRWSet.NameSpace
		// Validate public reads
		if valid, err := v.validateReadSet(ns, nsRWSet.KvRwSet.Reads, updates.publicUpdates); !valid || err != nil {...}
		// Validate range queries for phantom items
		if valid, err := v.validateRangeQueries(ns, nsRWSet.KvRwSet.RangeQueriesInfo, updates.publicUpdates); !valid || err != nil {...}
		// Validate hashes for private reads
		if valid, err := v.validateNsHashedReadSets(ns, nsRWSet.CollHashedRwSets, updates.hashUpdates); !valid || err != nil {...}
	}

}
```

```go
func (v *validator) validateReadSet(ns string, kvReads []*kvrwset.KVRead, updates *privacyenabledstate.PubUpdateBatch) (bool, error) {

	for _, kvRead := range kvReads {
		if valid, err := v.validateKVRead(ns, kvRead, updates); !valid || err != nil {...}
	}

}
```

```go
func (v *validator) validateKVRead(ns string, kvRead *kvrwset.KVRead, updates *privacyenabledstate.PubUpdateBatch) (bool, error) {

	// 比较版本是否匹配
	if !version.AreSame(committedVersion, readVersion) {...}

}
```

## 4 提交

### 4.1 commitToPvtAndBlockStore 函数

`core/ledger/kvledger/kv_ledger.go`

将区块写入文件系统。

```go
func (l *kvLedger) commitToPvtAndBlockStore(blockAndPvtdata *ledger.BlockAndPvtData) error {

	if err := l.blockStore.AddBlock(blockAndPvtdata.Block); err != nil {...}

}
```

`common/ledger/blkstorage/blockstore.go`

```go
func (store *BlockStore) AddBlock(block *common.Block) error {

	result := store.fileMgr.addBlock(block)

	// 更新区块高度和提交时间
	store.updateBlockStats(block.Header.Number, elapsedBlockCommit)

}
```

```go
func (mgr *blockfileMgr) addBlock(block *common.Block) error {

	blockBytes, info, err := serializeBlock(block)

	// append blockBytesEncodedLen to the file
	err = mgr.currentFileWriter.append(blockBytesEncodedLen, false)
	if err == nil {
		// append the actual block bytes to the file
		err = mgr.currentFileWriter.append(blockBytes, true)
	}

}
```

### 4.2 Commit 函数

`core/ledger/kvledger/txmgmt/txmgr/lockbased_txmgr.go`

提交有效更改到状态数据库。

```go
func (txmgr *LockBasedTxMgr) Commit() error {

	txmgr.commitRWLock.Lock()
	if err := txmgr.db.ApplyPrivacyAwareUpdates(txmgr.current.batch, commitHeight); err != nil {...}
	txmgr.commitRWLock.Unlock()

}
```

`core/ledger/kvledger/txmgmt/privacyenabledstate/db.go`

用于将更新批次应用到底层的数据库。

```go
func (s *DB) ApplyPrivacyAwareUpdates(updates *UpdateBatch, height *version.Height) error {

	// 这两个函数用于将私有数据更新和哈希更新添加到公共数据更新批次中
	addPvtUpdates(combinedUpdates, updates.PvtUpdates)
	addHashedUpdates(combinedUpdates, updates.HashUpdates, !s.BytesKeySupported())
	if err := s.metadataHint.setMetadataUsedFlag(updates); err != nil {...}

	return s.VersionedDB.ApplyUpdates(combinedUpdates.UpdateBatch, height)
}
```

### 4.3 ApplyUpdates 函数

`core/ledger/kvledger/txmgmt/statedb/stateleveldb/stateleveldb.go`

在 leveldb 中的实现。

```go
func (vdb *versionedDB) ApplyUpdates(batch *statedb.UpdateBatch, height *version.Height) error {

	for _, ns := range namespaces {
		updates := batch.GetUpdates(ns)
		for k, vv := range updates {
			dataKey := encodeDataKey(ns, k)

			if vv.Value == nil {
				// 如果值为 nil，表示需要删除该键，函数将在数据库更新批次中添加一个删除操作
				dbBatch.Delete(dataKey)
			} else {
				// 如果值不为 nil，将值编码为字节序列，并将键值对添加到数据库更新批次中
				encodedVal, err := encodeValue(vv)
				dbBatch.Put(dataKey, encodedVal)
			}
		}
	}

	// 将数据库更新批次写入到版本化数据库中，true 表示要求同步写入。
	return vdb.db.WriteBatch(dbBatch, true)

}
```