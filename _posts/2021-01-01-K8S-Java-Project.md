---
layout: post
title: 'Kubernetes 部署Java Web 项目(八)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1).  部署应用步骤
> 1. 通过Dockerfile,创建镜像,并提交到仓库(建议自建私有仓库).  
> 2. 编写yaml文件,部署镜像到K8S中.   
> 3. 编写yaml(service)文件,暴露容器端口.   

### (2).  项目准备
["hello 项目下载"](/assets/k8s/hello-example.zip)
> 项目结构如下: 

```
hello-example
├── Dockerfile
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           ├── App.java
│   │   │           └── controller
│   │   │               └── HelloController.java
│   │   └── resources
│   │       ├── application.properties
│   │       └── logback-spring.xml
│   └── test
│       └── java
└── target
```

> 1. 本地需要安装有docker环境.   
> 2. 执行: mvn clean package dockerfile:build    
> 3. 推送镜像到仓库(我这里用的是docker公有仓库:docker pull lixinhelp/hello:2.0.0-SNAPSHOT)

### (3). 部署镜像到K8S中

```
# 通过kubectl创建yaml模板
[root@master ~]# kubectl create deployment hello-world --image=lixinhelp/hello:2.0.0-SNAPSHOT --dry-run -o yaml > hello-world-deployment.yml
```
> hello-world-deployment.yml

```
[root@master ~]# cat hello-world-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-world
  name: hello-world
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - image: lixinhelp/hello:2.0.0-SNAPSHOT
        name: hello
```

> 应用配置到K8S里

```
# --record会把部署的信息记录到:kubectl rollout history的CHANGE-CAUSE信息里
[root@master ~]# kubectl apply -f hello-world-deployment.yml --record
deployment.apps/hello-world created

# 查看pod启动信息
[root@master ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-6b6d6fd479-d7tp4   1/1     Running   1          94m
hello-world-6b6d6fd479-ksgbr   1/1     Running   1          94m
hello-world-6b6d6fd479-zn85h   1/1     Running   2          94m


# 查看某个应用是否启动成功
[root@master ~]# kubectl logs hello-world-6b6d6fd479-ksgbr

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.0.RELEASE)
2021-01-12 08:25:32.371  INFO 1 --- [           main] help.lixin.App                           : Starting App on hello-world-6b6d6fd479-ksgbr with PID 1 (/opt/service/app.jar started by root in /)
```

### (4). 暴露应用

```
# 查看deploy
[root@master ~]# kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
hello-world   2/2     2            2           30h

# 通过export生成service配置文件.
# hello-world要与上面deploy的名称相同
# --port              : 集群内部容器通信端口
# --target-port       : 应用程序(Dockerfile)暴露(EXPOSE)的端口
# --type=NodePort     : NodePort会创建CLUSTER-IP并随机生成端口
# -o                  : 生成YAML格式的文件
# --dry-run           : 尝试运行(并不是真正的运行)
[root@master ~]# kubectl expose deployment hello-world --port=9090 --target-port=9090  --type=NodePort -o yaml --dry-run > hello-world-expose.yml


# 查看配置文件
[root@master ~]# cat hello-world-expose.yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hello-world
  name: hello-world
spec:
  ports:
  - port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: hello-world
  type: NodePort
  
  
# 应用配置
[root@master ~]# kubectl apply -f hello-world-expose.yml
service/hello-world created 

# 查看pod和service
[root@master ~]# kubectl get pods,svc
NAME                               READY   STATUS    RESTARTS   AGE
pod/hello-world-6b6d6fd479-4fntx   1/1     Running   0          11m
pod/hello-world-6b6d6fd479-d7tp4   1/1     Running   1          108m
pod/hello-world-6b6d6fd479-ksgbr   1/1     Running   1          108m

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
                                                               # 31355就是NodePort随机生成的
service/hello-world   NodePort    10.1.232.6     <none>        9090:31355/TCP   37s


# 测试访问
lixin-macbook:~ lixin$ curl http://10.211.55.100:31355/hello
Hello World!!!test-hello
lixin-macbook:~ lixin$ curl http://10.211.55.101:31355/hello
Hello World!!!test-hello
lixin-macbook:~ lixin$ curl http://10.211.55.102:31355/hello
Hello World!!!test-hello
```

### (5). 滚动更新
> kubectl set image升级应用.

```
# 为了测试方便(网络慢),我先在两台机器上分别拉取镜像.
[root@node-1 ~]# docker pull lixinhelp/hello:3.0.0-SNAPSHOT
[root@node-2 ~]# docker pull lixinhelp/hello:3.0.0-SNAPSHOT

# 查看待升级的镜像
[root@node-1 ~]# docker images|grep 3.0.0
lixinhelp/hello                                      3.0.0-SNAPSHOT      35fffb48c583        27 hours ago        576MB
[root@node-2 ~]# docker images|grep 3.0.0
lixinhelp/hello                                      3.0.0-SNAPSHOT      35fffb48c583        27 hours ago        576MB


# 查看K8S集群有哪些deployment
[root@master ~]# kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
hello-world   3/3     3            3           26h


# 升级deploy中的镜像
# hello-world  : 为deployment名称
# hello        : 为容器中定义的镜像名称
[root@master ~]# kubectl set image deployment hello-world  hello=lixinhelp/hello:3.0.0-SNAPSHOT
deployment.extensions/hello-world image updated

# 查看启动的service
[root@master ~]# kubectl get service
NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
hello-world   NodePort    10.1.232.6   <none>        9090:31355/TCP   25h

# 测试访问查看是否升级成功
lixin-macbook:~ lixin$ curl  http://10.211.55.100:31355/hello
Hello World-3.0.0-SNAPSHOT!!!test-hello
lixin-macbook:~ lixin$ curl  http://10.211.55.101:31355/hello
Hello World-3.0.0-SNAPSHOT!!!test-hello
lixin-macbook:~ lixin$ curl  http://10.211.55.102:31355/hello
Hello World-3.0.0-SNAPSHOT!!!test-hello
```

> kubectl edit实时修改配置

```
# 可实时修改deployment/hello-world的配置信息,都不需要重新应用配置.
# [root@master ~]# kubectl edit deployment/hello-world
# 可实时修改service/hello-world的配置信息,都不需要重新应用配置.
[root@master ~]# kubectl edit service/hello-world
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"hello-world"},"name":"hello-world","namespace":"default"},"spec":{"ports":[{"port":9090,"protocol":"TCP","targetPort":9090}],"selector":{"app":"hello-world"},"type":"NodePort"}}
  creationTimestamp: "2021-01-12T08:39:39Z"
  labels:
    app: hello-world
  name: hello-world
  namespace: default
  resourceVersion: "35142"
  selfLink: /api/v1/namespaces/default/services/hello-world
  uid: 05854b40-4413-4faa-aeba-d1d7e2ce0381
spec:
  clusterIP: 10.1.232.6
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 31355
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: hello-world
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```


> kubectl rollout status查看升级状态

```
# 查看资源(deployment/hello-world)升级状态
[root@master ~]# kubectl rollout status deployment/hello-world
deployment "hello-world" successfully rolled out
```

### (6). 回滚
```
# 查看资源(deployment/hello-world),升级记录
[root@master ~]# kubectl rollout history deployment/hello-world
deployment.extensions/hello-world
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

# 回滚到上一个版本
# [root@master ~]# kubectl rollout undo deployment/hello-world

# 回滚到指定版本
[root@master ~]# kubectl rollout undo deployment/hello-world --to-revision=1
deployment.extensions/hello-world rolled back

# 测试是否回滚成功
lixin-macbook:~ lixin$ curl  http://10.211.55.100:31355/hello
Hello World!!!test-hello
lixin-macbook:~ lixin$ curl  http://10.211.55.101:31355/hello
Hello World!!!test-hello
lixin-macbook:~ lixin$ curl  http://10.211.55.102:31355/hello
Hello World!!!test-hello
```

### (7). 弹性伸缩Pod
```
# 查看当前的pod
[root@master ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-5f8d77c9f7-589kd   1/1     Running   0          40m
hello-world-5f8d77c9f7-hg5fx   1/1     Running   0          38m
hello-world-5f8d77c9f7-xthp7   1/1     Running   0          6m28s

# 对hello-world进行扩容
[root@master ~]# kubectl scale deployment hello-world  --replicas=5
deployment.extensions/hello-world scaled

# 再次查看pod,已经增加到5个
[root@master ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-5f8d77c9f7-2srd8   1/1     Running   0          33s
hello-world-5f8d77c9f7-589kd   1/1     Running   0          41m
hello-world-5f8d77c9f7-5n7gg   1/1     Running   0          33s
hello-world-5f8d77c9f7-hg5fx   1/1     Running   0          40m
hello-world-5f8d77c9f7-xthp7   1/1     Running   0          8m20s
```

### (8). 删除pod和deployment
> deployment能够保证你预期的副本数的Pod,所以,如果,删除的是pod,deployment会立即再启动新的Pod.

```
# 查看有哪些pod
[root@master ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-5f8d77c9f7-2xls6   1/1     Running   0          33m
hello-world-5f8d77c9f7-589kd   1/1     Running   0          33m
hello-world-5f8d77c9f7-hg5fx   1/1     Running   0          32m

# 删除某个pod
[root@master ~]# kubectl delete pod hello-world-5f8d77c9f7-2xls6
pod "hello-world-5f8d77c9f7-2xls6" deleted

# 发现pod又重新启动了
[root@master ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-5f8d77c9f7-589kd   1/1     Running   0          35m
hello-world-5f8d77c9f7-hg5fx   1/1     Running   0          34m
hello-world-5f8d77c9f7-xthp7   1/1     Running   0          103s

# 删除deployment(hello-world)
[root@master ~]# kubectl delete deployment hello-world
```