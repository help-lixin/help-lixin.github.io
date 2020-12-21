---
layout: post
title: 'Chrome Extension Menu'
date: 2018-12-20
author: 李新
tags: Chrome Extension
---

### (1). 创建chrome ext(chrome-hello)
```
lixin-macbook:chrome-hello lixin$ tree
.
├── background.js
├── imgs
│   ├── icon128.png
│   ├── icon16.png
│   └── icon48.png
└── manifest.json
```
### (2). manifest.json
> https://www.iconfont.cn/ 下载图标.    
> icons 定义图标.     
> background 定义后台程序.    
> permissions授权设置.   

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
        "contextMenus"
    ]
}
```
### (3). background.js
> 在Chrome页面,右键在弹出的快捷键中选择:"~Welcome~".
> 可创建多个.

```
chrome.contextMenus.create(
    {
     type : "normal",
     title : "~Welcome~",
     contexts : [ "all" ],
     onclick : function(i){
        console.log(i);
     }
    }
    ,function(){
     
    }
);

chrome.contextMenus.create(
    {
     type : "checkbox",
     title : "~Hello~",
     contexts : [ "all" ],
     onclick : function(i){
        console.log(i);
     }
    }
    ,function(){
     
    }
);

```

### (4). 查看效果
!["Chrome Menus"](/assets/chrome-ext/imgs/chrome-ext-menus-click.jpg)

