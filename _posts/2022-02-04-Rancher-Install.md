---
layout: post
title: 'Rancher安装' 
date: 2022-02-03
author: 李新
tags:  Rancher
---

### (1). 概述

### (2). 机器准备

|  机器名称           |           IP  |           角色    |
|  ----              |         ----  |         ----     |
| rancher-server     |  172.30.50.16 |   rancher server | 
| rancher-master     |  172.30.50.17 |   rancher master | 
| rancher-node-1     |  172.30.50.18 |   rancher node   | 

### (3). Docker安装
```
# 以下命令所有的机器都要执行.
# 安装docker
yum -y install yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y  docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker

# 创建app用户,并添加用户到docker组
gpasswd -a app docker
systemctl restart docker

# 防火墙开放2380端口
firewall-cmd  --zone=public --add-port=2380/tcp --permanent
firewall-cmd  --reload

yum -y install wget
```
### (5). Rancher Server安装前准备
```
# 创建rancher server数据目录
[root@rancher-server ~]# mkdir -p /opt/rancher_home/rancher
[root@rancher-server ~]# mkdir -p /opt/rancher_home/log/auditlog
[root@rancher-server ~]# chown -R app:app /opt

# 防火墙开启端口
[root@rancher-server ~]# firewall-cmd  --zone=public --add-port=80/tcp --permanent
[root@rancher-server ~]# firewall-cmd  --zone=public --add-port=443/tcp --permanent
[root@rancher-server ~]# firewall-cmd  --zone=public --add-port=2380/tcp --permanent
[root@rancher-server ~]# firewall-cmd  --reload
```
### (6). Rancher Server镜像拉取
```
[app@rancher-1 ~]$ docker pull rancher/rancher
Using default tag: latest
latest: Pulling from rancher/rancher
a3009803982d: Pull complete
cf9e817c5d35: Pull complete
b380663f8ccc: Pull complete
1ca0e2238656: Pull complete
65086cb458c9: Pull complete
c6d25608690f: Pull complete
6c8ad6da7ce2: Pull complete
6a6940e66f68: Pull complete
b115b1ef2b5b: Pull complete
b4b03dbaa949: Pull complete
aef7deb59b77: Pull complete
0bbf7579a568: Pull complete
eaa5d6336f95: Pull complete
608f536609b9: Pull complete
fcaf65f7937c: Pull complete
e4ea550002d9: Pull complete
9d698b9289d2: Pull complete
caa4144aedf1: Pull complete
Digest: sha256:f411ee37efa38d7891c11ecdd5c60ca73eb03dcd32296678af808f6b4ecccfff
Status: Downloaded newer image for rancher/rancher:latest
docker.io/rancher/rancher:latest
```
### (7). 运行Rancher Server
```
[app@rancher-1 ~]$ docker run -d  --privileged --restart=unless-stopped -p 80:80 -p 443:443 \
-e CATTLE_SYSTEM_CATALOG=bundled \
-e AUDIT_LEVEL=3 \
-v /opt/rancher_home/rancher:/var/lib/rancher \
-v /opt/rancher_home/log/auditlog:/var/log/auditlog \
--name rancher rancher/rancher
```
### (8). 查看运行的Rancher Server容器
[app@rancher-server ~]$ docker ps
CONTAINER ID   IMAGE                     COMMAND           CREATED         STATUS         PORTS                                                                      NAMES
7e2fe3d86726   rancher/rancher:v2.4.17   "entrypoint.sh"   4 minutes ago   Up 4 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   rancher
### (9). Rancher Server登录界面
!["Rancher设置密码页面"](/assets/rancher/imgs/rancher-pwd.png)   
!["Rancher配置URL"](/assets/rancher/imgs/rancher-set-server-url.png)
!["Rancher首页"](/assets/rancher/imgs/rancher-home.png)
### (10). Rancher Server安装K8S
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster.png)
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster-2.png)
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster-3.png)
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster-4.png)
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster-5.png)
### (11). Rancher Server查看Rancher Master加入集群信息
!["Rancher Server查看Rancher Master加入集群信息"](/assets/rancher/imgs/rancher-master.png)
### (12). Rancher Master加入集群
```
# 注意要以root身份运行
[root@rancher-master ~]# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.4.17 --server https://172.30.50.16 --token 5s22qsk5qrm4mxfkj8k6hcf9fq69z6nh2bcsr55wkdxxg5lgcp2zz4 --ca-checksum 9fe2422f4eb5b94ab26206e09c324d3e4a07649b58cf8a7b2ad0c3510aa3d813 --etcd --controlplane
```
### (13). Rancher Server查看Rancher Work加入集群信息
!["Rancher Server查看Rancher Work加入集群信息"](/assets/rancher/imgs/rancher-work.png)
### (14). Rancher Work加入集群
```
[root@rancher-node1 ~]# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.4.17 --server https://172.30.50.16 --token hsxgspcld9855xz52j4xjcfdcxtp9mqkznl44f6khnbcl2l25hf5mx --ca-checksum 9fe2422f4eb5b94ab26206e09c324d3e4a07649b58cf8a7b2ad0c3510aa3d813 --worker
```
### (15). kubectl配置
```
[root@rancher-master ~]# wget http://rancher-mirror.cnrancher.com/kubectl/v1.18.20/linux-amd64-v1.18.20-kubectl
[root@rancher-master ~]# mv linux-amd64-v1.18.20-kubectl /usr/sbin/kubectl
[root@rancher-master ~]# chmod 777 /usr/sbin/kubectl

# 切换到app用户
[app@rancher-master ~]$ mkdir .kube

cat >>  ~/.kube/config << EOF
 apiVersion: v1
 kind: Config
 clusters:
 - name: "gerp-rancher"
   cluster:
     server: "https://172.30.50.16/k8s/clusters/c-6j4lz"
     certificate-authority-data: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJoekNDQ\
       VM2Z0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQTdNUnd3R2dZRFZRUUtFeE5rZVc1aGJXbGoKY\
       kdsemRHVnVaWEl0YjNKbk1Sc3dHUVlEVlFRREV4SmtlVzVoYldsamJHbHpkR1Z1WlhJdFkyRXdIa\
       GNOTWpJdwpNakExTURJeU5EVTRXaGNOTXpJd01qQXpNREl5TkRVNFdqQTdNUnd3R2dZRFZRUUtFe\
       E5rZVc1aGJXbGpiR2x6CmRHVnVaWEl0YjNKbk1Sc3dHUVlEVlFRREV4SmtlVzVoYldsamJHbHpkR\
       1Z1WlhJdFkyRXdXVEFUQmdjcWhrak8KUFFJQkJnZ3Foa2pPUFFNQkJ3TkNBQVN0Z25OUGE3bXhFd\
       m1EK2ZUaUhsKzV2TTQ2bllncWEzUVNYQnI4WWFyUQpYb0lwK00rSXZLSXVDT3ZiSFpNUDZETDVOM\
       llReGVGTFZ6bTRVc3NiYjFNQW95TXdJVEFPQmdOVkhROEJBZjhFCkJBTUNBcVF3RHdZRFZSMFRBU\
       UgvQkFVd0F3RUIvekFLQmdncWhrak9QUVFEQWdOSEFEQkVBaUE4RW1VRWliSzMKTWxIQ2tVNmdUS\
       HY2K0pCV0F4UU1HQkZGK2JhczdrL1NUZ0lnWU82QjUxaU5VZXdPMUYrS0Rtb25GQTJLUngwMwpRV\
       2RFVDY1YWtLMVR0Q0E9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0="
 - name: "gerp-rancher-rancher-master"
   cluster:
     server: "https://172.30.50.17:6443"
     certificate-authority-data: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN3akNDQ\
       WFxZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFTTVJBd0RnWURWUVFERXdkcmRXSmwKT\
       FdOaE1CNFhEVEl5TURJd05UQXlORGd5TUZvWERUTXlNREl3TXpBeU5EZ3lNRm93RWpFUU1BNEdBM\
       VVFQXhNSAphM1ZpWlMxallUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ\
       0VCQU5iamxKODd4cDRKCkhONy9XenpxeVNEWm41RHZwaDk0VlltanVxdmRVdkp5N1FaSGdJVTRwT\
       TUrbnp1ZmdFMURsYStkMk5lSzNpTGIKV2FJdEJEUGNScXRwVEM3bFI0SmU4QzBBKzdOcEwzc0Q3T\
       lE1ZjQ2bDVNRVJCUm16QzlldWM4MFM4emFmMmd1cgpPQW9YbG9JMVRFdERXYWtxaWZSa1ZUN01WL\
       3hYbTZORkZIWjZjKytwSU1JMW1KdmtuWTdnV0RnWUEwRFREbHQxCjlmTnc1RjZDaklHMnJUeEVLb\
       mlPVUZUbWR5NFdIS202anovdGdQTmhTMXB3bVYxOHZCSytvOHQvUEVoUzhEWFoKNlBOWTZnMXJNU\
       E1zaG00VnBqUjFiM0pLWjgyUkZ1WkJlQnV6bkE0ZU9QRmY5eXd3OEFjWno0WHh0WENOd3QvQQp2T\
       1lFbGx1SEc2VUNBd0VBQWFNak1DRXdEZ1lEVlIwUEFRSC9CQVFEQWdLa01BOEdBMVVkRXdFQi93U\
       UZNQU1CCkFmOHdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBQ3B6N0RZdWNsT1h1SnlHdXZnSmtuS\
       ml0Y1poTDdIUTduMmEKM0dlUE51blR1MjhyQXpmbHFFNFZzMDFvVXBaM1JicWdScmVETHhWcTMwb\
       Fg5dnF3R3kwZ1pkWFpZKy90WUh1TwpBZDhoMVlrQmUwTlRIcW9vLzVPMVpGUXdqTUZrRlpVNnhjU\
       zFmcTNJSlZIYlJmMzV5cjFycnJrdDZyak9zZjZICnJDMzF0K29TSnR6OHRkNTRLbEthTWphVW03Z\
       S9waGtVdStLd3VVK3FURlNqaGVjMEsrZVp2c1ZTdlk3dUtDZEYKSFdOQnVyN2kweTQyKzc2WlJ1c\
       VdqSm9lYlA1bTBSdlZQeFF3aWpsMjFTNUN6YW5jL2VGa1E3YmJVbWpiS0V4RQpVeVR3NmRnN0JCc\
       0xDS3h2c2JnazQvdXhMVWRqZUU4ZENXcUphWEFQQ1c2a1l1TzEyK0E9Ci0tLS0tRU5EIENFUlRJR\
       klDQVRFLS0tLS0K"
 
 users:
 - name: "gerp-rancher"
   user:
     token: "kubeconfig-user-pwfzk.c-6j4lz:kwwrx9tknh2bsx5rdv9m5dd46g5pb5l4sj56qh8ldd776wvqrsgcfj"
 
 
 contexts:
 - name: "gerp-rancher"
   context:
     user: "gerp-rancher"
     cluster: "gerp-rancher"
 - name: "gerp-rancher-rancher-master"
   context:
     user: "gerp-rancher"
     cluster: "gerp-rancher-rancher-master"
 
 current-context: "gerp-rancher"
EOF
```
### (16). 总结
尝试了N次,国内的网络感觉安装Rancher都有问题.