---
layout: post
title: 'Zeebe ClusterServicesStep源码之GroupMembershipProtocol(八)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述

### (2). AtomixCluster.buildMembershipProtocol
```
protected static GroupMembershipProtocol buildMembershipProtocol(final ClusterConfig config) {
    return config.getProtocolConfig().getType().newProtocol(config.getProtocolConfig());
}
```
### (3). SwimMembershipProtocol$Type.newProtocol
```
public class SwimMembershipProtocol
    extends AbstractListenerManager<GroupMembershipEvent, GroupMembershipEventListener>
    implements GroupMembershipProtocol {
		
	public static class Type implements GroupMembershipProtocol.Type<SwimMembershipProtocolConfig> {
		private static final String NAME = "swim";

		@Override
		public String name() {
		  return NAME;
		}

        // *****************************************************************
		// SwimMembershipProtocol$Type.newProtocol
		// *****************************************************************
		@Override
		public GroupMembershipProtocol newProtocol(final SwimMembershipProtocolConfig config) {
		  return new SwimMembershipProtocol(config);
		}
	  } // end SwimMembershipProtocol$Type
} 	  
```
### (4). 分析SwimMembershipProtocol类的成员属性
```
public class SwimMembershipProtocol
    extends AbstractListenerManager<GroupMembershipEvent, GroupMembershipEventListener>
    implements GroupMembershipProtocol {

  public static final Type TYPE = new Type();
  private static final Logger LOGGER = LoggerFactory.getLogger(SwimMembershipProtocol.class);
  
  private static final String MEMBERSHIP_SYNC = "atomix-membership-sync";
  private static final String MEMBERSHIP_GOSSIP = "atomix-membership-gossip";
  private static final String MEMBERSHIP_PROBE = "atomix-membership-probe";
  private static final String MEMBERSHIP_PROBE_REQUEST = "atomix-membership-probe-request";
  
  // 序列化
  private static final Serializer SERIALIZER =
      Serializer.using(
          new Namespace.Builder()
              .register(Namespaces.BASIC)
              .nextId(Namespaces.BEGIN_USER_CUSTOM_ID)
              .register(MemberId.class)
              .register(new AddressSerializer(), Address.class)
              .register(ImmutableMember.class)
              .register(State.class)
              .register(ImmutablePair.class)
              .name("ClusterMembershipService")
              .build());

  // 
  private final SwimMembershipProtocolConfig config;
  private final AtomicBoolean started = new AtomicBoolean();
  
  private final Map<MemberId, SwimMember> members = Maps.newConcurrentMap();
  private final List<SwimMember> randomMembers = Lists.newCopyOnWriteArrayList();
  private final Map<MemberId, ImmutableMember> updates = new LinkedHashMap<>();
  private final List<SwimMember> syncMembers = new ArrayList<>();
  
  private final ScheduledExecutorService swimScheduler =
      Executors.newSingleThreadScheduledExecutor(
          namedThreads("atomix-cluster-heartbeat-sender", LOGGER));
  private final ExecutorService eventExecutor =
      Executors.newSingleThreadExecutor(namedThreads("atomix-cluster-events", LOGGER));
  private final AtomicInteger probeCounter = new AtomicInteger();
  
  // *********************************************************************
  // NodeDiscoveryService底层委托给了:NodeDiscoveryProvider
  // NodeDiscoveryProvider的内容,上一篇有详细讲解
  // *********************************************************************
  private NodeDiscoveryService discoveryService;
  
  // *********************************************************************
  // BootstrapService实际是:AtomixClusterImpl
  // *********************************************************************
  private BootstrapService bootstrapService;
  private SwimMember localMember;
  
  // *********************************************************************
  // 节点发现事件监听器
  // *********************************************************************
  private final NodeDiscoveryEventListener discoveryEventListener = this::handleDiscoveryEvent;
  
  // *********************************************************************
  // 探测请求处理/同步/探测/goossip
  // *********************************************************************
  private final BiFunction<Address, byte[], CompletableFuture<byte[]>> probeRequestHandler =
      (address, payload) ->
          handleProbeRequest(SERIALIZER.decode(payload)).thenApply(SERIALIZER::encode);
  private final BiFunction<Address, byte[], byte[]> syncHandler =
      (address, payload) -> SERIALIZER.encode(handleSync(SERIALIZER.decode(payload)));
  private final BiFunction<Address, byte[], byte[]> probeHandler =
      (address, payload) -> SERIALIZER.encode(handleProbe(SERIALIZER.decode(payload)));
  private final BiConsumer<Address, byte[]> gossipListener =
      (address, payload) -> handleGossipUpdates(SERIALIZER.decode(payload));
  // ... ...
}  
```
### (5). SwimMembershipProtocol.handleDiscoveryEvent
```
private void handleDiscoveryEvent(final NodeDiscoveryEvent event) {
    switch (event.type()) {
      case JOIN:
	    // ************************************************************
		// 处理节点加入集群
		// ************************************************************
        handleJoinEvent(event.subject());
        break;
      case LEAVE:
	    // ************************************************************
		// 处理节点离开集群
	    // ************************************************************
        handleLeaveEvent(event.subject());
        break;
      default:
        throw new AssertionError();
    } // end switch
}
```
### (6). SwimMembershipProtocol.join
```
public CompletableFuture<Void> join(
      final BootstrapService bootstrap, final NodeDiscoveryService discovery, final Member member) {
    if (started.compareAndSet(false, true)) {
      bootstrapService = bootstrap;
      discoveryService = discovery;
	  
	  // 构建本地成员
      localMember =
          new SwimMember(
              member.id(),
              member.address(),
              member.zone(),
              member.rack(),
              member.host(),
              member.properties(),
              member.version(),
              System.currentTimeMillis());
      localProperties.putAll(localMember.properties());
	  
	  // 添加监听
      discoveryService.addListener(discoveryEventListener);

      // 设置本地成员状态为:激活
      localMember.setState(State.ALIVE);
	  
	  // 添加本地成员到成员列表集合里
      members.put(localMember.id(), localMember);
	  
	  // 触发成员添加事件
      post(new GroupMembershipEvent(GroupMembershipEvent.Type.MEMBER_ADDED, localMember));

      LOGGER.debug("Nodes from discovery service {}", discoveryService.getNodes());
	  
	  
      registerHandlers();
      
	  // **********************************************************
	  // 以下三个都是开启定时任务,发送TCP/UDP请求.更新成员属性信息.
	  // **********************************************************
      scheduleGossip();
      scheduleProbe();
      scheduleSync();

      LOGGER.info("Started");
    }
    return CompletableFuture.completedFuture(null);
} //end join


private void registerHandlers() {
	// ****************************************************
	// SWIM与Gossip还是有一些不一样,既用了TCP还用了UDP协议
	// ****************************************************
	// 为TCP配置Handler(成员通步/成员探测/成员探测请求)
    // Register TCP message handlers.
    bootstrapService
        .getMessagingService()
        .registerHandler(MEMBERSHIP_SYNC, syncHandler, swimScheduler);
    bootstrapService
        .getMessagingService()
        .registerHandler(MEMBERSHIP_PROBE, probeHandler, swimScheduler);
    bootstrapService
        .getMessagingService()
        .registerHandler(MEMBERSHIP_PROBE_REQUEST, probeRequestHandler);

    // 为UDP配置Handler(Gossip)
	// Register UDP message listeners.
    bootstrapService
        .getUnicastService()
        .addListener(MEMBERSHIP_GOSSIP, gossipListener, swimScheduler);
} // end registerHandlers
```
### (7). 总结
SwimMembershipProtocol的主要职责是:更新成员列表/根据成员ID获得成员详细信息/加入集群/离开集群.  