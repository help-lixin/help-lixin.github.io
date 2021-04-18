---
layout: post
title: 'JasperReport 简单案例:alterdesign学习(五)'
date: 2021-04-18
author: 李新
tags:  JasperReport
---

### (1). 前言
> 前面的map案例太复杂了,在这里选一个简单的案例(alterdesign),顺便可以对:JasperSoft Studio CE有一个简单的了解.

### (2). 先看下(alterdesign)工程目录
```
lixin-macbook:alterdesign lixin$ tree
.
├── bin
│   └── AlterDesignApp.class
├── readme.txt
├── reports            # jrxml文件
│   └── AlterDesignReport.jrxml
└── src                # java源文件
    └── AlterDesignApp.java
```
### (3). AlterDesignReport.jrxml
> AlterDesignReport.jrxml是由Eclipse设计器进行设计的,我们看下:AlterDesignReport.jrxml的内容

```
<?xml version="1.0" encoding="UTF-8"?>
<jasperReport xmlns="http://jasperreports.sourceforge.net/jasperreports" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://jasperreports.sourceforge.net/jasperreports http://jasperreports.sourceforge.net/xsd/jasperreport.xsd" name="AlterDesignReport" pageWidth="595" pageHeight="842" whenNoDataType="AllSectionsNoDetail" columnWidth="555" leftMargin="20" rightMargin="20" topMargin="30" bottomMargin="30" uuid="bfcdbee3-9db9-4688-bc22-273a1901e119">
	<style name="Sans_Normal" isDefault="true" fontName="DejaVu Sans" fontSize="8" isBold="false" isItalic="false" isUnderline="false" isStrikeThrough="false"/>
	<!-- 1. 定义title -->
	<title>
		<band height="782">
		    <!-- 2. 定义静态文本内容 --> 
			<staticText>
				<reportElement style="Sans_Normal" mode="Opaque" x="0" y="0" width="555" height="90" backcolor="#DDDDDD" uuid="7e8a0a22-e6be-454f-b8ef-c3cd4e6490eb"/>
				<textElement textAlignment="Center" verticalAlignment="Middle"/>
				<text><![CDATA[The rectangles below have random fore and back colors and even this text element has runtime supplied font settings.
Changing element colors or other settings do not require report design recompilation.]]></text>
			</staticText>
			<!-- 3. 定义了一个长方形,并定义了key为:first.rectangle -->
			<rectangle>
				<reportElement key="first.rectangle" x="0" y="100" width="555" height="90" uuid="78672687-b9a9-4c6a-9d63-39d6325ac535"/>
				<graphicElement>
					<pen lineWidth="4.0"/>
				</graphicElement>
			</rectangle>
			
			<!-- 4. 定义了一个长方形,并定义了key为:second.rectangle -->
			<rectangle>
				<reportElement key="second.rectangle" x="0" y="200" width="555" height="90" uuid="e43916e7-f5a2-49e3-ac43-469458943653"/>
				<graphicElement>
					<pen lineWidth="4.0"/>
				</graphicElement>
			</rectangle>
			
			<!-- 5. 定义了一个长方形,并定义了key为:third.rectangle -->
			<rectangle>
				<reportElement key="third.rectangle" x="0" y="300" width="555" height="90" uuid="633026b6-ca4f-4895-baf0-e70c5f169168"/>
				<graphicElement>
					<pen lineWidth="4.0"/>
				</graphicElement>
			</rectangle>
		</band>
	</title>
</jasperReport>
```
### (4). AlterDesignApp
```
import java.awt.Color;
import java.io.File;

import net.sf.jasperreports.engine.JRDataSource;
import net.sf.jasperreports.engine.JRException;
import net.sf.jasperreports.engine.JRRectangle;
import net.sf.jasperreports.engine.JRStyle;
import net.sf.jasperreports.engine.JasperExportManager;
import net.sf.jasperreports.engine.JasperFillManager;
import net.sf.jasperreports.engine.JasperPrint;
import net.sf.jasperreports.engine.JasperPrintManager;
import net.sf.jasperreports.engine.JasperReport;
import net.sf.jasperreports.engine.util.AbstractSampleApp;
import net.sf.jasperreports.engine.util.JRLoader;
import net.sf.jasperreports.engine.util.JRSaver;


public class AlterDesignApp extends AbstractSampleApp
{
	/**
	 * java AlterDesignApp test
	 */
	public static void main(String[] args)
	{
		main(new AlterDesignApp(), args);
	}

	
	@Override
	public void test() throws JRException
	{
		// 1. 填充数据
		fill();
		// 2. 导出PDF
		pdf();
	}
	
	public void fill() throws JRException
	{
		long start = System.currentTimeMillis();
		File sourceFile = new File("reports/AlterDesignReport.jasper");
		System.err.println(" : " + sourceFile.getAbsolutePath());
		// 1. 加载 *.jasper
		JasperReport jasperReport = (JasperReport)JRLoader.loadObject(sourceFile);
		
		// 2. 通过唯一的key(first.rectangle),获得xml对应的Java模型对象:JRRectangle,并设置:前景颜色和背景颜色
		JRRectangle rectangle = (JRRectangle)jasperReport.getTitle().getElementByKey("first.rectangle");
		rectangle.setForecolor(new Color((int)(16000000 * Math.random())));
		rectangle.setBackcolor(new Color((int)(16000000 * Math.random())));

		// 3. 通过唯一的key(second.rectangle),获得xml对应的Java模型对象:JRRectangle,并设置:前景颜色和背景颜色
		rectangle = (JRRectangle)jasperReport.getTitle().getElementByKey("second.rectangle");
		rectangle.setForecolor(new Color((int)(16000000 * Math.random())));
		rectangle.setBackcolor(new Color((int)(16000000 * Math.random())));

		// 4. 通过唯一的key(third.rectangle),获得xml对应的Java模型对象:JRRectangle,并设置:前景颜色和背景颜色
		rectangle = (JRRectangle)jasperReport.getTitle().getElementByKey("third.rectangle");
		rectangle.setForecolor(new Color((int)(16000000 * Math.random())));
		rectangle.setBackcolor(new Color((int)(16000000 * Math.random())));

		// 5. 设置第一页的文档字体大小和文本斜体.
		JRStyle style = jasperReport.getStyles()[0];
		style.setFontSize(16f);
		style.setItalic(Boolean.TRUE);

		// 6. 填充数据 
		//   *.jasper --> *.jrprint
		JasperPrint jasperPrint = JasperFillManager.fillReport(jasperReport, null, (JRDataSource)null);
		File destFile = new File(sourceFile.getParent(), jasperReport.getName() + ".jrprint");
		JRSaver.saveObject(jasperPrint, destFile);
		System.err.println("Filling time : " + (System.currentTimeMillis() - start));
	}

	public void pdf() throws JRException
	{
		long start = System.currentTimeMillis();
		// 7. 读取*jrprint,导出成:pdf
		// *.jrprint --> *.pdf
		JasperExportManager.exportReportToPdfFile("reports/AlterDesignReport.jrprint");
		System.err.println("PDF creation time : " + (System.currentTimeMillis() - start));
	}
}
```
### (5). JasperSoft Studio CE查看效果

!["JasperSoft Studio CE查看alterdesign"](/assets/jasper-report/imgs/jasper-report-alterdesign-demo.jpg)

!["JasperSoft生成PDF效果"](/assets/jasper-report/imgs/jasper-report-alterdesign-pdf.jpg)

### (6). 总结
> 在这一小节,拿了一个简单的案例来进行说明,下一小节开始,将重点放在:JasperSoft Studio CE,以及JasperSoft的业务模型(xml即业务模型)上.  