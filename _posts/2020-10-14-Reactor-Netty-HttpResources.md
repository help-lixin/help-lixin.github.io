---
layout: post
title: 'Project Reactor Netty源码(HttpResources)'
date: 2020-07-21
author: 李新
tags: ReactorNetty
---

### (1).HttpServer
```
public final class HttpServer
		implements NettyConnector<HttpServerRequest, HttpServerResponse> {
    private HttpServer(HttpServer.Builder builder) {
        // ... ...
        if (!serverOptionsBuilder.isLoopAvailable()) {
            // ***********************************
            // 1. 加载资源
            // ***********************************
            serverOptionsBuilder.loopResources(HttpResources.get());
        }
        // ... ...
    }// end HttpServer   
}// end HttpServer
```
### (2).HttpResources 继承关系
```
reactor.ipc.netty.resources.PoolResources
reactor.ipc.netty.resources.LoopResources
    reactor.ipc.netty.tcp.TcpResources
    reactor.ipc.netty.http.HttpResources

    
// TcpResources实现了:PoolResources, LoopResources 
// HttpResources 继承了:TcpResources
```
### (3).HttpResources
```
public final class HttpResources extends TcpResources {
    static final AtomicReference<HttpResources>                          httpResources;
    static final BiFunction<LoopResources, PoolResources, HttpResources> ON_HTTP_NEW;

    static {
        ON_HTTP_NEW = HttpResources::new;
        httpResources = new AtomicReference<>();
	}
 
    // 2.3 调用:HttpResources的构造器
    HttpResources(LoopResources loops, PoolResources pools) {
		super(loops, pools);
	}

    // 2. 获取:HttpResources
    public static HttpResources get() {
        // 2.1 调用父类:TcpResources.getOrCreate
        // null,null,null,lambad,http
        return getOrCreate(httpResources, null, null, ON_HTTP_NEW, "http");
	} //end get     
 
    
}
```
### (4).TcpResources
```
public class TcpResources implements PoolResources, LoopResources {
    
    // 2.1 获取或或创建Resources
    protected static <T extends TcpResources> T getOrCreate(
                    // null
                    AtomicReference<T> ref,
                    // null
			LoopResources loops,
                    //  null
			PoolResources pools,
                   // 指向:HttpResources的构造器
                   // reactor.ipc.netty.http.HttpResources$$Lambda$4
			BiFunction<LoopResources, PoolResources, T> onNew,
                   // http
			String name) {
        T update;
        for (; ; ) {
            // null
            T resources = ref.get();
            // true
            if (resources == null || loops != null || pools != null) {
                    // 2.2  创建:HttpResources(NioEventLoopGroup)
                    // ***********************************************
                    //                     
                    update = create(resources, loops, pools, name, onNew);
                    if (ref.compareAndSet(resources, update)) {
                        if(resources != null){ // false
                                if(loops != null){
                                    resources.defaultLoops.dispose();
                                }
                                if(pools != null){
                                    resources.defaultPools.dispose();
				    }
                        } //end if
                        // 返回:reactor.ipc.netty.http.HttpResources
                        return update;
                    } else {
                        update._dispose();
			}
                } else { 
                    return resources;
                } //end else
        } //end for
	}// end getOrCreate
 
 
    // 2.2 创建:HttpResources(NioEventLoopGroup)
    static <T extends TcpResources> T create(
                   // null
                   T previous,
                   // null
			LoopResources loops,
                   // null
			PoolResources pools,
                   // http
			String name,
                   // 指向:HttpResources的构建器
                   // reactor.ipc.netty.http.HttpResources$$Lambda$4
			BiFunction<LoopResources, PoolResources, T> onNew) {
        if (previous == null) { // true
                // 初始:LoopResources和PoolResources
                // 
                // ************************************************
                // 创建:NioEventLoopGroup
                // 留一章节再详细讲述这部份的内容
                // ************************************************
                // reactor.ipc.netty.resources.DefaultLoopResources
                loops = loops == null ? LoopResources.create("reactor-" + name) : loops;
                // reactor.ipc.netty.resources.DefaultPoolResources
                pools = pools == null ? PoolResources.elastic(name) : pools;
        } else {
                loops = loops == null ? previous.defaultLoops : loops;
                pools = pools == null ? previous.defaultPools : pools;
        }
        // ************************************
        // 2.3 调用:HttpResources的构造器
        // ************************************
        return onNew.apply(loops, pools);
    }// end create
 
} //end 
```
### (5).总结
1. HttpResources.get()方法委托给父类:  
    TcpResources.getOrCreate(...)方法
2. TcpResources.getOrCreate()方法委托给  
    TcpResources.create()方法,  
    创建:LoopResources/PoolResources(NioEventLoopGroup)  
3. TcpResources.create()方法会回调:  
    BiFunction<LoopResources, PoolResources, T> onNew.apply(...)方法,  
    传入:LoopResources/PoolResources(NioEventLoopGroup)
4. 一句话总结:HttpResources.get()方法主要用于创建:NioEventLoopGroup