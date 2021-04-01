---
layout: post
title: 'Chrome Extension Tabls'
date: 2018-12-20
author: 李新
tags: ChromeExtension
---

### (1). 创建chrome ext(chrome-other)
```
lixin-macbook:chrome-ext lixin$ tree chrome-other/
chrome-other/
├── background.js
├── imgs
│   ├── icon128.png
│   ├── icon16.png
│   └── icon48.png
└── manifest.json
```
### (2). manifest.json

```
{
    "manifest_version": 2,
    "name" : "Proxy",
    "version" : "2.0.0",
    "description" : "Welcome Proxy",
    "icons" : {
        "16"   : "imgs/icon16.png",
        "48"   : "imgs/icon48.png",
        "128"  : "imgs/icon128.png"
    },
    "background" : {
        "scripts" : ["background.js"]
    },
    "permissions" : [
        "<all_urls>",
        "proxy",
        "tabs"
    ]
}
```

### (3). background.js 
```
var tables = [];

function queryTables(arr){
    for(var index in  arr){
       var active = arr[index].active;
       var tableId = arr[index].id;
       var title = arr[index].title;
       var url = arr[index].url;
       var windowId = arr[index].windowId;
       tables.push({ tableId:tableId,title:title,url:url,active:active,windowId:windowId  });
    }// end for
    console.log(tables);
}

// 获取某个tab详细信息
function getTab(){
    var tab = tables[0];
    chrome.tabs.get(tab.tableId,function(t){
        console.log(t);
    });
}

// 创建tab
function createTab(){
     chrome.tabs.create({
        url:"https://www.lixin.help"
     },function(tab){
      var list = [tab];
      queryTables(list);
     });
}

// 跳转到某个URL
function toURL(){
    var tab = tables[0];
    chrome.tabs.update(tab.tableId,{url:"https://www.lixin.help"},function(t){
        console.log(t);
    });
}

// 不带任何查询条件,获得所有的tab
chrome.tabs.query({},function(arr){
    queryTables(arr);
    getTab();
    console.log("create tabe;");
    createTab();
    toURL();
});
```

### (4). 通过注入JS获取页面源码.
```
var html = document.documentElement.innerHTML;
chrome.runtime.sendMessage({html:html},function(res){
   console.log(res.body);
});
```

### (5). 查看效果
!["Chrome Proxy JD Fail"](/assets/chrome-ext/imgs/chrome-ext-proxy-jd.jpg)

!["Chrome Proxy Baidu Success"](/assets/chrome-ext/imgs/chrome-ext-proxy-baidu.jpg)