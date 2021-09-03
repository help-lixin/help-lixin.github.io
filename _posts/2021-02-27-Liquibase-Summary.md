---
layout: post
title: 'Liquibase 简介(一)'
date: 2021-02-27
author: 李新
tags: Liquibase
---

### (1). Liquibase是什么?
> Liquibase是一个用于跟踪,管理和应用数据库变化的开源的数据库重构工具.它将所有数据库的变化(包括结构和数据)都保存在XML文件中,便于版本控制.  
> 我来说几个痛点:   
> 1. 在发版时,DBA需要每个Schema执行相同SQL,Schema的增加,DBA负担也增加.  
> 2. 一个企业,如果没有很好的架构规范,随着时间,慢慢会发现:业务模型经常与表结构不一致,所以,也需要有一套东西,能解决表与业务模型的一致性.      
> 3. 是否还有其它解决方案?其实是有的,比如:Hibernate/Flyway.   

### (2). Hibernate解决方案
> 1. 在Entity增加注解,知道这个Entity属于哪个业务系统,同时,有映射信息记录:业务系统-->DataSource的关系.        
> 2. 扫描Entity上的注解,并进行分组.  
> 3. 遍历数据源(DataSource),创建Hibernate实例对象,Hibernate实例需要数据源和Entity集合.  
> 4. 配置(hibernate.hbm2ddl.auto=update),让Hibernate帮忙生成脚本(CREATE/ALTER/).  
> 5. 缺点:业务系统实际使用不到Hibernate,但是,却引入了这么大一个工程,确实有点不值得.  

### (3). Flyway解决方案
> Flyway与Liquibase差不多,但是,在业务模型的表现上没有Liquibase那么优秀,什么意思呢?  
> Flyway我所看到的只支持SQL文件,而Liquibase支持:SQL文件/XML/JSON/YAML...   
> 这是我选择Liquibase的原因,当然,最重要的是:SQL文件,无法解决跨平台的支持(MySQL/Oracle...),而XML文件,框架底层可以根据不同平台,生成不同的SQL语句.  
> 如果你想自定义业务模型的表现层(比如:从远程Rest请求,再渲染),实现:ChangeLogParser接口即可.   

### (4). Liquibase索引目录