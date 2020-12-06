---
layout: post
title: 'Chrome Debug Protocol--ChromeService.launch(二)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). LogRequestsExample
```
// ... ...
final ChromeLauncher launcher = new ChromeLauncher();
// ********************************************************
// 2. 调用launch,创建:ChromeService
// ********************************************************
final ChromeService chromeService = launcher.launch(false);
// ... ...

```
### (2). ChromeLauncher.launch
```
public ChromeService launch(boolean headless) throws ChromeProcessException {
  // 是否以无头方式创建:ChromeService
  // headless = false;会显示Chrome
  
  // ******************************************************************
  // 先获取chrome.exe所在位置,然后:调用内部私有方法:launch
  // ******************************************************************
  return launch(
    // ****************************************************************
    // 3. 获取chrome安装目录
    // ****************************************************************
    getChromeBinaryPath(), 
    // 创建ChromeArguments(运用早Build模式)
    ChromeArguments.defaults(headless).build()
  );
}


public ChromeService launch(Path chromeBinaryPath, ChromeArguments chromeArguments)
      throws ChromeProcessException {
  // **********************************************************
  // 4. ChromeLauncher.launchChromeProcess
  // **********************************************************
  int port = launchChromeProcess(chromeBinaryPath, chromeArguments);

  // **********************************************************
  // 6. 创建:ChromeServiceImpl(创建WebSocket连接,并连接到指定端口)
  // **********************************************************
  return new ChromeServiceImpl(port);
}
```
### (3). ChromeLauncher.getChromeBinaryPath

```
// 定义默认的:Chrome文件位置
private static final String[] CHROME_BINARIES = new String[] {
  "/usr/bin/chromium",
  "/usr/bin/chromium-browser",
  "/usr/bin/google-chrome-stable",
  "/usr/bin/google-chrome",
  "/snap/bin/chromium",
  "/Applications/Chromium.app/Contents/MacOS/Chromium",
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
  "/Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary",
  "C:/Program Files (x86)/Google/Chrome/Application/chrome.exe"
};

public Path getChromeBinaryPath() {
  // 1.从环境变量中获得:CHROME_PATH
  String envChrome = environment.getEnv(ENV_CHROME_PATH);
  if (envChrome != null) { // false
    boolean isExecutable = processLauncher.isExecutable(envChrome);

    if (isExecutable) { // 判断这个文件是否可执行
      return Paths.get(envChrome).toAbsolutePath();
    }

    throw new RuntimeException("CHROME_PATH environment value is not an executable file.");
  }


   // 遍历二进制目录
  for (String binary : CHROME_BINARIES) {
    // 判断文件是否能运行
    boolean isExecutable = processLauncher.isExecutable(binary);

    if (isExecutable) {
      // 能运行的情况下:返回绝对路径
      return Paths.get(binary).toAbsolutePath();
    }
  }

  throw new RuntimeException(
      "Could not find chrome binary! Try setting CHROME_PATH env to chrome binary path.");
}//end getChromeBinaryPath
```
### (4). ChromeLauncher.launchChromeProcess

// 定义关闭钩子线程处理
private Thread shutdownHookThread = new Thread(this::close);

```
private int launchChromeProcess(Path chromeBinary, ChromeArguments chromeArguments)
      throws ChromeProcessException {
   // chromeBinary = "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
   // chromeArguments = com.github.kklisura.cdt.launch.ChromeArguments

  // 判断chrome是否已经运行了
  if (isAlive()) { // false
    throw new IllegalStateException("Chrome process has already been started started.");
  }

  // 注册关闭钩子回调线程
  shutdownHookRegistry.register(shutdownHookThread);

  // 把:ChromeArguments所有参数转换成Map
  // {no-first-run=true, remote-debugging-port=0, disable-client-side-phishing-detection=true, disable-popup-blocking=true, disable-default-apps=true, disable-extensions=true, metrics-recording-only=true, no-default-browser-check=true, disable-background-timer-throttling=true, disable-translate=true, safebrowsing-disable-auto-update=true, disable-background-networking=true, disable-prompt-on-repost=true, disable-hang-monitor=true, disable-sync=true}
  Map<String, Object> argumentsMap = getArguments(chromeArguments);


  // Special case for user data directory.
  // 没有为Chrome指定临时目录的情况下,则创建临时文件,指定临时目录
  if (chromeArguments.getUserDataDir() == null) { // true
    // TEMP_PREFIX = cdt-user-data-dir
    // userDatDir= /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/cdt-user-data-dir7226844001987293869
    String userDatDir = randomTempDir(TEMP_PREFIX);
    
    userDataDirPath = Paths.get(userDatDir);

    // 把临时目录设以map中
    argumentsMap.put(ChromeArguments.USER_DATA_DIR_ARGUMENT, userDatDir);
  }


  // 对map进行转换
  [--no-first-run, --remote-debugging-port=0, --disable-client-side-phishing-detection, --disable-popup-blocking, --disable-default-apps, --disable-extensions, --metrics-recording-only, --no-default-browser-check, --disable-background-timer-throttling, --disable-translate, --safebrowsing-disable-auto-update, --disable-background-networking, --disable-prompt-on-repost, --user-data-dir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/cdt-user-data-dir7226844001987293869, --disable-hang-monitor, --disable-sync]
  List<String> arguments = argsMapToArgsList(argumentsMap);

  LOGGER.info(
      "Launching chrome process {} with arguments {}", chromeBinary.toString(), argumentsMap);

  try {
    // *******************************************************************
    // 留到下一节再分析
    // *******************************************************************
    chromeProcess = processLauncher.launch(chromeBinary.toString(), arguments);

    // *****************************************************************************
    // 5. ChromeLauncher.waitForDevToolsServer
    // *****************************************************************************
    return waitForDevToolsServer(chromeProcess);
  } catch (IOException e) {
    // Unsubscribe from registry on exceptions.
    shutdownHookRegistry.remove(shutdownHookThread);

    throw new ChromeProcessException("Failed starting chrome process.", e);
  } catch (Exception e) {
    close();
    throw e;
  }
} // end launchChromeProcess
```


### (5). ChromeLauncher.waitForDevToolsServer
```
private int waitForDevToolsServer(final Process process) throws ChromeProcessTimeoutException {
  final AtomicInteger port = new AtomicInteger();
  final AtomicBoolean success = new AtomicBoolean(false);
  final AtomicReference<String> chromeOutput = new AtomicReference<>("");

  // 创建线程读取:Process里的InputStream,获取创建Chrome随机创建的:ws端口
  Thread readLineThread =
      new Thread(
          () -> {
            StringBuilder chromeOutputBuilder = new StringBuilder();
            BufferedReader reader = null;
            try {
              reader = new BufferedReader(new InputStreamReader(process.getInputStream()));

            
              String line;
              while ((line = reader.readLine()) != null) {
                // 查看chrome是否有返回监听的端口信息
                // "^DevTools listening on ws:\\/\\/.+?:(\\d+)\\/"
                Matcher matcher = DEVTOOLS_LISTENING_LINE_PATTERN.matcher(line);
                if (matcher.find()) {
                  // 设置端口以及成功,并跳出循环
                  port.set(Integer.parseInt(matcher.group(1)));
                  success.set(true);
                  break;
                }

                if (chromeOutputBuilder.length() != 0) {
                  chromeOutputBuilder.append(System.lineSeparator());
                }
                chromeOutputBuilder.append(line);
                chromeOutput.set(chromeOutputBuilder.toString());
              }
            } catch (Exception e) {
              LOGGER.error("Failed while waiting for dev tools server.", e);
            } finally {
              closeQuietly(reader);
            }
          });// end start

  readLineThread.start();

  try {
    // 当前线程(main)等待:readLineThread 60秒
    readLineThread.join(TimeUnit.SECONDS.toMillis(configuration.getStartupWaitTime()));

    if (!success.get()) {  // 不成功的情况下做什么事情,
      close(readLineThread);
      throw new ChromeProcessTimeoutException(
          "Failed while waiting for chrome to start: "
              + "Timeout expired! Chrome output: "
              + chromeOutput.get());
    }
  } catch (InterruptedException e) {
    close(readLineThread);

    LOGGER.error("Interrupted while waiting for dev tools server.", e);
    throw new RuntimeException("Interrupted while waiting for dev tools server.", e);
  }
  // 返回端口信息
  return port.get();
} //end waitForDevToolsServer
```

### (6). ChromeServiceImpl构造器(WebSocket连接)
```
public ChromeServiceImpl(int port) {
  // port = 54614
  // *****************************************************************
  // 1. 委托给另一个构造器
  // *****************************************************************
  this(LOCALHOST, port);
}

public ChromeServiceImpl(String host, int port) {
  // host = locahost
  // port = 54614
  // *****************************************************************
  // 2. 创建websocket请求,并委给另一个构造器
  // *****************************************************************
  this(
        host, 
        port, 
        // 创建WebSodket
        (wsUrl -> WebSocketServiceImpl.create(URI.create(wsUrl)))
  );
}

// *****************************************************************
// 3. 最后的构造器
// *****************************************************************
public ChromeServiceImpl(String host, int port, WebSocketServiceFactory webSocketServiceFactory) {
  this.host = host;
  this.port = port;
  // (wsUrl -> WebSocketServiceImpl.create(URI.create(wsUrl)))
  this.webSocketServiceFactory = webSocketServiceFactory;
}

```
### (7). 总结
> 1. 创建默认的参数(ChromeArguments).    
> 2. 获取chrome.exe所在的位置("/Applications/Google Chrome.app/Contents/MacOS/Google Chrome").      
> 3. **委托给:ProcessLauncherImpl创建Chrome进程,并读取Chrome创建进程时监听的WS端口信息**.    
> 4. 创建:ChromeServiceImpl,该类实际为:**WebSocket的连接器.**   
> 5. <font color='red'>假如chrome的创建在远程,并把端口暴露信息给ZK,可从远程ZK在获得host和ip,然后创建:ChromeServiceImpl,即可.</font>