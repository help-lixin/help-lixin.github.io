---
layout: post
title: 'Spring Boot启用AccessLog'
date: 2020-11-12
author: 李新
tags: SpringBoot
---

### (1). Spring Boot启用AccessLog
```
# 配置日志目录
server.tomcat.basedir=.
# 启用accesslog
server.tomcat.accesslog.enabled=true
# 日志输出目录:basedir+directory=./logs
server.tomcat.accesslog.directory=logs
# 输出表达式
server.tomcat.accesslog.pattern=%t %h %r%q %s %b %D
# 启用缓存
server.tomcat.accesslog.buffered=true
# 日志文件名称
server.tomcat.accesslog.prefix=access_log
# 日志文件名称(./logs/access_log_${yyyy-MM-dd})
server.tomcat.accesslog.file-date-format=.yyyy-MM-dd
# 日志文件名称
server.tomcat.accesslog.suffix=.log
# 是否延迟在文件名中包含日期戳，直到旋转时间
server.tomcat.accesslog.rename-on-rotate=false
# 设置请求的IP地址，主机名，协议和端口的请求属性
server.tomcat.accesslog.request-attributes-enabled=false
# 是否启用访问日志轮换
server.tomcat.accesslog.rotate=true
```

### (2). Pattern的配置
```
%a - Remote IP address,远程ip地址,注意不一定是原始ip地址,中间可能经过nginx等的转发
%A - Local IP address,本地ip
%b - Bytes sent, excluding HTTP headers, or '-' if no bytes were sent
%B - Bytes sent, excluding HTTP headers
%h - Remote host name (or IP address if enableLookups for the connector is false),远程主机名称(如果resolveHosts为false则展示IP)
%H - Request protocol,请求协议
%l - Remote logical username from identd (always returns '-')
%m - Request method，请求方法（GET，POST）
%p - Local port,接受请求的本地端口
%q - Query string (prepended with a '?' if it exists, otherwise an empty string
%r - First line of the request,HTTP请求的第一行（包括请求方法，请求的URI）
%s - HTTP status code of the response，HTTP的响应代码,如:200,404
%S - User session ID
%t - Date and time, in Common Log Format format，日期和时间，Common Log Format格式
%u - Remote user that was authenticated
%U - Requested URL path
%v - Local server name
%D - Time taken to process the request, in millis，处理请求的时间，单位毫秒
%T - Time taken to process the request, in seconds，处理请求的时间，单位秒
%I - current Request thread name (can compare later with stacktraces)，当前请求的线程名，可以和打印的log对比查找问题.

```
### (3). Pattern获取cookie/header/session内容
```
Access log 也支持将cookie、header、session或者其他在ServletRequest中的对象信息打印到日志中，其配置遵循Apache配置的格式（{xxx}指值的名称:
%{xxx}i for incoming headers，request header信息
%{xxx}o for outgoing response headers，response header信息
%{xxx}c for a specific cookie
%{xxx}r xxx is an attribute in the ServletRequest
%{xxx}s xxx is an attribute in the HttpSession
%{xxx}t xxx is an enhanced SimpleDateFormat pattern (see Configuration Reference document for details on supported time patterns)

```
### (4). Pattern内置模板
> Access log内置了两个日志格式模板,可以直接指定pattern为模板名称,如:    

```
server.tomcat.accesslog.pattern=common
server.tomcat.accesslog.pattern=combined
```

```
common - %h %l %u %t "%r" %s %b
  远程主机名称
  远程用户名
  被认证的远程用户
  日期和时间
  请求的第一行
  response code
  发送的字节数
```

```
combined - %h %l %u %t "%r" %s %b "%{Referer}i" "%{User-Agent}i"
  远程主机名称
  远程用户名
  被认证的远程用户
  日期和时间
  请求的第一行
  response code
  发送的字节数
  request header的Referer信息
  request header的User-Agent信息
```

### (5). Pattern常用的配置
```
%t %a "%r" %s (%D ms)
  日期和时间
  请求来自的IP(不一定是原始IP)
  请求第一行
  response code
  响应时间(毫秒)

%t [%I] %{X-Forwarded-For}i %a %r %s (%D ms)
  日期和时间
  线程名 
  原始IP 
  请求来自的IP(不一定是原始IP)
  请求第一行
  response code
  响应时间(毫秒)
```