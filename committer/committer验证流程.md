# committer 验证交易及提交账本的工作流程

## 一、模块说明

### 1.1 coordinator 模块

`coordinator` 模块封装了交易验证器和账本提交器，分别用于验证交易和提交账本。

### 1.2 state 模块

`Peer` 节点调用 `cscc`（配置系统链码）加入一个应用通道时，创建 `state` 模块（ `gossip` 服务实例）用于接收并处理数据消息。

`state` 模块循环监听 `gossipChan` 通道和 `readyChan` 通道，调用 `queueNewMessage()`，`deliverPayload()` 和 `commitBlock()` 等方法处理、验证并提交账本。

`state` 模块通过 `gossipChan` 通道接收 `DataMsg` 类型消息，其中包含了区块和隐私数据；通过 `readyChan` 通道在模块内部通知已经获取到正在等待的 `DataMsg` 类型消息，需要进行验证和提交。

**核心方法**：`state` 模块通过 `commitBlock()` 方法调用 `s.ledger.StoreBlock()` 方法（state 模块中封装了 `coordinator` 模块作为 `ledger` 字段），进而调用 `coordinator.StoreBlock()` 方法（验证交易和提交账本的核心方法，也是本文分析重点）。

1.3 模块的创建过程（待补充

### 1.4 transient 隐私数据库

当通道上的某个组织或组织子集需要对通道上的其他组织保持数据私密的情况下，可以使用点对点传播的隐私数据集合（包含实际的隐私数据和用于验证的隐私数据哈希值）。

`transient` 隐私数据库相当于账本中隐私数据的本地“缓存”，`Committer` 节点提交账本之后要对 `transient` 隐私数据库进行更新。隐私数据的交易过程为

![隐私数据库1](https://picgo-yanzhao.oss-cn-shenzhen.aliyuncs.com/fabric/隐私数据库1.jpg)

1. 客户端执行包含隐私数据的链码，发送包含隐私数据的提案请求给隐私数据 `Endorsor` 节点；
2. `Endorsor` 节点进行模拟交易，将隐私数据存放到本节点的 `transient` 隐私数据库中，
3. `Endorsor` 节点将经过背书的隐私数据哈希值通过提案响应返回给客户端，并通过 `gossip` 协议发送隐私数据给其他授权节点；
4. 客户端将包含经过背书的隐私数据哈希值的交易发送给 `Orderer` 节点请求排序出块。不包含在组织内部的 `Orderer` 无权访问（事实上也获取不到）隐私数据，但因为有经过背书的隐私数据哈希值，能够检验交易的正确性。

## 二、committer 工作流程

本章第一节介绍了committer 工作的整体流程；本章后文介绍了核心方法 `coordinator.StoreBlock()` 。

### 2.1 committer 工作流程简述

`committer` 通过 `state` 模块接收 `DataMsg` 类型消息，验证交易并提交到账本的工作流程如图 1 所示，步骤说明如下：

1. 其他节点发来 `DataMsg` 类型的数据消息，放入 `gossipChan` 通道中；
2. `state` 模块建立消息监听循环，若扑捉到 `gossipChan` 通道中的消息，则调用 `queueNewMessage()` 方法进行处理；
3. `queueNewMessage()` 方法先调用 `addPayload()` 方法将负载消息放入本地的缓冲区，并检查区块号是否为 `NEXT`（等待提交的下一个区块），是则将消息发送到 `readyChan` 通道中（通知 `deliverPayload()` 方法已经获取到正在等待的消息了）；
4. `state` 模块的 `deliverPayload()` 方法捕捉 `readyChan` 通道的消息，从中获取有效消息（`payload`），并解析得到区块数据 `rowBlock`（`common.block` 类型）和隐私数据 `p` 或称为 `privateDataSets`（`util.PvtDataCollections` 类型）；
5. `state` 模块调用 `commitBlock(rowBlock, p)` 方法（值得注意的是，区块数据 `rowBlock` 和隐私数据 `p` 都作为参数传入），利用 `coordinator` 模块，验证交易数据并提交到账本。

![](https://picgo-yanzhao.oss-cn-shenzhen.aliyuncs.com/fabric/state模块提交账本流程1.jpg)

*图 1. committer 验证交易并提交到账本的工作流程图*

*符号说明：蓝色字体表示调用的方法（未标明参数），绿色字体表示数据，实心箭头表示数据流向。*

### 2.2 coordinator.StoreBlock() 方法

2.2.1 小节介绍了 `coordinator.StoreBlock()` 方法的源码，主要聚焦于该方法的工作步骤；随后的几个小节介绍了 `coordinator.StoreBlock()` 方法调用的其他方法；

2.2.2 小节介绍了`computeOwnedRWsets()` 方法；

（待补充

#### 2.2.1 coordinator.StoreBlock() 方法

##### `coordinator.StoreBlock()` 方法的工作流程：

1. 检查区块数据与消息头部的合法性（`block.Data` 和 `block.Header `非空则合法）；

2. 调用 `Validate()` 方法，验证区块中交易数据的合法性并调用 `vscc` 系统链码验证背书策略的合法性；`Validate()` 是一个核心方法，其分析见其他文档；

3. 处理隐私数据，包括获取已经随 `DataMsg` 消息发送到本节点的隐私数据集合，计算本地缺失的隐私数据和从远程节点拉取缺失的隐私数据集合；构造要提交到账本的区块与隐私数据对象（`BlockAndPvtData` 结构）：

   - 从隐私数据 `p`中获取已经达到本节点的隐私数据集合：调用 `computeOwnedRWsets()` 方法处理与当前区块数据相关的交易，构造在该节点上已经存在的隐私数据读写集 `ownedRWsets`；
- 计算本地缺失的隐私数据：调用 `coordinator.listMissingPrivateData(block, ownedRWsets)`, 计算出需要提交的但是没有随 `DataMsg` 消息到达本地的隐私数据读写集（参数 `ownedRWsets` 为已获取到的本地隐私数据读写集）；
   - 从远程节点拉取缺失的隐私数据集合：调用 `fetchFromPeers()` 方法，利用 `Fetcher` 组件从其他 `Peer` 节点请求拉取本节点缺失的隐私数据，并存放于到本地 `transient` 隐私数据库。
   
4. 调用 `coordinator.CommitWithPvtData()`方法，将区块与隐私数据对象提交到账本；

5. 清理本地的 `transient` 隐私数据库中临时存放的隐私数据。

##### `coordinator.StoreBlock()` 方法的源码分析：

```go
// coordinator.StoreBlock() 方法源码分析
// 源码位于文件：gossip/privdata/coordinator.go
// 为了方便阅读，出错处理和日志操作等操作均用省略号代替，部分代码顺序有所调整
// 本文所有源码均为 release-v1.4 版本

// coordinator.StoreBlock() 方法用于验证数据并提交账本
// 值得注意的是参数（区块数据 block 和隐私数据集合 privateDataSets）
// 均解析自 DataMsg 的有效消息载荷 payload
func (c *coordinator) StoreBlock(block *common.Block, privateDataSets util.PvtDataCollections) error {
	// 检查区块数据与消息头部的合法性
	if block.Data == nil {...}
    if block.Header == nil {...}
	...

    // 调用 Validate() 方法，验证区块中交易数据的合法性并调用 vscc 系统链码验证背书策略的合法性
	err := c.Validator.Validate(block)
	...
	
	// 处理隐私数据
	// 1. 获取当前节点现存的隐私数据读写集
	ownedRWsets, err := computeOwnedRWsets(block, privateDataSets)
	...
	
	// 2. 计算当前节点缺失的隐私数据
	privateInfo, err := c.listMissingPrivateData(block, ownedRWsets)
	...
	
	// 3. 从其他 peer 拉取本地缺失的隐私数据
	for len(privateInfo.missingKeys) > 0 && time.Now().Before(limit) {
		c.fetchFromPeers(block.Header.Number, ownedRWsets, privateInfo)
		time.Sleep(pullRetrySleepInterval)
	}
	...
    
	// 已有的和拉取的是什么区别？？？
    // 已经在本地的隐私数据代：是从 payload 来的
    // 是已经提交的吗？？？但那些都已经清理了啊（提交后清理掉的是临时隐私数据）
    // 是最新的吗？？？是，隐私数据会被 prune 以保证时效性
    // 那么拉取得就是新增的或者又发生改变的隐私数据吗？？？
    
    // 构造区块与隐私数据对象（BlockAndPvtData 结构）
	blockAndPvtData := &ledger.BlockAndPvtData{
		Block:        block,
		BlockPvtData: make(map[uint64]*ledger.TxPvtData),
	}
    
	// populate the private RWSets passed to the ledger
	// 1. 构造 blockAndPvtData 结构中的本地已有（已经提交到账本）的隐私数据
	for seqInBlock, nsRWS := range ownedRWsets.bySeqsInBlock() {
		rwsets := nsRWS.toRWSet()
		...
		blockAndPvtData.BlockPvtData[seqInBlock] = &ledger.TxPvtData{
			SeqInBlock: seqInBlock,
			WriteSet:   rwsets,
		}
	}
	
	// populate missing RWSets to be passed to the ledger
	// 2. 构造 blockAndPvtData 结构中需要提交到账本但本地缺失的隐私数据读写集
	for missingRWS := range privateInfo.missingKeys {
		blockAndPvtData.Missing = append(blockAndPvtData.Missing, ledger.MissingPrivateData{
			TxId:       missingRWS.txID,
			Namespace:  missingRWS.namespace,
			Collection: missingRWS.collection,
			SeqInBlock: int(missingRWS.seqInBlock),
		})
	}
	
	// commit block and private data
	// 提交区块和隐私数据
	err = c.CommitWithPvtData(blockAndPvtData)
	...
	
	// 清理 transient 隐私数据存储对象中的临时隐私数据 ？？？
	// 1. 删除指定交易关联的隐私数据读写集（已提交的隐私数据不再存于 transient）？？？
	if len(blockAndPvtData.BlockPvtData) > 0 {
		if err := c.PurgeByTxids(privateInfo.txns); err != nil {...}
	}
    
	// 2. 删除指定高度一下的隐私数据读写集（保证数据的时效性）？？？
	seq := block.Header.Number
	if seq%c.transientBlockRetention == 0 && seq > c.transientBlockRetention {
		err := c.PurgeByHeight(seq - c.transientBlockRetention)
		...
	}
	
	return nil

}
```



#### 2.2.2 computeOwnedRWsets() 方法

`coordinator.StoreBlock()` 方法调用 `computeOwnedRWsets(block, p)` 方法处理与当前区块数据相关的交易，构造在该节点上已经存在的隐私数据读写集对象 `ownedRWsets`.

从隐私数据 `p` 中获取已经达到本节点的隐私数据集合：调用 `computeOwnedRWsets()` 方法处理与当前区块数据相关的交易，构造在该节点上已经存在的隐私数据读写集对象 `ownedRWsets`；该对象为 `map[rwSetHey][]byte` 类型，其中值类型为适合做字符处理的 `[]byte` ，用来存放着隐私数据；键类型  `reSetKey` 为包含隐私数据相关信息的结构体，其定义见下方源码。

##### computeOwnedRWsets()相关源码分析

```go
// computeOwnedRWsets() 方法相关源码
// 源码位于文件：gossip/privdata/coordinator.go

// 隐私数据读写集对象 ownedRWsets 的键类型 reSetKey 定义
type rwSetKey struct {
    txID       string   // 交易 ID
    seqInBlock uint64   // 交易序号
    namespace  string   // 命名空间（链码名称）
    collection string   // 隐私数据集合名称
    hash       string   // 读写集哈希值，可供无法查看隐私数据的节点检验隐私数据的合法性
}

// computeOwnedRWsets() 方法获取已达到当前节点的且在区块交易序号内的隐私数据读写集
func computeOwnedRWsets(block *common.Block, blockPvtData util.PvtDataCollections) (rwsetByKeys, error) {
	// 获取区块内最后一个交易序号（index）
	lastBlockSeq := len(block.Data.Data) - 1

    // 构造在该节点上已经存在的隐私数据读写集对象 ownedRWsets
	ownedRWsets := make(map[rwSetKey][]byte)
    
    // 循环处理每一个隐私数据，筛选出在区块范围内的隐私数据，并计算哈希值
	for _, txPvtData := range blockPvtData {
		if lastBlockSeq < int(txPvtData.SeqInBlock) { 
			...
			continue // 隐私数据关联的交易序号超过区块内的最后一个序号则跳过处理
		}
		// 从区块获取特定交易的 envelope
		env, err := utils.GetEnvelopeFromBlock(block.Data.Data[txPvtData.SeqInBlock])
		...
		// 从 envelope 获取有效消息 payload
		payload, err := utils.GetPayload(env)
		...
	
		// 获取应用通道头部 chdr
		chdr, err := utils.UnmarshalChannelHeader(payload.Header.ChannelHeader)
		...
		
		for _, ns := range txPvtData.WriteSet.NsPvtRwset {
			for _, col := range ns.CollectionPvtRwset {
				// 计算隐私数据读写集的哈希值
				computedHash := hex.EncodeToString(util2.ComputeSHA256(col.Rwset))
				// 将隐私数据和相关信息写入要返回的隐私数据读写集字典
				ownedRWsets[rwSetKey{
					txID:       chdr.TxId,
					seqInBlock: txPvtData.SeqInBlock,
					collection: col.CollectionName,
					namespace:  ns.Namespace,
					hash:       computedHash,
				}] = col.Rwset
			} // 命名空间内迭代隐私数据集合
		} // 交易范围内迭代命名空间（链码名称）
	} // 区块范围内迭代交易
	return ownedRWsets, nil
}
```

##### 隐私数据集合数据结构说明

为了更直观地展示上述代码三层迭代的关系，例图给出了隐私数据集合的数据结构关系（黑色字体表示数据类型，蓝色字体表示属性名，绿色方框表示数组/切片），可以看到三层数组分别对应三层迭代。

![](https://picgo-yanzhao.oss-cn-shenzhen.aliyuncs.com/fabric/隐私数据集合数据结构.jpg)



```go
// 以下代码为隐私数据集合结构定义

// gossip/util/privdata.go
type PvtDataCollections []*ledger.TxPvtData

// core/ledger/ledger_interface.go
type TxPvtData struct {
	SeqInBlock uint64
	WriteSet   *rwset.TxPvtReadWriteSet
}

// protos/ledger/rwset/rwset.pb.go
type TxPvtReadWriteSet struct {
	DataModel  TxReadWriteSet_DataModel `protobuf:"varint,1,opt,name=data_model,json=dataModel,enum=rwset.TxReadWriteSet_DataModel" json:"data_model,omitempty"`
	NsPvtRwset []*NsPvtReadWriteSet     `protobuf:"bytes,2,rep,name=ns_pvt_rwset,json=nsPvtRwset" json:"ns_pvt_rwset,omitempty"`
}

type NsPvtReadWriteSet struct {
	Namespace          string                       `protobuf:"bytes,1,opt,name=namespace" json:"namespace,omitempty"`
	CollectionPvtRwset []*CollectionPvtReadWriteSet `protobuf:"bytes,2,rep,name=collection_pvt_rwset,json=collectionPvtRwset" json:"collection_pvt_rwset,omitempty"`
}

type CollectionPvtReadWriteSet struct {

  CollectionName string `protobuf:"bytes,1,opt,name=collection_name,json=collectionName" json:"collection_name,omitempty"`
  Rwset     []byte `protobuf:"bytes,2,opt,name=rwset,proto3" json:"rwset,omitempty"`

}
```

