---
layout: post
title: 'Guice 概述(一)'
date: 2020-12-12
author: 李新
tags: Guice
---

### (1). Binder
> <font color='red'>Binding其实就是一个接口和相应的实现类的映射关系.</font>
> 可以将接口与实现类进行映射.     
> 可以将接口与实例对象进行映射.     
> 可以半接口与Provider(工厂)进行映射.   
> 案例代码如下:
```
bind(UserService.class).annotatedWith(Names.named("userOne")).to(UserServiceOneImpl.class);
```

### (2). Injector
> <font color='red'>Injector通常在客户端(Client)使用</font>

```
injector.getInstance(Key.get(UserService.class, Names.named("userTwo")));
```

### (3). Module
> Module对象<font color='red'>维护一组Bindings</font>,在一个应用中可以有多个Module.Injector会通过Module来获得Bindings,只需要继承AbstractModule并重写configure即可.

### (4). Guice
> 客户端(Client)是通过Guice类直接和其他Object进行交互的.需要注意:    

```
# (Guice.createInjector)该方法接受一个Module对象作为参数,Module类必须要重写configure方法.
# 该方法用于传递一个默认Binder对象.
# 该Binder对象为应用程序填充特定的Bindings.
# 当客户端(Client)调用Injector.getInstance()方法时.
# Injector会维护Bindr对象与原生对象的关系.

Injector injector = Guice.createInjector(new BeanConfig());
```