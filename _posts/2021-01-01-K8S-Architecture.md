---
layout: post
title: 'Kubernetes 架构(一)'
date: 2020-12-27
author: 李新
tags: K8S
---

### (1). K8S架构图
> ["图片转载于"](https://www.kubernetes.org.cn/kubernetes%e8%ae%be%e8%ae%a1%e6%9e%b6%e6%9e%84)

!["K8S架构图"](/assets/k8s/imgs/k8s-architecture.png)

> ["图片转载于"](https://developer.51cto.com/art/201909/602646.htm)

!["K8S架构图"](/assets/k8s/imgs/k8s-architecture2.jpeg)

### (2). K8S核心组件(master)
> 1. etcd保存了整个集群的状态.   
> 2. apiserver提供了资源操作的<font color='red'>唯一入口</font>,并提供认证、授权、访问控制、API注册和发现等机制.   
> 3. controller manager负责维护集群的状态,比如故障检测、自动扩展、滚动更新等.   
> 4. scheduler负责资源的调度,按照预定的调度策略将Pod调度到相应的机器上.   

### (3). K8S核心组件(node)
> 1. kubelet负责维护Pod的生命周期,同时也负责Volume(CVI)和网络(CNI)的管理.     
> 2. kube-proxy负责为Service提供cluster内部的服务发现和负载均衡.    
> 3. K8S中最小的运行单位为Pod,在Pod内部可以维护多个容器(Container).   