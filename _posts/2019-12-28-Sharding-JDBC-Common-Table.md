---
layout: post
title: 'Sharding-JDBC 公共表(五)'
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
│   │   │                   └── DictMapper.java
│   │   └── resources
│   │       └── application.properties
│   └── test
│       └── java
│           └── help
│               └── lixin
│                   └── shardingjdbc
│                       └── DictTest.java
└── target
```

### (2). 表结构如下
> 在order_db_1/order_db_2/user_db库都要创建:t_dict

```
CREATE TABLE `t_dict` (
 `dict_id` bigint(20) NOT NULL COMMENT '字典id',
 `type` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '字典类型', `code` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '字典编码', `value` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '字典值', PRIMARY KEY (`dict_id`) USING BTREE
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
logging.level.com.itheima.dbsharding=debug
logging.level.druid.sql=debug

#配置数据源的名称(多个之间用逗号分隔).
spring.shardingsphere.datasource.names=d1,o1,o2

# 数据源名称:d1对应的详细信息
spring.shardingsphere.datasource.d1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d1.url=jdbc:mysql://localhost:3306/user_db?useUnicode=true
spring.shardingsphere.datasource.d1.username=root
spring.shardingsphere.datasource.d1.password=123456


spring.shardingsphere.datasource.o1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.o1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.o1.url=jdbc:mysql://localhost:3306/order_db_1?useUnicode=true
spring.shardingsphere.datasource.o1.username=root
spring.shardingsphere.datasource.o1.password=123456

# 数据源名称:d2对应的详细信息
spring.shardingsphere.datasource.o2.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.o2.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.o2.url=jdbc:mysql://localhost:3306/order_db_2?useUnicode=true
spring.shardingsphere.datasource.o2.username=root
spring.shardingsphere.datasource.o2.password=123456


############################################逻辑表配置项开始####################################################
# 重点:
# 指定t_dict为公共表
spring.shardingsphere.sharding.broadcast‐tables=t_dict


# t_user指定数据节点,以及分表信息
spring.shardingsphere.sharding.tables.t_user.actual-data-nodes=d$->{1}.t_user
spring.shardingsphere.sharding.tables.t_user.table-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_user.table-strategy.inline.algorithm-expression=t_user

# t_order指定娄数据节点,以及分库/主键生成策略/分表信息等
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=o$->{1..2}.t_order_$->{1..2}
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.algorithm-expression=o$->{user_id % 2 + 1}

spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE

spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order_$->{order_id % 2 + 1}
############################################逻辑表配置项结束####################################################

# 打开sharding-jdbc的日志.
spring.shardingsphere.props.sql.show=true
```

### (5). DictMapper
```
package help.lixin.shardingjdbc.mapper;

import org.apache.ibatis.annotations.Delete;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

@Mapper
public interface DictMapper {

	/**
	 * 新增字典
	 * 
	 * @param type 字典类型 * @param code 字典编码 * @param value 字典值 * @return
	 */
	@Insert("insert into t_dict(dict_id,type,code,value) values(#{dictId},#{type},#{code},#{value})")
	int insertDict(@Param("dictId") Long dictId, @Param("type") String type, @Param("code") String code,
			@Param("value") String value);

	/**
	 * 删除字典
	 * 
	 * @param dictId 字典id * @return
	 */
	@Delete("delete from t_dict where dict_id = #{dictId}")
	int deleteDict(@Param("dictId") Long dictId);
}
```
### (6). DictTest
```
package help.lixin.shardingjdbc;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import help.lixin.shardingjdbc.mapper.DictMapper;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = App.class)
public class DictTest {

	@Autowired
	private DictMapper dictMapper;

	@Test
	public void testInsertDict() {
		dictMapper.insertDict(1L, "user_type", "0", "管理员");
		dictMapper.insertDict(2L, "user_type", "1", "操作员");
	}

	@Test
	public void testDeleteDict() {
		dictMapper.deleteDict(1L);
		dictMapper.deleteDict(2L);
	}
}
```
### (7). 查看结果
```
# 查看order_db_1库
mysql> SELECT * FROM order_db_1.t_dict;
+---------+-----------+------+-----------+
| dict_id | type      | code | value     |
+---------+-----------+------+-----------+
|       1 | user_type | 0    | 管理员    |
|       2 | user_type | 1    | 操作员    |
+---------+-----------+------+-----------+
2 rows in set (0.00 sec)

# 查看order_db_2库
mysql> SELECT * FROM order_db_2.t_dict;
+---------+-----------+------+-----------+
| dict_id | type      | code | value     |
+---------+-----------+------+-----------+
|       1 | user_type | 0    | 管理员    |
|       2 | user_type | 1    | 操作员    |
+---------+-----------+------+-----------+
2 rows in set (0.00 sec)

# 查看user_db库
mysql> SELECT * FROM user_db.t_dict;
+---------+-----------+------+-----------+
| dict_id | type      | code | value     |
+---------+-----------+------+-----------+
|       1 | user_type | 0    | 管理员    |
|       2 | user_type | 1    | 操作员    |
+---------+-----------+------+-----------+
2 rows in set (0.00 sec)
```

### (8). 总结
> 在实际业务中,应该不会这样运用的.直接让开发在内存中对数据进行拼接,要不,性能会相当的差.     