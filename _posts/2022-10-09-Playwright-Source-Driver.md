---
layout: post
title: 'Playwright源码之Driver(三)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
这一小节,主要剖析:Driver
### (2). Driver.ensureDriverInstalled
```
public abstract class Driver {
  protected final Map<String, String> env = new LinkedHashMap<>();
  // 单例模式
  private static Driver instance;

  // *****************************************************************
  // 单例模式
  // 1. 创建:Driver
  // *****************************************************************
  public static synchronized Driver ensureDriverInstalled(Map<String, String> env, Boolean installBrowsers) {
    if (instance == null) {
      instance = createAndInstall(env, installBrowsers);
    }
    return instance;
  } // end ensureDriverInstalled
  
  
  public static Driver createAndInstall(Map<String, String> env, Boolean installBrowsers) {
	  try {
		// *****************************************************************
		// 1.1 创建Driver
        // *****************************************************************
		Driver instance = newInstance();
		logMessage("initializing driver");
		// *****************************************************************
		// 2. 初始化
		// *****************************************************************
		instance.initialize(env, installBrowsers);
		logMessage("driver initialized.");
		return instance;
	  } catch (Exception exception) {
		throw new RuntimeException("Failed to create driver", exception);
	  }
  } // end createAndInstall
  
  
  private static Driver newInstance() throws Exception {
	  // *****************************************************************
	  // 1.2 读取系统环境变量,看是否有配置Driver,如果有的话,通过:PreinstalledDriver包裹一层
	  // *****************************************************************
      String pathFromProperty = System.getProperty("playwright.cli.dir");
      if (pathFromProperty != null) { // false
        return new PreinstalledDriver(Paths.get(pathFromProperty));
      }
	  
	  // *****************************************************************
	  // 1.3 读取环境变量:playwright.driver.impl
	  // 如果环境变量没有配置,则默认值为:com.microsoft.playwright.impl.driver.jar.DriverJar
	  // 注意:这个类在另一个工程里:driver-bundle,这个工程很重要,呆会会深入讲解.
	  //  通过反射创建:Driver的实现
	  // *****************************************************************
      String driverImpl = System.getProperty("playwright.driver.impl", "com.microsoft.playwright.impl.driver.jar.DriverJar");
      Class<?> jarDriver = Class.forName(driverImpl);
      return (Driver) jarDriver.getDeclaredConstructor().newInstance();
  } // end newInstance
  
  
  // *****************************************************************
  // 2.1 初始化
  // *****************************************************************
  private void initialize(Map<String, String> env, Boolean installBrowsers) throws Exception {
  	this.env.putAll(env);
  	initialize(installBrowsers);
  } // end initialize
  
  // 2.2 initialize方法是一个抽象方法,预留给子类
  protected abstract void initialize(Boolean installBrowsers) throws Exception;
  
}  
```
### (3). DriverJar.initialize
```
protected void initialize(Boolean installBrowsers) throws Exception {
	// ... ...
	// ****************************************************
	// 2.3 提取Driver到临时目录
	// ****************************************************
	extractDriverToTempDir();
	// ... ...
	if (installBrowsers)
	// ****************************************************
	// 3. 安装浏览器
	// ****************************************************
	  installBrowsers(env);
}
```
### (4). DriverJar.extractDriverToTempDir
```
// 提取驱动程序到临时目录
void extractDriverToTempDir() throws URISyntaxException, IOException {
    ClassLoader classloader = Thread.currentThread().getContextClassLoader();
	// **********************************************************
	// 2.3.1 从ClassPath中寻找:driver/mac目录,我断点后,发现在这个JAR包里:driver-bundle-1.27.0.jar
	//       所以,能断定,在加载playwright依赖时,会加载driver-bundle-1.27.0.jar,并且这个包有点大.
	// jar:file:/Users/lixin/.m2/repository/com/microsoft/playwright/driver-bundle/1.21.0/driver-bundle-1.21.0.jar!/driver/mac
	// **********************************************************
    URI originalUri = classloader.getResource("driver/" + platformDir()).toURI();
	
	// **********************************************************
	// 2.3.2 解压JAR包里的内容到临时目录
	// **********************************************************
    URI uri = maybeExtractNestedJar(originalUri);
	// ... ...
}
```
### (5). DriverJar.maybeExtractNestedJar
```
private URI maybeExtractNestedJar(final URI uri) throws URISyntaxException {
    if (!"jar".equals(uri.getScheme())) {
      return uri;
    }
	
    final String JAR_URL_SEPARATOR = "!/";
    String[] parts = uri.toString().split("!/");
    if (parts.length != 3) {
      return uri;
    }
	
    String innerJar = String.join(JAR_URL_SEPARATOR, parts[0], parts[1]);
    URI jarUri = new URI(innerJar);
    try (FileSystem fs = FileSystems.newFileSystem(jarUri, Collections.emptyMap())) {
      Path fromPath = Paths.get(jarUri);
      Path toPath = driverTempDir.resolve(fromPath.getFileName().toString());
	  // ************************************************************
	  // 2.3.3 解压JAR包里的内容到临时目录
	  // ************************************************************
      Files.copy(fromPath, toPath);
      toPath.toFile().deleteOnExit();
      return new URI("jar:" + toPath.toUri() + JAR_URL_SEPARATOR + parts[2]);
    } catch (IOException e) {
      throw new RuntimeException("Failed to extract driver's nested .jar from " + jarUri + "; full uri: " + uri, e);
    }
}
```
### (6). 生成的临时目录内容如下
```
// ******************************************************************
// playwright.sh是重点
// ******************************************************************
lixin-macbook:playwright-java lixin$ tree -L 2
.
├── LICENSE
├── node
├── package
│   ├── README.md
│   ├── ThirdPartyNotices.txt
│   ├── api.json
│   ├── bin
│   ├── browsers.json
│   ├── cli.js
│   ├── index.d.ts
│   ├── index.js
│   ├── index.mjs
│   ├── lib
│   ├── package.json
│   ├── protocol.yml
│   └── types
└── playwright.sh         
```
### (7). DriverJar.installBrowsers
```
private void installBrowsers(Map<String, String> env) throws IOException, InterruptedException {
    // ... ... 
	// /Users/lixin/Desktop/playwright-java/playwright.sh
    Path driver = driverPath();
    if (!Files.exists(driver)) {
      throw new RuntimeException("Failed to find driver: " + driver);
    }
	
	// ******************************************************************
	// 3.1 调用操作系统,运行shell,这个步骤会要很长时间,会去下载所有浏览器驱动.
	// ******************************************************************
	// /Users/lixin/Desktop/playwright-java/playwright.sh install 
    ProcessBuilder pb = createProcessBuilder();
    pb.command().add("install");
    pb.redirectError(ProcessBuilder.Redirect.INHERIT);
    pb.redirectOutput(ProcessBuilder.Redirect.INHERIT);
    Process p = pb.start();
    boolean result = p.waitFor(10, TimeUnit.MINUTES);
    
	// ... ...
}// end 
```
### (8). Mac下安装浏览器后的位置
```
lixin-macbook:playwright-java lixin$ ll /Users/lixin/Library/Caches/ms-playwright
drwxr-xr-x    4 lixin  staff   128 10  9 23:54 chromium-1024/
drwxr-xr-x    4 lixin  staff   128 10 10 00:39 chromium-1028/
drwxr-xr-x    5 lixin  staff   160 10  9 23:54 ffmpeg-1007/
drwxr-xr-x    4 lixin  staff   128 10  9 23:58 firefox-1350/
drwxr-xr-x    4 lixin  staff   128 10 10 00:42 firefox-1357/
drwxr-xr-x   16 lixin  staff   512 10 10 00:00 webkit-1715/
drwxr-xr-x   16 lixin  staff   512 10 10 00:46 webkit-1724/
```
### (9). 总结
Driver的作用无非不过就是下载所有的Driver,并安装到系统目录下:/Users/lixin/Library/Caches/ms-playwright.   