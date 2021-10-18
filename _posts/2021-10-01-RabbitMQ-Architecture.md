---
layout: post
title: 'RabbitMQ架构以及基本概念(一)' 
date: 2021-10-01
author: 李新
tags:  RabbitMQ
---

### (1). RabbitMQ是什么
RabbitMQ是一个开源的AMQP实现,服务器端用Erlang语言编写,支持多种客户端,如:Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等,支持AJAX.用于在分布式系统中存储转发消息,具有很高的易用性和可用性.       
AMQP,即Advanced Message Queuing Protocol,高级消息队列协议,是应用层协议的一个开放标准,为面向消息的中间件设计.消息中间件主要用于组件之间的解耦和通讯.            
AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全.          

### (2). RabbitMQ架构图
!["RabbitMQ架构图"](/assets/rabbitmq/imgs/rabbitmq-architecture.png)

### (3). RabbitMQ的基本概念
+ 消息(Message): 由有效载荷(playload)和标签(label)组成,说白一点就是数据的载体.   
+ 生产者(producer): 创建消息,发布到代理服务器(Message Broker).   
+ 消费者(consumer): 连接到代理服务器,并订阅到队列(queue)上,代理服务器将发送消息给一个订阅的/监听的消费者,消费者其只能接收消息的一部分:有效载荷（playload）.
+ 代理服务器(Message Broker): 接收和分发消息的应用,RabbitMQ Server就是消息代理服务器,其中包含概念很多,以RabbitMQ为例:信道(channel)、队列(queue)、交换器(exchange)、路由键(routing key)、绑定(binding key)、虚拟主机(vhost)等.  
  - 信道(channel): 应用程序和代理服务器之间TCP连接内的虚拟连接,解决TCP连接数量限制及降低TCP连接代价.每个信道有一个ID,其概念与“频分多路复用”类似.  
  - 队列(queue): 消息最终到达队列中,等待消费者消费.
  - 交换器(exchange): 消息到达代理服务器的第一站,根据分发规则,匹配查询表中的(routing key)路由键(Fanout Exchange除外),分发消息到队列(queue)中去.常用的类型有: direct (point-to-point), topic (publish-subscribe) and fanout (multicast).  
  - 路由键(routing key): 消息发送给交换器时,消息将拥有一个路由键(默认为空),交换器根据这个路由键将消息发送到匹配的队列中.  
  - 绑定键(binding key): 队列需要通过绑定键(默认为空)绑定到交换器上,交换器将消息的路由键与所绑定队列的绑定键进行匹配,正确匹配的消息将发送到队列中.路由键是偏向生产的概念,而绑定键是偏向消费的概念.  
  - 虚拟主机(vhost): AMQP概念的基础,其本质上就是对数据进行权限控制(比如:不同vhost对应不同的租户)   

### (4). 交换机类型
+ Direct Exchange: 直连交换机,该交换机会完全匹配routing key(完全一致才算匹配),将消息转发到匹配的队列上.  
+ Topic Exchange: 主题交换机,该交换机会模糊匹配routing key,将消息转发到匹配的队列上.  
  - name.* : 匹配以name.开头的,后面是任意不含.字符串,比如:name.abc.  
  - name.# : 匹配以name.开头的,后面是任意字符串,比如:name.abc、name.abc.abc.
+ Fanout Exchange: 广播交换机,该交换机不管routing key将消息转发到所有绑定到该交换机的队列上(广播).
+ Headers Exchange: headers类型的交换机分发消息不依赖routingKey,是使用发送消息时basicProperties对象中的headers来匹配的.headers是一个键值对类型,发送者发送消息时将这些键值对放到basicProperties对象中的headers字段中,队列绑定交换机时绑定一些键值对,当两者匹配时,队列就可以收到消息.匹配模式有两种,在队列绑定到交换机时用x-match来指定,all代表定义的多个键值对都要满足,而any则代码只要满足一个就可以了.

