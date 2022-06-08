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

# 关闭防火墙
service firewalld stop
systemctl disable firewalld.service


# 安装docker
yum -y install yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y  docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker

# 创建app用户,并添加用户到docker组
gpasswd -a app docker
systemctl restart docker
```
### (5). Rancher Server安装前准备
```
# 创建rancher server数据目录
[root@rancher-server ~]# mkdir -p /opt/rancher_home/rancher
[root@rancher-server ~]# mkdir -p /opt/rancher_home/log/auditlog
[root@rancher-server ~]# chown -R app:app /opt
```
### (6). 运行Rancher Server
```
[app@rancher-1 ~]$ docker run -d  --privileged --restart=unless-stopped -p 80:80 -p 443:443 \
-v /opt/rancher_home/rancher:/var/lib/rancher \
-v /opt/rancher_home/log/auditlog:/var/log/auditlog \
--name rancher rancher/rancher
```
### (7). 查看运行的Rancher Server容器

[app@rancher-server ~]$ docker ps
CONTAINER ID   IMAGE                     COMMAND           CREATED         STATUS         PORTS                                                                      NAMES
7e2fe3d86726   rancher/rancher:v2.4.17   "entrypoint.sh"   4 minutes ago   Up 4 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   rancher

### (8). Rancher Server登录界面
!["Rancher设置密码页面"](/assets/rancher/imgs/rancher-pwd.png)   
!["Rancher配置URL"](/assets/rancher/imgs/rancher-set-server-url.png)
!["Rancher首页"](/assets/rancher/imgs/rancher-home.png)
### (9). Rancher Server安装K8S
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster.png)
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster-2.png)
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster-3.png)
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster-4.png)
!["Rancher 安装K8S"](/assets/rancher/imgs/rancher-add-cluster-5.png)
### (10). Rancher Server查看Rancher Master加入集群信息
!["Rancher Server查看Rancher Master加入集群信息"](/assets/rancher/imgs/rancher-master.png)
### (11). Rancher Master加入集群
```
[root@rancher-master ~]# yum -y install wget
[root@rancher-master ~]# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.6.3 --server https://172.30.50.16 --token 295bg677s2vzxgxth2krthf5gcbd6bn42sv2ktt4ssvqjgfbrh6f82 --ca-checksum 5357adc60ce6c0ce0660f06ec9c012f77c9cf97b75c02cfb31816e6c18ae551a --etcd --controlplane
```
### (12). Rancher Server查看Rancher Work加入集群信息
!["Rancher Server查看Rancher Work加入集群信息"](/assets/rancher/imgs/rancher-work.png)
### (13). Rancher Work加入集群
```
[root@rancher-node1 ~]# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.6.3 --server https://172.30.50.16 --token 295bg677s2vzxgxth2krthf5gcbd6bn42sv2ktt4ssvqjgfbrh6f82 --ca-checksum 5357adc60ce6c0ce0660f06ec9c012f77c9cf97b75c02cfb31816e6c18ae551a --worker
```
### (14). kubectl配置
```
[root@rancher-master ~]# wget http://rancher-mirror.cnrancher.com/kubectl/v1.21.8/linux-amd64-v1.21.8-kubectl
[root@rancher-master ~]# mv linux-amd64-v1.21.8-kubectl /usr/sbin/kubectl
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
### (15). 总结
尝试了N次,国内的网络感觉安装Rancher都有问题.