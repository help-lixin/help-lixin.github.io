---
layout: post
title: 'GRPC总结' 
date: 2022-09-19
author: 李新
tags:  GRPC
---


### (1). GRPC简介
GRPC是一个高性能、开源、通用的RPC框架,由Google推出,基于HTTP2协议标准设计开发,默认采用Protocol Buffers数据序列化协议,支持多种开发语言,GRPC提供了一种简单的方法来精确的定义服务,并且为客户端和服务端自动生成可靠的功能库.   
在GRPC客户端可以直接调用不同服务器上的远程程序,使得看起来就像调用本地程序一样,很容易去构建分布式应用和服务.和很多RPC系统一样,服务端负责实现定义好的接口并处理客户端的请求,客户端根据接口描述直接调用需要的服务.客户端和服务端可以分别使用GRPC支持的不同语言实现. 

### (2). 主要特性
+ 强大的IDL
  GRPC使用ProtoBuf来定义服务,ProtoBuf是由Google开发的一种数据序列化协议(类似于XML、JSON、hessian).ProtoBuf能够将数据进行序列化,并广泛应用在数据存储、通信协议等方面.  
+ 多语言支持
  GRPC支持多种语言,并能够基于语言自动生成客户端和服务端功能库.目前已提供了C版本grpc、Java版本grpc-java和Go版本grpc-go,其它语言的版本正在积极开发中,其中,grpc支持C、C++、Node.js、Python、Ruby、Objective-C、PHP和C#等语言,grpc-java已经支持Android开发. 
+ HTTP2
  GRPC基于HTTP2标准设计,所以相对于其他RPC框架,GRPC带来了更多强大功能,如双向流、头部压缩、多复用请求等.这些功能给移动设备带来重大益处,如节省带宽、降低TCP链接次数、节省CPU使用和延长电池寿命等.同时,gRPC还能够提高了云端服务和Web应用的性能.GRPC既能够在客户端应用,也能够在服务器端应用,从而以透明的方式实现客户端和服务器端的通信和简化通信系统的构建.  
### (3). GRPC学习目录
+ ["Mac环境安装Protoc"](/2022/09/19/Mac-Install-Protoc.html) 
+ ["GRPC HelloWorld入门"](/2022/09/19/GRPC-HelloWorld.html)
### (4). 
