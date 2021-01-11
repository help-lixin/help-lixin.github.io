---
layout: post
title: 'Docker DockerCompose'
date: 2021-01-11
author: 李新
tags: DockerCompose
---

### (1). Docker Compose是什么?
> [官网(https://docs.docker.com/compose/)](https://docs.docker.com/compose/)
> Compose是用于定义和运行多容器的编排工具.可通过YAML文件来配置应用程序.     
> 可以这样理解:容器之间存在着依赖,Compose定义容器与容器之间的依赖关系.   
> 更通俗的理解:docker-cpomse.sh会解析yml文件,转换成:docker run 命令.   

### (2). Docker Compose模板
> docker-compose.yml   

```
version: "3.0"
services:
  web: # 服务名称唯一
    image: nginx:laster   # 使用的镜像名称
    ports:
      - "8080:80"         # 宿主机端口与虚拟机端口进行映射
	volumes:
	  - ~/DockerWorkspace/nginx/html:/usr/share/nginx/html    #宿主机与虚拟机目录映射
	
```
### (3). docker-compose常用命令
```
$ docker-compose up
```
### (4). 
	
### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 
