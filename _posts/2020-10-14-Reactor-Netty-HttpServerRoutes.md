---
layout: post
title: 'Project Reactor Netty源码(HttpServerRoutes)'
date: 2020-07-21
author: 李新
tags: ReactorNetty
---

### (1).HttpServer.startRouter
```
HttpServer.create(8080)
    // 1.启动并配置路由信息
    // startRouter(Consumer<? super HttpServerRoutes> routesBuilder)
    // interface Consumer<T> { void accept(T t) };
    .startRouter(routes -> {
        // 2.
        routes.get("/hello", (req, resp) -> resp.sendString(Mono.just("hello!")));
        routes.get("/world", (req, resp) -> resp.sendString(Mono.just("world!")));
    });
```
### (2).HttpServerRoutes
```
package reactor.ipc.netty.http.server;

public interface HttpServerRoutes 
        extends BiFunction<HttpServerRequest, HttpServerResponse, Publisher<Void>> {
    
    static HttpServerRoutes newRoutes() {
		return new DefaultHttpServerRoutes();
    } //end newRoutes
    
    // 删除请求
    default HttpServerRoutes delete(String path,
			BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler) {
		return route(HttpPredicate.delete(path), handler);
	} //end delete                          
 
    // 映射到磁盘文件
    default HttpServerRoutes directory(String uri, Path directory) {
		return directory(uri, directory, null);
	} //end directory
 
    // 映射到目录,并配置拦截器
    HttpServerRoutes directory(String uri, Path directory,
			Function<HttpServerResponse, HttpServerResponse> interceptor); //end directory
   
    // 映射到文件
    default HttpServerRoutes file(String uri, Path path) {
		return file(HttpPredicate.get(uri), path, null);
	} //end file
    
    // 映射到文件
    default HttpServerRoutes file(String uri, String path) {
		return file(HttpPredicate.get(uri), Paths.get(path), null);
	} //end file
    
    // 处理文件的实现
    default HttpServerRoutes file(Predicate<HttpServerRequest> uri, Path path,
			Function<HttpServerResponse, HttpServerResponse> interceptor) {
		Objects.requireNonNull(path, "path");
		return route(uri, (req, resp) -> {
                if (!Files.isReadable(path)) {
				return resp.send(ByteBufFlux.fromPath(path));
                }
                if (interceptor != null) {
				return interceptor.apply(resp)
				                  .sendFile(path);
                }
                return resp.sendFile(path);
            }); //end return
	} //end file
 
    // get请求实现
    default HttpServerRoutes get(String path,
			BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler) {
		return route(HttpPredicate.get(path), handler);
	} // end get
 
    // 首页实现
    default HttpServerRoutes index(final BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler) {
		return route(INDEX_PREDICATE, handler);
	} //end index
 
    // post请求
    default HttpServerRoutes post(String path,
			BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler) {
		return route(HttpPredicate.post(path), handler);
	}// end post
 
    // put请求
    default HttpServerRoutes put(String path,
			BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler) {
		return route(HttpPredicate.put(path), handler);
	} //end put
 
    // route抽象方法
    HttpServerRoutes route(Predicate<? super HttpServerRequest> condition,
			BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler); //end route
   
   // websocket实现
   default HttpServerRoutes ws(String path,
			BiFunction<? super WebsocketInbound, ? super WebsocketOutbound, ? extends
					Publisher<Void>> handler) {
		return ws(path, handler, null);
	} //end ws
 
    //ws 实现
    default HttpServerRoutes ws(String path,
			BiFunction<? super WebsocketInbound, ? super WebsocketOutbound, ? extends
					Publisher<Void>> handler,
			String protocols) {
        Predicate<HttpServerRequest> condition = HttpPredicate.get(path);
        return route(condition, (req, resp) -> {
                if (req.requestHeaders()
                        .contains(HttpHeaderNames.CONNECTION, HttpHeaderValues.UPGRADE, true)) {
				HttpServerOperations ops = (HttpServerOperations) req;
				return ops.withWebsocketSupport(req.uri(), protocols,handler);
                }
                return resp.sendNotFound();
		});
	} //end ws
 
}
```
### (3).DefaultHttpServerRoutes
```
final class DefaultHttpServerRoutes implements HttpServerRoutes {
    private final CopyOnWriteArrayList<HttpRouteHandler> handlers = new CopyOnWriteArrayList<>();
    
@Override
	public HttpServerRoutes directory(String uri, Path directory,
			Function<HttpServerResponse, HttpServerResponse> interceptor) {
		Objects.requireNonNull(directory, "directory");
		return route(HttpPredicate.prefix(uri), (req, resp) -> {

			String prefix = URI.create(req.uri())
			                   .getPath()
			                   .replaceFirst(uri, "");

			if(prefix.charAt(0) == '/'){
				prefix = prefix.substring(1);
			}

			Path p = directory.resolve(prefix);
			if (Files.isReadable(p)) {

				if (interceptor != null) {
					return interceptor.apply(resp)
					                  .sendFile(p);
				}
				return resp.sendFile(p);
			}

			return resp.sendNotFound();
		});
	}

	@Override
	public HttpServerRoutes route(Predicate<? super HttpServerRequest> condition,
			BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler) {
		Objects.requireNonNull(condition, "condition");
		Objects.requireNonNull(handler, "handler");
            // *******************************************
            // 3.添加路由信息到:handlers中
            // *******************************************
		if (condition instanceof HttpPredicate) {
			handlers.add(new HttpRouteHandler(condition,
					handler,
					(HttpPredicate) condition));
		}
		else {
			handlers.add(new HttpRouteHandler(condition, handler, null));
		}
		return this;
	}

	@Override
	public Publisher<Void> apply(HttpServerRequest request, HttpServerResponse response) {
		final Iterator<HttpRouteHandler> iterator = handlers.iterator();
		HttpRouteHandler cursor;

		try {
			while (iterator.hasNext()) {
				cursor = iterator.next();
				if (cursor.test(request)) {
					return cursor.apply(request, response);
				}
			}
		}
		catch (Throwable t) {
			Exceptions.throwIfFatal(t);
			return Mono.error(t); //500
		}

		return response.sendNotFound();
	}


	static final class HttpRouteHandler
			implements BiFunction<HttpServerRequest, HttpServerResponse, Publisher<Void>>,
			           Predicate<HttpServerRequest> {

		final Predicate<? super HttpServerRequest>          condition;
		final BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>>
		                                                    handler;
		final Function<? super String, Map<String, String>> resolver;

		HttpRouteHandler(Predicate<? super HttpServerRequest> condition,
				BiFunction<? super HttpServerRequest, ? super HttpServerResponse, ? extends Publisher<Void>> handler,
				Function<? super String, Map<String, String>> resolver) {
			this.condition = Objects.requireNonNull(condition, "condition");
			this.handler = Objects.requireNonNull(handler, "handler");
			this.resolver = resolver;
		}

		@Override
		public Publisher<Void> apply(HttpServerRequest request,
				HttpServerResponse response) {
			return handler.apply(request.paramsResolver(resolver), response);
		}

		@Override
		public boolean test(HttpServerRequest o) {
			return condition.test(o);
		}
	} //end DefaultHttpServerRoutes.HttpRouteHandler
    
}
```
### (4).HttpServer
```
public final class HttpServer
		implements NettyConnector<HttpServerRequest, HttpServerResponse> {
    
    // 2.启动并配置路由
    public BlockingNettyContext startRouter(Consumer<? super HttpServerRoutes> routesBuilder) {
            Objects.requireNonNull(routesBuilder, "routeBuilder");
            // DefaultHttpServerRoutes
            HttpServerRoutes routes = HttpServerRoutes.newRoutes();
           // 实际是回调自己写的配置信息(
          // new Consumer<HttpServerRoutes>() {
           //   accept    
	       //    public void accept(HttpServerRoutes routes) {
	       //         routes.get("/hello", (req, resp) -> resp.sendString(Mono.just("hello!")));
	       //         routes.get("/world", (req, resp) -> resp.sendString(Mono.just("world!")));
		//    };
		// } //end 匿名内部类
           // )
            routesBuilder.accept(routes);
            // 后面留一章讲解这里                
            return start(routes);
    } //end startRouter
                  
}
```
### (5).总结
1. 用户通过lambad配置路由信息
2. DefaultHttpServerRoutes(该类内部有一个数组,保存有所有的路由信息)
3. 调用lambad配置路由信息的accept(DefaultHttpServerRoutes routes)方法,进行路由配置
4. 最终会调用:DefaultHttpServerRoutes.route()方法,保存所有的路由信息.