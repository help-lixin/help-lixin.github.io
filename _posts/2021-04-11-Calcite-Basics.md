---
layout: post
title: 'Calcite 介绍(一)'
date: 2021-04-30
author: 李新
tags:  Calcite
---

### (1). Calcite是什么?
> Apache Calcite是一款开源SQL解析工具,可以将各种SQL语句解析成抽象语法术AST(Abstract Syntax Tree),之后通过操作AST就可以把SQL中所要表达的算法与关系体现在具体代码之中.    
> 说大白话一点就是:Calcite是一个可以将*结构化的数据*,通过SQL进行检索.  
> 在Calcite的接口中,Schema和Table是数据关系化中最重要的两个接口.    
> Schema是对:Database的抽象,以兼容已经存在的各类数据库.    
> Table是对:表/视图/流的抽象,以兼容数据的各种场景.   

### (2). Schema
> Calcite利用Schema的层级关系,构造出来DataBase的概念,Schema是一个树形结构,这样设计的目的:让Schema有继承的功能.  

!["Calcite Schema"](/assets/calcite/imgs/calcite-scheam.jpg)
### (3). Table
> Table是schema的核心属性,一个Schema拥有多个Table,这就像一个数据库中有很多表一样.而Table的概念更为广泛,为了兼容到各类数据库或者消息队列,Calcite将Table类型细分为TableType.基本的类似传统关系型数据库中的表或者视图,流式的Stream等.  
> 另外对数据协议的兼容是非常重要的,像:json/csv/xml等等,Table抽象出了RelDataType接口,目的是将应用层的数据协议转关系化,从而可以为sql服务.  
> 拿csv格式的数据来说:假设csv数据的每一行数据和Table中的每一行一一对应,那么在关系化的过程中,必须将csv中每个字段的类型及一些元数据定义清晰,比如:字段是int类型还是long类型,主键是哪个字段,外键是哪个字段等.calcite提供了几乎所有已存在的字段类型. 

### (4). 学习目录
> ["Calcite 通过SQL读取CSV案例入门(一)"](/2021/04/11/Calcite-SQL-CSV.html)
> ["Calcite 通过SQL读取CSV案例入门(二)"](/2021/04/11/Calcite-SQL-ES.html)

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 
