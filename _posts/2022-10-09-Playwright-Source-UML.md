---
layout: post
title: 'Playwright源码学习之全局俯瞰核心类(一)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
在看Playwright之前,最好的方式是能俯瞰整个项目的一些核心类出来,所以,先对前面学习的类,把UML图画出来.  
### (2). Playwright类图
!["Playwright核心类图"](/assets/playwright/imgs/playwright-uml.jpg)
### (3). Playwright
从上面的UML中能分析出来,Playwright的主要职责是:根据不同的浏览器(chrome/firefox/webkit),创建:BrowserType.  
### (4). BrowserType
BrowserType的主要职责是:创建Browser(浏览器).
### (5). Browser
Browser的主要职责是:创建BrowserContext或Page.
### (6). Page
Page的主要职责是:定位元素/键盘事件/鼠标事件/JS执行. 
### (7). 总结
从全局来看这些核心接口与方法签名之后,无需去看所有的实现类了(后面只研究下底层到底是如何与浏览器通信的),这是一种看源码学习的方式. 