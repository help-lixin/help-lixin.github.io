---
layout: post
title: 'Chrome Extension 注入JS'
date: 2018-12-20
author: 李新
tags: Chrome Extension
---

### (1). 创建chrome ext(chrome-hello)
```
lixin-macbook:chrome-hello lixin$ tree
.
└── chrome-hello
    ├── background.js
    ├── content.js
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
> content_scripts为注入的JS.matches:符合正则的则注入JS.<all_urls>代表所有页面.     
> matches:["*://www.baidu.com/*"].     
> run_at:代表在什么时机加载:document_end/document_start/document_idle.    



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
    "content_scripts" : [
        { 
           "matches" :  ["<all_urls>"],
           "js" : [ "content.js" ],
           "run_at" : "document_end"
        }
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

### (4). content.js
```
console.log("~~~~~~~~~~~~hello world~~~~~~~~~~~~~~~~~");
```

### (5). 查看效果
!["Chrome注入JS文件"](/assets/chrome-ext/imgs/chrome-ext-inject-js.png)