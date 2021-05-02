---
layout: post
title: 'Apache HttpClient OOM'
date: 2020-12-24
author: 李新
tags: OOM
---

### (1). 问题
> 今天在外面,朋友给了一个dump,怀疑是内存泄露了,让我帮忙排查下.
> 由于我无法得知:开发人员的代码和业务逻辑,只能凭借泄露的信息来分析大概的业务代码.  
> 业务代码大概为:  
> 1. Tomcat接受请求.  
> 2. Apache HttpClient转发请求.    
> 3. 泄露提示为:sun.security.ssl.SSLSessionContextImpl 或 java.util.HashMap$Node[]   

### (2). Eclipse MAT打开可能泄露的Dump文件
!["内存泄露"](/assets/oom/imgs/eclipse-mat-leak-1.jpg)

### (3). Eclipse MAT打开可能泄露的Dump文件,查看泄露详细
!["内存泄露详解"](/assets/oom/imgs/eclipse-mat-leak-2.jpg)

### (4). 总结
> 从上图就能分析出来,属于ConnectionPool的问题.   
> 应该是开发借用了Connection,但是,未归还Connection.   
> 我没有用过HttpClient,但是我可以肯定的是:    
> MultiThreadedHttpConnectionManager应该有相应的close方法.去关闭这些(IDLE)空闲的连接.    
> 按照这个思路,后来,朋友反映:问题得已解决.