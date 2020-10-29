---
layout: post
title: 'Eureka源码-服务注册详解(EurekaClient)'
date: 2019-06-29
author: 李新
tags: Eureka
---

### (1).EurekaClientAutoConfiguration
```
public class EurekaClientAutoConfiguration {
    
    @Configuration
	@ConditionalOnRefreshScope
	protected static class RefreshableEurekaClientConfiguration {
     @Autowired
		private ApplicationContext context;

		@Autowired
		private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;

		@Bean(destroyMethod = "shutdown")
		@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
		@org.springframework.cloud.context.config.annotation.RefreshScope
		@Lazy
		public EurekaClient eurekaClient(
                     ApplicationInfoManager manager, 
                     EurekaClientConfig config, 
                     EurekaInstanceConfig instance,
                    @Autowired(required = false) HealthCheckHandler healthCheckHandler) 
            {
                  
			ApplicationInfoManager appManager;
			if(AopUtils.isAopProxy(manager)) {
				appManager = ProxyUtils.getTargetObject(manager);
			} else {
				appManager = manager;
			}
                    // ********************************************
                    //                  入口跟踪  
                    // ********************************************
			CloudEurekaClient cloudEurekaClient = new CloudEurekaClient(appManager, config, this.optionalArgs,
					this.context);
			cloudEurekaClient.registerHealthCheck(healthCheckHandler);
			return cloudEurekaClient;
		} // end eurekaClient
    }// end static class   RefreshableEurekaClientConfiguration
    
}
```
### (2).CloudEurekaClient
```
public class CloudEurekaClient extends DiscoveryClient {

    public CloudEurekaClient(
                 ApplicationInfoManager applicationInfoManager,
                 EurekaClientConfig config,
                 AbstractDiscoveryClientOptionalArgs<?> args,
                 ApplicationEventPublisher publisher) {
             // *****************************************
             //            重要入口
             // *****************************************
		super(applicationInfoManager, config, args);
  
		this.applicationInfoManager = applicationInfoManager;
		this.publisher = publisher;
		this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class, "eurekaTransport");
		ReflectionUtils.makeAccessible(this.eurekaTransportField);
	} //end CloudEurekaClient
}
```
### (3).EurekaClient
```
public class DiscoveryClient implements EurekaClient {
    
    public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args) {
        // 1. 调用另一个构造器
        this(applicationInfoManager, config, args, new Provider<BackupRegistry>() { // ... })
    }
    
    DiscoveryClient(
           //  应用程序管理
         ApplicationInfoManager applicationInfoManager, 
         // EurekaClient配置
         EurekaClientConfig config, 
         // 其他参数配置
         AbstractDiscoveryClientOptionalArgs args,
         //  备用服务注册提供者
         Provider<BackupRegistry> backupRegistryProvider) {
        
        if (args != null) { // true
            this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;      // null
            this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;    // null
            this.eventListeners.addAll(args.getEventListeners());    // null
            this.preRegistrationHandler = args.preRegistrationHandler;  // null
        } else {
            this.healthCheckCallbackProvider = null;
            this.healthCheckHandlerProvider = null;
            this.preRegistrationHandler = null;
        } 
       
        // 应用实例管理
        this.applicationInfoManager = applicationInfoManager;
        // 当前进程对应的实例信息(进程PID/进程名称/IP端口...)
        InstanceInfo myInfo = applicationInfoManager.getInfo();
        // eureka.client
        clientConfig = config;
        staticClientConfig = clientConfig;
        // 传输层配置项
        transportConfig = config.getTransportConfig();
        // 给实例属性赋值(实例信息)
        instanceInfo = myInfo;
        
        if (myInfo != null) { // true
            // TEST-PROVIDER/172.17.3.67:test-provider:8080
            appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
        } else {
            logger.warn("Setting instanceInfo to a passed in null value");
        }
        this.backupRegistryProvider = backupRegistryProvider;
        // 
        this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
        // *************************************
        //          hold住所有的区域信息
        // *************************************
        localRegionApps.set(new Applications());
        
        // 0
        fetchRegistryGeneration = new AtomicLong(0);
        // null
        remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
        // null
        remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));
        
        if (config.shouldFetchRegistry()) {  //true
            this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }                
        
        if (config.shouldRegisterWithEureka()) { //true
            this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }
        
        // us-east-1
        logger.info("Initializing Eureka in region {}", clientConfig.getRegion());
        
        if (!config.shouldRegisterWithEureka() && 
            !config.shouldFetchRegistry()) {  // false
            logger.info("Client configured to neither register nor query for data.");
            scheduler = null;
            heartbeatExecutor = null;
            cacheRefreshExecutor = null;
            eurekaTransport = null;
            instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

            // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
            // to work with DI'd DiscoveryClient
            DiscoveryManager.getInstance().setDiscoveryClient(this);
            DiscoveryManager.getInstance().setEurekaClientConfig(config);

            initTimestampMs = System.currentTimeMillis();
            logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                    initTimestampMs, this.getApplications().size());

            return;  // no need to setup up an network tasks and we are done
        } //end if         
        
        try {
            // default size of 2 - 1 each for heartbeat and cacheRefresh
            scheduler = Executors.newScheduledThreadPool(2,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

            heartbeatExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            cacheRefreshExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            // 创建传输层
            eurekaTransport = new EurekaTransport();
            // 配置EureakTransport相关信息
            scheduleServerEndpointTask(eurekaTransport, args);
            
            AzToRegionMapper azToRegionMapper;
            if (clientConfig.shouldUseDnsForFetchingServiceUrls()) { //false
                azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
            } else {
                // ...
                // 区域映射
                azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
            }
            
            // null
            if (null != remoteRegionsToFetch.get()) { //false
                azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
            }
            // 创建实例注册区域检测
            instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
        } catch (Throwable e) {
            throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
        } //end try
        
        // ******************************************
        //    从注册中心(Eureka)获取信息到本地
        // ******************************************
        if (
               clientConfig.shouldFetchRegistry()  // true
               && 
               // 1.从注册中心拉取信息到本地
               !fetchRegistry(false)               
            ) { // false
            // 备用注册拉取(不会走该方法)
            fetchRegistryFromBackup();
        }
        
        // false
        if (this.preRegistrationHandler != null) {
            this.preRegistrationHandler.beforeRegistration();
        } //end if
        
        // false
        if (clientConfig.shouldRegisterWithEureka() &&     // true
            clientConfig.shouldEnforceRegistrationAtInit() // false
            ) {
            // 该代码块不会进入
            try {
                if (!register() ) {
                    throw new IllegalStateException("Registration error at startup. Invalid server response.");
                }
            } catch (Throwable th) {
                logger.error("Registration error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        } //end if
        
        // *****************************************
        //  初始化定时任务
        // 1. 缓存刷新
        // 2. 心跳检测
        // 3. 实例注册
        // *****************************************
        initScheduledTasks();
        
        // 不重要部份 ...................
    } //end 构造器
    
    
    // 获取注册中心信息并注册
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();
        try {
            // 第一次进入时,该:applications == localRegionApps.set(new Applications());
            Applications applications = getApplications();
            
            if (
                // false
                clientConfig.shouldDisableDelta()
                // null
                || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                // false
                || forceFullRegistryFetch
                // false
                || (applications == null)
                // true
                || (applications.getRegisteredApplications().size() == 0)
                // -1
                || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
                logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
                logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
                logger.info("Application is null : {}", (applications == null));
                logger.info("Registered Applications size is zero : {}",
                        (applications.getRegisteredApplications().size() == 0));
                logger.info("Application version is -1: {}", (applications.getVersion() == -1));
                // 获取Eureka的所有信息,并存储
                getAndStoreFullRegistry();
            } else {
                // 更新
                getAndUpdateDelta(applications);
            }
            // 
            applications.setAppsHashCode(applications.getReconcileHashCode());
            // 
            logTotalInstances();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        // Notify about cache refresh before updating the instance remote status
        // 发布事件
        onCacheRefreshed();
        updateInstanceRemoteStatus();
        return true;
    } // end fetchRegistry
    
    
    private void getAndStoreFullRegistry() throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        logger.info("Getting all instance registry info from the eureka server");

        Applications apps = null;
        // eurekaTransport.queryClient = com.netflix.discovery.shared.transport.decorator.SessionedEurekaHttpClient
        EurekaHttpResponse<Applications> httpResponse = 
             clientConfig.getRegistryRefreshSingleVipAddress() == null   // true
             ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
             : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            apps = httpResponse.getEntity();
        }
        
        logger.info("The response status is {}", httpResponse.getStatusCode());

        if (apps == null) {  //false
            logger.error("The application is null for some reason. Not storing this information");
            // fetchRegistry自增
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
            // ***************************************
            //     对apps进行处理,并保存到:localRegionApps中
            // ***************************************
            localRegionApps.set(this.filterAndShuffle(apps));
            
            logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
        } else {
            logger.warn("Not updating applications as another thread is updating it already");
        }
    } //end getAndStoreFullRegistry    
    
    
    // 初始化所有的定时任务
    private void initScheduledTasks() {
        
        if (clientConfig.shouldFetchRegistry()) { //true
            // registry cache refresh timer
            // eureka.client.registry-fetch-interval-seconds
            // 表示eureka client间隔多久去拉取服务注册信息，默认为30秒
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            // eureka.client.cache-refresh-executor-exponential-back-off-bound
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            // 缓存刷新
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }

        // true
        if (clientConfig.shouldRegisterWithEureka()) {
            // eureka.instance.lease-renewal-interval-in-seconds
            // 续租时间默认为:30秒
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // Heartbeat timer
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "heartbeat",
                            scheduler,
                            heartbeatExecutor,
                            renewalIntervalInSecs,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new HeartbeatThread()
                    ),
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // ********************************************
            //              实例复制
            // ********************************************
            // InstanceInfo replicator
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize
                    
            
            // ***************************************
            // 创建一个状态监听器
            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                            InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                        // log at warn level if DOWN was involved
                        logger.warn("Saw local status change event {}", statusChangeEvent);
                    } else {
                        logger.info("Saw local status change event {}", statusChangeEvent);
                    }
                    // 调用实例信息复制器的进行更新
                    instanceInfoReplicator.onDemandUpdate();
                }
            };
            
            // 
            if (clientConfig.shouldOnDemandUpdateStatusChange()) { //true
                // 注册监听器
                // 当有statusChangeListener时,进行处理
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }
            // 实例复制器启动
            // 默认为40秒
            // eureka.instance.initial-instance-info-replication-interval-seconds
            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    } //end initScheduledTasks
    
}
```
