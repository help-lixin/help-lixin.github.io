---
layout: post
title: 'Lucene Luke查看建立索引后的数据结构(三)'
date: 2018-09-10
author: 李新
tags: Lucene
---


### (1). 建立索引后,通过Luke查看索引数据结构
> Luke下载地址:   
> https://github.com/DmitryKey/luke/releases 

### (2). 运行
!["luke查看索引目录"](/assets/lucene/imgs/luke-ui.jpg)



### (3). luke overriew
> Number Of Documents : Document的总数量   
> Number Of Fields    : Document中总共的域数量(Field)   
> Number Of Terms     : Document中进行分词后,总共切分了多少词.


!["overrew"](/assets/lucene/imgs/luke-overriew.png)

### (4). luke documents
> 查看所有的文档

!["Luke Documents"](/assets/lucene/imgs/luke-documents.jpg)

### (5). luke search
> 对文档进行检索

!["Luke search"](/assets/lucene/imgs/luke-search.jpg)

### (6). 总结
当Lucene建立索引后,可能过Luke查看最后的索引内容(数据结构). 