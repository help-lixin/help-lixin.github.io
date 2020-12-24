---
layout: post
title: 'EcmaScript6 + Axios + WebPack + 跨域'
date: 2020-04-10
author: 李新
tags: EcmaScript6 Axios WebPack
---

### (1). 定义工作目录(test-axios)
```
# 工作目录
lixin-macbook:Desktop lixin$ mkdir test-axios
lixin-macbook:Desktop lixin$ cd test-axios
```
### (2). npm初始化项目
```
# nnpm 初始化项目
lixin-macbook:test-axios lixin$ npm init -y
```
### (3). 安装webpack/webpack-cli/webpack-dev-server(全局安装)
> <font color='red'>注意:在webpack/webpack-dev-server/webpack-cli这几个版本之间的关系一定要对应上.我就因为这个处理了半天.</font>

```
# ******************************************************
# 在webpack5的情况下,此时最新的webpack-dev-server还是4,有版本冲突.
# ******************************************************
# 全局安装webpack/webpack-cli/webpack-dev-server

lixin-macbook:test lixin$ npm i webpack@4.43.0 webpack-cli@3.3.11 webpack-dev-server@3.11.0  -g
```
### (4). 安装axios
```
# 安装axios
lixin-macbook:test-axios lixin$ npm install axios --save
```
### (5). 创建源码目录和打包目录
```
lixin-macbook:test-axios lixin$ mkdir src dist
```
### (6). 创建index.html
> dist/index.html

```
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>test</title>
        <script type="text/javascript" src="main.js"></script>
    </head>
    <body>
    </body>
</html>
```
### (7). 创建API请求(HelloWorld.js)
> src/api/HelloWorld.js

```
const axios = require('axios');

export default class HelloWorld {
	// 定义hello方法
	hello(url,callback) {
		axios.get(url)
		.then(function(response) {
			callback(response);
		})
		.catch(function(error) {
			console.log(error);
		});
	}
}
```
### (8). 主程序(main.js)
> src/main.js

```
import HelloWorld from "./api/HelloWorld.js"

// 注意:此处的url,不需要写全:http://localhost:8000/devApi/hello
// webpack.config.js配置: proxy.target = http://localhost:8080
// webpack.config.js配置: proxy.pathRewrite = {'^/devApi':''}
// 1. webpack-dev-server拦截:/devApi/hello 
// 2. 执行正则(proxy.pathRewrite),将:/devApi/hello 替换成: /hello
// 3. 把/hello拼接在:proxy.target后面,变成:http://localhost:8080/hello

let url = "/devApi/hello";
let helloWorld = new HelloWorld();	
// 回调
let result = helloWorld.hello( url , (res) => {
	if(res.status == 200){
		console.log(res.data);
	}
});

```
### (9). webpack.config.js
> webpack配置跨域代理和JS处理.   
> 注意:开发在写URL时,不要写前缀.
> 1. 针对URL(/devApi/hello)会进行替换(/hello).   
> 2. 给替换的URL(/hello),添加前缀(http://localhost:8080),最后变成:http://localhost:8080/hello    

```
module.exports = {
    entry:  __dirname + '/src/main.js', //打包文件入口
    output: {               //打包文件出口
        path:  __dirname + '/dist',
        filename: '[name].js'
    },
	devServer: {
		contentBase : "./dist",
		host : "localhost",
		port: 8000,
		open:true,
		proxy: {
			"/devApi" : {
				target:'http://localhost:8080',
				pathRewrite:{
					'^/devApi':''
				},
				changeOrigin:true,
				secure : false
			}
		}
	}
}
```
### (10). package.json
```
{
  "name": "test-axios",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
	  "clean" : "rm -rf /Users/lixin/Desktop/test-axios/dist/main.js",
    "dev": "webpack --mode development",
    "build": "webpack --mode production",
    "start": "webpack-dev-server --mode development"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {},
  "dependencies": {
    "axios": "^0.21.1"
  }
}
```
### (11). 项目结构如下
lixin-macbook:Desktop lixin$ tree test-axios 

test-axios
├── dist
│   └── index.html
├── package-lock.json
├── package.json
├── src
│   ├── api
│   │   └── HelloWorld.js
│   └── main.js
└── webpack.config.js

### (12). 编译/打包/运行
```
# 清除打包文件.
lixin-macbook:test-axios lixin$ npm run clean

# 重新编译JS.
lixin-macbook:test-axios lixin$ npm run build

# 运行,并会自动打开浏览器.
lixin-macbook:test-axios lixin$ npm run start
```

### (13). 项目文件
["test-axios-project"](/assets/js/test-axios.zip)