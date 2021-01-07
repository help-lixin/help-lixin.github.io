---
layout: post
title: 'Vue 计算属性(三)'
date: 2020-04-10
author: 李新
tags: Vue
---

### (1). 计算属性
> 当在模板中,对属性进行处理,包含了业务逻辑时,建议使用计算属性来代替模板中的逻辑.    
> 在Vue中使用:computed来定义:计算属性.     
> <font color='red'>注意:在模板使用时,不需要带括号.</font>  

### (2). computed.html
```
<html>
   <head>
     <title>hello</title>
     <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
   </head>
   <body>
      <div id="example">
        <!-- 方法调用是需要带括号 -->
        <p>当前时间:{{ getNow() }}</p>
        <!-- computed会把方法转变成属性,所以,调用时不需要带括号,带括号反而报错.  -->
        <p>当前时间属性:{{ getTime }}</p>
      </div>
      <script>
          var app = new Vue({
              el: '#example',
              computed:{
                getTime:function(){
                  return Date.now();
                }
              },
              methods : {
                getNow:function(){
                  return Date.now();
                }
              }
          });
      </script>      
   </body>
</html>
```