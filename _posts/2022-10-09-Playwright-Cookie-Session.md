---
layout: post
title: 'Playwright Cookie和Session管理(七)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
当使用Playwright进行自动化测试时,遇到网页需要人工交付这种功能(比如:登录需要验证码或者方块滑动验证),一般的实现方式有几种: 
+ 让开发人员为人工交付留后门(这样做有点鸡肋).  
+ 通过打码平台进行智能识别,需要人工智能知识,代价是否有点高. 
+ 能否把人工交付功能交给用户,等待用户人工交付后,存储好Cookie/Session,以备后用?  

### (2). 功能介绍
以京东为例:
+ 打开浏览器,引导用户在京东登录,当用户登录之后,存储好Cookie和Session.
+ 打开新的浏览器,配置Cookie和Session的位置,引导用户到京东页面,查看是否具备查看订单权限.  

### (3). Cookie和Session存储
```
package org.example;

import com.microsoft.playwright.*;
import org.apache.commons.io.FileUtils;

import java.io.*;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.concurrent.TimeUnit;

public class StoreCookieAndSessionTest {
  public static void main(String[] args) throws Exception {
    try (Playwright playwright = Playwright.create()) {
      Browser browser = playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(false));
      BrowserContext browserContext = browser.newContext();
      Page page = browserContext.newPage();
      // 1. 导航到京东
      page.navigate("https://www.jd.com/");

      // 2. 休息80秒(等待用户输入账号/密码/验证码...登录)
      TimeUnit.SECONDS.sleep(80);
      // 3. 登录之后,存储Cookie到本地磁盘
      browserContext.storageState(new BrowserContext.StorageStateOptions().setPath(Paths.get("/Users/lixin/GitRepository/playwright-java/examples/jd-cookie.json")));

      // 4. 通过JS语法,把所有的session转换成为JSON
      String sessionStorage = (String) page.evaluate("JSON.stringify(sessionStorage)");
      // 5. 存储Session信息到本地磁盘
      Path session = Paths.get("/Users/lixin/GitRepository/playwright-java/examples/jd-session.txt");
      FileUtils.writeStringToFile(session.toFile(), sessionStorage);
      // 6. 在这种情况下,手上执有了:Cookie和Session了.
    }
  }
}
```
### (4). 读取Session和Cookie
```
package org.example;

import com.microsoft.playwright.*;
import org.apache.commons.io.FileUtils;
import org.apache.commons.io.IOUtils;

import java.io.File;
import java.io.FileOutputStream;
import java.io.PrintWriter;
import java.io.StringReader;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.concurrent.TimeUnit;

public class JDAutoLoginTest {
  public static void main(String[] args) throws Exception {
    try (Playwright playwright = Playwright.create()) {
      // 1. 读取磁盘上的Session
      Path sessionPath = Paths.get("/Users/lixin/GitRepository/playwright-java/examples/jd-session.txt");
      String sessionStorage = FileUtils.readFileToString(sessionPath.toFile(), "UTF-8");

      Browser browser = playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(false));
      // 2. 指定Cookie存储位置
      BrowserContext browserContext = browser.newContext(new Browser.NewContextOptions().setStorageStatePath(Paths.get("/Users/lixin/GitRepository/playwright-java/examples/jd-cookie.json")));
      // 3. 通过JS设置session信息
      browserContext.addInitScript("(storage => {\n" +
        "  if (window.location.hostname === 'jd.com') {\n" +
        "    const entries = JSON.parse(storage);\n" +
        "     for (const [key, value] of Object.entries(entries)) {\n" +
        "      window.sessionStorage.setItem(key, value);\n" +
        "    };\n" +
        "  }\n" +
        "})('" + sessionStorage + "')");

      Page page = browserContext.newPage();
      page.navigate("https://www.jd.com/");
      TimeUnit.HOURS.sleep(1);
    }
  }
}
```
### (5). 结果验证
!["京东自动登录功能"](/assets/playwright/imgs/playwright-cookie-session.png)
### (6). 总结
通过这个案例,可以证实一点,第一次需要用户介入登录,然后,复制会话信息,赋值到新的浏览器之后,登录的状态依然是有效的.  