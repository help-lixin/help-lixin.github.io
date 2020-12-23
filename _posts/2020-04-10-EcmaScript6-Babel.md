---
layout: post
title: 'EcmaScript6 Babel提前编译'
date: 2020-04-10
author: 李新
tags: EcmaScript6
---

### (1). nodeJS安装(略)

### (2). 安装babel
```
# 工作目录
lixin-macbook:ecma-script lixin$ pwd
/Users/lixin/Desktop/ecma-script

# 更改镜像源
lixin-macbook:ecma-script lixin$ npm config set registry https://registry.npm.taobao.org/

# 查看镜像源
lixin-macbook:ecma-script lixin$ npm config get registry
https://registry.npm.taobao.org/

# 全局安装babel
lixin-macbook:ecma-script lixin$ npm -g install babel-cli

# 查看babel
lixin-macbook:ecma-script lixin$ babel -V
6.26.0 (babel-core 6.26.3)
```

### (3). 创建项目

```
# npm搭建环境
lixin-macbook:ecma-script lixin$ npm init
    package name: (ecma-script) hello
    version: (1.0.0) 
    description: hello world
    entry point: (index.js) 
    test command: 
    git repository: 
    keywords: 
    author: lixin
    license: (ISC) 
    About to write to /Users/lixin/Desktop/ecma-script/package.json:

    {
    "name": "hello",
    "version": "1.0.0",
    "description": "hello world",
    "main": "index.js",
    "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1"
    },
    "author": "lixin",
    "license": "ISC"
    }
    Is this OK? (yes) yes


# 创建babel配置文件
lixin-macbook:ecma-script lixin$ vi  .babelrc
{
	"presets" : [ "es2015" , "stage-2" ],
	"plugins" : [ "transform-runtime" ]
}

# 为项目添加模块.
lixin-macbook:ecma-script lixin$ npm install babel-core babel-preset-es2015 babel-plugin-transform-runtime babel-preset-stage-2 --save-dev

# 创建:src和lib目录
lixin-macbook:ecma-script lixin$ mkdir src lib

# 配置:package.json(scripts部份).
# 用babel 将src目录里的文件转换到lib目录下.
# -w watch监听文件,实时编译输出.
lixin-macbook:ecma-script lixin$  vi package.json
{
  "name": "hello",
  "version": "1.0.0",
  "description": "hello world",
  "main": "index.js",
  "scripts": {
    "build": "babel src -w -d lib"
  },
  "author": "lixin",
  "license": "ISC",
  "devDependencies": {
    "babel-core": "^6.26.3",
    "babel-plugin-transform-runtime": "^6.23.0",
    "babel-preset-es2015": "^6.24.1",
    "babel-preset-stage-2": "^6.24.1"
  }
}

lixin-macbook:ecma-script lixin$ vi  ./src/index.js
    let a = 10;
    let b = 20;
    console.log(a+b);

# 运行npm 构建
lixin-macbook:ecma-script lixin$ npm run build

# babel会将src目录下的js文件转换.
lixin-macbook:ecma-script lixin$ ll ./lib/
-rw-r--r--  1 lixin  staff   59 12 22 23:37 index.js
```
### (4). hello.html
```
# 编辑hello.html
lixin-macbook:ecma-script lixin$ vi hello.html
<html>
	<header>
		<meta charset="utf-8"/>
		<title>hello world</title>
		<script src="lib/index.js"></script>
	</header>
	<body>
	</body>
</html>
```
### (5). 项目结构如下
```
lixin-macbook:Desktop lixin$ tree ecma-script -L 1
ecma-script
├── hello.html
├── lib
├── node_modules
├── package-lock.json
├── package.json
└── src
```