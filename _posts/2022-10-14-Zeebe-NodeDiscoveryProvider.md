---
layout: post
title: 'Zeebe ClusterServicesStep源码之NodeDiscoveryProvider(七)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
在这一小篇,我们分析另一个接口:NodeDiscoveryProvider,从名字上,能看出来,应该是属于服务发现.

### (2). AtomixCluster.buildLocationProvider
```
protected static NodeDiscoveryProvider buildLocationProvider(final ClusterConfig config) {
    final NodeDiscoveryConfig discoveryProviderConfig = config.getDiscoveryConfig();
    if (discoveryProviderConfig != null) { // true
      return discoveryProviderConfig.getType().newProvider(discoveryProviderConfig);
    }
    return new BootstrapDiscoveryProvider(Collections.emptyList());
  }
```
### (3). BootstrapDiscoveryProvider$Type.newProvider
```
public final class BootstrapDiscoveryProvider
    extends AbstractListenerManager<NodeDiscoveryEvent, NodeDiscoveryEventListener>
    implements NodeDiscoveryProvider {
		public static class Type implements NodeDiscoveryProvider.Type<BootstrapDiscoveryConfig> {
			private static final String NAME = "bootstrap";

			@Override
			public String name() {
			  return NAME;
			}

			// *************************************************************************************
			// BootstrapDiscoveryProvider$Type.newProvider
			// *************************************************************************************
			@Override
			public NodeDiscoveryProvider newProvider(final BootstrapDiscoveryConfig config) {
			  return new BootstrapDiscoveryProvider(config);
			}
		} // end
}
```
### (4). NodeDiscoveryProvider
> 先看一下接口:NodeDiscoveryProvider

```
public interface NodeDiscoveryProvider
    extends ListenerService<NodeDiscoveryEvent, NodeDiscoveryEventListener>,
        Configured<NodeDiscoveryConfig> {
  // 获取所有的成员信息			
  Set<Node> getNodes();

  // 加入集群
  CompletableFuture<Void> join(BootstrapService bootstrap, Node localNode);

  // 离开集群
  CompletableFuture<Void> leave(Node localNode);

  // 节点发现类型
  interface Type<C extends NodeDiscoveryConfig> extends NamedType {
    NodeDiscoveryProvider newProvider(C config);
  } // end Type
} // end NodeDiscoveryProvider
```
### (5). BootstrapDiscoveryProvider
```
public final class BootstrapDiscoveryProvider
    extends AbstractListenerManager<NodeDiscoveryEvent, NodeDiscoveryEventListener>
    implements NodeDiscoveryProvider { 
  public static final Type TYPE = new Type();
  private static final Logger LOGGER = LoggerFactory.getLogger(BootstrapDiscoveryProvider.class);
  private final ImmutableSet<Node> bootstrapNodes;
  private final BootstrapDiscoveryConfig config;

  public BootstrapDiscoveryProvider(final Node... bootstrapNodes) {
    this(Arrays.asList(bootstrapNodes));
  }

  public BootstrapDiscoveryProvider(final Collection<Node> bootstrapNodes) {
    this(
        new BootstrapDiscoveryConfig()
            .setNodes(
                bootstrapNodes.stream()
                    .map(node -> new NodeConfig().setId(node.id()).setAddress(node.address()))
                    .collect(Collectors.toList()))
		);
  }

  BootstrapDiscoveryProvider(final BootstrapDiscoveryConfig config) { 
    this.config = checkNotNull(config);
    bootstrapNodes = ImmutableSet.copyOf(config.getNodes().stream().map(Node::new).collect(Collectors.toList()));
  }

  public static BootstrapDiscoveryBuilder builder() { 
    return new BootstrapDiscoveryBuilder();
  }

  @Override
  public BootstrapDiscoveryConfig config() {
    return config;
  }

  @Override
  public Set<Node> getNodes() {
    return bootstrapNodes;
  }

  @Override
  public CompletableFuture<Void> join(final BootstrapService bootstrap, final Node localNode) {
    LOGGER.info("Local node {} joined the bootstrap service", localNode);
    return CompletableFuture.completedFuture(null);
  }

  @Override
  public CompletableFuture<Void> leave(final Node localNode) {
    LOGGER.info("Local node {} left the bootstrap servide", localNode);
    return CompletableFuture.completedFuture(null);
  }


  public static class Type implements NodeDiscoveryProvider.Type<BootstrapDiscoveryConfig> {
    private static final String NAME = "bootstrap";

    @Override
    public String name() {
      return NAME;
    } // 

    @Override
    public NodeDiscoveryProvider newProvider(final BootstrapDiscoveryConfig config) {
      return new BootstrapDiscoveryProvider(config);
    }  //end newProvider
  } // end Type
}
```
### (6). 总结
好像:BootstrapDiscoveryProvider仅仅是数据(List<Node>)的载体而已,加入集群和离开集群啥都没干.  