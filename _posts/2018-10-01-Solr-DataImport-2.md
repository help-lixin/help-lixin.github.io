---
layout: post
title: 'Solr 支持MySQL导入(二)'
date: 2018-10-01
author: 李新
tags: Solr
---

### (1). 配置solrconfig.xml(solr_home/core_example/conf/solrconfig.xml )
```
<!-- add dataimport support -->
<requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
  <lst name="defaults">
      <str name="config">import-config.xml</str>
  </lst>
</requestHandler>
```

### (2). 创建import-config.xml(solr_home/core_example/conf/import-config.xml)
> <field column="" name=""/>   
> column : 代表表中的列   
> name   : 代表solr的域
>  

```
<?xml version="1.0" encoding="UTF-8" ?>
<dataConfig>
  <dataSource type="JdbcDataSource"
              driver="com.mysql.jdbc.Driver"
              url="jdbc:mysql://localhost:3306/solr"
              user="root"
              password="123456"/>
  <document>
      <entity name="products"  query="select pid,pname,catalog_name,price,description,picture from products ">
          <field column="pid" name="id"/>
          <field column="pname" name="prod_pname"/>
          <field column="catalog_name" name="prod_catalog_name"/>
          <field column="price" name="prod_price"/>
          <field column="description" name="prod_description"/>
          <field column="picture" name="prod_picture"/>
    </entity>
  </document>
</dataConfig>
```
### (3). 配置solr中的field(solr_home/core_example/conf/managed-schema)
> 在managed-schema里添加如下XML   
> prod_pname 支持模糊查询

```
<field name="id" type="string" indexed="true" stored="true" required="true" multiValued="false" />
<field name="prod_pname" type="text_smartcn" indexed="true" stored="true"/>
<field name="prod_catalog_name" type="string" indexed="true" stored="true"/>
<field name="prod_price" type="pdouble" indexed="true" stored="true"/>
<field name="prod_description" type="string" indexed="true" stored="true"/>
<field name="prod_picture" type="string" indexed="true" stored="true"/>
```
### (4). 启动服务

```
lixin-macbook:solr lixin$ ./apache-tomcat-8.5.54/bin/startup.sh
```

### (5). 执行导入
!["Solr导入MySQL数据"](/assets/solr/imgs/solr-import-mysql.png)

### (6). 执行查询
!["Solr Query"](/assets/solr/imgs/solr-query.png)


### (7). 查询描述
> q       : 查询的关键字,例如:
> fq(filter query)  : 作用是:在q查询符合结果中,同时fq查询符合的条件,例如:prod_price:[1 TO 10]/prod_price:[1 TO *]    
> fl      : 返回结果的定义,用逗号分隔,例如:id,name
> start   : 返回结果的第几条记录开始,一般分页用.默认值为:0  
> rows    : 指定返回的结果最多有多少条记录,默认值为:10  
> sort    : 排序方式,多个排序用逗号分隔,例如:id desc,代表按照id降序排序.  
> df(default domain)   : 默认域,指定域名称,省去q查询,需要填写域名称.
