---
layout: post
title: 'Kubernetes 二进制安装之Node(kubelet/kube-proxy)部署(七)'
date: 2021-01-01
author: 李新
tags: 二进制安装K8S
---

### (1). Node节点需要部署以下组件 
> 1. kubelet    
> 2. kube-proxy    

### (2). 准备工作
```
# 部署node节点,需要的组件(从master拷贝)
[root@master ~]# scp kubernetes/server/bin/{kubelet,kube-proxy}  root@10.211.55.101:/opt/kubernetes/bin/

# 为node-1节点创建日志目录
[root@node-1 ~]# mkdir -p /opt/kubernetes/logs
```
### (3). 创建kubelet环境变量配置文件(/opt/kubernetes/config/kubelet)
> /opt/kubernetes/config/kubelet.kubeconfig在加入集群时,会自动创建. 

```
KUBELET_OPTS=" --logtostderr=false \
--log-dir=/opt/kubernetes/logs \
--v=4 \
--address=10.211.55.101 \
--hostname-override=10.211.55.101 \
--kubeconfig=/opt/kubernetes/config/kubelet.kubeconfig \
--experimental-bootstrap-kubeconfig=/opt/kubernetes/config/bootstrap.kubeconfig \
--config=/opt/kubernetes/config/kubelet.config \
--cert-dir=/opt/kubernetes/ssl \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0 "
```
### (4). 创建:/opt/kubernetes/config/kubelet.config
```
[root@node-1 ~]# vi /opt/kubernetes/config/kubelet.config
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 10.211.55.101
port: 10250
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
```
### (5). 通过systemd来管理kubelet(/usr/lib/systemd/system/kubelet.service)
```
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service
 
[Service]
EnvironmentFile=/opt/kubernetes/config/kubelet
ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
KillMode=process
 
[Install]
WantedBy=multi-user.target
```
### (6). 创建kube-proxy环境变量配置文件(/opt/kubernetes/config/kube-proxy)
```
# 注意两台node的ip不一样
KUBE_PROXY_OPTS=" --logtostderr=false \
--log-dir=/opt/kubernetes/logs \
--v=4 \
--hostname-override=10.211.55.101 \
--cluster-cidr=10.0.0.0/24 \
--proxy-mode=ipvs \
--kubeconfig=/opt/kubernetes/config/kube-proxy.kubeconfig "
```
### (7). 通过systemd来管理kube-proxy(/usr/lib/systemd/system/kube-proxy.service)
```
[Unit]
Description=Kubernetes Proxy
After=network.target
 
[Service]
EnvironmentFile=-/opt/kubernetes/config/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target　
```
### (8). 启动kubelet
```
[root@node-1 ~]#  systemctl daemon-reload
[root@node-1 ~]#  systemctl restart kubelet
[root@node-1 ~]#  systemctl enable kubelet
```
### (9). Master允许node-1节点加入集群
```
# 查看node-1节点发起的:csr请求
[root@master ~]# kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-NMZhSNEKprZdRpkShO84T0TJGglShZp2IxM_jXjMIVM   9m27s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 允许这个节点加入K8S集群
[root@master ~]# kubectl certificate approve node-csr-NMZhSNEKprZdRpkShO84T0TJGglShZp2IxM_jXjMIVM
certificatesigningrequest.certificates.k8s.io/node-csr-NMZhSNEKprZdRpkShO84T0TJGglShZp2IxM_jXjMIVM approved

# 再次查看信息,由:Pending -> Approved,Issued
[root@master ~]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-NMZhSNEKprZdRpkShO84T0TJGglShZp2IxM_jXjMIVM   11m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

# 查看node状态
[root@master ~]# kubectl get nodes
NAME            STATUS   ROLES    AGE    VERSION
10.211.55.101   Ready    <none>   102s   v1.19.7
```
### (10). 启动kube-proxy
 ```
 [root@node-1 ~]#  systemctl daemon-reload
 [root@node-1 ~]#  systemctl enable kube-proxy
 # 在执行:systemctl enable kube-proxy时,有可能报错(Failed to execute operation: Invalid argument),解决方案如下
 # 创建软链接即可
 # [root@node-1 ~]# ln -s /usr/lib/systemd/system/kube-proxy.service /etc/systemd/system/multi-user.target.wants/kube-proxy.service
 
 [root@node-1 ~]#  systemctl restart kube-proxy
 ```
 
 ### (11). 部署node-2节点(拷贝node-1节点数据到node-2)
```
# 拷贝node-1节点数据到:node-2
[root@node-1 ~]# scp -r  /opt/kubernetes root@10.211.55.102:/opt/
[root@node-1 ~]# scp -r  /usr/lib/systemd/system/kubelet.service  root@10.211.55.102:/usr/lib/systemd/system/kubelet.service
[root@node-1 ~]# scp -r /usr/lib/systemd/system/kube-proxy.service  root@10.211.55.102:/usr/lib/systemd/system/kube-proxy.service
```
### (12). 部署node-2节点
```

# 1. 删除master允许node-1加入K8S集群时,创建的证书
[root@node-2 ~]# rm -rf /opt/kubernetes/ssl/*

# 2. 删除日志目录
[root@node-2 ~]# rm -rf /opt/kubernetes/logs/*

# 3. 修改配置(node-2:10.211.55.102)
#    将这三个文件里的IP(10.211.55.101)改成:10.211.55.102
[root@node-2 ~]# vi /opt/kubernetes/config/kubelet
[root@node-2 ~]# vi /opt/kubernetes/config/kubelet.config
[root@node-2 ~]# vi /opt/kubernetes/config/kube-proxy

# 4. 启动kubelet/kube-proxy
[root@node-2 ~]#  systemctl daemon-reload
[root@node-2 ~]#  systemctl restart kubelet
[root@node-2 ~]#  systemctl enable kubelet

[root@node-2 ~]# systemctl daemon-reload
[root@node-2 ~]# systemctl enable kube-proxy
# 在执行:systemctl enable kube-proxy时,有可能报错(Failed to execute operation: Invalid argument),解决方案如下
 # 创建软链接即可
 # [root@node-1 ~]# ln -s /usr/lib/systemd/system/kube-proxy.service /etc/systemd/system/multi-user.target.wants/kube-proxy.service
[root@node-2 ~]# systemctl restart kube-proxy
```

### (13). Master允许node-2节点加入K8S集群
```
[root@master ~]# kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-IWVRVfEf2aZ_y_QVaqrxGos_1_pMkkjhUCpbrwShWmI   2m23s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-NMZhSNEKprZdRpkShO84T0TJGglShZp2IxM_jXjMIVM   48m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

[root@master ~]# kubectl certificate approve node-csr-IWVRVfEf2aZ_y_QVaqrxGos_1_pMkkjhUCpbrwShWmI
certificatesigningrequest.certificates.k8s.io/node-csr-IWVRVfEf2aZ_y_QVaqrxGos_1_pMkkjhUCpbrwShWmI approved
[root@master ~]# kubectl get csr
[root@master ~]# kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-IWVRVfEf2aZ_y_QVaqrxGos_1_pMkkjhUCpbrwShWmI   2m36s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-NMZhSNEKprZdRpkShO84T0TJGglShZp2IxM_jXjMIVM   48m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

[root@master ~]# kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
10.211.55.101   Ready    <none>   37m   v1.19.7
10.211.55.102   Ready    <none>   11s   v1.19.7
```

### (14). 在Master节点添加认证的用户

```
# exec进不了容器
[root@master ~]# kubectl exec -it dig  -- nslookup kubernetes
error: unable to upgrade connection: Unauthorized

# 在master节点上,添加认证用户
[root@master ~]# kubectl create clusterrolebinding system:anonymous --clusterrole=cluster-admin --user=system:anonymous
	clusterrolebinding.rbac.authorization.k8s.io/system:anonymous created
```