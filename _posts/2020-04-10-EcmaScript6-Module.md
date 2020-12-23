---
layout: post
title: 'EcmaScript6 Module模块化'
date: 2020-04-10
author: 李新
tags: EcmaScript6
---

### (1). one.js
```
// 定义Class
class Person{
	constructor(name,age) {
	    this.name = name;
		this.age = age;
	}
	toString(){
		return `"{
			"name":"${this.name}",
			"age":${this.age}
			}"`
	}
}

// 定义URL
let url = "https://www.baidu.com";

// 使用默认导出,在导入的时候,名称可以随意填写.
export default {
	url : url,
	Person: Person
}
```
### (2). two.js
```
// 导入的名称需要与export出的名称相同
// 只因导出(export)使用提默认(default),所以,在这里导入的名称可以随意.
import one from  "./one.js"

// 创建实例
let zhagnsan = new one.Person("zhgnsan",25);

// 查看下one模块内容
console.log(one);
// 输出URL
console.log("url: "+one.url);
// 输出张三的信息.
console.log("toString: "+zhagnsan.toString());
```
### (3). module.html
```
<html>
	<header>
		<meta charset="utf-8"/>
		<title>promise</title>
		<script type="module" src="./src/two.js"></script>
	</header>
	<body>
	</body>
</html>
```
### (4). 运行
> <font color='red'>在运行时,需要注意:要把代码扔到WEB应用程序下运行.否则,会提示跨域问题.因为我用的是HBuilderX开发,它能帮我代理成WEB应用部署.</font>    
> http://127.0.0.1:8848/ecma-script/module.html