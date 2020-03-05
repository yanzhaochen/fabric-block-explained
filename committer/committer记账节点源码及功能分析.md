# committer 记账节点源码及功能分析

![](https://picgo-yanzhao.oss-cn-shenzhen.aliyuncs.com/fabric/Screenshot_20200229_002920.png)

## Committer 概述

### Committer 是什么

Committer 记账节点是逻辑节点，通道上同一个组织加入的所有 Peer 节点都默认成为该组织的 Committer 节点。

### Committer 做什么

Committer 记账节点负责验证交易与提交账本，执行步骤如下

- 验证交易消息格式的正确性、签名合法性
- 调用 VSCC（验证系统链码）验证消息合法性、背书策略有效性
- 验证并准备模拟执行结果读写集， 执行 MVCC (多版本并发控制)检查，检查读写冲突并标记交易的一下那行
- 提交账本：提交区块数据到区块文件，提交隐私数据到隐私数据库
- 建立索引信息并保存到区块索引数据库
- 更新有效交易的区块数据与明文数据到状态数据库
- 清理隐私缓存数据库

### Committer 的功能模块

Committer 的功能模块包括交易验证模块和账本提交模块，

-   交易验证模块（Validator 接口）定义了 **Validate** (block *common.Block) 方法，位于

-   账本提交模块（Committer 接口）定义了 **CommitWithPvtData **(blockAndPvtData *ledger.BlockAndPvtDATA) 方法

    

```go
// Committer 相关接口
// core/committer/txvalidator/validator.go

type Validator interface {
	Validate(block *common.Block) error
}

type vsccValidator interface {
	VSCCValidateTx(seq int, payload *common.Payload, envBytes []byte, block *common.Block) (error, peer.TxValidationCode)
}

// core/committer/committer.go

type Committer interface {

	CommitWithPvtData(blockAndPvtData *ledger.BlockAndPvtData, commitOpts *ledger.CommitOptions) error

	GetPvtDataAndBlockByNum(seqNum uint64) (*ledger.BlockAndPvtData, error)

	GetPvtDataByNum(blockNum uint64, filter ledger.PvtNsCollFilter) ([]*ledger.TxPvtData, error)

	LedgerHeight() (uint64, error)

	DoesPvtDataInfoExistInLedger(blockNum uint64) (bool, error)

	GetBlocks(blockSeqs []uint64) []*common.Block

	Close()
}
```

peer.go 中 createChain 函数调用 txvalidator.NewTxValidator(vcs) 函数创建交易验证器，其中封装了 vscc-ValidatorImpl 结构



## 交易验证器

交易验证器对象（txValidator 类型）

-   实现了 Validator 接口的 Validate 方法，验证交易数据合法性
-   封装了 vscc-ValidatorImpl 结构对象（实现了 vsccValidator 接口），验证背书策略有效性

Validate 方法启动 goroutine 对本区块 block.Data.Data 的每个交易数据进行验证，执行过程：

1.  循环启动 goroutine 验证每一个交易
2.  goroutine 调用 validateTx 函数根据 blockValidationRequest 类型的区块验证请求对象验证某个交易，并将结果传入 results 管道
3.  循环等待管道传递每一个验证结果
4.  对有效交易调用 txsfltr.SetFlag 方法进行标识
5.  检查是否允许重复交易，允许则调用 markTXIdDuplicates 方法标记重复交易为合法
6.  调用 invalidTXsForUpgradeCC 方法标记因链码升级而失效的交易
7.  调用 utils.InitBlockMetadata 函数为本区块创建区块元数据，将交易验证结果标识列表写入区块元数据

### Validate 方法的源码分析

```go
// Validate 方法的源码分析
// core/committer/txvalidator/validator.go

// 区块验证请求对象的类型
type blockValidationResult struct {
	tIdx                 int     // 标识该交易在区块中的序号（index）
	validationCode       peer.TxValidationCode
	txsChaincodeName     *sysccprovider.ChaincodeInstance // 该交易所调用的链码实例
	txsUpgradedChaincode *sysccprovider.ChaincodeInstance // 该交易所升级的链码实例
	err                  error
	txid                 string  // 全局唯一的交易 ID，传入 ValidateTx 时为空，在验证过程中赋值为 payload.Header.ChannelHeader.TxId
}

// Validate 方法
func (v *txValidator) Validate(block *common.Block) error {
	...
	// 创建存放交易验证结果标识的列表（长度为该区块中的交易数量）
	txsfltr := ledgerUtil.NewTxValidationFlags(len(block.Data.Data))
	//根据交易序号 tIdx 记录本区块中调用的所有链码实例的字典
	txsChaincodeNames := make(map[int]*sysccprovider.ChaincodeInstance)
    //根据交易序号 tIdx 记录本区块中所有已升级的链码实例的字典
	txsUpgradedChaincodes := make(map[int]*sysccprovider.ChaincodeInstance)
	// 交易 ID 列表
	txidArray := make([]string, len(block.Data.Data))
    // 声明一个传输区块验证结果的管道，用于 goroutines 通信
	results := make(chan *blockValidationResult)
    // 循环启动 goroutine 验证交易
	go func() {
		for tIdx, d := range block.Data.Data {
			tIdxLcl := tIdx // 交易序号（index）
			dLcl := d 		// 交易数据
	
			// 请求信号量（用于控制并发验证）
			v.support.Acquire(context.Background(), 1)
	
			go func() {
                // 验证交易后释放信号量
				defer v.support.Release(1)
				// 传入 blockValidationRequest 类型的验证请求对象，
                // 调用 validateTx 函数验证交易
				validateTx(&blockValidationRequest{
					d:     dLcl,
					block: block,
					tIdx:  tIdxLcl,
					v:     v,
				}, results)
			}()
		}
	}()
	...
	
	// 循环等待管道传递每一个验证结果
	for i := 0; i < len(block.Data.Data); i++ {
		res := <-results // 从 results 管道中读取验证结果
		
		if res.err != nil {// 验证结果为错误
			...	
			}
		} else {// 验证无误
			...
			txsfltr.SetFlag(res.tIdx, res.validationCode) // 设置交易验证码
	
			if res.validationCode == peer.TxValidationCode_VALID {
				if res.txsChaincodeName != nil {
					txsChaincodeNames[res.tIdx] = res.txsChaincodeName // 记录调用的链码实例
				}
				if res.txsUpgradedChaincode != nil {
					txsUpgradedChaincodes[res.tIdx] = res.txsUpgradedChaincode // 记录升级的链码实例
				}
				txidArray[res.tIdx] = res.txid // 记录已经验证过的交易 ,以备后续检查重复交易 
			}
		}
	}
	
	if err != nil {
		return err
	}
	
	// 双花检查
	if v.support.Capabilities().ForbidDuplicateTXIdInBlock() {
		// 根据交易 ID 检查此交易是否和本区块其他已验证的交易重复
        markTXIdDuplicates(txidArray, txsfltr)
	}
	
	// 处理因为链码升级而需要标记为非法交易的情况	
	txsfltr = v.invalidTXsForUpgradeCC(txsChaincodeNames, txsUpgradedChaincodes, txsfltr)
	
	// 调用 utils.InitBlockMetadata 方法为本区块创建区块元数据
	utils.InitBlockMetadata(block)
	
	// 将交易验证结果标识列表写入区块元数据
	block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER] = txsfltr
	
	return nil

}
```

### validateTx 函数的源码分析