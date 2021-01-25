---
layout: post
title: 'Camunda ProcessEngine是如何构建出来的?(一)'
date: 2021-01-25
author: 李新
tags: Camunda
---

### (1). 查看ProcessEngine接口的职责

```
# 操作外部任务相关的服务
ExternalTaskService

# 操作授权相关的服务
AuthorizationService

# 操作过滤器相关的服务
FilterService

# 操作流程表单相关的服务
FormService

# 查询历史表的相关数据
HistoryService

# 操作用户/组/租户相关
IdentityService

# 执行命令以及Job相关的服务
ManagementService

# 操作流程定义相关
RepositoryService

# 操作流程实例相关
RuntimeService

# 操作任务(用户任务)
TaskService

# 操作CMMN
CaseService

# 操作CMMN
DecisionService

```
### (2). ProcessEngine类构结图
![ProcessEngine类构结图](/assets/camunda/imgs/ProcessEngines.jpg)

### (3). ProcessEngineTest
```
package help.lixin.camunda;

import org.camunda.bpm.engine.ProcessEngine;
import org.camunda.bpm.engine.ProcessEngines;

public class ProcessEngineTest {
	public static void main(String[] args) {
		ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
		System.out.println(processEngine);
		System.out.println(processEngine.getProcessEngineConfiguration());
		
		// **********************************************************
		// 操作外部任务相关的服务
		System.out.println(processEngine.getExternalTaskService());
		// 操作授权相关的服务
		System.out.println(processEngine.getAuthorizationService());
		// 操作DMN相关
		System.out.println(processEngine.getDecisionService());
		// 操作过滤器相关的服务
		System.out.println(processEngine.getFilterService());
		// 操作流程表单相关的服务 
		System.out.println(processEngine.getFormService());
		// 查询历史表的相关数据
		System.out.println(processEngine.getHistoryService());
		// 操作用户/组/租户相关
		System.out.println(processEngine.getIdentityService());
		// 执行CMD以及Job相关的服务
		System.out.println(processEngine.getManagementService());
		// 操作流程定义相关
		System.out.println(processEngine.getRepositoryService());
		// 操作流程实例相关
		System.out.println(processEngine.getRuntimeService());
		// 操作任务(用户任务)
		System.out.println(processEngine.getTaskService());
		// 操作CMMN
		System.out.println(processEngine.getCaseService());
	}
}
```
### (4). ProcessEngine初始化过程图解
!["ProcessEngine初始化过程图解"](/assets/camunda/imgs/ProcessEngine-Sequence.jpg)

### (5). ProcessEngines.getProcessEngine方法为入口
```
public static ProcessEngine getProcessEngine(
		 // processEngineName = "default"
         String processEngineName, 
		 // forceCreate = true
		 boolean forceCreate) {
	// 是否初始化		 
	if (!isInitialized) {
		// 6. 先进行初始化
	  init(forceCreate);
	}
	return processEngines.get(processEngineName);
} // end getProcessEngine
```
### (6). ProcessEngines.init
```
public synchronized static void init(boolean forceCreate) {
  if (!isInitialized) { // false
	if(processEngines == null) { // false
	  // Create new map to store process-engines if current map is null
	  processEngines = new HashMap<String, ProcessEngine>();
	}
	
	// 通过反射拿到ClassLoader
	ClassLoader classLoader = ReflectUtil.getClassLoader();
	
	// 用一个变量来保存所有的xml信息
	Enumeration<URL> resources = null;
	try {
		// 读取:camunda.cfg.xml
	  resources = classLoader.getResources("camunda.cfg.xml");
	} catch (IOException e) {
	  try {
		  // 在读取:camunda.cfg.xml出现异常的情况下,
		  // 次而读取:activiti.cfg.xml
		resources = classLoader.getResources("activiti.cfg.xml");
	  } catch(IOException ex) {
		if(forceCreate) {
		  throw new ProcessEngineException("problem retrieving camunda.cfg.xml and activiti.cfg.xml resources on the classpath: "+System.getProperty("java.class.path"), ex);
		} else {
		  return;
		}
	  }
	} 

	// 把所有的URL去掉,保存到临时变量:Set<URL>集合中
	// Remove duplicated configuration URL's using set. Some classloaders may return identical URL's twice, causing duplicate startups
	Set<URL> configUrls = new HashSet<URL>();
	while (resources.hasMoreElements()) {
	  configUrls.add( resources.nextElement() );
	}
	
	// 遍历xml文件
	for (Iterator<URL> iterator = configUrls.iterator(); iterator.hasNext();) {
	  URL resource = iterator.next();
	  // 7.初始化流程引擎
	  initProcessEngineFromResource(resource);
	}

	// 读取:activiti-context.xml
	try {
	  resources = classLoader.getResources("activiti-context.xml");
	} catch (IOException e) {
	  if(forceCreate) {
		throw new ProcessEngineException("problem retrieving activiti-context.xml resources on the classpath: "+System.getProperty("java.class.path"), e);
	  } else {
		return;
	  }
	}
	while (resources.hasMoreElements()) {
	  URL resource = resources.nextElement();
	  // 初始化流程引擎
	  initProcessEngineFromSpringResource(resource);
	}
	isInitialized = true;
  } else {
	LOG.processEngineAlreadyInitialized();
  }
}// end init
```
### (7). ProcessEngines.initProcessEngineFromResource
```
private static ProcessEngineInfo initProcessEngineFromResource(URL resourceUrl) {
	// 根据URL获得:ProcessEngineInfo
	ProcessEngineInfo processEngineInfo = processEngineInfosByResourceUrl.get(resourceUrl);
	// 我这里是第一次创建,所以这个对象为:null
	if (processEngineInfo!=null) { // false
	  processEngineInfos.remove(processEngineInfo);
	  if (processEngineInfo.getException()==null) {
		String processEngineName = processEngineInfo.getName();
		processEngines.remove(processEngineName);
		processEngineInfosByName.remove(processEngineName);
	  }
	  processEngineInfosByResourceUrl.remove(processEngineInfo.getResourceUrl());
	}

	// camunda.cfg.xml
	String resourceUrlString = resourceUrl.toString();
	try {
	  LOG.initializingProcessEngineForResource(resourceUrl);
	  
	  // 8. buildProcessEngine,构建:ProcessEngine
	  ProcessEngine processEngine = buildProcessEngine(resourceUrl);
	  String processEngineName = processEngine.getName();
	  LOG.initializingProcessEngine(processEngine.getName());
	  
	  processEngineInfo = new ProcessEngineInfoImpl(processEngineName, resourceUrlString, null);
	  
	  // ****************************************************************
	  // 把流程引擎的名称和刚创建的流程引擎对象,设置到:ProcessEngines对象的实例变量里(processEngines)
	  // 相当于Hold住流程引擎对象
	  // 在:ProcessEngines.getProcessEngine方法里会要获取.
	  // 如果没有指定:key,则:默认为:default
	  // ****************************************************************
	  processEngines.put(processEngineName, processEngine);
	  processEngineInfosByName.put(processEngineName, processEngineInfo);
	} catch (Throwable e) {
	  LOG.exceptionWhileInitializingProcessengine(e);
	  processEngineInfo = new ProcessEngineInfoImpl(null, resourceUrlString, getExceptionString(e));
	}
	
	processEngineInfosByResourceUrl.put(resourceUrlString, processEngineInfo);
	processEngineInfos.add(processEngineInfo);
	return processEngineInfo;
} // end
```
### (8). ProcessEngines.buildProcessEngine
```
private static  ProcessEngine buildProcessEngine(URL resource) {
    InputStream inputStream = null;
    try {
      inputStream = resource.openStream();
	  // 9. 通过Spring加载XML
      ProcessEngineConfiguration processEngineConfiguration = ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(inputStream);
	  
	  // 10. buildProcessEngine构建出:ProcessEngine对象
      return processEngineConfiguration.buildProcessEngine();

    } catch (IOException e) {
      throw new ProcessEngineException("couldn't open resource stream: "+e.getMessage(), e);
    } finally {
      IoUtil.closeSilently(inputStream);
    }
}
```
### (9). ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream
```
public static ProcessEngineConfiguration createProcessEngineConfigurationFromInputStream(InputStream inputStream) {
    return createProcessEngineConfigurationFromInputStream(inputStream, "processEngineConfiguration");
}

public static ProcessEngineConfiguration createProcessEngineConfigurationFromInputStream(InputStream inputStream, String beanName) {
	// BeansConfigurationHelper主法内部我就不跟进去了,实则就是创建:DefaultListableBeanFactory
	// 通过:DefaultListableBeanFactory加载XML
	// 1. 通过工具类,读取流:inputStream
	// 2. 创建DefaultListableBeanFactory加载:XML
	// 3. 从DefaultListableBeanFactory(Spring容器)里,根据名称:processEngineConfiguration获得:ProcessEngineConfiguration的实例
	return BeansConfigurationHelper.parseProcessEngineConfigurationFromInputStream(inputStream, beanName);
}
```
### (10). ProcessEngineConfigurationImpl.buildProcessEngine
```
// 这一部份的内容,我暂时不跟进去了.
// 反正,理解:ProcessEngine的职责就可以了.
public ProcessEngine buildProcessEngine() {
	init();
	processEngine = new ProcessEngineImpl(this);
	invokePostProcessEngineBuild(processEngine);
	return processEngine;
}
```
 