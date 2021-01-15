---
layout: post
title: 'Kubernetes StatefulSet详解(十)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). Deployment
> 1. Deployment部署的应用是无状态的.  
> 2. Pod都是一样的.   
> 3. 启动没有顺序要求.    
> 4. 不需要考虑在哪个Node节点运行.  
> 5. 可以随意的伸缩和扩展.   
### (2). StatefulSet
> 1. StatefulSet部署的应用是有状态的.   
> 2. 它要求每个Pod是独立的,并且要保证Pod启动顺序(比如:数据库有master和slave,要先启动master才能启动slave).  
> 3. 每个Pod有唯一的网络标识符.     
> 4. 需要利用无头Service生成唯一标识(实际就是设置:ClusterIP:none).   
> 5. StatefulSet会为每一个应用生成一个domain,生成规则(主机名称.service名称.namespace.svc.cluster.local).

### (3). 部署StatefulSet
> cat pod-statefulset.yaml

```
apiVersion: v1
kind: Service
metadata:
 name: nginx
 labels:
   app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
spec:
  serviceName: nginx
  replicas: 3
  selector:
     matchLabels:
        app: nginx
  template:
     metadata:
        labels:
            app: nginx
     spec:
        containers:
        -  name: myapp
           image: nginx:latest
           ports:
           - containerPort: 80
```
### (4). 应用配置
```
[root@master ~]# kubectl apply -f pod-statefulset.yaml
service/nginx created
statefulset.apps/nginx-statefulset created
```

### (5). 查看pod和service信息
```
# 查看pod和service信息
# statefulset是身份(即:唯一标识的)
# 每个pod都会有一个唯一的主机名称,并会生成唯一的域名.
# 唯一域名生成规则:
# 主机名称.service名称.namespace.svc.cluster.local
# nginx-statefulset-0.nginx.default.svc.cluster.local
[root@master ~]# kubectl get pods,svc
NAME                               READY   STATUS    RESTARTS   AGE
pod/hello-world-5f8d77c9f7-589kd   1/1     Running   7          2d
pod/hello-world-5f8d77c9f7-hg5fx   1/1     Running   8          2d
pod/hello-world-5f8d77c9f7-sndnh   1/1     Running   1          149m
pod/nginx-statefulset-0            1/1     Running   1          73m
pod/nginx-statefulset-1            1/1     Running   0          67m
pod/nginx-statefulset-2            1/1     Running   0          45m

NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/hello-world   NodePort    10.1.232.6   <none>        9090:31355/TCP   3d2h
service/kubernetes    ClusterIP   10.1.0.1     <none>        443/TCP          6d3h
service/nginx         ClusterIP   None         <none>        80/TCP           73m
```

### (6). 测试访问域名
```
# 进入某个容器内部
[root@master ~]# kubectl exec -it hello-world-5f8d77c9f7-589kd /bin/bash
# 测试domain
# [root@hello-world-5f8d77c9f7-589kd /]# ping nginx-statefulset-0.nginx
[root@hello-world-5f8d77c9f7-589kd /]# ping nginx-statefulset-0.nginx.default.svc.cluster.local
PING nginx-statefulset-0.nginx.default.svc.cluster.local (10.244.1.64) 56(84) bytes of data.
64 bytes from nginx-statefulset-0.nginx.default.svc.cluster.local (10.244.1.64): icmp_seq=3 ttl=62 time=0.430 ms

# [root@hello-world-5f8d77c9f7-589kd /]# ping nginx-statefulset-1.nginx
[root@hello-world-5f8d77c9f7-589kd /]# ping nginx-statefulset-1.nginx.default.svc.cluster.local
PING nginx-statefulset-1.nginx.default.svc.cluster.local (10.244.2.69) 56(84) bytes of data.
64 bytes from nginx-statefulset-1.nginx.default.svc.cluster.local (10.244.2.69): icmp_seq=1 ttl=64 time=0.148 ms

# [root@hello-world-5f8d77c9f7-589kd /]# ping nginx-statefulset-2.nginx
[root@hello-world-5f8d77c9f7-589kd /]# ping nginx-statefulset-2.nginx.default.svc.cluster.local
PING nginx-statefulset-2.nginx.default.svc.cluster.local (10.244.2.72) 56(84) bytes of data.
64 bytes from nginx-statefulset-2.nginx.default.svc.cluster.local (10.244.2.72): icmp_seq=1 ttl=64 time=0.350 ms

```
