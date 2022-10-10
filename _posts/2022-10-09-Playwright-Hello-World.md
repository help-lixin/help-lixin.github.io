---
layout: post
title: 'Playwright HelloWorld入门(一)' 
date: 2022-10-09
author: 李新
tags:  Playwright
---


### (1). Playwright是什么
Playwright是由Microsoft开源的一个Web测试和自动化框架,它支持:Chromium/Firefox/WebKit,Playwright旨在实现跨浏览器的web自动化,并支持:Python/.Net/Java等语言. 
### (2). pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>org.example</groupId>
  <artifactId>examples</artifactId>
  <version>0.1-SNAPSHOT</version>
  <name>Playwright Client Examples</name>
  
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  
  <dependencies>
    <dependency>
      <groupId>com.microsoft.playwright</groupId>
      <artifactId>playwright</artifactId>
      <version>1.28.0-SNAPSHOT</version>
    </dependency>
  </dependencies>
  
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.1</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```
### (3). PageScreenshot
```
package org.example;

import com.microsoft.playwright.*;

import java.nio.file.Paths;
import java.util.Arrays;
import java.util.List;

public class PageScreenshot {
  public static void main(String[] args) {
	//  1. 创建:Playwright
	//  1.1 解压:jar:file:/Users/lixin/.m2/repository/com/microsoft/playwright/driver-bundle/1.21.0/driver-bundle-1.21.0.jar!/driver/mac
	//  1.2 执行:playwright.sh install安装
	//  1.3 在mac下最后安装的driver在该目录下:
	//  1.4 通过java.lang.Process Hold住浏览器(比如:chromium)(playwright.sh run-driver) 
	//  1.5 Playwright的底层实际上是通过Chrome CDP通信来着的.
	// lixin-macbook:~ lixin$ ll /Users/lixin/Library/Caches/ms-playwright/
	// drwxr-xr-x    5 lixin  staff   160 10 10 12:24 .links/
	// drwxr-xr-x    4 lixin  staff   128 10  9 23:54 chromium-1024/
	// drwxr-xr-x    4 lixin  staff   128 10 10 00:39 chromium-1028/
	// drwxr-xr-x    5 lixin  staff   160 10  9 23:54 ffmpeg-1007/
	// drwxr-xr-x    4 lixin  staff   128 10  9 23:58 firefox-1350/
	// drwxr-xr-x    4 lixin  staff   128 10 10 00:42 firefox-1357/
	// drwxr-xr-x   16 lixin  staff   512 10 10 00:00 webkit-1715/
	// drwxr-xr-x   16 lixin  staff   512 10 10 00:46 webkit-1724/
	
    try (Playwright playwright = Playwright.create()) {
	  
      // 2. 配置运行哪些浏览器
      List<BrowserType> browserTypes = Arrays.asList(
        playwright.chromium(),
        playwright.webkit(),
        playwright.firefox()
      );

      // 3. 为浏览器配置可选项,为有头模式.
      BrowserType.LaunchOptions launchOptions = new BrowserType.LaunchOptions().setHeadless(false);
	  // 4. 挨个控制浏览器跳转到网址:http://whatsmyuseragent.org
      for (BrowserType browserType : browserTypes) {
        try (Browser browser = browserType.launch(launchOptions)) {
          BrowserContext context = browser.newContext();
          Page page = context.newPage();
          page.navigate("http://whatsmyuseragent.org/");
		  //  5. 截图
          page.screenshot(new Page.ScreenshotOptions().setPath(Paths.get("screenshot-" + browserType.name() + ".png")));
        }
      }
    }
  }
}
```
### (4). 总结
