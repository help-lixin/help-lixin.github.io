---
layout: post
title: 'Camunda源码剖析之CamundaBpmProperties(二)' 
date: 2023-06-10
author: 李新
tags:  Camunda
---

### (1). 概述
在前面,CamundaBpmAutoConfiguration类上有配置一个Properties,固名思义:这是一个配置类,由于:配置类是我们的研发常用的基本信息,所以,可以把它理解成业务模型,通过配置项,我们可以对Caunda进行一些开关配置. 

### (2). CamundaBpmProperties
```
package org.camunda.bpm.spring.boot.starter.property;

// camunda.bpm
@ConfigurationProperties(CamundaBpmProperties.PREFIX)
public class CamundaBpmProperties {

	public static final String PREFIX = "camunda.bpm";
	public static final String UNIQUE_ENGINE_NAME_PREFIX = "processEngine";
	public static final String UNIQUE_APPLICATION_NAME_PREFIX = "processApplication";

	public static final String[] DEFAULT_BPMN_RESOURCE_SUFFIXES = new String[]{"bpmn20.xml", "bpmn" };
	public static final String[] DEFAULT_CMMN_RESOURCE_SUFFIXES = new String[]{"cmmn11.xml", "cmmn10.xml", "cmmn" };
	public static final String[] DEFAULT_DMN_RESOURCE_SUFFIXES = new String[]{"dmn11.xml", "dmn" };

	// ***************************************************************************
	// 定义默认的部署的资源路径表达式
	// ***************************************************************************
	static String[] initDeploymentResourcePattern() {
		final Set<String> suffixes = new HashSet<>();
		// "dmn11.xml", "dmn"
		suffixes.addAll(Arrays.asList(DEFAULT_DMN_RESOURCE_SUFFIXES));
		// "bpmn20.xml", "bpmn"
		suffixes.addAll(Arrays.asList(DEFAULT_BPMN_RESOURCE_SUFFIXES));
		// "cmmn11.xml", "cmmn10.xml", "cmmn" 
		suffixes.addAll(Arrays.asList(DEFAULT_CMMN_RESOURCE_SUFFIXES));

		final Set<String> patterns = new HashSet<>();
		for (String suffix : suffixes) {
		  patterns.add(String.format("%s**/*.%s", CLASSPATH_ALL_URL_PREFIX, suffix));
		}
		return patterns.toArray(new String[patterns.size()]);
	} // end initDeploymentResourcePattern


	static StringJoiner joinOn(final Class<?> clazz) {
	  return new StringJoiner(", ", clazz.getSimpleName() + "[", "]");
	}

	public static String getUniqueName(String name) {
	  return name + RandomStringUtils.randomAlphanumeric(10);
	}

	//
	private String processEngineName = ProcessEngines.NAME_DEFAULT;

	private Boolean generateUniqueProcessEngineName = false;

	private Boolean generateUniqueProcessApplicationName = false;
	
	private String idGenerator = IdGeneratorConfiguration.STRONG;

	private Boolean jobExecutorAcquireByPriority = null;

	private Integer defaultNumberOfRetries = null;

	/**
	* the history level to use
	*/
	private String historyLevel = ProcessEngineConfiguration.HISTORY_FULL;

	/**
	* the default history level to use when 'historyLevel' is 'auto'
	*/
	private String historyLevelDefault = ProcessEngineConfiguration.HISTORY_FULL;

	/** 启用自动部署(默认会对:"bpmn20.xml", "bpmn" , "cmmn11.xml", "cmmn10.xml", "cmmn" , "dmn11.xml", "dmn")进行部署
	* enables auto deployment of processes
	*/
	private boolean autoDeploymentEnabled = true;

	/**
	 * 定这句话默认的资源路径
	* resource pattern for locating process sources
	*/
	private String[] deploymentResourcePattern = initDeploymentResourcePattern();

	/**
	* default serialization format to use
	*/
	private String defaultSerializationFormat = Defaults.INSTANCE.getDefaultSerializationFormat();

	private URL licenseFile;

	/**
	 * camunda是否启用
	* deactivate camunda auto configuration
	*/
	private boolean enabled = true;

	/**
	 * metrics与监控相关
	* metrics configuration
	*/
	@NestedConfigurationProperty
	private MetricsProperty metrics = new MetricsProperty();

	/**
	 * 数据库配置
	 * 主要用于配置:schmea/tablePrefix/
	* database configuration
	*/
	@NestedConfigurationProperty
	private DatabaseProperty database = new DatabaseProperty();

	/**
	* 配置事件监听(execution/task/history)
	* Spring eventing configuration
	*/
	@NestedConfigurationProperty
	private EventingProperty eventing = new EventingProperty();

	/**
	* JPA configuration
	*/
	@NestedConfigurationProperty
	private JpaProperty jpa = new JpaProperty();

	/**
	*  定时任务信息配置
	*  corePoolSize/maxPoolSize/queueCapacity/keepAliveSeconds/deploymentAware
	* job execution configuration
	*/
	@NestedConfigurationProperty
	private JobExecutionProperty jobExecution = new JobExecutionProperty();

	/**
	* webapp configuration
	* 定义web的静态文件路径
	*/
	@NestedConfigurationProperty
	private WebappProperty webapp = new WebappProperty();

	@NestedConfigurationProperty
	private AuthorizationProperty authorization = new AuthorizationProperty();

	// ***************************************************************
	// 扩展性属,允许通过map对SpringProcessEngineConfiguration对象的属性进行应用
	// ***************************************************************
	@NestedConfigurationProperty
	private GenericProperties genericProperties = new GenericProperties();
	
	// 管理员配置信息
	// camunda:
	//   bpm:
	//     admin-user:
	//       id: admin
	//       password: admin
	@NestedConfigurationProperty
	private AdminUserProperty adminUser = new AdminUserProperty();

	@NestedConfigurationProperty
	private FilterProperty filter = new FilterProperty();
}  
```
### (3). 总结
CamundaBpmProperties代表我们开发人员,可以配置的一些配置信息,Camunda能够给我们的配置我基本都写了注释. 
