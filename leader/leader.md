# Leader主节点请求Orderer服务节点发送通道账本区块。  
## 一、  
Leader主节点的作用是通过Dleliver服务接口代表组织从Orderer节点接受区块，并分发到组织内的其它节点。Leader主节点的产生方式主要有动态选举二、Leader主节点加入channel后，Gossip服务器调用InitializeChannel()的方法来启动指定通道上的deliveryService服务模块。InitializeChannel()方法首先调用了g.deliveryFactory.Service()方法，构造Deliver服务客户端配置信息(deliverClientconfig类型)，再创建指定ChannelD的deliver服务实例(deliverServiceImpl类型)并验证配置信息。接着，将创建好的实例注册到deliverService服务模块字典块中，检查当前节点的Leader配置的标志位。并调用StartDeliverForChannel()方法启动Learder主节点的deliver服务客户端，以请求所在通道账本的区块数据。  
源码在gossip/service/gossip_service.go下  
```
func (g *gossipServiceImpl) InitializeChannel(chainID string, endpoints []string, support Support) {

  ...... //部分内容省略	
	if g.deliveryService[chainID] == nil {
		var err error
		g.deliveryService[chainID], err = g.deliveryFactory.Service(g, endpoints, g.mcs)
		if err != nil {
			logger.Warningf("Cannot create delivery client, due to %+v", errors.WithStack(err))
		}
	}

	// Delivery service might be nil only if it was not able to get connected
	// to the ordering service
	if g.deliveryService[chainID] != nil {
		// Parameters:
		//              - peer.gossip.useLeaderElection
		//              - peer.gossip.orgLeader
		//
		// are mutual exclusive, setting both to true is not defined, hence
		// peer will panic and terminate
		leaderElection := viper.GetBool("peer.gossip.useLeaderElection")
		isStaticOrgLeader := viper.GetBool("peer.gossip.orgLeader")

		if leaderElection && isStaticOrgLeader {
			logger.Panic("Setting both orgLeader and useLeaderElection to true isn't supported, aborting execution")
		}

		if leaderElection {
			logger.Debug("Delivery uses dynamic leader election mechanism, channel", chainID)
			g.leaderElection[chainID] = g.newLeaderElectionComponent(chainID, g.onStatusChangeFactory(chainID, support.Committer))
		} else if isStaticOrgLeader {
			logger.Debug("This peer is configured to connect to ordering service for blocks delivery, channel", chainID)
			g.deliveryService[chainID].StartDeliverForChannel(chainID, support.Committer, func() {})
		} else {
			logger.Debug("This peer is not configured to connect to ordering service for blocks delivery, channel", chainID)
		}
	} else {
		logger.Warning("Delivery client is down won't be able to pull blocks for chain", chainID)
	}
}
```
## 二、
Deliver服务实例的StartDeliverForChannel()首先从blockProviders字典中获取指定通道的区块提供者对象，检查此时是否Deliver已被实例化。接着，该方法调用d.newClient()->NewBroadcastClient()函数，创建指定通道的broadcastClient结构客户端，用于和Orderer节点建立连接，以发送请求接收区块数据结果。源码在core/deliverservice/deliveryclient.go文件  
```
func (d *deliverServiceImpl) StartDeliverForChannel(chainID string, ledgerInfo blocksprovider.LedgerInfo, finalizer func()) error {
	d.lock.Lock()
	defer d.lock.Unlock()
	if d.stopping {
		errMsg := fmt.Sprintf("Delivery service is stopping cannot join a new channel %s", chainID)
		logger.Errorf(errMsg)
		return errors.New(errMsg)
	}
	if _, exist := d.blockProviders[chainID]; exist {
		errMsg := fmt.Sprintf("Delivery service - block provider already exists for %s found, can't start delivery", chainID)
		logger.Errorf(errMsg)
		return errors.New(errMsg)
	} else {
		client := d.newClient(chainID, ledgerInfo)
		logger.Debug("This peer will pass blocks from orderer service to other peers for channel", chainID)
		d.blockProviders[chainID] = blocksprovider.NewBlocksProvider(chainID, client, d.conf.Gossip, d.conf.CryptoSvc)
		go d.launchBlockProvider(chainID, finalizer)
	}
	return nil
}
```
在StartDeliverForChannel()方法调用d.newClient()->NewBroadcastClient()函数，创建指定通道的broadcastClient结构客户端。同时该函数还创建了新的区块请求对象request(blocksRequester类型)来封装broadcastClient的结构客户端。newClient函数源码在core/deliverservice/deliveryclient.go  
```

func (d *deliverServiceImpl) newClient(chainID string, ledgerInfoProvider blocksprovider.LedgerInfo) *broadcastClient {
	reconnectBackoffThreshold := getReConnectBackoffThreshold()
	reconnectTotalTimeThreshold := getReConnectTotalTimeThreshold()
	requester := &blocksRequester{
		tls:     viper.GetBool("peer.tls.enabled"),
		chainID: chainID,
	}
	broadcastSetup := func(bd blocksprovider.BlocksDeliverer) error {
		return requester.RequestBlocks(ledgerInfoProvider)
	}
	backoffPolicy := func(attemptNum int, elapsedTime time.Duration) (time.Duration, bool) {
		if elapsedTime > reconnectTotalTimeThreshold {
			return 0, false
		}
		sleepIncrement := float64(time.Millisecond * 500)
		attempt := float64(attemptNum)
		return time.Duration(math.Min(math.Pow(2, attempt)*sleepIncrement, reconnectBackoffThreshold)), true
	}
	connProd := comm.NewConnectionProducer(d.conf.ConnFactory(chainID), d.conf.Endpoints)
	bClient := NewBroadcastClient(connProd, d.conf.ABCFactory, broadcastSetup, backoffPolicy)
	requester.client = bClient
	return bClient
}
```
















