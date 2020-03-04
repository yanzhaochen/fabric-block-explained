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
## 三、 
### 通过broadcastClient客户端发送请求并等待区块  
   首先、broadcastClient客户端首先调用bc.connect()方法建立Deliver连接。bc.connect首先调用bc.prod.NewConnection()方法，该方法创建并返回新的gRPC连接conn与可用的Orderer服务端点endpoint。接着调用bc.createClient(conn).Deliver(ctx)请求调用Deliver服务。然后调用bc.afterConnect()方法来创建完成Deliver服务实例的broadcastClient客户端。bc.afterConnect()调用了bc.onConnect(bc)方法，此方法是自定义的broadcastSetup()方法。bc.connect()、bc.afterConnect()源码在core/deliverservice/client.go  
```
func (bc *broadcastClient) connect() error {
	bc.mutex.Lock()
	bc.endpoint = ""
	bc.mutex.Unlock()
	conn, endpoint, err := bc.prod.NewConnection()
	logger.Debug("Connected to", endpoint)
	if err != nil {
		logger.Error("Failed obtaining connection:", err)
		return err
	}
	ctx, cf := context.WithCancel(context.Background())
	logger.Debug("Establishing gRPC stream with", endpoint, "...")
	abc, err := bc.createClient(conn).Deliver(ctx)
	if err != nil {
		logger.Error("Connection to ", endpoint, "established but was unable to create gRPC stream:", err)
		conn.Close()
		cf()
		return err
	}
	err = bc.afterConnect(conn, abc, cf, endpoint)
	if err == nil {
		return nil
	}
	logger.Warning("Failed running post-connection procedures:", err)
	// If we reached here, lets make sure connection is closed
	// and nullified before we return
	bc.Disconnect(false)
	return err
}
func (bc *broadcastClient) afterConnect(conn *grpc.ClientConn, abc orderer.AtomicBroadcast_DeliverClient, cf context.CancelFunc, endpoint string) error {
	logger.Debug("Entering")
	defer logger.Debug("Exiting")
	bc.mutex.Lock()
	bc.endpoint = endpoint
	bc.conn = &connection{ClientConn: conn, cancel: cf}
	bc.blocksDeliverer = abc
	if bc.shouldStop() {
		bc.mutex.Unlock()
		return errors.New("closing")
	}
	bc.mutex.Unlock()
	// If the client is closed at this point- before onConnect,
	// any use of this object by onConnect would return an error.
	err := bc.onConnect(bc)
	// If the client is closed right after onConnect, but before
	// the following lock- this method would return an error because
	// the client has been closed.
	bc.mutex.Lock()
	defer bc.mutex.Unlock()
	if bc.shouldStop() {
		return errors.New("closing")
	}
	// If the client is closed right after this method exits,
	// it's because this method returned nil and not an error.
	// So- connect() would return nil also, and the flow of the goroutine
	// is returned to doAction(), where action() is invoked - and is configured
	// to check whether the client has closed or not.
	if err == nil {
		return nil
	}
	logger.Error("Failed setting up broadcast:", err)
	return err
}
```
   针对上文的bc.onConnect(bc)方法，其实是自定义的broadcastSetup()方法去调用requester.RequesterBlock()。requester.RequestBlocksO方法首先调用ledgerInfoProvider.LedgerHeight(O方法，获取Peer节点(如Leader主节点)本地的区块账本高度height。如果区块账本高度height大于0，则调用区块请求者的b.seekLatestFromCommitter(height)方法，否则，调用b.seekOldest()方法，利用封装的Deliver服务客户端向Orderer 节点发送请求，以获取指定区块范围内的区块数据。此源码在core/deliverservice/requester.go  
```
func (b *blocksRequester) RequestBlocks(ledgerInfoProvider blocksprovider.LedgerInfo) error {
	height, err := ledgerInfoProvider.LedgerHeight()
	if err != nil {
		logger.Errorf("Can't get ledger height for channel %s from committer [%s]", b.chainID, err)
		return err
	}

	if height > 0 {
		logger.Debugf("Starting deliver with block [%d] for channel %s", height, b.chainID)
		if err := b.seekLatestFromCommitter(height); err != nil {
			return err
		}
	} else {
		logger.Debugf("Starting deliver with oldest block for channel %s", b.chainID)
		if err := b.seekOldest(); err != nil {
			return err
		}
	}

	return nil
}
```  
   b.seekLatestFromCommitter(height)或b.seekOldest()方法首先构造指定区块请求范围的区块搜索信息对象( SeekInfo类型)。该对象指定起始区块号范围极大，几乎可以认为包括了Orderer 节点指定通道上的所有区块数据。如果Orderer节点没有找到指定的区块，则阻塞等待区块，直到指定区块提交成功后再回复给请求客户端。当Deliver服务处理句柄的deliverBlocks0方法将指定通道账本上的所有数据都发送完毕时，height,deliverBlocks()方法在调用nextBlock()-> cursor.Next()方法时会-直阻塞， 等待区块号为当前账本高度height的新区块提交账本后再发送给客户端，并更新height使其增1。如此循环处理，deliverBlocks() 方法会一直将最新创建提交的区块发送给请求客户端。接着，b.seekLatestFromCommitter(height)或b.seekOldest()创建新的签名请求消息, 然后，通过区块请求者的broadcastClient客户端调用b.client.Send()方法，利用底层的Deliver服务客户端通信流发送区块请求消息将该消息发送给Orderer节点。
```
func (b *blocksRequester) seekOldest() error {
	seekInfo := &orderer.SeekInfo{
		Start:    &orderer.SeekPosition{Type: &orderer.SeekPosition_Oldest{Oldest: &orderer.SeekOldest{}}},
		Stop:     &orderer.SeekPosition{Type: &orderer.SeekPosition_Specified{Specified: &orderer.SeekSpecified{Number: math.MaxUint64}}},
		Behavior: orderer.SeekInfo_BLOCK_UNTIL_READY,
	}

	//TODO- epoch and msgVersion may need to be obtained for nowfollowing usage in orderer/configupdate/configupdate.go
	msgVersion := int32(0)
	epoch := uint64(0)
	tlsCertHash := b.getTLSCertHash()
	env, err := utils.CreateSignedEnvelopeWithTLSBinding(common.HeaderType_DELIVER_SEEK_INFO, b.chainID, localmsp.NewSigner(), seekInfo, msgVersion, epoch, tlsCertHash)
	if err != nil {
		return err
	}
	return b.client.Send(env)
}

func (b *blocksRequester) seekLatestFromCommitter(height uint64) error {
	seekInfo := &orderer.SeekInfo{
		Start:    &orderer.SeekPosition{Type: &orderer.SeekPosition_Specified{Specified: &orderer.SeekSpecified{Number: height}}},
		Stop:     &orderer.SeekPosition{Type: &orderer.SeekPosition_Specified{Specified: &orderer.SeekSpecified{Number: math.MaxUint64}}},
		Behavior: orderer.SeekInfo_BLOCK_UNTIL_READY,
	}

	//TODO- epoch and msgVersion may need to be obtained for nowfollowing usage in orderer/configupdate/configupdate.go
	msgVersion := int32(0)
	epoch := uint64(0)
	tlsCertHash := b.getTLSCertHash()
	env, err := utils.CreateSignedEnvelopeWithTLSBinding(common.HeaderType_DELIVER_SEEK_INFO, b.chainID, localmsp.NewSigner(), seekInfo, msgVersion, epoch, tlsCertHash)
	if err != nil {
		return err
	}
	return b.client.Send(env)
}
```
### 处理Deliver服务相应消息 
   DeliverBlock()方法用于处理上文提到的Orderer节点返回的Deliver服务响应类型。其中DeliverResponse_Status类型用于描述Deliver服务请求的执行状态。DeliverResponse_Block类型包含了请求获取的区块数据。接着，DeliverBlocks0()方法调用createPayload(seqNum, marshaledBlock)方法，创建Gossip消息负载payload。该对象封装了区块号seqNum(SeqNum字段)与区块字节数组marshaledBlock (Data字段),未指定PrivateData字段上的隐私数据明文读写集(nil)。
   然后，DeliverBlocks方法调用createGossipMsg()方法，基于payload消息负载创建DataMsg类型的数据消息gossipMsg,接着调用b.gossip.AddPayload()方法，将payload消息负载(含有区块数据)添加到本地的消息负载缓冲区中,等待Comitter记账节点验证处理并提交到账本中。
   最后，DeliverBlocksO()方法调用Gossip服务器实例的b.gossip.Gossip(gossipMsg)方法，基于Gossip消息协议将DataMsg类型数据消息(只含有区块数据)分发到组织内的其他Peer节点上,并保存到该节点的消息存储器上。该源码在core/deliverservice/blocksprovider/blocksprovider.go   
```
func (b *blocksProviderImpl) DeliverBlocks() {
	errorStatusCounter := 0
	statusCounter := 0
	defer b.client.Close()
	for !b.isDone() {
		msg, err := b.client.Recv()
		if err != nil {
			logger.Warningf("[%s] Receive error: %s", b.chainID, err.Error())
			return
		}
		switch t := msg.Type.(type) {
		case *orderer.DeliverResponse_Status:
			if t.Status == common.Status_SUCCESS {
				logger.Warningf("[%s] ERROR! Received success for a seek that should never complete", b.chainID)
				return
			}
			if t.Status == common.Status_BAD_REQUEST || t.Status == common.Status_FORBIDDEN {
				logger.Errorf("[%s] Got error %v", b.chainID, t)
				errorStatusCounter++
				if errorStatusCounter > b.wrongStatusThreshold {
					logger.Criticalf("[%s] Wrong statuses threshold passed, stopping block provider", b.chainID)
					return
				}
			} else {
				errorStatusCounter = 0
				logger.Warningf("[%s] Got error %v", b.chainID, t)
			}
			maxDelay := float64(maxRetryDelay)
			currDelay := float64(time.Duration(math.Pow(2, float64(statusCounter))) * 100 * time.Millisecond)
			time.Sleep(time.Duration(math.Min(maxDelay, currDelay)))
			if currDelay < maxDelay {
				statusCounter++
			}
			if t.Status == common.Status_BAD_REQUEST {
				b.client.Disconnect(false)
			} else {
				b.client.Disconnect(true)
			}
			continue
		case *orderer.DeliverResponse_Block:
			errorStatusCounter = 0
			statusCounter = 0
			blockNum := t.Block.Header.Number

			marshaledBlock, err := proto.Marshal(t.Block)
			if err != nil {
				logger.Errorf("[%s] Error serializing block with sequence number %d, due to %s", b.chainID, blockNum, err)
				continue
			}
			if err := b.mcs.VerifyBlock(gossipcommon.ChainID(b.chainID), blockNum, marshaledBlock); err != nil {
				logger.Errorf("[%s] Error verifying block with sequnce number %d, due to %s", b.chainID, blockNum, err)
				continue
			}

			numberOfPeers := len(b.gossip.PeersOfChannel(gossipcommon.ChainID(b.chainID)))
			// Create payload with a block received
			payload := createPayload(blockNum, marshaledBlock)
			// Use payload to create gossip message
			gossipMsg := createGossipMsg(b.chainID, payload)

			logger.Debugf("[%s] Adding payload to local buffer, blockNum = [%d]", b.chainID, blockNum)
			// Add payload to local state payloads buffer
			if err := b.gossip.AddPayload(b.chainID, payload); err != nil {
				logger.Warningf("Block [%d] received from ordering service wasn't added to payload buffer: %v", blockNum, err)
			}

			// Gossip messages with other nodes
			logger.Debugf("[%s] Gossiping block [%d], peers number [%d]", b.chainID, blockNum, numberOfPeers)
			if !b.isDone() {
				b.gossip.Gossip(gossipMsg)
			}
		default:
			logger.Warningf("[%s] Received unknown: %v", b.chainID, t)
			return
		}
	}
}
```
