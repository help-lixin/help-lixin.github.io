---
layout: post
title: 'Atomix源码之服务发现之BootstrapDiscoveryBuilder(一)' 
date: 2022-12-02
author: 李新
tags:  Atomix
---

### (1). 概述
在前面的节点发现的例子中,我们能看到Atomix用到了构建者模式,通过构建者创建:BootstrapDiscoveryProvider,在这里,先剖析下:BootstrapDiscoveryBuilder.  

### (2). BootstrapDiscoveryBuilder
```
public class BootstrapDiscoveryBuilder extends NodeDiscoveryBuilder {
	
	// Hold住一个配置(BootstrapDiscoveryConfig),Builder类所有的操作,估计是对这个类进行配置
	  private final BootstrapDiscoveryConfig config = new BootstrapDiscoveryConfig();

	
	public BootstrapDiscoveryBuilder withNodes(Collection<Node> locations) {
	    config.setNodes(locations.stream().map(Node::config).collect(Collectors.toList()));
	    return this;
	}
	  
	public BootstrapDiscoveryBuilder withHeartbeatInterval(Duration heartbeatInterval) {
		config.setHeartbeatInterval(heartbeatInterval);
		return this;
	}
		
	public BootstrapDiscoveryBuilder withFailureThreshold(int failureThreshold) {
		config.setFailureThreshold(failureThreshold);
		return this;
	}
	
	public BootstrapDiscoveryBuilder withFailureTimeout(Duration failureTimeout) {
		config.setFailureTimeout(failureTimeout);
	    return this;
	}

	public NodeDiscoveryProvider build() {
	    return new BootstrapDiscoveryProvider(config);
	}
}	
```
### (3). BootstrapDiscoveryConfig
```
public class BootstrapDiscoveryConfig extends NodeDiscoveryConfig {
  private static final int DEFAULT_HEARTBEAT_INTERVAL = 1000;
  private static final int DEFAULT_FAILURE_TIMEOUT = 10000;
  private static final int DEFAULT_PHI_FAILURE_THRESHOLD = 10;

  private Duration heartbeatInterval = Duration.ofMillis(DEFAULT_HEARTBEAT_INTERVAL);
  private int failureThreshold = DEFAULT_PHI_FAILURE_THRESHOLD;
  private Duration failureTimeout = Duration.ofMillis(DEFAULT_FAILURE_TIMEOUT);
  private Collection<NodeConfig> nodes = Collections.emptySet();

  // ... ...
}
```
### (4). 总结
通过对上面的代码进行分析,不难看出:BootstrapDiscoveryBuilder内部持有一个BootstrapDiscoveryConfig对象,而:BootstrapDiscoveryBuilder内的所有方法,都是对BootstrapDiscoveryConfig的方法进行配置. 