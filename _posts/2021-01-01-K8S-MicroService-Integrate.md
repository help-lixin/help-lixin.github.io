---
layout: post
title: 'Kubernetes 微服务集成(解决方案)'
date: 2021-01-01
author: 李新
tags: K8S Spring SpringCloud
---

### (1). K8S与微服务集成
> 近来一直在用K8S,发现K8S有服务发现功能(Service),而Spring Cloud也有相应的服务发现功能,而两者的功能是不兼容的.    
> 故提出几套解决方案. 

### (2). 方案一
> 约定优于配置:
> 1. 从开发环境到生产环境,约定好K8S中的Service(domain)名称.    
> 2. 不依赖任何的服务(Zookeeper/Eureka/Nacos...)发现组件,<font color='red'>放弃微服务所提供的服务发现与注册组件</font>.   
> 3. 业务端(Java)在写代码时,直接用约定的Service(domain)名称+端口,去调用相应的服务即可.   
> 4. 缺陷: 需要用service:port才能访问.    
> 5. 原理:在一个K8S集群内部,容器之间是可以通过:Service(domain)名称,解析到相应的IP的(domain到DNS的解析),该功能由:kube-dns实现.  

### (3). 方案二
> Ingress + Spring Cloud Gateway + Eureka(Zookeeper/Nacos) + 业务微服务:   
> 1. eureka 和 业务微服务,<font color='red'>不使用Service功能(也就是不暴露服务,仅内部访问).</font>      
> 2. 业务微服务 和 Gateway 都向 eureka注册,这时在注册中心的元数据是:PodIP:port.   
> 3. Spring Cloud Gateway通过Service(NodePort)暴露出IP和端口.     
> 4. Ingress接受请求,全部转发到:Gateway上,再由:Gateway进行分发到业务微服务上.    
> 5. 注意:eureka注册时,需要稍微注意一些.它必须是有状态的,而且还要是基于域名的.      
> 6. 优势:不是所有的微服务都要使用Service功能.访方案:占用Node(宿主机)的端口也比较少.  
> 7. 原理:相当于Gateway往Nginx的upstream中注册,而Gateway和Pod是在同一网段,自然也能相互访问.  

### (4). 方案三
> Spring提供了:Spring Cloud Kubernetes解决方案(https://spring.io/projects/spring-cloud-kubernetes).   
> 该组件的原理:把微服注册和服务发现,基于:Kubenetes Service实现了一套,它和Eureka/Zookeeper/Nacos等是平级的.  

### (5). 方案四
> K8S所提供的解决方案:  
> Inggress + Service + 业务微服务 
> 2. 不依赖任何的服务(Zookeeper/Eureka/Nacos...)发现组件,<font color='red'>放弃微服务所提供的服务发现与注册组件</font>.   
> 3. 所有的业务微服务都使用Service和Ingress(不使用方案二中的:Gateway).   
> 4. 如果,业务有需求要对网关进行编排,可对Nginx进行二次开发(OpenRestry).   

### (6). 方案五
> K8S中Service的网络类型有三种:cluertip,nodeport,loadbanlance,可以选择自研:loadbanlance.   