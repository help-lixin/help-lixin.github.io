---
layout: post
title: 'Sharding-JDBC 垂直分库(四)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC
---

### (1). 项目结构如下

```
sharding-jdbc-demo
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── shardingjdbc
│   │   │               ├── App.java
│   │   │               └── mapper
│   │   │                   └── UserMapper.java
│   │   └── resources
│   │       └── application.properties
│   └── test
│       └── java
│           └── help
│               └── lixin
│                   └── shardingjdbc
│                       └── UserTest.java
└── target
```

### (2). 表结构如下
```
CREATE DATABASE `user_db` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';

USE `user_db`;

DROP TABLE IF EXISTS `t_user`;

CREATE TABLE `t_user` (
 `user_id` bigint(20) NOT NULL COMMENT '用户id',
 `fullname` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '用户姓名', `user_type` char(1) DEFAULT NULL COMMENT '用户类型',
 PRIMARY KEY (`user_id`) USING BTREE
 ) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
```

### (3). pom.xml配置
> 注意:不要引入druid-spring-boot-starter,因为它在启动时,会检查(spring.datasource.xxx)配置,如果不存在,就会抛出异常.  

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin.shardingjdbc</groupId>
	<artifactId>sharding-jdbc-demo</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>sharding-jdbc-demo</name>
	<url>http://maven.apache.org</url>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<java.version>1.8</java.version>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>2.1.0.RELEASE</version>
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
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>2.0.0</version>
		</dependency>
		<dependency>
			<groupId>com.baomidou</groupId>
			<artifactId>mybatis-plus-boot-starter</artifactId>
			<version>3.1.0</version>
		</dependency>
		<dependency>
			<groupId>com.baomidou</groupId>
			<artifactId>mybatis-plus-generator</artifactId>
			<version>3.1.0</version>
		</dependency>
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis-typehandlers-jsr310</artifactId>
			<version>1.0.2</version>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid</artifactId>
			<version>1.1.22</version>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
		</dependency>
		<dependency>
			<groupId>org.apache.shardingsphere</groupId>
			<artifactId>sharding-jdbc-spring-boot-starter</artifactId>
			<version>4.1.1</version>
		</dependency>
	</dependencies>

	<build>
		<finalName>${project.name}</finalName>
		<resources>
			<resource>
				<directory>src/main/resources</directory>
				<filtering>true</filtering>
				<includes>
					<include>**/*</include>
				</includes>
			</resource>
			<resource>
				<directory>src/main/java</directory>
				<includes>
					<include>**/*.xml</include>
				</includes>
			</resource>
		</resources>

		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
			<plugin>
				<artifactId>maven-resources-plugin</artifactId>
				<configuration>
					<encoding>utf-8</encoding>
					<useDefaultDelimiters>true</useDefaultDelimiters>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>
```
### (4). application.properties
```
server.port=8080
spring.application.name=sharding-jdbc-demo

spring.http.encoding.enabled=true
spring.http.encoding.charset=UTF-8
spring.http.encoding.force=true
spring.main.allow-bean-definition-overriding=true

# mybatis配置
mybatis.configuration.map-underscore-to-camel-case=true

# 日志配置信息
logging.level.root=info
logging.level.org.springframework.web=info
logging.level.com.itheima.dbsharding =debug
logging.level.druid.sql=debug

#配置数据源的名称(多个之间用逗号分隔).
spring.shardingsphere.datasource.names=d1

# 数据源名称:d1对应的详细信息
spring.shardingsphere.datasource.d1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d1.url=jdbc:mysql://localhost:3306/user_db?useUnicode=true
spring.shardingsphere.datasource.d1.username=root
spring.shardingsphere.datasource.d1.password=123456

############################################逻辑表配置项开始####################################################
spring.shardingsphere.sharding.tables.t_user.actual-data-nodes=d$->{1}.t_user

spring.shardingsphere.sharding.tables.t_user.table-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_user.table-strategy.inline.algorithm-expression=t_user
############################################逻辑表配置项结束####################################################

# 打开sharding-jdbc的日志.
spring.shardingsphere.props.sql.show=true
```
### (5). UserMapper
```
package help.lixin.shardingjdbc.mapper;

import java.util.List;
import java.util.Map;

import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

@Mapper
public interface UserMapper {
	/**
	* 新增用户
	* @param userId 用户id
	* @param fullname 用户姓名 * @return
	*/
	@Insert("insert into t_user(user_id, fullname) value(#{userId},#{fullname})") 
	int insertUser(@Param("userId")Long userId,@Param("fullname")String fullname);
	/**
	* 根据id列表查询多个用户
	* @param userIds 用户id列表 * @return
	*/
	@Select({"<script>", " select",
	" * ",
	" from t_user t ",
	" where t.user_id in",
	"<foreach collection='userIds' item='id' open='(' separator=',' close=')'>", "#{id}",
	"</foreach>",
	"</script>"
	})
	List<Map> selectUserbyIds(@Param("userIds")List<Long> userIds);
}
```
### (6). UserTest
```
package help.lixin.shardingjdbc;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import help.lixin.shardingjdbc.mapper.UserMapper;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = App.class)
public class UserTest {

	@Resource
	private UserMapper userMapper;

	@Test
	public void testInsertUser() throws Exception {
		userMapper.insertUser(1L, "张三");
		userMapper.insertUser(2L, "李四");
		userMapper.insertUser(3L, "王五");
		userMapper.insertUser(4L, "赵六");
		userMapper.insertUser(5L, "田七");
	}

	@Test
	public void testSelectUserbyIds() {
		List<Long> userIds = new ArrayList<>();
		userIds.add(1L);
		userIds.add(2L);
		List<Map> users = userMapper.selectUserbyIds(userIds);
		System.out.println(users);
	}

}
```
### (7). 查看MySQL
```
mysql> SELECT * FROM t_user;
+---------+----------+-----------+
| user_id | fullname | user_type |
+---------+----------+-----------+
|       1 | 张三     | NULL      |
|       2 | 李四     | NULL      |
|       3 | 王五     | NULL      |
|       4 | 赵六     | NULL      |
|       5 | 田七     | NULL      |
+---------+----------+-----------+
5 rows in set (0.00 sec)
```
### (8). 总结
> 会了前面的水平分库分表,这种垂直分库,实则还是比较简单的.   