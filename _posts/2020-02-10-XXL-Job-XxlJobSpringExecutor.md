---
layout: post
title: 'XXL-Job XxlJobSpringExecutor(三)'
date: 2020-02-10
author: 李新
tags: XXL-Job源码
---

### (1). 概述
> 在前面有剖析过,XxlJobSpringExecutor实现了:SmartInitializingSingleton,所以Spring初始化之后,会回调:SmartInitializingSingleton.afterSingletonsInstantiated方法.
> 所以:XxlJobSpringExecutor类的入口在:afterSingletonsInstantiated方法上.

### (2). 看下XxlJobSpringExecutor类结构图
!["XxlJobSpringExecutor类结构图"](/assets/xxl-job/imgs/XxlJobExecutor-Class-Diagram.jpg)

### (3). XxlJobSpringExecutor.afterSingletonsInstantiated
```
public void afterSingletonsInstantiated() {
	
	// ***********************************************************
	// 4. init JobHandler Repository (for method)
	//    根据方法上的注解,初始化:JobHandler
	// ***********************************************************
	initJobHandlerMethodRepository(applicationContext);

	// refresh GlueFactory
	GlueFactory.refreshInstance(1);

	// super start
	try {
		// *********************************************************
		// 6. 调用父类:XxlJobExecutor.start
		// *********************************************************
		super.start();
	} catch (Exception e) {
		throw new RuntimeException(e);
	}
}
```
### (4). XxlJobSpringExecutor.initJobHandlerMethodRepository
```
private void initJobHandlerMethodRepository(ApplicationContext applicationContext) {
	if (applicationContext == null) {
		return;
	}
	
	// 从Spring容器里获得所有的Bean的名称
	// init job handler from method
	String[] beanDefinitionNames = applicationContext.getBeanNamesForType(Object.class, false, true);
	// 遍历Bean名称,获得Bean信息
	for (String beanDefinitionName : beanDefinitionNames) {
		// 获得Bean对象
		Object bean = applicationContext.getBean(beanDefinitionName);
		Map<Method, XxlJob> annotatedMethods = null;   // referred to ：org.springframework.context.event.EventListenerMethodProcessor.processBean
		try {
			// 获取方法上:@XxlJob注解
			annotatedMethods = MethodIntrospector.selectMethods(
			        bean.getClass(),
					new MethodIntrospector.MetadataLookup<XxlJob>() {
						@Override
						public XxlJob inspect(Method method) {
							return AnnotatedElementUtils.findMergedAnnotation(method, XxlJob.class);
						}
					});
		} catch (Throwable ex) {
			logger.error("xxl-job method-jobhandler resolve error for bean[" + beanDefinitionName + "].", ex);
		}
		
		if (annotatedMethods==null || annotatedMethods.isEmpty()) {
			// 不存在注解,则跳过
			continue;
		}
		
		for (Map.Entry<Method, XxlJob> methodXxlJobEntry : annotatedMethods.entrySet()) {
			Method method = methodXxlJobEntry.getKey();
			XxlJob xxlJob = methodXxlJobEntry.getValue();
			if (xxlJob == null) {
				continue;
			}

			// @XxlJob(name = "")
			// 注解必须要配置名称
			String name = xxlJob.value();
			if (name.trim().length() == 0) {
				throw new RuntimeException("xxl-job method-jobhandler name invalid, for[" + bean.getClass() + "#" + method.getName() + "] .");
			}
			
			// ********************************************************
			// 5. XxlJobSpringExecutor.registJobHandler/loadJobHandler
			//    @XxlJob配置的名称不能有相同的,否则抛出异常
			// ********************************************************
			if (loadJobHandler(name) != null) {
				throw new RuntimeException("xxl-job jobhandler[" + name + "] naming conflicts.");
			}

			// 标注@XxlJob(name = "")的方法,入参是有要求的
			// 这个方法,必须要有一个参数,而且入参必须是String,否则抛出异常.
			// execute method
			if (!(method.getParameterTypes().length == 1 && method.getParameterTypes()[0].isAssignableFrom(String.class))) {
				throw new RuntimeException("xxl-job method-jobhandler param-classtype invalid, for[" + bean.getClass() + "#" + method.getName() + "] , " +
						"The correct method format like \" public ReturnT<String> execute(String param) \" .");
			}
			
			// 标注@XxlJob(name = "")的方法,返回值是有要求的
			// 这个方法,必须要有返回值,而且返回值必须要是:ReturnT
			if (!method.getReturnType().isAssignableFrom(ReturnT.class)) {
				throw new RuntimeException("xxl-job method-jobhandler return-classtype invalid, for[" + bean.getClass() + "#" + method.getName() + "] , " +
						"The correct method format like \" public ReturnT<String> execute(String param) \" .");
			}
			// 设置方法可以访问
			method.setAccessible(true);

			// init and destory
			Method initMethod = null;
			Method destroyMethod = null;
			
			// 注解上有配置init方法
			if (xxlJob.init().trim().length() > 0) {
				try {
					initMethod = bean.getClass().getDeclaredMethod(xxlJob.init());
					initMethod.setAccessible(true);
				} catch (NoSuchMethodException e) {
					throw new RuntimeException("xxl-job method-jobhandler initMethod invalid, for[" + bean.getClass() + "#" + method.getName() + "] .");
				}
			}
			
			// 注解上有配置destory方法
			if (xxlJob.destroy().trim().length() > 0) {
				try {
					destroyMethod = bean.getClass().getDeclaredMethod(xxlJob.destroy());
					destroyMethod.setAccessible(true);
				} catch (NoSuchMethodException e) {
					throw new RuntimeException("xxl-job method-jobhandler destroyMethod invalid, for[" + bean.getClass() + "#" + method.getName() + "] .");
				}
			}

			// ******************************************************************
			// 5. XxlJobSpringExecutor.registJobHandler/loadJobHandler
			// 5.1. 把@XxlJob注解信息,转换成:MethodJobHandler(属于:IJobHandler的实现)对象.
			// 5.2. 把MethodJobHandler向MAP注册.
			// ******************************************************************
			// registry jobhandler
			registJobHandler(name, new MethodJobHandler(bean, method, initMethod, destroyMethod));
		} // end for
	}// end for
}//end initJobHandlerMethodRepository
```
### (5). XxlJobSpringExecutor.registJobHandler/loadJobHandler
```
private static ConcurrentMap<String, IJobHandler> jobHandlerRepository = new ConcurrentHashMap<String, IJobHandler>();

// 注册IJobHandler
public static IJobHandler registJobHandler(String name, IJobHandler jobHandler){
	logger.info(">>>>>>>>>>> xxl-job register jobhandler success, name:{}, jobHandler:{}", name, jobHandler);
	return jobHandlerRepository.put(name, jobHandler);
}// end registJobHandler

// 根据名称获取:IJobHandler
public static IJobHandler loadJobHandler(String name){
	return jobHandlerRepository.get(name);
} // end loadJobHandler
```
### (6). XxlJobExecutor.start
```
public void start() throws Exception {
	// init logpath 
	// 初始化日志路径
	XxlJobFileAppender.initLogPath(logPath);

	
	// 7. init invoker, admin-client
	//    初始化与admin的交互客户端.
	initAdminBizList(adminAddresses, accessToken);


	// init JobLogFileCleanThread 
	// 日志轮询线程
	JobLogFileCleanThread.getInstance().start(logRetentionDays);

	// init TriggerCallbackThread
	// 触发器回调线程启动,当任务处理完之后,通过独立的线程向admin汇报处理结果.
	TriggerCallbackThread.getInstance().start();

	// ********************************************************
	// init executor-server
	// 8. XxlJobExecutor.initEmbedServer
	//    初始化内嵌的Web服务器,基于Netty开发.
	// ********************************************************
	initEmbedServer(address, ip, port, appname, accessToken);
}
```
### (7). XxlJobExecutor.initAdminBizList
```
private static List<AdminBiz> adminBizList;

// adminAddresses = "http://127.0.0.1:8080/xxl-job-admin"
private void initAdminBizList(String adminAddresses, String accessToken) throws Exception {
	if (adminAddresses!=null && adminAddresses.trim().length()>0) {
		// adminAddresses按照"逗号"分隔,创建:AdminBiz对象.
		for (String address: adminAddresses.trim().split(",")) {
			if (address!=null && address.trim().length()>0) {
				AdminBiz adminBiz = new AdminBizClient(address.trim(), accessToken);
				if (adminBizList == null) {
					adminBizList = new ArrayList<AdminBiz>();
				}
				// 添加到集合
				adminBizList.add(adminBiz);
			}
		}
	}
}// end initAdminBizList

public static List<AdminBiz> getAdminBizList(){
	return adminBizList;
}// end getAdminBizList
```
### (8). XxlJobExecutor.initEmbedServer
```
private EmbedServer embedServer = null;

private void initEmbedServer(String address, String ip, int port, String appname, String accessToken) throws Exception {

	// fill ip port
	// 如果没有指定port,则使用9999 ~ 65535之间的端口
	port = port > 0 ? port : NetUtil.findAvailablePort(9999);
	// 没有指定IP,则获取本机IP地址,注意,多网卡,如果是外网,建议配置某个网卡的IP
	ip = (ip != null && ip.trim().length() > 0 ) ? ip: IpUtil.getIp();

	// 没有指定address的情况下
	// generate address
	if (address==null || address.trim().length()==0) {
		String ip_port_address = IpUtil.getIpPort(ip, port);   // registry-address：default use address to registry , otherwise use ip:port if address is null
		// 产生一个address
		// 比如: http://192.168.1.100:9999
		address = "http://{ip_port}/".replace("{ip_port}", ip_port_address);
	}

	// *******************************************************************
	// 创建:EmbedServer并启动
	// *******************************************************************
	// start
	embedServer = new EmbedServer();
	embedServer.start(address, port, appname, accessToken);
}// end initEmbedServer
```
### (9). 总结
> 其实,上面一张UML图之后,XXL-Job基本功能尽收眼底,XxlJobSpringExecutor的职责如下:  
> 1. 把Method上的@XxlJob(name="helloJob")转换成:MethodJobHandler对象,通过Map(key="helloJob",value=MethodJobHandler) Hold住这些信息.   
> 2. 初始化日志路径.    
> 3. 创建AdminBiz(该对象主要是与admin进行交互).     
> 4. 初始化日志轮询线程.   
> 5. 初始化回调线程(用于向admin汇报jobId执行状态).  
> 6. 创建内嵌Web Server,用于接受来自于:admin的请求信息.      
> 7. 看完UML图后感觉,基本功能已经很清楚了,可以把重心可以放到xxl-job-admin了.   