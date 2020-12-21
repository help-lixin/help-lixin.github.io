---
layout: post
title: 'Chrome Extension background 创建通知'
date: 2018-12-20
author: 李新
tags: Chrome Extension
---

### (1). 创建chrome ext(chrome-hello2)
```
lixin-macbook:chrome-ext lixin$ tree
.
└── chrome-hello2
    ├── background.js
    ├── imgs
    │   ├── icon128.png
    │   ├── icon16.png
    │   └── icon48.png
    └── manifest.json
```
### (2). manifest.json
> permissions需要配置:notifications.

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
        "notifications",
        "proxy"
    ]
}
```
### (3). background.js
```
chrome.notifications.create({
    type : "basic",
    title : "最新通知",
    iconUrl : "imgs/icon48.png",
    message : "2019年最新放假通知..."
},function(){});
```
### (4). 查看效果
!["Chrome Notification"](/assets/chrome-ext/imgs/chrome-ext-notification.jpg)