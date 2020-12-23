---
layout: post
title: 'EcmaScript6 Promise'
date: 2020-04-10
author: 李新
tags: EcmaScript6
---

### (1). Promise作用
> 主要用来解决回调地狱问题.以及对异步请求进行编排. 

### (2). 没有Promise之前
```
function requestOne(callback){
  // 模似:延迟1秒后,再回调.
  setTimeout(()=>{
    callback({"action":"browser"});
  } , 1000)
}

function requestTwo(callback){
  // 模似:延迟1秒后,再回调.
  setTimeout(()=>{
    callback({"action":"cleanCookis"});
  } , 1000)
}

// 请求业务获取数据
requestOne((callbakc1)=>{
  let action = callbakc1.action;
  console.log("1. callback action: "+ action);
  
  // 请求业务获取数据.
  requestTwo((callbakc2)=>{
    let action = callbakc2.action;
    console.log("2. callback action: "+ action);
    // ... ...
  });
});
```
### (3). Promise简单使用
```
new Promise((resolve,reject)=>{
			setTimeout(()=>{
				resolve({"action" : "browser"});
			},1000);
		}).then(
			res => {
				let action = res.action;
				console.log("1. callback action: " + action);
				
				return new Promise((resolve,reject)=>{
					setTimeout(()=>{
						resolve({"action" : "cleanCookis"});
					},1000);
				})
			}
		).then(
		   res =>  {
			   let action = res.action;
			   console.log("2. callback action: " + action);
			   // TODO ...
		   }
		);
```
### (4). 添加全局异常处理
```
new Promise((resolve,reject)=>{
			setTimeout(()=>{
				resolve({"action" : "browser"});
			},1000);
		}).then(
			res => {
				let action = res.action;
				console.log("1. callback action: " + action);
				
				return new Promise((resolve,reject)=>{
					setTimeout(()=>{
						resolve({"action" : "cleanCookis"});
					},1000);
				});
			}
		).then(
		   res =>  {
			   let action = res.action;
			   console.log("2. callback action: " + action);
			   
			   // 添加异常
			   console.log(data);
			   
			   return new Promise((resolve,reject)=>{
			   	setTimeout(()=>{
			   		resolve({"action" : "cleanAll"});
			   	},1000);
			   })
		   }
		).then(
		  res=>{
			  let action = res.action;
			  console.log("3. callback action: " + action);
			  // TODO
		  }
		)
		.catch(ex=>{   // 全局异常处理.
			//catch exception:  ReferenceError: data is not defined
			console.error("catch exception: " + ex);
		});
```
### (5). 并行处理
```
let oneRequst = new Promise((resolve,reject)=>{
    setTimeout(()=>{
      resolve({"action" : "browser"});
    },1000);
  });
  
  
  let twoRequst = new Promise((resolve,reject)=>{
    setTimeout(()=>{
      resolve({"action" : "cleanCookis"});
    },1000);
  });
  
  let threeRequst = new Promise((resolve,reject)=>{
    setTimeout(()=>{
      resolve({"action" : "cleanAll"});
    },1000);
  });
  
  // 多个请求并行执行之后,会一次性把所有的结果给:then.
  Promise.all([oneRequst,twoRequst,threeRequst])
  .then(arr=>{
    let [one,two,three] = arr;
    // {action: "browser"}
    console.log(one.action);
    // {action: "cleanCookis"}
    console.log(two.action);
    // {action: "cleanAll"}
    console.log(three.action);
  });
```
### (6). Promise.race
> race多个请求并行执行,但,只获取最早返回的那个结果.

```
let oneRequst = new Promise((resolve,reject)=>{
  setTimeout(()=>{
    resolve({"action" : "browser"});
  },1100);
});


let twoRequst = new Promise((resolve,reject)=>{
  setTimeout(()=>{
    resolve({"action" : "cleanCookis"});
  },5000);
});

let threeRequst = new Promise((resolve,reject)=>{
  setTimeout(()=>{
    resolve({"action" : "cleanAll"});
  },1000);
});

// 多个请求并行执行,但是,只拿到最早执行的那个结果.
Promise.race([oneRequst,twoRequst,threeRequst])
.then(result=>{
  // {action: "cleanAll"}
  console.log(result);
});
```
### (7). Promise.async
> 当一个方法修饰为:async时,相当于这个方法是被Promise给包裹着的.  

```
// 修饰函数为:async,会返回一个:Promise对象.
async function oneRequest(){
  setTimeout(()=>{
    let command = {"action": "browser"};
    console.log("1. callback action: " + command.action);
    return command;
  },1000)
}

async function twoRequest(){
  setTimeout(()=>{
    let command = {"action": "cleanCookis"}
    console.log("2. callback action: " + command.action);
    return command;
  },2000)
}

async function threeRequest(){
  setTimeout(()=>{
    let command = {"action": "cleanAll"}
    console.log("3. callback action: " + command.action);
    return command;
  },3000)
}

// 编排任务.
oneRequest().then(twoRequest()).then(threeRequest());
```
### (8). await
> await需要等待方法结果才能继续往下走.

```
function demo(num){
  return new Promise((resolve,reject) => {
    setTimeout(()=>{
      resolve(2 * num)
    },2000);
  });
}
    
async function test(){
  // 等待demo方法执行完成.才能继续往下走
  let resoult1 = await demo(20);
  let resoult2 = await demo(21);
  let resoult3 = await demo(22);
  console.log(resoult1,resoult2,resoult3);
}

// 调用异步方法,在异步方法里,异步方法内部,会进行demo方法的同步.
test();
```
