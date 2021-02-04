---
layout: post
title: 'Seata 分支事务处理之ResourceManager(二)'
date: 2021-01-29
author: 李新
tags: Seata-AT源码
---

### (1). 概述
> 前面分析到:DataSourceProxy在构建时,会向TC进行注册,但未深入到里面去剖析.这一小节,深入剖析:ResourceManager

### (2). 看下ResourceManager的类图
!["ResourceManger类图"](/assets/seata/imgs/seata-ResourceManager.png)

### (2). ResourceManager在何时初始化的?
> 在RMClient初始化的时候,通过静态方法触发初始化,初始化的过程是通过SPI获得:ResourceManager的所有实现类,然后Hold住. 

```
// RMClient
public static void init(String applicationId, String transactionServiceGroup) {
	RmNettyRemotingClient rmNettyRemotingClient = RmNettyRemotingClient.getInstance(applicationId, transactionServiceGroup);
	// *************************************************************************
	// DefaultResourceManager.get会触发SPI获得所有的:ResourceManager接口的实现.
	// *************************************************************************
	rmNettyRemotingClient.setResourceManager(DefaultResourceManager.get());
	rmNettyRemotingClient.setTransactionMessageHandler(DefaultRMHandler.get());
	rmNettyRemotingClient.init();
}
```

### (3). DefaultResourceManager.get
```
public class DefaultResourceManager implements ResourceManager {
	
	// 这是通过SPI获得所有的:ResourceManager的实现
	// {
    //  AT=io.seata.rm.datasource.DataSourceManager@69d58731, 
	//  TCC=io.seata.rm.tcc.TCCResourceManager@3275a47f, 
	//  XA=io.seata.rm.datasource.xa.ResourceManagerXA@1b5af65b, 
	//  SAGA=io.seata.saga.rm.SagaResourceManager@2e0163cb
	// }
	protected static Map<BranchType, ResourceManager> resourceManagers = new ConcurrentHashMap<>();

	// 3. 构造器	
	private DefaultResourceManager() {
		initResourceManagers();
	}
	
	// 4. initResourceManagers
	protected void initResourceManagers() {
		//init all resource managers
		// 通过SPI获得所有的:ResourceManager的实现类
		List<ResourceManager> allResourceManagers = EnhancedServiceLoader.loadAll(ResourceManager.class);
		if (CollectionUtils.isNotEmpty(allResourceManagers)) {
			for (ResourceManager rm : allResourceManagers) {
				// 注册到静态变量:DefaultResourceManager.resourceManagers中
				resourceManagers.put(rm.getBranchType(), rm);
			}
		}
	} // end initResourceManagers
	
	
	// 1. get
	// 饿汉模式
	public static DefaultResourceManager get() {
		return SingletonHolder.INSTANCE;
	}
	
	// 2. inner calss
	private static class SingletonHolder {
		private static DefaultResourceManager INSTANCE = new DefaultResourceManager();
	}
}
```
### (4). DefaultResourceManager的职责
> 查看DefaultResourceManager的一些方法,就能看出:它是负责整个资源管理的统一入口.   
> 它会根据不通的:BranchType(AT/TCC/SAGA),分发任务给相应的:ResourceManager实现类.    

```
public class DefaultResourceManager implements ResourceManager {
	
    protected static Map<BranchType, ResourceManager> resourceManagers = new ConcurrentHashMap<>();

	// **************************************注册资源***********************************
	// 5. DataSourceManager.registerResource(resource)
	public void registerResource(Resource resource) {
		// 这段代码这样写: 
		// ResourceManager resourceManager= getResourceManager(resource.getBranchType());
		// resourceManager.registerResource(resource);
		
		getResourceManager(resource.getBranchType()).registerResource(resource);
	} // end registerResource
	
	// 取消资源注册
	public void unregisterResource(Resource resource) {
		getResourceManager(resource.getBranchType()).unregisterResource(resource);
	}
	
	// 分支commit
	 @Override
	public BranchStatus branchCommit(BranchType branchType, String xid, long branchId,
									 String resourceId, String applicationData)
		throws TransactionException {
		return getResourceManager(branchType).branchCommit(branchType, xid, branchId, resourceId, applicationData);
	}// end branchCommit

	// 分支rollback
	@Override
	public BranchStatus branchRollback(BranchType branchType, String xid, long branchId,
									   String resourceId, String applicationData)
		throws TransactionException {
		return getResourceManager(branchType).branchRollback(branchType, xid, branchId, resourceId, applicationData);
	} // end branchRollback

	// 分支注册
	@Override
	public Long branchRegister(BranchType branchType, String resourceId,
							   String clientId, String xid, String applicationData, String lockKeys)
		throws TransactionException {
		return getResourceManager(branchType).branchRegister(branchType, resourceId, clientId, xid, applicationData,
			lockKeys);
	} // end branchRegister

	// 分支汇报
	@Override
	public void branchReport(BranchType branchType, String xid, long branchId, BranchStatus status,
							 String applicationData) throws TransactionException {
		getResourceManager(branchType).branchReport(branchType, xid, branchId, status, applicationData);
	} // end branchReport


	// 锁定Query
	@Override
	public boolean lockQuery(BranchType branchType, String resourceId,
							 String xid, String lockKeys) throws TransactionException {
		return getResourceManager(branchType).lockQuery(branchType, resourceId, xid, lockKeys);
	} // end lockQuery
}
```

### (5). DefaultResourceManager.get().registerResource
> OK,我们再把焦点回到:DefaultResourceManager.get().registerResource.  
> 我这里以AT(在AT模式下DataSourceManager是ResourceSourceManager的实现)为例.   
> 先看一下:DataSourceManager的类图:  

!["DataSourceManager类图"](/assets/seata/imgs/seata-DataSourceManager.jpg)

```
// 1. DataSourceManager.registerResource(resource)
public void registerResource(Resource resource) {
	DataSourceProxy dataSourceProxy = (DataSourceProxy) resource;
	// 放到缓存里
	dataSourceCache.put(dataSourceProxy.getResourceId(), dataSourceProxy);
	
	// 调用父类注册资源
	super.registerResource(dataSourceProxy);
} // end registerResource


// 2. AbstractResourceManager.registerResource
// 最终是调用了:RmNettyRemotingClient
public void registerResource(Resource resource) {
	RmNettyRemotingClient.getInstance().registerResource(resource.getResourceGroupId(), resource.getResourceId());
}
```
### (6). RmNettyRemotingClient.registerResource

```
// 1. registerResource
public void registerResource(String resourceGroupId, String resourceId) {
	// Resource registration cannot be performed until the RM client is initialized
	if (StringUtils.isBlank(transactionServiceGroup)) {
		return;
	}

	if (getClientChannelManager().getChannels().isEmpty()) {
		getClientChannelManager().reconnect(transactionServiceGroup);
		return;
	}
	synchronized (getClientChannelManager().getChannels()) {
		for (Map.Entry<String, Channel> entry : getClientChannelManager().getChannels().entrySet()) {
			String serverAddress = entry.getKey();
			Channel rmChannel = entry.getValue();
			if (LOGGER.isInfoEnabled()) {
				LOGGER.info("will register resourceId:{}", resourceId);
			}
			// 2. 发送消息
			sendRegisterMessage(serverAddress, rmChannel, resourceId);
		}
	}
}// end registerResource

// 2. sendRegisterMessage
public void sendRegisterMessage(String serverAddress, Channel channel, String resourceId) {
	RegisterRMRequest message = new RegisterRMRequest(applicationId, transactionServiceGroup);
	message.setResourceIds(resourceId);
	try {
		// 3. 发送异步请求
		super.sendAsyncRequest(channel, message);
	} catch (FrameworkException e) {
		if (e.getErrcode() == FrameworkErrorCode.ChannelIsNotWritable && serverAddress != null) {
			getClientChannelManager().releaseChannel(channel, serverAddress);
			if (LOGGER.isInfoEnabled()) {
				LOGGER.info("remove not writable channel:{}", channel);
			}
		} else {
			LOGGER.error("register resource failed, channel:{},resourceId:{}", channel, resourceId, e);
		}
	}
} //end sendRegisterMessage

// 3. sendAsyncRequest
public void sendAsyncRequest(Channel channel, Object msg) {
	if (channel == null) {
		LOGGER.warn("sendAsyncRequest nothing, caused by null channel.");
		return;
	}
	RpcMessage rpcMessage = buildRequestMessage(msg, msg instanceof HeartbeatMessage
		? ProtocolConstants.MSGTYPE_HEARTBEAT_REQUEST
		: ProtocolConstants.MSGTYPE_RESQUEST_ONEWAY);
	if (rpcMessage.getBody() instanceof MergeMessage) {
		mergeMsgMap.put(rpcMessage.getId(), (MergeMessage) rpcMessage.getBody());
	}
	
	// 4. sendAsync
	super.sendAsync(channel, rpcMessage);
} // end sendAsyncRequest


// 4. sendAsync
protected void sendAsync(Channel channel, RpcMessage rpcMessage) {
	channelWritableCheck(channel, rpcMessage.getBody());
	if (LOGGER.isDebugEnabled()) {
		LOGGER.debug("write message:" + rpcMessage.getBody() + ", channel:" + channel + ",active?"
			+ channel.isActive() + ",writable?" + channel.isWritable() + ",isopen?" + channel.isOpen());
	}

	// 回调相应的钩子函数
	doBeforeRpcHooks(ChannelUtil.getAddressFromChannel(channel), rpcMessage);
	
	channel.writeAndFlush(rpcMessage).addListener((ChannelFutureListener) future -> {
		if (!future.isSuccess()) {
			destroyChannel(future.channel());
		}
	});
}// end

```
### (7). 总结
> DataSourceProxy在初始化时:会调用:DefaultResourceManager.get().registerResource(resource)方法向TC进行注册.  