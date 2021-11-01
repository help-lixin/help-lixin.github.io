---
layout: post
title: 'SpringWebFlow概念介绍' 
date: 2021-10-25
author: 李新
tags:  SpringWebFlow
---

### (1). 简介
最近在看CAS单点登录的源码,CAS底层实际是使用了Spring Web Flow,所以,需要对Web Flow的基本概念有一定的了解.  

### (2). Spring Web Flow是什么
Spring Web Flow是基于SpringMVC的基础上开发的,允许web应用对flows实现.一个Flow封装了一些列的操作步骤,引导用户执行某些业务任务.Flows可以包含多个HTTP的请求,它是有状态、能处理事务的数据,是可重用的,本质上是可以动态的长时间运行.   
### (3). Spring Web Flow的基本组件
+ Actions : 一个可执行任务和有返回结果的组件(业务代码).  
+ Transitions : 把Flow从一个状态转到另外一个状态,转换可能对整个FLow有全局影响.  
+ Views : 把表示层展现到客户端显示的组件
+ Decisions(决策) : 路由到其他的Flow,可以有逻辑的判定组件.  
+ Subflow: 子流程状态会在当前正在运行的流程上下文中启动一个新的流程.
### (4). Spring Web Flow的核心类
+ FlowRegistry : FlowRegistry是存放Flow的仓库,每个定义Flow的XML文档被解析后,都会被分配一个id,并以FlowDefinition对象的形式存放在FlowResigtry中.  
+ FlowExecutor : FlowExecutor是Spring Web Flow 的一个核心接口,启动某个flow,都要通过这个接口来进行.  
+ FlowBuilder Services : 用于对flow的一些基础设置(比如:所使用的EL表达式/ViewRsolve).   
### (5). 总结