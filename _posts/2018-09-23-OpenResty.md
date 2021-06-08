---
layout: post
title: 'OpenResty 介绍'
date: 2017-06-04
author: 李新
tags: OpenResty
---

### (1). 
```
server {
	listen       8080;
	server_name  localhost;
	
	location ~ .*\.(js|css|png|jpg|gif|html|woff|ttf|svg|eot|map|ico|woff2)$ {
		root /home/data/html;
	}
	
	location / { 
		 # 把:http://localhost:8080/hello-service/test.do 重写为: /test.do
		 #  最后请求的URL为:http://127.0.0.1:9090/test.do
		 # $2 :  代表取第正则表达式的第2个参数
		 rewrite ^/(.*)-service/(.*)$ /$2 break;
		 proxy_pass http://127.0.0.1:9090;
	}
	
}
```
### (2). 

### (3). 

### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 

### (11). 

### (12). 
