---
layout: post
title: 'egg.js 脚手架搭建(一)'
date: 2020-04-23
author: 李新
tags: egg.js
---

### (1). 官网
> ["egg.js"](https://eggjs.org/zh-cn/intro/quickstart.html)

### (2). 环境搭建
```
// 安装客户端
lixin-macbook:Desktop lixin$ npm install egg-init  -g
// 创建项目工作区
lixin-macbook:Desktop lixin$ mkdir egg-example && cd egg-example
// 创建egg项目
lixin-macbook:egg-example lixin$ npm init egg --type=simple
// 安装依赖
lixin-macbook:egg-example lixin$ npm install
```
### (3). 启动
```
// 启动脚本
lixin-macbook:egg-example lixin$ npm start
```
### (4). 测试访问
```
// 测试访问
lixin-macbook:egg-example lixin$ curl http://127.0.0.1:7001
	hi, egg
```
