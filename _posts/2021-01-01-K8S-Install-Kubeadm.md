---
layout: post
title: 'Kubernetes Kubeadmin(二)'
date: 2020-12-27
author: 李新
tags: K8S
---

### (1). 安装要求
> 一台或者多台机器,操作系统CentoOS7.x-86_x64.     
> 硬件配置:2G以上的RAM,2个CPU或者更多CPU.硬盘30G以上.    
> 集群中所有的机器之间网络互通.  
> 可能访问外网,因为需要拉取镜像.  
> 禁止Swap分区.

### (2). kubeadmin集群指令

```
# 在Master节点运行
# 创建一个Master节点
kubeadmin init

# 在Node节点运行
# 将Node节点加入到指定的Master集群中.
kubeadmin join <MASTER:PORT> 
```

### (3). 集群机器

|  IP            | 角色      |
|  ----          | ----     |
| 10.211.55.100  |  Master  |
| 10.211.55.101  |  Node1   |
| 10.211.55.102  |  Node2   |


### (4). 准备工作
```
# 所有机器关闭防火墙
$ systemctl stop firewalld
$ systemctl disable firewalld

# 所有机器关闭selinux
$ sed -i 's/enforcing/disabled/' /etc/selinux/config 
$ setenforce 0

# 所有机器关闭swap(编缉:/etc/fstab,注释掉最后一行:swap)
$ vi /etc/fstab
	# /dev/mapper/VolGroup-lv_swap swap                    swap    defaults        0 0


# 所有机器之间添加host与ip映射(/etc/hosts)
10.211.55.100   master
10.211.55.101   node-1
10.211.55.102   node-2

# 所有机器将桥接IPV4流量传递到iptables的链:
$ vi /etc/sysctl.d/k8s.conf
net.bridge.bridge‐nf‐call‐ip6tables = 1
net.bridge.bridge‐nf‐call‐iptables = 1


$ sysctl --system
```
### (5). 安装Docker
```
# 下载阿里云docker仓库
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

# 安装docker
$ yum -y install docker-ce-18.06.1.ce-3.el7

# 设置开机启动并运行docker
$ systemctl enable docker && systemctl start docker

$ docker -v
	Docker version 18.06.1-ce, build e68fc7a
```
### (6). 配置Yum仓库
```
$ cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
### (7). 所有机器安装(kubeadm/kubelet/kubectl)
```
# 安装:kubeadm/kubelet/kubectl
$ yum -y install kubelet-1.15.0 kubeadm-1.15.0 kubectl-1.15.0

# 启动kubelet
$ systemctl enable kubelet
```
### (8). 部署Kubenets Master(10.211.55.100)
> service-cidr               :     为kube-proxy指定虚拟IP网段             
> pod-network-cidr           :     为pod(容器)指定虚拟IP网段

```
[root@master ~]# kubeadm init \
  --apiserver-advertise-address=10.211.55.100 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.15.0 \
  --service-cidr=10.1.0.0/16 \
  --pod-network-cidr=10.244.0.0/16
 
[init] Using Kubernetes version: v1.15.0
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection

# 拉取镜像
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'

# 写入环境 
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service

# 生成证书
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [master localhost] and IPs [10.211.55.100 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [master localhost] and IPs [10.211.55.100 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key


[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.1.0.1 10.211.55.100]
[certs] Generating "apiserver-kubelet-client" certificate and key

# proxy证书
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key

# 生产配置文件
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file

# master节点的三个组件(kube-apiserver/kube-controller-manager/kube-scheduler/etcd)
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"

[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 38.003645 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]


[bootstrap-token] Using token: r0xuu3.ru4sqfpay7l8616k
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace

# 部署了两个组件
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

# node节点执行加入master节点
kubeadm join 10.211.55.100:6443 --token r0xuu3.ru4sqfpay7l8616k \
    --discovery-token-ca-cert-hash sha256:fede35b0b8565f919497a38426e59075748b87f26db8f3aeee77e14d4c0a5ba3
```



### (9). 手动拉取flannel镜像(所有节点执行)

```
$ docke pull quay.io/coreos/flannel:v0.10.0-amd64
```


### (10). 部署Kubenets Master(10.211.55.100)
> 拷贝配置文件到用户目录下

```
[root@master ~]# mkdir -p $HOME/.kube
[root@master ~]#   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@master ~]#   sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 查看所有的node节点
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
master   NotReady   master   10m   v1.15.0


# 安装pod网络插件(kube-flannel.yml我已提供下载)
[root@master ~]# kubectl apply -f  ./kube-flannel.yml
	clusterrole.rbac.authorization.k8s.io/flannel created
	clusterrolebinding.rbac.authorization.k8s.io/flannel created
	serviceaccount/flannel created
	configmap/kube-flannel-cfg created
	daemonset.extensions/kube-flannel-ds created

```

["kube-flannel.yml文件下载"](/assets/k8s/kube-flannel.yml)

### (11). coredns Pending状态

```
# 在Master查看pod状态
[root@master ~]# kubectl get pods -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-bccdc95cf-9vfbh          0/1     Pending   0          2m17s
coredns-bccdc95cf-kw752          0/1     Pending   0          2m17s
etcd-master                      1/1     Running   0          73s
kube-apiserver-master            1/1     Running   0          88s
kube-controller-manager-master   1/1     Running   0          75s
kube-proxy-mf658                 1/1     Running   0          2m17s
kube-scheduler-master            1/1     Running   0          82s
```


### (12). coredns Pending状态解决方案
> 该步骤要在所有的机器上执行

```
# 我的机器尝试过N次coredns都是Pending,尝试如下方法得到解决:
$ mkdir -p /etc/cni/net.d
$ vi /etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}

# 重新加载,并重启kubelet
$ systemctl daemon-reload
$ systemctl restart kubelet
```

### (13). node节点加入集群
> (node-1/node2都要执行加入集群)

```
[root@node-1 ~]# kubeadm join 10.211.55.100:6443 --token r0xuu3.ru4sqfpay7l8616k \
>     --discovery-token-ca-cert-hash sha256:fede35b0b8565f919497a38426e59075748b87f26db8f3aeee77e14d4c0a5ba3
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.15" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### (14). node查看日志

```
[root@node-2 ~]#  journalctl -f -u kubelet.service
-- Logs begin at Sat 2021-01-09 15:21:37 CST. --
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.814805    8323 remote_image.go:50] parsed scheme: ""
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.814811    8323 remote_image.go:50] scheme "" not registered, fallback to default scheme
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.815172    8323 asm_amd64.s:1337] ccResolverWrapper: sending new addresses to cc: [{/var/run/dockershim.sock 0  <nil>}]
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.815203    8323 clientconn.go:796] ClientConn switching balancer to "pick_first"
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.815291    8323 balancer_conn_wrappers.go:131] pickfirstBalancer: HandleSubConnStateChange: 0xc00097ebb0, CONNECTING
# 提示准备成功
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.815583    8323 balancer_conn_wrappers.go:131] pickfirstBalancer: HandleSubConnStateChange: 0xc00097ebb0, READY


Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.815619    8323 asm_amd64.s:1337] ccResolverWrapper: sending new addresses to cc: [{/var/run/dockershim.sock 0  <nil>}]
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.815634    8323 clientconn.go:796] ClientConn switching balancer to "pick_first"
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.815662    8323 balancer_conn_wrappers.go:131] pickfirstBalancer: HandleSubConnStateChange: 0xc00097ed00, CONNECTING
Jan 09 15:46:44 node-2 kubelet[8323]: I0109 15:46:44.815729    8323 balancer_conn_wrappers.go:131] pickfirstBalancer: HandleSubConnStateChange: 0xc00097ed00, READY
Jan 09 15:47:05 node-2 kubelet[8323]: E0109 15:47:05.081376    8323 aws_credentials.go:77] while getting AWS credentials NoCredentialProviders: no valid providers in chain. Deprecated.
Jan 09 15:47:05 node-2 kubelet[8323]: For verbose messaging see aws.Config.CredentialsChainVerboseErrors
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.083704    8323 kuberuntime_manager.go:205] Container runtime docker initialized, version: 18.06.1-ce, apiVersion: 1.38.0
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.085671    8323 server.go:1083] Started kubelet
Jan 09 15:47:05 node-2 kubelet[8323]: E0109 15:47:05.086488    8323 kubelet.go:1293] Image garbage collection failed once. Stats initialization may not have completed yet: failed to get imageFs info: unable to find data in memory cache
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.089446    8323 fs_resource_analyzer.go:64] Starting FS ResourceAnalyzer
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.089503    8323 status_manager.go:152] Starting to sync pod status with apiserver
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.089548    8323 kubelet.go:1805] Starting kubelet main sync loop.
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.089608    8323 kubelet.go:1822] skipping pod synchronization - [container runtime status check may not have completed yet, PLEG is not healthy: pleg has yet to be successful]
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.091013    8323 server.go:144] Starting to listen on 0.0.0.0:10250
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.097007    8323 volume_manager.go:243] Starting Kubelet Volume Manager
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.099017    8323 server.go:350] Adding debug handlers to kubelet server.
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.105222    8323 desired_state_of_world_populator.go:130] Desired state populator starts to run
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.189855    8323 kubelet.go:1822] skipping pod synchronization - container runtime status check may not have completed yet
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.200505    8323 kuberuntime_manager.go:924] updating runtime config through cri with podcidr 10.244.2.0/24
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.200725    8323 kubelet_node_status.go:286] Setting node annotation to enable volume controller attach/detach
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.203281    8323 docker_service.go:353] docker cri received runtime config &RuntimeConfig{NetworkConfig:&NetworkConfig{PodCidr:10.244.2.0/24,},}
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.203573    8323 kubelet_network.go:77] Setting Pod CIDR:  -> 10.244.2.0/24
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.210877    8323 kubelet_node_status.go:72] Attempting to register node node-2
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.225148    8323 kubelet_node_status.go:114] Node node-2 was previously registered
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.225332    8323 kubelet_node_status.go:75] Successfully registered node node-2
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.235846    8323 setters.go:521] Node became not ready: {Type:Ready Status:False LastHeartbeatTime:2021-01-09 15:47:05.235728799 +0800 CST m=+20.589261855 LastTransitionTime:2021-01-09 15:47:05.235728799 +0800 CST m=+20.589261855 Reason:KubeletNotReady Message:container runtime status check may not have completed yet}
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.257154    8323 cpu_manager.go:155] [cpumanager] starting with none policy
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.257198    8323 cpu_manager.go:156] [cpumanager] reconciling every 10s
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.257218    8323 policy_none.go:42] [cpumanager] none policy: Start
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.265979    8323 plugin_manager.go:116] Starting Kubelet Plugin Manager
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.406755    8323 reconciler.go:203] operationExecutor.VerifyControllerAttachedVolume started for volume "lib-modules" (UniqueName: "kubernetes.io/host-path/ae2dd8ef-f385-4471-9a65-6f6a0b3478b9-lib-modules") pod "kube-proxy-949bj" (UID: "ae2dd8ef-f385-4471-9a65-6f6a0b3478b9")
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.406810    8323 reconciler.go:203] operationExecutor.VerifyControllerAttachedVolume started for volume "kube-proxy-token-j6rtp" (UniqueName: "kubernetes.io/secret/ae2dd8ef-f385-4471-9a65-6f6a0b3478b9-kube-proxy-token-j6rtp") pod "kube-proxy-949bj" (UID: "ae2dd8ef-f385-4471-9a65-6f6a0b3478b9")
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.406848    8323 reconciler.go:203] operationExecutor.VerifyControllerAttachedVolume started for volume "run" (UniqueName: "kubernetes.io/host-path/40c493dd-7cc4-4d76-a395-8b3feb92c648-run") pod "kube-flannel-ds-946hf" (UID: "40c493dd-7cc4-4d76-a395-8b3feb92c648")
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.406879    8323 reconciler.go:203] operationExecutor.VerifyControllerAttachedVolume started for volume "cni" (UniqueName: "kubernetes.io/host-path/40c493dd-7cc4-4d76-a395-8b3feb92c648-cni") pod "kube-flannel-ds-946hf" (UID: "40c493dd-7cc4-4d76-a395-8b3feb92c648")
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.406910    8323 reconciler.go:203] operationExecutor.VerifyControllerAttachedVolume started for volume "flannel-cfg" (UniqueName: "kubernetes.io/configmap/40c493dd-7cc4-4d76-a395-8b3feb92c648-flannel-cfg") pod "kube-flannel-ds-946hf" (UID: "40c493dd-7cc4-4d76-a395-8b3feb92c648")
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.406951    8323 reconciler.go:203] operationExecutor.VerifyControllerAttachedVolume started for volume "flannel-token-h5h2l" (UniqueName: "kubernetes.io/secret/40c493dd-7cc4-4d76-a395-8b3feb92c648-flannel-token-h5h2l") pod "kube-flannel-ds-946hf" (UID: "40c493dd-7cc4-4d76-a395-8b3feb92c648")
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.406979    8323 reconciler.go:203] operationExecutor.VerifyControllerAttachedVolume started for volume "kube-proxy" (UniqueName: "kubernetes.io/configmap/ae2dd8ef-f385-4471-9a65-6f6a0b3478b9-kube-proxy") pod "kube-proxy-949bj" (UID: "ae2dd8ef-f385-4471-9a65-6f6a0b3478b9")
Jan 09 15:47:05 node-2 kubelet[8323]: I0109 15:47:05.407007    8323 reconciler.go:203] operationExecutor.VerifyControllerAttachedVolume started for volume "xtables-lock" (UniqueName: "kubernetes.io/host-path/ae2dd8ef-f385-4471-9a65-6f6a0b3478b9-xtables-lock") pod "kube-proxy-949bj" (UID: "ae2dd8ef-f385-4471-9a65-6f6a0b3478b9")
```

### (15). master查看所有node状态

```
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   26m   v1.15.0
node-1   Ready    <none>   12m   v1.15.0
node-2   Ready    <none>   11m   v1.15.0
```

### (16). 测试Kubernetes集群
```
[root@master ~]# kubectl create deployment nginx --image=nginx
deployment.apps/nginx created

[root@master ~]# kubectl expose deployment nginx --port=80 --type=NodePort
service/nginx exposed


# 副本扩容
[root@master ~]# kubectl scale deployment nginx --replicas=3
	deployment.extensions/nginx scaled

# 查看pod
[root@master ~]# kubectl get pod,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-554b9c67f9-2zpcx   1/1     Running   0          2m40s
pod/nginx-554b9c67f9-rpvfv   1/1     Running   0          2m40s
pod/nginx-554b9c67f9-xq9hj   1/1     Running   0          12m

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.1.0.1       <none>        443/TCP        88m
service/nginx        NodePort    10.1.209.212   <none>        80:31990/TCP   11m

# 访问nginx端口是否成功
[root@master ~]#  curl http://node-1:31990
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### (17). 安装WebUI

["kubernetes-dashboard.yml"](/assets/k8s/kubernetes-dashboard.yml)


```
# 应用配置
[root@master ~]# kubectl apply -f kubernetes-dashboard.yml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created

# 查看状态
[root@master ~]# kubectl get pods -n kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-bccdc95cf-9vfbh                1/1     Running   1          120m
coredns-bccdc95cf-kw752                1/1     Running   1          120m
etcd-master                            1/1     Running   1          119m
kube-apiserver-master                  1/1     Running   1          119m
kube-controller-manager-master         1/1     Running   1          119m
kube-flannel-ds-7p25f                  1/1     Running   1          104m
kube-flannel-ds-946hf                  1/1     Running   1          104m
kube-flannel-ds-x8xhj                  1/1     Running   1          113m
kube-proxy-949bj                       1/1     Running   1          106m
kube-proxy-g7pxw                       1/1     Running   1          107m
kube-proxy-mf658                       1/1     Running   1          120m
kube-scheduler-master                  1/1     Running   1          119m
kubernetes-dashboard-fb6fb4cdd-h4wnj   1/1     Running   0          49s

# 查看所有的命名空间下的pod
[root@master ~]# kubectl get svc --all-namespaces
NAMESPACE     NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes             ClusterIP   10.1.0.1       <none>        443/TCP                  131m
default       nginx                  NodePort    10.1.209.212   <none>        80:31990/TCP             54m
kube-system   kube-dns               ClusterIP   10.1.0.10      <none>        53/UDP,53/TCP,9153/TCP   130m
kube-system   kubernetes-dashboard   NodePort    10.1.167.148   <none>        443:30001/TCP            11m

```
### (18).  创建登录账户
```
// 删除用户
// [root@master ~]# kubectl delete serviceaccount dashboard-admin -n kube-system
[root@master ~]# kubectl create serviceaccount dashboard-admin -n kube-system
serviceaccount/dashboard-admin created

// 删除用户所绑定的角色
// [root@master ~]# kubectl delete clusterrolebinding dashboard-admin   
[root@master ~]# kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
clusterrolebinding.rbac.authorization.k8s.io/dashboard-admin created

# 获取token登录
[root@master ~]# kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
Name:         dashboard-admin-token-7h5m5
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: f91c7784-007a-4542-b801-6b7b2a428164

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes

# 登录webui时需要的token
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tN2g1bTUiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZjkxYzc3ODQtMDA3YS00NTQyLWI4MDEtNmI3YjJhNDI4MTY0Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.sawDsgCfALnwo9zGJ7naE5WOt5VWKw1xEt8RJvMh3uoYyi9BnnHLb8nSZTcOazM2H1GDEhIW6wjJLPUhLg8nlmhbitwaJDlF_Eyt7IAGSs3Bc9njs_ft_5YwZukSZop3qViunZkCCziel165hMoPEzNn5Owrxgkrea5t9O55YaeZj2at2fSqq5Tmd8QnI_9P1Gn8x9UfkklFBnB5vjw89BW1VXW3LNb85zSGUdF7gcT_TuNFdHwMYfJ4DTy1o9C7gA4_HIVQnVuNnSKNP02avVpR3BIuu59MD1gYUQe6QyqfDgubS2tesWp8eXIlYxkq7k97OOu1jxios5oaPpfcQQ


# 查看所有运行的pods
[root@master ~]# kubectl get pod --all-namespaces
```
### (19). WebUI
!["K8S WebUI Login"](/assets/k8s/imgs/k8s-webui-login.jpg)

!["K8S WebUI Home"](/assets/k8s/imgs/k8s-webui-home.png)
