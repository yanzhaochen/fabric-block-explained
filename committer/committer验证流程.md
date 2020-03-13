# committer 验证交易及提交账本的工作流程

Peer 节点调用 cscc（配置系统链码）加入一个应用通道时，创建 state 模块（ gossip 服务实例）用于接收并处理数据消息。committer 验证交易并提交账本的功能。

## 工作流程简述

committer 接收 DataMsg 类型消息，验证交易并提交到账本的工作流程如图 1 所示，步骤说明如下：

1. 其他节点发来 DataMsg 类型的数据消息，放入 gossipChan 通道中；
2. state 模块建立消息监听循环，若扑捉到 gossipChan 通道中的消息，则调用 queueNewMessage() 方法进行处理；
3. queueNewMessage() 方法先调用 addPayload() 方法将负载消息放入本地的缓冲区，并检查区块号是否为 NEXT（等待提交的下一个区块），实则将消息发送到 readyChan 通道中；
4. state 模块的 deliverPayload() 方法捕捉 readyChan 通道的消息，从中获取有效消息（payload），并解析得到区块数据 rowBlock 和隐私数据 p；
5. state 模块调用 commitBlock() 方法，利用 coordinator 模块，验证交易数据并提交到账本。

![](https://picgo-yanzhao.oss-cn-shenzhen.aliyuncs.com/fabric/state模块提交账本流程1 (2).jpg)

*图 1. committer 验证交易并提交到账本的工作流程图*

*符号说明：蓝色字体表示调用的方法（未标明参数），绿色字体表示数据，实心箭头表示数据流向。*

**coordinator 模块**：封装了交易验证器和账本提交器，分别用于验证交易和提交账本。

**state 模块**：循环监听 gossipChan 通道和 readyChan 通道，调用 queueNewMessage()，deliverPayload() 和 commitBlock() 等方法处理、验证并提交账本。

**核心方法**：state 模块通过 commitBlock() 方法调用 s.ledger.StoreBlock() 方法（state 模块中封装了 coordinator 模块作为 ledger 字段），进而调用 coordinator.StoreBlock() 方法（验证交易和提交账本的核心方法，也是本文分析重点）。

## coordinator.StoreBlock() 方法

```go
// coordinator.StoreBlock() 方法源码分析
// 源码文件：gossip/privdata/coordinator.go
// 为了方便阅读，出错处理和日志操作等操作均用省略号代替

// StoreBlock stores block with private data into the ledger
func (c *coordinator) StoreBlock(block *common.Block, privateDataSets util.PvtDataCollections) error {
	// ===检查区块数据与消息头部的合法性
	if block.Data == nil {
		return ...
	}
	if block.Header == nil {
		return ...
	}
	...

	// === 验证区块，包括调用 vscc 系统链码
	err := c.Validator.Validate(block)
	...
	
	blockAndPvtData := &ledger.BlockAndPvtData{
		Block:        block,
		BlockPvtData: make(map[uint64]*ledger.TxPvtData),
	}
	
	// === 处理隐私数据
	// 获取本地已有的隐私数据读写集
	ownedRWsets, err := computeOwnedRWsets(block, privateDataSets)
	...
	
	// 计算缺失的隐私数据
	privateInfo, err := c.listMissingPrivateData(block, ownedRWsets)
	...
	
	// 从其他 peer 拉取本地缺失的隐私数据
	for len(privateInfo.missingKeys) > 0 && time.Now().Before(limit) {
		c.fetchFromPeers(block.Header.Number, ownedRWsets, privateInfo)
		time.Sleep(pullRetrySleepInterval)
	}
	
	...
    
	// ？？？已有的和拉取的是什么区别？？？
    // 已经在本地的隐私数据代表着什么？？？
    // 是已经提交的吗？？？但那些都已经清理了啊（提交后清理掉的是临时隐私数据）
    // 是最新的吗？？？是，隐私数据会被 prune 以保证时效性
    // 那么拉取得就是新增的或者又发生改变的隐私数据吗？？？
    
	// populate the private RWSets passed to the ledger
	// 构造 blockAndPvtData 结构中的本地已有（已经提交到账本）的隐私数据
	for seqInBlock, nsRWS := range ownedRWsets.bySeqsInBlock() {
		rwsets := nsRWS.toRWSet()
		logger.Debugf("Added %d namespace private write sets for block [%d], tran [%d]", len(rwsets.NsPvtRwset), block.Header.Number, seqInBlock)
		blockAndPvtData.BlockPvtData[seqInBlock] = &ledger.TxPvtData{
			SeqInBlock: seqInBlock,
			WriteSet:   rwsets,
		}
	}
	
	// populate missing RWSets to be passed to the ledger
	// 构造 blockAndPvtData 结构中需要提交到账本但本地缺失的隐私数据读写集
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
	
    // 下面这两个方法中得 transient 是同一个吗？？？
	// === 清理 transient 隐私数据存储对象中的临时隐私数据
	// 从 transient 隐私数据存储对象中的删除指定交易关联的隐私数据读写集（已提交的隐私数据不必再暂存于 transient）
	if len(blockAndPvtData.BlockPvtData) > 0 {
		if err := c.PurgeByTxids(privateInfo.txns); err != nil {...}
	}
    
	// 从 transient 隐私数据存储对象中的删除指定高度一下的隐私数据读写集（保证数据的时效性）
	seq := block.Header.Number
	if seq%c.transientBlockRetention == 0 && seq > c.transientBlockRetention {
		err := c.PurgeByHeight(seq - c.transientBlockRetention)
		...
	}
	
	return nil

}
```

