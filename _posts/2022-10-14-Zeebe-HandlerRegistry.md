---
layout: post
title: 'Zeebe ClusterServicesStep源码之HandlerRegistry(十四)' 
date: 2022-10-14
author: 李新
tags:  Zeebe
---

### (1). 概述
前面对RemoteServerConnection的源码进行剖析,它的主要职责是:dispatch和reply,其中,在dispatch里又委托给了:HandlerRegistry处理,在这一小篇,主要剖析:HandlerRegistry.  

### (2). HandlerRegistry
```
final class HandlerRegistry {
  private final Map<String, BiConsumer<ProtocolRequest, ServerConnection>> handlers = new ConcurrentHashMap<>();

  void register(final String type, final BiConsumer<ProtocolRequest, ServerConnection> handler) {
    handlers.put(type, handler);
  }

  void unregister(final String type) {
    handlers.remove(type);
  }

  BiConsumer<ProtocolRequest, ServerConnection> get(final String type) {
    return handlers.get(type);
  }
}
```
### (3). 总结
HandlerRegistry是一个中介模式,仅仅只是对载体进行存储而已,但是,是否值得考虑下,到底是如何与业务通信的呢?下一小篇重点剖析.