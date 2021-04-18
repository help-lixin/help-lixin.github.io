---
layout: post
title: 'JasperReport 生命周期(三)'
date: 2021-04-18
author: 李新
tags:  JasperReport
---

### (1). JasperReport生命周期
> 在看JasperReport提供的案例源码前,需要先看下:JasperReport的生命周期.否则,无法理解为什么要这样做.

### (2). JasperReport生命周期图
!["JasperReport生命周期图"](/assets/jasper-report/imgs/jasper-report-life-cycle.jpeg)

### (3). Design Phase(设计阶段)
> 通过JasperSoft Studio CE,创建报表模板,模板包含:报表的布局与设计,包括执行计算的复杂公式、可选的从数据源获取数据的查询语句、以及其它的一些信息.模板设计完成之后,我们将模板保存为JRXML文件(JR代表JasperReports),其实JRXML就是一个XML文件.  
> JasperReport已经封装了一个dtd,只要按照规定的格式写这个xml文件,那么jasperReport就可以将其解析最终生成报表,但是jasperReport所解析的不是我们常见的.xml文件,而是.jrxml文件,其实跟xml是一样的,只是后缀不一样. 

### (4). Compile(编译阶段)
> 在这一步中JRXML被编译为二进制对象称为Jasper文件(*.jasper),做此编译是出于性能方面的考虑(后面,我会解剖jasper的内容).
> 解析完成后JasperReport就开始编译.jrxml文件,将其编译成.jasper文件,因为JasperReport只可以对*.jasper文件进行填充数据和转换,这步操作就跟我们java中将java文件编译成class文件是一样的.  

### (5). Execute Phase(执行阶段)
> 使用以JRXML文件编译为可执行的二进制文件(即.Jasper文件)结合数据进行执行,填充报表数据(即:<color ='red'>模板 + 数据</color>).
> 这一步才是JasperReport的核心所在,它会根据你在xml里面写好的查询语句来查询指定是数据库(也可以控制在后台编写查询语句,参数,数据库).  
> 在报表填充完后,会再生成一个.jrprint格式的文件(读取jasper文件进行填充,然后生成一个jrprint文件) 

### (6). Export Phase(导出阶段)
> 对执行阶段产生的结果进行展示,而展示的方式多种多样(HTML/PDF/XLS...).

### (7). 总结
> 为什么要了解生命周期,生命周期包含着:开发步骤.  
> 1. 通过:JasperSoft Studio CE进行设计报表模板(.jrxml).   
> 2. 通过JasperSoft Studio CE对报表进行编译(.jrxml->.jasper).   
> 3. 业务代码,通过JPrint读取:.jasper生成.jasper.     
> 4. 业务代码,选择导出的格式(PDF/DOC/HTML).   