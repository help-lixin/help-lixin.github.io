---
layout: post
title: 'JoptSimple Hello World'
date: 2020-12-13
author: 李新
tags: JoptSimple
---

### (1). pom.xml
```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin</groupId>
	<artifactId>demo</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>demo</name>
	<url>http://maven.apache.org</url>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
		    <groupId>net.sf.jopt-simple</groupId>
		    <artifactId>jopt-simple</artifactId>
		    <version>5.0.2</version>
		</dependency>
	</dependencies>
</project>

```
### (2). JoptTest
```
package help.lixin;

import java.util.Arrays;

import joptsimple.OptionParser;
import joptsimple.OptionSet;

public class JoptTest {
	// --zookeeper localhost:2181
	// --zk localhost:2181
	public static void main(String[] args) throws Exception {
		OptionParser parser = new OptionParser();
		parser.acceptsAll(Arrays.asList("zk","zookeeper"), "it is required").withRequiredArg().describedAs("zookeeper connect address.")
				.ofType(String.class);
		OptionSet options = parser.parse(args);
		if (!options.has("zookeeper")) {
			throw new Exception("zookeeper is required");
		}
		System.out.println((String) options.valueOf("zookeeper"));
	}
}

```
### (3). 配置参数运行
!["jopt hello world"](/assets/jopt/imgs/jopt-simple-hello-world.png)

