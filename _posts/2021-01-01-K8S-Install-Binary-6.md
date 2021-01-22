---
layout: post
title: 'Kubernetes 二进制安装之Master(kube-controller-manager/kube-scheduler)部署(六)'
date: 2021-01-01
author: 李新
tags: K8S K8S安装
---

### (1). Master节点需要部署的组件有以下三个
> 1. kube-apiserver   
> 2. kube-controller-manager   
> 3. kube-scheduler   

### (2). 创建kube-controller-manager环境变量配置文件(/opt/kubernetes/config/kube-controller-manager)
```
KUBE_CONTROLLER_MANAGER_OPTS=" --logtostderr=true \
--v=4 \
--master=127.0.0.1:8080 \
--leader-elect=true \
--address=127.0.0.1 \
--service-cluster-ip-range=10.0.0.0/24 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \
--root-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
--experimental-cluster-signing-duration=87600h0m0s "
```
### (3). 通过systemd来管理kube-controller-manager(/usr/lib/systemd/system/kube-controller-manager.service)
```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=-/opt/kubernetes/config/kube-controller-manager
ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```
### (4). 创建kube-scheduler环境变量配置文件(/opt/kubernetes/config/kube-scheduler)
```
KUBE_SCHEDULER_OPTS="--logtostderr=true \
--v=4 \
--master=127.0.0.1:8080 \
--leader-elect"
```
### (5). 通过systemd来管理kube-scheduler(/usr/lib/systemd/system/kube-scheduler.service)
```
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=-/opt/kubernetes/config/kube-scheduler
ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```
### (6). 启动
```
# 加载配置
$ systemctl daemon-reload
# 开启系统运行自动加载
$ systemctl enable kube-scheduler
# 启动kube-scheduler
$ systemctl start kube-scheduler

# 开启系统运行自动加载
$ systemctl enable kube-controller-manager
# 启动kube-controller-manager
$ systemctl start kube-controller-manager
```
### (7). 检查当前节点状态
```
# 查看集群状态
# [root@master k8s-cert]# /opt/kubernetes/bin/kubectl get componentstatuses
[root@master k8s-cert]# /opt/kubernetes/bin/kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```

### (8). 为刚才创建(/opt/kubernetes/config/token.csv)的用户:(kubelet-bootstrap)绑定到系统集群角色
```
[root@master k8s-cert]# kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```
### (9). 自签admin证书
```
# 工作目录
[root@master k8s-cert]# pwd
/root/k8s/k8s-cert

# 创建admin证书请求
[root@master k8s-cert]# vi admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

# 生成admin证书
[root@master k8s-cert]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
[root@master k8s-cert]# ll |grep admin
-rw-r--r-- 1 root root 1009 Jan 22 13:38 admin.csr
-rw-r--r-- 1 root root  230 Jan 21 22:50 admin-csr.json
-rw------- 1 root root 1675 Jan 22 13:38 admin-key.pem
-rw-r--r-- 1 root root 1354 Jan 22 13:38 admin.pem
```

### (10). 自签kube-proxy证书
```
[root@master k8s-cert]# cat kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

# 生成证书
[root@master k8s-cert]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```
### (11). 创建kubeconfig
```
[root@master k8s-cert]# vi kubeconfig.sh
#!/bin/bash
BOOTSTRAP_TOKEN=b7cceeff1ab3948f0f49185e64709331
APISERVER=$1
SSL_DIR=$2
export KUBE_APISERVER="https://$APISERVER:6443"
kubectl config set-cluster kubernetes \
  --certificate-authority=$SSL_DIR/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
kubectl config set-cluster kubernetes \
  --certificate-authority=$SSL_DIR/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --client-certificate=$SSL_DIR/kube-proxy.pem \
  --client-key=$SSL_DIR/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

> 运行脚本

```
[root@master k8s-cert]# sh kubeconfig.sh 10.211.55.100 /root/k8s/k8s-cert
Cluster "kubernetes" set.
User "kubelet-bootstrap" set.
Context "default" created.
Switched to context "default".
Cluster "kubernetes" set.
User "kube-proxy" set.
Context "default" created.
Switched to context "default".


# 查看生成的结果
[root@master k8s-cert]# ll |grep kubeconfig
-rw------- 1 root root 2043 Jan 22 13:51 bootstrap.kubeconfig
-rw------- 1 root root 6085 Jan 22 13:51 kube-proxy.kubeconfig
```

### (12). 拷贝(bootstrap.kubeconfig/kube-proxy.kubeconfig)到node-1和node-2的指定目录
```
# 拷贝文件到node-1和node-2节点(/opt/kubernetes/config/)
[root@master k8s-cert]# scp  bootstrap.kubeconfig kube-proxy.kubeconfig root@10.211.55.101:/opt/kubernetes/config/
[root@master k8s-cert]# scp  bootstrap.kubeconfig kube-proxy.kubeconfig root@10.211.55.102:/opt/kubernetes/config/

```