---
layout: post
title: 'Electron API文件访问'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). File相关API
> ["Electron File API"](https://www.electronjs.org/docs/api/file-object)

### (2). main.js
> <font color='red'>在createWindow时,需要配置:"nodeIntegration: true",才能在渲染进程访问:nodejs对象</font>
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
    <div id="holder">
      Drag your file here
    </div>
    <script src="./renderer.js"></script>
  </body>
</html>
```
### (4). renderer.js
```
const fs = require("fs")

document.addEventListener('drop', (e) => {
    e.preventDefault();
    e.stopPropagation();

    for (const f of e.dataTransfer.files) {
      // 读取文件
      console.log('File(s) you dragged here: ', f.path)
      let conent = fs.readFileSync(f.path)
      console.log("Content:",conent.toString())
    }
});

// 阻止默认行为
document.addEventListener('dragover', (e) => {
    e.preventDefault();
    e.stopPropagation();
});
```
### (5). 测试拖动文件到div(略)
> 能读取文件内容

### (6). 运行结果
!["Electron File操作"](/assets/electron/imgs/electron-file.jpg)
