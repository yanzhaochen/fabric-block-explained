### Gossip介绍
```Gossip```消息模块为```Peer```节点提供了安全、可靠、可扩展的P2P数据分发协议，用于广播数据与同步状态数据，以确保通道内所有Peer节点上的账本数据一致性。同时，周期性的同步更新节点存活信息、节点成员关系信息、节点身份信息、数据信息（区块数据和隐私数据）、Leader主节点选举信息等。
### 创建Gossip消息服务器
```Peer```启动时在serve()函数中调用了```initGossipService()```方法，```initGossipService()```方法会调用```service.InitGossipService()```方法，创建全局的```Gossip```服务器实例```gossipServiceInstance```(```service.gossipServiceImpl```类型)。
```go
// peer/node/start.go
func serve(args []string) error {
    ...
    // Initialize gossip component
	err = initGossipService(policyMgr, metricsProvider, peerServer, serializedIdentity, peerEndpoint.Address)
	if err != nil {
		return err
	}
	defer service.GetGossipService().Stop() // 退出时延迟终止服务器
    ...
}

// initGossipService will initialize the gossip service by:
// 1. Enable TLS if configured;
// 2. Init the message crypto service;
// 3. Init the security advisor;
// 4. Init gossip related struct.
func initGossipService(policyMgr policies.ChannelPolicyManagerGetter, metricsProvider metrics.Provider,
	peerServer *comm.GRPCServer, serializedIdentity []byte, peerAddr string) error {
	var certs *gossipcommon.TLSCertificates
	if peerServer.TLSEnabled() {
		serverCert := peerServer.ServerCertificate()
		clientCert, err := peer.GetClientCertificate()
		if err != nil {
			return errors.Wrap(err, "failed obtaining client certificates")
		}
		certs = &gossipcommon.TLSCertificates{}
		certs.TLSServerCert.Store(&serverCert)
		certs.TLSClientCert.Store(&clientCert)
	}

	messageCryptoService := peergossip.NewMCS(
		policyMgr,
		localmsp.NewSigner(),
		mgmt.NewDeserializersManager(),
	)
	secAdv := peergossip.NewSecurityAdvisor(mgmt.NewDeserializersManager())
	bootstrap := viper.GetStringSlice("peer.gossip.bootstrap")

	return service.InitGossipService(
		serializedIdentity,
		metricsProvider,
		peerAddr,
		peerServer.Server(),
		certs,
		messageCryptoService,
		secAdv,
		secureDialOpts,
		bootstrap...,
	)
}

```
```service.InitGossipService()```接着会调用```InitGossipServiceCustomDeliveryFactory()```方法，```InitGossipServiceCustomDeliveryFactory()```方法首先会调用```integration.NewGossipComponent()```方法构造```Gossip```服务实例对象```gossipInstance```（```gossip.gossipServiceImpl```类型），再基于```gossip```对象创建全局的```Gossip```服务器实例```gossipServiceInstance```(```service.gossipServiceImpl```类型)。
```go
// gossip/service/gossip_service.go

// InitGossipService initialize gossip service
func InitGossipService(
	peerIdentity []byte,
	metricsProvider metrics.Provider,
	endpoint string, s *grpc.Server,
	certs *gossipCommon.TLSCertificates,
	mcs api.MessageCryptoService,
	secAdv api.SecurityAdvisor,
	secureDialOpts api.PeerSecureDialOpts,
	bootPeers ...string,
) error {
	// TODO: Remove this.
	// TODO: This is a temporary work-around to make the gossip leader election module load its logger at startup
	// TODO: in order for the flogging package to register this logger in time so it can set the log levels as requested in the config
	util.GetLogger(util.ElectionLogger, "")
	return InitGossipServiceCustomDeliveryFactory(
		peerIdentity,
		metricsProvider,
		endpoint,
		s,
		certs,
		&deliveryFactoryImpl{},
		mcs,
		secAdv,
		secureDialOpts,
		bootPeers...,
	)
}

// InitGossipServiceCustomDeliveryFactory initialize gossip service with customize delivery factory
// implementation, might be useful for testing and mocking purposes
func InitGossipServiceCustomDeliveryFactory(
	peerIdentity []byte,
	metricsProvider metrics.Provider,
	endpoint string,
	s *grpc.Server,
	certs *gossipCommon.TLSCertificates,
	factory DeliveryServiceFactory,
	mcs api.MessageCryptoService,
	secAdv api.SecurityAdvisor,
	secureDialOpts api.PeerSecureDialOpts,
	bootPeers ...string,
) error {
	...
		gossip, err = integration.NewGossipComponent(peerIdentity, endpoint, s, secAdv,
			mcs, secureDialOpts, certs, gossipMetrics, bootPeers...)
		gossipServiceInstance = &gossipServiceImpl{
			mcs:             mcs,
			gossipSvc:       gossip,
			privateHandlers: make(map[string]privateHandler),
			chains:          make(map[string]state.GossipStateProvider),
			leaderElection:  make(map[string]election.LeaderElectionService),
			deliveryService: make(map[string]deliverclient.DeliverService),
			deliveryFactory: factory,
			peerIdentity:    peerIdentity,
			secAdv:          secAdv,
			metrics:         gossipMetrics,
		}
	})
	...
}
```
```integration.NewGossipComponent()```方法构造了```Gossip```服务实例对象```gossipInstance```（```gossip.gossipServiceImpl```类型），
```go
// gossip/integration/integration.go

// NewGossipComponent creates a gossip component that attaches itself to the given gRPC server
func NewGossipComponent(peerIdentity []byte, endpoint string, s *grpc.Server,
	secAdv api.SecurityAdvisor, cryptSvc api.MessageCryptoService,
	secureDialOpts api.PeerSecureDialOpts, certs *common.TLSCertificates, gossipMetrics *metrics.GossipMetrics,
	bootPeers ...string) (gossip.Gossip, error) {

	...
	gossipInstance := gossip.NewGossipService(conf, s, secAdv, cryptSvc,
		peerIdentity, secureDialOpts, gossipMetrics)

	return gossipInstance, nil
}
```
```gossip.NewGossipService()```函数初始化```Gossip```服务实例（```gossip.gossipServiceImpl```类型），创建```Gossip```服务模块(包括后文提到的通道消息处理模块```chanstate```)提供服务，并启用```goroutine```等服务程序。
```go
// gossip/gossip/gossip_impl.go

// NewGossipService creates a gossip instance attached to a gRPC server
func NewGossipService(conf *Config, s *grpc.Server, sa api.SecurityAdvisor,
	mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts, gossipMetrics *metrics.GossipMetrics) Gossip {
	var err error

	lgr := util.GetLogger(util.GossipLogger, conf.ID)

	g := &gossipServiceImpl{
		selfOrg:               sa.OrgByPeerIdentity(selfIdentity),
		secAdvisor:            sa,
		selfIdentity:          selfIdentity,
		presumedDead:          make(chan common.PKIidType, presumedDeadChanSize),
		disc:                  nil,
		mcs:                   mcs,
		conf:                  conf,
		ChannelDeMultiplexer:  comm.NewChannelDemultiplexer(),
		logger:                lgr,
		toDieChan:             make(chan struct{}, 1),
		stopFlag:              int32(0),
		stopSignal:            &sync.WaitGroup{},
		includeIdentityPeriod: time.Now().Add(conf.PublishCertPeriod),
		gossipMetrics:         gossipMetrics,
	}
	g.stateInfoMsgStore = g.newStateInfoMsgStore()

	g.idMapper = identity.NewIdentityMapper(mcs, selfIdentity, func(pkiID common.PKIidType, identity api.PeerIdentityType) {
		g.comm.CloseConn(&comm.RemotePeer{PKIID: pkiID})
		g.certPuller.Remove(string(pkiID))
	}, sa)

	commConfig := comm.CommConfig{
		DialTimeout:  conf.DialTimeout,
		ConnTimeout:  conf.ConnTimeout,
		RecvBuffSize: conf.RecvBuffSize,
		SendBuffSize: conf.SendBuffSize,
	}
	g.comm, err = comm.NewCommInstance(s, conf.TLSCerts, g.idMapper, selfIdentity, secureDialOpts, sa,
		gossipMetrics.CommMetrics, commConfig)

	if err != nil {
		lgr.Error("Failed instntiating communication layer:", err)
		return nil
	}

	g.chanState = newChannelState(g)
	g.emitter = newBatchingEmitter(conf.PropagateIterations,
		conf.MaxPropagationBurstSize, conf.MaxPropagationBurstLatency,
		g.sendGossipBatch)

	g.discAdapter = g.newDiscoveryAdapter()
	g.disSecAdap = g.newDiscoverySecurityAdapter()

	discoveryConfig := discovery.DiscoveryConfig{
		AliveTimeInterval:            conf.AliveTimeInterval,
		AliveExpirationTimeout:       conf.AliveExpirationTimeout,
		AliveExpirationCheckInterval: conf.AliveExpirationCheckInterval,
		ReconnectInterval:            conf.ReconnectInterval,
		BootstrapPeers:               conf.BootstrapPeers,
	}
	g.disc = discovery.NewDiscoveryService(g.selfNetworkMember(), g.discAdapter, g.disSecAdap, g.disclosurePolicy,
		discoveryConfig)
	g.logger.Infof("Creating gossip service with self membership of %s", g.selfNetworkMember())

	g.certPuller = g.createCertStorePuller()
	g.certStore = newCertStore(g.certPuller, g.idMapper, selfIdentity, mcs)

	if g.conf.ExternalEndpoint == "" {
		g.logger.Warning("External endpoint is empty, peer will not be accessible outside of its organization")
	}
	// Adding delta for handlePresumedDead and
	// acceptMessages goRoutines to block on Wait
	g.stopSignal.Add(2)
	go g.start()               // 启动gossip服务实例（3个goroutine：go g.syncDiscovery(),go g.HandlePresumedDead(),go g.acceptMessage(incMsgs)）
	go g.connect2BootstrapPeers() 

	return g
}
```

### 通道状态更新同步
```Gossip```服务实例的```chanState```模块在创建新通道对象```GossipChannel```时(在```peer/node/start.go```中的```serve()```函数会调用```peer.Initialize```来创建或加入已有通道)会执行两个```gc.periodicalInvocation()```分别周期性执行```gc.publishStateInfo()```发送通道的```StateInfo```类型的状态信息消息到其他节点以及执行```gc.requestStateInfo()```请求其他节点上的通道```StateInfo```消息。
```go
// gossip/channel/ channel.go

// NewGossipChannel creates a new GossipChannel
func NewGossipChannel(pkiID common.PKIidType, org api.OrgIdentityType, mcs api.MessageCryptoService,
	chainID common.ChainID, adapter Adapter, joinMsg api.JoinChannelMessage,
	metrics *metrics.MembershipMetrics) GossipChannel {
    ...
	// Periodically publish state info
	go gc.periodicalInvocation(gc.publishStateInfo, gc.stateInfoPublishScheduler.C)
	// Periodically request state info
	go gc.periodicalInvocation(gc.requestStateInfo, gc.stateInfoRequestScheduler.C)
    ...
}
```
```StateInfo```消息包含了节点账本的区块链信息,其他节点的```StateInfo```消息都被缓存到本地节点上指定通道的```GossipChannel```通道对象中，并保存在```stateInfoMsgStore```消息存储对象中，提供给本地节点用于获取通道上其他节点的最新账本高度，计算账本区块链的高度差以判断其与其他节点的账本数据差异，从而标识出本地账本的缺失数据（包括区块数据和隐私数据）范围，接着通过反熵算法的数据同步机制从其他节点拉取数据，并更新到本地账本。```StateInfo```消息结构如下：
```go
// proto/gossip/message.pb.go
// StateInfo is used for a peer to relay its state information
// to other peers
type StateInfo struct {
	Timestamp *PeerTime `protobuf:"bytes,2,opt,name=timestamp,proto3" json:"timestamp,omitempty"`
	PkiId     []byte    `protobuf:"bytes,3,opt,name=pki_id,json=pkiId,proto3" json:"pki_id,omitempty"`
	// channel_MAC is an authentication code that proves
	// that the peer that sent this message knows
	// the name of the channel.
	Channel_MAC          []byte      `protobuf:"bytes,4,opt,name=channel_MAC,json=channelMAC,proto3" json:"channel_MAC,omitempty"`
	Properties           *Properties `protobuf:"bytes,5,opt,name=properties,proto3" json:"properties,omitempty"`
	XXX_NoUnkeyedLiteral struct{}    `json:"-"`
	XXX_unrecognized     []byte      `json:"-"`
	XXX_sizecache        int32       `json:"-"`
}

type Properties struct {
	LedgerHeight         uint64       `protobuf:"varint,1,opt,name=ledger_height,json=ledgerHeight,proto3" json:"ledger_height,omitempty"`
	LeftChannel          bool         `protobuf:"varint,2,opt,name=left_channel,json=leftChannel,proto3" json:"left_channel,omitempty"`
	// fabric1.2新增，用于保存当前通道中已经实例化的链码信息集合
	Chaincodes           []*Chaincode `protobuf:"bytes,3,rep,name=chaincodes,proto3" json:"chaincodes,omitempty"` 
	XXX_NoUnkeyedLiteral struct{}     `json:"-"`
	XXX_unrecognized     []byte       `json:"-"`
	XXX_sizecache        int32        `json:"-"`
}

```
#### 1.发布状态信息
```gc.publishStateInfo```周期性地检查```shouldGossipStateInfo```标志位，如果```shouldGossipStateInfo```标志位是1，则调用```gc.Gossip()```方法，获取本地GossipChannel通道对象上的```gc.stateInfoMsg```消息，也就是本通道的```StateInfo```类型状态信息消息，并通过```emitter```模块发送给其他节点，否则，不执行任何操作。如果```GossipChannel```通道对象上还存在组织内的其他合法节点，则重新设置```shouldGossipStateInfo```标志位为0。其他节点接收到```StateInfo```消息后，交由```Gossip```服务实例的```handleMessage()```方法处理，检查通过后保存到本地```GossipChannel```通道的```StateInfoMsgStore```消息存储对象中。
```go
// gossip/channel/channel.go
func (gc *gossipChannel) publishStateInfo() {
	if atomic.LoadInt32(&gc.shouldGossipStateInfo) == int32(0) {
		return
	}
	gc.RLock()
	stateInfoMsg := gc.stateInfoMsg        // 获取通道状态信息消息
	gc.RUnlock()
	gc.Gossip(stateInfoMsg)					// 发送状态信息消息
	if len(gc.GetMembership()) > 0 {		// 检查是否存在组织内的存活节点成员
		atomic.StoreInt32(&gc.shouldGossipStateInfo, int32(0))
	}
}
```
#### 2.请求通道状态信息
```gc.requestStateInfo()```方法调用```gc.createStateInfoRequest()```方法，创建```StateInfoPullRequest```类型的状态信息请求消息，再调用```filter.SelectPeers()```方法，从组织内的合法节点中选择3个节点（若不足3个选择则选择全部）发送请求消息。
```go
// gossip/channel/channel.go
func (gc *gossipChannel) requestStateInfo() {
	req, err := gc.createStateInfoRequest() //创建状态信息请求消息
	if err != nil {
		gc.logger.Warningf("Failed creating SignedGossipMessage: %+v", errors.WithStack(err))
		return
	}
	endpoints := filter.SelectPeers(gc.GetConf().PullPeerNum, gc.GetMembership(), gc.IsMemberInChan)  // 获取发送节点列表
	gc.Send(req, endpoints...) // 发送请求消息到指定节点
}
```
远程节点接收消息后过滤转发给```GossipChannel```通道对象的```HandleMessage()```方法进行处理(这一步出现在```gossip```服务实例启动文件```gossip/integration/integration.go```的```gossip.NewGossipService()```方法里最后的```g.start()```里)，验证消息合法性后，调用```gc.createStateInfoSnapshot(orgID)```方法，创建```StateInfoSnapshot```类型的状态信息快照消息，最后，调用```msg.Respond()```方法对消息签名并回复给请求节点。
```go
// gossip/channel/channel.go
// HandleMessage processes a message sent by a remote peer
func (gc *gossipChannel) HandleMessage(msg proto.ReceivedMessage) {
	if !gc.verifyMsg(msg) {
		gc.logger.Warning("Failed verifying message:", msg.GetGossipMessage().GossipMessage)
		return
	}
	m := msg.GetGossipMessage()
	if !m.IsChannelRestricted() {
		gc.logger.Warning("Got message", msg.GetGossipMessage(), "but it's not a per-channel message, discarding it")
		return
	}
	orgID := gc.GetOrgOfPeer(msg.GetConnectionInfo().ID)
	if len(orgID) == 0 {
		gc.logger.Debug("Couldn't find org identity of peer", msg.GetConnectionInfo())
		return
	}
	if !gc.IsOrgInChannel(orgID) {
		gc.logger.Warning("Point to point message came from", msg.GetConnectionInfo(),
			", org(", string(orgID), ") but it's not eligible for the channel", string(gc.chainID))
		return
	}
	if m.IsStateInfoPullRequestMsg() {
		msg.Respond(gc.createStateInfoSnapshot(orgID)) 
		return
	}
	...
}
```
其中```gc.createStateInfoSnapshot(orgID)```方法调用```gc.stateInfoMsgStore.Get()```方法，获取```GossipChannel```通道对象上```stateInfoMsgStore```对象所保存的```StateInfo```消息列表。接着，遍历该列表中的每个消息，筛选出符合要求的```StateInfo```消息，将其```Envelope```字段都对象添加到```elements```消息列表（类型为[]*proto.Envelope），并封装为```StateInfoSnapshot```消息。
```go
// gossip/channel/channel.go
func (gc *gossipChannel) createStateInfoSnapshot(requestersOrg api.OrgIdentityType) *proto.GossipMessage {
	sameOrg := bytes.Equal(gc.selfOrg, requestersOrg)
	rawElements := gc.stateInfoMsgStore.Get() // 获取StateInfo消息列表
	elements := []*proto.Envelope{}
	for _, rawEl := range rawElements {
		msg := rawEl.(*proto.SignedGossipMessage)
		orgOfCurrentMsg := gc.GetOrgOfPeer(msg.GetStateInfo().PkiId)
		// If we're in the same org as the requester, or the message belongs to a foreign org
		// don't do any filtering
		if sameOrg || !bytes.Equal(orgOfCurrentMsg, gc.selfOrg) {
			elements = append(elements, msg.Envelope)
			continue
		}
		// Else, the requester is in a different org, so disclose only StateInfo messages that their
		// corresponding AliveMessages have external endpoints
		if netMember := gc.Lookup(msg.GetStateInfo().PkiId); netMember == nil || netMember.Endpoint == "" {
			continue
		}
		elements = append(elements, msg.Envelope)
	}

	return &proto.GossipMessage{
		Channel: gc.chainID,
		Tag:     proto.GossipMessage_CHAN_OR_ORG,
		Nonce:   0,
		Content: &proto.GossipMessage_StateSnapshot{
			StateSnapshot: &proto.StateInfoSnapshot{ //返回StateInfoSnapshot类型的状态信息快照消息
				Elements: elements,
			},
		},
	}
}
```
### 通道数据更新同步

```chanState```模块的```GossipChannel```通道对象通过```gc.createBlockPuller()```方法调用```pull.NewPullMediator(conf, adapter)```创建```blocksPuller```对象。
```go
// gossip/gossip/chanstate.go
// NewGossipChannel creates a new GossipChannel
func NewGossipChannel(pkiID common.PKIidType, org api.OrgIdentityType, mcs api.MessageCryptoService,
	chainID common.ChainID, adapter Adapter, joinMsg api.JoinChannelMessage,
	metrics *metrics.MembershipMetrics) GossipChannel {
    ...
	gc.blocksPuller = gc.createBlockPuller()
    ...
}
```

```go
// gossip/gossip/channel/channel.go

func (gc *gossipChannel) createBlockPuller() pull.Mediator {
	conf := pull.Config{
		MsgType:           proto.PullMsgType_BLOCK_MSG,
		Channel:           []byte(gc.chainID),
		ID:                gc.GetConf().ID,
		PeerCountToSelect: gc.GetConf().PullPeerNum,
		PullInterval:      gc.GetConf().PullInterval,
		Tag:               proto.GossipMessage_CHAN_AND_ORG,
		PullEngineConfig: algo.PullEngineConfig{
			DigestWaitTime:   gc.GetConf().DigestWaitTime,
			RequestWaitTime:  gc.GetConf().RequestWaitTime,
			ResponseWaitTime: gc.GetConf().ResponseWaitTime,
		},
	}
	seqNumFromMsg := func(msg *proto.SignedGossipMessage) string {
		dataMsg := msg.GetDataMsg()
		if dataMsg == nil || dataMsg.Payload == nil {
			gc.logger.Warning("Non-data block or with no payload")
			return ""
		}
		return fmt.Sprintf("%d", dataMsg.Payload.SeqNum)
	}
	adapter := &pull.PullAdapter{
		Sndr:        gc,
		MemSvc:      gc.memFilter,
		IdExtractor: seqNumFromMsg,
		MsgCons: func(msg *proto.SignedGossipMessage) {
			gc.DeMultiplex(msg)
		},
	}

	adapter.IngressDigFilter = func(digestMsg *proto.DataDigest) *proto.DataDigest {
		gc.RLock()
		height := gc.ledgerHeight
		gc.RUnlock()
		digests := digestMsg.Digests
		digestMsg.Digests = nil
		for i := range digests {
			seqNum, err := strconv.ParseUint(string(digests[i]), 10, 64)
			if err != nil {
				gc.logger.Warningf("Can't parse digest %s : %+v", digests[i], errors.WithStack(err))
				continue
			}
			if seqNum >= height {
				digestMsg.Digests = append(digestMsg.Digests, digests[i])
			}

		}
		return digestMsg
	}

	return pull.NewPullMediator(conf, adapter)
}
```
```blocksPuller```模块利用```goroutine```创建消息处理循环，隔一段时间就调用一次```engine.initialPull()```方法，循环启动```Pull```类消息的交互进程，选择特定节点发送```Hello```消息以请求获取```DataMsg```消息。
```go 
// gossip/gossip/algo/pull.go

// NewPullEngineWithFilter creates an instance of a PullEngine with a certain sleep time
// between pull initiations, and uses the given filters when sending digests and responses
func NewPullEngineWithFilter(participant PullAdapter, sleepTime time.Duration, df DigestFilter,
	config PullEngineConfig) *PullEngine {
	engine := &PullEngine{
		PullAdapter:        participant,
		stopFlag:           int32(0),
		state:              util.NewSet(),
		item2owners:        make(map[string][]string),
		peers2nonces:       make(map[string]uint64),
		nonces2peers:       make(map[uint64]string),
		acceptingDigests:   int32(0),
		acceptingResponses: int32(0),
		incomingNONCES:     util.NewSet(),
		outgoingNONCES:     util.NewSet(),
		digFilter:          df,
		digestWaitTime:     config.DigestWaitTime,
		requestWaitTime:    config.RequestWaitTime,
		responseWaitTime:   config.ResponseWaitTime,
	}

	go func() {
		for !engine.toDie() {
			time.Sleep(sleepTime)
			if engine.toDie() {
				return
			}
			engine.initiatePull() 
		}
	}()

	return engine
}

func (engine *PullEngine) initiatePull() {
	engine.lock.Lock()
	defer engine.lock.Unlock()

	engine.acceptDigests()
	for _, peer := range engine.SelectPeers() {
		nonce := engine.newNONCE()
		engine.outgoingNONCES.Add(nonce)
		engine.nonces2peers[nonce] = peer
		engine.peers2nonces[peer] = nonce
		engine.Hello(peer, nonce)
	}

	time.AfterFunc(engine.digestWaitTime, func() {
		engine.processIncomingDigests()
	})
}

```
同时，其他节点将收到的```Pull```类数据消息过滤处理后交给```GossipChannel```通道对象的```HandleMessage()```方法处理

```go
// gossip/gossip/gossip_impl.go

func (g *gossipServiceImpl) handleMessage(m proto.ReceivedMessage) {
	...
	msg := m.GetGossipMessage()
	...
	if msg.IsChannelRestricted() {  //检查是否为通道内传播消息
		...
		} else {
			...
			gc.HandleMessage(m) //调用GossipChannel通道对象的HandleMessage()方法处理
		}
		return
	}
}
```

检查消息的合法性后调用```blocksPuller```模块的```HandleMessage()```方法。
```go
// gossip/channel/chanel.go

// HandleMessage processes a message sent by a remote peer
func (gc *gossipChannel) HandleMessage(msg proto.ReceivedMessage) {
	if !gc.verifyMsg(msg) {
		gc.logger.Warning("Failed verifying message:", msg.GetGossipMessage().GossipMessage)
		return
	}
	m := msg.GetGossipMessage()
	if !m.IsChannelRestricted() {
		gc.logger.Warning("Got message", msg.GetGossipMessage(), "but it's not a per-channel message, discarding it")
		return
	}
	orgID := gc.GetOrgOfPeer(msg.GetConnectionInfo().ID)
	if len(orgID) == 0 {
		gc.logger.Debug("Couldn't find org identity of peer", msg.GetConnectionInfo())
		return
	}
	...

	if m.IsPullMsg() && m.GetPullMsgType() == proto.PullMsgType_BLOCK_MSG {
		if gc.hasLeftChannel() {
			gc.logger.Info("Received Pull message from", msg.GetConnectionInfo().Endpoint, "but left the channel", string(gc.chainID))
			return
		}
		// If we don't have a StateInfo message from the peer,
		// no way of validating its eligibility in the channel.
		if gc.stateInfoMsgStore.MsgByID(msg.GetConnectionInfo().ID) == nil {
			gc.logger.Debug("Don't have StateInfo message of peer", msg.GetConnectionInfo())
			return
		}
		if !gc.eligibleForChannelAndSameOrg(discovery.NetworkMember{PKIid: msg.GetConnectionInfo().ID}) {
			gc.logger.Warning(msg.GetConnectionInfo(), "isn't eligible for pulling blocks of", string(gc.chainID))
			return
		}
		if m.IsDataUpdate() { 
			// Iterate over the envelopes, and filter out blocks
			// that we already have in the blockMsgStore, or blocks that
			// are too far in the past.
			var msgs []*proto.SignedGossipMessage
			var items []*proto.Envelope
			filteredEnvelopes := []*proto.Envelope{}
			for _, item := range m.GetDataUpdate().Data {
				gMsg, err := item.ToGossipMessage()
				if err != nil {
					gc.logger.Warningf("Data update contains an invalid message: %+v", errors.WithStack(err))
					return
				}
				if !bytes.Equal(gMsg.Channel, []byte(gc.chainID)) {
					gc.logger.Warning("DataUpdate message contains item with channel", gMsg.Channel, "but should be", gc.chainID)
					return
				}
				// Would this block go into the message store if it was verified?
				if !gc.blockMsgStore.CheckValid(gMsg) {
					continue
				}
				if !gc.verifyBlock(gMsg.GossipMessage, msg.GetConnectionInfo().ID) {
					return
				}
				msgs = append(msgs, gMsg)
				items = append(items, item)
			}

			gc.Lock()
			defer gc.Unlock()

			for i, gMsg := range msgs {
				item := items[i]
				added := gc.blockMsgStore.Add(gMsg)
				if !added {
					// If this block doesn't need to be added, it means it either already
					// exists in memory or that it is too far in the past
					continue
				}
				filteredEnvelopes = append(filteredEnvelopes, item)
			}

			// Replace the update message with just the blocks that should be processed
			m.GetDataUpdate().Data = filteredEnvelopes
		}
		gc.blocksPuller.HandleMessage(msg)
	}

	...
}
```
处理```Hello```消息并继续交互，以确认本地节点所缺失的数据信息并打包到```DataUpdate```摘要更新消息，再回复给本地节点。接着，本地节点在```blocksPuller```模块的```HandleMessage()```方法中调用```p.MsgCons()```方法，以处理摘要更新信息。
```go
// gossip/gossip/pull/pullstore.go
func (p *pullMediatorImpl) HandleMessage(m proto.ReceivedMessage) {
	if m.GetGossipMessage() == nil || !m.GetGossipMessage().IsPullMsg() {
		return
	}
	msg := m.GetGossipMessage()
	msgType := msg.GetPullMsgType()
	if msgType != p.config.MsgType {
		return
	}

	p.logger.Debug(msg)

	itemIDs := []string{}
	items := []*proto.SignedGossipMessage{}
	var pullMsgType MsgType

	if helloMsg := msg.GetHello(); helloMsg != nil {
		pullMsgType = HelloMsgType
		p.engine.OnHello(helloMsg.Nonce, m)
	}
	if digest := msg.GetDataDig(); digest != nil {
		d := p.PullAdapter.IngressDigFilter(digest)
		itemIDs = util.BytesToStrings(d.Digests)
		pullMsgType = DigestMsgType
		p.engine.OnDigest(itemIDs, d.Nonce, m)
	}
	if req := msg.GetDataReq(); req != nil {
		itemIDs = util.BytesToStrings(req.Digests)
		pullMsgType = RequestMsgType
		p.engine.OnReq(itemIDs, req.Nonce, m)
	}
	if res := msg.GetDataUpdate(); res != nil {
		itemIDs = make([]string, len(res.Data))
		items = make([]*proto.SignedGossipMessage, len(res.Data))
		pullMsgType = ResponseMsgType
		for i, pulledMsg := range res.Data {
			msg, err := pulledMsg.ToGossipMessage()
			if err != nil {
				p.logger.Warningf("Data update contains an invalid message: %+v", errors.WithStack(err))
				return
			}
			p.MsgCons(msg)
			itemIDs[i] = p.IdExtractor(msg)
			items[i] = msg
			p.Lock()
			p.itemID2Msg[itemIDs[i]] = msg
			p.logger.Debugf("Added %s to the in memory item map, total items: %d", itemIDs[i], len(p.itemID2Msg))
			p.Unlock()
		}
		p.engine.OnRes(itemIDs, res.Nonce)
	}
	...
}
```
实际上是调用```gc.DeMultiplex(msg)```方法(在之前提到的```createBlockPuller()```里面进行构造的)，不过这里没有找到```DeMultiplex()```方法关于```gossipChannel```的具体实现。
```go
// gossip/gossip/channel/channel.go
func (gc *gossipChannel) createBlockPuller() pull.Mediator {
	...
	adapter := &pull.PullAdapter{
		Sndr:        gc,
		MemSvc:      gc.memFilter,
		IdExtractor: seqNumFromMsg,
		MsgCons: func(msg *proto.SignedGossipMessage) {
			gc.DeMultiplex(msg)
		},
	}
	...
```
接着过滤筛选出```DataMsg```类型的数据消息，并发送到```gossipChan```通道，交由```state```模块的```GossipStateProviderImpl.queueNewMessage()```方法处理，将其添加到本地的消息负载缓冲区，等待处理并提交到账本。注意，```DataMsg```消息包含区块数据消息与隐私数据，因此，会同时更新本地账本中的区块数据文件与隐私数据库。
```go
//gossip/state/state.go(goroutine：s.listen())

func (s *GossipStateProviderImpl) listen() {
	defer s.done.Done()

	for {
		select {
		case msg := <-s.gossipChan:
			logger.Debug("Received new message via gossip channel")
			go s.queueNewMessage(msg)
		case msg := <-s.commChan:
			logger.Debug("Dispatching a message", msg)
			go s.dispatch(msg)
		case <-s.stopCh:
			s.stopCh <- struct{}{}
			logger.Debug("Stop listening for new messages")
			return
		}
	}
}

// queueNewMessage makes new message notification/handler
func (s *GossipStateProviderImpl) queueNewMessage(msg *proto.GossipMessage) {
	if !bytes.Equal(msg.Channel, []byte(s.chainID)) {
		logger.Warning("Received enqueue for channel",
			string(msg.Channel), "while expecting channel", s.chainID, "ignoring enqueue")
		return
	}

	dataMsg := msg.GetDataMsg()
	if dataMsg != nil {
		if err := s.addPayload(dataMsg.GetPayload(), NonBlocking); err != nil {
			logger.Warningf("Block [%d] received from gossip wasn't added to payload buffer: %v", dataMsg.Payload.SeqNum, err)
			return
		}

	} else {
		logger.Debug("Gossip message received is not of data message type, usually this should not happen.")
	}
}

// addPayload adds new payload into state. It may (or may not) block according to the
// given parameter. If it gets a block while in blocking mode - it would wait until
// the block is sent into the payloads buffer.
// Else - it may drop the block, if the payload buffer is too full.
func (s *GossipStateProviderImpl) addPayload(payload *proto.Payload, blockingMode bool) error {
	if payload == nil {
		return errors.New("Given payload is nil")
	}
	logger.Debugf("[%s] Adding payload to local buffer, blockNum = [%d]", s.chainID, payload.SeqNum)
	height, err := s.ledger.LedgerHeight()
	if err != nil {
		return errors.Wrap(err, "Failed obtaining ledger height")
	}

	if !blockingMode && payload.SeqNum-height >= uint64(s.config.MaxBlockDistance) {
		return errors.Errorf("Ledger height is at %d, cannot enqueue block with sequence of %d", height, payload.SeqNum)
	}

	for blockingMode && s.payloads.Size() > s.config.MaxBlockDistance*2 {
		time.Sleep(enqueueRetryInterval)
	}

	s.payloads.Push(payload)
	logger.Debugf("Blocks payloads buffer size for channel [%s] is %d blocks", s.chainID, s.payloads.Size())
	return nil
}

```
最后，将```DataMsg```消息及其摘要信息（区块号）更新到```blocksPuller```模块中，包括```itemID2Msg```摘要消息列表与```engine.state```摘要存储对象。
```go
// gossip/gossip/pull/pullstore.go

func (p *pullMediatorImpl) HandleMessage(m proto.ReceivedMessage) {
	if m.GetGossipMessage() == nil || !m.GetGossipMessage().IsPullMsg() {
		return
	}
	msg := m.GetGossipMessage()
	msgType := msg.GetPullMsgType()
	if msgType != p.config.MsgType {
		return
	}

	p.logger.Debug(msg)

	itemIDs := []string{}
	items := []*proto.SignedGossipMessage{}
	var pullMsgType MsgType
	...
	if res := msg.GetDataUpdate(); res != nil {
		itemIDs = make([]string, len(res.Data))
		items = make([]*proto.SignedGossipMessage, len(res.Data))
		pullMsgType = ResponseMsgType
		for i, pulledMsg := range res.Data {
			msg, err := pulledMsg.ToGossipMessage()
			if err != nil {
				p.logger.Warningf("Data update contains an invalid message: %+v", errors.WithStack(err))
				return
			}
			p.MsgCons(msg)
			itemIDs[i] = p.IdExtractor(msg)
			items[i] = msg
			p.Lock()
			p.itemID2Msg[itemIDs[i]] = msg
			p.logger.Debugf("Added %s to the in memory item map, total items: %d", itemIDs[i], len(p.itemID2Msg))
			p.Unlock()
		}
		p.engine.OnRes(itemIDs, res.Nonce)
	}
	// Invoke hooks for relevant message type
	for _, h := range p.hooksByMsgType(pullMsgType) {
		h(itemIDs, items, m)
	}
}
```
### Gossip反熵算法
Peer节点在创建Gossip服务实例时的state模块时会执行反熵算法函数，负责周期性地检查本地节点与其他节点的最大账本高度差，以构造StateRequest类型的远程状态请求信息，并从其他节点拉取本地的缺失数据（区块数据与隐私数据）。如此重复执行上述数据请求操作，直到	账本高度差降为0。这样，所有节点就都能与Orderer节点上账本的区块数据以及Endorser背书节点上的隐私数据保持同步更新。
```go
// gossip/state/state.go

func (s *GossipStateProviderImpl) antiEntropy() {
	defer s.done.Done()
	defer logger.Debug("State Provider stopped, stopping anti entropy procedure.")

	for {
		select {
		case <-s.stopCh:
			s.stopCh <- struct{}{}
			return
		case <-time.After(s.config.AntiEntropyInterval):
			ourHeight, err := s.ledger.LedgerHeight()
			if err != nil {
				// Unable to read from ledger continue to the next round
				logger.Errorf("Cannot obtain ledger height, due to %+v", errors.WithStack(err))
				continue
			}
			if ourHeight == 0 {
				logger.Error("Ledger reported block height of 0 but this should be impossible")
				continue
			}
			maxHeight := s.maxAvailableLedgerHeight()
			if ourHeight >= maxHeight {
				continue
			}

			s.requestBlocksInRange(uint64(ourHeight), uint64(maxHeight)-1)
		}
	}
}

```