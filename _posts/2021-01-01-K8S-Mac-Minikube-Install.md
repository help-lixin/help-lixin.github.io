---
layout: post
title: 'Mac Minikube安装Kubernetes与使用'
date: 2021-01-01
author: 李新
tags: K8S Minikube
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
> minikube实际还是个虚拟机,让k8s在这个虚拟机里工作(有时下载镜像慢之类,需要自己手工处理),以下是进入虚拟机的方法.

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

### (8). 本地(Mac)环境下的Docker客户端(DockerDaemo)指向Minikube(虚拟机)中的Docker
> 通过这种方式,可以不用进到虚拟机,拷贝镜像之类的,直接可以操作docker,直方便.

```
# 未指向之前,本地有centos镜像
lixin-macbook:eureka-server lixin$ docker images
REPOSITORY                  TAG                                              IMAGE ID       CREATED        SIZE
lixinhelp/centos            centos7                                          e698e1ac21bc   11 days ago    298MB

# 本地(Mac)Docker客户端指向Minikube(虚拟机)中的Docker
lixin-macbook:~ lixin$ eval $(minikube -p minikube docker-env)

# 查询出来的镜像是:Minikube(虚拟机)中的Docker
# 此时就可以运用:docker build . -t 打包了,或者docker load本地磁盘文件之类的.
lixin-macbook:~ lixin$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED         SIZE
kubernetesui/dashboard                    v2.1.0     9a07b5b4bfac   5 weeks ago     226MB
k8s.gcr.io/kube-proxy                     v1.20.0    10cc881966cf   5 weeks ago     118MB
... ...
```

### (9). 访问Service(NodePort)暴露的服务

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

### (10). 通过Ingress暴露的服务
> 我这里(minikube addons enable ingress)一直都有问题,查看日志是pull镜像失败,我根据错误信息,在Docker Hub下载:    
> docker pull siriuszg/nginx-ingress-controller:v0.40.2     
> 重新打标签即可: us.gcr.io/k8s-artifacts-prod/ingress-nginx/controller:v0.40.2     
> 直到下面这个提示,代表ingress插件安装成功.   

```
# 添加扩展ingress
lixin-macbook:~ lixin$ minikube addons enable ingress
🔎  Verifying ingress addon...
🌟  启动 'ingress' 插件

# 默认是在:kube-system命名空间下的
lixin-macbook:~ lixin$ kubectl get pods  -n  kube-system
NAME                                        READY   STATUS      RESTARTS   AGE
coredns-74ff55c5b-b2d9k                     1/1     Running     3          5h17m
etcd-minikube                               1/1     Running     3          5h17m
ingress-nginx-admission-create-svtxw        0/1     Completed   0          162m
ingress-nginx-controller-558664778f-wdrkx   1/1     Running     0          162m
kube-apiserver-minikube                     1/1     Running     3          5h17m
kube-controller-manager-minikube            1/1     Running     3          5h17m
kube-proxy-74bhd                            1/1     Running     3          5h17m
kube-scheduler-minikube                     1/1     Running     3          5h17m
storage-provisioner                         1/1     Running     6          5h17m

# 创建ingress
lixin-macbook:test lixin$ cat nginx-ingress.yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web
spec:
  rules:
  - host: "nginx.hello.world"
    http:
      paths:
        - backend:
            serviceName: nginx-service
            servicePort: 9090

# 发部nginx-ingress.yml配置
lixin-macbook:test lixin$ kubectl apply -f nginx-ingress.yml
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
ingress.extensions/web created

# 查看ingress是否创建成功
lixin-macbook:test lixin$ kubectl get ingress
NAME   CLASS    HOSTS               ADDRESS        PORTS   AGE
web    <none>   nginx.hello.world   192.168.64.3   80      46s

# 查看ingress详细信息
lixin-macbook:test lixin$ kubectl describe ingress web
Name:             web
Namespace:        default
Address:          192.168.64.3
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host               Path                  Backends
  ----               ----                 --------
  nginx.hello.world  nginx-service:9090   172.17.0.3:80)

  
# 查看虚拟机(minikube)IP
lixin-macbook:test lixin$ minikube ip
192.168.64.3
  
# 在Mac机器上配置(nginx.hello.world域名与192.168.64.3的关系)
lixin-macbook:~ lixin$ cat /etc/hosts|grep nginx.hello.world
192.168.64.3 nginx.hello.world

# 测试访问
lixin-macbook:test lixin$ curl   http://nginx.hello.world
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
... ...
```