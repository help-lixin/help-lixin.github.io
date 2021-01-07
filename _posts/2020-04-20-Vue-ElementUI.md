---
layout: post
title: 'Vue ElementUI结合'
date: 2020-04-20
author: 李新
tags: Vue ElementUI
---

### (1). 官网
> ["ElementUI"](https://element.eleme.cn/)   

### (2). 创建项目
```
lixin-macbook:Desktop lixin$ vue-init webpack vue-elementui

? Project name vue-elementui
? Project description A Vue.js project
? Author xxxx@126.com
? Vue build standalone
? Install vue-router? No   //自己安装路由
? Use ESLint to lint your code? No
? Set up unit tests No
? Setup e2e tests with Nightwatch? No
? Should we run `npm install` for you after the project has been created? (recom
mended) npm
```
### (3). 安装模块
```
# vue-router
lixin-macbook:vue-elementui lixin$ cnpm install vue-router --save
# element-ui
lixin-macbook:vue-elementui lixin$ cnpm install element-ui  --save
# saas加载器(最新版本8.X好像有问题)
lixin-macbook:vue-elementui lixin$ npm install sass-loader@7.3.1 node-sass@4.14.1  --save-dev
```
### (4). 项目下载
["Vue ElementUI案例"](/assets/vue/vue-elementui.zip)