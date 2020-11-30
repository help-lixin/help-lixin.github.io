---
layout: post
title: 'ElasticSearch Head下载和安装(三)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 下载head插件

> git clone https://github.com/mobz/elasticsearch-head.git    

### (2). 安装grunt
```
npm install -g grunt-cli
```

### (3). 编译head
```
cd  elasticsearch-head
npm install
```

### (4). Unexpected end of JSON input while parsing near
Unexpected end of JSON input while parsing near '...CGsOGUiu9\nmdUUNwK3IF'

```
# 清除缓存
npm cache clean --force
# 禁用淘宝仓库
npm set registry https://registry.npmjs.org/
```

### (5). 修改ES配置文件,支持跨域请求
> 修改:elasticsearch-5.6.16/config/elasticsearch.yml

```
# 是否支持跨域
http.cors.enabled: true

# *表示支持所有域名
http.cors.allow-origin: "*"
```

### (6). 启动head插件
> npm run start

### (7). 访问(http://localhost:9100/)

!["Head UI"](/assets/elasticsearch/imgs/elastic-head-ui.png)


!["Head 添加索引库"](/assets/elasticsearch/imgs/elasticsearch-head-add-index.png)