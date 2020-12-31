---
layout: post
title: 'Electron API进程访问'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). 进程相关API
> ["Electron Process API"](https://www.electronjs.org/docs/api/process)

### (2). main.js
> <font color='red'>在createWindow时,需要配置:"nodeIntegration: true",才能在渲染进程访问:nodejs对象</font>

```
const {app, BrowserWindow} = require('electron')
const path = require('path')

// 创建一个Window
function createWindow () {
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
	   // 配置:nodeIntegration后,渲染进程才能访问:process变量
	   // 是否完整的支持node集成,默认是禁用,目的是为了防止跨站脚本攻击.
      nodeIntegration: true,
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
    <title>Hello World!</title>
  </head>
  <body>
    <div>
      <h2>process</h2>
      <button id="viewProcessInfoBtn">view process info</button>
    </div>
    <script src="./renderer.js"></script>
  </body>
</html>
```
### (4). renderer.js
```
// 给页面的按钮绑定事件
let viewProcessInfoBtn = document.getElementById("viewProcessInfoBtn");
viewProcessInfoBtn.onclick = function(){
	// 获取CPU使用率
    console.log("getCPUUsage",process.getCPUUsage());
	// 获取环境变量
    console.log("env",process.env);
	// 获取node的版本
	console.log("version",process.version);
	// 获取平台
	console.log("platform",process.platform);
	// 获取CPU是64位还是32位
	console.log("arch",process.arch);
}
```
### (5). 总结
> <font color='red'>注意:要想在渲染过程访问nodejs对象,就一定要在:main.js中配置:"nodeIntegration: true".</font> 

