---
layout: post
title: 'Electron API Dialog访问'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). Dialog相关API
> ["Electron Dialog API"](https://www.electronjs.org/docs/api/dialog)

### (2). 注意
> <font color='red'>注意:如果从渲染进程使用dialog,需要在main.js中配置:nodeIntegration和enableRemoteModule都为:true</font>   
> 下面是一个选择多个文件的对话框示例：

```
const { dialog } = require('electron')
console.log(dialog.showOpenDialog({ properties: ['openFile', 'multiSelections'] }))
```
 
> <font color='red'>如果你想要从一个渲染线程使用对话框对象,记得通过remote去访问它.</font>

```
const { dialog } = require('electron').remote
console.log(dialog)
```
 
### (3). main.js
```
const {app, BrowserWindow , BrowserView , screen} = require('electron')
const path = require('path')

// 创建一个Window
function createWindow () {
  // 获取屏幕信息(宽和高)
  const { width, height } = screen.getPrimaryDisplay().workAreaSize

  const mainWindow = new BrowserWindow({
    width: width,
    height: height,
    // 控制有无边框的窗口页面.
    frame : true,
    // 控制窗口是否显示(可结合:ready-to-show事件)
    show:false,
    // 设置背景颜色
    // backgroundColor: '#2e2c29',
    webPreferences: {
      // 是否集成nodejs
      nodeIntegration: true,
      // 开启远程模块
      enableRemoteModule : true,
      // webviewtag开启
      webviewTag : true,
      // 预先需要加载的脚本
      preload: path.join(__dirname, 'preload.js')
    }
  })

  // and load the index.html of the app.
  mainWindow.loadFile('index.html')
  // mainWindow.loadURL('https://github.com');


  // 等到事件触发了才让窗口显示出来
  mainWindow.on("ready-to-show",()=>{
    mainWindow.show();
  });

  mainWindow.webContents.on("dom-ready",()=>{
    const content = mainWindow.webContents;
    console.log("page load finish *****************************");
  });

  // Open the DevTools.
  // mainWindow.webContents.openDevTools()
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
### (4). index.html
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
        <h2>打开文件选择框</h2>      
        <button id="dialogBtn">请选择文件</button>
    </div>
    <script src="./renderer.js"></script>
  </body>
</html>
```
### (5). renderer.js
```

const { dialog } = require('electron').remote
const dialogBtn = document.getElementById("dialogBtn");
dialogBtn.onclick = function(){
    let result = dialog.showOpenDialogSync({
        title : "请选择文件",
        defaultPath:"/Users/lixin/WorkspaceJS/electron-quick-start",
        buttonLabel : "OK",
        filters: [
            { name: 'All Files', extensions: ['*'] }
        ],
        properties:[ "openFile","openDirectory" ]
    });
    console.log(result);
}
```
### (6). 运行效果(showOpenDialogSync)
!["Electron Dialog showOpenDialogSync"](/assets/electron/imgs/electorn-dialog.png)
