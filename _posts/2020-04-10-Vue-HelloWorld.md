---
layout: post
title: 'Vue Hello World'
date: 2020-04-10
author: 李新
tags: Vue
---

### (1). 步骤
> 1. 引入Vue.js    
> 2. 定义Template   
> 3. 模板和数据绑定   

### (2). index.html
```
<html>
   <head>
     <title>hello</title>
	 <!-- 1. 引入vue js -->
     <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
   </head>
   <body>
      <!-- 2. 定义模板 -->
      <div id="app">
          {{ message }}
      </div>
      <script>
	      // 3. 把模板和数据进行绑定
          var app = new Vue({
              el: '#app',
              data: {
                message: 'Hello Vue!'
              }
          });
      </script>      
   </body>
</html>
```
### (3). 测试
> 通过控制台改变数据,测试页面DOM是否会一起改变

!["Vue Hello World"](/assets/vue/imgs/vue-hello-world.jpg)


