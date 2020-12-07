---
layout: post
title: 'Chrome Debug Protocol 总结'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). CDP源码目录
["CDP Hello World"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-HelloWorld.html)

["CDP Launch过程"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-ChromeService.launch-2.html)

["CDP 创建Chrome系统进程"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-ProcessLauncher-3.html)

["CDP 创建ChromeTab(1)"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-ChromeService-4.1.html)

["CDP 创建WebSocket"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-ChromeService-4.2.html)

["CDP Network配置事件"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-Network-1.html)

["CDP Network网络启用"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-Network-2.html)

["CDP WebSocket Client业务处理"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-Network-3.html)

["CDP Page访问报文详解"](https://www.lixin.help/2020/12/06/Chrome-Debug-Protocol-Page.html)


### (2). 为什要看CDP源码,并绕过WebDriver协议?
> 以前有看过Selenium源码,总体来说:<font color='red'>它的业务模型我不太喜欢.主要有以下几点原因:        
> 1. Selenium编程,经常看到在代码里添加Sleep之类的语句,难道就没有等待浏览器执行完,再回调业务,告诉业务已经执行完了吗?然后,业务再进行处理吗?也有可能是我对Selenium不太了解!       
> 2. 阻塞式编程,阻塞式编程有优点:在理解上容易,但缺点:CPU利用率下降.      
> 3. 为爬虫或者自动化测试留下伏笔.     
> 4. 同时,也是:知其然知其所以然.    

### (3). CDP用到了哪些设计模式
> 1. 动态代理.  
> 2. 观察者模式.    
> 3. Builde模式.    
> 4. 工厂模式.    

### (4). 爬虫实现思路 
> 实现思路如下(Boss/Work/MetaCenter):   
> 1. 在宿主机上创建:N个Work(Java)进程.   
> 2. 每个Work进程里创建N个:ChromeTab,每个ChromeTab有一个ws地址.     
> 3. Work进程实时向MetaCenter汇报当前宿主机的情况,自己的任务情况.
> 4. Work接受任务去实现真正的爬,可以利用:Disruptor把"爬取"和"解析分离"(<font color='red'>CDP是可以拿到JS解析析之后的DOM节点的,参考:DumpHtmlFromPageExample</font>).   
> 4. Master接受任务,获取:MetaCenter的指标,再分派任务给:Work.

### (5). 自动化测试实现思路 
> 自动化测试,实际应该可以做到(可视化)任务编排的,这部份需要做一个比较好的设计.后续有时间,会将UML设计图画出来.  
