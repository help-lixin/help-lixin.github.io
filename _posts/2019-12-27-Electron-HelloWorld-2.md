---
layout: post
title: 'Electron HelloWorld(2)'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). electron-quick-start项目结构
```
electron-quick-start
├── LICENSE.md
├── README.md
├── index.html
├── main.js
├── node_modules
├── package.json
├── preload.js
└── renderer.js
```
### (2). package.json
> npm run start/debug       
> <font color='red'>"electron .",中的点代表运行:main.js</font>    
> <font color='red'>"electron --inspect=5888 .",代表对electron主进程进行debug.

```
{
  "name": "electron-quick-start",
  "version": "1.0.0",
  "description": "A minimal Electron application",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "debug": "electron --inspect=5888 ."
  },
  "repository": "https://github.com/electron/electron-quick-start",
  "keywords": [
    "Electron",
    "quick",
    "start",
    "tutorial",
    "demo"
  ],
  "author": "GitHub",
  "license": "CC0-1.0",
  "devDependencies": {
    "electron": "^11.1.1"
  }
}
```
### (3). 
```

const {app, BrowserWindow} = require('electron')
const path = require('path')

function createWindow () {
  // Create the browser window.
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  })

  // and load the index.html of the app.
  mainWindow.loadFile('index.html')

  // Open the DevTools.
  // 打开渲染进程的DEBUG
  mainWindow.webContents.openDevTools()
}


app.whenReady().then(() => {
  createWindow()
  
  app.on('activate', function () {
    if (BrowserWindow.getAllWindows().length === 0) createWindow()
  })
})

app.on('window-all-closed', function () {
  if (process.platform !== 'darwin') app.quit()
})
```
### (4). preload.js
```
// All of the Node.js APIs are available in the preload process.
// It has the same sandbox as a Chrome extension.
window.addEventListener('DOMContentLoaded', () => {
  const replaceText = (selector, text) => {
    const element = document.getElementById(selector)
    if (element) element.innerText = text
  }

  for (const type of ['chrome', 'node', 'electron']) {
    replaceText(`${type}-version`, process.versions[type])
  }
})
```
### (5). index.html
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
    <h1>Hello World!</h1>
    We are using Node.js <span id="node-version"></span>,
    Chromium <span id="chrome-version"></span>,
    and Electron <span id="electron-version"></span>.

    <!-- You can also require other files to run in this process -->
    <script src="./renderer.js"></script>
  </body>
</html>
```
### (6). renderer.js
```
// This file is required by the index.html file and will
// be executed in the renderer process for that window.
// No Node.js APIs are available in this process because
// `nodeIntegration` is turned off. Use `preload.js` to
// selectively enable features needed in the rendering
// process.
```
### (7). 主进程调试方式
> debug 控制台信息

```
lixin-macbook:electron-quick-start lixin$ npm run debug

> electron-quick-start@1.0.0 debug /Users/lixin/WorkspaceJS/electron-quick-start
> electron --inspect=5888 .

Debugger listening on ws://127.0.0.1:5888/0dbd7761-515a-456f-9c52-f5ca733368e4
For help, see: https://nodejs.org/en/docs/inspector
(node:2711) electron: The default of contextIsolation is deprecated and will be changing from false to true in a future release of Electr
on.  See https://github.com/electron/electron/issues/23506 for more information
```
### (8). vs code 调试
> launch.json

```
{ 
    "version": "0.2.0", 
    "configurations": 
    [
        {
            "name": "Attach",
            "type": "node", 
            "request": "attach", 
            "port": 5888, 
            "sourceMaps": false, 
            "outDir": null, 
            "localRoot": "${workspaceRoot}", 
            "remoteRoot": null, 
            "address": "localhost"
         } 
    ] 
}
```

!["Electron Vs Code Debug"](/assets/electron/imgs/electron-vscode-debug.jpg)