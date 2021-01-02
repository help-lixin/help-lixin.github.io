---
layout: post
title: 'Koa2 Hello World(一)'
date: 2020-04-22
author: 李新
tags: Koa2
---

### (1). 初始化项目
```
# 工作目录
lixin-macbook:Desktop lixin$ pwd
	/Users/lixin/Desktop
# 创建项目工程
lixin-macbook:Desktop lixin$ mkdir koa-example
# 初始化骨架
lixin-macbook:koa-example lixin$ npm init -y
# 下载koa
lixin-macbook:koa-example lixin$ cnpm install koa --save
```
### (2). app.js
```
// 引入koa
const Koa = require("koa");
const app = new Koa();

// 配置中间件(类似于Java中的Filter)
app.use(async (ctx) => {
    ctx.body = "hello world koa2";
});

// 监听端口
app.listen(8080);
```
### (3). 项目结构如下
```
lixin-macbook:koa-example lixin$ tree -L 1
.
├── app.js
├── node_modules
├── package-lock.json
└── package.json

```
### (4). 运行项目
```
lixin-macbook:koa-example lixin$ node app.js
```
### (5). 测试
```
lixin-macbook:~ lixin$ curl http://localhost:8080
	hello world koa2
```
