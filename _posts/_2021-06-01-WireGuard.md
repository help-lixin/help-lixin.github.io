---
layout: post
title: 'VPN(WireGuard) 搭建'
date: 2021-06-01
author: 李新
tags:  WireGuard VPN
---

### (1). 为什么研究VPN
> 用K8S后,为了解决开发环境联调问题,所以,才会研究下VPN(当然,还有其它的解决方案,比如:阿里针对K8S研发了类似的工具:KT Connect/SSH隧道).   
> <font color='red'>注意:请合理运用工具去做正当的事情.</font>   

### (2). WireGuard介绍
> WireGuard是最新VPN协议,它利用了最新的加密技术,比IPsec更快,更简单,比OpenVPN具有更高的性能.WireGuard被设计为通用VPN,可在嵌入式接口和超级计算机上运行.
> Wireguard最初针对Linux内核发布,现已跨平台(Windows,MacOS,BSD,iOS,Android)并可广泛部署.它目前正在开发中,但是已经被认为是业内最安全,最易用和最简单的VPN解决方案.  

### (3). WireGuard安装
> ["wireguard-install.sh"](/assets/wireguard/wireguard-install.sh)   

```
# 参考地址:
https://github.com/angristan/wireguard-install

[root@app-1 opt]# uname -a
Linux app-1 3.10.0-1160.el7.x86_64 #1 SMP Mon Oct 19 16:18:59 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

# 1. 下载一键安装的脚本
[root@app-1 opt]# curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
[root@app-1 opt]# chmod +x wireguard-install.sh
# 安装
[root@app-1 opt]# ./wireguard-install.sh
Welcome to the WireGuard installer!
The git repository is available at: https://github.com/angristan/wireguard-install
I need to ask you a few questions before starting the setup.
You can leave the default options and just press enter if you are ok with them.

# 1. IP地址/网卡/生成的IP段/WireGuard端口
IPv4 or IPv6 public address: 10.211.55.100
Public interface: eth0
WireGuard interface name: wg0
Server's WireGuard IPv4: 10.66.66.1
Server's WireGuard IPv6: fd42:42:42::1
Server's WireGuard port [1-65535]: 57413
First DNS resolver to use for the clients: 8.8.8.8
Second DNS resolver to use for the clients (optional): 8.8.8.8

Okay, that was all I needed. We are ready to setup your WireGuard server now.
You will be able to generate a client at the end of the installation.
Press any key to continue...
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.ustc.edu.cn
 * epel: my.mirrors.thegigabit.com
 * extras: mirrors.ustc.edu.cn
 * updates: ftp.sjtu.edu.cn
软件包 epel-release-7-13.noarch 已安装并且是最新版本
正在解决依赖关系
--> 正在检查事务
---> 软件包 elrepo-release.noarch.0.7.0-5.el7.elrepo 将被 安装
--> 解决依赖关系完成

依赖关系解决

======================================================================================================================
 Package                        架构                   版本                              源                      大小
======================================================================================================================
正在安装:
 elrepo-release                 noarch                 7.0-5.el7.elrepo                  extras                 9.5 k

事务概要
======================================================================================================================
安装  1 软件包

总下载量：9.5 k
安装大小：5.0 k
Downloading packages:
elrepo-release-7.0-5.el7.elrepo.noarch.rpm                                                     | 9.5 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : elrepo-release-7.0-5.el7.elrepo.noarch                                                            1/1
  验证中      : elrepo-release-7.0-5.el7.elrepo.noarch                                                            1/1

已安装:
  elrepo-release.noarch 0:7.0-5.el7.elrepo

完毕！
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.ustc.edu.cn
 * elrepo: mirror-hk.koddos.net
 * epel: my.mirrors.thegigabit.com
 * extras: mirrors.ustc.edu.cn
 * updates: ftp.sjtu.edu.cn
elrepo                                                                                         | 3.0 kB  00:00:00
elrepo/primary_db                                                                              | 425 kB  00:00:02
正在解决依赖关系
--> 正在检查事务
---> 软件包 yum-plugin-elrepo.noarch.0.7.5.3-1.el7.elrepo 将被 安装
--> 解决依赖关系完成

依赖关系解决

======================================================================================================================
 Package                          架构                  版本                              源                     大小
======================================================================================================================
正在安装:
 yum-plugin-elrepo                noarch                7.5.3-1.el7.elrepo                elrepo                 13 k

事务概要
======================================================================================================================
安装  1 软件包

总下载量：13 k
安装大小：24 k
Downloading packages:
警告：/var/cache/yum/x86_64/7/elrepo/packages/yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch.rpm: 头V4 DSA/SHA1 Signature, 密钥 ID baadae52: NOKEY
yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch.rpm 的公钥尚未安装
yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch.rpm                                                |  13 kB  00:00:00
从 file:///etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org 检索密钥
导入 GPG key 0xBAADAE52:
 用户ID     : "elrepo.org (RPM Signing Key for elrepo.org) <secure@elrepo.org>"
 指纹       : 96c0 104f 6315 4731 1e0b b1ae 309b c305 baad ae52
 软件包     : elrepo-release-7.0-5.el7.elrepo.noarch (@extras)
 来自       : /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch                                                       1/1
  验证中      : yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch                                                       1/1

已安装:
  yum-plugin-elrepo.noarch 0:7.5.3-1.el7.elrepo

完毕！
已加载插件：elrepo, fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.ustc.edu.cn
 * elrepo: mirror.rackspace.com
 * epel: my.mirrors.thegigabit.com
 * extras: mirrors.ustc.edu.cn
 * updates: ftp.sjtu.edu.cn
[elrepo]: 56 kmod packages excluded due to dependency errors
软件包 wireguard-tools-1.0.20210424-1.el7.x86_64 已安装并且是最新版本
软件包 iptables-1.4.21-35.el7.x86_64 已安装并且是最新版本
正在解决依赖关系
--> 正在检查事务
---> 软件包 kmod-wireguard.x86_64.9.1.0.20210606-1.el7_9.elrepo 将被 安装
---> 软件包 qrencode.x86_64.0.3.4.1-3.el7 将被 安装
--> 解决依赖关系完成

依赖关系解决

======================================================================================================================
 Package                     架构                版本                                       源                   大小
======================================================================================================================
正在安装:
 kmod-wireguard              x86_64              9:1.0.20210606-1.el7_9.elrepo              elrepo              101 k
 qrencode                    x86_64              3.4.1-3.el7                                base                 19 k

事务概要
======================================================================================================================
安装  2 软件包

总下载量：119 k
安装大小：303 k
Downloading packages:
(1/2): qrencode-3.4.1-3.el7.x86_64.rpm                                                         |  19 kB  00:00:00
(2/2): kmod-wireguard-1.0.20210606-1.el7_9.elrepo.x86_64.rpm                                   | 101 kB  00:00:02
----------------------------------------------------------------------------------------------------------------------
总计                                                                                   44 kB/s | 119 kB  00:00:02
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : qrencode-3.4.1-3.el7.x86_64                                                                       1/2
  正在安装    : 9:kmod-wireguard-1.0.20210606-1.el7_9.elrepo.x86_64                                               2/2
Working. This may take some time ...
Done.
  验证中      : 9:kmod-wireguard-1.0.20210606-1.el7_9.elrepo.x86_64                                               1/2
  验证中      : qrencode-3.4.1-3.el7.x86_64                                                                       2/2

已安装:
  kmod-wireguard.x86_64 9:1.0.20210606-1.el7_9.elrepo                  qrencode.x86_64 0:3.4.1-3.el7

完毕！
* Applying /usr/lib/sysctl.d/00-system.conf ...
* Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
kernel.yama.ptrace_scope = 0
* Applying /usr/lib/sysctl.d/50-default.conf ...
# ************************************************************
# 内核转发
# ************************************************************
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
* Applying /usr/lib/sysctl.d/60-libvirtd.conf ...
fs.aio-max-nr = 1048576
* Applying /etc/sysctl.d/99-sysctl.conf ...
net.ipv4.ip_forward = 1
net.ipv4.conf.all.proxy_arp = 1
* Applying /etc/sysctl.d/wg.conf ...
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
* Applying /etc/sysctl.conf ...
net.ipv4.ip_forward = 1
net.ipv4.conf.all.proxy_arp = 1
Created symlink from /etc/systemd/system/multi-user.target.wants/wg-quick@wg0.service to /usr/lib/systemd/system/wg-quick@.service.

Tell me a name for the client.
The name must consist of alphanumeric character. It may also include an underscore or a dash and can't exceed 15 chars.
Client name: test
Client's WireGuard IPv4: 10.66.66.2
Client's WireGuard IPv6: fd42:42:42::2
Here is your client config file as a QR Code:

// 此处会为client生在一张二维码
// ... ... 

It is also available in /root/wg0-client-test.conf
If you want to add more clients, you simply need to run this script another time!
```
### (4). 查看服务端配置(/etc/wireguard/wg0.conf)
```
### 服务器部份设置 ### 
[Interface]
### 设置虚拟网卡的内网地址
Address = 10.66.66.1/24
### 设置udp监听端口,可选范围为49152到65535 
ListenPort = 57413
### 填写本机的私钥,默认存储在本机的/etc/wireguard/params文本中
PrivateKey = MCWyJCLQIheYxudzeBgWJyLhOwjq/Bk55vbLwPjjaG8=
### wg-quick up wg0启动后执行的内核防火墙(iptables)规则,可以打通VPN,服务器端需此参数.
PostUp = iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT; iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
### wg-quick down wg0执行删除启动时定义的内核防火墙(iptables)规则,服务器端需此参数
PostDown = iptables -D FORWARD -i eth0 -o wg0 -j ACCEPT; iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

### Client test
[Peer]
### client连接的公钥
PublicKey = mctMsDFkGmzgMwWrhupMvlMEeCIudPNypzfnwyfzSCk=
### 
PresharedKey = oGejVfpR+Qps1z6g9LzkFVxuUtqRTEeXnMUVnlCnVKc=
### 允许连接的ip地址
AllowedIPs = 0.0.0.0/32,::/0
```
### (5). 查看服务端/etc/wireguard/params
```
[root@app-1 wireguard]# cat params
SERVER_PUB_IP=10.211.55.100
SERVER_PUB_NIC=eth0
SERVER_WG_NIC=wg0
SERVER_WG_IPV4=10.66.66.1
SERVER_WG_IPV6=fd42:42:42::1
SERVER_PORT=57413
SERVER_PRIV_KEY=MCWyJCLQIheYxudzeBgWJyLhOwjq/Bk55vbLwPjjaG8=
SERVER_PUB_KEY=VQph4ERk/tWP1wuanSpxb4U2PakFLMkV7/MdBHKtBjw=
CLIENT_DNS_1=8.8.8.8
CLIENT_DNS_2=8.8.8.8
```
### (6). mac安装wireguard-tools
```
# 1. 安装wireguard-tools
lixin-macbook:~ lixin$ brew install wireguard-tools
lixin-macbook:~ lixin$ sudo mkdir /usr/local/etc/wireguard
lixin-macbook:~ lixin$ sudo touch /usr/local/etc/wireguard/wg0.conf
```
### (7). mac配置wg0.conf(/usr/local/etc/wireguard/wg0.conf)
```
[Interface] 
### 通过命令,生成公私钥:wg genkey | tee privatekey | wg pubkey > publickey
PrivateKey = WLKjwWDu2N7TlZX/0G+ldsYzPxv6tLuR4iaJgaEuO3E= 
Address = 10.66.66.2/32
DNS = 8.8.8.8,8.8.8.8

[Peer] 
### 服务器端的公钥
PublicKey = VQph4ERk/tWP1wuanSpxb4U2PakFLMkV7/MdBHKtBjw= 
PresharedKey = oGejVfpR+Qps1z6g9LzkFVxuUtqRTEeXnMUVnlCnVKc= 
Endpoint = 10.211.55.100:57413 
AllowedIPs = 0.0.0.0/0,::/0
```
### (8). mac启动WireGuard
```
# 启动网卡
lixin-macbook:~ lixin$ sudo wg-quick up wg0
Warning: `/usr/local/etc/wireguard/wg0.conf' is world accessible
[#] wireguard-go utun
[+] Interface for wg0 is utun4
[#] wg setconf utun4 /dev/fd/63
[#] ifconfig utun4 inet 10.66.66.2/32 10.66.66.2 alias
[#] ifconfig utun4 inet6 fd42:42:42::2/128 alias
[#] ifconfig utun4 up
[#] route -q -n add -inet6 ::/1 -interface utun4
[#] route -q -n add -inet6 8000::/1 -interface utun4
[#] route -q -n add -inet 0.0.0.0/1 -interface utun4
[#] route -q -n add -inet 128.0.0.0/1 -interface utun4
[#] route -q -n add -inet 10.211.55.100 -gateway 172.17.0.2
[#] networksetup -getdnsservers Wi-Fi
[#] networksetup -getsearchdomains Wi-Fi
[#] networksetup -getdnsservers Bluetooth PAN
[#] networksetup -getsearchdomains Bluetooth PAN
[#] networksetup -setdnsservers Bluetooth PAN 8.8.8.8 8.8.8.8
[#] networksetup -setsearchdomains Bluetooth PAN Empty
[#] networksetup -setdnsservers Wi-Fi 8.8.8.8 8.8.8.8
[#] networksetup -setsearchdomains Wi-Fi Empty
[+] Backgrounding route monitor

# 2. 查看状态
lixin-macbook:~ lixin$ sudo wg show
interface: utun4
  public key: mctMsDFkGmzgMwWrhupMvlMEeCIudPNypzfnwyfzSCk=
  private key: (hidden)
  listening port: 63432

peer: VQph4ERk/tWP1wuanSpxb4U2PakFLMkV7/MdBHKtBjw=
  preshared key: (hidden)
  endpoint: 10.211.55.100:57413
  allowed ips: 0.0.0.0/0, ::/0
  transfer: 0 B received, 1.16 KiB sent
```
