---
layout: post
title: 'Playwright源码之PlaywrightImpl初始化(二)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
在这一小节,开始剖析Playwright源码,先从:PlaywrightImpl类开始,要深入了解下,Playwright底层到底做了什么.   
### (2). PlaywrightImpl.create
```
# 通过静态方法,快建构建:Playwright
public static PlaywrightImpl create(CreateOptions options) {
	return createImpl(options, false);
}
```
### (3). PlaywrightImpl.createImpl
```
public static PlaywrightImpl createImpl(CreateOptions options, boolean forceNewDriverInstanceForTests) {
	// 创建空的env
	Map<String, String> env = Collections.emptyMap();
	if (options != null && options.env != null) {
	  env = options.env;
	}
	
	// *******************************************************
	// forceNewDriverInstanceForTests = false
	// *******************************************************
	Driver driver = forceNewDriverInstanceForTests ?
	  //  强制创建目录,并安装Driver
	  Driver.createAndInstall(env, true) :
	  
	  // *******************************************************
	  // 安装Driver(这一步实际是下载Driver)
	  // *******************************************************
	  Driver.ensureDriverInstalled(env, true);
	try {
		
	  
	  // *******************************************************
	  // playwright.sh 	run-driver
	  // 通过:java.lang.Process来管理浏览器与浏览器通信.
	  // *******************************************************
	  ProcessBuilder pb = driver.createProcessBuilder();
	  pb.command().add("run-driver");
	  pb.redirectError(ProcessBuilder.Redirect.INHERIT);
	  Process p = pb.start();
	  
	  // *******************************************************
	  // PipeTransport Hold住操作系统进程,Playwright的底层实际是通过:进程通信来传递消息来着的
	  // 可以这样理解,底层会通过:Connection与Driver进程通信.
	  // PipeTransport与Connection内容放到最后进行剖析.
	  // *******************************************************
	  Connection connection = new Connection(new PipeTransport(p.getInputStream(), p.getOutputStream()), env);
  	  PlaywrightImpl result = connection.initializePlaywright();
	  result.driverProcess = p;
	  result.initSharedSelectors(null);
	  return result;
	} catch (IOException e) {
	  throw new PlaywrightException("Failed to launch driver", e);
	}
}
```
### (4). 总结
从分析的结果来看:  
+ PlaywrightImpl会委托给Driver下载浏览器驱动. 
+ 运行Driver(playwright.sh run-driver),通过:Process Hold住这个后台进程.     
+ 通过PipeTransport包裹着进程(Process). 
+ 通过Connection包裹PipeTransport(可以理解:Connection可能是一个更高级的API).  