---
layout: post
title: 'Electron API事件'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). Electron App事件
> ["Electron App事件参考"](https://www.electronjs.org/docs/all#%E4%BA%8B%E4%BB%B6-will-finish-launching)    
> ready :  当Electron完成初始化时触发.    
> activate : 当Electron活动时触发. 
> window-all-closed : 当Electron所有窗口被关闭时触发.

### (2). Electron WebContent事件
> ["Electron WebConent事件"](https://www.electronjs.org/docs/api/web-contents)   
> <font color='red'>dom-ready : 一个框架中的文本加载完成后触发该事件.</font>   
> did-finish-load : 导航完成时触发,即选项卡的旋转器将停止旋转,并指派onload事件.   

### (3). (Event事件案例)main.js
```
// Modules to control application life and create native browser window
const {app, BrowserWindow} = require('electron')
const path = require('path')

// 创建一个Window
function createWindow () {
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  })

  // and load the index.html of the app.
  // mainWindow.loadFile('index.html')
  // 加载github
  mainWindow.loadURL('https://github.com');
  
  // dom加载完成之后,触发事件
  mainWindow.webContents.on("dom-ready",()=>{
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
