---
layout: post
title: 'Kubernetes 部署Java Web 项目实践(四)'
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
[root@master ~]# kubectl apply -f hello-world-deployment.yml
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
# 通过export生成service配置文件.
# hello-world要与pod定义的hello-world相同.
# --port              : 容器内部通信端口
# --target-port       : 容器暴露的端口
# --type=NodePort     : NodePort随机生成端口
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
