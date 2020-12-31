---
layout: post
title: 'Electron API WebView访问'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). WebView相关API
> ["Electron WebView API"](https://www.electronjs.org/docs/api/webview-tag)

### (2). main.js
> <font color='red'>在createWindow时,需要配置:"webviewTag : true",才能在渲染页面显示webview</font>

```
// Modules to control application life and create native browser window
const {app, BrowserWindow} = require('electron')
const path = require('path')

// 创建一个Window
function createWindow () {
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 800,
    webPreferences: {
	   //渲染进程,集成nodejs
      nodeIntegration: true,
	  // 开启webview
      webviewTag : true,
      preload: path.join(__dirname, 'preload.js')
    }
  })

  // and load the index.html of the app.
  mainWindow.loadFile('index.html')
  // mainWindow.loadURL('https://github.com');

  mainWindow.webContents.on("dom-ready",()=>{
    const content = mainWindow.webContents;
    console.log("page load finish *****************************");
  });

  // Open the DevTools.
  mainWindow.webContents.openDevTools()
}

// 当Electron完成初始化时触发
app.whenReady().then(() => {
  console.log(" ******************************** event ready ");
  createWindow();
  
  // 当Electron活动时触发
  app.on('activate', function () {
    if (BrowserWindow.getAllWindows().length === 0) createWindow()
  })
})

// 当Electron所有窗口被关闭时触发 
app.on('window-all-closed', function () {
  if (process.platform !== 'darwin') app.quit()
})
```

### (3). index.html
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
      <span id="loading"></span>
    </div>
    <div>
      <webview src="https://www.baidu.com" preload="./webview/preload.js"></webview>/> 
    </div>
    <script src="./renderer.js"></script>
  </body>
</html>

```
### (4). css/styles.css
```
webview {
    width: 100%;
    height: 800px;
    display: block;
}
```
### (5). renderer.js
```
const fs = require("fs")

onload = () => {
    const webview = document.querySelector('webview')
    const loading = document.getElementById('loading')

    const loadstart = () => {
        // 加载的时候显示的内容
        loading.innerText = 'loading...'
    }

    const loadstop = () => {
        // 加载完成之后清空内容
        loading.innerText = ''

        // 打开webview控制台
        webview.openDevTools();

        // 页面加载完成后,增加CSS效果
        // 增加CSS效果
        webview.insertCSS(`
          #su { background-color:#4e6 !important; }
        `);

        // 执行JS脚本
        let conent = webview.executeJavaScript(`
            let conent = document.getElementsByTagName('html')[0].innerHTML;
            console.log(conent);
        `);
    }

    webview.addEventListener('did-start-loading', loadstart)
    webview.addEventListener('did-stop-loading', loadstop)
}
```
### (6). webview/preload.js
```
setTimeout(()=>{
    // let img = document.getElementById("s_lg_img_new").src;
    // alert(img);
},5000);
```
### (7). 运行结果
!["Electron WebView案例"](/assets/electron/imgs/electron-webview.jpg)