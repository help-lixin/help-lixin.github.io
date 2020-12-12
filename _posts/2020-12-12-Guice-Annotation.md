---
layout: post
title: 'Guice 注解(二)'
date: 2020-12-12
author: 李新
tags: Guice
---

### (1). @ImplementedBy
> 该注解用于为<font color='red'>接口指定指定实现类(注解在接口上).</font>    
> 例如,一个接口有多个实现类,希望为接口指定一个默认的实现类.    

### (2). @Inject
> 该注解用于(Client)注入一个实例.注解可以配置在:构造器,set方法,属性上.   

### (3). @Provider
> 该注解用于在接口上指定对象的流程(<font color='red'>为接口指定对象工厂</font>).

### (4). @Singleton
> 在默认情况下,调用:Injector.getInstance(),每一次都会返回一个新创建的对象.如果想要使用单例模式(Singleton Pattern)来获取对象.


