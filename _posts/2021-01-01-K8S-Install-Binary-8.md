---
layout: post
title: 'Kubernetes 二进制安装之测试集群(八)'
date: 2021-01-01
author: 李新
tags: 二进制安装K8S
---

### (1). 对K8S集群进行测试
```
# 创建pod为nginx
[root@master ~]# kubectl create deployment nginx --image=nginx
pod/nginx created

# 查看运行的pod
[root@master ~]# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE            NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-ksgln   1/1     Running   0          19s   172.20.97.2   10.211.55.101   <none>           <none>

# 创建Service
[root@master ~]# kubectl expose deployment nginx --port=80 --type=NodePort
service/nginx exposed

# 查看pod和service的详细
[root@master ~]# kubectl get pods,svc -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP            NODE            NOMINATED NODE   READINESS GATES
pod/nginx-6799fc88d8-ksgln   1/1     Running   0          90s   172.20.97.2   10.211.55.101   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP        16h   <none>
service/nginx        NodePort    10.0.0.31    <none>        80:38677/TCP   45s   app=nginx
```
### (2). 在node节点上访问pod IP
```
[root@node-2 ~]# curl http://172.20.97.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
... ...
```
### (3).  在node节hok上访问service ip
```
[root@node-1 ~]# curl http://10.0.0.31:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```
### (4). 在node节点上访问(38677)
```
[root@node-2 ~]# curl http://10.211.55.101:38677
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
... ...


[root@node-2 ~]# curl http://10.211.55.102:38677
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
... ...
```

### (5). 为应用扩容测试
```
# 应用扩容
[root@master ~]# kubectl scale deployment nginx --replicas=3
deployment.apps/nginx scaled

# 查看详细信息
[root@master ~]# kubectl get pods,svc -o wide
NAME                         READY   STATUS              RESTARTS   AGE     IP            NODE            NOMINATED NODE   READINESS GATES
pod/nginx-6799fc88d8-8wmjm   1/1     Running             0          4m26s   172.20.97.3   10.211.55.101   <none>           <none>
pod/nginx-6799fc88d8-hr6nc   0/1     ContainerCreating   0          59s     <none>        10.211.55.102   <none>           <none>
pod/nginx-6799fc88d8-ksgln   1/1     Running             2          34m     172.20.97.2   10.211.55.101   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP        16h   <none>
service/nginx        NodePort    10.0.0.31    <none>        80:38677/TCP   33m   app=nginx

# 通过VIP,测试访问.
[root@node-2 ~]# curl http://10.0.0.31
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
... ...
```