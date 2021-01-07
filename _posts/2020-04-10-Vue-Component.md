---
layout: post
title: 'Vue 自定义组件(二)'
date: 2020-04-10
author: 李新
tags: Vue
---

### (1). 组件
> 组件是一组可以重复使用的模板.  

### (2). component.html
```
<html>
   <head>
     <title>hello</title>
     <meta charset="utf8"/>
     <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
   </head>
   <body>
      <div id="app">
        <div id="components-demo">
          <!-- 2. 遍历:blogs   -->
          <!-- 3. 给临时变量blog,传递给组件(<blog-post>).props里 -->
          <blog-post v-for="blog in blogs" v-bind:item="blog"></blog-post>
        </div>
      </div>
      <script>

        // 通过props向子组件传递数据
        Vue.component('blog-post', {
          // 4. item = blog 
          props: ['item'],  
          template: '<h4> {{item.id}} -- {{item.title}} </h4>'
        });

        var app = new Vue({
            el: '#components-demo',
            data: {
              // 1. 定义数据源
              blogs: [
                { id: 1, title: 'My journey with Vue' },
                { id: 2, title: 'Blogging with Vue' },
                { id: 3, title: 'Why Vue is so fun' }
              ]
          }
        });
      </script>      
   </body>
</html>
```