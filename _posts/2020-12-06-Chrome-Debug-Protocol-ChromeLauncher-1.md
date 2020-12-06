---
layout: post
title: 'Chrome Debug Protocol--ChromeLauncher(一)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). LogRequestsExample
```
// ... ...
// ********************************************************
// 2. 创建:ChromeLauncher
// ********************************************************
final ChromeLauncher launcher = new ChromeLauncher();
// ... ...

```
### (2). ChromeLauncher构造器
```
public ChromeLauncher() {
  // ********************************************************
  // 3. 创建:ChromeLauncherConfiguration
  // ********************************************************

  // ********************************************************
  // 4. 调用构造器
  // ********************************************************
  this(new ChromeLauncherConfiguration());
}
```
### (3). ChromeLauncherConfiguration
> Chrome默认的一些配置(启动等待时间/关闭等待时间...)

```
public class ChromeLauncherConfiguration {
  /** Default startup wait time in seconds. */
  private static final int DEFAULT_STARTUP_WAIT_TIME = 60;

  /** Default shutdown wait time in seconds. */
  private static final int DEFAULT_SHUTDOWN_WAIT_TIME = 60;

  /** 5 seconds wait time for threads to stop. */
  private static final int THREAD_JOIN_WAIT_TIME = 5;

  /** Startup wait time in seconds. */
  private int startupWaitTime = DEFAULT_STARTUP_WAIT_TIME;

  /** Shutdown wait time in seconds. */
  private int shutdownWaitTime = DEFAULT_SHUTDOWN_WAIT_TIME;

  /** Waits for threads to quite in seconds. */
  private int threadWaitTime = THREAD_JOIN_WAIT_TIME;
}
```
### (4). ChromeLauncher构造器
```
public ChromeLauncher(ChromeLauncherConfiguration configuration) {
  // 5. 
    this(
        //  创建LauncherImpl
        new ProcessLauncherImpl(),
        // 获得系统环境
        System::getenv,
        // 创建为空的关闭回调函数
        new RuntimeShutdownHookRegistry(),
        configuration);
  }
```
### (5). ChromeLauncher构造器
```
public ChromeLauncher(
      ProcessLauncher processLauncher,
      Environment environment,
      ShutdownHookRegistry shutdownHookRegistry,
      ChromeLauncherConfiguration configuration) {
  
  this.processLauncher = processLauncher;
  this.environment = environment;
  this.shutdownHookRegistry = shutdownHookRegistry;
  this.configuration = configuration;
}
```
### (6). UML类图
![](/assets/chrome/imgs/ChromeLauncher-class.jpg)

### (7). 总结
> 创建:ChromeLauncher时,创建:ChromeLauncherConfiguration和ProcessLauncher对象