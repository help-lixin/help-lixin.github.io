---
layout: post
title: 'Lucene Analyzer分词器(五)'
date: 2018-09-10
author: 李新
tags: Lucene
---

### (1). Analyzer分词器
> 分词器会在什么情况下使用:   
> 当保存一个Document时,会根据:是否分词,调用:Analyzer对相应的Field Value进行分词.    
> 当向Lucene发起检索功能时,需要调用相应的Analyzer对:待检索内容(用户要检索的内容),进行分词.与索引库的进行比较,检索出结果.     
> <font color='red'>注意:这时候保存用的分词器要和检索用的分词吕一样的,否则,会出现结果不一致.</font> 

### (2). Lucene内置分词器种类
> StandardAnalyzer    
> WhitespaceAnalyzer      
> SimpleAnalyzer    
> CJKAnalyzer   

### (5). StandardAnalyzer

> 按空格进行分词,可以对英文进行分词,对中文是按单个字进行分词. 

### (6). WhitespaceAnalyzer
> 仅仅是去掉了空格,没有任何的操作,不支持中文.

### (7). SimpleAnalyzer
> 将字母以外的符号全部去除(包括数字,同样不支持中文),并且将所有的字母变成小写.  

### (8). CJKAnalyzer

### (9). 总结
> 分词器的目的是按照一定的算法,将"原始数据"进行切割,保存到索引库里.后续会分析这部份的源码. 
