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
# lixin-macbook:Downloads lixin$ minikube start --docker-env HTTPS_PROXY=${https://registry.docker-cn.com} --registry-mirror=https://registry-mirror.com --vm-driver=hyperkit
# lixin-macbook:Downloads lixin$ minikube start --registry-mirror=https://registry.docker-cn.com
# lixin-macbook:Downloads lixin$ minikube start --vm-driver=hyperkit
lixin-macbook:Downloads lixin$ minikube start --registry-mirror=https://registry.docker-cn.com
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
### (7). 进入Node节点
> minikube实际还是一样,在虚拟机里工作(有时下载镜像慢之类,需要自己手工处理),以下是进入虚拟机的方法.

```
# minikube进入ssh(查看IP)
lixin-macbook:~ lixin$ minikube ssh
                         _             _
            _         _ ( )           ( )
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

# 查看IP
$ ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 26:57:60:ca:c0:b6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.64.2/24 brd 192.168.64.255 scope global dynamic eth0

# 查看用户名
$ whoami
docker

# 通过SSH进入(为什么这样?因为要通过SSH拷贝镜像进虚拟机里)
# 用户名和密码为:docker/tcuser
lixin-macbook:~ lixin$ ssh docker@192.168.64.2
ECDSA key fingerprint is SHA256:4BKtGOWjnZZ0dA0GyWnNMUDJg47jh/kobG1Mnv29hTM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.64.2' (ECDSA) to the list of known hosts.
docker@192.168.64.2's password:
```