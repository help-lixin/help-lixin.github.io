---
layout: post
title: 'Centos 7安装Minikube' 
date: 2023-02-08
author: 李新
tags:  Linux K8S 
---

### (1). 概述
最近在弄CI/CD这一块,想省钱,不想买两台Linux服务器搭K8S集群,所以,想把Minikube运用起来,在这里记录下:Centos 7环境下安装的过程.

### (2). 安装步骤
```
# 1. 查看操作系统版本信息
[root@lixin ~]# uname -a
	Linux lixin 3.10.0-1160.81.1.el7.x86_64 #1 SMP Fri Dec 16 17:29:43 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux

# 2. 禁用swap分区
[root@lixin ~]# swapoff -a

# 3. 安装依赖:conntrack
[root@lixin ~]# yum -y install conntrack

# 4. 下载安装依赖:containerd/cri-dockerd
[root@lixin ~]# wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.2-3.el7.x86_64.rpm
[root@lixin ~]# wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd-0.3.1-3.el7.x86_64.rpm
[root@lixin ~]# rpm -ivh containerd.io-1.2.2-3.el7.x86_64.rpm
[root@lixin ~]# rpm -ivh cri-dockerd-0.3.1-3.el7.x86_64.rpm
[root@lixin ~]# systemctl daemon-reload
[root@lixin ~]# systemctl start docker.service
[root@lixin ~]# systemctl enable cri-docker
[root@lixin ~]# systemctl status docker.service

# 5. 下载安装依赖crictl
[root@lixin ~]#  VERSION="v1.25.0"
[root@lixin ~]#  wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
[root@lixin ~]#  tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/bin
[root@lixin ~]#  rm -f crictl-$VERSION-linux-amd64.tar.gz

# 6. 安装minikube/kubectl/kubeadm/kubelet,并配置
[root@lixin ~]# curl -Lo minikube http://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64&& chmod +x minikube&&mv minikube /usr/local/bin/
[root@lixin ~]# curl -Lo kubectl http://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl&& chmod +x kubectl&& mv kubectl /usr/local/bin/&&ln -sf /usr/local/bin/kubectl /usr/bin/kubectl 
[root@lixin ~]# curl -Lo kubeadm http://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubeadm
[root@lixin ~]# curl -Lo kubelet http://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubelet

[root@lixin ~]# mkdir -p $HOME/.kube
[root@lixin ~]# touch $HOME/.kube/config

[root@lixin ~]# export MINIKUBE_WANTUPDATENOTIFICATION=false
[root@lixin ~]# export MINIKUBE_WANTREPORTERRORPROMPT=false
[root@lixin ~]# export MINIKUBE_HOME=$HOME
[root@lixin ~]# export CHANGE_MINIKUBE_NONE_USER=true
[root@lixin ~]# export KUBECONFIG=$HOME/.kube/config


# 7. 启动minikube
[root@lixin ~]# minikube start --vm-driver=none --image-mirror-country='cn' --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
* minikube v1.29.0 on Centos 7.9.2009 (amd64)
  - KUBECONFIG=/root/.kube/config
  - MINIKUBE_WANTREPORTERRORPROMPT=false
  - MINIKUBE_WANTUPDATENOTIFICATION=false
  - MINIKUBE_HOME=/root
* Using the none driver based on existing profile
* Starting control plane node minikube in cluster minikube
* Restarting existing none bare metal machine for "minikube" ...
* OS release is CentOS Linux 7 (Core)
E0224 12:19:45.317298    8088 docker.go:149] "Failed to enable" err=<
        sudo systemctl enable docker.socket: exit status 1
        stdout:

        stderr:
        Failed to execute operation: No such file or directory
 > service="docker.socket"
* Preparing Kubernetes v1.26.1 on Docker 18.06.1-ce ...
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Configuring bridge CNI (Container Networking Interface) ...
* Configuring local host environment ...
*
! The 'none' driver is designed for experts who need to integrate with an existing VM
* Most users should use the newer 'docker' driver instead, which does not require root!
* For more information, see: https://minikube.sigs.k8s.io/docs/reference/drivers/none/
*
  - Using image registry.cn-hangzhou.aliyuncs.com/google_containers/storage-provisioner:v5
* Verifying Kubernetes components...
* Enabled addons: default-storageclass, storage-provisioner
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```
### (3). 修改Docker容器端口信息
```
# 这一步不是必须的,是因为,我这台宿主机上,已经有docker容器占用了8443端口了,所以,才会启动不了.  
[root@lixin ~]# systemctl stop docker
[root@lixin ~]# cd /var/lib/docker/containers/16feb0b3564d3acff0daac0011bdff9a259d6546d946fe0c416ae5743ed8ffab/
[root@lixin 16feb0b3564d3acff0daac0011bdff9a259d6546d946fe0c416ae5743ed8ffab]# cat hostconfig.json
"443/tcp":[{"HostIp":"","HostPort":"9443"}
[root@lixin ~]# systemctl restart docker

# 443  : 为容器端口
# 9443 : 为宿主机端口
```

### (4). 查看用了哪些端口

```
[root@lixin ~]# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:10259         0.0.0.0:*               LISTEN      10783/kube-schedule
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      11025/kubelet
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      10927/etcd
tcp        0      0 103.215.125.86:2379     0.0.0.0:*               LISTEN      10927/etcd
tcp        0      0 103.215.125.86:2380     0.0.0.0:*               LISTEN      10927/etcd
tcp        0      0 127.0.0.1:2381          0.0.0.0:*               LISTEN      10927/etcd
tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN      10800/kube-controll
tcp6       0      0 :::40186                :::*                    LISTEN      8690/cri-dockerd
tcp6       0      0 :::8443                 :::*                    LISTEN      10752/kube-apiserve
tcp6       0      0 :::10249                :::*                    LISTEN      11365/kube-proxy
tcp6       0      0 :::10250                :::*                    LISTEN      11025/kubelet
tcp6       0      0 :::10256                :::*                    LISTEN      11365/kube-proxy
```
### (5). 测试

```
# 1. 创建工作目录
[root@lixin ~]# mkdir test
[root@lixin ~]# cd test

# 2. 创建pod
[root@lixin test]# cat nginx-pod.yml
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

# 3. 创建service暴露端口:9090
[root@lixin test]#  cat nginx-service.yml
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
 
 # 4. 应用配置
[root@lixin test]#  kubectl apply -f .


# 5. 查看所有的pod
[root@lixin test]# kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Pending   0          2m5s

# 6. 查看pod信息
# 我这里是因为磁盘空间不足引起的,所以,需要对磁盘扩容.
[root@lixin test]# kubectl describe pod nginx
Name:             nginx
Namespace:        default
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=nginx
Annotations:      <none>
Status:           Pending
IP:
IPs:              <none>
Containers:
  nginx:
    Image:        nginx:latest
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-6xl9g (ro)
Conditions:
  Type           Status
  PodScheduled   False
Volumes:
  kube-api-access-6xl9g:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  35s (x2 over 6m4s)  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling..


# ***********************************************************************
# 7. 卸载minikube 
# 我是要先在这台机器上做测试,然后,才会买新的机器专门安装这个. 
# # ***********************************************************************
[root@lixin test]# minikube delete
```
### (6). 总结
安装个Minikube比用Rancher安装更复杂,要不是因为省钱,我真的想放弃的心都有了. 
