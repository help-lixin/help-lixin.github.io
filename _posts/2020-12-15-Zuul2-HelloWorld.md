---
layout: post
title: 'Zuul2 入门'
date: 2020-12-15
author: 李新
tags: Zuul2
---

### (0). 整个项目git地址
> git clone https://github.com/dashprateek/zuul2-sample.git

### (1). 项目结构如下
```
├── README.md
├── pom.xml
├── src
│   ├── main
│   │   ├── groovy
│   │   │   └── com
│   │   │       └── netflix
│   │   │           └── zuul
│   │   │               └── sample
│   │   │                   └── filters
│   │   │                       ├── endpoint
│   │   │                       │   └── Healthcheck.groovy
│   │   │                       ├── inbound
│   │   │                       │   ├── Debug.groovy
│   │   │                       │   ├── DebugRequest.groovy
│   │   │                       │   └── Routes.groovy
│   │   │                       └── outbound
│   │   │                           └── ZuulResponseFilter.groovy
│   │   ├── java
│   │   │   └── com
│   │   │       └── gateway
│   │   │           └── api
│   │   │               ├── Bootstrap.java
│   │   │               ├── module
│   │   │               │   └── GatewayModule.java
│   │   │               ├── server
│   │   │               │   └── GatewayServerStartup.java
│   │   │               └── util
│   │   │                   └── RouteUtils.java
│   │   └── resources
│   │       ├── application-test.properties
│   │       ├── application.properties
│   │       └── log4j.properties

```
### (2). application.properties
```
### Instance env settings

region=us-east-1
environment=test

### Eureka instance registration for this app

#Name of the application to be identified by other services
eureka.name=zuul

#The port where the service will be running and serving requests
eureka.port=7001

#Virtual host name by which the clients identifies this service
eureka.vipAddress=${eureka.name}:${eureka.port}

#For eureka clients running in eureka server, it needs to connect to servers in other zones
eureka.preferSameZone=false

# Don't register locally running instances.
eureka.registration.enabled=false


# Loading Filters
zuul.filters.root=src/main/groovy/com/netflix/zuul/sample/filters
zuul.filters.locations=${zuul.filters.root}/inbound,${zuul.filters.root}/outbound,${zuul.filters.root}/endpoint
zuul.filters.packages=com.netflix.zuul.filters.common


### Load balancing backends with Eureka

#eureka.shouldUseDns=true
#eureka.eurekaServer.context=discovery/v2
#eureka.eurekaServer.domainName=discovery${environment}.netflix.net
#eureka.eurekaServer.gzipContent=true
#
#eureka.serviceUrl.default=http://${region}.${eureka.eurekaServer.domainName}:7001/${eureka.eurekaServer.context}
#
#api.ribbon.NIWSServerListClassName=com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
#api.ribbon.DeploymentContextBasedVipAddresses=api-test.netflix.net:7001


### Load balancing backends without Eureka

eureka.shouldFetchRegistry=false
eureka.validateInstanceId=false

demo.ribbon.listOfServers=127.0.0.1:8080
demo.ribbon.client.NIWSServerListClassName=com.netflix.loadbalancer.ConfigurationBasedServerList
demo.ribbon.readTimeout=10000
# http://localhost:7001/test-provider/demo  -->  http://localhost:8080/demo
routes.demo.path=/test-provider/**


# This has to be the last line
#@next=application-${@environment}.properties
```
### (3). GatewayModule
```
package com.gateway.api.module;

import com.netflix.discovery.AbstractDiscoveryClientOptionalArgs;
import com.netflix.discovery.DiscoveryClient;
import com.netflix.netty.common.accesslog.AccessLogPublisher;
import com.netflix.netty.common.status.ServerStatusManager;
import com.netflix.spectator.api.DefaultRegistry;
import com.netflix.spectator.api.Registry;
import com.netflix.zuul.BasicRequestCompleteHandler;
import com.netflix.zuul.FilterFileManager;
import com.netflix.zuul.RequestCompleteHandler;
import com.netflix.zuul.context.SessionContextDecorator;
import com.netflix.zuul.context.ZuulSessionContextDecorator;
import com.netflix.zuul.init.ZuulFiltersModule;
import com.netflix.zuul.netty.server.BaseServerStartup;
import com.netflix.zuul.netty.server.ClientRequestReceiver;
import com.netflix.zuul.origins.BasicNettyOriginManager;
import com.netflix.zuul.origins.OriginManager;
import com.netflix.zuul.stats.BasicRequestMetricsPublisher;
import com.netflix.zuul.stats.RequestMetricsPublisher;
import com.gateway.api.server.GatewayServerStartup;
import com.google.inject.AbstractModule;

public class GatewayModule extends AbstractModule {

	    @Override
	    protected void configure() {
	        // sample specific bindings
	        bind(BaseServerStartup.class).to(GatewayServerStartup.class);

	        // use provided basic netty origin manager
	        bind(OriginManager.class).to(BasicNettyOriginManager.class);

	        // zuul filter loading
	        install(new ZuulFiltersModule());
	        bind(FilterFileManager.class).asEagerSingleton();

	        // general server bindings
	        bind(ServerStatusManager.class); // health/discovery status
	        bind(SessionContextDecorator.class).to(ZuulSessionContextDecorator.class); // decorate new sessions when requests come in
	        bind(Registry.class).to(DefaultRegistry.class); // atlas metrics registry
	        bind(RequestCompleteHandler.class).to(BasicRequestCompleteHandler.class); // metrics post-request completion
	        bind(AbstractDiscoveryClientOptionalArgs.class).to(DiscoveryClient.DiscoveryClientOptionalArgs.class); // discovery client
	        bind(RequestMetricsPublisher.class).to(BasicRequestMetricsPublisher.class); // timings publisher

	        // access logger, including request ID generator
	        bind(AccessLogPublisher.class).toInstance(new AccessLogPublisher("ACCESS",
	                (channel, httpRequest) -> ClientRequestReceiver.getRequestFromChannel(channel).getContext().getUUID()));
	    }
}
```
### (4). GatewayServerStartup
```
package com.gateway.api.server;

import java.util.HashMap;
import java.util.Map;

import javax.inject.Inject;

import com.netflix.appinfo.ApplicationInfoManager;
import com.netflix.config.DynamicIntProperty;
import com.netflix.discovery.EurekaClient;
import com.netflix.netty.common.accesslog.AccessLogPublisher;
import com.netflix.netty.common.channel.config.ChannelConfig;
import com.netflix.netty.common.channel.config.CommonChannelConfigKeys;
import com.netflix.netty.common.metrics.EventLoopGroupMetrics;
import com.netflix.netty.common.proxyprotocol.StripUntrustedProxyHeadersHandler;
import com.netflix.netty.common.status.ServerStatusManager;
import com.netflix.spectator.api.Registry;
import com.netflix.zuul.FilterLoader;
import com.netflix.zuul.FilterUsageNotifier;
import com.netflix.zuul.RequestCompleteHandler;
import com.netflix.zuul.context.SessionContextDecorator;
import com.netflix.zuul.netty.server.BaseServerStartup;
import com.netflix.zuul.netty.server.DirectMemoryMonitor;
import com.netflix.zuul.netty.server.ZuulServerChannelInitializer;

import io.netty.channel.ChannelInitializer;
import io.netty.channel.group.ChannelGroup;

public class GatewayServerStartup extends BaseServerStartup {

	@Inject
	public GatewayServerStartup(ServerStatusManager serverStatusManager, FilterLoader filterLoader,
			SessionContextDecorator sessionCtxDecorator, FilterUsageNotifier usageNotifier,
			RequestCompleteHandler reqCompleteHandler, Registry registry, DirectMemoryMonitor directMemoryMonitor,
			EventLoopGroupMetrics eventLoopGroupMetrics, EurekaClient discoveryClient,
			ApplicationInfoManager applicationInfoManager, AccessLogPublisher accessLogPublisher) {
		super(serverStatusManager, filterLoader, sessionCtxDecorator, usageNotifier, reqCompleteHandler, registry,
				directMemoryMonitor, eventLoopGroupMetrics, discoveryClient, applicationInfoManager,
				accessLogPublisher);
	}

	@Override
	protected Map<Integer, ChannelInitializer> choosePortsAndChannels(ChannelGroup clientChannels,
			ChannelConfig channelDependencies) {

		Map<Integer, ChannelInitializer> portsToChannels = new HashMap<>();

		int port = new DynamicIntProperty("zuul.server.port.main", 7001).get();

		ChannelConfig channelConfig = BaseServerStartup.defaultChannelConfig();
		/*
		 * These settings may need to be tweaked depending if you're running
		 * behind an ELB HTTP listener, TCP listener, or directly on the
		 * internet.
		 *
		 * The below settings can be used when running behind an ELB HTTP
		 * listener that terminates SSL for you and passes XFF headers.
		 */
		channelConfig.set(CommonChannelConfigKeys.allowProxyHeadersWhen,
				StripUntrustedProxyHeadersHandler.AllowWhen.ALWAYS);
		channelConfig.set(CommonChannelConfigKeys.preferProxyProtocolForClientIp, false);
		channelConfig.set(CommonChannelConfigKeys.isSSlFromIntermediary, false);
		channelConfig.set(CommonChannelConfigKeys.withProxyProtocol, false);

		portsToChannels.put(port,
				new ZuulServerChannelInitializer(port, channelConfig, channelDependencies, clientChannels));
		logPortConfigured(port, null);

		return portsToChannels;
	}

}
```
### (5). RouteUtils
```
package com.gateway.api.util;

import java.util.Map;
import java.util.Map.Entry;
import java.util.Spliterator;
import java.util.Spliterators;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

import org.apache.commons.lang3.StringUtils;
import org.apache.shiro.util.AntPathMatcher;
import org.apache.shiro.util.PatternMatcher;

import com.netflix.config.ConfigurationManager;

public class RouteUtils {

	private static final Map<String, String> routes = RoutesHolder.ROUTES;

	private static final PatternMatcher matcher = new AntPathMatcher();

	public static Map<String, String> getRoutes() {
		return StreamSupport
				.stream(Spliterators.spliteratorUnknownSize(ConfigurationManager.getConfigInstance().getKeys("routes"),
						Spliterator.DISTINCT), false)
				.collect(
						Collectors.toMap(
								key -> StringUtils.removeEndIgnoreCase(
										StringUtils.removeStartIgnoreCase(key, "routes."), ".path"),
								key -> ConfigurationManager.getConfigInstance().getString(key)));
	}

	public static String getRouteVip(String path) {
		return routes.entrySet().stream().filter(entry -> matcher.matches(entry.getValue(), path)).findFirst()
				.map(Entry::getKey).orElse("");
	}

	static class RoutesHolder {
		private static final Map<String, String> ROUTES = getRoutes();
	}

}
```
### (6). Bootstrap
```
package com.gateway.api;

import com.gateway.api.module.GatewayModule;
import com.google.inject.Injector;
import com.netflix.config.ConfigurationManager;
import com.netflix.governator.InjectorBuilder;
import com.netflix.zuul.netty.server.BaseServerStartup;
import com.netflix.zuul.netty.server.Server;

public class Bootstrap {

    public static void main(String[] args) {
        new Bootstrap().start();
    }

    public void start() {
        System.out.println("Zuul Sample: starting up.");
        long startTime = System.currentTimeMillis();
        int exitCode = 0;
        Server server = null;
        try {
            ConfigurationManager.loadCascadedPropertiesFromResources("application");
            Injector injector = InjectorBuilder.fromModule(new GatewayModule()).createInjector();
            BaseServerStartup serverStartup = injector.getInstance(BaseServerStartup.class);
            server = serverStartup.server();
            long startupDuration = System.currentTimeMillis() - startTime;
            System.out.println("Zuul Sample: finished startup. Duration = " + startupDuration + " ms");
            server.start(true);
        }
        catch (Throwable t) {
            t.printStackTrace();
            System.err.println("###############");
            System.err.println("Zuul Sample: initialization failed. Forcing shutdown now.");
            System.err.println("###############");
            exitCode = 1;
        }
        finally {
            // server shutdown
            if (server != null) server.stop();
            System.exit(exitCode);
        }
    }
}
```
### (7). Routes
```
/*
 * Copyright 2018 Netflix, Inc.
 *
 *      Licensed under the Apache License, Version 2.0 (the "License");
 *      you may not use this file except in compliance with the License.
 *      You may obtain a copy of the License at
 *
 *          http://www.apache.org/licenses/LICENSE-2.0
 *
 *      Unless required by applicable law or agreed to in writing, software
 *      distributed under the License is distributed on an "AS IS" BASIS,
 *      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *      See the License for the specific language governing permissions and
 *      limitations under the License.
 */

package com.netflix.zuul.sample.filters.inbound;

import java.util.Map;
import java.util.Map.Entry;
import java.util.Spliterator;
import java.util.Spliterators;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

import org.apache.commons.lang3.StringUtils;

import com.gateway.api.util.RouteUtils;
import com.netflix.config.ConfigurationManager;
import com.netflix.zuul.context.SessionContext;
import com.netflix.zuul.filters.http.HttpInboundSyncFilter;
import com.netflix.zuul.message.http.HttpRequestMessage;
import com.netflix.zuul.netty.filter.ZuulEndPointRunner;

/**
 * Routes configuration
 *
 * Author: Arthur Gonigberg Date: November 21, 2017
 */
public class Routes extends HttpInboundSyncFilter {


    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter(HttpRequestMessage httpRequestMessage) {
        return true;
    }

    @Override
    public HttpRequestMessage apply(HttpRequestMessage request) {
        SessionContext context = request.getContext();
		// *********************************************************************
		// 这是重点:相当于配置URL与后端应用的关系.
		// *********************************************************************
        context.setEndpoint(ZuulEndPointRunner.PROXY_ENDPOINT_FILTER_NAME);
        context.setRouteVIP(RouteUtils.getRouteVip(request.getPath()));
		
		// *********************************************************************
		// 这一部份是实现对URL的重写.
		// *********************************************************************
        // (http://localhost:7001/test-provider/demo) 实际代理之后变成了: (http://localhost:8080/test-provider/demo)
        // 重写URL请求(去掉serviceId)
        // (http://localhost:7001/demo)  实际代理之后变成了: (http://localhost:8080/demo)
        request.setPath("/demo");
        return request;
    }
}
```
### (8). pom.xml
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.gateway</groupId>
	<artifactId>zuul2-sample</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<properties>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
		<java.version>1.8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>com.netflix.zuul</groupId>
			<artifactId>zuul-core</artifactId>
			<version>2.1.3</version>
		</dependency>
		<dependency>
			<groupId>org.apache.shiro</groupId>
			<artifactId>shiro-core</artifactId>
			<version>1.4.0</version>
		</dependency>
		<dependency>
			<groupId>commons-configuration</groupId>
			<artifactId>commons-configuration</artifactId>
			<version>1.8</version>
		</dependency>
		<dependency>
			<groupId>com.netflix.governator</groupId>
			<artifactId>governator-core</artifactId>
			<version>1.17.4</version>
		</dependency>
		<dependency>
			<groupId>com.google.inject</groupId>
			<artifactId>guice</artifactId>
			<version>4.1.0</version>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.8.0</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
			<plugin>
				<!-- Build an executable JAR -->
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-jar-plugin</artifactId>
				<version>3.1.0</version>
				<configuration>
					<archive>
						<manifest>
							<addClasspath>true</addClasspath>
							<classpathPrefix>lib/</classpathPrefix>
							<mainClass>com.gateway.api.Bootstrap</mainClass>
						</manifest>
					</archive>
				</configuration>
			</plugin>
			<plugin>
				<artifactId>maven-assembly-plugin</artifactId>
				<configuration>
					<archive>
						<manifest>
							<mainClass>com.gateway.api.Bootstrap</mainClass>
						</manifest>
					</archive>
					<descriptorRefs>
						<descriptorRef>jar-with-dependencies</descriptorRef>
					</descriptorRefs>
				</configuration>
				<executions>
					<execution>
						<id>make-assembly</id> <!-- this is used for inheritance merges -->
						<phase>package</phase> <!-- bind to the packaging phase -->
						<goals>
							<goal>single</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>
```
### (9). test-provider
```
package help.lixin.samples.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
	private Logger logger = LoggerFactory.getLogger(HelloController.class);

	@GetMapping("/demo")
	public String index() {
		return "index DEMO Hello World!!!";
	}
}

```
### (10). 总结
> 从整体来看Zuul2从Zuul1的Servlet模型向Reactor模型靠扰了,但是,在代码可读性来说,真的不敢苟同(相比:Spring Cloud Gateway).  
> 唯一的优势在于,用了Groovy脚本实现热加载.
