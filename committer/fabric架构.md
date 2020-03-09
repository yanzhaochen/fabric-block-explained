# Block 数据结构

![](https://picgo-yanzhao.oss-cn-shenzhen.aliyuncs.com/fabric/20200304221227.png)

![](https://picgo-yanzhao.oss-cn-shenzhen.aliyuncs.com/fabric/20200304222839.png)

# Peer

Peer 节点为逻辑节点

- Endorser：接收 Client 的签名提案消息请求，模拟交易提案，为模拟结果签名背书，发送提案响应消息给 Client
- Committer：检查交易消息，验证背书，检查读写集冲突，标记交易有效性，提交账本

# Orderer

Orderer 节点亦为逻辑节点，负责管理 channel（其账本与配置），提供 Broadcast、Orderer 和 Deliver 服务

# Client

用户与 Fabric Network 交互的接口

- Fabric-CA Client：负责节点注册登记
- Fabric Client：负责网络配置与节点管理
    1. CLI 命令行客户端
    2. 各种语言的 SDK 客户端

# CA

CA 节点管理网络中的成员身份信息

# Consensus

共识是对交易达成一致观点并保证账本最终状态一致；包含于交易背书、排序、验证的过程中

- 背书阶段
- 排序阶段
- 验证阶段

# MSP 

Membership Service Provider 成员关系服务模块，提供身份验证的实体抽象概念，通常一个 MSP 对应一个组织或联盟

# Organization

拥有共同信任的根证书的成员集合，包含

- Admin
- Member

# Consortium

相互合作的多个组织，使用相同 Orderer 和 channel 创建策略

# Transaction

调用 chaincode 改变 ledger 状态数据的一次操作

- 普通交易：排序后成块
- 配置交易：单独成块

# Block

一段时间内的交易集合；数据结构包含

- Header：区块号，前块 Hash，当前块 Hash
- Data：交易集合
- Metadata：区块签名，最新配置块块号，交易过滤器（表示交易有效性），Orderer 配置信息

# Chaincode

链码/智能合约，有

- 系统链码（scc）

    1. CSCC （配置）
    2. ESCC （背书）
    3. LSCC （生命周期）
    4. QSCC （查询）
    5. VSCC （验证）

- 用户链码

必须实现 Init 和 Invoke 方法；必须经过 Install 和 Initiate 才能调用

# Channel

Orderer 管理的原子广播渠道，各 channel 间相互隔离

- System Channel
- Application Channel

# Ledger

Ledger 与 Channel 一一对应；提供了多个数据库与文件用于存储账本数据

- idStore 数据库
- 区块数据文件
- 隐私数据库
- 状态数据库
- 历史数据库
- 区块索引数据库
- transient 隐私数据库

# Policy

策略是对系统资源访问权限进行管理的方法

- 交易背书策略
- 链码实例化策略
- 通道管理策略

