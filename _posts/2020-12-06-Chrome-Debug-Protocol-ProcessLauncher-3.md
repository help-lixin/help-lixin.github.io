---
layout: post
title: 'Chrome Debug Protocol--ProcessLauncherImpl(三)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). 概述
> 上一节分析到了:ChromeLauncher.launchChromeProcess为会将请求委托给:ProcessLauncherImpl进行处理,在这里我开始剖析:ProcessLauncherImpl.launch方法

### (2). ChromeLauncher.launchChromeProcess

```
private int launchChromeProcess(Path chromeBinary, ChromeArguments chromeArguments)
      throws ChromeProcessException {
   // chromeBinary = "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
   // chromeArguments = com.github.kklisura.cdt.launch.ChromeArguments

  // .... .... 
  // *********************************************************************
  // 3.委托给:ProcessLauncherImpl.launch
  // *********************************************************************
  chromeProcess = processLauncher.launch(chromeBinary.toString(), arguments);
  // .... .... 
} // end launchChromeProcess
```
### (3). ProcessLauncherImpl.launch
> 注意:Process为JDK1.5开始新添加的类,该类用于创建操作系统进程.  
> Process参考地址: https://blog.csdn.net/shadow_zed/article/details/93545843   

```
public Process launch(String program, List<String> args) throws IOException {
  // program = "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
  // args = [--no-first-run, --remote-debugging-port=0, --disable-client-side-phishing-detection, --disable-popup-blocking, --disable-default-apps, --disable-extensions, --metrics-recording-only, --no-default-browser-check, --disable-background-timer-throttling, --disable-translate, --safebrowsing-disable-auto-update, --disable-background-networking, --disable-prompt-on-repost, --user-data-dir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/cdt-user-data-dir7226844001987293869, --disable-hang-monitor, --disable-sync] 
  List<String> arguments = new ArrayList<>();
  arguments.add(program);
  arguments.addAll(args);

  // 创建:ProcessBuilder(Builder模式)
  ProcessBuilder processBuilder = new ProcessBuilder()
          .command(arguments)
          //将标准输入流和错误输入流合并，通过标准输入流读取信息
          .redirectErrorStream(true)
          .redirectOutput(Redirect.PIPE);
  return processBuilder.start();
}
```
### (4). ProcessLauncherImpl.launch原理
> ProcessLauncherImpl内部实际是:调用chrome.exe创建一个系统(chrome)进程. 

```
501  4787  4772   0  3:17下午 ??         0:01.07 /Applications/Google Chrome.app/Contents/Frameworks/Google Chrome Framework.framework/Versions/87.0.4280.88/Helpers/Google Chrome Helper.app/Contents/MacOS/Google Chrome Helper --type=utility --utility-sub-type=network.mojom.NetworkService --field-trial-handle=1718379636,7411747236790916307,2697264592481909618,131072 --lang=zh-CN --service-sandbox-type=network --user-data-dir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/cdt-user-data-dir7226844001987293869 --shared-files --seatbelt-client=33
```

> 查看监听的WS端口   

```
lixin-macbook:chrome-devtools-java-client lixin$ lsof -i tcp:54614
COMMAND    PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
Google    4772 lixin   57u  IPv4 0x35e21e1a9ba06c61      0t0  TCP localhost:54614 (LISTEN)
```
### (5). 总结
> ProcessLauncherImpl会根据传入的命令和参数,创建:Chrome进程.