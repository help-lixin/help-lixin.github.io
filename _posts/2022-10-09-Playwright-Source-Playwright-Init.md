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
	  // 确保Driver安装
	  // *******************************************************
	  Driver.ensureDriverInstalled(env, true);
	try {
	  // 通过:java.lang.Process来管理浏览器与浏览器通信.
	  // playwright.sh 	run-driver
	  ProcessBuilder pb = driver.createProcessBuilder();
	  pb.command().add("run-driver");
	  pb.redirectError(ProcessBuilder.Redirect.INHERIT);
	  Process p = pb.start();
	  
	  
	  // ... ...
	} catch (IOException e) {
	  throw new PlaywrightException("Failed to launch driver", e);
	}
}
```
### (4). 总结
从现在分析的结果来看,PlaywrightImpl好像也没干啥,那是因为:PlaywrightImpl把代码委托给了:Driver.  