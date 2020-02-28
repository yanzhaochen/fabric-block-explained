# fabric-block-explained

## 要求/目标

- 结合源码详细分析 block 的创建、记账以及同步过程

## 分工

- Wei：orderer 节点交易收集及区块创建
- Song：leader peer 收集区块以及分发给组织内其他 peer
- Chen：commiter peer 验证及提交至账本
- Zhang：peer 之间状态与数据同步

## 产出物

- Markdown 格式的文档若干
- 整合的分析报告