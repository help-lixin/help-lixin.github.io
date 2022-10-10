---
layout: post
title: 'Playwright下拉列表框(三)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
在这一小篇开始,对每一个HTML元素,通过Playwright进行定位.
### (2). html
> 在设计HTML时,特意让有一部份Select选项是通过JS脚本动态生成的.

```
<!DOCTYPE html>
<html>
<head>
  <title>Test</title>
  <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
  <script src="./jquery.min.js"></script>
</head>
<body>
	<label>Select Test </label>
	<select class="select">
	  <option value="black">Black</option>
	  <option value="blue">Blue</option>
	</select>
	<br/>

	<script>
		$(function(){
		  // 8秒以后,动态添加option元素	
		  setTimeout(function(){
			$(".select").append('<option value="brown">Brown</option>');
		  },8000);
		  
		});
	</script>
</body>
```
### (3). SelectorsTest
```
package org.example;

import com.microsoft.playwright.*;
import com.microsoft.playwright.options.SelectOption;
import com.microsoft.playwright.options.WaitForSelectorState;
import com.microsoft.playwright.options.WaitUntilState;

public class SelectorsTest {
  public static void main(String[] args) throws Exception {
    try (Playwright playwright = Playwright.create()) {
      Browser browser = playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(false));
      BrowserContext browserContext = browser.newContext();
      Page page = browserContext.newPage();
      page.navigate("file:///Users/lixin/GitRepository/playwright-java/examples/src/main/resources/index.html", new Page.NavigateOptions().setWaitUntil(WaitUntilState.LOAD));

       // 期待某个元素出现,程序才会继续往下走(默认是30超时来着)
      System.out.println(System.currentTimeMillis());
      page.locator("text=Brown").waitFor(new Locator.WaitForOptions().setState(WaitForSelectorState.ATTACHED));
      System.out.println(System.currentTimeMillis());

      // 获取所有下列表表框的属性和内容
      Locator allOptions = page.locator(".select").locator("option");
      int count = allOptions.count();
      for (int i = 0; i < count; i++) {
        Locator optionItem = allOptions.nth(i);
        String value = optionItem.getAttribute("value");
        String label = optionItem.textContent();
        System.out.println(" onload label:" + label + " value: " + value);
      }

      // select设置值
      // page.evaluate("$(\".select\").val(\"brown\")");
      page.selectOption(".select", new SelectOption().setValue("brown"));

      // 获取select选中的值
      Object selectValue = page.evaluate("$(\".select\").val()");
      System.out.println(selectValue);
    }
  }
}
```
### (4). 控制台输出结果
```
# 通过控制台时间能看出,等待了8秒钟,因为JS设置了8秒.
1665399004954
1665399012984
 onload label:Black value: black
 onload label:Blue value: blue
 onload label:Brown value: brown

# 最后选择的值是:brown
brown
```
### (5). 结果
通过Select,学习了几个API(locator/getAttribute/evaluate).  