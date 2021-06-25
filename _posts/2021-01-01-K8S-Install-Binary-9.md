---
layout: post
title: 'Kubernetes 二进制安装之DNS解析(九)'
date: 2021-01-01
author: 李新
tags: K8S二进制安装
---

### (1). 获取kube-dns.yml文件
> https://github.com/kubernetes/kubernetes/tree/release-1.9/cluster/addons/dns   
> 建议提前拉取镜像

["kube-dns.yaml下载"](/assets/k8s/kube-dns.yaml)

```
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.0.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    addonmanager.kubernetes.io/mode: EnsureExists
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  # replicas: not specified here:
  # 1. In order to make Addon Manager do not reconcile this replicas parameter.
  # 2. Default is 1.
  # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    rollingUpdate:
      maxSurge: 10%
      maxUnavailable: 0
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      volumes:
      - name: kube-dns-config
        configMap:
          name: kube-dns
          optional: true
      containers:
      - name: kubedns
        image: sapcc/k8s-dns-kube-dns-amd64:1.14.10
        resources:
          # TODO: Set memory limits when we've profiled the container for large
          # clusters, then set request = limit to keep this container in
          # guaranteed class. Currently, this container falls into the
          # "burstable" category so the kubelet doesn't backoff from restarting it.
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        livenessProbe:
          httpGet:
            path: /healthcheck/kubedns
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8081
            scheme: HTTP
          # we poll on pod startup for the Kubernetes master service and
          # only setup the /readiness HTTP server once that's available.
          initialDelaySeconds: 3
          timeoutSeconds: 5
        args:
        - --domain=cluster.local.
        - --dns-port=10053
        - --config-dir=/kube-dns-config
        - --v=2
        env:
        - name: PROMETHEUS_PORT
          value: "10055"
        ports:
        - containerPort: 10053
          name: dns-local
          protocol: UDP
        - containerPort: 10053
          name: dns-tcp-local
          protocol: TCP
        - containerPort: 10055
          name: metrics
          protocol: TCP
        volumeMounts:
        - name: kube-dns-config
          mountPath: /kube-dns-config
      - name: dnsmasq
        image: sapcc/k8s-dns-dnsmasq-nanny-amd64:1.14.10
        livenessProbe:
          httpGet:
            path: /healthcheck/dnsmasq
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        args:
        - -v=2
        - -logtostderr
        - -configDir=/etc/k8s/dns/dnsmasq-nanny
        - -restartDnsmasq=true
        - --
        - -k
        - --cache-size=1000
        - --no-negcache
        - --log-facility=-
        - --server=/cluster.local/127.0.0.1#10053
        - --server=/in-addr.arpa/127.0.0.1#10053
        - --server=/ip6.arpa/127.0.0.1#10053
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        # see: https://github.com/kubernetes/kubernetes/issues/29055 for details
        resources:
          requests:
            cpu: 150m
            memory: 20Mi
        volumeMounts:
        - name: kube-dns-config
          mountPath: /etc/k8s/dns/dnsmasq-nanny
      - name: sidecar
        image: sapcc/k8s-dns-sidecar-amd64:1.14.10
        livenessProbe:
          httpGet:
            path: /metrics
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        args:
        - --v=2
        - --logtostderr
        - --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local,5,SRV
        - --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local,5,SRV
        ports:
        - containerPort: 10054
          name: metrics
          protocol: TCP
        resources:
          requests:
            memory: 20Mi
            cpu: 10m
      dnsPolicy: Default  # Don't use cluster DNS.
      serviceAccountName: kube-dns
```
### (2). 部署kube-dns
```
[root@master ~]# kubectl apply -f kube-dns.yml
```
### (3). 创建Deploy(busybox.yml)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh","-ce","tail -f /dev/null"]
```
### (4). 部署busybox
```
[root@master ~]# kubectl apply -f busybox.yml
```
### (5).  验证
```
# 1. 查看:pod/service
[root@master ~]# kubectl get pods,svc -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP            NODE            NOMINATED NODE   READINESS GATES
pod/busybox-deployment-74c549f577-rwl2w   1/1     Running   0          39m     172.20.28.2   10.211.55.102   <none>           <none>
pod/nginx-6799fc88d8-mx7t6                1/1     Running   0          5m54s   172.20.97.2   10.211.55.101   <none>           <none>
pod/nginx-6799fc88d8-n4t5j                1/1     Running   0          25m     172.20.28.4   10.211.55.102   <none>           <none>
pod/nginx-6799fc88d8-shrn8                1/1     Running   0          5m54s   172.20.97.3   10.211.55.101   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP        21h   <none>
service/nginx        NodePort    10.0.0.142   <none>        80:36826/TCP   25m   app=nginx


# 2. 进入(busybox-deployment-74c549f577-rwl2w)容器.
# 测试:ping nginx
[root@master ~]# kubectl exec -it busybox-deployment-74c549f577-rwl2w -- sh

# ********************************************************
# 尝试ping nginx域名
# 只能解析service(name),居然不能解析pod(name),意味着,在Pod内部要使用内部域名,就必须得要用上:Service组件.
# ********************************************************
/ # ping nginx
PING nginx (10.0.0.142): 56 data bytes
64 bytes from 10.0.0.142: seq=0 ttl=64 time=0.052 ms

# 查询路由信息
/ # nslookup nginx
Server:         10.0.0.2
Address:        10.0.0.2:53

Non-authoritative answer:
# nginx对应的域名信息
Name:   nginx.default.svc.cluster.local
Address: 10.0.0.142
```