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
        "webRequestBlocking",
        "webRequest",
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
> 注意:chrome.runtime.onMessage.addListener方法中,callback设置返回值只能同步,不能异步,
> 如果要实现异步,则必须:return true;   

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

// 在请求之前,进行拦截,如果:cancel为:true,则放弃请求,否则,继续往后请求.
chrome.webRequest.onBeforeRequest.addListener(
  function(details) {
    return {cancel: details.url.indexOf("://www.jd.com/") != -1};
  },
  {urls: ["<all_urls>"]},
  ["blocking"]
);

// 拦截发送协议头信息,添加或删除Header.
chrome.webRequest.onBeforeSendHeaders.addListener(
  function(details) {
    for (var i = 0; i < details.requestHeaders.length; ++i) {
      if (details.requestHeaders[i].name === 'User-Agent') {
        details.requestHeaders.splice(i, 1);
        break;
      }
    }
    return {requestHeaders: details.requestHeaders};
  },
  {urls: ["<all_urls>"]},
  ["blocking", "requestHeaders"]
);

chrome.runtime.onMessage.addListener(function(message,sender,callback){
    var url = message.url;
    request(url,function(res){
        if("" != res){
            callback({body:res});
        }
    }); 
    // callback只能在同步的模式下使用,如果要在异步模式下使用,必须要加上这一句:return true
    // 否则会在sendMessage的回调函数里出现:undefined.
    return true;
});
```
### (5). 查看HTTP效果
!["Chrome 发起Http请求并返回结果"](/assets/chrome-ext/imgs/chrome-ext-http-result.jpg)

### (6). 查看拦截黑名单效果
!["Chrome 处理黑名单"](/assets/chrome-ext/imgs/chrome-ext-addEvent-onBeforeRequest.jpg)
