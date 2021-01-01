---
layout: post
title: 'Koa2 中间件(三)'
date: 2020-04-22
author: 李新
tags: Koa2
---

### (1). 什么是中间件
> 中间件就是:路由之前或者路由之后做的一系列操作(类似于Java中的Filter),可实现拦截判断.
> 中间件的功能包括:    
> 1. 执行任何代码.
> 2. 修改请求和响应对象.  
> 3. 终结请求. 
> 4. 调用堆栈中的下一个中间件.  

### (2). app.js
```
// 引入koa
const Koa = require("koa");
// 引入koa-router
const Router = require('koa-router');
// 创建Koa实例
const app = new Koa();
// 创建Router实例
const router = new Router();

// 配置路由
// curl -X GET http://localhost:8080?name=zhgnsan
router.get('/', async (ctx, next) => {
    let query = ctx.query;
    // {name : "zhgnsan"}
    ctx.body = 'GET Hello World! request param name:' + query.name;
});

// 配置动态路由
router.get('/get/:name', async (ctx, next) => {
    let name = ctx.params.name;
    ctx.body = "GET /get/" + name;
});

// **************************************应用级中间件**************************************
// app.use(function(){...})  代表拦截所有的请求.
// app.use("/get/:name",function(){...})  代表仅拦截/get/:name请求.
// app.use(async (ctx,next) => {
//     let nowDate = new Date();
//     let url = ctx.request.url;
//     console.log(`
//       ${nowDate} request before url:${url}
//     `);

//     // 继续执行下一个中间件(Filter)
//     await next();
// });

// **************************************路由级中间件**************************************
// 路由是从上至下match的,当match到某一个了,就认为是命中了.而next代表继续往下执行.
// match到:/hello请求后,继续向下路由.
// router.get("/hello",async (ctx,next)=>{
//     console.log("这是hello1");
//     await next();
// });

// router.get("/hello",async (ctx,next)=>{
//     ctx.body = "这是Hello2";
//     await next();
// });


// **************************************错误处理中间件**************************************
// 如果页面找不到,则:404
// curl http://localhost:8080/xxx
app.use(async (ctx,next) => {
    console.log(`request url:${ctx.request.url}`)
    await next();
    if(ctx.status == 404){
        ctx.status = 404;
        ctx.body = "这是一个404错误";
    }
});


// 配置第三方中间件 
app.use(router.routes()).use(router.allowedMethods());
// 监听端口
app.listen(8080);
```
### (3). 运行
```
lixin-macbook:koa-example lixin$ node app.js 
```
### (4). 测试
```
// 404错误请求
lixin-macbook:~ lixin$ curl http://localhost:8080/xxx
这是一个404错误
```
### (5). 总结
> Koa里的中间件,和Java中的Filter一样.有请求之前或者请求之后,可进行一些处理.比如:权限拦截/请求日志记录.