---
layout: post
title: 'Activiti源码(ProcessEngineConfigurationImpl.initConfigurators)'
date: 2020-01-24
author: 李新
tags: Activiti
---

### (1).ProcessEngineConfigurationImpl.buildProcessEngine
```
public ProcessEngine buildProcessEngine() {
    // 初始化
    init();
    ProcessEngineImpl processEngine = new ProcessEngineImpl(this);

    // trigger build of Activiti 5 Engine
    if (isActiviti5CompatibilityEnabled && activiti5CompatibilityHandler != null) {
      Context.setProcessEngineConfiguration(processEngine.getProcessEngineConfiguration());
      activiti5CompatibilityHandler.getRawProcessEngine();
    }

    postProcessEngineInitialisation();

    return processEngine;
}
```
### (2).ProcessEngineConfigurationImpl.init
```
public void init() {
    // 初始化配置器
    initConfigurators();    
    // 调用配置器初始化之前
    configuratorsBeforeInit();
    // 初始化流程图片生成器
    initProcessDiagramGenerator();
    // 初始化历史记录归档级别
    initHistoryLevel();
    // 初始化表达式管理器
    initExpressionManager();
    // 初始化数据源
    if (usingRelationalDatabase) {
      initDataSource();
    }
    // 
    initAgendaFactory();
    initHelpers();
    // 初始化变量类型管理
    initVariableTypes();
    // 初始化可以管理的bean
    initBeans();
    // 初始化表单引擎
    initFormEngines();
    // 初始化表单类型
    initFormTypes();
    // 初始化Script引擎
    initScriptingEngines();
    // 初始化时间类,主要负责提供设置当前时间等
    initClock();
    // 初始化日期管理器
    initBusinessCalendarManager();
    // 初始化命令上下文管理工厂
    initCommandContextFactory();
    // 初始化事务管理上下文
    initTransactionContextFactory();
    // 初始化执行器
    initCommandExecutors();
    // 初始化所有的service
    initServices();
    // 初始化ID生成器
    initIdGenerator();
    // 初始化BehaviorFactory
    initBehaviorFactory();
    // 初始化监听器工厂
    initListenerFactory();
    // 初始化bpmn解析
    initBpmnParser();
    // 初始化流程定义缓存
    initProcessDefinitionCache();
    // 初始化流程定义信息缓存
    initProcessDefinitionInfoCache();
    // 
    initKnowledgeBaseCache();
    // 初始化定时任务处理器
    initJobHandlers();
    // 初始化定时任务管理
    initJobManager();
    // 初始化异步执行器
    initAsyncExecutor();
    
    // 初始化事管理工厂
    initTransactionFactory();

    if (usingRelationalDatabase) {
      // 初始化 SqlSessionFactory
      initSqlSessionFactory();
    }

    // 初始化Session
    initSessionFactories();
    // 
    initDataManagers();
    // 实体管理
    initEntityManagers();
    // 历史管理
    initHistoryManager();
    // JPA
    initJpa();
    // 初始化部署
    initDeployers();
    // 
    initDelegateInterceptor();
    initEventHandlers();
    initFailedJobCommandFactory();
    // 事件分发器
    initEventDispatcher();
    // 流程验证
    initProcessValidator();
    // DB记录事件历程
    initDatabaseEventLogging();
    initActiviti5CompatibilityHandler();
    configuratorsAfterInit();
} //end init
```
### (3).ProcessEngineConfigurationImpl.initConfigurators
```
// ==================================field===================
protected boolean enableConfiguratorServiceLoader = true; // Enabled by default. In certain environments this should be set to false (eg osgi)
protected List<ProcessEngineConfigurator> allConfigurators; // Including auto-discovered configurators
protected List<ProcessEngineConfigurator> configurators; // The injected configurators

public void initConfigurators() {
    // 初始化所有的配置容器
    allConfigurators = new ArrayList<ProcessEngineConfigurator>();
     // 如果:configurators不为空,则加入.
    // 预留:configurators自由配置.
    // <list>
    //   <ref bean="customerProcessEngineConfigurator"/>
    // </list>
    if (configurators != null) {   //configurators=null
      for (ProcessEngineConfigurator configurator : configurators) {
        allConfigurators.add(configurator);
      }
    }
    
     // enableConfiguratorServiceLoader = true
     if (enableConfiguratorServiceLoader) {  //true 
      ClassLoader classLoader = getClassLoader();
      if (classLoader == null) {
        classLoader = ReflectUtil.getClassLoader();
      }
      // 通过Java SPI获得:ProcessEngineConfigurator接口的实现
      ServiceLoader<ProcessEngineConfigurator> configuratorServiceLoader = ServiceLoader.load(ProcessEngineConfigurator.class, classLoader);
      int nrOfServiceLoadedConfigurators = 0;
      for (ProcessEngineConfigurator configurator : configuratorServiceLoader) {
        allConfigurators.add(configurator);
        nrOfServiceLoadedConfigurators++;
      }

      if (nrOfServiceLoadedConfigurators > 0) {
        log.info("Found {} auto-discoverable Process Engine Configurator{}", nrOfServiceLoadedConfigurators++, nrOfServiceLoadedConfigurators > 1 ? "s" : "");
      }
      
      // ProcessEngineConfigurator有实现情况下
      if (!allConfigurators.isEmpty()) {  
       // ProcessEngineConfigurator有优先级,根据优先级进行排序
        Collections.sort(allConfigurators, new Comparator<ProcessEngineConfigurator>() {
          @Override
          public int compare(ProcessEngineConfigurator configurator1, ProcessEngineConfigurator configurator2) {
            int priority1 = configurator1.getPriority();
            int priority2 = configurator2.getPriority();

            if (priority1 < priority2) {
              return -1;
            } else if (priority1 > priority2) {
              return 1;
            }
            return 0;
          }
        });

        log.info("Found {} Process Engine Configurators in total:", allConfigurators.size());
        for (ProcessEngineConfigurator configurator : allConfigurators) {
          log.info("{} (priority:{})", configurator.getClass(), configurator.getPriority());
        }  // end for
      }  //end if( !allConfigurators.isEmpty() )
    } //end if enableConfiguratorServiceLoader
} //end initConfigurators
```