---
layout: post
title: 'Chrome Extension background与popup通信'
date: 2018-12-20
author: 李新
tags: ChromeExtension
---

### (1). 创建chrome ext(chrome-hello3)
```
lixin-macbook:chrome-ext lixin$ tree chrome-hello3
chrome-hello3
├── background.js
├── demo.html
├── imgs
│   ├── icon128.png
│   ├── icon16.png
│   └── icon48.png
├── js
│   └── demo.js
└── manifest.json
```
### (2). manifest.json
```
{
    "manifest_version": 2,
    "name" : "Hello World-lixin",
    "version" : "2.0.0",
    "description" : "Welcome ...",
    "icons" : {
        "16"   : "imgs/icon16.png",
        "48"   : "imgs/icon48.png",
        "128"  : "imgs/icon128.png"
    },
    "background" : {
        "scripts" : ["background.js"]
    },
    "permissions" : [
        "contextMenus",
        "proxy"
    ],
    "browser_action" : {
        "default_title" : "test",
        "default_popup" : "demo.html",
        "default_icon" : {
           "48" : "imgs/icon48.png"
        }
    }
}
```
### (3). demo.html 
> 弹出页,引用了外部JS

```
<html>
<head>
<style>
* {
    margin: 0;
    padding: 0;
}

body {
    width: 200px;
    height: 100px;
}

div {
    line-height: 100px;
    font-size: 42px;
    text-align: center;
}
</style>
</head>
<body>
<div id="div">
   <button id="send">send to...</button>
</div>
<div id="result">
   
</div>
<script src="js/demo.js"></script>
</body>
</html>
```
### (4). js/demo.js 
> 弹出页面的业务逻辑

```
// 监听按钮事件
document.getElementById("send").onclick = function(){
   //  向bg发送消息
   chrome.runtime.sendMessage({greeting:"hello"},function(res){
      var result = "request result:" + res.success + " - message: " + res.message;
      // 把返回的结果写入到:弹出页div中.
      document.getElementById("result").innerHTML = result;
   });   
}
```
### (5). background.js 
```
chrome.runtime.onMessage.addListener(function(message,sender,sendResponse){
    // 向sendResponse发送消息.
    sendResponse({success:true,message:"success process handler"});
});
```
### (6). 总结
> chrome.runtime.sendMessage 发送消息.    
> chrome.runtime.onMessage.addListener 接受消息.

### (7). 查看效果
!["Chrome 通信效果"](/assets/chrome-ext/imgs/chrome-ext-sendmessage.jpg)
