---
layout: post
title: 'K8S整合Kong(五)'
date: 2021-06-09
author: 李新
tags:  Kong K8S
---

### (1). 参考链接
```
https://konghq.com/blog/kubernetes-ingress-controller-for-kong/
```
### (2). 下载yaml文件
```
# all-in-one-postgres.yaml为K8S定义Kong的文件
https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/single/all-in-one-postgres.yaml

# ****************************************************************
# dummy-application.yaml是我们的HTTP服务文件(我修改成了nginx)
# ****************************************************************
https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/manifests/dummy-application.yaml
```

["all-in-one-postgres.yaml为定义K8S资源文件"](/assets/kong/imgs/all-in-one-postgres.yaml)   
["dummy-application.yaml为HTTP Service文件"](/assets/kong/imgs/dummy-application.yaml)    
### (3). 查看依赖哪些镜像(提前pull,避免在应用yaml时等待时间过长)
```
lixin-macbook:Desktop lixin$ cat all-in-one-postgres.yaml |grep image
        image: kong:2.4
        image: kong/kubernetes-ingress-controller:1.3
        image: postgres:9.6
        image: busybox
```

### (4). minikube运行
```
lixin-macbook:~ lixin$ minikube start --vm-driver=hyperkit --registry-mirror=https://registry.docker-cn.com --cpus 2 --memory 4096
😄  Darwin 11.2.3 上的 minikube v1.16.0
✨  根据现有的配置文件使用 hyperkit 驱动程序
👍  Starting control plane node minikube in cluster minikube
🔄  Restarting existing hyperkit VM for "minikube" ...
❗  This VM is having trouble accessing https://k8s.gcr.io
💡  To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
🐳  正在 Docker 20.10.0 中准备 Kubernetes v1.20.0…
🔎  Verifying Kubernetes components...
🔎  Verifying ingress addon...
🌟  Enabled addons: storage-provisioner, default-storageclass, ingress
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

### (5). 设置本地docker与minukube关联
```
lixin-macbook:~ lixin$ eval $(minikube -p minikube docker-env)
```
### (6). 应用配置
```
lixin-macbook:~ lixin$ kubectl apply -f all-in-one-postgres.yaml
namespace/kong created
customresourcedefinition.apiextensions.k8s.io/kongclusterplugins.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongconsumers.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongingresses.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongplugins.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/tcpingresses.configuration.konghq.com created
serviceaccount/kong-serviceaccount created
clusterrole.rbac.authorization.k8s.io/kong-ingress-clusterrole created
clusterrolebinding.rbac.authorization.k8s.io/kong-ingress-clusterrole-nisa-binding created
service/kong-proxy created
service/kong-validation-webhook created
service/postgres created
deployment.apps/ingress-kong created
statefulset.apps/postgres created
job.batch/kong-migrations created
```
### (7). 检查kong命名空间
```
# 查看命名空间(kong)下所有的pod和service
lixin-macbook:~ lixin$ kubectl get pods,svc -n kong -o wide
NAME                               READY   STATUS      RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
pod/ingress-kong-6bcd9bb89-sf4s4   2/2     Running     6          26h   172.17.0.4   minikube   <none>           <none>
pod/kong-migrations-xwfdb          0/1     Completed   0          26h   172.17.0.5   minikube   <none>           <none>
pod/postgres-0                     1/1     Running     2          26h   172.17.0.6   minikube   <none>           <none>

NAME                              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
service/kong-proxy                LoadBalancer   10.107.235.154   <pending>     80:31616/TCP,443:30627/TCP   26h   app=ingress-kong
service/kong-validation-webhook   ClusterIP      10.110.0.140     <none>        443/TCP                      26h   app=ingress-kong
service/postgres                  ClusterIP      10.98.75.34      <none>        5432/TCP                     26h   app=postgres
```

### (8). 测试访问kong
```
# ***************************************************************************
# 1. 通过minikube打开命名空间(kong)下的service(kong-proxy)
# 这一步的原理是在宿主机(mac)上与minikube建立虚拟隧道.
# bridge100: flags=8a63<UP,BROADCAST,SMART,RUNNING,ALLMULTI,SIMPLEX,MULTICAST> mtu 1500
	options=3<RXCSUM,TXCSUM>
	ether ae:bc:32:98:dd:64
	inet 192.168.64.1 netmask 0xffffff00 broadcast 192.168.64.255
# ***************************************************************************	
lixin-macbook:~ lixin$ minikube service -n kong kong-proxy
|-----------|------------|---------------|---------------------------|
| NAMESPACE |    NAME    |  TARGET PORT  |            URL            |
|-----------|------------|---------------|---------------------------|
| kong      | kong-proxy | proxy/80      | http://192.168.64.3:31616 |
|           |            | proxy-ssl/443 | http://192.168.64.3:30627 |
|-----------|------------|---------------|---------------------------|
🎉  正通过默认浏览器打开服务 kong/kong-proxy...
🎉  正通过默认浏览器打开服务 kong/kong-proxy...


# 2. 测试访问
lixin-macbook:~ lixin$ curl -vvv -i http://192.168.64.3:31616
*   Trying 192.168.64.3...
* TCP_NODELAY set
* Connected to 192.168.64.3 (192.168.64.3) port 31616 (#0)
> GET / HTTP/1.1
> Host: 192.168.64.3:31616
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 404 Not Found
HTTP/1.1 404 Not Found
< Date: Mon, 14 Jun 2021 10:53:01 GMT
Date: Mon, 14 Jun 2021 10:53:01 GMT
< Content-Type: application/json; charset=utf-8
Content-Type: application/json; charset=utf-8
< Connection: keep-alive
Connection: keep-alive
< Content-Length: 48
Content-Length: 48
< X-Kong-Response-Latency: 0
X-Kong-Response-Latency: 0
# **********************************************
# 这部份才是我想要看到的内容.
# # **********************************************
< Server: kong/2.4.1
Server: kong/2.4.1

<
* Connection #0 to host 192.168.64.3 left intact
{"message":"no Route matched with those values"}* Closing connection 0
```

### (9). 部署Service(http-svc)
```
lixin-macbook:~ lixin$ kubectl  apply -f dummy-application.yaml
deployment.apps/http-svc created
service/http-svc created
```
### (10). 检查默认命名空间
```
lixin-macbook:~ lixin$ kubectl get pods,svc -o wide
# 对应的nginxIP地址为:172.17.0.2
NAME                            READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
pod/http-svc-75f96fb595-gmkxl   1/1     Running   2          26h   172.17.0.2   minikube   <none>           <none>

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE    SELECTOR
service/http-svc     NodePort    10.101.114.15   <none>        80:30429/TCP   26h    app=http-svc
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        149d   <none>
```
### (11). 定义Ingress 
```
echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-hello-world
  annotations:
      kubernetes.io/ingress.class: kong
spec:
  rules:
  - host: "nginx.hello.world"
    http:
      paths:
         - backend:
            serviceName: http-svc
            servicePort: 80
" | kubectl create -f -
```
### (12). 部署ingoress
```
lixin-macbook:~ lixin$ kubectl get ingress -o wide
# nginx.hello.world 对应的ip地址是:192.168.64.3

NAME                CLASS    HOSTS               ADDRESS        PORTS   AGE
nginx-hello-world   <none>   nginx.hello.world   192.168.64.3   80      25h

# 查看ingress详细
lixin-macbook:~ lixin$ kubectl describe ingress nginx-hello-world
Name:             nginx-hello-world
Namespace:        default
Address:          192.168.64.3
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host               Path  Backends
  ----               ----  --------
  nginx.hello.world
                        http-svc:80 (172.17.0.2:80)   # 这里与上面的nginxIP地址对应上了.
Annotations:         <none>
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  10m   nginx-ingress-controller  Ingress default/nginx-hello-world
  Normal  CREATE  10m   nginx-ingress-controller  Ingress default/nginx-hello-world
  Normal  UPDATE  10m   nginx-ingress-controller  Ingress default/nginx-hello-world
```
### (13). 验证
```
# 一定要先执行这一步,否则,你本地(mac)无法访问到:http://192.168.64.3:31616
# 1. 通过minikube service
lixin-macbook:~ lixin$ minikube service -n kong kong-proxy
|-----------|------------|---------------|---------------------------|
| NAMESPACE |    NAME    |  TARGET PORT  |            URL            |
|-----------|------------|---------------|---------------------------|
| kong      | kong-proxy | proxy/80      | http://192.168.64.3:31616 |
|           |            | proxy-ssl/443 | http://192.168.64.3:30627 |
|-----------|------------|---------------|---------------------------|
🎉  正通过默认浏览器打开服务 kong/kong-proxy...
🎉  正通过默认浏览器打开服务 kong/kong-proxy...

# 2. 测试下能否访问(显示no Route...代表kong启动成功)
lixin-macbook:~ lixin$ curl http://192.168.64.3:31616
{"message":"no Route matched with those values"}

# 3. 设置环境变量
lixin-macbook:~ lixin$ export PROXY_IP=$(minikube service -n kong kong-proxy --url | head -1)
lixin-macbook:~ lixin$ echo $PROXY_IP
http://192.168.64.3:31616

# ***********************************************************
# 4. 测试(访问kong,并设置要问访的host)
#    在访问时,必须要先走前面1-3步,让本机能和192.168.64.3通信.
# # ***********************************************************
lixin-macbook:~ lixin$ curl -vvv -i $PROXY_IP -H "Host: nginx.hello.world"
*   Trying 192.168.64.3...
* TCP_NODELAY set
* Connected to 192.168.64.3 (192.168.64.3) port 31616 (#0)
> GET / HTTP/1.1
> Host: nginx.hello.world
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Content-Type: text/html; charset=UTF-8
Content-Type: text/html; charset=UTF-8
< Content-Length: 612
Content-Length: 612
< Connection: keep-alive
Connection: keep-alive
< Server: nginx/1.21.0
Server: nginx/1.21.0
< Date: Mon, 14 Jun 2021 10:58:48 GMT
Date: Mon, 14 Jun 2021 10:58:48 GMT
< Last-Modified: Tue, 25 May 2021 12:28:56 GMT
Last-Modified: Tue, 25 May 2021 12:28:56 GMT
< ETag: "60aced88-264"
ETag: "60aced88-264"
< Accept-Ranges: bytes
Accept-Ranges: bytes
< X-Kong-Upstream-Latency: 0
X-Kong-Upstream-Latency: 0
< X-Kong-Proxy-Latency: 1
X-Kong-Proxy-Latency: 1
# ******************************************************************
# 能看到这些协议头:代表路由是经过kong
# ******************************************************************
< Via: kong/2.4.1
Via: kong/2.4.1
// ... ...
* Connection #0 to host 192.168.64.3 left intact
* Closing connection 0
```