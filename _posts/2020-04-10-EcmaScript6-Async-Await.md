---
layout: post
title: 'EcmaScript6 Async Await'
date: 2020-04-10
author: 李新
tags: EcmaScript6
---

### (1). Async
>  Async用于申明一个function是异步的(让方法变成异步的,实际就是用:Promise包装函数结果).     
>  Await用于等待一个异步方法执行完成(等待异步方法执行完成).   
>  <font color='red'>注意:await必须在async方法中才可以使用,因为await本来身就会阻塞代码继续往下执行.</font>   

### (2). 
```
async function testAsync(){
    console.log("2");
    return "Hello World";
}

// *************************第一种获取数据的方式*************************
// let result = testAsync();
// 不使用Await情况下,和Promise使用方法一样
// result.then((data)=>{
    // Hello World
    // console.log(data);
// });

// ***************************错误使用方式***************************
// let result = await testAsync();
// console.log(result);
// ***************************错误使用方式***************************

// *************************第二种获取数据的方式*************************
// 正确使用方法
async function call(){
    console.log("1");
    // await必须要用在async方法内部
    let result = await testAsync();
    // Hello World
    console.log(result);
    console.log("3");
}

// 调用call
call();
```
### (3). 执行结果
```
1
2
Hello World
3
```
### (3). 总结
> 稍微注意:await必须要在async function里面使用.  