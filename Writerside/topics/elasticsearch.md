# Elasticsearch

https://blog.csdn.net/2401_84025139/article/details/137834859

在看模块的功能之前，先看下ES是如何做模块插件化和内部是如何路由调用module的

## 模块插件化
以NetworkModule为例，所有的Network模块会extends Plugin implements NetworkPlugin，在Node.java初始化的时候，把所有的NetworkPlugin实现传入到NetworkModule的构造方法
```Java
final NetworkModule networkModule = new NetworkModule(settings, false, pluginsService.filterPlugins(NetworkPlugin.class),
                threadPool, bigArrays, pageCacheRecycler, circuitBreakerService, namedWriteableRegistry, xContentRegistry,
```
然后以key-value的形式保存到map中

```Java
for (NetworkPlugin plugin : plugins) {
    // 获取插件中定义的HttpTransport实现
    Map<String, Supplier<HttpServerTransport>> httpTransportFactory = plugin.getHttpTransports(settings, threadPool, bigArrays,
        pageCacheRecycler, circuitBreakerService, xContentRegistry, networkService, dispatcher, clusterSettings);
    if (transportClient == false) {
        for (Map.Entry<String, Supplier<HttpServerTransport>> entry : httpTransportFactory.entrySet()) {
            registerHttpTransport(entry.getKey(), entry.getValue());
        }
    }
```

使用的时候，根据配置文件的指定对应的实现或者使用默认值

```Java
/**
 * 返回注册的HTTP传输模块
 */
public Supplier<HttpServerTransport> getHttpServerTransportSupplier() {
    final String name;
    // 配置中是否指定了实现
    if (HTTP_TYPE_SETTING.exists(settings)) {
        // 可以配置 nio....
        name = HTTP_TYPE_SETTING.get(settings);
    } else {
        // 默认netty4
        name = HTTP_DEFAULT_TYPE_SETTING.get(settings);
    }
```

## module的调用

```Java
void inboundMessage(TcpChannel channel, InboundMessage message){
    final long startTime = threadPool.relativeTimeInMillis();
    channel.getChannelStats().markAccessed(startTime);
    TransportLogger.logInboundMessage(channel, message);

    if (message.isPing()) {
        keepAlive.receiveKeepAlive(channel);
    } else {
        messageReceived(channel, message, startTime); // 接收到请求
    }
    
    
private void messageReceived(...) {
   。。。。。
   if (header.isRequest()) {
      handleRequest(channel, header, message);
   }
}

private <T extends TransportRequest> 
    void handleRequest(TcpChannel channel, Header header, InboundMessage message){
    final String action = header.getActionName();
    ....
    // 根据action的名字取拿对应的Handlers
    final RequestHandlerRegistry<T> reg = requestHandlers.getHandler(action);
```

那这些Handlers是哪里去注册的呢，主要是两个地方注册

1. ActionModule->setupActions(..)方法 会向guice IOC框架提供一组需要实例化的Transport*Action的类型, 该类型比较特殊,都是继承自HandledTransportAction类型，该类型的构造方法会负责将本handler注册到这里
2. 代码层使用TransportService->registerRequestHandler(...)方法强制向requestHandlers map中加入请求处理器

```Java
public MembershipAction(...) {
    this.transportService = transportService;
    this.listener = listener;

    transportService.registerRequestHandler(DISCOVERY_JOIN_ACTION_NAME,
       ThreadPool.Names.GENERIC, JoinRequest::new, new JoinRequestRequestHandler());
```
## 选举流程

ES 7.0之前默认用的是内置的ZenDiscovery，在Node.java中启动
```Java
discovery.start(); // start before cluster service so that it can set initial state on ClusterApplierService
discovery.startInitialJoin(); // 选主
```

discovery.start()调用，只是初始化一些默认值
```Java
protected void doStart() {
    // 获取表示本地节点信息的node对象
    DiscoveryNode localNode = transportService.getLocalNode();
    assert localNode != null;
    synchronized (stateMutex) {
        // set initial state
        assert committedState.get() == null;
        assert localNode != null;

        // 集群状态构造器
        ClusterState.Builder builder = ClusterState.builder(clusterName);
        ClusterState initialState = builder
            .blocks(ClusterBlocks.builder()
                .addGlobalBlock(STATE_NOT_RECOVERED_BLOCK) // 集群未恢复
                .addGlobalBlock(noMasterBlockService.getNoMasterBlock())) // 集群无主状态
            .nodes(DiscoveryNodes.builder().add(localNode).localNodeId(localNode.getId()))
            .build();

        // 将初始的集群信息写给committedState。因为committedState要始终保持最新的集群状态
        committedState.set(initialState);

        // clusterApplier内部封装了应用集群状态的逻辑
        // 内部注册了一些Listener，Listener收到newState之后，做什么事情，由它决定。
        clusterApplier.setInitialState(initialState);

        // nodesFD: 普通node节点join到master之后，启动，会定时1s向master发起ping，用于监控master存活
        // 这里设置localNode，是因为ping的request中包含了local信息
        nodesFD.setLocalNode(localNode);

        // joinThreadControl：该Control用于当前节点控制joinThread，避免本地同时有多个joinThread去工作。确保
        // 在需要joinThread工作的时候，仅仅只有一个该线程。内部封装了启动 joinThread 的逻辑。
        // 这一步的start()仅仅是设置running开关为true，并不会启动线程。
        joinThreadControl.start();
    }
    zenPing.start();
}
```

discovery.startInitialJoin()则是启动一个异步任务，放到generic线程池中

找不到不停地循环，如果选举失败，node会开启一个新的joinThread去顶替当前线程的工作
```Java
 while (masterNode == null && joinThreadControl.joinThreadActive(currentThread)) {
    masterNode = findMaster();
}
```

findMaster会向discovery.zen.ping.unicast.hosts配置项的节点发送ping请求，并且线程等待对端响应，返回的是PingResponse对象，包含对端节点和集群的信息
```Java
// 内部还封装了ping失败后，重试等等逻辑...
List<ZenPing.PingResponse> fullPingResponses = pingAndWait(pingTimeout).toList();

class PingResponse implements Writeable {
    // 返回的PingResponse对象
    private final long id;
    // 对方节点的集群名
    private final ClusterName clusterName;
    // 对方节点的基本信息
    private final DiscoveryNode node;
    // 对方所在master的节点的基本信息
    private final DiscoveryNode master;
    // 集群状态版本号
    private final long clusterStateVersion;
    
// 本地节点加入到PingResponse
fullPingResponses.add(new ZenPing.PingResponse(localNode, null, this.clusterState()));

// 默认情况下，全部节点都是默认投票权，如果配置ignore_non_master_pings=true的配置的话，则会过滤掉非master权限
final List<ZenPing.PingResponse> pingResponses = filterPingResponses(fullPingResponses, masterElectionIgnoreNonMasters, logger);

// masterCandidates: 该列表保存所有可能成为master节点的信息
List<ElectMasterService.MasterCandidate> masterCandidates = new ArrayList<>();
for (ZenPing.PingResponse pingResponse : pingResponses) {
    if (pingResponse.node().isMasterNode()) {
        masterCandidates.add(new ElectMasterService.MasterCandidate(pingResponse.node(), pingResponse.getClusterStateVersion()));
    }
}

// CASE1: 当前集群无主
if (activeMasters.isEmpty()) {

    // 判断是否有足够数量的候选者
    if (electMaster.hasEnoughCandidates(masterCandidates)) {

        // 从候选者列表中，选举winner：对masterCandidates进行sort，先按照集群版本大小排序，大的靠前，如果
        // 集群版本号一致，则再按照 masterCandidate->node->id进行排序，小的靠前..
        // 排序完之后，选择 list.get(0)为胜出者
        final ElectMasterService.MasterCandidate winner = electMaster.electMaster(masterCandidates);
        logger.trace("candidate {} won election", winner);
        return winner.getNode();
    } else {
        // if we don't have enough master nodes, we bail, because there are not enough master to elect from
        logger.warn("not enough master nodes discovered during pinging (found [{}], but needed [{}]), pinging again",
                    masterCandidates, electMaster.minimumMasterNodes());
        return null;
    }
} else {
    // CASE2: 当前集群有主
    assert !activeMasters.contains(localNode) :
        "local node should never be elected as master when other nodes indicate an active master";
    // lets tie break between discovered nodes
    return electMaster.tieBreakActiveMasters(activeMasters);
}
```

