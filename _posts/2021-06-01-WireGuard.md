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

[root@iZj6cgalu8r574j7e3khydZ ~]# uname -a
Linux iZj6cgalu8r574j7e3khydZ 3.10.0-1160.25.1.el7.x86_64 #1 SMP Wed Apr 28 21:49:45 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux

# 1. 下载一键安装的脚本
[root@iZj6cgalu8r574j7e3khydZ ~]# curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
[root@iZj6cgalu8r574j7e3khydZ ~]# chmod +x wireguard-install.sh
# 安装
[root@iZj6cgalu8r574j7e3khydZ ~]# ./wireguard-install.sh

Welcome to the WireGuard installer!
The git repository is available at: https://github.com/angristan/wireguard-install
I need to ask you a few questions before starting the setup.
You can leave the default options and just press enter if you are ok with them.

# 1. IP地址/网卡/生成的IP段/WireGuard端口
IPv4 or IPv6 public address: 8.210.32.174
Public interface: eth0
WireGuard interface name: wg0
Server's WireGuard IPv4: 172.31.41.100
Server's WireGuard IPv6:
Server's WireGuard IPv6: fd42:42:42::1
Server's WireGuard port [1-65535]: 50506
First DNS resolver to use for the clients: 1.1.1.1
Second DNS resolver to use for the clients (optional): 8.8.8.8

Okay, that was all I needed. We are ready to setup your WireGuard server now.
You will be able to generate a client at the end of the installation.
# 按任意键,将继续
Press any key to continue...
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
---> Package elrepo-release.noarch 0:7.0-5.el7.elrepo will be installed
---> Package epel-release.noarch 0:7-13 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================
 Package                        Arch                   Version                           Repository              Size
======================================================================================================================
Installing:
 elrepo-release                 noarch                 7.0-5.el7.elrepo                  extras                 9.5 k
 epel-release                   noarch                 7-13                              epel                    15 k

Transaction Summary
======================================================================================================================
Install  2 Packages

Total download size: 25 k
Installed size: 30 k
Downloading packages:
(1/2): epel-release-7-13.noarch.rpm                                                            |  15 kB  00:00:00
(2/2): elrepo-release-7.0-5.el7.elrepo.noarch.rpm                                              | 9.5 kB  00:00:00
----------------------------------------------------------------------------------------------------------------------
Total                                                                                 169 kB/s |  25 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : epel-release-7-13.noarch                                                                           1/2
warning: /etc/yum.repos.d/epel.repo created as /etc/yum.repos.d/epel.repo.rpmnew
  Installing : elrepo-release-7.0-5.el7.elrepo.noarch                                                             2/2
  Verifying  : elrepo-release-7.0-5.el7.elrepo.noarch                                                             1/2
  Verifying  : epel-release-7-13.noarch                                                                           2/2

Installed:
  elrepo-release.noarch 0:7.0-5.el7.elrepo                         epel-release.noarch 0:7-13

Complete!
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * elrepo: mirror-hk.koddos.net
elrepo                                                                                         | 3.0 kB  00:00:00
elrepo/primary_db                                                                              | 425 kB  00:00:00
Resolving Dependencies
--> Running transaction check
---> Package yum-plugin-elrepo.noarch 0:7.5.3-1.el7.elrepo will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================
 Package                          Arch                  Version                           Repository             Size
======================================================================================================================
Installing:
 yum-plugin-elrepo                noarch                7.5.3-1.el7.elrepo                elrepo                 13 k

Transaction Summary
======================================================================================================================
Install  1 Package

Total download size: 13 k
Installed size: 24 k
Downloading packages:
warning: /var/cache/yum/x86_64/7/elrepo/packages/yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch.rpm: Header V4 DSA/SHA1 Signature, key ID baadae52: NOKEY
Public key for yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch.rpm is not installed
yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch.rpm                                                |  13 kB  00:00:00
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
Importing GPG key 0xBAADAE52:
 Userid     : "elrepo.org (RPM Signing Key for elrepo.org) <secure@elrepo.org>"
 Fingerprint: 96c0 104f 6315 4731 1e0b b1ae 309b c305 baad ae52
 Package    : elrepo-release-7.0-5.el7.elrepo.noarch (@extras)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch                                                        1/1
  Verifying  : yum-plugin-elrepo-7.5.3-1.el7.elrepo.noarch                                                        1/1

Installed:
  yum-plugin-elrepo.noarch 0:7.5.3-1.el7.elrepo

Complete!
Loaded plugins: elrepo, fastestmirror
Loading mirror speeds from cached hostfile
 * elrepo: mirror.rackspace.com
[elrepo]: 56 kmod packages excluded due to dependency errors
Package iptables-1.4.21-35.el7.x86_64 already installed and latest version
Resolving Dependencies
--> Running transaction check
---> Package kmod-wireguard.x86_64 9:1.0.20210606-1.el7_9.elrepo will be installed
---> Package qrencode.x86_64 0:3.4.1-3.el7 will be installed
---> Package wireguard-tools.x86_64 0:1.0.20210424-1.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================
 Package                     Arch               Version                                      Repository          Size
======================================================================================================================
Installing:
 kmod-wireguard              x86_64             9:1.0.20210606-1.el7_9.elrepo                elrepo             101 k
 qrencode                    x86_64             3.4.1-3.el7                                  base                19 k
 wireguard-tools             x86_64             1.0.20210424-1.el7                           epel               122 k

Transaction Summary
======================================================================================================================
Install  3 Packages

Total download size: 241 k
Installed size: 594 k
Downloading packages:
(1/3): qrencode-3.4.1-3.el7.x86_64.rpm                                                         |  19 kB  00:00:00
(2/3): wireguard-tools-1.0.20210424-1.el7.x86_64.rpm                                           | 122 kB  00:00:00
(3/3): kmod-wireguard-1.0.20210606-1.el7_9.elrepo.x86_64.rpm                                   | 101 kB  00:00:00
----------------------------------------------------------------------------------------------------------------------
Total                                                                                 696 kB/s | 241 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : qrencode-3.4.1-3.el7.x86_64                                                                        1/3
  Installing : wireguard-tools-1.0.20210424-1.el7.x86_64                                                          2/3
  Installing : 9:kmod-wireguard-1.0.20210606-1.el7_9.elrepo.x86_64                                                3/3
Working. This may take some time ...
Done.
  Verifying  : 9:kmod-wireguard-1.0.20210606-1.el7_9.elrepo.x86_64                                                1/3
  Verifying  : wireguard-tools-1.0.20210424-1.el7.x86_64                                                          2/3
  Verifying  : qrencode-3.4.1-3.el7.x86_64                                                                        3/3

Installed:
  kmod-wireguard.x86_64 9:1.0.20210606-1.el7_9.elrepo                  qrencode.x86_64 0:3.4.1-3.el7
  wireguard-tools.x86_64 0:1.0.20210424-1.el7

Complete!
* Applying /usr/lib/sysctl.d/00-system.conf ...
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
vm.swappiness = 0
kernel.sysrq = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_slow_start_after_idle = 0
* Applying /etc/sysctl.d/wg.conf ...
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
* Applying /etc/sysctl.conf ...
vm.swappiness = 0
kernel.sysrq = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_slow_start_after_idle = 0
Created symlink from /etc/systemd/system/multi-user.target.wants/wg-quick@wg0.service to /usr/lib/systemd/system/wg-quick@.service.

Tell me a name for the client.
The name must consist of alphanumeric character. It may also include an underscore or a dash and can't exceed 15 chars.

# 配置一个client名称/IP
Client name: test
Client's WireGuard IPv4: 172.31.41.101
Client's WireGuard IPv6: fd42:42:42::101
Here is your client config file as a QR Code:

// 此处会为client生在一张二维码,并把二维码的内容,保存在:/root/wg0-client-test.conf
// ... ... 

# *************************************************************************
# 为client生成的内容
It is also available in /root/wg0-client-test.conf
# *************************************************************************
If you want to add more clients, you simply need to run this script another time!
```
### (4). 查看服务端配置(/etc/wireguard/wg0.conf)
```
[Interface]
### 设置虚拟网卡的内网地址
Address = 172.31.41.100/24,fd42:42:42::1/64
### 设置udp监听端口,可选范围为49152到65535 
ListenPort = 50506
### 填写本机的私钥,默认存储在本机的/etc/wireguard/params文本中
PrivateKey = qKeo5zgUxNoktwXs2x8NsBkSgyfyQBmRubf1GKy1AWw=
### wg-quick up wg0启动后执行的内核防火墙(iptables)规则,可以打通VPN,服务器端需此参数.
PostUp = iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT; iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
### wg-quick down wg0执行删除启动时定义的内核防火墙(iptables)规则,服务器端需此参数
PostDown = iptables -D FORWARD -i eth0 -o wg0 -j ACCEPT; iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

### Client test
[Peer]
### client连接的公钥
PublicKey = zmijB4jE1QMMkRtTe+doL+x8ONm3NOgF7HZbP4sTc3E=
PresharedKey = HixRDQSAkMXPE9iEOshLl9jVFKBm3tz5PK6IueVE49s=
### 允许连接的ip地址
AllowedIPs = 172.31.41.101/32,fd42:42:42::101/128
```
### (5). 查看服务端/etc/wireguard/params
```
[root@app-1 wireguard]# cat params
SERVER_PUB_IP=8.210.32.174
SERVER_PUB_NIC=eth0
SERVER_WG_NIC=wg0
SERVER_WG_IPV4=172.31.41.100
SERVER_WG_IPV6=fd42:42:42::1
SERVER_PORT=50506
SERVER_PRIV_KEY=qKeo5zgUxNoktwXs2x8NsBkSgyfyQBmRubf1GKy1AWw=
SERVER_PUB_KEY=pVtk+KUkNWcORxWYgPbrBkcr8SNKemAL75cJqs26124=
CLIENT_DNS_1=1.1.1.1
CLIENT_DNS_2=8.8.8.8
```
### (6). Mac(Client)安装wireguard-tools
```
# 1. 安装wireguard-tools
lixin-macbook:~ lixin$ brew install wireguard-tools
lixin-macbook:~ lixin$ sudo mkdir /usr/local/etc/wireguard
lixin-macbook:~ lixin$ sudo touch /usr/local/etc/wireguard/wg0.conf
```
### (7). Mac(Client)配置wg0.conf(/usr/local/etc/wireguard/wg0.conf)
> 这里的内容,直接从服务端生成的文件(/root/wg0-client-test.conf)里拷过来即可.

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
### (8). Mac启动WireGuard
```
# 1. 启用网卡
lixin-macbook:~ lixin$ sudo wg-quick up wg0
Warning: `/usr/local/etc/wireguard/wg0.conf' is world accessible
[#] wireguard-go utun
[+] Interface for wg0 is utun4
[#] wg setconf utun4 /dev/fd/63
[#] ifconfig utun4 inet 172.31.41.101/32 172.31.41.101 alias
[#] ifconfig utun4 inet6 fd42:42:42::101/128 alias
[#] ifconfig utun4 up
[#] route -q -n add -inet6 ::/1 -interface utun4
[#] route -q -n add -inet6 8000::/1 -interface utun4
[#] route -q -n add -inet 0.0.0.0/1 -interface utun4
[#] route -q -n add -inet 128.0.0.0/1 -interface utun4
[#] route -q -n add -inet 8.210.32.174 -gateway 172.17.0.2
[#] networksetup -getdnsservers Wi-Fi
[#] networksetup -getsearchdomains Wi-Fi
[#] networksetup -getdnsservers Bluetooth PAN
[#] networksetup -getsearchdomains Bluetooth PAN
[#] networksetup -setdnsservers Bluetooth PAN 1.1.1.1 8.8.8.8
[#] networksetup -setsearchdomains Bluetooth PAN Empty
[#] networksetup -setdnsservers Wi-Fi 1.1.1.1 8.8.8.8
[#] networksetup -setsearchdomains Wi-Fi Empty
[+] Backgrounding route monitor

# 2. 查看状态(正确情况下)
lixin-macbook:~ lixin$ sudo wg show
interface: utun4
  public key: zmijB4jE1QMMkRtTe+doL+x8ONm3NOgF7HZbP4sTc3E=
  private key: (hidden)
  listening port: 49927

peer: pVtk+KUkNWcORxWYgPbrBkcr8SNKemAL75cJqs26124=
  preshared key: (hidden)
  endpoint: 8.210.32.174:50506
  allowed ips: 0.0.0.0/0, ::/0
  # ***************************************************
  # 能看到这个代表连接是通了
  # ***************************************************
  latest handshake: 7 seconds ago
  transfer: 897.90 KiB received, 202.21 KiB sent


# 3. 查看状态(错误的情况下)
lixin-macbook:~ lixin$ sudo wg show
interface: utun4
  public key: mctMsDFkGmzgMwWrhupMvlMEeCIudPNypzfnwyfzSCk=
  private key: (hidden)
  listening port: 63432

peer: VQph4ERk/tWP1wuanSpxb4U2PakFLMkV7/MdBHKtBjw=
  preshared key: (hidden)
  endpoint: 8.210.32.174:50506
  allowed ips: 0.0.0.0/0, ::/0
  transfer: 0 B received, 1.16 KiB sent

# 4. 关闭虚拟网络
lixin-macbook:test_db lixin$ sudo wg-quick down wg0
Password:
Warning: `/usr/local/etc/wireguard/wg0.conf' is world accessible
[+] Interface for wg0 is utun4
[#] rm -f /var/run/wireguard/utun4.sock
[#] rm -f /var/run/wireguard/wg0.name
```

### (9). 查看Client(Mac)网卡信息
```
lixin-macbook:~ lixin$ ifconfig
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=400<CHANNEL_IO>
	ether ac:bc:32:89:41:c9
	inet6 fe80::1844:8493:138c:7410%en0 prefixlen 64 secured scopeid 0x4
	inet 172.17.12.179 netmask 0xffff8000 broadcast 172.17.127.255
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect
	status: active
# ***********************************************************
# mac client的IP地址为:172.31.41.101
# ***********************************************************
utun4: flags=8051<UP,POINTOPOINT,RUNNING,MULTICAST> mtu 1420
	inet 172.31.41.101 --> 172.31.41.101 netmask 0xffffffff
	inet6 fe80::aebc:32ff:fe89:41c9%utun4 prefixlen 64 scopeid 0x10
	inet6 fd42:42:42::101 prefixlen 128
	nd6 options=201<PERFORMNUD,DAD>
```
### (10). 查看WireGuard服务器网卡信息
```
[root@iZj6cgalu8r574j7e3khydZ ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.31.41.22  netmask 255.255.240.0  broadcast 172.31.47.255
        inet6 fe80::216:3eff:fe06:b30a  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:06:b3:0a  txqueuelen 1000  (Ethernet)
        RX packets 4668  bytes 1644906 (1.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4811  bytes 1536692 (1.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# *****************************************************************
# WireGuard服务器网卡为:172.31.41.100
# *****************************************************************
wg0: flags=209<UP,POINTOPOINT,RUNNING,NOARP>  mtu 1420
        inet 172.31.41.100  netmask 255.255.255.0  destination 172.31.41.100
        inet6 fe80::35b9:19f3:e80c:87c5  prefixlen 64  scopeid 0x20<link>
        inet6 fd42:42:42::1  prefixlen 64  scopeid 0x0<global>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 1000  (UNSPEC)
        RX packets 1518  bytes 220728 (215.5 KiB)
        RX errors 2  dropped 0  overruns 0  frame 2
        TX packets 1287  bytes 941996 (919.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### (11). 验证
```
utun4: flags=8051<UP,POINTOPOINT,RUNNING,MULTICAST> mtu 1420
	inet 172.31.41.101 --> 172.31.41.101 netmask 0xffffffff

# 1. 本地mac(172.31.41.101) ping远程 WireGuard(172.31.41.100)
lixin-macbook:~ lixin$
lixin-macbook:~ lixin$ ping 172.31.41.100
PING 172.31.41.100 (172.31.41.100): 56 data bytes
64 bytes from 172.31.41.100: icmp_seq=0 ttl=64 time=241.171 ms
64 bytes from 172.31.41.100: icmp_seq=1 ttl=64 time=91.418 ms


# 2. 本地mac(172.31.41.101)ping google
lixin-macbook:~ lixin$ ping www.google.com
PING www.google.com (142.250.66.132): 56 data bytes
64 bytes from 142.250.66.132: icmp_seq=0 ttl=117 time=59.791 ms
64 bytes from 142.250.66.132: icmp_seq=1 ttl=117 time=85.304 ms
64 bytes from 142.250.66.132: icmp_seq=2 ttl=117 time=97.349 ms
```
### (12). Chrome 验证访问墙外世界
!["测试访问Google"](/assets/vpn/imgs/WireGuard.jpg)