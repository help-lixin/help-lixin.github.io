---
layout: post
title: '自建Headscale' 
date: 2023-04-15
author: 李新
tags:  Headscale Tailscale VPN WireGuard
---

### (1). 概述
最近项目,团队成员分散在各处,其中,开发环境是通过:ESXI虚拟出来几十台的机器,为了协同办公,解决组网问题,真的是遇到了不少问题(电信封端口/NAT需要中继服务器,相当的耗流量),
最终,选择了:Tailscale这款产品(SD WAN的实现),在这里真的要夸一下这个产品,同事深圳到长沙机器之间的延迟在30ms以内,相当的流畅. 
### (2). 为个么这么流畅
Tailscale是是基于WireGuard进行了二次封装开发的,而WireGuard的优点在于,通信过程中不依赖中心服务器来着的,各节点之间是平等的,仅仅是第一次节点发现时,需要连接中心服务器.  
### (3). Tailscale是什么
Tailscale是一种基于WireGuard的虚拟组网工具,和Netmaker类似,最大的区别在于Tailscale是在用户态实现了WireGuard协议,而Netmaker直接使用了内核态的WireGuard,所以,Tailscale相比于内核态WireGuard性能会有所损失,但与OpenVPN之流相比还是能甩好几十条街的. 
### (4). Headscale是什么
Headscale看名字就知道是和Tailscale对着干的,Tailscale的客户端是不收费的,服务端是不开源的,超过20个设备就需要付费了,并且Tailscale的服务不在国内,我们不可能把服务端安全的交给Tailscale,私有自建才是出入.Headscale是个第三方开源版本的Tailscale的服务端,除了「网站界面」之外该有的功能都有,因此我们使用Headscale自建私有服务. 
### (5). Headscale安装
```
[root@headscale ~]# wget -O /usr/local/bin/headscale https://github.com/juanfont/headscale/releases/download/v0.20.0/headscale_0.20.0_linux_amd64
[root@headscale ~]# chmod +x /usr/local/bin/headscale
[root@headscale ~]# ln -s /usr/local/bin/headscale /usr/bin/headscale

#创建配置目录
[root@headscale ~]# mkdir -p /etc/headscale

#创建目录用来存储数据与证书
[root@headscale ~]# mkdir -p /var/lib/headscale

#创建空的 SQLite 数据库文件
[root@headscale ~]# touch /var/lib/headscale/db.sqlite

# 下载模板配置文件
[root@headscale ~]# wget https://github.com/juanfont/headscale/raw/main/config-example.yaml -O /etc/headscale/config.yaml
```
### (6). 配置config.yaml
> 我主要改了这三个配置:   
> 1. server_url    
> 2. listen_addr    
> 3. magic_dns   

```
[root@headscale ~]# cat /etc/headscale/config.yaml |  grep -Ev '^$|#'
---
server_url: http://192.168.1.14:8080
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090
grpc_listen_addr: 127.0.0.1:50443
grpc_allow_insecure: false
private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key
ip_prefixes:
  - fd7a:115c:a1e0::/48
  - 100.64.0.0/10
derp:
  server:
    enabled: false
    region_id: 999
    region_code: "headscale"
    region_name: "Headscale Embedded DERP"
    stun_listen_addr: "0.0.0.0:3478"
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  paths: []
  auto_update_enabled: true
  update_frequency: 24h
disable_check_updates: false
ephemeral_node_inactivity_timeout: 30m
node_update_check_interval: 10s
db_type: sqlite3
db_path: /var/lib/headscale/db.sqlite
acme_url: https://acme-v02.api.letsencrypt.org/directory
acme_email: ""
tls_letsencrypt_hostname: ""
tls_letsencrypt_cache_dir: /var/lib/headscale/cache
tls_letsencrypt_challenge_type: HTTP-01
tls_letsencrypt_listen: ":http"
tls_cert_path: ""
tls_key_path: ""
log:
  format: text
  level: info
acl_policy_path: ""
dns_config:
  override_local_dns: true
  nameservers:
    - 1.1.1.1
  domains: []
  magic_dns: false
  base_domain: example.com
unix_socket: /var/run/headscale/headscale.sock
unix_socket_permission: "0770"
logtail:
  enabled: false
randomize_client_port: false
```
### (7). 创建服务启动文件
```
# /etc/systemd/system/headscale.service
[Unit]
Description=headscale controller
After=syslog.target
After=network.target

[Service]
Type=simple
User=headscale
Group=headscale
ExecStart=/usr/local/bin/headscale serve
Restart=always
RestartSec=5

# Optional security enhancements
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/headscale /var/run/headscale
AmbientCapabilities=CAP_NET_BIND_SERVICE
RuntimeDirectory=headscale

[Install]
WantedBy=multi-user.target
```
### (8). 准备工作
```
#创建用户
[root@headscale ~]# useradd headscale -d /home/headscale -m

#修改权限
[root@headscale ~]# chown -R headscale:headscale /var/lib/headscale


#默认文件root权限启动会失败,会提示:"listen unix /var/run/headscale/headscale.sock: bind: no such file or directory"
#注意,测试在root用户下手动执行命令启动没有报错信息,需切换到headscale用户启动. 
[root@headscale ~]# mkdir /var/run/headscale
```
### (9). 启动
```
[root@headscale ~]# systemctl daemon-reload
[root@headscale ~]# systemctl enable --now headscale

#查看运行状态
[root@headscale ~]# systemctl status headscale

#查看端口
[root@headscale ~]#  ss -tnlp | grep headscale
LISTEN     0      4096   127.0.0.1:9090                     *:*                   users:(("headscale",pid=12067,fd=13))
LISTEN     0      4096      [::]:8080                  [::]:*                   users:(("headscale",pid=12067,fd=12))
```
### (10). 验证安装是否成功
```
# 对于headscale,你可以理解user为命名空间,不同的命名空间是隔离的,相同命名空间是可以互相访问的. 
# 列出命名空间
[root@headscale ~]# headscale users list
ID | Name | Created


# 2. 创建命名空间
[root@headscale ~]# headscale users create default
User created

# 3. 列出命名空间
[root@headscale ~]# headscale users list
ID | Name    | Created            
1  | default | 2023-04-14 16:58:45
```
### (11). Linux客户端安装tailscale
```

[root@jenkins ~]# wget https://pkgs.tailscale.com/stable/tailscale_1.34.2_amd64.tgz
[root@jenkins ~]# tar -xf tailscale_1.34.2_amd64.tgz 
[root@jenkins ~]# cp tailscale_1.34.2_amd64/tailscale* /usr/sbin/
[root@jenkins ~]# cp tailscale_1.34.2_amd64/systemd/tailscaled.service /etc/systemd/system/
[root@jenkins ~]# cp tailscale_1.34.2_amd64/systemd/tailscaled.defaults /etc/default/tailscaled
[root@jenkins ~]# systemctl enable --now tailscaled

#查看启动状态
[root@jenkins ~]# systemctl status tailscaled
```
### (12). Linux客户端接入
```
# 这里推荐将 DNS 功能关闭，因为它会覆盖系统的默认 DNS
# 通过浏览器打开下面这个链接
[root@jenkins ~]# tailscale up --login-server=http://192.168.1.14:8080 --accept-routes=true --accept-dns=false
To authenticate, visit:
        http://192.168.1.14:8080/register/nodekey:9a6ba5f1442a17562fc720a3762438bed89f8fbb1ed5e19da9a049883399fa73
```
### (13). 浏览器提示,在Headscale主机上,进行注册.
```
headscale
Machine registration
Run the command below in the headscale server to add this machine to your network:
headscale nodes register --user USERNAME --key nodekey:9a6ba5f1442a17562fc720a3762438bed89f8fbb1ed5e19da9a049883399fa73
```
### (14). Headscale服务器配置,节点注册. 
```
# default为上面创建的user
[app@headscale ~]$ sudo headscale nodes register --user default --key nodekey:9a6ba5f1442a17562fc720a3762438bed89f8fbb1ed5e19da9a049883399fa73
 Machine jenkins registered
```
### (15). Headscale服务器验证
```
[app@headscale ~]$ sudo headscale nodes list 
ID | Hostname | Name    | MachineKey | NodeKey | User    | IP addresses                  | Ephemeral | Last seen           | Expiration          | Online | Expired
1  | jenkins  | jenkins | [RqMb9]    | [mmul8] | default | 100.64.0.1, fd7a:115c:a1e0::1 | false     | 2023-04-15 09:26:22 | 0001-01-01 00:00:00 | online | no   
```
### (16). Tailscale客户端验证
```
[root@jenkins ~]# ip addr
3: tailscale0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1280 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 100.64.0.1/32 scope global tailscale0
       valid_lft forever preferred_lft forever
    inet6 fd7a:115c:a1e0::1/128 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::b919:83a4:2c46:9b25/64 scope link flags 800 
       valid_lft forever preferred_lft forever
```