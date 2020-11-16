---
layout: post
title: 'SolrCloud集群'
date: 2018-10-01
author: 李新
tags: Solr
---

### (1). 先创建单机

 > 请自行参考(https://www.lixin.help/2018/10/01/Solr-Tomcat-Integration.html)  

### (2). SolrCloud规划

Tomcat名称 | 端口信息 | SOLR-HOME
- | :-: | :-: |
apache-tomcat-8.5.54   | 9005/<font color='red'>9090</font>/9443   |  solr_home
apache-tomcat-8.5.54-2 | 7005/<font color='red'>7070</font>/7443   |  solr_home2
apache-tomcat-8.5.54-3 | 6005/<font color='red'>6060</font>/6443   |  solr_home3
apache-tomcat-8.5.54-4 | 5005/<font color='red'>5050</font>/5443   |  solr_home4

### (3). 克隆单机Solr(Tomcat)

```
# 当前工作目录
lixin-macbook:solr lixin$ pwd
/Users/lixin/Developer/solr

# 单机版Solr信息
lixin-macbook:solr lixin$ ll
drwxr-xr-x  17 lixin  staff   544 11 14 15:07 apache-tomcat-8.5.54/
drwxr-xr-x  10 lixin  staff   320 11 14 17:01 solr_home/


lixin-macbook:solr lixin$ cp -rf apache-tomcat-8.5.54 apache-tomcat-8.5.54-2
lixin-macbook:solr lixin$ cp -rf apache-tomcat-8.5.54 apache-tomcat-8.5.54-3
lixin-macbook:solr lixin$ cp -rf apache-tomcat-8.5.54 apache-tomcat-8.5.54-4

# 先清空单机版创建的:core_example,否则启动会报错
lixin-macbook:solr lixin$ rm -rf solr_home/core_example

lixin-macbook:solr lixin$ cp -rf solr_home solr_home2
lixin-macbook:solr lixin$ cp -rf solr_home solr_home3
lixin-macbook:solr lixin$ cp -rf solr_home solr_home4

```
### (4). 修改所有Tomcat的端口号

> 省略

### (5). 修改所有Tomcat的web.xml

> <font color='red'>apache-tomcat-8.5.54</font>   

```
<env-entry>
    <env-entry-name>solr/home</env-entry-name>
    <env-entry-value>/Users/lixin/Developer/solr/solr_home</env-entry-value>
    <env-entry-type>java.lang.String</env-entry-type>
</env-entry>
```

> <font color='red'>apache-tomcat-8.5.54-2</font>   

```
<env-entry>
    <env-entry-name>solr/home</env-entry-name>
    <env-entry-value>/Users/lixin/Developer/solr/solr_home2</env-entry-value>
    <env-entry-type>java.lang.String</env-entry-type>
</env-entry>
```

> <font color='red'>apache-tomcat-8.5.54-3</font>   

```
<env-entry>
    <env-entry-name>solr/home</env-entry-name>
    <env-entry-value>/Users/lixin/Developer/solr/solr_home3</env-entry-value>
    <env-entry-type>java.lang.String</env-entry-type>
</env-entry>
```

> <font color='red'>apache-tomcat-8.5.54-4</font>  

```
<env-entry>
    <env-entry-name>solr/home</env-entry-name>
    <env-entry-value>/Users/lixin/Developer/solr/solr_home4</env-entry-value>
    <env-entry-type>java.lang.String</env-entry-type>
</env-entry>
```

### (6). Zookeeper搭建

> 省略(**发现ZK必须要有三台,否则有节点启动无法提供服务**)  

### (7). 配置所有SOLR-HOME/solr.xml
>  把solr.xml与IP和端口做映射配置  

> <font color='red'>apache-tomcat-8.5.54</font>  

```
<str name="host">127.0.0.1</str>
<int name="hostPort">9090</int>
```

> <font color='red'>apache-tomcat-8.5.54-2</font> 

```
<str name="host">127.0.0.1</str>
<int name="hostPort">7070</int>
```

> <font color='red'>apache-tomcat-8.5.54-3</font> 

```
<str name="host">127.0.0.1</str>
<int name="hostPort">6060</int>
```

> <font color='red'>apache-tomcat-8.5.54-4</font>  

```
<str name="host">127.0.0.1</str>
<int name="hostPort">5050</int>
```
### (8). 上传配置到ZK
> 启动ZK(略) 
> 上传配置到ZK中 

```
~/Developer/solr/solr-7.7.3/server/scripts/cloud-scripts/zkcli.sh -zkhost 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183 -cmd upconfig -confdir /Users/lixin/Developer/solr/solr_home/configsets/sample_techproducts_configs/conf/ -confname solrconf
```

> 说明:  
> ~/Developer/solr/solr-7.7.3/server/scripts/cloud-scripts/zkcli.sh              :  Solr提供访问ZK的脚本   
> -zkhost       :  为zookeeper地址   
> -cmd upconfig :  指定上传配置  
> -confdir      :  指定要上传的配置文件目录,我们上传Solr的样例中的配置   
> -confname     :  指定注册到Zookeeper中后配置文件目录名称   

> ZK检查

```
[zk: localhost:2181(CONNECTED) 14] ls /configs 
[solrconf]
```
### (9). Tomcat关联Zookeeper

> 修改所有的Tomcat目录下的catalina.sh文件.添加如下内容:  

```
JAVA_OPTS="-DzkHost=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183"
```
### (10). 创建Collection
> 添加Collection  

!["Add Collection"](/assets/solr/imgs/solr-add-collection.jpg)

!["Add Collection Result"](/assets/solr/imgs/solr-add-collection-result.png)

!["SolrCloud Graph"](/assets/solr/imgs/solrcloud-graph.png)
