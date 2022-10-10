---
layout: post
title: 'Playwright Checkbox标签学习(六)'
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
	<label>Checkbox Test </label>
	<input class="body checkbox" type="checkbox" name="love" value="苹果" checked="true"/>
	<input class="body checkbox" type="checkbox" name="love" value="香蕉"/>
	<input class="body checkbox" type="checkbox" name="love" value="梨子"/>
	<br/>
</body>
```
### (3). CheckboxTest
```
package org.example;

import com.microsoft.playwright.*;
import com.microsoft.playwright.options.WaitUntilState;

import java.util.ArrayList;
import java.util.List;

public class CheckboxTest {
  public static void main(String[] args) throws Exception {
    try (Playwright playwright = Playwright.create()) {
      Browser browser = playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(false));
      BrowserContext browserContext = browser.newContext();
      Page page = browserContext.newPage();
      page.navigate("file:///Users/lixin/GitRepository/playwright-java/examples/src/main/resources/index.html", new Page.NavigateOptions().setWaitUntil(WaitUntilState.LOAD));

      Locator checkboxLocator = page.locator(".body.checkbox");
	  // 针对第2型进行选中
	  // checkboxLocator.nth(2).check();
      int count = checkboxLocator.count();
      for (int i = 0; i < count; i++) {
        Locator checkboxItem = checkboxLocator.nth(i);
        String name = checkboxItem.getAttribute("name");
        String value = checkboxItem.getAttribute("value");
        System.out.println("name:" + name + " value: " + value);
        
		if (i == 2) {
          // 针对某一个元素进行选中
          checkboxItem.check();
        }
      }

      // 遍历所有的元素,判断是否被选中
      List<String> checkValues = new ArrayList<>();
      for (int i = 0; i < count; i++) {
        Locator checkboxItem = checkboxLocator.nth(i);
        String value = checkboxItem.getAttribute("value");
        if (checkboxItem.isChecked()) {
          checkValues.add(value);
        }
      }
      System.out.println(checkValues);
    }
  }
}
```
### (4). 输出结果
```
name:love value: 苹果
name:love value: 香蕉
name:love value: 梨子

// 被选中的内容
[苹果, 梨子]
```
### (5). 总结
在这一小篇主要是对Checkbox进行定位,设置值和获取值.