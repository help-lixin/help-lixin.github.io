---
layout: post
title: 'Electron API BrowserView访问'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). BrowserView相关API
> ["Electron BrowserView API"](https://www.electronjs.org/docs/api/browser-view)

### (2). main.js
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
      // webviewtag开启
      webviewTag : true,
      // 预先需要加载的脚本
      preload: path.join(__dirname, 'preload.js')
    }
  })

  // 创建:BrowserView
  const view = new BrowserView()
  view.setBounds({ x: 0, y: 0, width: width, height: height })
  view.webContents.loadURL('https://www.baidu.com')
  // 将view与window绑定
  mainWindow.setBrowserView(view)

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
