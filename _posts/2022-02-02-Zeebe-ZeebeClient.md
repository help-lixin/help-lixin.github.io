---
layout: post
title: 'Zeebe ZeebeClient源码部析(四)' 
date: 2022-02-02
author: 李新
tags:  Zeebe
---

### (1). 概述

在这里,对ZeebeClient的源码进行剖析,首先需要了解下,ZeebeClient底层是GRpc通信(PB+Netty+Http2).

### (2). ZeebeClient类图
![ZeebeClient类图](/assets/zeebe/imgs/ZeebeClient.jpg)

### (3). ZeebeClient源码入口
```
ZeebeClient.newClientBuilder().gatewayAddress(envVarAddress);
```
### (4). ZeebeClient
```
static ZeebeClientBuilder newClientBuilder() {
	return new ZeebeClientBuilderImpl();
}
```
### (5). ZeebeClientBuilderImpl
```
public final class ZeebeClientBuilderImpl implements ZeebeClientBuilder, ZeebeClientConfiguration {
  public static final String PLAINTEXT_CONNECTION_VAR = "ZEEBE_INSECURE_CONNECTION";
  public static final String CA_CERTIFICATE_VAR = "ZEEBE_CA_CERTIFICATE_PATH";
  public static final String KEEP_ALIVE_VAR = "ZEEBE_KEEP_ALIVE";
  public static final String DEFAULT_GATEWAY_ADDRESS = "0.0.0.0:26500";

  private final List<ClientInterceptor> interceptors = new ArrayList<>();
  private String gatewayAddress = DEFAULT_GATEWAY_ADDRESS;
  private int jobWorkerMaxJobsActive = 32;
  private int numJobWorkerExecutionThreads = 1;
  private String defaultJobWorkerName = "default";
  private Duration defaultJobTimeout = Duration.ofMinutes(5);
  private Duration defaultJobPollInterval = Duration.ofMillis(100);
  private Duration defaultMessageTimeToLive = Duration.ofHours(1);
  private Duration defaultRequestTimeout = Duration.ofSeconds(10);
  private boolean usePlaintextConnection = false;
  private String certificatePath;
  private CredentialsProvider credentialsProvider;
  private Duration keepAlive = Duration.ofSeconds(45);
  private JsonMapper jsonMapper = new ZeebeObjectMapper();

  
  public ZeebeClient build() {
	  applyOverrides();
	  applyDefaults();
	
	  // *********************************************************************
	  // 创建ZeebeClientImpl,传递ZeebeClientConfiguration
	  // *********************************************************************
	  return new ZeebeClientImpl(this);
  } // end build
}
```
### (6). ZeebeClientImpl构建器
```

// 1. 构造器委托
public ZeebeClientImpl(final ZeebeClientConfiguration configuration) {
	// ******************************************************************
	// 构建GRPC中的ManagedChannel
	// ******************************************************************
	this(configuration, buildChannel(configuration));
}

// 2. 构造器委托
public ZeebeClientImpl(
      final ZeebeClientConfiguration configuration, final ManagedChannel channel) {
    // ******************************************************************
	// 构建GRPC中的Stub
	// ******************************************************************
	this(configuration, channel, buildGatewayStub(channel, configuration));
}

// 3. 构造器委托
public ZeebeClientImpl(
      final ZeebeClientConfiguration configuration,
      final ManagedChannel channel,
      final GatewayStub gatewayStub) {
     // ******************************************************************
	 // 构建线程池
	 // ******************************************************************
	this(configuration, channel, gatewayStub, buildExecutorService(configuration));
}

// 4. 构造器委托
public ZeebeClientImpl(
      final ZeebeClientConfiguration config,
      final ManagedChannel channel,
      final GatewayStub gatewayStub,
      final ScheduledExecutorService executorService) {
	this.config = config;
	this.jsonMapper = config.getJsonMapper();
	this.channel = channel;
	asyncStub = gatewayStub;
	this.executorService = executorService;

	if (config.getCredentialsProvider() != null) {
	  credentialsProvider = config.getCredentialsProvider();
	} else {
	  credentialsProvider = new NoopCredentialsProvider();
	}
	
	// ******************************************************************
	// 构建JobClient
	// ******************************************************************
	jobClient = newJobClient();
}
```
### (7). ZeebeClientImpl.buildChannel
```
public static ManagedChannel buildChannel(final ZeebeClientConfiguration config) {
	final URI address;

	try {
	  address = new URI("zb://" + config.getGatewayAddress());
	} catch (final URISyntaxException e) {
	  throw new RuntimeException("Failed to parse broker contact point", e);
	}
	
	// *************************************************************
	// GRPC中创建Channel
	// *************************************************************
	final NettyChannelBuilder channelBuilder = NettyChannelBuilder.forAddress(address.getHost(), address.getPort());

	configureConnectionSecurity(config, channelBuilder);
	channelBuilder.keepAliveTime(config.getKeepAlive().toMillis(), TimeUnit.MILLISECONDS);
	channelBuilder.userAgent("zeebe-client-java/" + VersionUtil.getVersion());

	return channelBuilder.build();
}
```
### (8). ZeebeClientImpl.buildGatewayStub
```
public static GatewayStub buildGatewayStub(
      final ManagedChannel channel, final ZeebeClientConfiguration config) {
	final CallCredentials credentials = buildCallCredentials(config);
	// GRPC通过PB生成的:GatewayStub
	final GatewayStub gatewayStub = GatewayGrpc.newStub(channel).withCallCredentials(credentials);
	if (!config.getInterceptors().isEmpty()) {
	  return gatewayStub.withInterceptors(
		  config.getInterceptors().toArray(new ClientInterceptor[] {}));
	}
	return gatewayStub;
}
```
### (9). ZeebeClientImpl.buildExecutorService
```
private static ScheduledExecutorService buildExecutorService(
      final ZeebeClientConfiguration configuration) {
	// JobWorker线程数	  
    final int threadCount = configuration.getNumJobWorkerExecutionThreads();
    return Executors.newScheduledThreadPool(threadCount);
}
```
### (10). ZeebeClientImpl.newJobClient
```
private JobClient newJobClient() {
	// 包裹:GatewayStub,GatewayStub是GRPC通信的核心
	return new JobClientImpl(asyncStub, config, jsonMapper, credentialsProvider::shouldRetryRequest);
}
``` 
### (11). ZeebeClient接口扫盲
> 为什么在这里要稍微扫盲一下ZeebeClient,原因在于:ZeebeClient代表着Client拥有哪些能力. 

```
public interface ZeebeClient extends AutoCloseable, JobClient {
	TopologyRequestStep1 newTopologyRequest();
	
	ZeebeClientConfiguration getConfiguration();
	
	// 创建流程部署命令
	DeployProcessCommandStep1 newDeployCommand();
	
	// 创建新的流程实例命令
	CreateProcessInstanceCommandStep1 newCreateInstanceCommand();
	
	// 创建流程实例取消命令
	CancelProcessInstanceCommandStep1 newCancelInstanceCommand(long processInstanceKey);
	
	// 为某个流程,设置变量命令
	SetVariablesCommandStep1 newSetVariablesCommand(long elementInstanceKey);
	
	// 发布消息命令
	PublishMessageCommandStep1 newPublishMessageCommand();
	
	ResolveIncidentCommandStep1 newResolveIncidentCommand(long incidentKey);
	
	UpdateRetriesJobCommandStep1 newUpdateRetriesCommand(long jobKey);
	
	UpdateRetriesJobCommandStep1 newUpdateRetriesCommand(ActivatedJob job);
	
	// 创建Worker命令(Worker是指我们的业务逻辑Handler)
	JobWorkerBuilderStep1 newWorker();
	
	// 创建一个活动的Job命令
	ActivateJobsCommandStep1 newActivateJobsCommand();
} // end ZeebeClient


public interface JobClient { 
	// 完成命令
	CompleteJobCommandStep1 newCompleteCommand(long jobKey);
	CompleteJobCommandStep1 newCompleteCommand(ActivatedJob job);
	
	// 失败命令
	FailJobCommandStep1 newFailCommand(long jobKey);
	FailJobCommandStep1 newFailCommand(ActivatedJob job);
	
	// 异常命令
	ThrowErrorCommandStep1 newThrowErrorCommand(long jobKey);
	ThrowErrorCommandStep1 newThrowErrorCommand(ActivatedJob job);
} // end JobClient
```
### (12). 总结
总体来说,ZeebeClient设计还是挺不错的,用到了:构建者模式,命令模式,责任链模式,ZeebClient有不少值得学习的地方,从始至终能发现,它基本都是面向接口编程.  