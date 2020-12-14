---
layout: post
title: 'ElasticSearch 源码Node-1(四)'
date: 2020-12-14
author: 李新
tags: ElasticSearch源码
---

### (1). 概述
> 在上一节,剖析到:Bootstrap.init方法,在内部,它首先会:new Node,然后,再调用:Node.start();
> 这一节,以及后面几节,都会剖析Node.

### (2). Bootstrap创建Node
```
private void setup(
        boolean addShutdownHook, 
        Environment environment) throws BootstrapException {
    // 创建Node        
    node = new Node(environment) {
            @Override
            protected void validateNodeBeforeAcceptingRequests(
                final BootstrapContext context,
                final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {
                BootstrapChecks.check(context, boundTransportAddress, checks);
            }
    };// end node
}
```
### (3). Node构造器
```
public Node(Environment environment) {
    this(environment, Collections.emptyList(), true);
}

protected Node(
            final Environment environment, 
            Collection<Class<? extends Plugin>> classpathPlugins, 
            boolean forbidPrivateIndexSettings) {
    
    logger = LogManager.getLogger(Node.class);


}// end Node
```
### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 
    
### (10). 

### (11). 

### (12). 

### (13). 

### (14). 

### (15). 

