---
layout: post
title: 'Playwright Textarea标签学习(五)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述

### (2). html
```
<!DOCTYPE html>
<html>
<head>
  <title>Test</title>
  <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
  <script src="./jquery.min.js"></script>
</head>
<body>
	<label>Textarea Test </label>
	<textarea>hello world</textarea>
	</br>
</body>
```
### (3). TextareaTest
```
package org.example;

import com.microsoft.playwright.*;
import com.microsoft.playwright.options.AriaRole;
import com.microsoft.playwright.options.WaitUntilState;

public class TextareaTest {
  public static void main(String[] args) throws Exception {
    try (Playwright playwright = Playwright.create()) {
      Browser browser = playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(false));
      BrowserContext browserContext = browser.newContext();
      Page page = browserContext.newPage();
      page.navigate("file:///Users/lixin/GitRepository/playwright-java/examples/src/main/resources/index.html", new Page.NavigateOptions().setWaitUntil(WaitUntilState.LOAD));

      // 通过querySelector定位元素
      ElementHandle textarea = page.querySelector("textarea");
      textarea.evaluate("textarea => textarea.value = 'Hello World!!!!'");
      textarea.selectText();

      Object evaluate = textarea.evaluate("textarea => textarea.value");
      System.out.println(evaluate);
    }
  }
}
```
### (4). 输出结果
```
Hello World!!!!
```
### (5). 总结
在这一篇通过:querySelector定位元素,并设置元素的内容,以及获取最新的内容.