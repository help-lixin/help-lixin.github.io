---
layout: post
title: 'Canal Server源码之四(CanalInstanceWithSpring)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). Canal逻辑架构图
!["Canal逻辑架构图"](https://camo.githubusercontent.com/08a9e8b2cd33638ae4b3f71e8c331660eb2edf1bab567d88afc509a02bd18c83/687474703a2f2f646c2e69746579652e636f6d2f75706c6f61642f6174746163686d656e742f303038322f353138372f63383936663831632d623563642d336164362d383034362d3436346664333865346436632e6a7067)
### (2). CanalInstanceWithSpring类的继承图
!["CanalInstanceWithSpring类的继承图"](/assets/canal/imgs/CanalInstanceWithSpring-UML-Class.jpg)

### (3). file-instance.xml内部结构
> file-instance.xml 定义了:instance对象的的依赖关系   

```
<bean id="eventParser" class="com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser"> <!-- ... --> </bean>

<bean id="eventSink" class="com.alibaba.otter.canal.sink.entry.EntryEventSink"> <!-- ... -->  </bean>

<bean id="eventStore" class="com.alibaba.otter.canal.store.memory.MemoryEventStoreWithBuffer"> <!-- ... -->  </bean>

<bean id="metaManager" class="com.alibaba.otter.canal.meta.FileMixedMetaManager"> <!-- ... -->   </bean>

<bean id="alarmHandler" class="com.alibaba.otter.canal.common.alarm.LogAlarmHandler" />

<bean id="instance" class="com.alibaba.otter.canal.instance.spring.CanalInstanceWithSpring">
    <property name="destination" value="${canal.instance.destination}" />
    <property name="eventParser">
        <ref local="eventParser" />
    </property>
    <property name="eventSink">
        <ref local="eventSink" />
    </property>
    <property name="eventStore">
        <ref local="eventStore" />
    </property>
    <property name="metaManager">
        <ref local="metaManager" />
    </property>
    <property name="alarmHandler">
        <ref local="alarmHandler" />
    </property>
</bean>

```

### (4). CanalInstanceWithSpring入口
>  在前面分析,我们知道:   
> 1. SpringCanalInstanceGenerator会创建ApplicationContext加载XML,并从XML获取实例对象:CanalInstanceWithSpring.   
> 2. 调用:CanalInstanceWithSpring.start方法  
> 3. 在这里,主要剖析CanalInstanceWithSpring.start方法. 

### (5). CanalInstanceWithSpring.start方法
```
public void start() {
    super.start();
    // 元数据管理
    if (!metaManager.isStart()) {
        metaManager.start();
    }

    // 告警处理
    if (!alarmHandler.isStart()) {
        alarmHandler.start();
    }

    // 事件存储
    if (!eventStore.isStart()) {
        eventStore.start();
    }

    // 事件加工处理
    if (!eventSink.isStart()) {
        eventSink.start();
    }

    // 事件解析
    if (!eventParser.isStart()) {
        beforeStartEventParser(eventParser);
        eventParser.start();
        afterStartEventParser(eventParser);
    }
    logger.info("start successful....");
}

```
### (6). 总结
> 启动时依次调用:   
> CanalMetaManager.start  
> CanalAlarmHandler.start  
> CanalEventStore.start  
> CanalEventSink.start  
> CanalEventParser.start  
> 只有当所有都准备就绪后:CanalEventParser才能开始启动,自然,关闭是肯定是按相反的方向进行关闭.   

