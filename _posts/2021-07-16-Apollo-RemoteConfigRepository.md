---
layout: post
title: 'Apollo源码学习之:拉取配置过程剖析(七)' 
date: 2021-07-16
author: 李新
tags:  Apollo
---

### (1). 概述
前面对Apollo与Spring的整合部份的源码进行了剖析,Spring的PropertySource最终会委托给Apollo的Config.getProperty方法.  
在这一小节,主要剖析Config,带着问题来看源码:  
1) Config与Meta Server进行交互的,请求报文是什么?  
2) Config与Config Service进行交互的,请求报文是什么?  

### (2). ConfigServiceLocator
> Config与Meta Server进行交互的,请求报文是什么?     
> ConfigServiceLocator主要负责与Meta Server通信,获得:Config Service服务列表.     
> Meta Server主要是用来做负载均衡的,当Config Service不可用时,可以,换另一个Config Service.   

```
public class ConfigServiceLocator {
  private static final Logger logger = LoggerFactory.getLogger(ConfigServiceLocator.class);
  private HttpClient m_httpClient;
  private ConfigUtil m_configUtil;
  // 最终的数据载体:ConfigService服务列表
  private AtomicReference<List<ServiceDTO>> m_configServices;
  private Type m_responseType;
  // 定时任务
  private ScheduledExecutorService m_executorService;
  private static final Joiner.MapJoiner MAP_JOINER = Joiner.on("&").withKeyValueSeparator("=");
  private static final Escaper queryParamEscaper = UrlEscapers.urlFormParameterEscaper();

  // 1. 构建器
  public ConfigServiceLocator() {
    List<ServiceDTO> initial = Lists.newArrayList();
    m_configServices = new AtomicReference<>(initial);
    m_responseType = new TypeToken<List<ServiceDTO>>() {
    }.getType();
    m_httpClient = ApolloInjector.getInstance(HttpClient.class);
    m_configUtil = ApolloInjector.getInstance(ConfigUtil.class);
    this.m_executorService = Executors.newScheduledThreadPool(1, ApolloThreadFactory.create("ConfigServiceLocator", true));
	// ****************************************************************************
	// 2. 初始化:ConfigService列表
	// ****************************************************************************
    initConfigServices();
  }


  // ****************************************************************************
  // 3. 初始化:ConfigService列表
  // ****************************************************************************
  private void initConfigServices() {
    
	// 4. 读取配置(apollo.configService),并转换成业务模型:ServiceDTO
	// get from run time configurations
    List<ServiceDTO> customizedConfigServices = getCustomizedConfigService();

    // 5. 如果结果集有内容,则,把结果,设置到当前属性(m_configServices)上,并直返回了,不会走HTTP请求
    if (customizedConfigServices != null) {
      setConfigServices(customizedConfigServices);
	  // 注意,直接返回了
      return;
    }

    // ***************************************************************************** 
	// 6. 调用meat server,获得confi service服务列表
	// ***************************************************************************** 
    // update from meta service
    this.tryUpdateConfigServices();
	
	// ***************************************************************************** 
	// 创建定时任务(apollo.refreshInterval=5),继续请求:tryUpdateConfigServices方法
	// ***************************************************************************** 
    this.schedulePeriodicRefresh();
  }

  private boolean tryUpdateConfigServices() {
    try {
      updateConfigServices();
      return true;
    } catch (Throwable ex) {
      //ignore
    }
    return false;
  }

  // ***************************************************************************** 
  // 7. 获取config service列表
  // ***************************************************************************** 
  private synchronized void updateConfigServices() {
    // 8. 构建HTTP请求
	//     http://127.0.0.1:8080/services/config?appId=7BBB492B-62F8-453F-B50B-0D568308E87A&ip=172.16.95.197
    String url = assembleMetaServiceUrl();
	
	
    HttpRequest request = new HttpRequest(url);
	// 最大重试次数
    int maxRetries = 2;
    Throwable exception = null;

    for (int i = 0; i < maxRetries; i++) {
	  Transaction transaction = Tracer.newTransaction("Apollo.MetaService", "getConfigService");
      transaction.addData("Url", url);
      try {
		// 9. 发起HTTP请求
        HttpResponse<List<ServiceDTO>> response = m_httpClient.doGet(request, m_responseType);
        transaction.setStatus(Transaction.SUCCESS);
		
		// 10. 解析response内容,转换成业务模型:ServiceDTO
        List<ServiceDTO> services = response.getBody();
        if (services == null || services.isEmpty()) { // 如果,services为空,则跳过
          logConfigService("Empty response!");
          continue;
        }
		
		// 11. 把解析后的结果(List<ServiceDTO>),设置到当前的属性上(m_configServices)
        setConfigServices(services);
        return;
      } catch (Throwable ex) {
        Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
        transaction.setStatus(ex);
        exception = ex;
      } finally {
        transaction.complete();
      }
	  
      try {
        m_configUtil.getOnErrorRetryIntervalTimeUnit().sleep(m_configUtil.getOnErrorRetryInterval());
      } catch (InterruptedException ex) {
        //ignore
      }
    }
    throw new ApolloConfigException(String.format("Get config services failed from %s", url), exception);
  }
  
  // 8.1 构建HTTP请求
  //  http://127.0.0.1:8080/services/config?appId=7BBB492B-62F8-453F-B50B-0D568308E87A&ip=172.16.95.197
  private String assembleMetaServiceUrl() {
    String domainName = m_configUtil.getMetaServerDomainName();
    String appId = m_configUtil.getAppId();
    String localIp = m_configUtil.getLocalIp();

    Map<String, String> queryParams = Maps.newHashMap();
    queryParams.put("appId", queryParamEscaper.escape(appId));
    if (!Strings.isNullOrEmpty(localIp)) {
      queryParams.put("ip", queryParamEscaper.escape(localIp));
    }
    return domainName + "/services/config?" + MAP_JOINER.join(queryParams);
  }
}
```
### (3). RemoteConfigRepository
> Config与Config Service进行交互的,请求报文是什么?         
> RemoteConfigRepository主要负责与Config Service进行交互,发起HTTP请求,获得所有的配置信息.   

```
public class RemoteConfigRepository extends AbstractConfigRepository {
	private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR);
	private static final Joiner.MapJoiner MAP_JOINER = Joiner.on("&").withKeyValueSeparator("=");
	private static final Escaper pathEscaper = UrlEscapers.urlPathSegmentEscaper();
	private static final Escaper queryParamEscaper = UrlEscapers.urlFormParameterEscaper();
	
	// 前面有介绍了,主要负责存储config service列表信息.
	private final ConfigServiceLocator m_serviceLocator;
	private final HttpClient m_httpClient;
	private final ConfigUtil m_configUtil;
	
	
	private final RemoteConfigLongPollService remoteConfigLongPollService;
	private volatile AtomicReference<ApolloConfig> m_configCache;
	private final String m_namespace;
	private final static ScheduledExecutorService m_executorService;
	private final AtomicReference<ServiceDTO> m_longPollServiceDto;
	private final AtomicReference<ApolloNotificationMessages> m_remoteMessages;
	private final RateLimiter m_loadConfigRateLimiter;
	private final AtomicBoolean m_configNeedForceRefresh;
	private final SchedulePolicy m_loadConfigFailSchedulePolicy;
	private static final Gson GSON = new Gson();
	
	public RemoteConfigRepository(String namespace) {
		m_namespace = namespace;
		m_configCache = new AtomicReference<>();
		m_configUtil = ApolloInjector.getInstance(ConfigUtil.class);
		m_httpClient = ApolloInjector.getInstance(HttpClient.class);
		m_serviceLocator = ApolloInjector.getInstance(ConfigServiceLocator.class);
		remoteConfigLongPollService = ApolloInjector.getInstance(RemoteConfigLongPollService.class);
		m_longPollServiceDto = new AtomicReference<>();
		m_remoteMessages = new AtomicReference<>();
		m_loadConfigRateLimiter = RateLimiter.create(m_configUtil.getLoadConfigQPS());
		m_configNeedForceRefresh = new AtomicBoolean(true);
		m_loadConfigFailSchedulePolicy = new ExponentialSchedulePolicy(m_configUtil.getOnErrorRetryInterval(), m_configUtil.getOnErrorRetryInterval() * 8);
		
		// *****************************************************************************
		// 1. 同步配置
		// *****************************************************************************
		this.trySync();
		
		// *****************************************************************************
		// 创建订时任务(apollo.refreshInterval=5),内部仍然是调用:trySync方法
		// *****************************************************************************
		this.schedulePeriodicRefresh();
		
		// 委托给:RemoteConfigLongPollService进行长轮询处理,这部份内容,留到后面剖析.
		this.scheduleLongPollingRefresh();
	} // end Constructor
	
	
	protected synchronized void sync() {
	    Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "syncRemoteConfig");
	    try {
		  // 上一次的配置	
	      ApolloConfig previous = m_configCache.get();
		  
		  // *****************************************************************************
		  // 2. 当前配置为最新的,加载配置
		  // *****************************************************************************
	      ApolloConfig current = loadApolloConfig();
	
	      //reference equals means HTTP 304
	      if (previous != current) {
	        logger.debug("Remote Config refreshed!");
	        m_configCache.set(current);
			// 触发Event
	        this.fireRepositoryChange(m_namespace, this.getConfig());
	      }
	
	      if (current != null) {
	        Tracer.logEvent(String.format("Apollo.Client.Configs.%s", current.getNamespaceName()), current.getReleaseKey());
	      }
	      transaction.setStatus(Transaction.SUCCESS);
	    } catch (Throwable ex) {
	      transaction.setStatus(ex);
	      throw ex;
	    } finally {
	      transaction.complete();
	    }
	}// end sync
	
	// 获取:config service列表
	private List<ServiceDTO> getConfigServices() {
	    List<ServiceDTO> services = m_serviceLocator.getConfigServices();
	    if (services.size() == 0) {
	      throw new ApolloConfigException("No available config service");
	    }
	    return services;
	}
	
	// *****************************************************************************
	// 2. 调用config service,获得最新的配置
	// *****************************************************************************
	private ApolloConfig loadApolloConfig() {

	  // 限流控制,请求过快的情况下,sleep
	  if (!m_loadConfigRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
	    //wait at most 5 seconds
	    try {
	      TimeUnit.SECONDS.sleep(5);
	    } catch (InterruptedException e) {
	    }
	  }
	  
	  // 请求的参数
	  String appId = m_configUtil.getAppId();
	  String cluster = m_configUtil.getCluster();
	  // ******************************************************
	  // 在有些特殊情况下,应用有需求对不同的集群做不同的配置.
	  // 比如部署在A机房的应用连接的es服务器地址和部署在B机房的应用连接的es服务器地址不一样. 
	  // 在这种情况下，可以通过在Apollo创建不同的集群来解决
	  // 会读取:/opt/settings/server.properties(linux)或C:\opt\settings\server.properties(windows)文件中的idc属性作为集群名字. 
	  // ******************************************************
	  String dataCenter = m_configUtil.getDataCenter();
	  String secret = m_configUtil.getAccessKeySecret();
	  Tracer.logEvent("Apollo.Client.ConfigMeta", STRING_JOINER.join(appId, cluster, m_namespace));
	  int maxRetries = m_configNeedForceRefresh.get() ? 2 : 1;
	  long onErrorSleepTime = 0; // 0 means no sleep
	  Throwable exception = null;
	
	  // 3. 获得所有的config service列表(前面有分析过了的)
	  List<ServiceDTO> configServices = getConfigServices();
	  String url = null;
	  retryLoopLabel:
	  for (int i = 0; i < maxRetries; i++) {
	    List<ServiceDTO> randomConfigServices = Lists.newLinkedList(configServices);
	    Collections.shuffle(randomConfigServices);
	    
		//Access the server which notifies the client first
	    if (m_longPollServiceDto.get() != null) {
	      randomConfigServices.add(0, m_longPollServiceDto.getAndSet(null));
	    }
		
		// 随机取一个:config service
	    for (ServiceDTO configService : randomConfigServices) {
	      // ... ...

		  // 4. 构建HTTP请求(向config service发起请求)
		  //    http://172.16.95.197:8080/configs/7BBB492B-62F8-453F-B50B-0D568308E87A/default/TEST1.jdbc?ip=172.16.95.197
	      url = assembleQueryConfigUrl(configService.getHomepageUrl(), appId, cluster, m_namespace,dataCenter, m_remoteMessages.get(), m_configCache.get());
	      logger.debug("Loading config from {}", url);
	      
		  HttpRequest request = new HttpRequest(url);
		  // *****************************************************************************
		  // 如果有配置(apollo.accesskey.secret=xxx)的情况下.
		  // 会把:url/appId/secret进行签名产生签名的结果,并设置到请求头里
		  // 比如:  
		  //    Authorization: Apollo ${appId} ${signature}
		  //    Timestamp: ${currentTimeMillis}  
		  // *****************************************************************************
	      if (!StringUtils.isBlank(secret)) { 
	        Map<String, String> headers = Signature.buildHttpHeaders(url, appId, secret);
	        request.setHeaders(headers);
	      }
		  
	      Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "queryConfig");
	      transaction.addData("Url", url);
	      try {
			// 5. 发送HTTP请求(GET)
	        HttpResponse<ApolloConfig> response = m_httpClient.doGet(request, ApolloConfig.class);
	        m_configNeedForceRefresh.set(false);
	        m_loadConfigFailSchedulePolicy.success();
	
	        transaction.addData("StatusCode", response.getStatusCode());
	        transaction.setStatus(Transaction.SUCCESS);
	
	        if (response.getStatusCode() == 304) { // 如果返回协议头Code是:304,则,直接返回
	          logger.debug("Config server responds with 304 HTTP status code.");
	          return m_configCache.get();
	        }
			
			// 6. 解析HTTP返回结果,并转换成:ApolloConfig
	        ApolloConfig result = response.getBody();
	        logger.debug("Loaded config for {}: {}", m_namespace, result);
	        return result;
	      } catch (ApolloConfigStatusCodeException ex) {  // 当异常是404时,代表:config not found
	        ApolloConfigStatusCodeException statusCodeException = ex;
	        //config not found
	        if (ex.getStatusCode() == 404) {
	          String message = String.format("Could not find config for namespace - appId: %s, cluster: %s, namespace: %s, " +"please check whether the configs are released in Apollo!",appId, cluster, m_namespace);
	          statusCodeException = new ApolloConfigStatusCodeException(ex.getStatusCode(),message);
	        }
			
	        Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(statusCodeException));
	        transaction.setStatus(statusCodeException);
	        exception = statusCodeException;
	        if(ex.getStatusCode() == 404) { // 404 的情况下,都不需要再重试了
	          break retryLoopLabel;
	        }
	      } catch (Throwable ex) {
	        Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
	        transaction.setStatus(ex);
	        exception = ex;
	      } finally {
	        transaction.complete();
	      }
		  //... ...
	    }
	  }
	  
	  String message = String.format("Load Apollo Config failed - appId: %s, cluster: %s, namespace: %s, url: %s", appId, cluster, m_namespace, url);
	  throw new ApolloConfigException(message, exception);
	} // end loadApolloConfig
	
}
```
### (4). RemoteConfigLongPollService
RemoteConfigLongPollService固名思义,就是一个轮询的操作,轮询的目的就是为了近可能实时的知道有配置变化(官网说是60秒,实际超时为:90秒),返回的结果有两种情况:  
1) 返回协议头为:200,并且,有内容,但不是配置详细信息.  
2) 返回协议头为:304,继续进行轮询.

```
public boolean submit(String namespace, RemoteConfigRepository remoteConfigRepository) {
	boolean added = m_longPollNamespaces.put(namespace, remoteConfigRepository);
	m_notifications.putIfAbsent(namespace, INIT_NOTIFICATION_ID);
	if (!m_longPollStarted.get()) {
	  // ****************************************************************
	  // 1. 开启长轮询
	  // ****************************************************************
	  startLongPolling();
	}
	return added;
} // end submit

private void startLongPolling() {
    if (!m_longPollStarted.compareAndSet(false, true)) {
      //already started
      return;
    }
    try {
      
	  // 发起HTTP请求需要的参烽
	  final String appId = m_configUtil.getAppId();
      final String cluster = m_configUtil.getCluster();
      final String dataCenter = m_configUtil.getDataCenter();
      final String secret = m_configUtil.getAccessKeySecret();
      final long longPollingInitialDelayInMills = m_configUtil.getLongPollingInitialDelayInMills();
	  
	  // ****************************************************************
	  // 2. 提交到线程池去运行.
	  // ****************************************************************
      m_longPollingService.submit(new Runnable() {
        @Override
        public void run() {
			
          if (longPollingInitialDelayInMills > 0) {
            try {
              logger.debug("Long polling will start in {} ms.", longPollingInitialDelayInMills);
              TimeUnit.MILLISECONDS.sleep(longPollingInitialDelayInMills);
            } catch (InterruptedException e) {
              //ignore
            }
          }
		  
		  // ****************************************************************
		  // 3.开始轮询
		  // ****************************************************************
          doLongPollingRefresh(appId, cluster, dataCenter, secret);
        }
      });
    } catch (Throwable ex) {
      m_longPollStarted.set(false);
      ApolloConfigException exception = new ApolloConfigException("Schedule long polling refresh failed", ex);
      Tracer.logError(exception);
      logger.warn(ExceptionUtil.getDetailMessage(exception));
    }
} // end startLongPolling


private void doLongPollingRefresh(String appId, String cluster, String dataCenter, String secret) {
    final Random random = new Random();
    ServiceDTO lastServiceDto = null;
    
	while (!m_longPollingStopped.get() && !Thread.currentThread().isInterrupted()) {
      
	  // 对请求进行限流,保护服务器
	  if (!m_longPollRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
        //wait at most 5 seconds
        try {
          TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
        }
      }
	  
      Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "pollNotification");
      String url = null;
      try {
        
		// 上一次用过的config service,如果没有,则随取一个出来用.
		if (lastServiceDto == null) {
          List<ServiceDTO> configServices = getConfigServices();
          lastServiceDto = configServices.get(random.nextInt(configServices.size()));
        }

	    // ****************************************************************
		// 构建HTTP请求
		// http://172.16.95.197:8080/notifications/v2?cluster=default&appId=7BBB492B-62F8-453F-B50B-0D568308E87A&ip=172.16.95.197&notifications=%5B%7B%22namespaceName%22%3A%22TEST1.jdbc%22%2C%22notificationId%22%3A-1%7D%5D
		// ****************************************************************
        url = assembleLongPollRefreshUrl(lastServiceDto.getHomepageUrl(), appId, cluster, dataCenter,m_notifications);
        logger.debug("Long polling from {}", url);
		
		
        HttpRequest request = new HttpRequest(url);
		// *******************************************************************
		// private static final int LONG_POLLING_READ_TIMEOUT = 90 * 1000;
		// 官网说是60秒,而客户端的请求超时为:90秒,为什么不是70秒就够了呢?要设置那么长
		// 有一个问题哦,底层是BIO模型吗?CLIENT是否会对应相应的FD文件?
		// *******************************************************************
        request.setReadTimeout(LONG_POLLING_READ_TIMEOUT);
        
		// 针对secret的处理,前面有剖析,这里不再重复.
		if (!StringUtils.isBlank(secret)) {
          Map<String, String> headers = Signature.buildHttpHeaders(url, appId, secret);
          request.setHeaders(headers);
        }
        transaction.addData("Url", url);

		// 发起HTTP请求
        final HttpResponse<List<ApolloConfigNotification>> response = m_httpClient.doGet(request, m_responseType);
        logger.debug("Long polling response: {}, url: {}", response.getStatusCode(), url);
        
		
		if (response.getStatusCode() == 200 && response.getBody() != null) { // 针对200的处理
          updateNotifications(response.getBody());
          updateRemoteNotifications(response.getBody());
          transaction.addData("Result", response.getBody().toString());
		  // 仅仅是发个通知而已,response不会有配置文件信息的
          notify(lastServiceDto, response.getBody());
        }

        //try to load balance
        if (response.getStatusCode() == 304 && random.nextBoolean()) { // 针对304的处理
          lastServiceDto = null;
        }
		
        m_longPollFailSchedulePolicyInSecond.success();
        transaction.addData("StatusCode", response.getStatusCode());
        transaction.setStatus(Transaction.SUCCESS);
      } catch (Throwable ex) {  // 针对异常处理
        lastServiceDto = null;
        Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
        transaction.setStatus(ex);
        long sleepTimeInSecond = m_longPollFailSchedulePolicyInSecond.fail();
        logger.warn("Long polling failed, will retry in {} seconds. appId: {}, cluster: {}, namespaces: {}, long polling url: {}, reason: {}",
            sleepTimeInSecond, appId, cluster, assembleNamespaces(), url, ExceptionUtil.getDetailMessage(ex));
        try {
          TimeUnit.SECONDS.sleep(sleepTimeInSecond);
        } catch (InterruptedException ie) {
          //ignore
        }
      } finally {
        transaction.complete();
      }
    }
} // end doLongPollingRefresh

```
### (5). 总结
至此,Apollo的主流代码已经分析完毕,后面的一些内容也就不分析了,除非业务上有需要,对Apollo做一个总结:   
1) Apollo会先与Meta Server进行通信,获得config service列表.  
2) 在RemoteConfigRepository内部,会随机取一个config service,获得最新的配置信息.   
3) 创建长轮询任务,如果配置有更新,仅发出一个通知,会继续由:RemoteConfigRepository去拉取配置.  