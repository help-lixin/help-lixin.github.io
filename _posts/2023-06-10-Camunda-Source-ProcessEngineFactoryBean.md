---
layout: post
title: 'Camunda源码剖析之ProcessEngineFactoryBean(四)' 
date: 2023-06-10
author: 李新
tags:  Camunda
---

### (1). 概述
在前面剖析CamundaBpmAutoConfiguration类时,有说过:ProcessEngineFactoryBean很重要,因为,它最终会创建:ProcessEngine,其实,可以这样理解,前面那些配置类,大部份是为创建ProcessEngine做铺垫. 

### (2). ProcessEngineFactoryBean
```
public class ProcessEngineFactoryBean 
	  // ************************************************************************************** 
	  // FactoryBean是Spring定义的工厂类,用于创建Bean
	  // ************************************************************************************** 
      implements FactoryBean<ProcessEngine>, 
	             DisposableBean, 
				 ApplicationContextAware {
	
	public ProcessEngine getObject() throws Exception {
	    if (processEngine == null) {
		  // 配置表达式管理
	      initializeExpressionManager();
		  // 配置事务扩展
	      initializeTransactionExternallyManaged();
	      // ************************************************************************************** 
		  // 最终的做法是:通过buildProcessEngine方法创建:ProcessEngine
		  // ************************************************************************************** 
	      processEngine = (ProcessEngineImpl) processEngineConfiguration.buildProcessEngine();
	    }
	
	    return processEngine;
	  }
}	
```
### (3). 总结
通过源码能看出来:ProcessEngine创建是由:ProcessEngineConfigurationImpl.buildProcessEngine构建而来. 
