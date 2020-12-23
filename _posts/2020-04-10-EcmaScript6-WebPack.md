---
layout: post
title: 'EcmaScript6+WebPack'
date: 2020-04-10
author: 李新
tags: EcmaScript6
---

### (1). nodejs安装(略)

### (2). 创建项目目录
```
lixin-macbook:Desktop lixin$ pwd
/Users/lixin/Desktop
lixin-macbook:Desktop lixin$ mkdir test
```
### (3). 通过nodejs初始化项目
```
# 进入工作目录
lixin-macbook:Desktop lixin$ cd test
# 可以一路回车,最后输入:yes
lixin-macbook:test lixin$ npm init
  package name: (test) 
  version: (1.0.0) 
  description: 
  entry point: (index.js) 
  test command: 
  git repository: 
  keywords: 
  author: 
  license: (ISC) 
  About to write to /Users/lixin/Desktop/test/package.json:

  {
    "name": "test",
    "version": "1.0.0",
    "description": "",
    "main": "index.js",
    "scripts": {
      "test": "echo \"Error: no test specified\" && exit 1"
    },
    "author": "",
    "license": "ISC"
  }
  Is this OK? (yes)
```
### (4). 安装webpack和webpack-cli
```
# 安装webpack和webpack-cli
# 建议全局安装这两件插件(npm install webpack webpack-cli -g)  
lixin-macbook:test lixin$ npm install webpack webpack-cli --save-dev

# 查看项目结构
lixin-macbook:test lixin$ tree -L 1
.
├── node_modules
├── package-lock.json
└── package.json

# 查看webpack版本
lixin-macbook:test lixin$ ./node_modules/.bin/webpack -v
webpack-cli 4.2.0
webpack 5.11.0
```
### (5). 创建项目目录(src/dist/html)
```
lixin-macbook:test lixin$ mkdir src dist html
```
### (6). src用于存放JS文件
> src/Person.js   

```
export default class Person{
	constructor(name,age) {
	    this.name = name;
		this.age = age;
	}
	
	hello(){
		return `"{
			"name":"${this.name}",
			"age" : ${this.age}
		}"`
	}
}
```

> src/index.js 

```
lixin-macbook:test lixin$ cat src/index.js 
import Person from "./Person.js"

let person = new Person("zhangsan",26);
console.log(person.hello());
```

### (7). 编写html
> html/demo.html 

```
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8">
		<title>test</title>
    <!-- 加载js文件 -- >
		<script type="text/javascript" src="../dist/main.js"></script>
	</head>
	<body>
	</body>
</html>
```
### (8). 配置webpack.config.js
```
# 配置webpack.config.js
lixin-macbook:test lixin$ echo "" >  webpack.config.js
module.exports = {
    entry:  __dirname + '/src/index.js', //打包文件入口
    output: {               //打包文件出口
        path:  __dirname + '/dist',
        filename: '[name].js'
    }
}
```
### (9). webpack打包
```
lixin-macbook:test lixin$ ./node_modules/.bin/webpack -w
[webpack-cli] Compilation starting...
[webpack-cli] Compilation finished
asset main.js 203 bytes [compared for emit] [minimized] (name: main)
orphan modules 182 bytes [orphan] 1 module
./src/index.js + 1 modules 284 bytes [built] [code generated]
webpack 5.11.0 compiled successfully in 209 ms
[webpack-cli] watching files for updates...
```

### (10). 项目结构
```
lixin-macbook:test lixin$ tree -L 1
.
├── dist
├── html
├── node_modules
├── package-lock.json
├── package.json
├── src
└── webpack.config.js
```

### (11). package.json
```
{
  "name": "test",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "webpack": "^5.11.0",
    "webpack-cli": "^4.2.0"
  }
}
```

### (12). 访问
> 通过Chrome直接打开文件(file:///Users/lixin/Desktop/test/html/demo.html)
!["ES6+WebPack打包"](/assets/js/imgs/es6-webpack.jpg)

