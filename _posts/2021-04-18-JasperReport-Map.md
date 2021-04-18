---
layout: post
title: 'JasperReport 源码案例(Map)深入(四)'
date: 2021-04-18
author: 李新
tags:  JasperReport
---

### (1). 前言
> 这一小节,主要对前面导入工程的Map进行深入冲剖析.

### (2). 目的
> 我们先不要在乎jrxml和jasper是啥?先把官方的案例跑通并理解,通过这样的方式,可以增加你对这个框架的入门,否则,在门外徘徊太久,你会放弃.  

### (3). 先看下(map)工程结构
```
# 查看当前所在目录(demo/samples/map)
lixin-macbook:map lixin$ pwd
/Users/lixin/GitRepository/jasperreports/jasperreports/demo/samples/map

# 查看项目结构
lixin-macbook:map lixin$ tree
.
├── build
│   ├── classes
│   │   ├── MapApp.class
│   │   ├── jasperreports.properties
│   │   └── jasperreports_extension.properties
│   └── reports
├── build.xml
├── data
│   ├── CsvDataAdapter.xml
│   ├── CsvDataSource.csv
│   ├── PathLocationDataAdapter.xml
│   ├── PathLocationDataSource.csv
│   ├── PathStyleDataAdapter.xml
│   └── PathStyleDataSource.csv
├── docs
│   └── index.xml
├── ivy.xml
├── readme.txt
├── reports                                   # jrxml文件
│   ├── MapReport.jrxml                       # 在这里我只以这个文件作为案例分析.
│   ├── MapReport1.jrxml
│   ├── MapReport2.jrxml
│   ├── MapReport3.jrxml
│   ├── MapReport4.jrxml
│   └── MapReport5.jrxml
└── src                                       # 源码目录
    ├── MapApp.java                           # 仅仅一个MapApp.java文件.
    ├── jasperreports.properties
    └── jasperreports_extension.properties
```

### (4). MapReport.jrxml是需要经过编译成(MapReport.jasper).
> 注意:你也可以把.jrxml扔到JasperSoft Studio CE,进行编译,然后,再(.jasper)拿回到当前项目里.  
> 我这里,自己写了一个编译程序,对:MapReport.jrxml进行编译.  
> 后期,会有几篇专讲:JasperSoft Studio CE如何创建:jrxml报表.

```
import net.sf.jasperreports.engine.JasperCompileManager;

public class JrxmlCompile {
	public static void main(String[] args) throws Exception{
		// 1. 编译jrxml->jasper(会在同级目录产生一个.jasper文件).
		JasperCompileManager.compileReportToFile("/Users/lixin/GitRepository/jasperreports/jasperreports/demo/samples/map/reports/MapReport.jrxml");
		JasperCompileManager.compileReportToFile("/Users/lixin/GitRepository/jasperreports/jasperreports/demo/samples/map/reports/MapReport1.jrxml");
		JasperCompileManager.compileReportToFile("/Users/lixin/GitRepository/jasperreports/jasperreports/demo/samples/map/reports/MapReport2.jrxml");
		JasperCompileManager.compileReportToFile("/Users/lixin/GitRepository/jasperreports/jasperreports/demo/samples/map/reports/MapReport3.jrxml");
		JasperCompileManager.compileReportToFile("/Users/lixin/GitRepository/jasperreports/jasperreports/demo/samples/map/reports/MapReport4.jrxml");
		JasperCompileManager.compileReportToFile("/Users/lixin/GitRepository/jasperreports/jasperreports/demo/samples/map/reports/MapReport5.jrxml");
	}
}
```
### (5). MapApp(MapAppTest)
> 我这里对:MapApp抽取,只留下一部份值得参考的内容.因为其余部份变化的是:exportReportToXXXX.

```
import net.sf.jasperreports.engine.JREmptyDataSource;
import net.sf.jasperreports.engine.JRException;
import net.sf.jasperreports.engine.JasperExportManager;
import net.sf.jasperreports.engine.JasperFillManager;

public class MapAppTest {
	public static void main(String[] args) throws JRException {
		MapAppTest test = new MapAppTest();
		test.fill();
		test.pdf();
	}
	
	// 1. 填充数据
	public void fill() throws JRException {
		long start = System.currentTimeMillis();
		JasperFillManager.fillReportToFile("reports/MapReport.jasper", null, new JREmptyDataSource(5));
		System.err.println("Filling time : " + (System.currentTimeMillis() - start));
	}

    // 2. 导出成pdf
	public void pdf() throws JRException {
		long start = System.currentTimeMillis();
		JasperExportManager.exportReportToPdfFile("reports/MapReport.jrprint");
		System.err.println("PDF creation time : " + (System.currentTimeMillis() - start));
	}
}
```
### (6). 效果如下
!["jasper-report-map报表"](/assets/jasper-report/imgs/jasper-report-map-report.jpg)
!["jasper-report-map-pdf"](/assets/jasper-report/imgs/jasper-report-map-report-pdf.jpg)

### (7). 总结
> 从生产的报表结果来看:这个案例还是比较复杂的(用到了:自定义数据源读取CSV等).但是,代码能跑通,增加入门的信心.  
> 后面的章节,会是专心的.jrxml的学习了.  