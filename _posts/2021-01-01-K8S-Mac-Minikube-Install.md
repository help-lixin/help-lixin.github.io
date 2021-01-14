---
layout: post
title: 'Mac环境下通过Minikube安装Kubernetes'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). 安装 docker-machine-driver-hyperkit
```
lixin-macbook:~ lixin$  brew update
lixin-macbook:~ lixin$  brew install hyperkit
lixin-macbook:~ lixin$  brew install docker-machine-driver-hyperkit
lixin-macbook:~ lixin$ sudo chown root:wheel $(brew --prefix)/opt/docker-machine-driver-hyperkit/bin/docker-machine-driver-hyperkit
lixin-macbook:~ lixin$ sudo chmod u+s $(brew --prefix)/opt/docker-machine-driver-hyperkit/bin/docker-machine-driver-hyperkit
```

### (2). 安装 Minikube
> 在 https://github.com/kubernetes/minikube/releases 上下载最新的 minikube-darwin-amd64(我下载的是v1.16.0) 

```
lixin-macbook:~ lixin$  mv ~/Downloads/minikube-darwin-amd64 ~/Downloads/minikube
lixin-macbook:~ lixin$  mv ~/Downloads/minikube /usr/local/bin/
lixin-macbook:~ lixin$  chmod +x /usr/local/bin/minikube
lixin-macbook:~ lixin$  minikube version
	minikube version: v1.16.0
	commit: 9f1e482427589ff8451c4723b6ba53bb9742fbb1
```
### (3). 安装kubctl(需要翻墙)

```
lixin-macbook:~ lixin$ cd ~/Downloads
lixin-macbook:Downloads lixin$ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/darwin/amd64/kubectl
lixin-macbook:Downloads lixin$ chmod +x ./kubectl
lixin-macbook:Downloads lixin$ mv ./kubectl /usr/local/bin/
lixin-macbook:Downloads lixin$ kubectl --help
```
### (4). 启动minikube
```
# 启动minikube
# lixin-macbook:Downloads lixin$ minikube start --vm-driver=hyperkit
lixin-macbook:Downloads lixin$ minikube start --docker-env HTTPS_PROXY=${https://registry.docker-cn.com} --registry-mirror=https://registry-mirror.com --vm-driver=hyperkit
😄  Darwin 10.15.7 上的 minikube v1.16.0
✨  根据现有的配置文件使用 hyperkit 驱动程序
👍  Starting control plane node minikube in cluster minikube
🔄  Restarting existing hyperkit VM for "minikube" ...
❗  This VM is having trouble accessing https://k8s.gcr.io
💡  To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
🐳  正在 Docker 20.10.0 中准备 Kubernetes v1.20.0…
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass, dashboard
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

```

### (5). 使用kubectl
```
lixin-macbook:Downloads lixin$ kubectl get nodes
NAME       STATUS   ROLES                  AGE    VERSION
minikube   Ready    control-plane,master   161m   v1.20.0
```

### (6). 关闭minikube
```
lixin-macbook:Downloads lixin$ minikube stop
```