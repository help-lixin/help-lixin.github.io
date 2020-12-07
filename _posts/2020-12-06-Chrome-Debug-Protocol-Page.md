---
layout: post
title: 'Chrome Debug Protocol--Page(六)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). 概述
> 前面针对:websocket发送请求和websocket接受请求进行了剖析.这一节,主要剖析下:page.navigate("http://github.com"),以及network.onLoadingFinished事件到底是如何触发的.


### (2). page.navigate("http://github.com")
> 发送报文

```
// {"id":2,"method":"Page.navigate","params":{"url":"http://github.com"}}
// ChromeDevToolsServiceImpl.invoke
webSocketService.send(OBJECT_MAPPER.writeValueAsString(methodInvocation));
```

> 接受报文

```
// ChromeDevToolsServiceImpl.accept
// {"id":2,"result":{"frameId":"97464506B7CA9FB152249411BB3B94CE","loaderId":"74339EC0A949B59AA7BFC791136AFBBE"}}
```

### (3). 总结
> 不管你(websocket client)是否订阅相应的Event,Chrome都会发布Event事件的.只是你不订阅的话,相当于是忽略了这个事件.<font color='red'>需要注意:一个ChromeDevToolsService可以包含N个Event,而事件是全局性的,不是一个请求配置一个事件来着的.</font>    
> 其实,最后:所有的事情,最终都是走websocket.