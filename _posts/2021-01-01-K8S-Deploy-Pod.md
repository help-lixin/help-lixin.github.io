---
layout: post
title: 'Kubernetes Pod部署多个容器以及通信原理(六)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). 创建pod配置(部署多个容器) 

```
[root@master ~]# cat pod.yml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: my-test-pod
  name: my-test-pod
spec:
  containers: # 指定多个镜像,代表这些镜像在一个Pod内.
  - image: nginx:latest
    name: nginx
  - image: lixinhelp/hello:2.0.0-SNAPSHOT
    name: java-hello
```

### (2). 应用配置到K8S

```
# 应用配置
[root@master ~]# kubectl apply -f pod.yml
pod/my-test-pod created

# 查看pod信息(READY为2代表有两个容器运行)
[root@master ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
my-test-pod                     2/2     Running   0          11s


# 进入容器内部(java-hello)
# -c 指定要进入哪个容器内部(名称在上面pod.yml中指定的:image.name)
[root@master ~]# kubectl exec -it my-test-pod -c java-hello  /bin/bash
# 查看容器内部运行的进程信息(这确实是一个JAVA进程)
[root@my-pod /]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  7 09:47 ?        00:00:10 java -jar /opt/service/app.jar

# 查看IP地址
[root@my-pod /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.2.23  netmask 255.255.255.0  broadcast 10.244.2.255
        ether 5a:71:21:46:07:95  txqueuelen 0  (Ethernet)


# 进入容器内部(nginx)
[root@master ~]# kubectl exec -it  my-test-pod -c nginx   /bin/bash

# 查看容器内部的IP地址
root@my-pod:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.2.23  netmask 255.255.255.0  broadcast 10.244.2.255
```


### (2). 共享磁盘空间
```
# 定义共享磁盘空间pod
[root@master ~]# cat container-share.yml
apiVersion: v1
kind: Pod
metadata:
  name: container-share
  namespace: default
spec:
  containers:
  - name: producer
    image: centos:7
    command: ["bash","-c","for i in {1..100};do echo $i >> /producer_dir/hello;sleep 1;done"]
    volumeMounts:
    - name: shared-volume  # 挂载数据卷名称
      mountPath: /producer_dir

  - name: consumer
    image: centos:7
    command: ["bash","-c","tail -f /consumer_dir/hello"]
    volumeMounts:
    - name: shared-volume  # 挂载数据卷名称
      mountPath: /consumer_dir

  volumes:
  - name: shared-volume # 创建数据卷名称
    emptyDir: {}


# 应用配置
[root@master ~]# kubectl apply -f container-share.yml
pod/container-share created


# 查看pods
[root@master ~]# kubectl get pods|grep container-share
container-share                2/2     Running   0          47s


# 查看消费者的日志信息
# container-share 为pod名称
# consumer : 为容器名称
[root@master ~]# kubectl logs -f  container-share -c consumer
80
81
... ...
```

### (4). 总结
> K8S在创建Pod时:    
> 1. 创建一个Infrastructure容器,用于维护整个Pod网络空间.    
> 2. 创建一个InitContainer容器.    
> 3. 将业务容器加入到:Infrastructure这个容器内(--net模式).      
> 4. 所以,它们可以共享网络命名空间.    