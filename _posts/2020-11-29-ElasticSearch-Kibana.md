---
layout: post
title: 'ElasticSearch Kibana安装(五)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 下载kibana(略)
> https://artifacts.elastic.co/downloads/kibana/kibana-7.1.0-darwin-x86_64.tar.gz    
> <font color='red'>注意:kibana版本要和es版本是一致,否则,kibana运行不起来.</font>   

```
# 工作目录
lixin-macbook:kibana-7.1.0-darwin-x86_64 lixin$ pwd
/Users/lixin/Developer/elastic-search/kibana-7.1.0-darwin-x86_64
```
### (2). 启动kibana
```
lixin-macbook:kibana-7.1.0-darwin-x86_64 lixin$ ./bin/kibana
  log   [08:45:05.969] [info][task_manager] Installing .kibana_task_manager index template version: 7010099.
  log   [08:45:06.058] [info][task_manager] Installed .kibana_task_manager index template: version 7010099 (API version 1)
  log   [08:45:06.969] [info][migrations] Creating index .kibana_1.
  log   [08:45:07.115] [info][migrations] Pointing alias .kibana to .kibana_1.
  log   [08:45:07.160] [info][migrations] Finished in 201ms.
  # 启动并监听:5601端口
  log   [08:45:07.162] [info][listening] Server running at http://localhost:5601
  log   [08:45:07.258] [info][status][plugin:spaces@7.1.0] Status changed from yellow to green - Ready
```
### (3). kibana界面
!["Kibana界面"](/assets/elasticsearch/imgs/kibana-home.jpg)

!["Kibana DevTools"](/assets/elasticsearch/imgs/kibana-devtools.jpg)
