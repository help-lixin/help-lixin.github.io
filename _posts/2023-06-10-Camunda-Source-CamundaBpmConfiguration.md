---
layout: post
title: 'Camunda源码剖析之CamundaBpmConfiguration(三)' 
date: 2023-06-10
author: 李新
tags:  Camunda
---

### (1). 概述

在这一小篇,主要了解:ProcessEngineConfigurationImpl的初始化,在了解该类之前,我们要先了解一个接口:ProcessEnginePlugin.  

### (2). ProcessEnginePlugin接口能力
```
public interface ProcessEnginePlugin {
	// 创建ProcessEngine之前,调用:preInit方法,允许对:ProcessEngineConfigurationImpl进行扩展.
	void preInit(ProcessEngineConfigurationImpl processEngineConfiguration);
	// 创建ProcessEngine后,调用:postInit方法,允许对:ProcessEngineConfigurationImpl进行扩展.
	void postInit(ProcessEngineConfigurationImpl processEngineConfiguration);
	// 在生成ProcessEngine实例后,调用方法.
	void postProcessEngineBuild(ProcessEngine processEngine);
}	
```
### (3). CamundaBpmConfiguration配置类
```
public class CamundaBpmConfiguration {

  // ********************************************************************************************
  // 1. 创建:ProcessEngineConfigurationImpl对象
  // ********************************************************************************************
  @Bean
  @ConditionalOnMissingBean(ProcessEngineConfigurationImpl.class)
  public ProcessEngineConfigurationImpl processEngineConfigurationImpl(List<ProcessEnginePlugin> processEnginePlugins) {
    final SpringProcessEngineConfiguration configuration = CamundaSpringBootUtil.springProcessEngineConfiguration();
    configuration.getProcessEnginePlugins().add(new CompositeProcessEnginePlugin(processEnginePlugins));
    return configuration;
  }

  // ********************************************************************************************
  // 2. CamundaProcessEngineConfiguration实际为:ProcessEnginePlugin接口的实现
  // ********************************************************************************************
  @Bean
  @ConditionalOnMissingBean(DefaultProcessEngineConfiguration.class)
  public static CamundaProcessEngineConfiguration camundaProcessEngineConfiguration() {
    return new DefaultProcessEngineConfiguration();
  }

  @Bean
  @ConditionalOnMissingBean(CamundaDatasourceConfiguration.class)
  public static CamundaDatasourceConfiguration camundaDatasourceConfiguration() {
    return new DefaultDatasourceConfiguration();
  }

  @Bean
  @ConditionalOnBean(name = "entityManagerFactory")
  @ConditionalOnMissingBean(CamundaJpaConfiguration.class)
  @ConditionalOnProperty(prefix = "camunda.bpm.jpa", name = "enabled", havingValue = "true", matchIfMissing = true)
  public static CamundaJpaConfiguration camundaJpaConfiguration() {
    return new DefaultJpaConfiguration();
  }

  @Bean
  @ConditionalOnMissingBean(CamundaJobConfiguration.class)
  @ConditionalOnProperty(prefix = "camunda.bpm.job-execution", name = "enabled", havingValue = "true", matchIfMissing = true)
  public static CamundaJobConfiguration camundaJobConfiguration() {
    return new DefaultJobConfiguration();
  }

  @Bean
  @ConditionalOnMissingBean(CamundaHistoryConfiguration.class)
  public static CamundaHistoryConfiguration camundaHistoryConfiguration() {
    return new DefaultHistoryConfiguration();
  }

  @Bean
  @ConditionalOnMissingBean(CamundaMetricsConfiguration.class)
  public static CamundaMetricsConfiguration camundaMetricsConfiguration() {
    return new DefaultMetricsConfiguration();
  }

  //TODO to be removed within CAM-8108
  @Bean(name = "historyLevelAutoConfiguration")
  @ConditionalOnMissingBean(CamundaHistoryLevelAutoHandlingConfiguration.class)
  @ConditionalOnProperty(prefix = "camunda.bpm", name = "history-level", havingValue = "auto", matchIfMissing = false)
  @Conditional(NeedsHistoryAutoConfigurationCondition.class)
  public static CamundaHistoryLevelAutoHandlingConfiguration historyLevelAutoHandlingConfiguration() {
    return new DefaultHistoryLevelAutoHandlingConfiguration();
  }

  //TODO to be removed within CAM-8108
  @Bean(name = "historyLevelDeterminator")
  @ConditionalOnMissingBean(name = { "camundaBpmJdbcTemplate", "historyLevelDeterminator" })
  @ConditionalOnBean(name = "historyLevelAutoConfiguration")
  public static HistoryLevelDeterminator historyLevelDeterminator(CamundaBpmProperties camundaBpmProperties, JdbcTemplate jdbcTemplate) {
    return createHistoryLevelDeterminator(camundaBpmProperties, jdbcTemplate);
  }

  //TODO to be removed within CAM-8108
  @Bean(name = "historyLevelDeterminator")
  @ConditionalOnBean(name = { "camundaBpmJdbcTemplate", "historyLevelAutoConfiguration", "historyLevelDeterminator" })
  @ConditionalOnMissingBean(name = "historyLevelDeterminator")
  public static HistoryLevelDeterminator historyLevelDeterminatorMultiDatabase(CamundaBpmProperties camundaBpmProperties,
      @Qualifier("camundaBpmJdbcTemplate") JdbcTemplate jdbcTemplate) {
    return createHistoryLevelDeterminator(camundaBpmProperties, jdbcTemplate);
  }

  @Bean
  @ConditionalOnMissingBean(CamundaAuthorizationConfiguration.class)
  public static CamundaAuthorizationConfiguration camundaAuthorizationConfiguration() {
    return new DefaultAuthorizationConfiguration();
  }

  @Bean
  @ConditionalOnMissingBean(CamundaDeploymentConfiguration.class)
  public static CamundaDeploymentConfiguration camundaDeploymentConfiguration() {
    return new DefaultDeploymentConfiguration();
  }

  @Bean
  public GenericPropertiesConfiguration genericPropertiesConfiguration() {
    return new GenericPropertiesConfiguration();
  }

  @Bean
  @ConditionalOnProperty(prefix = "camunda.bpm.admin-user", name = "id")
  public CreateAdminUserConfiguration createAdminUserConfiguration() {
    return new CreateAdminUserConfiguration();
  }

  @Bean
  @ConditionalOnMissingBean(CamundaFailedJobConfiguration.class)
  public static CamundaFailedJobConfiguration failedJobConfiguration() {
    return new DefaultFailedJobConfiguration();
  }

  @Bean
  @ConditionalOnProperty(prefix = "camunda.bpm.filter", name = "create")
  public CreateFilterConfiguration createFilterConfiguration() {
    return new CreateFilterConfiguration();
  }

  @Bean
  public EventPublisherPlugin eventPublisherPlugin(CamundaBpmProperties properties, ApplicationEventPublisher publisher) {
    return new EventPublisherPlugin(properties.getEventing(), publisher);
  }

  @Bean
  public CamundaIntegrationDeterminator camundaIntegrationDeterminator() {
    return new CamundaIntegrationDeterminator();
  }
}
```
### (3). GenericPropertiesConfiguration
```
@Order(Ordering.DEFAULT_ORDER - 1)
public class GenericPropertiesConfiguration 
		// *****************************************************************************
		// AbstractCamundaConfiguration实际是:SpringProcessEnginePlugin接口的实现类
		// *****************************************************************************
       extends AbstractCamundaConfiguration {

  protected static final SpringBootProcessEngineLogger LOG = SpringBootProcessEngineLogger.LOG;

  @Override
  public void preInit(SpringProcessEngineConfiguration springProcessEngineConfiguration) {
    // GenericProperties就是一个Map
    GenericProperties genericProperties = camundaBpmProperties.getGenericProperties();
    final Map<String, Object> properties = genericProperties.getProperties();

    if (!CollectionUtils.isEmpty(properties)) {
      SpringBootStarterPropertyHelper
          .applyProperties(springProcessEngineConfiguration, properties, genericProperties.isIgnoreUnknownFields());
      LOG.propertiesApplied(genericProperties);
    }
  } // end preInit
  
}

```
### (4). SpringBootStarterPropertyHelper
```
public class SpringBootStarterPropertyHelper {

  protected static final SpringBootProcessEngineLogger LOG = SpringBootProcessEngineLogger.LOG;

  // ***********************************************************************************************
  // T: SpringProcessEngineConfiguration
  // 把sourceMap的信息应用到:SpringProcessEngineConfiguration实例上
  // ***********************************************************************************************
  public static <T> void applyProperties(T target, Map<String, Object> sourceMap, boolean ignoreUnknownFields) {
    ConfigurationPropertySource source = new MapConfigurationPropertySource(sourceMap);
    Binder binder = new Binder(source);
    try {
      if (ignoreUnknownFields) {
        binder.bind(ConfigurationPropertyName.EMPTY, Bindable.ofInstance(target));
      } else {
        binder.bind(ConfigurationPropertyName.EMPTY, Bindable.ofInstance(target), new NoUnboundElementsBindHandler(BindHandler.DEFAULT));
      }
    } catch (Exception e) {
      throw LOG.exceptionDuringBinding(e.getMessage());
    }
  } // end applyProperties
}
```
### (5). 总结
CamundaBpmConfiguration类配置的Bean大多数是:ProcessEnginePlugin接口的实现类,通过随机对ProcessEnginePlugin的实现类进行剖析,内部的原理为:      
1. 创建:ProcessEngineConfigurationImpl对象的实现类(SpringProcessEngineConfiguration),并把该类交给Spring进行托管.   
2. 在初始化ProcessEngine时,调用:ProcessEnginePlugin的方法,对:ProcessEngineConfigurationImpl进行配置. 
3. 一句话理解就是:ProcessEnginePlugin就是对:ProcessEngineConfigurationImpl进行配置化.   