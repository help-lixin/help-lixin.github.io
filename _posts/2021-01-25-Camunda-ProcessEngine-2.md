---
layout: post
title: 'Camunda ProcessEngine初始化过程(六)'
date: 2021-01-25
author: 李新
tags: Camunda
---

### (1). 概述
> 在前面,分析到:ProcessEngine.buildProcessEngine就没有继续往下分析,
> 在这一小节,我将对:ProcessEngine.buildProcessEngine方法进入深剖.

### (2). ProcessEngine.buildProcessEngine
> 在这里,只针对init方法中,我认为比较重要(Camunda代码写得优秀)的地方进行剖析. 

```
public ProcessEngine buildProcessEngine() {
	// 3. 调用init()方法
	init();
	processEngine = new ProcessEngineImpl(this);
	invokePostProcessEngineBuild(processEngine);
	return processEngine;
}
```
### (3). ProcessEngine.init
> init方法很复杂,我只挑几个重要的点进行分析.   
> 1. initDataSource(数据源初始化).    
> 2. initCommandExecutors(命令执行器初始化,典型的:责任链模式和命令模式).   
> 3. initEventHandlers(事件处理器,典型的:观察者模式).    
> 4. initDeployers(流程部署/流程解析的入口[BpmnParse]),这一部份也不会深剖,会另开一篇来分析.   

```
protected void init() {
	// Camunda的新功能,允许用户对PorcessEngine进行定制
	// 类似于Spring中定义的一些回调函数.
	// 扩展:ProcessEnginePlugin接口
    invokePreInit();
	// 设置默认的字符串
    initDefaultCharset();
    initHistoryLevel();
	
    initHistoryEventProducer();
    initCmmnHistoryEventProducer();
    initDmnHistoryEventProducer();
    initHistoryEventHandler();
    initExpressionManager();
    initBeans();
    initArtifactFactory();
    initFormEngines();
    initFormTypes();
    initFormFieldValidators();
    initScripting();
    initDmnEngine();
    initBusinessCalendarManager();
    initCommandContextFactory();
    initTransactionContextFactory();

    // Database type needs to be detected before CommandExecutors are initialized
	// *******************************************************************
	// 4. 数据源初始化
	// *******************************************************************
    initDataSource();
	
	// *******************************************************************
	// 5. 命令执行器初始化
	// *******************************************************************
    initCommandExecutors();
	
	// xxxService初始化
    initServices();
	// ID生成策略
    initIdGenerator();
    initFailedJobCommandFactory();
	// *******************************************************************
	// 7. 初始化部署
	// *******************************************************************
    initDeployers();
    initJobProvider();
    initExternalTaskPriorityProvider();
    initBatchHandlers();
    initJobExecutor();
	
	// *******************************************************************
	// 事务管理器
	// *******************************************************************
    initTransactionFactory();
	
	// MyBatis
    initSqlSessionFactory();
    initIdentityProviderSessionFactory();
    initSessionFactories();
    initValueTypeResolver();
    initTypeValidator();
    initSerialization();
    initJpa();
    initDelegateInterceptor();
	// *******************************************************************
	// 6. 事件处理器初始化
	// *******************************************************************
    initEventHandlers();
    initProcessApplicationManager();
    initCorrelationHandler();
    initConditionHandler();
    initIncidentHandlers();
    initPasswordDigest();
    initDeploymentRegistration();
    initDeploymentHandlerFactory();
    initResourceAuthorizationProvider();
    initPermissionProvider();
    initHostName();
    initMetrics();
    initTelemetry();
    initMigration();
    initCommandCheckers();
    initDefaultUserPermissionForTask();
    initHistoryRemovalTime();
    initHistoryCleanup();
    initInvocationsPerBatchJobByBatchType();
    initAdminUser();
    initAdminGroups();
    initPasswordPolicy();
    invokePostInit();
}// end init


protected void invokePreInit() {
    for (ProcessEnginePlugin plugin : processEnginePlugins) {
      LOG.pluginActivated(plugin.toString(), getProcessEngineName());
      plugin.preInit(this);
    }
} //end invokePreInit

protected void invokePostInit() {
    for (ProcessEnginePlugin plugin : processEnginePlugins) {
      plugin.postInit(this);
    }
} //end invokePostInit
```
### (4). ProcessEngineConfigurationImpl.initDataSource
> 从这里的代码分析能看出来,连接池是复用了MyBatis的.从性能角度着想:建议自己配置:DataSource. 

```
// domain
protected String databaseType;
protected String databaseVendor;
protected String databaseVersion;
protected String databaseSchemaUpdate = DB_SCHEMA_UPDATE_FALSE;
protected String jdbcDriver = "org.h2.Driver";
protected String jdbcUrl = "jdbc:h2:tcp://localhost/activiti";
protected String jdbcUsername = "sa";
protected String jdbcPassword = "";
protected String dataSourceJndiName = null;
protected int jdbcMaxActiveConnections;
protected int jdbcMaxIdleConnections;
protected int jdbcMaxCheckoutTime;
protected int jdbcMaxWaitTime;
protected boolean jdbcPingEnabled = false;
protected String jdbcPingQuery = null;
protected int jdbcPingConnectionNotUsedFor;
protected DataSource dataSource;


// initDataSource
protected void initDataSource() {
	if (dataSource == null) {
	   // 判断数据源是否为:jndi,如果是jndi配置,则通过:InitialContext去获取:DataSource
	  if (dataSourceJndiName != null) {
		try {
		  dataSource = (DataSource) new InitialContext().lookup(dataSourceJndiName);
		} catch (Exception e) {
		  throw new ProcessEngineException("couldn't lookup datasource from " + dataSourceJndiName + ": " + e.getMessage(), e);
		}
	  } else if (jdbcUrl != null) {  // 查看是否有配置jdbcUrl
		if ((jdbcDriver == null) || (jdbcUrl == null) || (jdbcUsername == null)) {
		  throw new ProcessEngineException("DataSource or JDBC properties have to be specified in a process engine configuration");
		}
		
		// 在这里引用的是:org.apache.ibatis.datasource.pooled.PooledDataSource
		// 内部实则为:org.apache.ibatis.datasource.unpooled.UnpooledDataSource.UnpooledDataSource
		// 这时要注意了:建议换成自己的:dataSource
		PooledDataSource pooledDataSource =
			new PooledDataSource(ReflectUtil.getClassLoader(), jdbcDriver, jdbcUrl, jdbcUsername, jdbcPassword);

		if (jdbcMaxActiveConnections > 0) {
		  pooledDataSource.setPoolMaximumActiveConnections(jdbcMaxActiveConnections);
		}
		if (jdbcMaxIdleConnections > 0) {
		  pooledDataSource.setPoolMaximumIdleConnections(jdbcMaxIdleConnections);
		}
		if (jdbcMaxCheckoutTime > 0) {
		  pooledDataSource.setPoolMaximumCheckoutTime(jdbcMaxCheckoutTime);
		}
		if (jdbcMaxWaitTime > 0) {
		  pooledDataSource.setPoolTimeToWait(jdbcMaxWaitTime);
		}
		if (jdbcPingEnabled == true) {
		  pooledDataSource.setPoolPingEnabled(true);
		  if (jdbcPingQuery != null) {
			pooledDataSource.setPoolPingQuery(jdbcPingQuery);
		  }
		  pooledDataSource.setPoolPingConnectionsNotUsedFor(jdbcPingConnectionNotUsedFor);
		}
		dataSource = pooledDataSource;
	  }

	  if (dataSource instanceof PooledDataSource) {
		// ACT-233: connection pool of Ibatis is not properely initialized if this is not called!
		((PooledDataSource) dataSource).forceCloseAll();
	  }
	}// end if
	
	// 这一步是根据Connection获得所连接的是:MySQL/MariaDB/PostgreSQL/CockroachDB
	if (databaseType == null) {
	  initDatabaseType();
	}
}// end initDataSource
```
### (5). ProcessEngineConfigurationImpl.initCommandExecutors
```
// domain
protected List<CommandInterceptor> customPreCommandInterceptorsTxRequired;
protected List<CommandInterceptor> customPostCommandInterceptorsTxRequired;
protected List<CommandInterceptor> commandInterceptorsTxRequired;

protected List<CommandInterceptor> customPreCommandInterceptorsTxRequiresNew;
protected List<CommandInterceptor> customPostCommandInterceptorsTxRequiresNew;
protected List<CommandInterceptor> commandInterceptorsTxRequiresNew;

protected CommandExecutor commandExecutorTxRequiresNew;
protected CommandExecutor commandExecutorSchemaOperations;
protected CommandExecutor commandExecutorTxRequired;
protected CommandInterceptor actualCommandExecutor;


// methods
protected void initCommandExecutors() {
	// 5.1 创建:CommandExecutorImpl
    initActualCommandExecutor();
	
	// 5.2 创建Command拦截器(针对Require传播行为)
    initCommandInterceptorsTxRequired();
	// 5.3 对5.2步产生的List<CommandInterceptor>组合成链表
    initCommandExecutorTxRequired();
	
	// 5.4 创建Command拦截器(针对RequireNew传播行为)
    initCommandInterceptorsTxRequiresNew();
	// 5.5 对5.4步产生的List<CommandInterceptor>组合成链表
    initCommandExecutorTxRequiresNew();
	
    initCommandExecutorDbSchemaOperations();
}// end initCommandExecutors

// 5.1 创建:CommandExecutorImpl
protected void initActualCommandExecutor() {
	// 创建真实的Command实现类
    actualCommandExecutor = new CommandExecutorImpl();
} //end initActualCommandExecutor


// 5.2 创建Command拦截器
// 这一步主要是配置:CommandInterceptor
// 可以简化理解为这样: 
// commandInterceptorsTxRequired = new ArrayList(
//   customPreCommandInterceptorsTxRequired ,
//   getDefaultCommandInterceptorsTxRequired() , 
//   customPostCommandInterceptorsTxRequired ,
//   actualCommandExecutor
// );
protected void initCommandInterceptorsTxRequired() {
    if (commandInterceptorsTxRequired == null) { // true
	
	   // customPreCommandInterceptors
      if (customPreCommandInterceptorsTxRequired != null) {
        commandInterceptorsTxRequired = new ArrayList<>(customPreCommandInterceptorsTxRequired);
      } else {
		//  初始化(commandInterceptorsTxRequired)
        commandInterceptorsTxRequired = new ArrayList<>();
      }
	  
	  // defaultCommandInterceptors
      commandInterceptorsTxRequired.addAll(getDefaultCommandInterceptorsTxRequired());
	  
	  // postCommandInterceptors
      if (customPostCommandInterceptorsTxRequired != null) {
        commandInterceptorsTxRequired.addAll(customPostCommandInterceptorsTxRequired);
      }
	  
	  // actualCommandExecutor
      commandInterceptorsTxRequired.add(actualCommandExecutor);
    }
}//end initCommandInterceptorsTxRequired


// 5.3 对5.2步产生的:List<CommandInterceptor>(commandInterceptorsTxRequired)集合,组合成链表
// CommandInterceptor是一个链表结构.内部持有下一个:CommandExecutor
protected void initCommandExecutorTxRequired() {
    if (commandExecutorTxRequired == null) {  //commandExecutorTxRequired是链表结构的:CommandInterceptor 
      commandExecutorTxRequired = initInterceptorChain(commandInterceptorsTxRequired);
    }
}// end initCommandExecutorTxRequired

protected CommandInterceptor initInterceptorChain(List<CommandInterceptor> chain) {
	// 集合不能为空
    if (chain == null || chain.isEmpty()) {
      throw new ProcessEngineException("invalid command interceptor chain configuration: " + chain);
    }
	
	// *****************************************************************
	// 把集合,转换成链表.
	// *****************************************************************
    for (int i = 0; i < chain.size() - 1; i++) {
      chain.get(i).setNext(chain.get(i + 1));
    }
	
	// 返回链表的首元素:CommandInterceptor
    return chain.get(0);
}// end initInterceptorChain



protected void initCommandInterceptorsTxRequiresNew() {
    if (commandInterceptorsTxRequiresNew == null) {
      if (customPreCommandInterceptorsTxRequiresNew != null) {
        commandInterceptorsTxRequiresNew = new ArrayList<>(customPreCommandInterceptorsTxRequiresNew);
      } else {
        commandInterceptorsTxRequiresNew = new ArrayList<>();
      }
      commandInterceptorsTxRequiresNew.addAll(getDefaultCommandInterceptorsTxRequiresNew());
      if (customPostCommandInterceptorsTxRequiresNew != null) {
        commandInterceptorsTxRequiresNew.addAll(customPostCommandInterceptorsTxRequiresNew);
      }
      commandInterceptorsTxRequiresNew.add(actualCommandExecutor);
    }
} //end initCommandInterceptorsTxRequiresNew

protected void initCommandExecutorTxRequiresNew() {
    if (commandExecutorTxRequiresNew == null) {
      commandExecutorTxRequiresNew = initInterceptorChain(commandInterceptorsTxRequiresNew);
    }
}//end initCommandExecutorTxRequiresNew

 protected void initCommandExecutorDbSchemaOperations() {
    if (commandExecutorSchemaOperations == null) {
      commandExecutorSchemaOperations = commandExecutorTxRequired;
    }
} // end initCommandExecutorDbSchemaOperations
```
### (6). ProcessEngineConfigurationImpl.initEventHandlers
```

// domain
protected Map<String, EventHandler> eventHandlers;
protected List<EventHandler> customEventHandlers;

// methods
protected void initEventHandlers() {
    if (eventHandlers == null) {
	  //  对:eventHandlers进行初始化
      eventHandlers = new HashMap<>();

	  // 创建:SignalEventHandler
      SignalEventHandler signalEventHander = new SignalEventHandler();
      eventHandlers.put(signalEventHander.getEventHandlerType(), signalEventHander);

      // 创建:CompensationEventHandler
      CompensationEventHandler compensationEventHandler = new CompensationEventHandler();
      eventHandlers.put(compensationEventHandler.getEventHandlerType(), compensationEventHandler);

      // 创建:EventType.MESSAGE
      EventHandler messageEventHandler = new EventHandlerImpl(EventType.MESSAGE);
      eventHandlers.put(messageEventHandler.getEventHandlerType(), messageEventHandler);

      // 创建:ConditionalEventHandler
      EventHandler conditionalEventHandler = new ConditionalEventHandler();
      eventHandlers.put(conditionalEventHandler.getEventHandlerType(), conditionalEventHandler);
    } // end if
	
	// 配置自定义的EventHandler
	// 要注意:要是想覆盖系统的Event,只需要自定义的EventHandler的Type与系统的相当即可.
    if (customEventHandlers != null) {
      for (EventHandler eventHandler : customEventHandlers) {
        eventHandlers.put(eventHandler.getEventHandlerType(), eventHandler);
      }
    }// end if
} // end initEventHandlers
```

### (7). ProcessEngineConfigurationImpl.initDeployers
```
// domain
protected List<Deployer> customPreDeployers;
protected List<Deployer> customPostDeployers;
protected List<Deployer> deployers;
protected DeploymentCache deploymentCache;

protected CacheFactory cacheFactory;
protected int cacheCapacity = 1000;
protected boolean enableFetchProcessDefinitionDescription = true;

// methods
protected void initDeployers() {
    if (this.deployers == null) {
	   // 初始化:deployers
      this.deployers = new ArrayList<>();
	  
	  // 前置自定义部署组件
      if (customPreDeployers != null) {
        this.deployers.addAll(customPreDeployers);
      }
	  
	  // 默认的部署组件
      this.deployers.addAll(getDefaultDeployers());
	  
	  // 后置自定义部署组件
      if (customPostDeployers != null) {
        this.deployers.addAll(customPostDeployers);
      }
    }// end if
	
	
    if (deploymentCache == null) { 
      List<Deployer> deployers = new ArrayList<>();
      if (customPreDeployers != null) {
        deployers.addAll(customPreDeployers);
      }
      deployers.addAll(getDefaultDeployers());
      if (customPostDeployers != null) {
        deployers.addAll(customPostDeployers);
      }
	  // 初始化缓存工厂(工厂模式)
      initCacheFactory();
      deploymentCache = new DeploymentCache(cacheFactory, cacheCapacity);
      deploymentCache.setDeployers(deployers);
    } // end if
}

protected Collection<? extends Deployer> getDefaultDeployers() {
    List<Deployer> defaultDeployers = new ArrayList<>();

	// getBpmnDeployer()
    BpmnDeployer bpmnDeployer = getBpmnDeployer();
    defaultDeployers.add(bpmnDeployer);

	// 以下是cmmn/dmn了
    if (isCmmnEnabled()) {
      CmmnDeployer cmmnDeployer = getCmmnDeployer();
      defaultDeployers.add(cmmnDeployer);
    }

    if (isDmnEnabled()) {
      DecisionRequirementsDefinitionDeployer decisionRequirementsDefinitionDeployer = getDecisionRequirementsDefinitionDeployer();
      DecisionDefinitionDeployer decisionDefinitionDeployer = getDecisionDefinitionDeployer();
      // the DecisionRequirementsDefinition cacheDeployer must be before the DecisionDefinitionDeployer
      defaultDeployers.add(decisionRequirementsDefinitionDeployer);
      defaultDeployers.add(decisionDefinitionDeployer);
    }

    return defaultDeployers;
}// end

protected BpmnDeployer getBpmnDeployer() {
    BpmnDeployer bpmnDeployer = new BpmnDeployer();
    bpmnDeployer.setExpressionManager(expressionManager);
    bpmnDeployer.setIdGenerator(idGenerator);
    if (bpmnParseFactory == null) {
      bpmnParseFactory = new DefaultBpmnParseFactory();
    }
	// ***************************************************************************
	// 创建:BpmnParser,它是流程定义解析的入口.内容比较多,会另外再抽一节进行剖析
	// ***************************************************************************
    BpmnParser bpmnParser = new BpmnParser(expressionManager, bpmnParseFactory);
    if (preParseListeners != null) {
      bpmnParser.getParseListeners().addAll(preParseListeners);
    }
    bpmnParser.getParseListeners().addAll(getDefaultBPMNParseListeners());
    if (postParseListeners != null) {
      bpmnParser.getParseListeners().addAll(postParseListeners);
    }
    bpmnDeployer.setBpmnParser(bpmnParser);
    return bpmnDeployer;
} // end 

```
### (8). CommandExecutor类结构图解
!["CommandExecutor类结构图解"](/assets/camunda/imgs/CommandExecutor.jpg)
### (9). EventHandler类结构图解
!["EventHandler类结构图解"](/assets/camunda/imgs/EventHandler.jpg)
