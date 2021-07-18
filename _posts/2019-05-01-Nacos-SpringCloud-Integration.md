---
layout: post
title: 'Nacos是如何与Spring无缝整合的(六)'
date: 2019-05-01
author: 李新
tags: Nacos
---

### (1). 概述
与Spring整合时,只需要添加依赖即可,那么,Nacos是如何与Spring整合的呢?答案就在:spring-cloud-starter-alibaba-nacos-config-xxxx.jar包里.  

### (2). /META-INF/spring.factories
```
# 1. NacosConfigBootstrapConfiguration
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
com.alibaba.cloud.nacos.NacosConfigBootstrapConfiguration

# 2. NacosConfigAutoConfiguration
#    NacosConfigEndpointAutoConfiguration在这里了不参与剖析
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.alibaba.cloud.nacos.NacosConfigAutoConfiguration,\
com.alibaba.cloud.nacos.endpoint.NacosConfigEndpointAutoConfiguration

# 启动失败处理,在这里不剖析
org.springframework.boot.diagnostics.FailureAnalyzer=\
com.alibaba.cloud.nacos.diagnostics.analyzer.NacosConnectionFailureAnalyzer
```
### (3). NacosConfigBootstrapConfiguration
```
@Configuration
@ConditionalOnProperty(name = "spring.cloud.nacos.config.enabled", matchIfMissing = true)
public class NacosConfigBootstrapConfiguration {
	
	// 1. 构建出业务模型所需要的配置信息
	@Bean
	@ConditionalOnMissingBean
	public NacosConfigProperties nacosConfigProperties() {
		return new NacosConfigProperties();
	}

	// 2. 构建:NacosConfigManager	
	@Bean
	@ConditionalOnMissingBean
	public NacosConfigManager nacosConfigManager(
	        // 依赖:NacosConfigProperties
			NacosConfigProperties nacosConfigProperties) {
		return new NacosConfigManager(nacosConfigProperties);
	}

    // ****************************************************************************************
    // 3. 构建:NacosPropertySourceLocator
	//    Spring启动时,会回调该方法:PropertySource<?>  NacosPropertySourceLocator.locate(Environment environment);
	// ****************************************************************************************
	@Bean
	public NacosPropertySourceLocator nacosPropertySourceLocator(
	        // 依赖NacosConfigManager
			NacosConfigManager nacosConfigManager) {
		return new NacosPropertySourceLocator(nacosConfigManager);
	}
}
```
### (4). NacosConfigAutoConfiguration
```
@Configuration
@ConditionalOnProperty(name = "spring.cloud.nacos.config.enabled", matchIfMissing = true)
public class NacosConfigAutoConfiguration {
	
	// 1. NacosRefreshHistory(动态更新配置的记录)
	@Bean
	public NacosRefreshHistory nacosRefreshHistory() {
		return new NacosRefreshHistory();
	}
	
	// 2. NacosRefreshProperties(动态更新配置是否启用的配置项)
	@Bean
	public NacosRefreshProperties nacosRefreshProperties() {
		return new NacosRefreshProperties();
	}
	
	// *******************************************************************
	// 3. NacosContextRefresher.registerNacosListener方法会触发:RefreshEvent事件.  
	//    applicationContext.publishEvent(new RefreshEvent(this, null, "Refresh Nacos config"));
	// *******************************************************************
	@Bean
	public NacosContextRefresher nacosContextRefresher(
			NacosConfigManager nacosConfigManager,
			NacosRefreshHistory nacosRefreshHistory) {
        return new NacosContextRefresher(nacosConfigManager, nacosRefreshHistory);				
	}
}
```
### (5). NacosConfigManager
> NacosConfigManager的作用:
> 1. 转换NacosConfigProperties为Properties.   
> 2. NacosFactory.createConfigService(Properties)创建:ConfigService.   

```
public class NacosConfigManager {

	private static final Logger log = LoggerFactory.getLogger(NacosConfigManager.class);

	private static ConfigService service = null;

	private NacosConfigProperties nacosConfigProperties;

	// *****************************************************
	// 1. 构造器
	// *****************************************************
	public NacosConfigManager(NacosConfigProperties nacosConfigProperties) {
		this.nacosConfigProperties = nacosConfigProperties;
		// *****************************************************
		// 2. 创建:ConfigService
		// *****************************************************
		createConfigService(nacosConfigProperties);
	}

	static ConfigService createConfigService(NacosConfigProperties nacosConfigProperties) {
		if (Objects.isNull(service)) {
			// 单例模式
			synchronized (NacosConfigManager.class) {
				try {
					if (Objects.isNull(service)) {
						// ******************************************************************
						// 3. 收集:NacosConfigProperties里的信息,并转换成:Properties
						//    这个方法是,Nacos的入口方法,并不陌生吧! NacosFactory.createConfigService(Properties)
						// ******************************************************************
						service = NacosFactory.createConfigService(nacosConfigProperties.assembleConfigServiceProperties());
					}
				}
				catch (NacosException e) {
					log.error(e.getMessage());
					throw new NacosConnectionFailureException(nacosConfigProperties.getServerAddr(), e.getMessage(), e);
				}
			}
		}
		return service;
	}

	public ConfigService getConfigService() {
		if (Objects.isNull(service)) {
			createConfigService(this.nacosConfigProperties);
		}
		return service;
	}

	public NacosConfigProperties getNacosConfigProperties() {
		return nacosConfigProperties;
	}
}
```
### (6). NacosContextRefresher
> NacosContextRefresher的作用:  
> 1. NacosPropertySource属于:PropertySource的实现,它是数据的载体.   
> 2. 获得系统中所有的:NacosPropertySource.  
> 3. 为ConfigService配置Listener,Listener内部会触发RefreshEvent事件.      


```
public class NacosContextRefresher
		// ApplicationReadyEvent是Spring启动时的事件.
		implements ApplicationListener<ApplicationReadyEvent>, ApplicationContextAware {
  
  private NacosConfigProperties nacosConfigProperties;
  private final boolean isRefreshEnabled;
  private final NacosRefreshHistory nacosRefreshHistory;
  
  // *******************************************************************
  // 不陌生吧,ConfigService是Nacos的入口程序.
  // *******************************************************************
  private final ConfigService configService;
  
  private ApplicationContext applicationContext;
  private AtomicBoolean ready = new AtomicBoolean(false);
  private Map<String, Listener> listenerMap = new ConcurrentHashMap<>(16); 
  
  // 1. 构造器
  public NacosContextRefresher(
			// 依赖:NacosConfigManager
            NacosConfigManager nacosConfigManager,
  			NacosRefreshHistory refreshHistory) {
	this.nacosConfigProperties = nacosConfigManager.getNacosConfigProperties();
	this.nacosRefreshHistory = refreshHistory;
	this.configService = nacosConfigManager.getConfigService();
	this.isRefreshEnabled = this.nacosConfigProperties.isRefreshEnabled();
  }

  // 2. Spring回调:ApplicationContextAware.setApplicationContext
  @Override
  public void setApplicationContext(ApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
  }
  
  // 3. Spring启动,触发:ApplicationReadyEvent时,会回调该方法.
  public void onApplicationEvent(ApplicationReadyEvent event) {
	if (this.ready.compareAndSet(false, true)) {  //ready是元子性的操作,保证只调用该方法一次
		this.registerNacosListenersForApplications();
	}
  }// end onApplicationEvent
  
  
  private void registerNacosListenersForApplications() {
	if (isRefreshEnabled()) {
		// *****************************************************************
		//  4. NacosPropertySource属于:PropertySource的实现,它是数据的载体.
		// *****************************************************************
		for (NacosPropertySource propertySource : NacosPropertySourceRepository.getAll()) {
			if (!propertySource.isRefreshable()) { // 不支持刷新的情况下,跳过
				continue;
			}
			
			// 取出:dataId和group
			String dataId = propertySource.getDataId();
			// 5. 调用:registerNacosListener方法,为:ConfigService配置:Listener
			registerNacosListener(propertySource.getGroup(), dataId);
		}
	}
  }
  
  
  private void registerNacosListener(final String groupKey, final String dataKey) {
		String key = NacosPropertySourceRepository.getMapKey(dataKey, groupKey);
		// 为key,创建监听器
		Listener listener = listenerMap.computeIfAbsent(key,
				lst -> new AbstractSharedListener() {
					@Override
					public void innerReceive(String dataId, String group,
							String configInfo) {
						refreshCountIncrement();
						nacosRefreshHistory.addRefreshRecord(dataId, group, configInfo);
						// todo feature: support single refresh for listening
						applicationContext.publishEvent(
								new RefreshEvent(this, null, "Refresh Nacos config"));
						if (log.isDebugEnabled()) {
							log.debug(String.format(
									"Refresh Nacos config group=%s,dataId=%s,configInfo=%s",
									group, dataId, configInfo));
						}
					}
				});
		try {
			// ****************************************************************
			// 6.为ConfigService配置监听器
			// ****************************************************************
			configService.addListener(dataKey, groupKey, listener);
		}
		catch (NacosException e) {
			log.warn(String.format(
					"register fail for nacos listener ,dataId=[%s],group=[%s]", dataKey,
					groupKey), e);
		}
  }  
}
```
### (7). 总结
1) 通过NacosConfigManager创建ConfigService.     
2) NacosPropertySourceLocator.locate方法,会返回PropertySource,Spring会把PropertySource添加到:Environment中.  
3) 为ConfigService配置监听器,当监听器触发时,触发:RefreshEvent事件,刷新Spring中的配置.   