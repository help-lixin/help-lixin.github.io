---
layout: post
title: 'Zeebe Gateway源码之StandaloneGateway创建AtomixCluster详解(一)' 
date: 2022-09-16
author: 李新
tags:  Zeebe
---

### (1). 概述
在这小一节,将对Gateway的源码进行剖析,Gateway的源码入口在哪?像这样的框架源码,我们首要的任务是先运行起来,然后,进行一个简单的HelloWorld入门后,去Linux进程查看下Gateway进程信息,所以,Gateway的入口代码在gateway的脚本程序里.

### (2). /bin/gateway
```
// ... ...
// StandaloneGateway就是程序的入口
exec "$JAVACMD" $JAVA_OPTS -Xms128m -XX:+ExitOnOutOfMemoryError -Dfile.encoding=UTF-8 \
  -classpath "$CLASSPATH" \
  -Dapp.name="gateway" \
  -Dapp.pid="$$" \
  -Dapp.repo="$REPO" \
  -Dapp.home="$BASEDIR" \
  -Dbasedir="$BASEDIR" \
  io.camunda.zeebe.gateway.StandaloneGateway 

// ... ...  
```
### (3). StandaloneGateway
```
// **********************************************************
// 配置了sprig自动扫描如下package,咱先不管它,先看main函数
// **********************************************************
@SpringBootApplication(
    scanBasePackages = {
      "io.camunda.zeebe.gateway",
      "io.camunda.zeebe.shared",
      "io.camunda.zeebe.util.liveness"
    })
// **********************************************************
// 配置了ConfigurationPropertie的扫描
// **********************************************************
@ConfigurationPropertiesScan(basePackages = {"io.camunda.zeebe.gateway", "io.camunda.zeebe.shared"})
public class StandaloneGateway
    //  **********************************************************
	// 要稍微注意下,实现了哪些接口,其中:CommandLineRunner是Spring容器启动完成之后的一个Callback函数.
	//  **********************************************************
    implements CommandLineRunner, ApplicationListener<ContextClosedEvent>, CloseableSilently {
	// ... ... 
}
```
### (4). StandaloneGateway.main
```
public static void main(final String[] args) {
	// 1. 为线程配置当出现异常时的处理
	Thread.setDefaultUncaughtExceptionHandler(FatalErrorHandler.uncaughtExceptionHandler(Loggers.GATEWAY_LOGGER));

	// 2. 配置banner
	System.setProperty("spring.banner.location", "classpath:/assets/zeebe_gateway_banner.txt");
	// 3. 配置容器为servlet/profiles
	final var application = new SpringApplicationBuilder(StandaloneGateway.class)
			.web(WebApplicationType.SERVLET)
			.logStartupInfo(true)
			.profiles(Profile.GATEWAY.getId())
			.build(args);
    // 4. 运行容器
	application.run();
}
```
### (5). StandaloneGateway构造器
> 对main函数进行分析,好像啥也没干,这时候,要把目光放到:scanBasePackages上,或者,其它上面了,我用Spring一般不喜欢用@Component注解,喜欢用@Configuration去配置所有Bean之间的依赖关系,这样有利于其它开发人员,到一个集中的地方进行配置Bean.我习惯于把:@Configuration注解当成一个xml,当然,在我的世界里,我理解注解确实只是XML的一种变种而已. 

```
@Autowired
public StandaloneGateway(
   // 依赖注入:GatewayCfg/SpringGatewayBridge/ActorClockConfiguration
   // ActorClockConfiguration是一个时钟管理,我们不分析,直接扔掉不管理.
  final GatewayCfg configuration,
  final SpringGatewayBridge springGatewayBridge,
  final ActorClockConfiguration clockConfig) {
	this.configuration = configuration;
	this.springGatewayBridge = springGatewayBridge;
	this.clockConfig = clockConfig;
}
```
### (6). GatewayCfg
> 咦,GatewayCfg是一个纯配置,这个配置对照application.yaml看一下就好了. 

```
@Component
@ConfigurationProperties(prefix = "zeebe.gateway")
public class GatewayCfg {

  private NetworkCfg network = new NetworkCfg();
  private ClusterCfg cluster = new ClusterCfg();
  private ThreadsCfg threads = new ThreadsCfg();
  private SecurityCfg security = new SecurityCfg();
  private LongPollingCfg longPolling = new LongPollingCfg();
  private List<InterceptorCfg> interceptors = new ArrayList<>();
  private boolean initialized = false;
  // ... ... 
}
```
### (7). SpringGatewayBridge
> 从代码上分析SpringGatewayBridge好像就是一个状态管理以及BrokerClient(与Broker通信的Client SDK),这是一个典型的桥梁模式哈.  

```
@Component
public class SpringGatewayBridge {

  private Supplier<Status> gatewayStatusSupplier;
  private Supplier<Optional<BrokerClusterState>> clusterStateSupplier;
  private Supplier<BrokerClient> brokerClientSupplier;
  
  // ... ... 
  
  // ************************************************
  // 注册:BrokerClient
  // ************************************************
  public void registerBrokerClientSupplier(final Supplier<BrokerClient> brokerClientSupplier) {
    this.brokerClientSupplier = brokerClientSupplier;
  }
  
  // ************************************************
  // 获取:BrokerClient
  // ************************************************
  public Optional<BrokerClient> getBrokerClient() {
    return Optional.ofNullable(brokerClientSupplier).map(Supplier::get);
  }
}
```
### (8). StandaloneGateway.run
> 通过前几步的分析,好像还是没有找到我想要的东西,这时候,要开始怀疑是不是忽略了什么,再细看下:StandaloneGateway发现有实现几个接口,其中有一个接口CommandLineRunner里有做一些处理,不妨看下这个接口的细节.   

```

// 通过Spring容器注入的GatewayCfg,主要用于对配置文件的管理.
private final GatewayCfg configuration;

private ActorScheduler actorScheduler;


public void run(final String... args) throws Exception {
    // ... ...
    // ****************************************************************
	// 看到了我们重点:AtomixCluster
	// Atomix是RAFT和Gossip的实现,咱先不去详细分析RAFT和Gossip.
	// ****************************************************************
	atomixCluster = createAtomixCluster(configuration.getCluster());
	// ... ...
	atomixCluster.start();
}
```
### (9). StandaloneGateway.createAtomixCluster
> createAtomixCluster方法的目的在于创建:AtomixCluster,监听着:26502,并与node节点保持联系.  

```
// ********************************************************
private AtomixCluster createAtomixCluster(final ClusterCfg config) {
	// ********************************************************
	// 创建成员协议,这个协议实际是一个Gossip协议
	// createMembershipProtocol函数的内容,请参考下面(10)小点,大部份参数是默认的,只是配置了一个:BroadcastDispute
	// ********************************************************
	final var membershipProtocol = createMembershipProtocol(config.getMembership());
	final var builder =
		AtomixCluster.builder()
		     // gateway
			.withMemberId(config.getMemberId())
			// gateway:26502
			.withAddress(Address.from(config.getHost(), config.getPort()))
			// zeebe-cluster
			.withClusterId(config.getClusterName())
			// ****************************************************************
			// 配置node节点(broker)的成员列表
			// ****************************************************************
			.withMembershipProvider(
				BootstrapDiscoveryProvider.builder()
				     //  node-1:26502
					.withNodes(Address.from(config.getContactPoint()))
					.build())
			// 配置上面的成员协议信息		
			.withMembershipProtocol(membershipProtocol)
			// 配置消息压缩信息
			.withMessageCompression(config.getMessageCompression());

     // 判断Security是否开启
	if (config.getSecurity().isEnabled()) {
	  applyClusterSecurityConfig(config, builder);
	}
    
	// 构建AtomixCluster
	return builder.build();
}
```
### (10). StandaloneGateway.createMembershipProtocol
```
private GroupMembershipProtocol createMembershipProtocol(final MembershipCfg config) {
	
	return SwimMembershipProtocol.builder()
	     // PT10S
		.withFailureTimeout(config.getFailureTimeout())
		// PT0.25S
		.withGossipInterval(config.getGossipInterval())
		// PT1S
		.withProbeInterval(config.getProbeInterval())
		// PT0.1S
		.withProbeTimeout(config.getProbeTimeout())
		// true
		.withBroadcastDisputes(config.isBroadcastDisputes())
		// false
		.withBroadcastUpdates(config.isBroadcastUpdates())
		// 2
		.withGossipFanout(config.getGossipFanout())
		// false
		.withNotifySuspect(config.isNotifySuspect())
		// 3
		.withSuspectProbes(config.getSuspectProbes())
		// PT10S
		.withSyncInterval(config.getSyncInterval())
		.build();
}
```
### (11). 总结
在这里仅分析到AtomixCluster的创建过程,暂时先分析到这里,毕竟内容太多,要拆开来写文章,从源码分析上能看出来:Zeebe有一部份内容(配置/Metrics)是向Spring靠扰了,会监听到9600端口,而AtomixCluster会监听26502端口,并且与node节点(node-1:26502)进行通信.  