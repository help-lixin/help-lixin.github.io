---
layout: post
title: 'Electron API Window访问'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). Window相关API
> ["Electron WebView API"](https://www.electronjs.org/docs/api/window-open)

### (2). index.html
```
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <!-- https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP -->
    <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
    <meta http-equiv="X-Content-Security-Policy" content="default-src 'self'; script-src 'self'">
    <link rel="stylesheet" href="css/styles.css"/>
    <title>Hello World!</title>
  </head>
  <body>
    <div>
      <h2>弹出子窗口</h2>
      <button id="windowOpenBtn">弹出子窗口</button>
      <button id="windowCloseBtn">关闭子窗口</button>
    </div>
    <script src="./renderer.js"></script>
  </body>
</html>
```
### (3). renderer.js
```
let windowOpenBtn = document.getElementById("windowOpenBtn");
let browserWindow = null;
windowOpenBtn.onclick = function(){
    browserWindow = window.open("https://www.baidu.com","百度");
}

let windowCloseBtn = document.getElementById("windowCloseBtn");
windowCloseBtn.onclick = function(){
    browserWindow.close();
}
```
