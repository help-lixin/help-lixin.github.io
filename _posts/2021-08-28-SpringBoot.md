---
layout: post
title: '独立租户部署Sass系统,成本节约解决方案(二)' 
date: 2021-08-21
author: 李新
tags:  解决方案
---

### (1). 背景
随着微服务的流行,业务模块越来越多后,对机器的要求也越来越高,一个小项目(20个微服务),按最小内存计算(512M),部署起来,都要10G左右.
在前面的篇幅,有类似的解决方案,但是,要把所有的依赖放在指定目录下,并配置xml.这一篇将会更加的优雅(仅提供简单思想).  

### (2). 添加依赖
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>boot-service</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>boot-service ${project.version}</name>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Greenwich.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>2.1.0.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-netflix</artifactId>
				<version>2.1.1.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<!-- 重点在这里 --> 
		<dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-loader</artifactId>
		</dependency>
	</dependencies>
</project>
```
### (3). 
```
// 只能写这个包名称,是因为:JarLauncher类的很多方法只能是相同包名称才可见.  
package org.springframework.boot.loader;

import java.io.File;

import org.springframework.boot.loader.archive.Archive;
import org.springframework.boot.loader.archive.JarFileArchive;

public class Application {
	public static void main(String[] args) {
		try {
			File root = new File("/Users/lixin/Workspace/boot-service/plugins/spring-web-demo/spring-web-demo-1.1.0.jar");
			Archive archive = new JarFileArchive(root);
			JarLauncher jarLauncher = new JarLauncher(archive);
			jarLauncher.launch(args);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```
### (4). 原理
在项目初期,可以,把多个进程,合在一个进程里(共享弹性内存),随着用户基线的增加,再弹性的拿出来,独立部署.   
