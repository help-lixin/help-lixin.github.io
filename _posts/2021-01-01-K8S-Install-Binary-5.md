---
layout: post
title: 'Kubernetes 二进制安装之Master(kube-apiserver)部署(五)'
date: 2021-01-01
author: 李新
tags: K8S二进制安装
---

### (1). Master节点需要部署的组件有以下三个
> 1. kube-apiserver   
> 2. kube-controller-manager   
> 3. kube-scheduler   

### (2). 自签kube-apiserver证书
```
# 当前工作目录
[root@master k8s-cert]# pwd
/root/k8s/k8s-cert

[root@master k8s-cert]# vi ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}

[root@master k8s-cert]# vi ca-csr.json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}


# 10.0.0.1      是Service的地址
# kubernets*    是kubernets要求填写的地址
# 10.211.55.100 是本机master
# 10.211.55.*   是LB或者其它机器地址
[root@master k8s-cert]# vi server-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "10.211.55.100",
      "10.211.55.107",
      "10.211.55.108",
      "10.211.55.109",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
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

# 生成ca
[root@master k8s-cert]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

# 生成api-server签名证书
[root@master k8s-cert]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

# 拷贝生成的证书到安装目录(/opt/kubernetes/ssl/)
[root@master k8s-cert]# cp /root/k8s/k8s-cert/{ca.pem,ca-key.pem,server.pem,server-key.pem} /opt/kubernetes/ssl/

# 检查下证书是否存在
[root@master k8s-cert]# ll /opt/kubernetes/ssl/
-rw------- 1 root root 1679 Jan 21 23:07 ca-key.pem
-rw-r--r-- 1 root root 1265 Jan 21 23:07 ca.pem
-rw------- 1 root root 1679 Jan 21 23:07 server-key.pem
-rw-r--r-- 1 root root 1590 Jan 21 23:07 server.pem
```
### (3). 生成api-token(/opt/kubernetes/config/token.csv)文件
```
# 生成一个token
[root@master k8s-cert]# head -c 16 /dev/urandom | od -An -t x|tr -d ' '
b7cceeff1ab3948f0f49185e64709331

# 创建token文件(/opt/kubernetes/config/token.csv)
# 格式：token,用户名,UID,用户组
[root@master k8s-cert]# vi /opt/kubernetes/config/token.csv
b7cceeff1ab3948f0f49185e64709331,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```
### (4). 下载kubernetes-server-linux-amd64
> https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG   
> https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.19.md   

```
# 创建k8s应用程序目录
[root@master ~]# mkdir  -p /opt/kubernetes/{bin,config,ssl,logs}

# 下载有一些,建议:翻墙下载后再上传
[root@master ~]# wget https://dl.k8s.io/v1.19.7/kubernetes-server-linux-amd64.tar.gz

# 解压
[root@master ~]# tar -zxvf kubernetes-server-linux-amd64-v1.19.7.tar.gz
[root@master ~]# cd kubernetes/server/bin/

# 查看:kubernetes/server/bin下的内容
[root@master bin]# ls
apiextensions-apiserver    kube-apiserver.tar                  kubelet                kube-scheduler.docker_tag
kubeadm                    kube-controller-manager             kube-proxy             kube-scheduler.tar
kube-aggregator            kube-controller-manager.docker_tag  kube-proxy.docker_tag  mounter
kube-apiserver             kube-controller-manager.tar         kube-proxy.tar
kube-apiserver.docker_tag  kubectl                             kube-scheduler

# 拷贝(kube-apiserver/kube-controller-manager/kube-scheduler/kubectl)
[root@master bin]# cp kube-apiserver kube-controller-manager kube-scheduler kubectl /opt/kubernetes/bin/
[root@master bin]# cp kubectl /usr/local/bin/
```

### (5). 创建kube-apiserver环境变量配置文件(/opt/kubernetes/config/kube-apiserver)
> --logtostderr 启用日志   
> --v 日志等级   
> --etcd-servers etcd集群地址   
> --bind-address 监听地址   
> --secure-port https安全端口   
> --advertise-address 集群通告地址   
> --allow-privileged 启用授权   
> --service-cluster-ip-range Service虚拟IP地址段   
> --enable-admission-plugins 准入控制模块   
> --authorization-mode 认证授权，启用RBAC授权和节点自管理  
> --enable-bootstrap-token-auth启用TLS bootstrap功能   
> --token-auth-file token文件   
> --service-node-port-range Service Node类型默认分配端口范围   

```
KUBE_APISERVER_OPTS=" --logtostderr=false \
--log-dir=/opt/kubernetes/logs \
--v=4 \
--etcd-servers=https://10.211.55.100:2379,https://10.211.55.101:2379,https://10.211.55.102:2379 \
--bind-address=10.211.55.100 \
--secure-port=6443 \
--advertise-address=10.211.55.100 \
--allow-privileged=true \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--kubelet-https=true \
--enable-bootstrap-token-auth \
--token-auth-file=/opt/kubernetes/config/token.csv \
--service-cluster-ip-range=10.0.0.0/24 \
--service-node-port-range=30000-50000 \
--tls-cert-file=/opt/kubernetes/ssl/server.pem \
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/opt/etcd/ssl/ca.pem \
--etcd-certfile=/opt/etcd/ssl/server.pem \
--etcd-keyfile=/opt/etcd/ssl/server-key.pem "
```
### (6). 通过systemd来管理kube-apiserver(/usr/lib/systemd/system/kube-apiserver.service)
```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=-/opt/kubernetes/config/kube-apiserver
ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
```
### (7). 启动kube-apiserver
```
# 加载配置
$ systemctl daemon-reload
# 开启系统运行自动加载
$ systemctl enable kube-apiserver
# 启动kube-apiserver
$ systemctl start kube-apiserver
```
### (8). 检查kube-apiserver进程是否启动成功
```
[root@master k8s-cert]# ps -ef|grep kube-apiserver
root     18899     1 57 23:24 ?        00:00:12 /opt/kubernetes/bin/kube-apiserver --logtostderr=false --log-dir=/opt/kubernetes/logs --v=4 --etcd-servers=https://10.211.55.100:2379,https://10.211.55.101:2379,https://10.211.55.102:2379 --bind-address=10.211.55.100 --secure-port=6443 --advertise-address=10.211.55.100 --allow-privileged=true --service-cluster-ip-range=10.0.0.0/24 --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction --authorization-mode=RBAC,Node --kubelet-https=true --enable-bootstrap-token-auth --token-auth-file=/opt/kubernetes/config/token.csv --service-node-port-range=30000-50000 --tls-cert-file=/opt/kubernetes/ssl/server.pem --tls-private-key-file=/opt/kubernetes/ssl/server-key.pem --client-ca-file=/opt/kubernetes/ssl/ca.pem --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem --etcd-cafile=/opt/etcd/ssl/ca.pem --etcd-certfile=/opt/etcd/ssl/server.pem --etcd-keyfile=/opt/etcd/ssl/server-key.pem
```
