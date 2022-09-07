---
layout: post
title: 'Spring StateMachine源码总结(四)' 
date: 2022-09-07
author: 李新
tags:  SpringStateMachine
---

### (1). 概述
Spring StateMachine的底层原理到底是啥?是否好奇?在没有看源码之前,我以为会运用:观察者模式来做设计,不过看完底层部份源码后,与我的想法冲突还是比较大的.   

### (2). 底层原理
```
1. 收集注解(@WithStateMachine/@OnTransition),通过StateMachineHandler(Object/Method)包裹并注册到Spring容器里. 
2. 收集,开发人员,通过EnumStateMachineConfigurerAdapter(StateMachineTransitionConfigurer)配置Source与Event,相当于有一个所有Source的记录表.
3. 当触发事件时(StateMachine.sendEvent(OrderEvents.PAY)),遍历第二步所有的所有的Source与Event的关联,如果Event相同,则调用:StateMachineHandler类的方法.
```
### (3). 源码最底层在哪?
```
# 负责与StateMachineHandler打交道的一个桥梁.
org.springframework.statemachine.processor.StateMachineHandlerCallHelper

# 状态机处理对象.
org.springframework.statemachine.support.StateMachineObjectSupport
```
### (4). 总结
Spring StateMachine官网基本上是没有相应的架构文档介绍来着的,只能,从一个简单的案例入门,然后,简单的剖析下源码之后,通篇来看源码,感觉有点杂乱无章(类里大量的内容都全部集中在StateMachine里,没有更进一步抽象),也有可能是自己的能力不足.   