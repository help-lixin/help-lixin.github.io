---
layout: post
title: 'Atomix源码之服务发现之BootstrapDiscoveryProvider(二)' 
date: 2022-12-02
author: 李新
tags:  Atomix
---

### (1). 概述
前面对BootstrapDiscoveryBuilder源码分析,最终是调用:BootstrapDiscoveryBuilder.build方法,构建:BootstrapDiscoveryProvider,所以,这一篇主要对:BootstrapDiscoveryProvider进行剖析.  

### (2). 先看下BootstrapDiscoveryProvider的接口
```
public interface NodeDiscoveryProvider
    extends ListenerService<NodeDiscoveryEvent, NodeDiscoveryEventListener>,
    Configured<NodeDiscoveryConfig> {

    // 返回活跃的其它节点
	Set<Node> getNodes();

	// 本节点加入集群(集群其它成员在配置文件里)
	CompletableFuture<Void> join(BootstrapService bootstrap, Node localNode);

	
	// 本节点离开集群
	CompletableFuture<Void> leave(Node localNode);
}
```
### (3). BootstrapDiscoveryProvider构建器
```
BootstrapDiscoveryProvider(BootstrapDiscoveryConfig config) {
    // Hold住:BootstrapDiscoveryConfig
    this.config = checkNotNull(config);
    
	//Hold住种子节点
    this.bootstrapNodes = ImmutableSet.copyOf(config.getNodes().stream().map(Node::new).collect(Collectors.toList()));
}
```
### (4). BootstrapDiscoveryProvider属性
```
public class BootstrapDiscoveryProvider
    extends AbstractListenerManager<NodeDiscoveryEvent, NodeDiscoveryEventListener>
    implements NodeDiscoveryProvider {

	private static final Serializer SERIALIZER = Serializer.using(Namespace.builder()
		  .register(Namespaces.BASIC)
		  .nextId(Namespaces.BEGIN_USER_CUSTOM_ID)
		  .register(Node.class)
		  .register(NodeId.class)
		  .register(new AddressSerializer(), Address.class)
		  .build());

	private static final String HEARTBEAT_MESSAGE = "atomix-cluster-heartbeat";
	
	// ***********************************************************************
	// 引导时的种子节点
	// ***********************************************************************
	private final Collection<Node> bootstrapNodes;

	private final BootstrapDiscoveryConfig config;
	
	// ***********************************************************************
	// 这个接口先不要关心,只要清楚这个类主要提供通信服务
	// ***********************************************************************
	private volatile BootstrapService bootstrap;

	// ***********************************************************************
	// 最终活跃的所有节点
	// ***********************************************************************
	private Map<Address, Node> nodes = Maps.newConcurrentMap();

	
	// ***********************************************************************
	// 定时任务
	// ***********************************************************************
	private final ScheduledExecutorService heartbeatScheduler = Executors.newSingleThreadScheduledExecutor(namedThreads("atomix-bootstrap-heartbeat-sender", LOGGER));
	private final ExecutorService heartbeatExecutor = Executors.newSingleThreadExecutor(namedThreads("atomix-bootstrap-heartbeat-receiver", LOGGER));
}
```
### (5). NodeDiscoveryProvider.join
```
public CompletableFuture<Void> join(BootstrapService bootstrap, Node localNode) {
    // nodes 是最终活跃的节点
    // 判断本节点是否存在,如果不存在,则进行初始化
    if (nodes.putIfAbsent(localNode.address(), localNode) == null) {
      // hold住:BootstrapService
      this.bootstrap = bootstrap;
      // 典型的观察者模式
      post(new NodeDiscoveryEvent(NodeDiscoveryEvent.Type.JOIN, localNode));

       // ***********************************************************************
      // 配置接收心跳请求的处理.
	  // ***********************************************************************
      bootstrap.getMessagingService().registerHandler(
          HEARTBEAT_MESSAGE,
          (BiFunction<Address, byte[], byte[]>) (a, p) ->
                  // 解码,并反序列化
              handleHeartbeat(localNode, SERIALIZER.decode(p)), heartbeatExecutor);

      ComposableFuture<Void> future = new ComposableFuture<>();
      // 发送心跳
      sendHeartbeats(localNode)
      .whenComplete((r, e) -> {
        future.complete(null);
      });

      // 开启心跳定时任务
      heartbeatFuture = heartbeatScheduler.scheduleAtFixedRate(() -> {
        sendHeartbeats(localNode);
      }, 0, config.getHeartbeatInterval().toMillis(), TimeUnit.MILLISECONDS);

      return future.thenRun(() -> {
        LOGGER.info("Joined");
      });
    }
    return CompletableFuture.completedFuture(null);
} 
```
### (6). NodeDiscoveryProvider.handleHeartbeat
```
private byte[] handleHeartbeat(Node localNode, Node node) { // node为cliet发起心跳的节点.
    LOGGER.trace("{} - Received heartbeat: {}", localNode.address(), localNode.address());
    failureDetectors.computeIfAbsent(localNode.address(), n -> new PhiAccrualFailureDetector()).report();
    Node oldNode = nodes.put(node.address(), node);
    if (oldNode != null && !oldNode.id().equals(node.id())) {
      failureDetectors.remove(oldNode.address());
      post(new NodeDiscoveryEvent(NodeDiscoveryEvent.Type.LEAVE, oldNode));
      post(new NodeDiscoveryEvent(NodeDiscoveryEvent.Type.JOIN, node));
    } else if (oldNode == null) {
      post(new NodeDiscoveryEvent(NodeDiscoveryEvent.Type.JOIN, node));
    }
    // 序列化本节点所有的活跃节点返回给发送请求的远程节点.
    return SERIALIZER.encode(Lists.newArrayList(nodes.values()));
} 
```
### (7). NodeDiscoveryProvider.sendHeartbeats
```
private CompletableFuture<Void> sendHeartbeat(Node localNode, Address address) {
    return bootstrap.getMessagingService()
      // address : 为远程节点
      // type : 业务类型(心跳)
      // payload : 消息体
      .sendAndReceive(address, HEARTBEAT_MESSAGE, SERIALIZER.encode(localNode))
      .whenCompleteAsync((response, error) -> {
      if (error == null) {  
         // *****************************************************************************
		 // 远程节点,会返回它所有活跃的成员信息.
		 // *****************************************************************************
	    // 没有错误的情况下,代表发送心跳请求是成功的,对返回的结果进行转码.
        // 解码消息体
        Collection<Node> nodes = SERIALIZER.decode(response);
        for (Node node : nodes) {
          if (node.address().equals(address)) { // 如果,发送远程请求的网络地址包含在请求的网络地址中,则...
            Node oldNode = this.nodes.put(node.address(), node);
            if (oldNode != null && !oldNode.id().equals(node.id())) { // 如果节点已经存在,但是节点的id不同
              failureDetectors.remove(oldNode.address());
              // 让以前的节点离开
              post(new NodeDiscoveryEvent(NodeDiscoveryEvent.Type.LEAVE, oldNode));
              // 新的节点加入
              post(new NodeDiscoveryEvent(NodeDiscoveryEvent.Type.JOIN, node));
            } else if (oldNode == null) { // 节点不存在的情况下,代表节点是一个新的节点
              // 新的节点加入
              post(new NodeDiscoveryEvent(NodeDiscoveryEvent.Type.JOIN, node));
            }
          } else if (!this.nodes.containsKey(node.address()) || !this.nodes.get(node.address()).id().equals(node.id())) {
            // 向其它的节点发送心跳请求
            sendHeartbeat(localNode, node.address());
          }
        }
      } else {
        // 网络请求失败的情况下处理,在这里先不关心
        LOGGER.debug("{} - Sending heartbeat to {} failed", localNode, address, error);
        PhiAccrualFailureDetector failureDetector = failureDetectors.computeIfAbsent(address, n -> new PhiAccrualFailureDetector());
        double phi = failureDetector.phi();
        if (phi >= config.getFailureThreshold()
            || (phi == 0.0 && System.currentTimeMillis() - failureDetector.lastUpdated() > config.getFailureTimeout().toMillis())) {
          Node node = this.nodes.remove(address);
          if (node != null) {
            failureDetectors.remove(node.address());
            post(new NodeDiscoveryEvent(NodeDiscoveryEvent.Type.LEAVE, node));
          }
        }
      }
    }, heartbeatExecutor).exceptionally(e -> null)
        .thenApply(v -> null);
} 
```
### (8). UML图
!["BootstrapDiscoveryProvider UML图"](/assets/atomix/imgs/Discovery.jpg) 

### (9). 总结
BootstrapDiscoveryProvider在启动时,会向所有的种子节点(远程节点)进行心跳,而种子节点(远程节点)会返回它所有活跃的其它节点,这样就实现了节点之间的发现.  