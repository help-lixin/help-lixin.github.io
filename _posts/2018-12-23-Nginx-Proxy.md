---
layout: post
title: 'Nginx 正向代理'
date: 2018-12-22
author: 李新
tags: Nginx
---

### (1). 安装依赖库
```
[root@lixin ~]# yum -y install gcc gcc-c++  pcre pcre-devel zlib zlib-devel git patch
```
### (2). 创建nginx运行用户
```
[root@lixin]# useradd lixin
```
### (3). 安装的编译openssl
```
[root@lixin]# wget http://distfiles.macports.org/openssl/openssl-1.0.2h.tar.gz
[root@lixin]# tar -zxvf openssl-1.0.2h.tar.gz 
[root@lixin]# cd openssl-1.0.2h
[root@lixin openssl-1.0.2h]# ./config -fPIC --prefix=/usr/local/openssl-1.0.2h enable-shared enable-tlsext
[root@lixin openssl-1.0.2h]# make && make install
[root@lixin openssl-1.0.2h]# ln -s /usr/local/openssl-1.0.2h  /usr/local/openssl
```
### (4). 下载正向代理支持https模块
```
[root@lixin ~]# git clone https://github.com/chobits/ngx_http_proxy_connect_module.git
```
### (5). 下载Nginx
```
[root@lixin ~]# wget http://nginx.org/download/nginx-1.14.2.tar.gz
```
### (6). 解压和安装nginx
```
[root@lixin ~]# tar -zxvf nginx-1.14.2.tar.gz
[root@lixin ~]# cd nginx-1.14.2

# 打补丁
[root@lixin nginx-1.14.2]# patch -p1 < /root/ngx_http_proxy_connect_module/patch/proxy_connect.patch 

# configure
[root@lixin nginx-1.14.2]# ./configure --user=lixin --group=lixin --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-openssl=/root/openssl-1.0.2h --add-module=/root/ngx_http_proxy_connect_module/

[root@lixin nginx-1.14.2]#  make && make install

```
### (7). nginx配置
> /usr/local/nginx/conf/nginx.conf

```
# nginx配置
server {  
    resolver 8.8.8.8;
    listen 8081;

    proxy_connect;
    proxy_connect_allow            443 563;
    proxy_connect_connect_timeout  10s;
    proxy_connect_read_timeout     10s;
    proxy_connect_send_timeout     10s;

    # forward proxy for non-CONNECT request
    location / {
        proxy_pass http://$host;
        proxy_set_header Host $host;
    }
} 
```