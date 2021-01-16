---
layout: post
title: 'Kubernetes ConfigMap(十三)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). ConfigMap作用
> ConfigMap是用于保存配置数据的键值对,可以用来保存单个属性,也可以保存配置文件.比如应用的配置信息,则可以使用ConfigMap来保存.  
> 当ConfigMap配置发生变化后,ConfigMap可以近实时的传播到应用的Pod内部

### (2). test-configmap.yml
```
lixin-macbook:k8s-test lixin$ cat test-configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
data:
  DATA_SOURCE_URL: jdbc:mysql://mysql-0/test
  DATA_SOURCE_USER_NAME: root
  DATA_SOURCE_USER_PWD: root
---
apiVersion: v1
kind: Pod
metadata:
  name: test-configmap-pod
spec:
  containers:
    - name: test-configmap
      image: busybox:latest
      command: [ "/bin/tail", "-f", "/etc/hostname" ]  #Hold住容器
      envFrom:
        - configMapRef:
            name: env-config  # 引用另外一个configmap的名称
```
### (3). 部署应用
```
lixin-macbook:k8s-test lixin$ kubectl apply -f test-configmap.yml
configmap/env-config created
pod/test-configmap-pod created
```
### (4). 验证环境变量是否传入到容器中
```
# 查看pod信息
lixin-macbook:k8s-test lixin$ kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
test-configmap-pod   1/1     Running   0          94s

# 查看日志
lixin-macbook:k8s-test lixin$ kubectl logs test-configmap-pod
HOSTNAME=test-configmap-pod
DATA_SOURCE_URL=jdbc:mysql://mysql-0/test
DATA_SOURCE_USER_PWD=root
DATA_SOURCE_USER_NAME=root
```
### (5). 查看configmap资源
```
# 查看configmap资源
# lixin-macbook:~ lixin$ kubectl get configmap
lixin-macbook:~ lixin$ kubectl get cm
NAME               DATA   AGE
env-config         3      13m
kube-root-ca.crt   1      8h

# 查看configmap详细
# lixin-macbook:~ lixin$ kubectl describe cm env-config
lixin-macbook:~ lixin$ kubectl describe configmap env-config
Name:         env-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
DATA_SOURCE_URL:
----
jdbc:mysql://mysql-0/test
DATA_SOURCE_USER_NAME:
----
root
DATA_SOURCE_USER_PWD:
----
root
Events:  <none>
```
