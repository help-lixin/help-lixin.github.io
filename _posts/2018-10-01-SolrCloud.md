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
### (9). Tomcat配置Zookeeper

> 修改所有的Tomcat目录下的catalina.sh文件.添加如下内容:  

```
JAVA_OPTS="-DzkHost=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183"
```
### (10). 创建Collection
> 添加Collection  

!["Add Collection"](/assets/solr/imgs/solr-add-collection.jpg)

!["Add Collection Result"](/assets/solr/imgs/solr-add-collection-result.png)

!["SolrCloud Graph"](/assets/solr/imgs/solrcloud-graph.png)


### (12). 添加中文分词(IK)器支持
> github下载IK(https://github.com/magese/ik-analyzer-solr)  
> 编译ik-analyzer-solr  

```
# 1.ik-analyzer-solr编译后目录
lixin-macbook:target lixin$ pwd
/Users/lixin/GitRepository/ik-analyzer-solr-v7.7.1/target

# 2.拷贝ik-analyzer-7.7.1.jar到Tomcat下(/WEB-INF/lib/)
lixin-macbook:target lixin$ cp ik-analyzer-7.7.1.jar ~/Developer/solr/apache-tomcat-8.5.54/webapps/solr/WEB-INF/lib/
lixin-macbook:target lixin$ cp ik-analyzer-7.7.1.jar ~/Developer/solr/apache-tomcat-8.5.54-2/webapps/solr/WEB-INF/lib/
lixin-macbook:target lixin$ cp ik-analyzer-7.7.1.jar ~/Developer/solr/apache-tomcat-8.5.54-3/webapps/solr/WEB-INF/lib/
lixin-macbook:target lixin$ cp ik-analyzer-7.7.1.jar ~/Developer/solr/apache-tomcat-8.5.54-4/webapps/solr/WEB-INF/lib/

# 3.拷贝(IKAnalyzer.cfg.xml/ext.dic/stopword.dic)到Tomcat(WEB-INF/classes/) 
lixin-macbook:target lixin$ cp classes/IKAnalyzer.cfg.xml classes/ext.dic classes/stopword.dic ~/Developer/solr/apache-tomcat-8.5.54/webapps/solr/WEB-INF/classes/
lixin-macbook:target lixin$ cp classes/IKAnalyzer.cfg.xml classes/ext.dic classes/stopword.dic ~/Developer/solr/apache-tomcat-8.5.54-2/webapps/solr/WEB-INF/classes/
lixin-macbook:target lixin$ cp classes/IKAnalyzer.cfg.xml classes/ext.dic classes/stopword.dic ~/Developer/solr/apache-tomcat-8.5.54-3/webapps/solr/WEB-INF/classes/
lixin-macbook:target lixin$ cp classes/IKAnalyzer.cfg.xml classes/ext.dic classes/stopword.dic ~/Developer/solr/apache-tomcat-8.5.54-4/webapps/solr/WEB-INF/classes/


# 4.拷贝(ik.conf/dynamicdic.txt)放入solr配置文件夹中,与solr的managed-schema文件同目录中(**实际数据是存储在ZK中**),该步骤可忽略.
lixin-macbook:target lixin$ cp classes/ik.conf classes/dynamicdic.txt ~/Developer/solr/solr_home/configsets/sample_techproducts_configs/conf/
lixin-macbook:target lixin$ cp classes/ik.conf classes/dynamicdic.txt ~/Developer/solr/solr_home2/configsets/sample_techproducts_configs/conf/
lixin-macbook:target lixin$ cp classes/ik.conf classes/dynamicdic.txt ~/Developer/solr/solr_home3/configsets/sample_techproducts_configs/conf/
lixin-macbook:target lixin$ cp classes/ik.conf classes/dynamicdic.txt ~/Developer/solr/solr_home4/configsets/sample_techproducts_configs/conf/


# 当前工作目录
lixin-macbook:solr lixin$ pwd
/Users/lixin/Developer/solr

# 5.拷贝ik.conf到zk中
./solr-7.7.3/server/scripts/cloud-scripts/zkcli.sh -zkhost 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183 -cmd putfile /configs/solrconf/ik.conf /Users/lixin/Developer/solr/solr_home/configsets/sample_techproducts_configs/conf/ik.conf

# 6.拷贝dynamicdic.txt到zk中
lixin-macbook:solr lixin$ ./solr-7.7.3/server/scripts/cloud-scripts/zkcli.sh -zkhost 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183 -cmd putfile /configs/solrconf/dynamicdic.txt /Users/lixin/Developer/solr/solr_home/configsets/sample_techproducts_configs/conf/dynamicdic.txt
```

> 修改ZK配置(/configs/solrconf/managed-schema),添加中文分启器

```
<!-- ik分词器 -->
<fieldType name="text_ik" class="solr.TextField">
  <analyzer type="index">
      <tokenizer class="org.wltea.analyzer.lucene.IKTokenizerFactory" useSmart="false" conf="ik.conf"/>
      <filter class="solr.LowerCaseFilterFactory"/>
  </analyzer>
  <analyzer type="query">
      <tokenizer class="org.wltea.analyzer.lucene.IKTokenizerFactory" useSmart="true" conf="ik.conf"/>
      <filter class="solr.LowerCaseFilterFactory"/>
  </analyzer>
</fieldType>
```

> 重启SolorCloud,或<font color='red'>Reload</font>

### (13). 测试IK分词器

!["SolrColud IK分词器测试"](/assets/solr/imgs/solr-ik-analysis.png)
