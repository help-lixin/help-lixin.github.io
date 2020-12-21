---
layout: post
title: 'Chrome Extension XMLHttpRequest获取数据'
date: 2018-12-20
author: 李新
tags: Chrome Extension
---

### (1). 创建chrome ext(chrome-http)
```
lixin-macbook:chrome-ext lixin$ tree chrome-http/
chrome-http/
├── background.js
├── content.js
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
        "<all_urls>",
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
### (3). content.js
> 页面加载完成之后,注入JS文件(content.js).这个JS文件会发起远程HTTP请求.把请求交给:background.   

```
var url = "https://open.tongtool.com/open-platform-service/devApis/devApisById?apiId=43a41f3680e04756a122d8671f2fc0ca";

chrome.runtime.sendMessage({url:url},function(data){
    console.log("request result:" + data.body);
});
```
### (4). background.js 
> background为后台程序处理.

```
function request(url,callback){
   var xhr = new XMLHttpRequest();
   xhr.open("GET", url);
   xhr.onreadystatechange = function(){
       if(xhr.status == 200){
           callback(xhr.responseText);
       }
   };
   xhr.send();
}

// 监听事件,发起HTTP请求,然后把结果回调回去.
chrome.runtime.onMessage.addListener(function(message,sender,callback){
    var url = message.url;
    request(url,function(res){
        if( "" != res){
            callback({body:res});
        }
    }); 
    // request是异步的情况下,必须要:return true
    return true;
});
```
### (5). 查看效果
!["Chrome 发起Http请求并返回结果"](/assets/chrome-ext/imgs/chrome-ext-http-result.jpg)
