---
layout: post
title: 'Spring WebFlux源码(NettyWebServer)'
date: 2020-07-21
author: 李新
tags: Spring WebFlux
---

### (1).查看WebServer声明
```
package org.springframework.boot.web.server;

public interface WebServer {
    // 启动
    void start() throws WebServerException;
    // 停止
    void stop() throws WebServerException;
    // 获得端口
    int getPort();
}
```

### (2).NettyWebServer
```
public class NettyWebServer implements WebServer {
    private final HttpServer httpServer;

	private final ReactorHttpHandlerAdapter handlerAdapter;

	private final Duration lifecycleTimeout;

	private DisposableServer disposableServer;

   // **************************************************************************
   // 1.NetttyWebServer实例化需要依赖以下接口,所以找到在哪构造了这个实例
   // **************************************************************************
	public NettyWebServer(HttpServer httpServer, 
                         ReactorHttpHandlerAdapter handlerAdapter,
			      Duration lifecycleTimeout) {
		Assert.notNull(httpServer, "HttpServer must not be null");
		Assert.notNull(handlerAdapter, "HandlerAdapter must not be null");
		this.httpServer = httpServer;
		this.handlerAdapter = handlerAdapter;
		this.lifecycleTimeout = lifecycleTimeout;
	}
 
}
```
### (3). NettyReactiveWebServerFactory
```
package org.springframework.boot.web.embedded.netty;

public class NettyReactiveWebServerFactory extends AbstractReactiveWebServerFactory {
    
    // ******************************************************************************
    // 3.通过构造器去追踪谁构建了当前对象
    // ******************************************************************************
    public NettyReactiveWebServerFactory() {
	}

	public NettyReactiveWebServerFactory(int port) {
		super(port);
	}

	@Override
	public WebServer getWebServer(HttpHandler httpHandler) {
		HttpServer httpServer = createHttpServer();
		ReactorHttpHandlerAdapter handlerAdapter = new ReactorHttpHandlerAdapter(
				httpHandler);
    // **************************************************************************
    // 2. NettyReactiveWebServerFactory.getWebServer() 构建了NettyWebServer
    // 那么,又是谁构建了:NettyReactiveWebServerFactory.通过构建器去追踪
    // **************************************************************************
    return new NettyWebServer(httpServer, handlerAdapter, this.lifecycleTimeout);
	}

}
```
### (4).ReactiveWebServerFactoryConfiguration
```
package org.springframework.boot.autoconfigure.web.reactive;

abstract class ReactiveWebServerFactoryConfiguration {
    
    @Configuration
    // 在Spring容器中找不到Bean:ReactiveWebServerFactory
	@ConditionalOnMissingBean(ReactiveWebServerFactory.class)
    // ClassPath中存在有Class:HttpServer
	@ConditionalOnClass({ HttpServer.class })
	static class EmbeddedNetty {

		@Bean
		@ConditionalOnMissingBean
		public ReactorResourceFactory reactorServerResourceFactory() {
			return new ReactorResourceFactory();
		}

		@Bean
		public NettyReactiveWebServerFactory nettyReactiveWebServerFactory(
				ReactorResourceFactory resourceFactory) {
                   // ********************************************************************
                   // 4. 构建:NettyReactiveWebServerFactory
                   // ********************************************************************
			NettyReactiveWebServerFactory serverFactory = new NettyReactiveWebServerFactory();
			serverFactory.setResourceFactory(resourceFactory);
			return serverFactory;
		}

	}
    // ....
}
```
### (5).总结
WebServer接口定义了Web容器的启动/停止方法.
<font color='red'>NettyWebServer</font>  / TomcatWebServer   / JettyWebServer...都是属于:WebServer的实现.  
ReactiveWebServerFactoryConfiguration配置了WebServer的实例化.
需要注意:NettyWebServer内部持有reactor.netty.http.server.HttpServer对象,会将请求委托给:reactor.netty.http.server.HttpServer进行处理.  
ReactiveWebServerApplicationContext.finishRefresh()会回调:WebServer.start()方法.