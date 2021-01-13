---
layout: post
title: 'Kubernetes Deployment模板样式(四)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). 这个模板内容摘抄于网络
!["https://www.cnblogs.com/reaperhero/articles/10804850.html"](https://www.cnblogs.com/reaperhero/articles/10804850.html)

### (2). Deployment 模板
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: detault
spec:
  replicas: 3 
  selector:
    matchLabels:
      project: dev
      app: nginx
  template:
    metadata:
      labels:
        project: dev
        app: nginx
    spec:
      imagePullSecrets: # 拉取镜像时使用的秘钥信息,BASE64
      - name: "xxxxx" # 例如(admin:Harbor12345),最后(YWRtaW46SGFyYm9yMTIzNDU=)
      containers:

      - name: nginx
        image: nginx:1.14.2
        imagePullPolicy: "Always"

        args:
        - /bin/sh
        - -c
        - sleep 60;nginx -s quit

        ports:
        - protocol: TCP
          containerPort: 80 # 定义pod对外开放的服务端口号,容器要监听的端口.

        env: #定义容器变量列表
        - name: JAVA_OPTS
          value: "-Xmx1G"

        resources: # 资源限制
          requests: # 最低启动限制设置
            cpu: 0.5 # 最低容器启动可用CPU核数
            memory: 256Mi # 最低容器启动可用内存数 单位MiB、GiB
          limits:
            cpu: 1 # 容器启动后最多可用CPU核数
            memory: 1Gi # 容器启动最多可用内存数 单位MiB、GiB

        readinessProbe: # 只有Pod中的容器都处于就绪状态,service才会将"请求转发给该容器处理"(exec/httpGet/tcpSocket任选其一)
          exec: #检测类型(exec)
            command: "" #命令或脚本

          httpGet: #检测类型(http)
            path: ""
            port: ""
            host: ""
            scheme: ""
            httpHeaders:
            - name: ""
              value: ""

          tcpSocket:
           port: 8010

          initialDelaySeconds: 60 # 表示在容器启动后进行第一次检查的等待时间(默认是0秒)
          periodSeconds: 10       # 表示每隔多长时间进行检查(默认是30秒)
          successThreshold: 1     # 表示几次检查通过才算成功(默认是1次)
          failureThreshold: 5     # 表示几次检查失败才算失败(默认是3次),失败后会重启容器.
          timeoutSeconds: 5       # 检查的超时时间(默认是1秒),当时我们用的就是默认值,而容器中的Java应用第一次请求时预热时间比较长,使用默认值很容易造成检查超时,现在改为5秒.

        livenessProbe: # 检查到Pod内容器,无响应之后会将会重新创建该容器(exec/httpGet/tcpSocket任选其一)
          exec: #检测类型(exec)
            command: "" #命令或脚本

          httpGet: #检测类型(http)
            path: ""
            port: ""
            host: ""
            scheme: ""
            httpHeaders:
            - name: ""
              value: ""

          tcpSocket:
            port: 8010

          initialDelaySeconds: 60 # 表示在容器启动后进行第一次检查的等待时间(默认是0秒)
          periodSeconds: 10       # 表示每隔多长时间进行检查(默认是30秒)
          successThreshold: 1     # 表示几次检查通过才算成功(默认是1次)
          failureThreshold: 5     # 表示几次检查失败才算失败(默认是3次),失败后会重启容器.
          timeoutSeconds: 5       # 检查的超时时间(默认是1秒),当时我们用的就是默认值,而容器中的Java应用第一次请求时预热时间比较长,使用默认值很容易造成检查超时,现在改为5秒.
```
