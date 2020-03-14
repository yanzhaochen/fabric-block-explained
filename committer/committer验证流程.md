# committer 验证交易及提交账本的工作流程分析

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

当通道上的某个组织或组织子集需要对通道上的其他组织保持数据私密的情况下，可以使用点对点传播的隐私数据集合（包含实际的隐私数据和用于验证的隐私数据哈希值）。隐私数据集合的数据结构见后文分析。

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

   - 从隐私数据集合 `p` 中获取已经达到本节点的隐私数据读写集：调用 `computeOwnedRWsets()` 方法处理与当前区块数据相关的交易隐私数据，构造在该节点上已经存在的隐私数据读写集 `ownedRWsets`；
   - 计算本地缺失的隐私数据读写集：调用 `coordinator.listMissingPrivateData(block, ownedRWsets)`, 计算出需要提交的但是没有随 `DataMsg` 消息到达本地的隐私数据读写集（参数 `ownedRWsets` 为已获取到的本地隐私数据读写集），并从本地的 `transient` 隐私数据库补充 `ownedRWsets`；
   - 从远程节点拉取缺失的隐私数据集合：调用 `fetchFromPeers()` 方法，利用 `Fetcher` 组件从其他 `Peer` 节点请求拉取本节点缺失的隐私数据，并存放于到本地 `transient` 隐私数据库，更新缺失的隐私数据键列表 `missingKeys`。至此 `coordinator.StoreBlock()` 获取到所有隐私数据。
   
4. 调用 `coordinator.CommitWithPvtData()`方法，将区块与隐私数据对象提交到账本；

5. 清理本地的 `transient` 隐私数据库中已提交的隐私数据和指定高度以下的隐私数据。

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

验证与提交账本的核心方法 `coordinator.StoreBlock()` 调用 `computeOwnedRWsets(block, p)` 方法遍历隐私数据集合（`PvtDataCollections` 类型），迭代处理与当前区块相关的交易数据，构造已到达当前节点且在区块 `block` 交易序号内的隐私数据读写集对象 `ownedRWsets`；该对象为 `map[rwSetHey][]byte` 类型，其中值类型为适合做字符处理的 `[]byte` ，用来存放着隐私数据；键类型  `reSetKey` 为包含隐私数据相关信息的结构体，其定义见下方源码。

##### computeOwnedRWsets() 方法源码分析

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

// computeOwnedRWsets() 方法获取已到达当前节点且在区块交易序号内的隐私数据读写集
func computeOwnedRWsets(block *common.Block, blockPvtData util.PvtDataCollections) (rwsetByKeys, error) {
	// 获取区块内最后一个交易序号（index）
	lastBlockSeq := len(block.Data.Data) - 1

    // 构造在该节点上已经存在的隐私数据读写集对象 ownedRWsets
	ownedRWsets := make(map[rwSetKey][]byte)
    
    // 循环处理每一个隐私数据，筛选出在区块范围内的隐私数据，并计算哈希值
	for _, txPvtData := range blockPvtData {
        // 只处理在区块交易序号内的隐私数据
		if lastBlockSeq < int(txPvtData.SeqInBlock) { 
			...
			continue // 交易序号超过区块内的最后一个序号则跳过处理
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
			} // 命名空间内迭代隐私数据
		} // 交易范围内迭代命名空间（链码名称）
	} // 区块范围内迭代交易
	return ownedRWsets, nil
}
```

##### 隐私数据集合数据结构说明

隐私数据集合类型 `PvtDataCollection` 即从 `DataMsg` 解析得到的隐私数据 `p` 的类型，为了深入了解其结构，也为了更直观地展示上述代码三层迭代的关系，例图给出了隐私数据集合的数据结构关系（黑色字体表示数据类型，蓝色字体表示属性名，绿色方框表示数组/切片），可以看到三层数组分别对应三层迭代。

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

#### 2.2.3 listMissingPrivateData() 方法

##### 获取当前节点缺失的隐私数据键列表的流程

![](https://picgo-yanzhao.oss-cn-shenzhen.aliyuncs.com/fabric/获取缺失的隐私数据键列表1.jpg)

*矩形框表示数据，黑色箭头表示生成数据，绿色箭头表示更新数据，蓝色字体表示调用的方法，`transient` 为节点本地的隐私数据库，红色字体标注的 `ownedRWsets` 和 `missingKeys` 分别为流程图重点关注的已在本地的隐私数据（读写集）和本地缺失的隐私数据（键）。*

步骤 1，2 已经在调用 `listMissingPrivateData` 方法前完成了；步骤 3 到 7 为 `listMissingPrivateData` 方法主要工作。

1. 当前节点通过 `state` 模块接收和处理 `DataMsg` 类型消息，并从中解析出区块数据 `block` 和隐私数据集合 `privateDataSets`；
2. 验证交易和提交账本的过程中，调用 `computeOwnedRWsets` 方法构造已随 `DataMsg` 发送到当前节点的隐私数据读写集 `ownedRWsets`；
3.  `listMissingPrivateData` 方法从区块数据 `block` 中获取需要用到的信息，包括记录区块内各个交易有效性的交易过滤器 `txsFilter`（`txValidationFlags` 类型）和在当前区块范围内隐私数据读写集 `privateRWsetsInBlock`（实际上只记录了隐私数据键，用于步骤 6 检查已有的隐私数据是否在当前区块的交易范围内）；
4. 调用 `forEachTxn` 方法遍历当前区块中的有效交易，解析执行结果读写集和背书信息列表，提供给 `inspectTransaction` 方法调用，由后者计算出缺失的隐私数据键列表 `missingKeys` 及背书节点列表 `sources`；
5. 调用 `fetchMissingFromTransientStore` 方法，根据缺失隐私数据键的列表，获取当前缺失且存在于本地 `transient` 隐私数据库中隐私数据读写集， 并更新到已有的隐私数据读写集 `ownedRWsets` 中；
6. 遍历已有的隐私数据读写集 `ownedRWsets`，检查其键是否的 `privateRWsetsInBlock` 中出现，删除掉不在当前区块交易范围内隐私数据，使得 `ownedRWsets` 只保留当前区块交易范围内的隐私数据读写集；
7. 遍历缺失的隐私数据键列表 `missingKeys` ，检查隐私数据键是否在已有的隐私数据读写集 `ownedRWsets` 中出现，有则说明该键对应的隐私数据已经在本地，将该键从缺失列表中删除。

至此  `listMissingPrivateData` 方法完成了根据本地 transsient 隐私数据库更新已有的隐私数据读写集 `ownedRWsets` 和获取缺失的隐私数据键列表 `missingKeys` 的任务。

##### `listMissingPrivateData` 方法源码：

```go
// gossip/privdata/coordinator.go
// listMissingPrivateData 方法根据区块数据 block 和已经计算出来的本地已有的隐私数据读写集 ownedRWsets，
// 计算出本地缺失的隐私数据读写集，并尝试从本地 transien 隐私数据库中获取，
// 最终获得当前节点缺失的隐私数据键列表 privateInfo.missingKeys
func (c *coordinator) listMissingPrivateData(block *common.Block, ownedRWsets map[rwSetKey][]byte) (*privateDataInfo, error) {
	// 检查区块元数据的合法性
	if block.Metadata == nil || len(block.Metadata.Metadata) <= int(common.BlockMetadataIndex_TRANSACTIONS_FILTER) {
		return nil, errors.New("Block.Metadata is nil or Block.Metadata lacks a Tx filter bitmap")
	}
	// 获取区块元数据中标识交易结果的列表 txsFilter（包含成功的和失败的交易）
	txsFilter := txValidationFlags(block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER])
	// 标识交易结果的列表长度应当等于区块中所有交易的数量
	if len(txsFilter) != len(block.Data.Data) {
		...
	}

	sources := make(map[rwSetKey][]*peer.Endorsement)
	privateRWsetsInBlock := make(map[rwSetKey]struct{})
	missing := make(rwSetKeysByTxIDs)
	data := blockData(block.Data.Data)
    // 构建 bi 对象用于查看当前节点上的隐私数据情况
    // (注意到这里是引用，直接修改 sources 或 missing
    // 等效于修改 bi.sources 或 bi.missingKeys)
	bi := &transactionInspector{ 
		sources:              sources,              // 拥有隐私数据的背书节点列表
		missingKeys:          missing,              // 缺失隐私数据键列表
		ownedRWsets:          ownedRWsets,          // 当前节点已获取到的隐私数据读写集
		privateRWsetsInBlock: privateRWsetsInBlock, // 在当前区块范围内隐私数据读写集
		coordinator:          c,                    // 调用此方法的 coordinator 对象本身
	}
	
	// == 获取当前区块中经过 Endorsor 背书的有效交易的交易 ID 列表 txList
	// forEachTxn 方法遍历当前区块中的有效交易，解析执行结果读写集和背书信息列表，
	// 提供给 bi.inspectTransaction 方法调用，
	// 由后者计算出缺失的隐私数据键列表 bi.missingKeys 及背书节点列表 bi.sources
	txList, err := data.forEachTxn(txsFilter, bi.inspectTransaction)
	...
	
	// 构造隐私数据信息对象，是本方法要返回的结果
    // (同样注意到这里是引用)
	privateInfo := &privateDataInfo{
		sources: sources,
		// missingKeys 属性未显式赋值，初始化为 nil
		missingKeysByTxIDs: missing,
		txns:               txList,
	}
	...
	
	// fetchMissingFromTransientStore 方法根据缺失隐私数据键的列表，
	// 获取当前缺失且存在于本地 transient 隐私数据库中隐私数据读写集，
	// 并补充到已获取到的隐私数据读写集 ownedRWsets 中
    // (注意到实参 privateInfo.missingKeysByTxIDs 即 bi.inspectTransaction 计算出来的 missingKeys 对象)
	c.fetchMissingFromTransientStore(privateInfo.missingKeysByTxIDs, ownedRWsets)
	
	// 遍历已获取到的隐私数据读写集 ownedRWsets，
	// 检查其键是否的 privateRWsetsInBlock 中出现，
	// 否则说明隐私数据不在当前区块范围内，剔除掉，
	// 使得 ownedRWsets 只保留当前区块交易范围内的读写集
	for k := range ownedRWsets {
		if _, exists := privateRWsetsInBlock[k]; !exists {
			...
			delete(ownedRWsets, k)
		}
	}
	
	// 将隐私数据信息对象的 missingKeysByTxIDs 属性转换到 missingKeys 属性上
	// missingKeys 为 rwsetKeys 类型（type rwsetKeys map[rwSetKey]struct{}）
	privateInfo.missingKeys = privateInfo.missingKeysByTxIDs.flatten()
	
	// Remove all keys we already own
	// 剔除已有读写集的键，
	// 最终得到当前节点缺失的隐私数据键列表 privateInfo.missingKeys，
	// 随 privateInfo 对象返回给调用者（coordinator.StoreBlock 方法）
	privateInfo.missingKeys.exclude(func(key rwSetKey) bool {
		_, exists := ownedRWsets[key]
		return exists
	})
	
	return privateInfo, nil

}
```



##### `bi.inspectTransaction` 方法源码

```go
// 计算出缺失的隐私数据键列表及背书节点列表，保存在 bi 中
func (bi *transactionInspector) inspectTransaction(seqInBlock uint64, chdr *common.ChannelHeader, txRWSet *rwsetutil.TxRwSet, endorsers []*peer.Endorsement) {
	// 遍历每个交易
	for _, ns := range txRWSet.NsRwSets {
        // 遍历交易中的每个读写集（一个交易可）
		for _, hashedCollection := range ns.CollHashedRwSets {
			if !containsWrites(chdr.TxId, ns.NameSpace, hashedCollection) {
				continue
			}
			// 获取隐私数据集合访问策略
			policy := bi.accessPolicyForCollection(chdr, ns.NameSpace, hashedCollection.CollectionName)
			if policy == nil {
				continue
			}
			// 检查当前节点是够具有访问该隐私数据集合的权限
			if !bi.isEligible(policy, ns.NameSpace, hashedCollection.CollectionName) {
				continue
			}
			// 根据该隐私数据集合的信息构造隐私数据键（rwSetKey 类型），
            // 标记到 bi.privateRWsetsInBlock 中
			key := rwSetKey{
				txID:       chdr.TxId,
				seqInBlock: seqInBlock,
				hash:       hex.EncodeToString(hashedCollection.PvtRwSetHash),
				namespace:  ns.NameSpace,
				collection: hashedCollection.CollectionName,
			}
			bi.privateRWsetsInBlock[key] = struct{}{}
			// 根据键 key 检查已获取到的隐私数据读写集中是否已经存在当前隐私数据，
			// 如果不存在则将隐私数据键 key 保存到
            // 相应的缺失隐私数据的键列表 missingKeys[txAndSeq] 中，
			// 并将拥有相应隐私数据的背书节点列表保存到 bi.sources[key] 中
			if _, exists := bi.ownedRWsets[key]; !exists {
				txAndSeq := txAndSeqInBlock{
					txID:       chdr.TxId,
					seqInBlock: seqInBlock,
				}
				bi.missingKeys[txAndSeq] = append(bi.missingKeys[txAndSeq], key)
				bi.sources[key] = endorsersFromOrgs(ns.NameSpace, hashedCollection.CollectionName, endorsers, policy.MemberOrgs())
			}
		} // for all hashed RW sets
	} // for all RW sets
}
```

