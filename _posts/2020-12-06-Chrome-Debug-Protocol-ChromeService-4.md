---
layout: post
title: 'Chrome Debug Protocol--ChromeService(1)(四)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). 概述
> 前几节剖析到,会根据chrome的安装目录,创建一个chrome进程,并从chrome进程里获取到监听到的端口.然后委托给:ChromeService创建websocket请求.  
> 这一节,主要剖析:ChromeService相关的API   

### (2). LogRequestsExample
```
// Create chrome launcher.
final ChromeLauncher launcher = new ChromeLauncher();

// Launch chrome either as headless (true) or regular (false).
final ChromeService chromeService = launcher.launch(false);

// *****************************************************************
// Create empty tab ie about:blank.
// 3. 调用chrome创建:ChromeTab
// *****************************************************************
final ChromeTab tab = chromeService.createTab();
```
### (3). ChromeServiceImpl.createTab
```
public static final String ABOUT_BLANK_PAGE = "about:blank";

public ChromeTab createTab() throws ChromeServiceException {
  // 1. 创建一个空白页面
  return createTab(ABOUT_BLANK_PAGE);
} //end createTab



public ChromeTab createTab(String tab) throws ChromeServiceException {
  // 2. 调用内部私有:request方法
  return request(ChromeTab.class, "http://%s:%d/%s?%s", host, port, CREATE_TAB, tab);
}

```
### (4). ChromeServiceImpl.request
```
private static <T> T request(
      // com.github.kklisura.cdt.services.types.ChromeTab
      Class<T> responseType, 
      // http://%s:%d/%s?%s
      String path, 
      // [localhost, 54614, json/new, about:blank]
      Object... params)
      throws ChromeServiceException {

    HttpURLConnection connection = null;
    InputStream inputStream = null;

  try {
    // uri = "http://localhost:54614/json/new?about:blank"
    URL uri = new URL(String.format(path, params));
    connection = (HttpURLConnection) uri.openConnection();

    int responseCode = connection.getResponseCode();
    // 成功的情况下
    if (HttpURLConnection.HTTP_OK == responseCode) { 
      if (Void.class.equals(responseType)) {
        return null;
      }

      inputStream = connection.getInputStream();
      // 读取InputStream内容转换成对象:com.github.kklisura.cdt.services.types.ChromeTab
      return OBJECT_MAPPER.readerFor(responseType).readValue(inputStream);
    }

    // 不成功的情况下,读取错误信息
    inputStream = connection.getErrorStream();
    final String responseBody = inputStreamToString(inputStream);

    String message =
        MessageFormat.format(
            "Server responded with non-200 code: {0} - {1}. {2}",
            responseCode, connection.getResponseMessage(), responseBody);
    throw new ChromeServiceException(message);
  } catch (IOException ex) {
    throw new ChromeServiceException("Failed sending HTTP request.", ex);
  } finally {
    if (inputStream != null) {
      try {
        inputStream.close();
      } catch (IOException e) {
        // We can ignore this.
      }
    }

    if (connection != null) {
      connection.disconnect();
    }
  }
} // end request
```

### (5). ChromeService接口定义
```
package com.github.kklisura.cdt.services;

public interface ChromeService {
  // 获取所有的Table
  List<ChromeTab> getTabs() throws ChromeServiceException;

  // 创建一个(about:blank)页面
  ChromeTab createTab() throws ChromeServiceException;

  // 创建指定URL的页面
  ChromeTab createTab(String url) throws ChromeServiceException;

  // 激活某个tab
  void activateTab(ChromeTab tab) throws ChromeServiceException;

  // 关闭某个tabe
  void closeTab(ChromeTab ta) throws ChromeServiceException;

  // 获取chrome的版本
  ChromeVersion getVersion() throws ChromeServiceException;;

  // 创建:ChromeDevToolsService
  ChromeDevToolsService createDevToolsService(
      ChromeTab tab, ChromeDevToolsServiceConfiguration chromeDevToolsServiceConfiguration)
      throws ChromeServiceException;

  // 创建:ChromeDevToolsService
  ChromeDevToolsService createDevToolsService(ChromeTab tab) throws ChromeServiceException;
} // end ChromeService

```
### (6). 总结
> 向Chrome发起Http(http://localhost:54614/json/new?about:blank)请求,创建Tab.  
> 从接口:ChromeService的定久,能判断:ChromeService主要负责:创建(关闭/激活)ChromeTab,并根据:ChromeTab创建:ChromeDevToolsService.   
