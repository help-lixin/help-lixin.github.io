---
layout: post
title: 'Kubernetes 微服务集成(解决方案)'
date: 2021-01-01
author: 李新
tags: K8S Spring SpringCloud
---

### (1). K8S与微服务集成
> 近来一直在用K8S,发现K8S有服务发现功能(Service),而Spring Cloud也有相应的服务发现功能,而两者的功能是不兼容的.    
> 故提出几套解决方案. 

### (2). 方案一
> OpenRestry(网关) + Redis(注册中心) + 业务微服务:   
> 1. <font color='red'>业务微服务,不使用K8S的Service功能.</font>   
> 2. 业务微服务启动,向:Redis注册(key=微服务名称  field = podIp:port value=now()).  
> 3. OpenRestry提供Service功能(暴露80/443端口),读取Redis,根据路由规则,发分请求到容器.   
> 4. 所有一切都是自研,需要注意:Redis之前还要加缓存.但是,问题是:缓存多长时间呢?暂时能考虑的是:监听ETCD变化来刷缓存.     

### (3). 方案二
> Ingress + Spring Cloud Gateway + Eureka(Zookeeper/Nacos) + 业务微服务:   
> 1. eureka 和 业务微服务,<font color='red'>不使用Service功能(也就是不暴露服务,仅内部访问).</font>      
> 2. 业务微服务 和 Gateway 都向 eureka注册,这时在注册中心的元数据是:PodIP:port.   
> 3. Spring Cloud Gateway通过Service(NodePort)暴露出IP和端口.     
> 4. Ingress接受请求,全部转发到:Gateway上,再由:Gateway进行分发到业务微服务上.    
> 5. 注意:eureka注册时,需要稍微注意一些.它必须是有状态的,而且还要是基于域名的.      
> 6. 优势:不是所有的微服务都要使用Service功能.访方案:占用Node(宿主机)的端口也比较少.  
> 7. 原理:相当于Gateway往Nginx的upstream中注册,而Gateway和Pod是在同一网段,自然也能相互访问.  

### (4). 方案三
> Spring提供了:Spring Cloud Kubernetes解决方案(https://spring.io/projects/spring-cloud-kubernetes).   
> 该组件的原理:把微服注册和服务发现,基于:Kubenetes Service实现了一套,它和Eureka/Zookeeper/Nacos等是平级的.  

### (5). 方案四
> K8S所提供的解决方案:  
> Ingress + Service(NodePort) + 业务微服务      
> 1. 不依赖任何的服务(Zookeeper/Eureka/Nacos...)发现组件,<font color='red'>放弃微服务所提供的服务发现与注册组件</font>.   
> 2. 所有的业务微服务都使用Service和Ingress(不使用方案二中的:Gateway).   
> 3. 如果,业务有需求要对网关进行编排,可对Ingress(Go语言基于OpenRestry开发)进行二次开发.     
> 4. 该方案是:K8S默认方案,但是:Node(宿主机上暴露的端口太多了),可自由编程度相比方案一比较低.    


### (6). 方案五
> 约定优于配置:  
> Ingress + 网关服务(Service+NodPort) + 业务微服务(Service+ClusterIP)
> 1. 从开发环境到生成环境,提前定义好service名称(domain).   
> 2. <font color='red'>为业务微服务,创建Service指定网络类型以及参数:ClusterIP/ports.port为80端口,一定要指定为为:80端口,否则就要用域名+端口访问,很是头痛.</font>    
> 3. 创建网关服务(Gateway/NodeJS/Zuul),并指定Service的网络类型为:NodePort.    
> 4. Ingress接受所有动态请求,把请求:全部分发给(第三步的)网关服务,网关服务直接转发给业务微服务.    
> 5. 原理: 网关服务和其他服务(业务微服务)都是一个集群之内.k8s提供了kube-dns功能(域名到IP的解析),
> 所以,网关服务和业务微服之间,可以完全基于域名(service)来访问.    

> 业务微服务Service配置

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: test-web
  name: test-web
spec:
  ports:
  - port: 80            # Service暴露的接口
    protocol: TCP
    targetPort: 80      # 容器暴露的端口
  selector:
    app: test-web
  type: ClusterIP       # 注意:是ClusterIP,这样就不会Node上占用端口
```

> 测试

```
# 应用Service配置之前查看:pods和service
[root@master ~]# kubectl get pods,svc
NAME                               READY   STATUS    RESTARTS   AGE
pod/hello-world-5f8d77c9f7-589kd   1/1     Running   5          27h

#########################################################################
# 有一个Service,网络类型为:NodePort,暴露的端口为:9090:31355
#########################################################################
NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/hello-world   NodePort    10.1.232.6   <none>        9090:31355/TCP   2d4h
service/kubernetes    ClusterIP   10.1.0.1     <none>        443/TCP          5d5h

# 应用配置
[root@master ~]# kubectl apply -f test-web-expose.yml
service/test-web created

# 应用配置之后,查看pods和service
[root@master ~]# kubectl get pods,svc
NAME                               READY   STATUS    RESTARTS   AGE
pod/hello-world-5f8d77c9f7-589kd   1/1     Running   5          27h
pod/web-697d9c7964-hxpqh           1/1     Running   3          23h
pod/web-697d9c7964-sd4ww           1/1     Running   3          23h
pod/web-697d9c7964-x2f4d           1/1     Running   3          23h

#########################################################################
#                                     看这两者的不同
#   service/hello-world 为NodePort,内部访问的端口是:9090,外部访问的(宿主机IP:端口)端口是:31355
#   service/test-web 为ClusterIP,只有内部访问,端口为:80,并没有占用宿主机的端口.
#########################################################################
NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/hello-world   NodePort    10.1.232.6   <none>        9090:31355/TCP   2d4h
service/kubernetes    ClusterIP   10.1.0.1     <none>        443/TCP          5d5h
service/test-web      ClusterIP   10.1.121.5   <none>        80/TCP           4s

# 在宿主机通过:CLUSTER-IP访问
[root@master ~]# curl http://10.1.121.5/hello
Hello World-3.0.0-SNAPSHOT!!!test-hello


# 进入hello-world容器内部
# 在hello-world-5f8d77c9f7-589kd容器内部通过:service名称(test-web)来访问.
[root@master ~]# kubectl  exec -it hello-world-5f8d77c9f7-589kd  /bin/bash

#########################################################################
# ************************************重点*******************************
# 在容器内部可以直接通过域名访问.那么Java代码直接用域名就好了.
# 稍微的改动下网关服务.
# ************************************重点*******************************
#########################################################################
[root@hello-world-5f8d77c9f7-589kd /]# curl http://test-web/hello
Hello World-3.0.0-SNAPSHOT!!!test-hello


# 那么宿主机,能域名访问吗?肯下是不行的,因为:宿主机和kube-dns是隔离的
[root@master ~]# curl http://test-web/hello
curl: (6) Could not resolve host: test-web; Unknown error
```

### (7). 方案六
> K8S中Service的网络类型有三种:cluertip,nodeport,loadbanlance,可以选择自研:loadbanlance.   