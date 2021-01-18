---
layout: post
title: 'Kubernetes 集群安全机制(十七)'
date: 2021-01-01
author: 李新
tags: K8S
---

### (1). 概述
> 我们访问K8S集群时,大体的流程如下:   
> 1. 认证.  
> 2. 鉴权.   
> 3. 准入控制.   
> 4. apiServer做统一协调和调度(需要:证书/token/serviceAccount).  
> 5. ... ...
### (2). 认证
> 1. https证书认证,基于ca证书.   
> 2. http token认证,通过token识别用户.   
> 3. http基于用户名+密码进行认证.   
### (3). 鉴权
>  基于RBAC进行鉴权操作(基于角色的访问控制).  
>  
### (4). 准入控制
> 发送的请求资源存在,则允许调用(实际是因为:鉴权太粗). 
### (5). K8S有哪些角色和主体呢?
> 角色有哪些呢?     
> 1. role : 对某个命名空间访问权限控制. 
> 2. ClusterRole: 所有命名空间访问权限.

> 角色绑定有哪些呢?   
> 1. roleBinding : 将角色绑定到主体(用户上)
> 2. ClusterRoleBinding  : 将集群角色绑定到主体上. 

> 主体有哪些呢?   
> 1. user : 普通用户.  
> 2. group : 用户组.  
> 3. serviceAccount :  服务账号,一般用于pod的访问
### (6). 实操
```

# 1. 创建命名空间
[root@master k8s-pv-pvc]# kubectl get ns
NAME              STATUS   AGE
default           Active   9d
ingress-nginx     Active   3d2h
kube-node-lease   Active   9d
kube-public       Active   9d
kube-system       Active   9d

# 1.1 创建一个命名空间
[root@master k8s-pv-pvc]# kubectl create ns dev
namespace/dev created

# 1.2 查看命名空间
[root@master k8s-pv-pvc]# kubectl get ns
NAME              STATUS   AGE
default           Active   9d
dev               Active   2s
ingress-nginx     Active   3d2h
kube-node-lease   Active   9d
kube-public       Active   9d
kube-system       Active   9d

# 2. 在dev命令空间下创建pod
[root@master ~]# cat test-ns-dev-nginx.yml
apiVersion: v1
kind: Pod
metadata:
  namespace: dev
  labels:
    app: nginx
  name: nginx
spec:
  containers: # 指定多个镜像,代表这些镜像在一个Pod内.
  - image: nginx:latest
    name: nginx

# 2.1 发布pod配置
[root@master ~]# kubectl apply -f test-ns-dev-nginx.yml
pod/nginx created

# 2.2 查看dev命名空间下的pod
[root@master ~]# kubectl get pods -n dev
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          80s

# 3. 创建角色,为角色:绑定资源
[root@master ~]# cat rbac-role.yml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: dev
  name: dev  # 角色名称
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"] # 角色名(dev),拥有对pod的get,watch,list等资源的权限
  verbs: ["get", "watch", "list"]

# 3.1 创建角色,为角色:绑定资源
[root@master ~]# kubectl apply -f rbac-role.yml
role.rbac.authorization.k8s.io/dev created

# 3.2 查看dev命名空间下有哪些角色名称
[root@master ~]# kubectl get role -n dev
NAME   AGE
dev    31s

# 4. 创建角色与用户的绑定
[root@master ~]# cat rbac-rolebinding.yml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev-read-pods
  namespace: dev #命名空间
subjects:
- kind: User
  name: zhangsan
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev  # 引用角色名称
  apiGroup: rbac.authorization.k8s.io
  
# 4.1 创建角色与用户的绑定
[root@master ~]# kubectl apply -f rbac-rolebinding.yml
rolebinding.rbac.authorization.k8s.io/dev-read-pods created

# 4.2 查看dev命名空间下有哪些rolebinding
[root@master ~]# kubectl get role,rolebinding -n dev
NAME                                 AGE
role.rbac.authorization.k8s.io/dev   10m

NAME                                                  AGE
rolebinding.rbac.authorization.k8s.io/dev-read-pods   3m1s

# 4.3 查看(dev-read-pods)详细
[root@master ~]# kubectl describe rolebinding dev-read-pods -n dev
Name:         dev-read-pods
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"rbac.authorization.k8s.io/v1beta1","kind":"RoleBinding","metadata":{"annotations":{},"name":"dev-read-pods","namespace":"de...
Role:
  Kind:  Role
  Name:  dev
Subjects:
  Kind  Name      Namespace
  ----  ----      ---------
  User  zhangsan
  

# 5. 使用证书来识别身份


```
### (7). 

### (8). 

### (9). 

### (10).
