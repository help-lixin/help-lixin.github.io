---
layout: post
title: 'Ribbon 扩展RibbonLoadBalancerClient'
date: 2019-06-29
author: 李新
tags: Ribbon
---

### (1). 需求
!["需求"](/assets/ribbon/imgs/needs.jpeg)

> 随着微服务的流行,对开发人员的要求也越来越高,
> 特别是当一个新入职的程序员,如何在开发环境给新手开发员更好的体验,而不是在开发环境上造成入职当天就离职呢?所以,这个需求是针对开发的环境的.     
> 当微服务工程越来越多之后,开发人员能否如上图所示,只下载并开发它所要的工程,而工程所依赖的其它信息,都来自于一个稳定的环境呢?        

### (2). 开发步骤
> 1. 一套稳定测试环境(包括:Eureka).   
> 2. 开发环境都依赖这套稳定测试环境(全都往这个Eureka过行注册).  
> 3. 开发下载工程,进行开发,那么开发,如何调试?如何对Request路由呢?.     
> 4. 在要路由的HTTP请求头上,打标记(例如:route=user-service/192.168.100.202:8080)
> 5. 配置一个拦截器,拦截请求头上(route),并设置到ThreadLocal里.    
> 6. 对RibbonLoadBalancerClient进行扩展.当ThreadLocal里有标记(user-service/192.168.100.202:8080),并且微服务名称(user-service)也相同,则直接返回:192.168.100.202:8080.   
> 7. 要自定义一个:ClientHttpRequestInterceptor,让ThreadLocal里的信息(route),一直在整个请求链路上传递.   
> 8. 我在这里,只做一个引导,不把所有逻辑写出来.

### (3). RibbonLoadBalancerClientProxy
```
package help.lixin.samples.ribbon;

import java.io.IOException;
import java.net.URI;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.LoadBalancerClient;
import org.springframework.cloud.client.loadbalancer.LoadBalancerRequest;
import org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient;
import org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient.RibbonServer;

import com.netflix.appinfo.InstanceInfo;
import com.netflix.loadbalancer.Server;
import com.netflix.niws.loadbalancer.DiscoveryEnabledServer;

public class RibbonLoadBalancerClientProxy implements LoadBalancerClient {

	private RibbonLoadBalancerClient ribbonLoadBalancerClient;

	public void setProxyTarget(RibbonLoadBalancerClient ribbonLoadBalancerClient) {
		this.ribbonLoadBalancerClient = ribbonLoadBalancerClient;
	}

	public RibbonLoadBalancerClient getProxyTarget() {
		return ribbonLoadBalancerClient;
	}

	public void setRibbonLoadBalancerClient(RibbonLoadBalancerClient ribbonLoadBalancerClient) {
		this.ribbonLoadBalancerClient = ribbonLoadBalancerClient;
	}

	public RibbonLoadBalancerClient getRibbonLoadBalancerClient() {
		return ribbonLoadBalancerClient;
	}

	@Override
	public ServiceInstance choose(String serviceId) {
		return ribbonLoadBalancerClient.choose(serviceId);
	}

	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
		// TODO 
		// 1. 从ThreadLocal中获得用户定义的serviceId与IP:Port的关系.
		// 2. serviceId进行比较相同的话,直接构建:RibbonServer
		// 3. serviceId不同的话,直接放过.
		InstanceInfo instanceInfo = InstanceInfo.Builder.newBuilder() 
				// serviceId
				.setAppName("test-provider")
				.build();
		Server server = new DiscoveryEnabledServer(instanceInfo, false);
		server.setHost("127.0.0.1");
		server.setPort(8081);
		server.setSchemea("http");
		
		RibbonServer ribbonServer = new RibbonServer(serviceId, server, false, null);
		return this.execute(serviceId, ribbonServer, request);
	}

	@Override
	public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request)
			throws IOException {
		return ribbonLoadBalancerClient.execute(serviceId, serviceInstance, request);
	}

	@Override
	public URI reconstructURI(ServiceInstance instance, URI original) {
		return ribbonLoadBalancerClient.reconstructURI(instance, original);
	}
}
```
### (4). RouteRequestInterceptor
```
package help.lixin.samples.ribbon;

import java.io.IOException;

import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;


public class RouteRequestInterceptor implements ClientHttpRequestInterceptor {

	@Override
	public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution)
			throws IOException {
		// ... ...
		// 从ThreadLocal获得route信息,并设置到:HttpRequest对象里
		return execution.execute(request, body);
	}
}
```

### (5). RibbonLoadBalancerClientProxyConfig
```
package help.lixin.samples.config;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.cloud.client.loadbalancer.LoadBalancerClient;
import org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient;
import org.springframework.cloud.netflix.ribbon.SpringClientFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.web.client.RestTemplate;

import help.lixin.samples.ribbon.RibbonLoadBalancerClientProxy;
import help.lixin.samples.ribbon.RouteRequestInterceptor;

@Configuration
public class RibbonLoadBalancerClientProxyConfig implements ApplicationContextAware, InitializingBean {

	private ApplicationContext applicationContext;

	/**
	 * 对LoadBalancerClient进行扩展.
	 * @param springClientFactory
	 * @return
	 */
	@Bean
//	@Primary
	public LoadBalancerClient loadBalancerClient(SpringClientFactory springClientFactory) {
		// 自定义逻辑的的:LoadBalancerClient
		RibbonLoadBalancerClientProxy proxy = new RibbonLoadBalancerClientProxy();
		// 要代理的目标对象
		RibbonLoadBalancerClient target = new RibbonLoadBalancerClient(springClientFactory);
		proxy.setProxyTarget(target);
		return proxy;
	}

	/**
	 * 创建:RouteRequestInterceptor,把route信息继续往下链路中传递
	 * @return
	 */
	@Bean
	public ClientHttpRequestInterceptor routeRequestInterceptor() {
		return new RouteRequestInterceptor();
	}

	
	@Override
	public void afterPropertiesSet() throws Exception {
		// 获取:RestTemplate
		RestTemplate restTemplate = applicationContext.getBean(RestTemplate.class);
		if (null != restTemplate) {
			List<ClientHttpRequestInterceptor> list = new ArrayList<>(restTemplate.getInterceptors());
			// 配置自定义的拦截器.
			// 要在:
			// LoadBalancerAutoConfiguration$LoadBalancerInterceptorConfig.restTemplateCustomizer
			// 之前配置好自定义的拦截器.
			list.add(routeRequestInterceptor());
			
			restTemplate.setInterceptors(list);
		}
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}
}
```
### (6). 总结
> RibbonLoadBalancerClientProxy可以代理你的业务逻辑,也可以直接放过,这样对开发比较透明.  