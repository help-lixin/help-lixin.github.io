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
# --cpus 2                :  CPU数量
# --memory 4096           : 内存大小4096M(4G)
lixin-macbook:Downloads lixin$ minikube start --vm-driver=hyperkit --registry-mirror=https://registry.docker-cn.com --cpus 2 --memory 4096
😄  Darwin 10.15.7 上的 minikube v1.16.0
✨  根据用户配置使用 hyperkit 驱动程序
👍  Starting control plane node minikube in cluster minikube
🔥  Creating hyperkit VM (CPUs=2, Memory=4096MB, Disk=20000MB) ...
❗  This VM is having trouble accessing https://k8s.gcr.io
💡  To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
🐳  正在 Docker 20.10.0 中准备 Kubernetes v1.20.0…
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
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

### (8). 本地(Mac)Docker客户端指向Minikube(虚拟机)中的Docker
```
# 未指只前
lixin-macbook:eureka-server lixin$ docker images
REPOSITORY                  TAG                                              IMAGE ID       CREATED        SIZE
lixinhelp/centos            centos7                                          e698e1ac21bc   11 days ago    298MB

# 本地(Mac)Docker客户端指向Minikube(虚拟机)中的Docker
lixin-macbook:~ lixin$ eval $(minikube -p minikube docker-env)

# 查询出来的镜像是:Minikube(虚拟机)中的Docker
# 此时就可以运用:docker build . -t 打包了.
lixin-macbook:~ lixin$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED         SIZE
kubernetesui/dashboard                    v2.1.0     9a07b5b4bfac   5 weeks ago     226MB
k8s.gcr.io/kube-proxy                     v1.20.0    10cc881966cf   5 weeks ago     118MB
k8s.gcr.io/kube-apiserver                 v1.20.0    ca9843d3b545   5 weeks ago     122MB
k8s.gcr.io/kube-scheduler                 v1.20.0    3138b6e3d471   5 weeks ago     46.4MB
k8s.gcr.io/kube-controller-manager        v1.20.0    b9fa1895dcaa   5 weeks ago     116MB
gcr.io/k8s-minikube/storage-provisioner   v4         85069258b98a   6 weeks ago     29.7MB
k8s.gcr.io/etcd                           3.4.13-0   0369cf4303ff   4 months ago    253MB
k8s.gcr.io/coredns                        1.7.0      bfe3a36ebd25   7 months ago    45.2MB
kubernetesui/metrics-scraper              v1.0.4     86262685d9ab   9 months ago    36.9MB
k8s.gcr.io/pause                          3.2        80d28bedfe5d   11 months ago   683kB
```

### (9). 访问Service(NodePort)


```
# 创建pod
lixin-macbook:test lixin$ cat nginx-pod.yml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  containers: # 指定多个镜像,代表这些镜像在一个Pod内.
  - image: nginx:latest
    name: nginx

# 创建service暴露端口:9090
lixin-macbook:test lixin$ cat nginx-service.yml
apiVersion: v1
kind: Service   # 定义一个Service
metadata:
  labels:
    app: nginx-service    # Service自己的元数据信息
  name: nginx-service
spec:
  ports:
  - port: 9090
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx      # 重点:Service通过该配置关联Pod的资源
  type: NodePort
 
 # 应用配置
lixin-macbook:test lixin$ kubectl apply -f .

# 查看minikube虚拟机的IP地址
lixin-macbook:test lixin$ minikube ip
192.168.64.3

# 查看service和pod信息
lixin-macbook:test lixin$ kubectl get pods,svc
NAME        READY   STATUS    RESTARTS   AGE
pod/nginx   1/1     Running   0          119m

# nginx-service端口为:31224
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP          148m
service/nginx-service   NodePort    10.96.111.210   <none>        9090:31224/TCP   6m12s

# 用浏览器打开:http://192.168.64.3:31224
lixin-macbook:test lixin$ minikube service  nginx-service
|-----------|---------------|-------------|---------------------------|
| NAMESPACE |     NAME      | TARGET PORT |            URL            |
|-----------|---------------|-------------|---------------------------|
| default   | nginx-service |        9090 | http://192.168.64.3:31224 |
|-----------|---------------|-------------|---------------------------|
🎉  正通过默认浏览器打开服务 default/nginx-service...
```
