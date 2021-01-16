---
layout: post
title: 'Kubernetes StatefulSet 部署Eureka(十一)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). eureka项目下载
!["K8S Eureka集群项目"](/assets/k8s/eureka-server.zip)

### (2). application.yml
```
#logging:
  #config: classpath:logback-spring.xml

server:
  port: ${PORT:8761}
eureka:
  instance:
    hostname: ${EUREKA_INSTANCE_HOSTNAME:${spring.application.name}}
    appname: ${spring.application.name}
  client:
    registerWithEureka: ${BOOL_REGISTER:false}
    fetchRegistry: ${BOOL_FETCH:false}
    serviceUrl:
      defaultZone: ${EUREKA_SERVER:http://${eureka.instance.hostname}:${server.port}/eureka/}
  server:
    enable-self-preservation: ${SELF_PRESERVATION:false}
spring:
  application:
    name: eureka-server
```
### (3). Dockerfile
```
FROM lixinhelp/jdk:1.8
ARG JAR_FILE
ENV APP_PATH /opt/service
EXPOSE 8761
RUN mkdir -p /opt/service/bin && mkdir -p /opt/service/conf && mkdir -p /opt/service/logs
ADD ./target/${JAR_FILE} /opt/service/bin/app.jar
ADD ./src/main/resources/application.yml /opt/service/conf/application.yml
ADD ./src/main/resources/logback-spring.xml /opt/service/conf/logback-spring.xml
ENTRYPOINT [ "java" , "-jar" , "/opt/service/bin/app.jar" , "--spring.config.location=/opt/service/conf/application.yml" , "--logging.config=/opt/service/conf/logback-spring.xml" ]
```

### (4). eureka-pod-statefulset.yml

```
---
apiVersion: v1
kind: Service
metadata:
  name: eureka
  labels:
    app: eureka
spec:
  ports:
    - port: 8761
     # name: eureka
  clusterIP: None
  selector:
    app: eureka
  # type: NodePort
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka
spec:
  serviceName: eureka
  replicas: 3
  selector:
    matchLabels:
      app: eureka
  template:
    metadata:
      labels:
        app: eureka
    spec:
      containers:
      - name: eureka
        image: lixinhelp/eureka:1.1.0
        ports:
        - containerPort: 8761
        resources:
          limits:
            memory: 512Mi
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: EUREKA_INSTANCE_HOSTNAME
          value: ${POD_NAME}.eureka
        - name: EUREKA_SERVER
          value: "http://eureka-0.eureka:8761/eureka/,http://eureka-1.eureka:8761/eureka/,http://eureka-2.eureka:8761/eureka/"
```
### (5). 部署有状态应用
```
[root@master ~]# kubectl apply -f eureka-pod-statefulset.yml
service/eureka created
statefulset.apps/eureka created

# 查看pod和service
[root@master ~]# kubectl get pods,svc
NAME                               READY   STATUS    RESTARTS   AGE
pod/eureka-0                       1/1     Running   0          31s
pod/eureka-1                       1/1     Running   0          29s
pod/eureka-2                       1/1     Running   0          28s
pod/hello-world-5f8d77c9f7-sndnh   1/1     Running   9          9h

# 注意eurek配置了:ClusterIP:None
NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/eureka        ClusterIP   None         <none>        8761/TCP         31s
service/hello-world   NodePort    10.1.232.6   <none>        9090:31355/TCP   3d9h
service/kubernetes    ClusterIP   10.1.0.1     <none>        443/TCP          6d10h
```
### (6). 验证
> 无头Service,代表不给这个Service分配:CLUSTERIP,容器内部通过:域名进行通信.这样减少了一次网络设备请求.


```
# 连接到某个容器
[root@master ~]# kubectl exec -it hello-world-5f8d77c9f7-sndnh /bin/bash

# 通过域名去访问
[root@hello-world-5f8d77c9f7-sndnh /]# ping  eureka-0.eureka
PING eureka-0.eureka.default.svc.cluster.local (10.244.1.98) 56(84) bytes of data.
64 bytes from eureka-0.eureka.default.svc.cluster.local (10.244.1.98): icmp_seq=1 ttl=64 time=0.045 ms

# 通过域名去访问
[root@hello-world-5f8d77c9f7-sndnh /]# ping  eureka-1.eureka
PING eureka-1.eureka.default.svc.cluster.local (10.244.2.101) 56(84) bytes of data.
64 bytes from eureka-1.eureka.default.svc.cluster.local (10.244.2.101): icmp_seq=1 ttl=62 time=1.92 ms

# 通过域名去访问
[root@hello-world-5f8d77c9f7-sndnh /]# ping  eureka-2.eureka
PING eureka-2.eureka.default.svc.cluster.local (10.244.1.99) 56(84) bytes of data.
64 bytes from eureka-2.eureka.default.svc.cluster.local (10.244.1.99): icmp_seq=1 ttl=64 time=0.121 ms
```
