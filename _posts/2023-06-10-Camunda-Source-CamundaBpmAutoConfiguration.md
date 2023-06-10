---
layout: post
title: 'Camunda源码剖析之CamundaBpmAutoConfiguration(一)' 
date: 2023-06-10
author: 李新
tags:  Camunda
---

### (1). 概述
对于Spring的项目来说,我们看代码的入口为start即可.

### (2). META-INF/spring.factories
```
# AutoConfigurations
org.springframework.boot.autoconfigure.EnableAutoConfiguration=org.camunda.bpm.spring.boot.starter.CamundaBpmAutoConfiguration

# Application Listeners
org.springframework.context.ApplicationListener=org.camunda.bpm.spring.boot.starter.runlistener.PropertiesListener

```
### (2). CamundaBpmAutoConfiguration注解
```
@EnableConfigurationProperties({
  // **********************************************************
  // 1. 启用配置类,与camunda相关的配置类都在这个类进行配置.
  // **********************************************************
  CamundaBpmProperties.class,
  ManagementProperties.class
})

// **********************************************************
// 2. 导入其它的配置类
// **********************************************************
@Import({
  CamundaBpmConfiguration.class,
  CamundaBpmActuatorConfiguration.class,
  CamundaBpmPluginConfiguration.class,
  CamundaBpmTelemetryConfiguration.class,
  SpringProcessEngineServicesConfiguration.class
})
@Configuration

// **********************************************************
// 3. camunda.bpm.enabled=true
// matchIfMissing: 缺少上面的配置(camunda.bpm.enabled),是否生效
// **********************************************************
@ConditionalOnProperty(prefix = CamundaBpmProperties.PREFIX, name = "enabled", matchIfMissing = true)
@AutoConfigureAfter(HibernateJpaAutoConfiguration.class)
public class CamundaBpmAutoConfiguration {
	// ... ... 
}
```
### (4). CamundaBpmAutoConfiguration
```
public class CamundaBpmAutoConfiguration {
	// BPMN版本管理
	@Bean
	public CamundaBpmVersion camundaBpmVersion() {
		return new CamundaBpmVersion();
	}
	
	// 监听:ApplicationReadyEvent/ContextClosedEvent事件,并重新发布成:
	// ProcessApplicationStartedEvent
	// ProcessApplicationStoppedEvent
	@Bean
	public ProcessApplicationEventPublisher processApplicationEventPublisher(ApplicationEventPublisher publisher) {
		return new ProcessApplicationEventPublisher(publisher);
	}
}
```
### (5). CamundaBpmAutoConfiguration$CamundaBpmAutoConfiguration
```
public class CamundaBpmAutoConfiguration {
  
  @Configuration
  class ProcessEngineConfigurationImplDependingConfiguration {
    
	
	// ******************************************************************
	// 1. 注入:ProcessEngineConfigurationImpl,先不管这个类是在哪初始化的,后面再剖析
	// ******************************************************************
    @Autowired
    protected ProcessEngineConfigurationImpl processEngineConfigurationImpl;

	
	// ******************************************************************
	// 2. 通过:ProcessEngineFactoryBean来创建:ProcessEngine实例
	// ******************************************************************
    @Bean
    public ProcessEngineFactoryBean processEngineFactoryBean() {
      final ProcessEngineFactoryBean factoryBean = new ProcessEngineFactoryBean();
      factoryBean.setProcessEngineConfiguration(processEngineConfigurationImpl);
      return factoryBean;
    }

	// ******************************************************************
	// 配置:CommandExecutor到Spring容器里.
	// ******************************************************************
    @Bean
    @Primary
    public CommandExecutor commandExecutorTxRequired() {
      return processEngineConfigurationImpl.getCommandExecutorTxRequired();
    }

    @Bean
    public CommandExecutor commandExecutorTxRequiresNew() {
      return processEngineConfigurationImpl.getCommandExecutorTxRequiresNew();
    }

    @Bean
    public CommandExecutor commandExecutorSchemaOperations() {
      return processEngineConfigurationImpl.getCommandExecutorSchemaOperations();
    }
  } // end ProcessEngineConfigurationImplDependingConfiguration
  
}  
```
### (6). 总结
CamundaBpmAutoConfiguration类主要是配置了一堆的Bean,并不太好分析,原因是什么呢?因为:不太清楚Camunda的模型,如果过度分析,会造成分析的文章过长,怕读者理解不了. 
