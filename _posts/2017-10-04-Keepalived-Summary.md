---
layout: post
title: 'Keepalived 介绍'
date: 2017-10-04
author: 李新
tags: Keepalived
---

### (1). 前言
> 高可用HA（High Availability)是分布式系统架构设计中必须考虑的因素之一,它通常是指:通过设计减少系统不能提供服务的时间.如果一个系统能够一直提供服务,那么这个可用性则是百分之百,但是天有不测风云.所以,我们只能尽可能的去减少服务的故障.  
> 在生产环境上很多时候是以Nginx做反向代理对外提供服务,但是某一天Nginx遇到故障,如:服务器宕机.Nginx所承载的服务都将无法访问.  
> 虽然我们无法保证服务器百分之百可用,但是也得想办法避免这种悲剧,今天我们使用keepalived来实现Nginx的高可用.  

### (2). Keepalived是什么?
> Keepalived软件起初是专为LVS负载均衡软件设计的,用来管理并监控LVS集群系统中各个服务节点的状态,后来又加入了可以实现高可用的VRRP(Virtual Router Redundancy Protocol,虚拟路由器冗余协议)功能.
> 因此,Keepalived除了能够管理LVS软件外,还可以作为其他服务(例如:Nginx、Haproxy、MySQL等)的高可用解决方案软件.  

### (3). Keepalived原理
> Keepalived高可用服务之间的故障切换转移是通过:VRRP来实现的.
> 在Keepalived服务正常工作时,Master会不断地向Backup节点发送(多播的方式)心跳消息,用以告诉Backup节点自己还活着.
> 当主Master节点发生故障时,Backup节点没有接受到Master节点的心跳,于是(Backup)调用自身的接管程序,接管主Master节点的IP资源及服务.
> 而当主Master节点恢复时,备Backup节点又会释放主节点故障时自身接管的IP资源及服务,恢复到原来的备用角色.    

### (4). Keepalived学习目录
> ["Keepalived 主备详解(二)"](/2017/10/04/Keepalived-Master-Backup.html)    
> ["Keepalived 双机热备(三)"](/2017/10/04/Keepalived-Double-Master-Backup.html)