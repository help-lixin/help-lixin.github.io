---
layout: post
title: 'Selenium原理'
date: 2020-12-08
author: 李新
tags: Selenium
---

### (1). 查看当前Chrome版本
!["Chrome Version"](/assets/chrome/imgs/chrome-version.jpg)

### (2). 下载与Chrome对应的ChromeDriver
> Chrome与ChromeDriver版本一定要对应上.    
> https://npm.taobao.org/mirrors/chromedriver/87.0.4280.88/   


### (3). pom.xml配置
```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin.selenium</groupId>
	<artifactId>selenium-demo</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>selenium-demo</name>
	<url>http://maven.apache.org</url>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.seleniumhq.selenium</groupId>
			<artifactId>selenium-java</artifactId>
			<version>3.8.1</version>
		</dependency>
		<dependency>
			<groupId>commons-io</groupId>
			<artifactId>commons-io</artifactId>
			<version>2.4</version>
		</dependency>
	</dependencies>
</project>

```

### (4). App.java
```
package help.lixin.selenium;

import java.io.File;
import java.util.concurrent.TimeUnit;

import org.apache.commons.io.FileUtils;
import org.openqa.selenium.By;
import org.openqa.selenium.Keys;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

public class App {
	public static void main(String[] args) throws Exception {
		String path = "/Users/lixin/Downloads/chromedriver";
		System.setProperty("webdriver.chrome.driver", path);

		WebDriver driver = new ChromeDriver();
		driver.manage().timeouts().implicitlyWait(10, TimeUnit.MINUTES);
		// 窗口最大化
		driver.manage().window().maximize();
		driver.get("https://www.baidu.com");

		WebElement element = driver.findElement(By.xpath("//*[@id='kw']"));
		element.sendKeys("Iphone");

		// 发送回车
		element.sendKeys(Keys.ENTER);
		// WebElement submit = driver.findElement(By.id("su"));
		// submit.submit();

		// 期待某个元素出现之后,业务代码才继续往下走
		new WebDriverWait(driver, 20, 5).until(ExpectedConditions.presenceOfElementLocated(By.id("content_left")));

		// 保存页面为图片
		File scrFile = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
		FileUtils.copyFile(scrFile, new File("/Users/lixin/Workspace/selenium-demo/target/a.png"));
		driver.quit();
	}
}
```
### (5). ChromeDriver是什么?
>  ChromeDriver是WebDriver的实现.WebDriver是W3C提出来操作浏览器的规范,由各大厂商进行实现.   
>  https://www.w3.org/TR/webdriver/

### (6). Selenium内部原理是什么?
> 1. W3C提供WebDriver(HTTP)规范.   
> 2. 各大浏览器(firefox/chrome...)对WebDriver做实现.
> 3. WebDriver(ChromDriver)负责提供API接口,接受请求,并分派请求给浏览器(Chrome Debug Protocol)进行交互.   

### (7). 如何证实ChromDriver与Chrome Debug Protocol进行了交互?
> 查看进程信息.
> 1. 运行上面的App.java,断点在:driver.quit方法(防止程序退出).   
> 2. 查看(chromedriver)进程信息,<font color='red'>查看是否开启了端口</font>.   
> 3. 查看(chrome)进程信息,<font color='red'>查看是否开启了远程端口</font>.   

```
// chromedriver负责与selenium进行交互.开启了:48760端口
lixin-macbook:Downloads lixin$ ps -ef|grep chromedriver
  501  1762  1760   0  2:12下午 ??         0:00.18 /Users/lixin/Downloads/chromedriver --port=48760
```

```
// 能检测到,chrome开启了多个:--remote-debugging-port
// --remote-debugging-port=0;代表端口由Chrome随机开启. 
lixin-macbook:~ lixin$ ps -ef|grep google

  501  1763  1762   0  2:12下午 ??         0:03.42 /Applications/Google Chrome.app/Contents/MacOS/Google Chrome --disable-background-networking --disable-client-side-phishing-detection --disable-default-apps --disable-hang-monitor --disable-popup-blocking --disable-prompt-on-repost --disable-sync --enable-automation --enable-blink-features=ShadowDOMV0 --enable-logging --log-level=0 --no-first-run --no-service-autorun --password-store=basic --remote-debugging-port=0 --test-type=webdriver --use-mock-keychain --user-data-dir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/.com.google.Chrome.S9VaV6 data:,


  501  1765     1   0  2:12下午 ??         0:00.02 /Applications/Google Chrome.app/Contents/Frameworks/Google Chrome Framework.framework/Versions/87.0.4280.88/Helpers/chrome_crashpad_handler --monitor-self-annotation=ptype=crashpad-handler --database=/Users/lixin/Library/Application Support/Google/Chrome/Crashpad --metrics-dir=/Users/lixin/Library/Application Support/Google/Chrome --url=https://clients2.google.com/cr/report --annotation=channel= --annotation=plat=OS X --annotation=prod=Chrome_Mac --annotation=ver=87.0.4280.88 --handshake-fd=6
  501  1773  1763   0  2:12下午 ??         0:01.15 /Applications/Google Chrome.app/Contents/Frameworks/Google Chrome Framework.framework/Versions/87.0.4280.88/Helpers/Google Chrome Helper (GPU).app/Contents/MacOS/Google Chrome Helper (GPU) --type=gpu-process --field-trial-handle=1718379636,4955986443181843131,4522484406338959736,131072 --enable-logging --log-level=0 --user-data-dir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/.com.google.Chrome.S9VaV6 --gpu-preferences=MAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAABgAAAAAAAQAAAAAAAAAAAAAAAAAAAA6AAAABwAAADgAAAAAAAAAOgAAAAAAAAA8AAAAAAAAAD4AAAAAAAAAAABAAAAAAAACAEAAAAAAAAQAQAAAAAAABgBAAAAAAAAIAEAAAAAAAAoAQAAAAAAADABAAAAAAAAOAEAAAAAAABAAQAAAAAAAEgBAAAAAAAAUAEAAAAAAABYAQAAAAAAAGABAAAAAAAAaAEAAAAAAABwAQAAAAAAAHgBAAAAAAAAgAEAAAAAAACIAQAAAAAAAJABAAAAAAAAmAEAAAAAAACgAQAAAAAAAKgBAAAAAAAAsAEAAAAAAAC4AQAAAAAAABAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAGAAAAEAAAAAAAAAAAAAAABwAAABAAAAAAAAAAAAAAAAgAAAAQAAAAAAAAAAAAAAAKAAAAEAAAAAAAAAAAAAAACwAAABAAAAAAAAAAAAAAAA0AAAAQAAAAAAAAAAEAAAAAAAAAEAAAAAAAAAABAAAABgAAABAAAAAAAAAAAQAAAAcAAAAQAAAAAAAAAAEAAAAIAAAAEAAAAAAAAAABAAAACgAAABAAAAAAAAAAAQAAAAsAAAAQAAAAAAAAAAEAAAANAAAAEAAAAAAAAAAEAAAAAAAAABAAAAAAAAAABAAAAAYAAAAQAAAAAAAAAAQAAAAHAAAAEAAAAAAAAAAEAAAACAAAABAAAAAAAAAABAAAAAoAAAAQAAAAAAAAAAQAAAALAAAAEAAAAAAAAAAEAAAADQAAABAAAAAAAAAABgAAAAAAAAAQAAAAAAAAAAYAAAAGAAAAEAAAAAAAAAAGAAAABwAAABAAAAAAAAAABgAAAAgAAAAQAAAAAAAAAAYAAAAKAAAAEAAAAAAAAAAGAAAACwAAABAAAAAAAAAABgAAAA0AAAA= --enable-logging --log-level=0 --shared-files
  
  501  1774  1763   0  2:12下午 ??         0:01.30 /Applications/Google Chrome.app/Contents/Frameworks/Google Chrome Framework.framework/Versions/87.0.4280.88/Helpers/Google Chrome Helper.app/Contents/MacOS/Google Chrome Helper --type=utility --utility-sub-type=network.mojom.NetworkService --field-trial-handle=1718379636,4955986443181843131,4522484406338959736,131072 --lang=zh-CN --service-sandbox-type=network --use-mock-keychain --enable-logging --log-level=0 --user-data-dir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/.com.google.Chrome.S9VaV6 --enable-logging --log-level=0 --shared-files --seatbelt-client=38
  
  501  1777  1763   0  2:12下午 ??         0:02.15 /Applications/Google Chrome.app/Contents/Frameworks/Google Chrome Framework.framework/Versions/87.0.4280.88/Helpers/Google Chrome Helper (Renderer).app/Contents/MacOS/Google Chrome Helper (Renderer) --type=renderer --enable-automation --enable-logging --log-level=0 --remote-debugging-port=0 --test-type=webdriver --field-trial-handle=1718379636,4955986443181843131,4522484406338959736,131072 --enable-blink-features=ShadowDOMV0 --lang=zh-CN --user-data-dir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/.com.google.Chrome.S9VaV6 --disable-client-side-phishing-detection --num-raster-threads=2 --enable-zero-copy --enable-gpu-memory-buffer-compositor-resources --enable-main-frame-before-activation --renderer-client-id=5 --no-v8-untrusted-code-mitigations --shared-files --seatbelt-client=66
  
  501  1787  1763   0  2:12下午 ??         0:00.11 /Applications/Google Chrome.app/Contents/Frameworks/Google Chrome Framework.framework/Versions/87.0.4280.88/Helpers/Google Chrome Helper (Renderer).app/Contents/MacOS/Google Chrome Helper (Renderer) --type=renderer --enable-automation --enable-logging --log-level=0 --remote-debugging-port=0 --test-type=webdriver --field-trial-handle=1718379636,4955986443181843131,4522484406338959736,131072 --enable-blink-features=ShadowDOMV0 --lang=zh-CN --user-data-dir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/.com.google.Chrome.S9VaV6 --disable-client-side-phishing-detection --num-raster-threads=2 --enable-zero-copy --enable-gpu-memory-buffer-compositor-resources --enable-main-frame-before-activation --renderer-client-id=6 --no-v8-untrusted-code-mitigations --shared-files --seatbelt-client=120
```

```
// 查看chromedriver开启了哪些端口
lixin-macbook:~ lixin$  lsof -nP  | grep TCP | grep LISTEN|grep chromedri
chromedri 1762 lixin    6u     IPv4 0x73ddde930e621453        0t0                 TCP 127.0.0.1:48760 (LISTEN)
chromedri 1762 lixin    7u     IPv6 0x73ddde9300493283        0t0                 TCP [::1]:48760 (LISTEN)

// 查看chrome(Google),开启了哪些端口
lixin-macbook:~ lixin$  lsof -nP  | grep TCP | grep LISTEN|grep Google
Google    1763 lixin   56u     IPv4 0x73ddde930e57ea73        0t0                 TCP 127.0.0.1:50605 (LISTEN)
```

### (8). 求证
> 访问Chrome(Google)监听的端口(50605),求证是否为:Chrome Debug Protocol协议.

```
// 访问(http://127.0.0.1:50605/json)
// 证实:WebDriver是负责与Chrom Debug Protocol进行交互.
lixin-macbook:~ lixin$ curl http://127.0.0.1:50605/json
[ {
   "description": "",
   "devtoolsFrontendUrl": "/devtools/inspector.html?ws=127.0.0.1:50605/devtools/page/C78EE204F678EDA3F5AF7417C0832A0B",
   "faviconUrl": "https://www.baidu.com/favicon.ico",
   "id": "C78EE204F678EDA3F5AF7417C0832A0B",
   "title": "Iphone_百度搜索",
   "type": "page",
   "url": "https://www.baidu.com/s?ie=utf-8&f=8&rsv_bp=1&rsv_idx=1&tn=baidu&wd=Iphone&fenlei=256&rsv_pq=9a2803040000299c&rsv_t=4d52ZoPgFHRxXbU%2Fp7Ho6D3CwiywFL9J7opY29ff9QoS1ffU8uyAMwj8SI8&rqlang=cn&rsv_enter=1&rsv_dl=tb&rsv_sug3=6&rsv_sug2=0&rsv_btype=i&inputT=135&rsv_sug4=136",
   "webSocketDebuggerUrl": "ws://127.0.0.1:50605/devtools/page/C78EE204F678EDA3F5AF7417C0832A0B"
} ]
```

### (9). 结论
> 结果,证实:WebDriver是负责与Chrom Debug Protocol进行交互.

!["Selenium 原理图"](/assets/chrome/imgs/selenium-principle.jpg)
