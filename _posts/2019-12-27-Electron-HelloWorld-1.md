---
layout: post
title: 'Electron HelloWorld(1)'
date: 2019-12-27
author: 李新
tags: Electron
---

### (1). NodeJS安装(略)

### (2). Hello World
```
# 工作目录
lixin-macbook:WorkspaceJS lixin$ pwd
/Users/lixin/WorkspaceJS

lixin-macbook:WorkspaceJS lixin$ git clone https://github.com/electron/electron-quick-start.git
```
### (3). 安装依赖
> 自行安装taobao cnpm

```
lixin-macbook:WorkspaceJS lixin$ cd electron-quick-start/

# 全局安装electron
lixin-macbook:~ lixin$ cnpm install electron -g

# 安装依赖
lixin-macbook:electron-quick-start lixin$ cnpm install
```
### (4). 启动
```
lixin-macbook:electron-quick-start lixin$ npm start
```
### (5). 运行结果
!["Electron Hello World"](/assets/electron/imgs/electron-helloworld.jpg)