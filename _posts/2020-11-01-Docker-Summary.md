---
layout: post
title: 'Docker 概念'
date: 2020-11-01
author: 李新
tags: Docker
---

### (1). Repositories
> 仓库是存储镜像(image)的地方.可以类比:Maven仓库一样,专门存储jar包的地方.

### (2). Images
> 镜像是容器(Container)的模板,可以理解为:自己买一台电脑,要装操作系统,需要买一个Win7的系统光盘.在我们那个年代,还是用光盘安装操作系统的.  
> 镜像的结构:     
> ${register_name}/${repository_name}/${image_name}:${tag_name}     
> 例如: docker.io/libary/alpine:3.10.1     

### (3). Container
> 容器是镜像生成后的实例,可以理解为:Win7系统光盘安装到机器后,我们实际使用的win7系统.

