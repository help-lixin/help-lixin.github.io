---
layout: post
title: 'Tigase介绍(一)' 
date: 2021-07-17
author: 李新
tags:  Tigase 
---

### (1). Tigase是什么
是一个轻量级的可伸缩的Jabber/XMPP服务器.无需其他第三方库支持,可以处理非常高的复杂和大量的用户数,可以根据需要进行水平扩展. 

### (2). Tigase优势
+ 全面: tigase完全实现了XMPP协议,除了全面实施的两个核心协议,它支持大多数的你可能永远都需要的扩展协议.  
+ 开源: Tigase是开源的,如果你有有那能力,你可以定制自己的XMPPServer. 
+ 可靠: Tigase在设计的时候已经充分的考虑到他的容错能力,它的代码可以自动处理错误保证你的应用尽可能的不down掉.它已经被部署在很多企业的生产环境,并且被测试通过. 
+ 集群支持: Tigase支持集群,通过集群多个节点在一块,支持百万级用户并发,Seesmic是最好的例子. 
+ 性能和效率: Tigase支持单机50W并发,另外,Tigase还可以部署在只有10M内存的机器上.Tigase支持虚拟域,单服务器可以安装多个域. 
+ 监控策略: Tigase提供很多开箱即用的自我监控功能.你可以通过XMPP，JMX，Http和SNMP查看服务运行时的错误.
+ 可扩展: 设计之初,Tigase就是可扩展的,支持自定义插件,开发者可以很轻松的扩展他的功能.Tigase支持多种语言插件开发. 
+ 轻便: 使用Java编写的,它可以在Linux,Windows,Solaris和Mac OSX上运行.
+ 国际化: Tigase支持UTF-8,允许任何语言.

### (2). Tigase学习目录
+ ["Tigase安装与部署(二)"](/2021/07/17/Tigase-Install-Deploy.html)   
