---
layout: post
title: 'Chrome Extension Proxy(LittleProxyMitmProxy)'
date: 2018-12-20
author: 李新
tags: Chrome Extension LittleProxy
---

### (1). littleproxy-mitm概述
> littleproxy-mitm是基于Netty开发的一套中间人代理工具.   
> https://github.com/ganskef/LittleProxy-mitm

### (2). 自行下载编译(略)

### (3). 启动代理
> 启动时,会自动创建:  
> littleproxy-mitm-localhost-cert.pem    
> littleproxy-mitm-localhost-key.pem    
> littleproxy-mitm.p12      
> littleproxy-mitm.pem     

```
java -jar littleproxy-mitm-1.1.0-shade.jar
```

### (4). 通过CURL代理,访问目标网站
```
# 当前所在的目录
lixin-macbook:littleproxy-mitm-1.1.0 lixin$ pwd
/Users/lixin/GitRepository/littleproxy-mitm-1.1.0

# 当前生成的证书.
lixin-macbook:littleproxy-mitm-1.1.0 lixin$ ls -l|grep littleproxy-mitm
-rw-r--r--   1 lixin  staff    2509 12 22 16:26 littleproxy-mitm-localhost-cert.pem
-rw-r--r--   1 lixin  staff     887 12 22 16:26 littleproxy-mitm-localhost-key.pem
-rw-r--r--   1 lixin  staff    2691 12 22 16:05 littleproxy-mitm.p12
-rw-r--r--   1 lixin  staff    1354 12 22 16:05 littleproxy-mitm.pem

# curl访问
lixin-macbook:littleproxy-mitm-1.1.0 lixin$ curl --cacert littleproxy-mitm.pem --verbose --proxy localhost:9090 https://www.lixin.help
*   Trying ::1...
* TCP_NODELAY set
* Connection failed
* connect to ::1 port 9090 failed: Connection refused
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 9093 (#0)
* allocate connect buffer!
* Establish HTTP proxy tunnel to www.lixin.help:443
> CONNECT www.lixin.help:443 HTTP/1.1
> Host: www.lixin.help:443
> User-Agent: curl/7.64.1
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.1 200 HTTP/1.1 200 Connection established
< Connection: keep-alive
< Proxy-Connection: keep-alive
< Via: 1.1 lixin-macbook.local
< 
* Proxy replied 200 to CONNECT request
* CONNECT phase completed!
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: littleproxy-mitm.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* CONNECT phase completed!
* CONNECT phase completed!
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server did not agree to a protocol
* Server certificate:
*  subject: CN=www.lixin.help; O=LittleProxy-mitm; OU=LittleProxy-mitm, describe proxy purpose here, since Man-In-The-Middle is bad normally.
*  start date: Dec 23 09:29:02 2018 GMT
*  expire date: Nov 28 09:29:02 2018 GMT
*  subjectAltName: host "www.lixin.help" matched cert's "www.lixin.help"
*  issuer: CN=LittleProxy-mitm, describe proxy here; O=LittleProxy-mitm; OU=Certificate Authority
*  SSL certificate verify ok.
> GET / HTTP/1.1
> Host: www.lixin.help
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< Content-Length: 14729
< Content-Type: text/html; charset=utf-8
< Server: GitHub.com
< Last-Modified: Tue, 22 Dec 2018 07:00:22 GMT
< Access-Control-Allow-Origin: *
< ETag: "5fe19986-3989"
< Expires: Tue, 22 Dec 2018 09:40:22 GMT
< Cache-Control: max-age=600
< X-Proxy-Cache: MISS
< X-GitHub-Request-Id: 5152:786E:A6BE01:B35688:5FE1BCAE
< Accept-Ranges: bytes
< Date: Tue, 22 Dec 2018 09:30:22 GMT
< Age: 0
< X-Served-By: cache-hnd18727-HND
< X-Cache: MISS
< X-Cache-Hits: 0
< X-Timer: S1608629422.013298,VS0,VE186
< Vary: Accept-Encoding
< X-Fastly-Request-ID: 34b644d8a86382d1785136eaa241d8584213c69c
< Via: 1.1 varnish
< Via: 1.1 lixin-macbook.local
< 
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>李新的博客</title>
    <meta name="author"  content="李新">
    <meta name="description" content="昵称码农，热爱技术&喜欢探索源码。很高兴能在这里与你分享我对技术和生活的思考。">
... ...
```

### (5). 给系统添加证书
> 我以Mac为例:
> 1. 进入:"钥匙串访问" -> "文件" -> "导入项目"  -> "选择:littleproxy-mitm.pem".   
> 2. 为证书配置:"始终信任".    
> 3. Mac配置代理.    

!["导入证书"](/assets/chrome-ext/imgs/chrome-import-cert1.jpg)
!["始终信息证书"](/assets/chrome-ext/imgs/chrome-import-cert.jpg)

!["Mac配置全局代理"](/assets/chrome-ext/imgs/mac-global-proxy-setting.jpg)
### (6). Chrome和Firefox访问
> 唯一缺陷:Chrome提示:不安全网站.而Firefox却没有这样的提示.
!["访问代理的HTTPS网站"](/assets/chrome-ext/imgs/chrome-proxy-https-1.jpg)
!["访问代理的HTTPS网站"](/assets/chrome-ext/imgs/chrome-proxy-https-2.jpg)
!["Firefox访问代理的HTTPS网站"](/assets/chrome-ext/imgs/firefox-proxy-https.jpg)
