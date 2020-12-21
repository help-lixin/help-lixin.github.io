---
layout: post
title: 'Chrome Extension Proxy'
date: 2018-12-20
author: 李新
tags: Chrome Extension
---

### (1). 创建chrome ext(chrome-proxy)
```
lixin-macbook:chrome-ext lixin$ tree chrome-proxy/
chrome-proxy/
├── background.js
├── imgs
│   ├── icon128.png
│   ├── icon16.png
│   └── icon48.png
└── manifest.json
```
### (2). manifest.json
> permissions需要配置:proxy.   

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
        "proxy"
    ]
}
```

### (3). background.js 
> 当访问jd.com时,会将请求向:127.0.0.1:8888进行发送,我特意不开代理.以查看运行效果.

```
var config = {
  mode: "pac_script",
  pacScript: {
      data: "function FindProxyForURL(url, host) {\n" +
            "  console.log(url + ' -  ' + host);  \n" +
            "  if ( host == 'www.jd.com' || host == 'jd.com' )\n" +
            "    return 'PROXY 127.0.0.1:8888';\n" +
            "  return 'DIRECT';\n" +
      "}"
   }
};

chrome.proxy.settings.set({value: config, scope: 'regular'}, function() { console.log("代理设置完成") });
```
### (4). 查看效果
!["Chrome Proxy JD Fail"](/assets/chrome-ext/imgs/chrome-ext-proxy-jd.jpg)

!["Chrome Proxy Baidu Success"](/assets/chrome-ext/imgs/chrome-ext-proxy-baidu.jpg)