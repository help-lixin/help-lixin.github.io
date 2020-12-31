---
layout: post
title: 'Electron 爬取网站,获得JS渲染后的HTML'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). 需求
> AJAX请求之后,会对Document进行渲染,期望在ajax之后,还能拿到Ajax渲染的数据.

### (2). main.js
> 在man.js里,尝试过用(mainWindow.webContents.on("dom-ready"...)),也未能解决该方法,而且代码有些丑.

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
    show:true,
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
  // mainWindow.loadFile('index.html')
  // mainWindow.loadURL('https://www.baidu.com');
  mainWindow.loadURL('https://open.tongtool.com/apiDoc.html#/?docId=43a41f3680e04756a122d8671f2fc0ca');

  // Open the DevTools.
  mainWindow.webContents.openDevTools()

  // webContents加载完dom后触发事件
  mainWindow.webContents.on("dom-ready",(evnet)=>{
    // 保存页面到指定路径,该方法不太靠谱.
    // 按理来说:JS渲染HTML之后,才应该回调该方法,但是,从现在情况来看
    // 该方法是只要DOM渲染完成就回调.
    // setTimeout(()=>{
    //   const content = mainWindow.webContents;
    //   content.savePage("/Users/lixin/WorkspaceJS/electron-quick-start/open-api.html","HTMLComplete",(error)=>{
    //     if (!error) console.log('Save page successfully')
    //   });
    //   console.log("page load finish *****************************");
    // },5000);
  });
}

// 当Electron完成初始化时触发
app.whenReady().then(() => {
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
### (3). preload.js
> DOMContentLoaded 和 onload不同之处在于,DOMContentLoaded属于DOM节点加载完成,而onload代表所有元素(CSS/JS)都加载完成.

```

// 加载文件系统
const fs = require("fs");

// 页面所有的依赖元素都加载完成后,再执行该onload方法
onload = () => {
   //  整个页面所有内容加载完成之后,再去dump HTML内容
   // 稍微延迟一下(2000).防止JS还在渲染HTML.
    setTimeout(()=>{
	  // 获得Document
      let doc = document.documentElement.outerHTML;
	  // 写出到指定路径
      let path = "/Users/lixin/WorkspaceJS/electron-quick-start/open-api.html"
	  // 这里可以进行异步写入.
      fs.writeFileSync(path,doc)
    } , 2000);
}



window.addEventListener('DOMContentLoaded', () => {
  const replaceText = (selector, text) => {
    const element = document.getElementById(selector)
    if (element) element.innerText = text
  }

  for (const type of ['chrome', 'node', 'electron']) {
    replaceText(`${type}-version`, process.versions[type])
  }
});

```

### (4). 测试效果(10次)
!["Electron onload"](/assets/electron/imgs/electron-onload.jpg)
### (5). 总结
> 完美的解决方案还是用:onload而不是:DOMContentLoaded.所以,对于JS还是需要更深的了解.
