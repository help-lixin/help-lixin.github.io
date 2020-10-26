---
layout: post
title: 'Project Reactor Netty Hello World'
date: 2020-07-21
author: 李新
tags: ReactorNetty
---

### (1).ReactorNettyTest
```
package help.lixin.examples;

import java.util.concurrent.CountDownLatch;

import org.junit.Test;

import reactor.core.publisher.Mono;
import reactor.ipc.netty.http.server.HttpServer;
import reactor.ipc.netty.tcp.BlockingNettyContext;

public class ReactorNettyTest {

	@Test
	public void tetsHello() {
		BlockingNettyContext facade = // 创建HttpServer并配置路由
                HttpServer.create(8080).startRouter(routes -> {
                    routes.get("/hello", (req, resp) -> resp.sendString(Mono.just("hello!")));
                    routes.get("/world", (req, resp) -> resp.sendString(Mono.just("world!")));
                });
              
		CountDownLatch latch = new CountDownLatch(1);
		try {
			latch.await();
		} catch (InterruptedException ignore) {
		}
	}
}
```
### (2). 访问
!["hello"](/assets/project-reactory-netty/imgs/reactory-netty-hello.png)
!["world"](/assets/project-reactory-netty/imgs/reactory-netty-world.png)