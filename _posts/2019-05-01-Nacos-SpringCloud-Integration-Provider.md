---
layout: post
title: 'Nacos与Spring Cloud集成-提供者(四)'
date: 2019-05-01
author: 李新
tags: Nacos SpringCloud
---

### (1). spring-cloud-nacos-provider项目结构

```
spring-cloud-nacos-provider
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── samples
│   │   │               ├── ProviderApplication.java
│   │   │               └── controller
│   │   │                   └── HelloController.java
│   │   └── resources
│   │       ├── bootstrap.properties
│   │       └── logback-spring.xml
│   └── test
│       ├── java
│       │   └── help
│       │       └── lixin
│       │           └── samples
│       └── resources
└── target
```
### (2). pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>help.lixin</groupId>
	<artifactId>spring-colue-nacos-provider</artifactId>
	<packaging>jar</packaging>
	<version>1.1.0</version>
	<name>spring-cloud-nacos-provider ${project.version}</name>

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
			<dependency>
				<groupId>com.alibaba.cloud</groupId>
				<artifactId>spring-cloud-alibaba-dependencies</artifactId>
				<version>2.1.2.RELEASE</version>
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
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
		    <groupId>com.alibaba.cloud</groupId>
		    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
		</dependency>
		<dependency>
		    <groupId>com.alibaba.cloud</groupId>
		    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<mainClass>help.lixin.samples.ProviderApplication</mainClass>
				</configuration>
				<executions>
					<execution>
						<goals>
							<goal>repackage</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>
```
### (3). bootstrap.properties
```
server.port=8080

# 注意:配置文件加载的优先级为:   
# spring.application.name > spring.cloud.nacos.config.ext-config >  spring.cloud.nacos.config.shared-configs

# 指定DataId,nacos默认的dataId是: 
# spring.application.name + spring.cloud.nacos.config.file-extension
# 该例中是:test-provider.properties
spring.application.name=test-provider
# nacos配置地址
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
# 配置内容的数据格式
spring.cloud.nacos.config.file-extension=properties
# 指定namespace
spring.cloud.nacos.config.namespace=45a6112b-a866-4e92-a8d6-9e440bcdebbe
# 指定group(建议group直接指向项目组)
spring.cloud.nacos.config.group=erp:test-provider

# 分析: com.alibaba.cloud.nacos.NacosConfigProperties$Config 
# 只有三个属性:dataId/group/refresh,所以,扩展data-id,不支持:namespace
# 扩展data-id
spring.cloud.nacos.config.ext-config[0].dataId=datasource.properties
spring.cloud.nacos.config.ext-config[0].group=common
spring.cloud.nacos.config.ext-config[0].refresh=true

# 也可以这样配置
#spring.cloud.nacos.config.shared-configs[0].dataId=datasource.properties
#spring.cloud.nacos.config.shared-configs[0].group=common
#spring.cloud.nacos.config.shared-configs[0].refresh=true

# 不建议这样配置,因为它不支持:group,也就意味着,这样配置group=DEFAULT_GROUP
# spring.cloud.nacos.config.shared-dataids=datasource.properties,datasource-2.properties
# spring.cloud.nacos.config.refreshable-dataids=dataids=datasource.properties

# nacos注册中心地址
spring.cloud.nacos.discovery.server-addr=${spring.cloud.nacos.config.server-addr:127.0.0.1:8848}
# 指定namespace
spring.cloud.nacos.discovery.namespace=45a6112b-a866-4e92-a8d6-9e440bcdebbe
# nacos注册中心group(不建议配置,或者:统一好一个group)
#spring.cloud.nacos.discovery.group=erp:test-provider
```
### (4). HelloController
```
package help.lixin.samples.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.core.env.Environment;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RefreshScope
public class HelloController {
	private Logger logger = LoggerFactory.getLogger(HelloController.class);

	@Value("${useLocalCache:false}")
	private boolean useLocalCache;

	@Autowired
	private Environment env;

	@GetMapping("/hello")
	public String hello() {
		String jdbcurl = env.getProperty("spring.datasource.url", "");
		logger.debug("useLocalCache", useLocalCache);
		return "Hello World!!! " + useLocalCache + "  jdbcUrl:" + jdbcurl;
	}
}
```
### (5). ProviderApplication
```
package help.lixin.samples;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@EnableDiscoveryClient
@SpringBootApplication
public class ProviderApplication {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(ProviderApplication.class, args);
	}
}
```
### (6). 项目源码下载
["spring-cloud-nacos-provider.zip"](/assets/nacos/spring-cloud-nacos-provider.zip)

### (7). 启动服务并查看nacos
!["Nacos 服务提供者列表"](/assets/nacos/imgs/nacos-provider-register.png)