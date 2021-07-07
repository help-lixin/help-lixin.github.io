---
layout: post
title: 'Spring Cache 入门(一)'
date: 2021-07-05
author: 李新
tags:  Spring 
---

### (1). Spring Cache是什么
Spring比较喜欢做的一件事情就是:定义规范(抽象),然后,相同类型的产品对规范进行实现(类似于:桥梁模式),可以理解:Spring Cache是Spring针对Cache定义的一套规范.    
比如:你在工作中要用到:Redis/Memcache/Tair/Guava...等等,使用Spring Cache你可以无缝自由切换(组合)这些缓存的实现.     
<font color='red'>底层实则是:对标有注解的类进行AOP拦截(注意:private/final/this call,Spring是无法代理的.)</font>   

### (2). @Cacheable注解
> 使用@Cacheable标注的方法,Spring在每次执行前都会检查Cache中是否存在相同key的缓存元素,如果存在就不再执行该方法,而是直接从缓存中获取结果进行返回,否则才会执行并将返回结果存入指定的缓存中.   

```
@Cacheable(
  // 定义缓存的名称
  cacheNames="users",
  
  // 使用key属性自定义key
  // #p0                : 通过下标读取参数的第一个元数
  // #p0.id             : 通过下标读取参数的第一个元数的属性id
  // #user.id           : 通过参数名称读取属性id
  // #root.methodName   : 当前方法名
  // #root.method.name  : 当前方法
  // #root.target       : 目标对象
  // #root.targetClass  : 目标Class
  // #root.args[0]      : 方法参数组
  // #root.caches[0].name   : 当前被调用的方法使用的Cache(caceNames)
  key="#p0" , 
  
  // 通过condition指定符合条件,则缓存,不符合条件的数据,则不缓存.
  condition = "#user.id % 2 == 0" ,
  
  // 告诉底层的缓存提供者将缓存的入口锁住,这样就只能有一个线程计算操作的结果值,而其它线程需要等待,这样就避免了 n-1 次数据库访问.
  // sync=true,可以避免缓存击穿问题
  // Cache.get(Object key, Callable<T> valueLoader)
  // 实际上是依赖于java的synchronized关键字
  // synchronized <T> T get(Object key, Callable<T> valueLoader)
  // sync = false
)
```
### (4). @CachePut注解
> @CachePut与@Cacheable不同的是,@CachePut标注的方法在执行前不会去检查缓存中是否存在之前执行过的结果,而是每次都会执行该方法,并将执行结果以键值对的形式存入指定的缓存中.  

### (5). @CacheEvict注解
>  @CacheEvict是用来标注在需要清除缓存元素

```
@CacheEvict(
  // allEntries为true时,Spring Cache将忽略指定的key,清除所有的元素.
  allEntries = false,
  // 清除操作默认是在"调用代理类的方法成功后"执行之后触发清除操作.
  // 如果被代理后的方法抛出异常而未能成功返回时,则不会触发清除操作.
  // 使用beforeInvocation可以改变触发清除操作的时间.
  // 当我们指定该属性值为true时,Spring会在调用"代理方法之前"清除缓存中的指定元素. 
  beforeInvocation = false
)
```
### (6). @Caching注解
> @Caching注解可以让我们在一个方法或者类上同时指定多个Spring Cache相关的注解.
> 其拥有三个属性:cacheable、put和evict,分别用于指定@Cacheable、@CachePut和@CacheEvict. 
