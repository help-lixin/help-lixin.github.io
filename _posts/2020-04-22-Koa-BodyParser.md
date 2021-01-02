---
layout: post
title: 'Koa2 POST body解析(五)'
date: 2020-04-22
author: 李新
tags: Koa2
---

### (1). 安装模块(koa-bodyparser)
```
lixin-macbook:koa-example lixin$ cnpm install koa-views --save

lixin-macbook:koa-example lixin$ cnpm install ejs --save

lixin-macbook:koa-example lixin$ cnpm install koa-bodyparser  --save
```
### (2). 创建模板目录
```
lixin-macbook:koa-example lixin$ mkdir views
```
### (3). 创建ejs模板文件(views/login.ejs)
> views/login.ejs

```
<html>
	<head>
		<meta charset="utf-8" />
		<title>ejs hello world</title>
	</head>
	<body>
		<form action="/doLogin" method="post">
			用户名： <input type="text" name="username" />
			密码： <input type="password" name="password" />
			<button type="submit">提交</button>
		  </form>
	</body>
</html>
```
### (4). app.js
```
// 引入koa
const Koa = require("koa");
// 引入koa-router
const Router = require('koa-router');
// 引入koa-views
const views = require('koa-views');
// 引入 koa-bodyparser
const bodyParser = require('koa-bodyparser');


// 创建Koa实例
const app = new Koa();
// 创建Router实例
const router = new Router();

// 配置模板所在的路径,以及所使用的模板引擎
const render = views(__dirname + '/views', {extension:'ejs'} );

// 配置登录页面
router.get("/login",async (ctx,next) => {
    await ctx.render("login");
});
// 配置登录操作
router.post("/doLogin",async (ctx,next) => {
    // ctx.request.body 获得的就是POST数据(JSON格式)
    ctx.body = ctx.request.body;
});


// 配置第三方中间件 
app
   .use(bodyParser())  // 配置koa-bodyparser
   .use(render)  // 配置ejs中间件
   .use(router.routes())
   .use(router.allowedMethods());

// 监听端口
app.listen(8080);
```
### (5). 运行
```
lixin-macbook:koa-example lixin$ node app.js 
```
### (6). 测试
!["登录界面"](/assets/koa/imgs/koa-post-login.jpg)
!["登录后返回页面"](/assets/koa/imgs/koa-post-dologin.jpg)