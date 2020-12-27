---
layout: post
title: 'Tinyproxy 正向代理'
date: 2020-12-20
author: 李新
tags: Tinyproxy 正向代理
---

### (1). 下载
```
# 安装gcc
[root@lixin ~]# yum -y install gcc

# 下载:tinyproxy-1.11.0
[root@lixin ~]# wget https://github.com/tinyproxy/tinyproxy/releases/download/1.11.0-rc1/tinyproxy-1.11.0-rc1.tar.gz

# 解压:tinyproxy-1.11.0
[root@lixin ~]# tar -zxvf tinyproxy-1.11.0-rc1.tar.gz

```
### (2). 安装
```
# 添加用户和用户组
[root@lixin ~]# useradd lixin

# 进入安装目录
[root@lixin ~]# cd tinyproxy-1.11.0-rc1

[root@lixin tinyproxy-1.11.0-rc1]# ./autogen.sh 

# 编译并安装(/usr/local/bin/tinyproxy)
[root@lixin tinyproxy-1.11.0-rc1]# make && make install

# 创建配置目录
[root@lixin ~]# mkdir  -p /etc/tinyproxy

# 创建日志目录
[root@lixin ~]# mkdir -p  /var/log/tinyproxy/

# 给日专目录授权.
[root@lixin ~]# chown -R lixin:lixin /var/log/tinyproxy
```
### (3). 配置
> vi /etc/tinyproxy/tinyproxy.conf

```
# 用户和组
User lixin
Group lixin

# 监听端口
Port 8888

# 在多网卡的情况下，设置出口 IP 是否与入口 IP 相同。默认情况下是关闭的
BindSame yes

# 超时时间
Timeout 20

DefaultErrorFile "/usr/local/share/tinyproxy/default.html"

StatFile "/usr/local/share/tinyproxy/stats.html"

# 指定日志位置
LogFile "/var/log/tinyproxy/tinyproxy.log"

LogLevel Info

# 设置最大客户端链接数
MaxClients 500

ViaProxyName "tinyproxy"

# BindSame yes

# 多网卡的情况下,指定发送给上游代理的IP出口.
# Bind 192.168.1.22

# 权限校验
BasicAuth admin 123456
```
### (4). 启动
```
[root@lixin ~]# nohup tinyproxy -d -c /etc/tinyproxy/tinyproxy.conf 2>&1 &
```
### (5). CURL 测试访问
> curl测试访问.
```
[root@lixin ~]# curl -x 'http://admin:123456@47.241.71.47:8888' -v  https://www.lixin.help
* About to connect() to proxy 47.241.71.47 port 8888 (#0)
*   Trying 47.241.71.47... connected
* Connected to 47.241.71.47 (47.241.71.47) port 8888 (#0)
* Establish HTTP proxy tunnel to www.lixin.help:443
* Proxy auth using Basic with user 'admin'
> CONNECT www.lixin.help:443 HTTP/1.1
> Host: www.lixin.help:443
> Proxy-Authorization: Basic YWRtaW46MTIzNDU2
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.27.1 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.0 200 Connection established
< Proxy-agent: tinyproxy/1.11.0-rc1
< 
* Proxy replied OK to CONNECT request
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: /etc/pki/tls/certs/ca-bundle.crt
  CApath: none
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=www.lixin.help
* 	start date: 12月 24 16:50:54 2020 GMT
* 	expire date: 3月 24 16:50:54 2021 GMT
* 	common name: www.lixin.help
* 	issuer: CN=R3,O=Let's Encrypt,C=US
> GET / HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.27.1 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: www.lixin.help
> Accept: */*
> 
< HTTP/1.1 200 OK
< Connection: keep-alive
< Content-Length: 15242
< Content-Type: text/html; charset=utf-8
< Server: GitHub.com
< Last-Modified: Thu, 24 Dec 2020 13:36:49 GMT
< Access-Control-Allow-Origin: *
< ETag: "5fe49971-3b8a"
< Expires: Sat, 26 Dec 2020 23:41:56 GMT
< Cache-Control: max-age=600
< X-Proxy-Cache: MISS
< X-GitHub-Request-Id: 6B34:3B57:8B901A:97A60B:5FE7C7EB
< Accept-Ranges: bytes
< Date: Sat, 26 Dec 2020 23:32:05 GMT
< Via: 1.1 varnish
< Age: 10
< X-Served-By: cache-sin18020-SIN
< X-Cache: HIT
< X-Cache-Hits: 1
< X-Timer: S1609025526.824180,VS0,VE1
< Vary: Accept-Encoding
< X-Fastly-Request-ID: c8957baa31d21b14141389552120b577a295c0f2
```
### (6). Mac 配置代理

!["配置Proxy"](/assets/proxy/imgs/mac-setting-proxy.jpg)

!["查看IP"](/assets/proxy/imgs/ip-138-show.jpg)

### (7). HTTPS证书问题
> 通过Tinyproxy代理之后,HTTPS证书是显示正常的.