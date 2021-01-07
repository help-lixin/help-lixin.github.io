---
layout: post
title: 'Vue Router'
date: 2020-04-10
author: 李新
tags: Vue
---

### (1). Vue-Router是什么
> Vue Router是Vue.js官方提供的路由管理器.   

### (2). Vue-Router安装
```
lixin-macbook:first-vue lixin$ cnpm install vue-router  --save
```

### (3). 步骤
> 1. 修改:main.js,引入vue-router.   
> 2. 修改:main.js,声明使用vue-router.   
> 3. 创建一个组件(Content.vue)   
> 4. 创建路由文件(router/index.js).  
> 5. 修改:main.js,引入路由配置文件.<font color='red'>注意:import时指向router目录,即可,不需要指向router/index</font>   
> 6. 修改:main.js,配置Vue实例,使用:router信息.  
> 7. 修改:App.vue,增加<route-view>该组件用于当路由后页面承接的组件.   

### (4). src/main.js
```
import Vue from 'vue'
// 1. 引入vue-router
import VueRouter from 'vue-router'
import App from './App'

// 5. 引入路由配置信息(注意:不需要引用到:js文件,我就是在这里出了错)
import router from './router'

// 2. 使用vue-router
Vue.use(VueRouter);

Vue.config.productionTip = false

new Vue({
  el: '#app',
  router,
  components: { App },
  template: '<App/>'
})

```
### (5). components/Content.vue
```
<template>
    <div>
        {{message}}
    </div>
</template>

<script>
// 3. 定义组件
export default {
    name : "Content",
    data : function(){
        return { 
            message:"我是内容"
        }
    }
}
</script>
```
### (6). src/router/index.js
```
import Vue from 'vue'
import VueRouter from 'vue-router'
// 引入组件:Content
import Content from '../components/Content'

Vue.use(VueRouter);

// 4. 统一路由定义配置文件
export default new VueRouter({
    routes: [
        // 定义路由信息(path:路径   component:路由组件)
        { path: "/content", name: "Content", component: Content }
    ],
    mode : "history"
});
```
### (7). App.vue
```
<template>
  <div id="app">
    <img src="./assets/logo.png">
    <router-link to="/">首页</router-link>
    <router-link to="/content">内容页</router-link>
    <!-- 7.定义路由的组件 -->
    <router-view></router-view>
  </div>
</template>

<script>
export default {
  name: 'App',
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
```
### (8). 项目下载
["VueRouter项目"](/assets/vue/first-vue.zip)