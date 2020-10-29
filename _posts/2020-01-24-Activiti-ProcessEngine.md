---
layout: post
title: 'Activiti源码(ProcessEngine)'
date: 2020-01-24
author: 李新
tags: Activiti
---

### (1).Activiti入口
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans   http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration">
  	<property name="dataSource" ref="dataSource"/>
    <property name="databaseSchemaUpdate" value="drop-create" />
  </bean>
  
  <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
  		<!-- 基本属性 url、user、password -->
        <property name="url" value="jdbc:mysql://localhost:3306/activiti" />
        <property name="username" value="root" />
        <property name="password" value="123456" />
        <!-- 配置初始化大小、最小、最大 -->
        <property name="initialSize" value="5" />
        <property name="minIdle" value="5" />
        <property name="maxActive" value="10" />
        <!-- 配置从连接池获取连接等待超时的时间 -->
        <property name="maxWait" value="10000" />
        <property name="timeBetweenEvictionRunsMillis" value="600000" />
        <!-- 配置一个连接在池中最大空闲时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="300000" />
        <!-- 设置从连接池获取连接时是否检查连接有效性，true时，每次都检查;false时，不检查 -->
        <property name="testOnBorrow" value="false" />
        <!-- 设置往连接池归还连接时是否检查连接有效性，true时，每次都检查;false时，不检查 -->
        <property name="testOnReturn" value="false" />
        <!-- 设置从连接池获取连接时是否检查连接有效性，true时，如果连接空闲时间超过minEvictableIdleTimeMillis进行检查，否则不检查;false时，不检查 -->
        <property name="testWhileIdle" value="true" />
        <!-- 检验连接是否有效的查询语句。如果数据库Driver支持ping()方法，则优先使用ping()方法进行检查，否则使用validationQuery查询进行检查。(Oracle jdbc Driver目前不支持ping方法) -->
        <property name="validationQuery" value="SELECT 1" />
        <!-- 打开PSCache，并且指定每个连接上PSCache的大小，Oracle等支持游标的数据库，打开此开关，会以数量级提升性能，具体查阅PSCache相关资料 -->
        <property name="poolPreparedStatements" value="true" />
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20" />
        <property name="filters" value="stat,slf4j" />   
  </bean>
</beans>
```
### (2).ProcessEngineTest
```
package help.lixin.activiti;

import org.activiti.engine.ProcessEngine;
import org.activiti.engine.ProcessEngines;
import org.activiti.engine.TaskService;
import org.junit.Test;
import static org.junit.Assert.*;

public class ProcessEngineTest {
	@Test
	public void getDefaultProcessEngine() {
          // 1. 获得默认的流程引敬对象
		ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
		TaskService taskService = processEngine.getTaskService();
		assertNotNull("task servier not null", taskService);
	}
}
```
### (3).ProcessEngines 初始化
```
//          field                  
public static final String NAME_DEFAULT = "default";
protected static boolean isInitialized;


public static ProcessEngine getDefaultProcessEngine() {
return getProcessEngine(NAME_DEFAULT);
}

public static ProcessEngine getProcessEngine(String processEngineName) {
  // 判断是否初始化过
  if (!isInitialized()) {
    // 初始化
    init();
  }
  return processEngines.get(processEngineName);
}
```
### (4).ProcessEngines 初始化
```
//             field
protected static Map<String, ProcessEngine> processEngines = new HashMap<String, ProcessEngine>();



public synchronized static void init() {
    // 再次判断是否否初始化过
    if (!isInitialized()) {
      if (processEngines == null) {
        // 创建流程引擎载体
        processEngines = new HashMap<String, ProcessEngine>();
      }
      // 获得ClassLoader
      ClassLoader classLoader = ReflectUtil.getClassLoader();
      Enumeration<URL> resources = null;
      try {
        // 尝试从ClassLoader中获得资源文件(activiti.cfg.xml)
        resources = classLoader.getResources("activiti.cfg.xml");
      } catch (IOException e) {
        throw new ActivitiIllegalArgumentException("problem retrieving activiti.cfg.xml resources on the classpath: " + System.getProperty("java.class.path"), e);
      }
      // 把资源存入Set集合中,去除重复资源
      Set<URL> configUrls = new HashSet<URL>();
      while (resources.hasMoreElements()) {
        configUrls.add(resources.nextElement());
      }
      
      // 遍历(activiti.cfg.xml)资源,挨个进行初始化 
      for (Iterator<URL> iterator = configUrls.iterator(); iterator.hasNext();) {
        URL resource = iterator.next();
        log.info("Initializing process engine using configuration '{}'", resource.toString());
        // **********************初始化流程***************************
        initProcessEngineFromResource(resource);
      }

      try {
        resources = classLoader.getResources("activiti-context.xml");
      } catch (IOException e) {
        throw new ActivitiIllegalArgumentException("problem retrieving activiti-context.xml resources on the classpath: " + System.getProperty("java.class.path"), e);
      }
      
      while (resources.hasMoreElements()) {
        URL resource = resources.nextElement();
        log.info("Initializing process engine using Spring configuration '{}'", resource.toString());
        initProcessEngineFromSpringResource(resource);
      }

      // 设置初始化完成
      setInitialized(true);
    } else {
      log.info("Process engines already initialized");
    }
  }
```
### (5).ProcessEngines 初始化
```
//             field 
protected static Map<String, ProcessEngineInfo> processEngineInfosByName = new HashMap<String, ProcessEngineInfo>();
protected static Map<String, ProcessEngineInfo> processEngineInfosByResourceUrl = new HashMap<String, ProcessEngineInfo>();
protected static List<ProcessEngineInfo> processEngineInfos = new ArrayList<ProcessEngineInfo>();


private static ProcessEngineInfo initProcessEngineFromResource(URL resourceUrl) {
    // 判断资源是否存在
    ProcessEngineInfo processEngineInfo = processEngineInfosByResourceUrl.get(resourceUrl.toString());

    // 如果资源存在的情况下
    if (processEngineInfo != null) {   // processEngineInfo=null
      processEngineInfos.remove(processEngineInfo);
      if (processEngineInfo.getException() == null) {
        String processEngineName = processEngineInfo.getName();
        processEngines.remove(processEngineName);
        processEngineInfosByName.remove(processEngineName);
      }
      processEngineInfosByResourceUrl.remove(processEngineInfo.getResourceUrl());
    }

    // 获得资源的路径
    // file:/Users/lixin/WorkspaceNetty/activiti-demo/target/classes/activiti.cfg.xml
    String resourceUrlString = resourceUrl.toString();
    try {
      log.info("initializing process engine for resource {}", resourceUrl);
      // 根据URL创建:ProcessEngine
      ProcessEngine processEngine = buildProcessEngine(resourceUrl);
      String processEngineName = processEngine.getName();
      log.info("initialised process engine {}", processEngineName);
      processEngineInfo = new ProcessEngineInfoImpl(processEngineName, resourceUrlString, null);
      processEngines.put(processEngineName, processEngine);
      processEngineInfosByName.put(processEngineName, processEngineInfo);
    } catch (Throwable e) {
      log.error("Exception while initializing process engine: {}", e.getMessage(), e);
      processEngineInfo = new ProcessEngineInfoImpl(null, resourceUrlString, getExceptionString(e));
    }
    processEngineInfosByResourceUrl.put(resourceUrlString, processEngineInfo);
    processEngineInfos.add(processEngineInfo);
    return processEngineInfo;
  }
```
### (6).ProcessEngines 初始化
```
// 根据URL构建:ProcessEngine
private static ProcessEngine buildProcessEngine(URL resource) {
    InputStream inputStream = null;
    try {
      inputStream = resource.openStream();
      // ************************创建:ProcessEngineConfiguration******************************
      ProcessEngineConfiguration processEngineConfiguration = ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(inputStream);
      // ************************调用:ProcessEngineConfiguration.buildProcessEngine()******************************
      return processEngineConfiguration.buildProcessEngine();

    } catch (IOException e) {
      throw new ActivitiIllegalArgumentException("couldn't open resource stream: " + e.getMessage(), e);
    } finally {
      IoUtil.closeSilently(inputStream);
    }
}
```
### (7).ProcessEngineConfiguration 创建引擎
```
public static ProcessEngineConfiguration createProcessEngineConfigurationFromInputStream(InputStream inputStream) {
    return createProcessEngineConfigurationFromInputStream(inputStream, "processEngineConfiguration");
}

public static ProcessEngineConfiguration createProcessEngineConfigurationFromInputStream(InputStream inputStream, String beanName) {
    // ******************************************************
    return BeansConfigurationHelper.parseProcessEngineConfigurationFromInputStream(inputStream, beanName);
}
```
### (8).BeansConfigurationHelper
```
public static ProcessEngineConfiguration parseProcessEngineConfigurationFromResource(String resource, String beanName) {
    Resource springResource = new ClassPathResource(resource);
    return parseProcessEngineConfiguration(springResource, beanName);
}

public static ProcessEngineConfiguration parseProcessEngineConfiguration(Resource springResource, String beanName) {
    // 创建Spring 框架中 DefaultListableBeanFactory
    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
   // 创建XmlBeanDefinitionReader,解析XML中的Bean后将Bean信息存入:DefaultListableBeanFactory中
   XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
   xmlBeanDefinitionReader.setValidationMode(XmlBeanDefinitionReader.VALIDATION_XSD);
   // 解析资源
   xmlBeanDefinitionReader.loadBeanDefinitions(springResource);
   
   // 从Spring容器中获得流程引擎对象(beanName = processEngineConfiguration)
   // Spring 实例化:StandaloneProcessEngineConfiguration时
   // 同时会实例化父类(ProcessEngineConfigurationImpl)的各个成员属性
   //   repositoryService
   //   runtimeService
   //   historyService
   //   identityService
   //   taskService
   //   formService
   //   managementService
   //   dynamicBpmnService
   // **************************将Bean初始化交给Spring**********************************
   ProcessEngineConfigurationImpl processEngineConfiguration = (ProcessEngineConfigurationImpl) beanFactory.getBean(beanName);
   // 将Spring BeanFactory通过:SpringBeanFactoryProxyMap进行包裹
   // 并设置回: processEngineConfiguration 中
   processEngineConfiguration.setBeans(new SpringBeanFactoryProxyMap(beanFactory));
   return processEngineConfiguration;
}
```
### (9).StandaloneProcessEngineConfiguration 继承关系
```
Object
    ProcessEngineConfiguration
       ProcessEngineConfigurationImpl
           StandaloneProcessEngineConfiguration
```
### (10).StandaloneProcessEngineConfiguration
```
public class StandaloneProcessEngineConfiguration 
       // 继承:ProcessEngineConfigurationImpl
       extends ProcessEngineConfigurationImpl {  

  @Override
  public CommandInterceptor createTransactionInterceptor() {
    return null;
  }// end createTransactionInterceptor
  
}
```
### (11).ProcessEngineConfigurationImpl 
```
public abstract class ProcessEngineConfigurationImpl 
                // 继承:ProcessEngineConfiguration
                extends ProcessEngineConfiguration {
    
  // SERVICES /////////////////////////////////////////////////////////////////

  protected RepositoryService repositoryService = new RepositoryServiceImpl();
  protected RuntimeService runtimeService = new RuntimeServiceImpl();
  protected HistoryService historyService = new HistoryServiceImpl(this);
 protected IdentityService identityService = new IdentityServiceImpl();
 protected TaskService taskService = new TaskServiceImpl(this);
 protected FormService formService = new FormServiceImpl();
 protected ManagementService managementService = new ManagementServiceImpl();
 protected DynamicBpmnService dynamicBpmnService = new DynamicBpmnServiceImpl(this);
    
    

```
### (12).ProcessEngines
```
private static ProcessEngine buildProcessEngine(URL resource) {
    InputStream inputStream = null;
    try {
      inputStream = resource.openStream();
      ProcessEngineConfiguration processEngineConfiguration = ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(inputStream);
      // buildProcessEngine
      return processEngineConfiguration.buildProcessEngine();
    } catch (IOException e) {
      throw new ActivitiIllegalArgumentException("couldn't open resource stream: " + e.getMessage(), e);
    } finally {
      IoUtil.closeSilently(inputStream);
    }
  }
```
### (13).ProcessEngineConfigurationImpl 构建引擎
```
public ProcessEngine buildProcessEngine() {
    // 调用 init 
    init();
    // 创建流程引擎对象(并传入当前对象)
    ProcessEngineImpl processEngine = new ProcessEngineImpl(this);
    
    if (isActiviti5CompatibilityEnabled && activiti5CompatibilityHandler != null) {
      Context.setProcessEngineConfiguration(processEngine.getProcessEngineConfiguration());
      activiti5CompatibilityHandler.getRawProcessEngine();
    }
    
    // 调用后置处理
    postProcessEngineInitialisation();
    return processEngine;
} //end buildProcessEngine
```