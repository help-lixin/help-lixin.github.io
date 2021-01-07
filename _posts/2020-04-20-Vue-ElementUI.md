---
layout: post
title: 'Vue + ElementUI + Axios'
date: 2020-04-20
author: 李新
tags: Vue ElementUI
---

### (1). 官网
> ["ElementUI"](https://element.eleme.cn/)   

### (2). 创建项目
```
lixin-macbook:Desktop lixin$ vue-init webpack vue-elementui

? Project name vue-elementui
? Project description A Vue.js project
? Author xxxx@126.com
? Vue build standalone
? Install vue-router? No   //自己安装路由
? Use ESLint to lint your code? No
? Set up unit tests No
? Setup e2e tests with Nightwatch? No
? Should we run `npm install` for you after the project has been created? (recom
mended) npm
```
### (3). 安装模块
```
# 安装axios
lixin-macbook:vue-elementui lixin$ cnpm install axios  --save

# vue-router
lixin-macbook:vue-elementui lixin$ cnpm install vue-router --save

# element-ui
lixin-macbook:vue-elementui lixin$ cnpm install element-ui  --save

# saas加载器(最新版本8.X好像有问题)
lixin-macbook:vue-elementui lixin$ npm install sass-loader@7.3.1 node-sass@4.14.1  --save-dev
```

### (4). axios与Vue整合
> main.js

```
// 导入axios
import axios from 'axios'

// 声明使用axios
Vue.prototype.axios=axios;
```

> config/index.js(配置反向代理)

```
proxyTable: {
  "/devApi" : {
		target:'http://localhost:9090',
		pathRewrite:{
			'^/devApi':''
		},
		changeOrigin:true,
		secure : false
	}
}
```

> UserModify.vue

```
getData:function(id){
	let url = "http://localhost:8080/devApi/user/" + id ;
	this.axios({
		method : "GET",
		url : url
	}).then((res)=>{
		if(res.status == 200){
			this.from = res.data;
		}
	}).catch((err)=>{
		console.log(err);
	});
}
```



### (5). ElementUI与Vue整合
> main.js

```
import ElementUI from 'element-ui';
import 'element-ui/lib/theme-chalk/index.css';

Vue.use(ElementUI);

new Vue({
  el: '#app',
  router,
  // 改成:render: h => h(App)
  render: h => h(App)
})
```

> App.vue
```
<template>
  <div id="app">
    <!-- 配置路由组件 -->
    <router-view/>
  </div>
</template>

<script>
export default {
  name: 'App',
}
</script>

<style>

</style>
```

> Login.vue
```
<template>
    <div>
        <!-- ref:相当于ID   model:v-bind:model="from"   -->
        <el-form ref="form" :model="form" :rules="rules"  class="login-box">
            <h3 class="login-title">欢迎登录</h3>
            <el-form-item label="用户名:" prop="userName">
                <el-input placeholder="请输入用户名" v-model="form.userName"></el-input>
            </el-form-item>
            <el-form-item label="密码:"  prop="password">
                <el-input placeholder="请输入密码" v-model="form.password" show-password>></el-input>
            </el-form-item>
            <el-form-item>
                <el-button type="primary" @click="onSubmit('form');">登录</el-button>
                <el-button @click="onCancel('form');">取消</el-button>
            </el-form-item>
        </el-form>
    </div>    
</template>

<script>
export default {
    name : "Login",
    data() {
      return {
          form: {
            userName : '',
            password : '',
        },
        rules: {
            // 这里的名称要与prop定义要相同
            userName: [
                { required: true, message: '请输入用户名', trigger: 'blur' }
            ],
            password: [
                { required: true, message: '请输入密码', trigger: 'blur' }
            ]
        }
      }
    },
    methods : {
        onSubmit(formName){
            this.$refs[formName].validate((valid) => {
            if (valid) {
                // 跳转到首页:/main
                // this.$router.push("/main");
                // 跳转到首页,并传递参数:name:Main是router/index.js中指定的Name.
                this.$router.push({name:'Main',params:{ userName : this.form.userName } });
            } else {
                // 发送警告消息.
                this.$message({message: '请输入用户名或者密码',type: 'warning'});
                return false;
            }
            });
        },
        onCancel(formName){
            this.$refs[formName].resetFields();
        }
    }
}
</script>

<style lang="scss" scoped>
.login-box {
    width: 350px;
    margin: 120px auto;
    border: 1px solid #DCDFE6;
    padding: 20px;
    border-radius: 5px;
    box-shadow: 0 0 30px #DCDFE6;
}

.login-title {
    text-align: center;
}
</style>
```


> Main.vue
```
<template>
    <div>
         <el-container>

            <!-- <el-aside>：侧边栏容器开始  -->
            <el-aside width="200px">
                <el-menu :default-openeds="['1']">
                    <el-submenu index="1">
                        <template slot="title"><i class="el-icon-setting"></i>用户管理</template>
                        <el-menu-item-group>
                            <el-menu-item index="1-1">
                                <router-link to="/user/add">添加用户</router-link>
                            </el-menu-item>
                            <el-menu-item index="1-2">
                                <!-- <router-link to="/user/modify/1">修改用户</router-link> -->
                                <router-link :to="{name:'UserModify',params:{id:2}}">修改用户</router-link>
                            </el-menu-item>
                            <el-menu-item index="1-4">
                                <router-link to="/user/list">查看用户列表</router-link>
                            </el-menu-item>
                        </el-menu-item-group>
                    </el-submenu>

                    <el-submenu index="2">
                        <template slot="title"><i class="el-icon-menu"></i>角色管理</template>
                        <el-menu-item-group>
                            <el-menu-item index="1-1">添加角色</el-menu-item>
                            <el-menu-item index="1-2">修改角色</el-menu-item>
                            <el-menu-item index="1-3">删除角色</el-menu-item>
                            <el-menu-item index="1-4">查看角色列表</el-menu-item>
                        </el-menu-item-group>
                    </el-submenu>
                </el-menu>
            </el-aside>
            <!-- <el-aside>：侧边栏容器结束  -->
            
            <!-- 创建新的容器开始 -->
            <el-container>
                <!-- <el-header>:顶栏容器开始 -->
                <el-header style="text-align: right; font-size: 12px">
                    <el-dropdown>
                        <i class="el-icon-setting" style="margin-right: 15px"></i>
                        <el-dropdown-menu slot="dropdown">
                        <el-dropdown-item>用户中心</el-dropdown-item>
                        <el-dropdown-item>
                            <router-link to="/logout">退出登录</router-link>
                        </el-dropdown-item>
                        </el-dropdown-menu>
                    </el-dropdown>
                    <span>{{userName}}</span>
                </el-header>
                <!-- <el-header>:顶栏容器结束 -->
                
                <!-- <el-main>：主要区域容器开始 -->
                <el-main>
                    <router-view/>
                </el-main>
                <!-- <el-main>：主要区域容器结束 -->

            </el-container>
            <!-- 创建新的容器结束 -->

        </el-container>
    </div>   
</template>

<script>
export default {
    props : ['userName'] ,
    name : "Main",
    data() {
        return {
            
        }// end return 
    }
}
</script>

<style lang="scss" scoped>
  .el-header {
    background-color: #B3C0D1;
    color: #333;
    line-height: 30px;
  }
  
  .el-aside {
    color: #333;
  }
</style>
```

### (6). 项目下载
["Vue+ElementUI+Axios案例"](/assets/vue/vue-elementui.zip)