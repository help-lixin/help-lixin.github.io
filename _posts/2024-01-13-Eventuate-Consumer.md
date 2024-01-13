---
layout: post
title: 'Eventuate Command Consumer消费者' 
date: 2024-01-12
author: 李新
tags:  Eventuate
---

### (1). 背景

这一篇,主要剖析:消费者消息的过程(不剖析具体代码行),从中能发现,消费者是要依赖于具体的存储引擎,在这里以Kafka为例

### (2). Eventuate Command消费者类图

![Eventuate Command消费者类图](/assets/eventuate/imgs/Consumer-ClassDiagram.jpg)

### (3). Eventuate Command消费者时序图

![Eventuate Command消费者时序图](/assets/eventuate/imgs/Consumer-SequenceDiagram.jpg)

### (4). Eventuate Command消费者总结
```
其实,总体的实现思路是订阅MQ中的消息而已,只是中间多了一些环节(责任链模式),可以对消息进行幂等性处理之类的. 
```
