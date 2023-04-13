---
layout: post
title: 'Sealos安装K8S集群' 
date: 2023-04-08
author: 李新
tags:  K8S Sealos
---

### (1). 概述
这段时间在弄服务器相关的东西,以前是用Rancher安装K8S,前几天和一朋友聊天,得知一个新的工具,可以在国内快速装K8S的新秀,即:Sealos. 

### (2). 机器准备

|  IP            | 机器名称   |  部署组件      | 
|  ----          | ----     | ----           |
| 192.168.1.11   |  k8s-master  | kube-apiserver,kube-controller-manger,kube-scheduler,etcd |
| 192.168.1.12   |  k8s-node1  | kubelet,kube-proxy,docker,etcd |
| 192.168.1.13   |  k8s-node2  | kubelet,kube-proxy,docker,etcd |

### (3). 安装先决条件
> 1. 计算机的名称不能有下划线.   
> 2. 要求关闭防火墙.  
> 3. 要求关闭selinux  

### (4). 安装sealos
```
[root@k8s-master ~]# cat > /etc/yum.repos.d/labring.repo << EOF
[fury]
name=labring Yum Repo
baseurl=https://yum.fury.io/labring/
enabled=1
gpgcheck=0
EOF


[root@k8s-master ~]# yum clean all
[root@k8s-master ~]# yum -y  install sealos
```
### (5). 安装k8s集群
```
[app@k8s-master ~]$ sudo sealos run labring/kubernetes:v1.25.0 labring/helm:v3.8.2 labring/calico:v3.24.1 \
--masters 192.168.1.11 \
--nodes 192.168.1.12,192.168.1.13 \
-p 88888888

2023-04-13T13:47:33 info Start to create a new cluster: master [192.168.1.11], worker [192.168.1.12 192.168.1.13], registry 192.168.1.11
2023-04-13T13:47:33 info Executing pipeline Check in CreateProcessor.
2023-04-13T13:47:33 info checker:hostname [192.168.1.11:22 192.168.1.12:22 192.168.1.13:22]
2023-04-13T13:47:33 info checker:timeSync [192.168.1.11:22 192.168.1.12:22 192.168.1.13:22]
2023-04-13T13:47:33 info Executing pipeline PreProcess in CreateProcessor.
Resolving "labring/kubernetes" using unqualified-search registries (/etc/containers/registries.conf)
Trying to pull docker.io/labring/kubernetes:v1.25.0...
Getting image source signatures
Copying blob 583917f417c4 done
Copying blob 4013845ba3fe done
Copying blob 88af23a6a8b4 done
Copying blob 0ad330619635 done
Copying config aa502e66bc done
Writing manifest to image destination
Storing signatures
Resolving "labring/helm" using unqualified-search registries (/etc/containers/registries.conf)
Trying to pull docker.io/labring/helm:v3.8.2...
Getting image source signatures
Copying blob 53a6eade9e7e done
Copying config 1123e8b4b4 done
Writing manifest to image destination
Storing signatures
Resolving "labring/calico" using unqualified-search registries (/etc/containers/registries.conf)
Trying to pull docker.io/labring/calico:v3.24.1...
Getting image source signatures
Copying blob 740f1fdd328f done
Copying config 6bbbb5354a done
Writing manifest to image destination
Storing signatures
2023-04-13T13:51:03 info Executing pipeline RunConfig in CreateProcessor.
2023-04-13T13:51:03 info Executing pipeline MountRootfs in CreateProcessor.
2023-04-13T13:51:14 info Executing pipeline MirrorRegistry in CreateProcessor.
 INFO [2023-04-13 13:51:14] >> untar-registry.sh was not found, skip decompression registry or execute sealos run labring/registry:untar
 INFO [2023-04-13 13:51:14] >> untar-registry.sh was not found, skip decompression registry or execute sealos run labring/registry:untar
 INFO [2023-04-13 13:51:15] >> untar-registry.sh was not found, skip decompression registry or execute sealos run labring/registry:untar
2023-04-13T13:51:15 info Executing pipeline Bootstrap in CreateProcessor
192.168.1.13:22 which: no docker in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin)
192.168.1.13:22  WARN [2023-04-13 13:51:15] >> Replace disable_apparmor = false to disable_apparmor = true
192.168.1.13:22  INFO [2023-04-13 13:51:15] >> check root,port,cri success
192.168.1.12:22 which: no docker in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin)
192.168.1.12:22  WARN [2023-04-13 13:51:15] >> Replace disable_apparmor = false to disable_apparmor = true
192.168.1.12:22  INFO [2023-04-13 13:51:15] >> check root,port,cri success
which: no docker in (/sbin:/bin:/usr/sbin:/usr/bin)
 WARN [2023-04-13 13:51:15] >> Replace disable_apparmor = false to disable_apparmor = true
 INFO [2023-04-13 13:51:15] >> check root,port,cri success
2023-04-13T13:51:15 info domain sealos.hub:192.168.1.11 append success
192.168.1.12:22 2023-04-13T13:51:15 info domain sealos.hub:192.168.1.11 append success
192.168.1.13:22 2023-04-13T13:51:15 info domain sealos.hub:192.168.1.11 append success
Created symlink from /etc/systemd/system/multi-user.target.wants/registry.service to /etc/systemd/system/registry.service.
 INFO [2023-04-13 13:51:16] >> Health check registry!
 INFO [2023-04-13 13:51:16] >> registry is running
 INFO [2023-04-13 13:51:16] >> init registry success
192.168.1.12:22 Created symlink from /etc/systemd/system/multi-user.target.wants/containerd.service to /etc/systemd/system/containerd.service.
192.168.1.13:22 Created symlink from /etc/systemd/system/multi-user.target.wants/containerd.service to /etc/systemd/system/containerd.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/containerd.service to /etc/systemd/system/containerd.service.
192.168.1.12:22  INFO [2023-04-13 13:51:19] >> Health check containerd!
192.168.1.12:22  INFO [2023-04-13 13:51:19] >> containerd is running
192.168.1.12:22  INFO [2023-04-13 13:51:19] >> init containerd success
192.168.1.13:22  INFO [2023-04-13 13:51:19] >> Health check containerd!
192.168.1.13:22  INFO [2023-04-13 13:51:19] >> containerd is running
192.168.1.13:22  INFO [2023-04-13 13:51:19] >> init containerd success
192.168.1.12:22 Created symlink from /etc/systemd/system/multi-user.target.wants/image-cri-shim.service to /etc/systemd/system/image-cri-shim.service.
192.168.1.13:22 Created symlink from /etc/systemd/system/multi-user.target.wants/image-cri-shim.service to /etc/systemd/system/image-cri-shim.service.
192.168.1.12:22  INFO [2023-04-13 13:51:19] >> Health check image-cri-shim!
192.168.1.12:22  INFO [2023-04-13 13:51:19] >> image-cri-shim is running
192.168.1.12:22  INFO [2023-04-13 13:51:19] >> init shim success
192.168.1.13:22  INFO [2023-04-13 13:51:19] >> Health check image-cri-shim!
192.168.1.13:22  INFO [2023-04-13 13:51:19] >> image-cri-shim is running
192.168.1.13:22  INFO [2023-04-13 13:51:19] >> init shim success
 INFO [2023-04-13 13:51:19] >> Health check containerd!
 INFO [2023-04-13 13:51:19] >> containerd is running
 INFO [2023-04-13 13:51:19] >> init containerd success
Created symlink from /etc/systemd/system/multi-user.target.wants/image-cri-shim.service to /etc/systemd/system/image-cri-shim.service.
192.168.1.12:22 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
192.168.1.12:22 ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.12:22 * Applying /usr/lib/sysctl.d/00-system.conf ...
192.168.1.12:22 net.bridge.bridge-nf-call-ip6tables = 0
192.168.1.12:22 net.bridge.bridge-nf-call-iptables = 0
192.168.1.12:22 net.bridge.bridge-nf-call-arptables = 0
192.168.1.12:22 * Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
192.168.1.12:22 kernel.yama.ptrace_scope = 0
192.168.1.12:22 * Applying /usr/lib/sysctl.d/50-default.conf ...
192.168.1.12:22 kernel.sysrq = 16
192.168.1.12:22 kernel.core_uses_pid = 1
192.168.1.12:22 kernel.kptr_restrict = 1
192.168.1.12:22 net.ipv4.conf.default.rp_filter = 1
192.168.1.12:22 net.ipv4.conf.all.rp_filter = 1
192.168.1.12:22 net.ipv4.conf.default.accept_source_route = 0
192.168.1.12:22 net.ipv4.conf.all.accept_source_route = 0
192.168.1.12:22 net.ipv4.conf.default.promote_secondaries = 1
192.168.1.12:22 net.ipv4.conf.all.promote_secondaries = 1
192.168.1.12:22 fs.protected_hardlinks = 1
192.168.1.12:22 fs.protected_symlinks = 1
192.168.1.12:22 * Applying /etc/sysctl.d/99-sysctl.conf ...
192.168.1.12:22 * Applying /etc/sysctl.d/k8s.conf ...
192.168.1.12:22 net.bridge.bridge-nf-call-ip6tables = 1
192.168.1.12:22 net.bridge.bridge-nf-call-iptables = 1
192.168.1.12:22 net.ipv4.conf.all.rp_filter = 0
192.168.1.12:22 * Applying /etc/sysctl.conf ...
192.168.1.13:22 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
192.168.1.13:22 ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.12:22 net.ipv4.ip_forward = 1
192.168.1.13:22 * Applying /usr/lib/sysctl.d/00-system.conf ...
192.168.1.13:22 net.bridge.bridge-nf-call-ip6tables = 0
192.168.1.13:22 net.bridge.bridge-nf-call-iptables = 0
192.168.1.13:22 net.bridge.bridge-nf-call-arptables = 0
192.168.1.13:22 * Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
192.168.1.13:22 kernel.yama.ptrace_scope = 0
192.168.1.13:22 * Applying /usr/lib/sysctl.d/50-default.conf ...
192.168.1.13:22 kernel.sysrq = 16
192.168.1.13:22 kernel.core_uses_pid = 1
192.168.1.13:22 kernel.kptr_restrict = 1
192.168.1.13:22 net.ipv4.conf.default.rp_filter = 1
192.168.1.13:22 net.ipv4.conf.all.rp_filter = 1
192.168.1.13:22 net.ipv4.conf.default.accept_source_route = 0
192.168.1.13:22 net.ipv4.conf.all.accept_source_route = 0
192.168.1.13:22 net.ipv4.conf.default.promote_secondaries = 1
192.168.1.13:22 net.ipv4.conf.all.promote_secondaries = 1
192.168.1.13:22 fs.protected_hardlinks = 1
192.168.1.13:22 fs.protected_symlinks = 1
192.168.1.13:22 * Applying /etc/sysctl.d/99-sysctl.conf ...
192.168.1.13:22 * Applying /etc/sysctl.d/k8s.conf ...
192.168.1.13:22 net.bridge.bridge-nf-call-ip6tables = 1
192.168.1.13:22 net.bridge.bridge-nf-call-iptables = 1
192.168.1.13:22 net.ipv4.conf.all.rp_filter = 0
192.168.1.13:22 * Applying /etc/sysctl.conf ...
192.168.1.13:22 net.ipv4.ip_forward = 1
 INFO [2023-04-13 13:51:19] >> Health check image-cri-shim!
 INFO [2023-04-13 13:51:19] >> image-cri-shim is running
 INFO [2023-04-13 13:51:19] >> init shim success
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
* Applying /usr/lib/sysctl.d/00-system.conf ...
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
* Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
kernel.yama.ptrace_scope = 0
* Applying /usr/lib/sysctl.d/50-default.conf ...
kernel.sysrq = 16
kernel.core_uses_pid = 1
kernel.kptr_restrict = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.promote_secondaries = 1
net.ipv4.conf.all.promote_secondaries = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
* Applying /etc/sysctl.d/99-sysctl.conf ...
* Applying /etc/sysctl.d/k8s.conf ...
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.conf.all.rp_filter = 0
* Applying /etc/sysctl.conf ...
net.ipv4.ip_forward = 1
192.168.1.13:22 Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.
192.168.1.13:22  INFO [2023-04-13 13:51:19] >> init kubelet success
192.168.1.13:22  INFO [2023-04-13 13:51:19] >> init rootfs success
192.168.1.12:22 Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.
192.168.1.12:22  INFO [2023-04-13 13:51:19] >> init kubelet success
192.168.1.12:22  INFO [2023-04-13 13:51:19] >> init rootfs success
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.
 INFO [2023-04-13 13:51:20] >> init kubelet success
 INFO [2023-04-13 13:51:20] >> init rootfs success
2023-04-13T13:51:20 info Executing pipeline Init in CreateProcessor.
2023-04-13T13:51:20 info start to copy kubeadm config to master0
2023-04-13T13:51:20 info start to generate cert and kubeConfig...
2023-04-13T13:51:20 info start to generator cert and copy to masters...
2023-04-13T13:51:20 info apiserver altNames : {map[apiserver.cluster.local:apiserver.cluster.local k8s-master:k8s-master kubernetes:kubernetes kubernetes.default:kubernetes.default kubernetes.default.svc:kubernetes.default.svc kubernetes.default.svc.cluster.local:kubernetes.default.svc.cluster.local localhost:localhost] map[10.103.97.2:10.103.97.2 10.96.0.1:10.96.0.1 127.0.0.1:127.0.0.1 192.168.1.11:192.168.1.11]}
2023-04-13T13:51:20 info Etcd altnames : {map[k8s-master:k8s-master localhost:localhost] map[127.0.0.1:127.0.0.1 192.168.1.11:192.168.1.11 ::1:::1]}, commonName : k8s-master
2023-04-13T13:51:22 info start to copy etc pki files to masters
2023-04-13T13:51:22 info start to copy etc pki files to masters
2023-04-13T13:51:22 info start to create kubeconfig...
2023-04-13T13:51:23 info start to copy kubeconfig files to masters
2023-04-13T13:51:23 info start to copy static files to masters
2023-04-13T13:51:23 info start to init master0...
2023-04-13T13:51:24 info domain apiserver.cluster.local:192.168.1.11 append success
W0413 13:51:24.036814    2165 initconfiguration.go:119] Usage of CRI endpoints without URL scheme is deprecated and can cause kubelet errors in the future. Automatically prepending scheme "unix" to the "criSocket" with value "/run/containerd/containerd.sock". Please update your configuration!
W0413 13:51:24.036886    2165 utils.go:69] The recommended value for "healthzBindAddress" in "KubeletConfiguration" is: 127.0.0.1; the provided value is: 0.0.0.0
[init] Using Kubernetes version: v1.25.0
[preflight] Running pre-flight checks
        [WARNING FileExisting-socat]: socat not found in system path
        [WARNING Hostname]: hostname "k8s-master" could not be reached
        [WARNING Hostname]: hostname "k8s-master": lookup k8s-master on 114.114.114.114:53: no such host
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Using existing ca certificate authority
[certs] Using existing apiserver certificate and key on disk
[certs] Using existing apiserver-kubelet-client certificate and key on disk
[certs] Using existing front-proxy-ca certificate authority
[certs] Using existing front-proxy-client certificate and key on disk
[certs] Using existing etcd/ca certificate authority
[certs] Using existing etcd/server certificate and key on disk
[certs] Using existing etcd/peer certificate and key on disk
[certs] Using existing etcd/healthcheck-client certificate and key on disk
[certs] Using existing apiserver-etcd-client certificate and key on disk
[certs] Using the existing "sa" key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/admin.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/kubelet.conf"
W0413 13:51:40.980963    2165 kubeconfig.go:249] a kubeconfig file "/etc/kubernetes/controller-manager.conf" exists already but has an unexpected API Server URL: expected: https://192.168.1.11:6443, got: https://apiserver.cluster.local:6443
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/controller-manager.conf"
W0413 13:51:41.182110    2165 kubeconfig.go:249] a kubeconfig file "/etc/kubernetes/scheduler.conf" exists already but has an unexpected API Server URL: expected: https://192.168.1.11:6443, got: https://apiserver.cluster.local:6443
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/scheduler.conf"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 7.002384 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join apiserver.cluster.local:6443 --token <value withheld> \
        --discovery-token-ca-cert-hash sha256:b2aef17a5ae028e9f6a8303d0daa9aaac24bd181d773869457a3d71bb92a8103 \
        --control-plane --certificate-key <value withheld>

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join apiserver.cluster.local:6443 --token <value withheld> \
        --discovery-token-ca-cert-hash sha256:b2aef17a5ae028e9f6a8303d0daa9aaac24bd181d773869457a3d71bb92a8103
2023-04-13T13:51:49 info Executing pipeline Join in CreateProcessor.
2023-04-13T13:51:49 info [192.168.1.12:22 192.168.1.13:22] will be added as worker
2023-04-13T13:51:49 info start to get kubernetes token...
2023-04-13T13:51:52 info start to join 192.168.1.13:22 as worker
2023-04-13T13:51:52 info start to copy kubeadm join config to node: 192.168.1.13:22
2023-04-13T13:51:52 info start to join 192.168.1.12:22 as worker
2023-04-13T13:51:52 info start to copy kubeadm join config to node: 192.168.1.12:22
192.168.1.13:22 2023-04-13T13:51:53 info domain apiserver.cluster.local:10.103.97.2 append success
192.168.1.13:22 2023-04-13T13:51:53 info domain lvscare.node.ip:192.168.1.13 append success
2023-04-13T13:51:53 info run ipvs once module: 192.168.1.13:22
192.168.1.13:22l2023-04-13T13:51:53 info Trying to add route (1/1, 1074 it/s)
192.168.1.13:22 2023-04-13T13:51:53 info success to set route.(host:10.103.97.2, gateway:192.168.1.13)
2023-04-13T13:51:53 info start join node: 192.168.1.13:22
192.168.1.12:22 2023-04-13T13:51:53 info domain apiserver.cluster.local:10.103.97.2 append success
192.168.1.13:22 W0413 13:51:53.546461    2115 initconfiguration.go:119] Usage of CRI endpoints without URL scheme is deprecated and can cause kubelet errors in the future. Automatically prepending scheme "unix" to the "criSocket" with value "/run/containerd/containerd.sock". Please update your configuration!
192.168.1.13:22 [preflight] Running pre-flight checks
192.168.1.13:22         [WARNING FileExisting-socat]: socat not found in system path
192.168.1.12:22 2023-04-13T13:51:53 info domain lvscare.node.ip:192.168.1.12 append success
2023-04-13T13:51:53 info run ipvs once module: 192.168.1.12:22
192.168.1.12:22 2023-04-13T13:51:53 info Trying to add route
192.168.1.12:22 2023-04-13T13:51:53 info success to set route.(host:10.103.97.2, gateway:192.168.1.12)
2023-04-13T13:51:53 info start join node: 192.168.1.12:22
192.168.1.12:22 W0413 13:51:53.978836    2129 initconfiguration.go:119] Usage of CRI endpoints without URL scheme is deprecated and can cause kubelet errors in the future. Automatically prepending scheme "unix" to the "criSocket" with value "/run/containerd/containerd.sock". Please update your configuration!
192.168.1.12:22 [preflight] Running pre-flight checks
192.168.1.12:22         [WARNING FileExisting-socat]: socat not found in system path
192.168.1.12:22         [WARNING Hostname]: hostname "k8s-node1" could not be reached
192.168.1.12:22         [WARNING Hostname]: hostname "k8s-node1": lookup k8s-node1 on 114.114.114.114:53: no such host
192.168.1.13:22         [WARNING Hostname]: hostname "k8s-node2" could not be reached
192.168.1.13:22         [WARNING Hostname]: hostname "k8s-node2": lookup k8s-node2 on 114.114.114.114:53: no such host
192.168.1.12:22 [preflight] Reading configuration from the cluster...
192.168.1.12:22 [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
192.168.1.12:22 W0413 13:52:06.517401    2129 utils.go:69] The recommended value for "healthzBindAddress" in "KubeletConfiguration" is: 127.0.0.1; the provided value is: 0.0.0.0
192.168.1.12:22 [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
192.168.1.12:22 [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
192.168.1.12:22 [kubelet-start] Starting the kubelet
192.168.1.13:22 [preflight] Reading configuration from the cluster...
192.168.1.13:22 [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
192.168.1.13:22 W0413 13:52:06.589942    2115 utils.go:69] The recommended value for "healthzBindAddress" in "KubeletConfiguration" is: 127.0.0.1; the provided value is: 0.0.0.0
192.168.1.13:22 [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
192.168.1.13:22 [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
192.168.1.13:22 [kubelet-start] Starting the kubelet
192.168.1.12:22 [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
192.168.1.13:22 [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
192.168.1.12:22
192.168.1.12:22 This node has joined the cluster:
192.168.1.12:22 * Certificate signing request was sent to apiserver and a response was received.
192.168.1.12:22 * The Kubelet was informed of the new secure connection details.
192.168.1.12:22
192.168.1.12:22 Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
192.168.1.12:22
2023-04-13T13:52:18 info succeeded in joining 192.168.1.12:22 as worker
192.168.1.13:22
192.168.1.13:22 This node has joined the cluster:
192.168.1.13:22 * Certificate signing request was sent to apiserver and a response was received.
192.168.1.13:22 * The Kubelet was informed of the new secure connection details.
192.168.1.13:22
192.168.1.13:22 Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
192.168.1.13:22
2023-04-13T13:52:20 info succeeded in joining 192.168.1.13:22 as worker
2023-04-13T13:52:20 info start to sync lvscare static pod to node: 192.168.1.13:22 master: [192.168.1.11:6443]
2023-04-13T13:52:20 info start to sync lvscare static pod to node: 192.168.1.12:22 master: [192.168.1.11:6443]
192.168.1.13:22 2023-04-13T13:52:21 info generator lvscare static pod is success
192.168.1.12:22 2023-04-13T13:52:21 info generator lvscare static pod is success
2023-04-13T13:52:21 info Executing pipeline RunGuest in CreateProcessor.
Release "calico" does not exist. Installing it now.
NAME: calico
LAST DEPLOYED: Thu Apr 13 13:52:24 2023
NAMESPACE: tigera-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
2023-04-13T13:52:25 info succeeded in creating a new cluster, enjoy it!
2023-04-13T13:52:25 info
      ___           ___           ___           ___       ___           ___
     /\  \         /\  \         /\  \         /\__\     /\  \         /\  \
    /::\  \       /::\  \       /::\  \       /:/  /    /::\  \       /::\  \
   /:/\ \  \     /:/\:\  \     /:/\:\  \     /:/  /    /:/\:\  \     /:/\ \  \
  _\:\~\ \  \   /::\~\:\  \   /::\~\:\  \   /:/  /    /:/  \:\  \   _\:\~\ \  \
 /\ \:\ \ \__\ /:/\:\ \:\__\ /:/\:\ \:\__\ /:/__/    /:/__/ \:\__\ /\ \:\ \ \__\
 \:\ \:\ \/__/ \:\~\:\ \/__/ \/__\:\/:/  / \:\  \    \:\  \ /:/  / \:\ \:\ \/__/
  \:\ \:\__\    \:\ \:\__\        \::/  /   \:\  \    \:\  /:/  /   \:\ \:\__\
   \:\/:/  /     \:\ \/__/        /:/  /     \:\  \    \:\/:/  /     \:\/:/  /
    \::/  /       \:\__\         /:/  /       \:\__\    \::/  /       \::/  /
     \/__/         \/__/         \/__/         \/__/     \/__/         \/__/

                  Website: https://www.sealos.io/
                  Address: github.com/labring/sealos
                  Version: 4.2.0-alpha2-22baae1a
```
### (6). namespace测试
```
[app@k8s-master ~]$ sudo kubectl get ns
NAME               STATUS   AGE
calico-apiserver   Active   20m
calico-system      Active   21m
default            Active   21m
kube-node-lease    Active   21m
kube-public        Active   21m
kube-system        Active   21m
tigera-operator    Active   21m


[app@k8s-master ~]$ sudo kubectl create namespace test
namespace/test created


[app@k8s-master ~]$ sudo kubectl get ns
NAME               STATUS   AGE
calico-apiserver   Active   20m
calico-system      Active   21m
default            Active   22m
kube-node-lease    Active   22m
kube-public        Active   22m
kube-system        Active   22m
test               Active   34s
tigera-operator    Active   21m


[app@k8s-master ~]$ sudo kubectl delete namespace test
namespace "test" deleted

[app@k8s-master ~]$ sudo kubectl get ns
NAME               STATUS   AGE
calico-apiserver   Active   21m
calico-system      Active   21m
default            Active   22m
kube-node-lease    Active   22m
kube-public        Active   22m
kube-system        Active   22m
tigera-operator    Active   22m
```
### (7). 总结
额,好像是秒级安装完了,不得不说,Sealos安装k8s真的爽. 