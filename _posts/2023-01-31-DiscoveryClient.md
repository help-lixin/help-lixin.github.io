---
layout: post
title: 'Eureka源码分析之DiscoveryClient' 
date: 2023-01-31
author: 李新
tags:  SpringCloud
---

### (1). 概述
今天对Eureka进行扩展(要注意:Feign的调用,并不是走Spring包装的:DiscoveryClient,而是,走Ribbon实际是:Eureka定义的DiscoveryClient),同时,还发现一段优秀的代码,特意记录下来.  

### (2). DiscoveryClient实现关系
> 要注意,这里的DiscoveryClient是Spring封装的,而非最底层Eureka的:DiscoveryClient,咱们挑两个实现类进行简单的剖析. 


```
org.springframework.cloud.client.discovery.DiscoveryClient
  org.springframework.cloud.netflix.eureka.EurekaDiscoveryClient
  org.springframework.cloud.client.discovery.noop.NoopDiscoveryClient
  org.springframework.cloud.client.discovery.simple.SimpleDiscoveryClient
  org.springframework.cloud.client.discovery.composite.CompositeDiscoveryClient
```
### (3). EurekaDiscoveryClient
```
public class EurekaDiscoveryClient implements DiscoveryClient {
	
	// ****************************************************************************
	// 直接委托给:Eureka了,这里用到了装饰器模式.
	// ****************************************************************************
	private final com.netflix.discovery.EurekaClient eurekaClient;
	
	public List<ServiceInstance> getInstances(String serviceId) {
		List<InstanceInfo> infos = this.eurekaClient.getInstancesByVipAddress(serviceId,
				false);
		List<ServiceInstance> instances = new ArrayList<>();
		for (InstanceInfo info : infos) {
			instances.add(new EurekaServiceInstance(info));
		}
		return instances;
	} // end getInstances
}	
```
### (4). SimpleDiscoveryClient
```
public class SimpleDiscoveryClient implements DiscoveryClient {
	
	// ************************************************************************
	// 定义了配置文件,意味着:可以通过配置文件来定义:微服务名称与实例之间的关系,意味着:我们可以让一些微服务,不用走Eureka获取
	// ************************************************************************
	private SimpleDiscoveryProperties simpleDiscoveryProperties;
	
	public List<ServiceInstance> getInstances(String serviceId) {
		List<ServiceInstance> serviceInstances = new ArrayList<>();
		List<SimpleServiceInstance> serviceInstanceForService = this.simpleDiscoveryProperties
				.getInstances().get(serviceId);

		if (serviceInstanceForService != null) {
			serviceInstances.addAll(serviceInstanceForService);
		}
		return serviceInstances;
	} // end 
}	
```
### (5). CompositeDiscoveryClient
```
public class CompositeDiscoveryClient implements DiscoveryClient {

	private final List<DiscoveryClient> discoveryClients;

	public List<ServiceInstance> getInstances(String serviceId) {
		if (this.discoveryClients != null) {
			// *********************************************************************
			// 遍历,谁先拿到微服务与服务实例之间的关系,就直接返回.
			// *********************************************************************
			for (DiscoveryClient discoveryClient : discoveryClients) {
				List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);
				if (instances != null && !instances.isEmpty()) {
					return instances;
				}
			}
		}
		return Collections.emptyList();
	}// end
}
```
### (6). 设计模式
> 简单的几行代码,却透露着两个设计模式: 
> 1. 策略模式. 
> 2. 组合模式.

### (7). 总结
为什么,我说代码优秀,是因为,我在开发阶段,可以通过配置(SimpleDiscoveryClient/CompositeDiscoveryClient),指定某个微服务的IP和地址,而其余的微服务,继续走服务发现功能,挺优秀的代码. 
