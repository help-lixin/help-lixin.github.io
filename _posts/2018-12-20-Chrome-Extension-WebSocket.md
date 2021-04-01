---
layout: post
title: 'Chrome Extension WebSocket'
date: 2018-12-20
author: 李新
tags: ChromeExtension
---

### (1). 创建chrome ext(chrome-websocket)
> https://blog.mn886.net/chenjianhua/show/6b02fa4173ed/index.html 

```
lixin-macbook:chrome-ext lixin$ tree chrome-websocket/
chrome-websocket/
├── background.js
├── imgs
│   ├── icon128.png
│   ├── icon16.png
│   └── icon48.png
└── manifest.json
```
### (2). manifest.json
> Chrome天生支持:WebSocket,所以,不需要加入任何的权限.

```
{
    "manifest_version": 2,
    "name" : "WebSocket",
    "version" : "2.0.0",
    "description" : "Welcome WebSocket",
    "icons" : {
        "16"   : "imgs/icon16.png",
        "48"   : "imgs/icon48.png",
        "128"  : "imgs/icon128.png"
    },
    "background" : {
        "scripts" : ["background.js"]
    },
    "permissions" : [
        // "<all_urls>",
        // "proxy"
    ]
}

```

### (3). background.js 
```
var connection = new WebSocket('ws://html5rocks.websocket.org/echo');
var connectionSuccess = false;

// 连接成功的回调
connection.onopen = function(){
    console.log("============websocket connection success==========");
    connectionSuccess = true;
    // 尝试发送信息.
    connection.send("hello wolrd!!!");
}

// 接受消息回调
connection.onmessage = function(result){
    // 接受消息并打印
    console.log("revice message " + result);
}

// 连接出错回调
connection.onerror = function(error){
    console.log(error);
};
```

### (4). 查看效果

!["Chrome WebSocket演示"](/assets/chrome-ext/imgs/chrome-ext-websocket.jpg)