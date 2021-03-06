---
layout: post
title: 'JasperReport 介绍(一)'
date: 2021-04-25
author: 李新
tags:  JasperReport
---

### (1). JasperReport是什么?
> JasperReport是一个强大、灵活的"报表生成工具",能够展示丰富的页面内容,并将之转换成PDF/HTML/XML格式.该库完全由Java写成,可以用于在各种Java应用程序,包括J2EE Web应用程序中生成动态内容.只需要将JasperReport引入工程中即可完成PDF报表的编译、显示、输出等工作.    

### (2). JasperSoft Studio CE是什么?
> JasperSoft Studio是JasperReports基于Eclipse的设计器,它可以作为Eclipse插件或作为独立的应用程序使用.  
> Jaspersoft Studio允许您创建包含图表,图像,子报表,交叉表等的复杂布局.   
> 你可以通过JDBC,TableModels,JavaBeans,XML,Hibernate,Hive,CSV,XML以及自定义来源等各种来源访问数据.  
> 然后将报告发布为:PDF,RTF,XML,XLS,CSV,HTML,XHTML,文本,DOCX/OpenOffice.    

> ["JasperSoft Studio CE下载地址"](https://community.jaspersoft.com/community-download)   

### (3). JasperReport索引目录
> ["JasperReport 源码(map)导入Eclipse(二)"](/2021/04/18/JasperReport-Demo-Import.html)     
> ["JasperReport 生命周期(三)"](/2021/04/18/JasperReport-Life-Cycle.html)     
> ["JasperReport 源码案例(Map)学习(四)"](/2021/04/18/JasperReport-Map.html)     
> ["JasperReport 简单案例:alterdesign学习(五)"](/2021/04/18/JasperReport-alterdesign.html)    

### (4). JasperReport总结
> JasperReport的缺点是什么?   
> 1. 生命周期过于复杂.    
> 2. JasperReport框架太重了(需要设计/编译/执行/导出,不太理解为什么这样设计?).     
> 3. 在我的视角里,无非不过就是抽象成三个内容(选择数据源/执行业务逻辑/选择输出模板).      