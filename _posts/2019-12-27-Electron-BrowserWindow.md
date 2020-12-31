---
layout: post
title: 'Electron API BrowserWindow访问'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). BrowserWindow相关API
> ["Electron BrowserWindow API"](https://www.electronjs.org/docs/api/browser-window#browserwindowgetallwindows)

### (2). main.js
```
const {app, BrowserWindow , screen} = require('electron')
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

  // 创建子窗口
  // const child = new BrowserWindow({ parent: mainWindow })
  // child.show();
  // child.loadURL("https://www.baidu.com");

  // 创建模态窗口(只允许操作子窗口后,才能操作父窗口,比如:登录/广告)
  // const child = new BrowserWindow({ parent: mainWindow , modal: true , show: false  })
  // child.loadURL("https://www.baidu.com");
  // child.once('ready-to-show', () => {
  //   child.show()
  // });

  // and load the index.html of the app.
  mainWindow.loadFile('index.html')
  // mainWindow.loadURL('https://github.com');


  // 等到事件触发了才让窗口显示出来
  mainWindow.on("ready-to-show",()=>{
    mainWindow.show();
  });

  // webContents加载完dom后触发事件
	mainWindow.webContents.on("dom-ready",(evnet)=>{
	  // 保存页面到指定路径,该方法不太靠谱.
	  // 按理来说:JS渲染HTML之后,才应该回调该方法,但是,从现在情况来看
	  // 该方法是只要DOM渲染完成就回调.所以,我只能5秒后再保存到文件,但是:5秒有时也不够用.
	  setTimeout(()=>{
	     const content = mainWindow.webContents;
	     content.savePage("/Users/lixin/WorkspaceJS/electron-quick-start/open-api.html","HTMLComplete",(error)=>{
	       if (!error) console.log('Save page successfully')
	     });
	     console.log("page load finish *****************************");
	   },5000);
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
