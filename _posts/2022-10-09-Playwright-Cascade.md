---
layout: post
title: 'Playwright级联选择器学习(四)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
在这一小篇开始,对每一个HTML元素,通过Playwright进行定位.
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
	<label>UL/LI Test </label>
	<ul class="ulClass liClass">
	  <li><a>1111</a></li>
	  <li><a>2222</a></li>
	  <li><a>3333</a></li>
	  <li><a>4444</a></li>
	</ul>
	
	<script>
	$(function(){
	   // 动态追加一个元素
	  $(".liClass").append('<li><a>55555</a></li>');
	
	});
	</script>
</body>
```
### (3). ULLITest
```
package org.example;

import com.microsoft.playwright.*;
import com.microsoft.playwright.options.WaitUntilState;

public class ULLITest {
  public static void main(String[] args) throws Exception {
    try (Playwright playwright = Playwright.create()) {
      Browser browser = playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(false));
      BrowserContext browserContext = browser.newContext();
      Page page = browserContext.newPage();
      page.navigate("file:///Users/lixin/GitRepository/playwright-java/examples/src/main/resources/index.html", new Page.NavigateOptions().setWaitUntil(WaitUntilState.LOAD));

      // 第一种定位元素的方式
      //Locator aLabel = page.locator(".ulClass.liClass").locator("li").locator("a");
	  // 第二种定位元素的方式
      Locator aLabel = page.locator(".ulClass.liClass >> li >> a");
      int count = aLabel.count();
      for (int i = 0; i < count; i++) {
        Locator liItem = aLabel.nth(i);
        System.out.println(liItem.innerText());
      }
    }
  }
}
```
### (4). 输出结果
```
1111
2222
3333
4444
55555
```
### (5). 总结
在这一小篇学习了级联(>>)选择器.
