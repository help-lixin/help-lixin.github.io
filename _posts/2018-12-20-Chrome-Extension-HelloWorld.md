---
layout: post
title: 'Chrome Extension Hello World'
date: 2018-12-20
author: 李新
tags: ChromeExtension
---

### (1). 创建chrome ext(chrome-hello)
> 参考学习链接:     
https://www.ituring.com.cn/book/1421
https://developer.chrome.com/docs/extensions/mv2/devguide/


```
lixin-macbook:chrome-ext lixin$ tree
.
└── chrome-hello
    ├── imgs
    │   ├── icon128.png
    │   ├── icon16.png
    │   └── icon48.png
    └── manifest.json
```
### (2). manifest.json
> https://www.iconfont.cn/ 下载图标.    
> icons 定义图标.    

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
    } 
}
```
### (3). Chrome查看插件
!["Chrome查看"](/assets/chrome-ext/imgs/chrome-ext-helloworld.png)
