---
layout: post
title: 'Prometheus 安装'
date: 2021-04-26
author: 李新
tags:  Prometheus 运维监控
---

### (1). Prometheus是什么?
> Prometheus是最近几年开始流行的一个新兴监控告警工具,特别是kubernetes的流行带动了Prometheus的应用. 

### (2). 安装前准备工作
```
# 1. 创建独立用户
[root@prometheus ~]# useradd --no-create-home -s /sbin/nologin prometheus
# 2. 创建配置文件夹路径
[root@prometheus ~]# mkdir -p /etc/prometheus/
# 3. 创建tsdb存储路径
[root@prometheus ~]# mkdir -p /var/lib/prometheus/

```
### (3). Prometheus安装
```
# 1. 下载
[root@prometheus ~]# wget https://github.com/prometheus/prometheus/releases/download/v2.26.0/prometheus-2.26.0.linux-amd64.tar.gz
# 2. 解压
[root@prometheus ~]# tar -zxvf prometheus-2.26.0.linux-amd64.tar.gz -C /usr/local/bin/
# 3. 创建软链接
[root@prometheus ~]# cd /usr/local/bin/
# 创建软链接
[root@prometheus bin]# ln -s prometheus-2.26.0.linux-amd64 prometheus
[root@prometheus bin]# ll
lrwxrwxrwx 1 root root   29 Apr  9 21:32 prometheus -> prometheus-2.26.0.linux-amd64
drwxr-xr-x 4 3434 3434 4096 Mar 31 20:10 prometheus-2.26.0.linux-amd64

# 4. 配置prometheus和promtool的所属组
[root@prometheus ~]# chown prometheus:prometheus /usr/local/bin/prometheus/promtool
[root@prometheus ~]# chown prometheus:prometheus /usr/local/bin/prometheus/prometheus

# 5. 把"consoles"和"console_libraries"目录,移到"/etc/prometheus"文件夹下
#    把"prometheus.yml"文件 移到到: "/etc/prometheus/"目录下
[root@prometheus ~]# mv /usr/local/bin/prometheus/prometheus.yml     /etc/prometheus/
[root@prometheus ~]# mv  /usr/local/bin/prometheus/consoles          /etc/prometheus/
[root@prometheus ~]# mv /usr/local/bin/prometheus/console_libraries  /etc/prometheus/

# 6. 更改(/etc/prometheus和/var/lib/prometheus/的所属组)
[root@prometheus ~]# chown -R prometheus:prometheus /etc/prometheus
[root@prometheus ~]# chown -R prometheus:prometheus /var/lib/prometheus/
```
### (4). 为Prometheus创建Systemd
```
vi /etc/systemd/system/prometheus.service

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```
### (5). 启动Prometheus
```
[root@prometheus ~]# systemctl daemon-reload
[root@prometheus ~]# systemctl start prometheus
[root@prometheus ~]# systemctl status prometheus
```

### (6). node_exporter安装
> node_exporter提供了对:监控服务器的CPU、内存、磁盘、I/O等数据收集.  

```
# 我这里是另一台机器了
#[root@prometheus ~]# useradd --no-create-home -s /sbin/nologin prometheus

# 1.下载node_exporter
[root@node-1 ~]# wget https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-amd64.tar.gz
# 2.解压到:/usr/local/bin目录下 
[root@node-1 ~]# tar -zxvf node_exporter-1.1.2.linux-amd64.tar.gz -C /usr/local/bin/
# 3. 创建软链接
[root@node-1 ~]# ln -s /usr/local/bin/node_exporter-1.1.2.linux-amd64 /usr/local/bin/node_exporter
```
### (7). 为node_exporter创建Systemd
```
[root@node-1 ~]# vi  /etc/systemd/system/node_exporter.service

[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target
[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/node_exporter/node_exporter
Restart=on-failure
[Install]
WantedBy=multi-user.target
```

### (7). 启动node_exporter.service
```
[root@node-1 ~]# systemctl daemon-reload
[root@node-1 ~]# systemctl start node_exporter
[root@node-1 ~]# systemctl status node_exporter

# 查看端口禁听是否成功
[root@node-1 ~]# lsof -i tcp:9100
COMMAND     PID       USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
node_expo 11590 prometheus    3u  IPv6  62265      0t0  TCP *:jetdirect (LISTEN)
```

### (8). 将node_exporter与prometheus绑定
> 将node_exporter纳入:prometheus监控范围内.  

```
[root@prometheus ~]# cat /etc/prometheus/prometheus.yml
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

alerting:
  alertmanagers:
  - static_configs:
    - targets:

rule_files:

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
  
  # 这是我新增的内容
  - job_name: 'node-1'
    static_configs:
    - targets: ['10.211.55.102:9100']
      labels:
        instance: node-1
```

### (8). 重启prometheus
```
[root@prometheus ~]# systemctl daemon-reload
[root@prometheus ~]# systemctl restart prometheus
[root@prometheus ~]# systemctl status prometheus
```
### (9). 查看结果
!["Prometheus 安装结果"](/assets/prometheus/imgs/prometheus-status.png)