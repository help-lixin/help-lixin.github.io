---
layout: post
title: 'Sharding-JDBC 水平分库分表(三)'
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
│   │   │                   └── OrderMapper.java
│   │   └── resources
│   │       └── application.properties
│   └── test
│       └── java
│           └── help
│               └── lixin
│                   └── shardingjdbc
│                       └── OrderTest.java
└── target
```

### (2). 表结构如下
```
CREATE DATABASE `order_db` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';

USE `order_db`;

DROP TABLE IF EXISTS `t_order_1`;

CREATE TABLE `t_order_1` (
`order_id` bigint(20) NOT NULL COMMENT '订单id',
`price` decimal(10, 2) NOT NULL COMMENT '订单价格',
`user_id` bigint(20) NOT NULL COMMENT '下单用户id',
`status` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '订单状态', 
PRIMARY KEY (`order_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

DROP TABLE IF EXISTS `t_order_2`;

CREATE TABLE `t_order_2` (
`order_id` bigint(20) NOT NULL COMMENT '订单id',
`price` decimal(10, 2) NOT NULL COMMENT '订单价格',
`user_id` bigint(20) NOT NULL COMMENT '下单用户id',
`status` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '订单状态', 
PRIMARY KEY (`order_id`) USING BTREE
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
### (4). OrderMapper
```
package help.lixin.shardingjdbc.mapper;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;

import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

@Mapper
public interface OrderMapper {
	 /**
     * 插入订单
     * @param price
     * @param userId
     * @param status
     * @return
     */
    @Insert("insert into t_order(user_id,price,status)values(#{userId},#{price},#{status})")
    int insertOrder(@Param("userId") Long userId, @Param("price") BigDecimal price, @Param("status") String status);

    /**
     * 根据id列表查询订单
     * @param orderIds
     * @return
     */
    @Select("<script>" +
            "select" +
            " * " +
            " from t_order t " +
            " where t.order_id in " +
            " <foreach collection='orderIds' open='(' separator=',' close=')' item='id'>" +
            " #{id} " +
            " </foreach>" +
            "</script>")
    List<Map> selectOrderbyIds(@Param("orderIds") List<Long> orderIds);
}
```
### (5). application.properties
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
spring.shardingsphere.datasource.names=d1,d2

# 水平分库案例
# 创建两个MySQL实例,并创建两个库(order_db)
# 在d1和d2表结构都一样.

# 数据源名称:d1对应的详细信息
# 3306
spring.shardingsphere.datasource.d1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d1.url=jdbc:mysql://localhost:3306/order_db?useUnicode=true
spring.shardingsphere.datasource.d1.username=root
spring.shardingsphere.datasource.d1.password=123456

# 数据源名称:d2对应的详细信息
# 3307
spring.shardingsphere.datasource.d2.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d2.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d2.url=jdbc:mysql://localhost:3307/order_db?useUnicode=true
spring.shardingsphere.datasource.d2.username=root
spring.shardingsphere.datasource.d2.password=123456

############################################逻辑表配置项开始####################################################
# org.apache.shardingsphere.shardingjdbc.spring.boot.sharding.SpringBootShardingRuleConfigurationProperties
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingRuleConfiguration

# 指定逻辑表:t_order的数据节点:[d1/d2].t_order_1,d1.t_order_2
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=d$->{1..2}.t_order_$->{1..2}

# 分库实际要做的是找到对应上面的数据源名称
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.algorithm-expression=d$->{user_id % 2 + 1}


# 指定逻辑表t_order表的主键(order_id)生成策略为:雪花算法.
# org.apache.shardingsphere.core.yaml.config.sharding.YamlKeyGeneratorConfiguration
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE

# 分表实际要做的找到真实的表名称(t_order_1/t_order2)
# 指定t_order表的分片策略，分片策略包括分片键和分片算法
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingStrategyConfiguration
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order_$->{order_id % 2 + 1}
############################################逻辑表配置项结束####################################################

# 打开sharding-jdbc的日志.
spring.shardingsphere.props.sql.show=true
```
### (6). App
```
package help.lixin.shardingjdbc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class App {
	public static void main(String[] args) {
		SpringApplication.run(App.class, args);
	}
}
```
### (7). OrderTest
```
package help.lixin.shardingjdbc;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import help.lixin.shardingjdbc.mapper.OrderMapper;
import junit.framework.Assert;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = App.class)
public class OrderTest {

	@Autowired
	private OrderMapper orderMapper;

	@Test
	public void batchSave() {
		// 测试d2.t_order插入数据
		for (long i = 1; i < 20; i++) {
			// user_id = 1
			orderMapper.insertOrder(1L, new BigDecimal(20.5 + i), "SUCCESS");
		}
		
		// 测试d1.t_order插入数据
		for (long i = 1; i < 20; i++) {
			// user_id = 2
			orderMapper.insertOrder(2L, new BigDecimal(20.5 + i), "SUCCESS");
		}
	}
	

	// 由于是分库和分表的策略,而分库策略order_id不在SQL语句中,所以,会所有的库和表都进行Query然后合并.	
	// 笛卡尔查询
	@SuppressWarnings({ "deprecation", "rawtypes" })
	@Test
	public void query() {
		List<Long> ids = new ArrayList<Long>();
		ids.add(565586106691616768L);
		ids.add(565586106611924993L);
		ids.add(565586058801053696L);
		ids.add(565586091139137537L);
		List<Map> results = orderMapper.selectOrderbyIds(ids);
		Assert.assertNotNull(results);
	}
}
```
### (8). 查看表信息
```
// 3306机器
mysql> select * from t_order_1;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 565586106691616768 | 22.50 |       2 | SUCCESS |
| 565586106901331968 | 24.50 |       2 | SUCCESS |
| 565586107043938304 | 26.50 |       2 | SUCCESS |
| 565586107236876288 | 28.50 |       2 | SUCCESS |
| 565586107421425664 | 30.50 |       2 | SUCCESS |
| 565586107568226304 | 32.50 |       2 | SUCCESS |
| 565586107706638336 | 34.50 |       2 | SUCCESS |
| 565586107836661760 | 36.50 |       2 | SUCCESS |
| 565586107987656704 | 38.50 |       2 | SUCCESS |
+--------------------+-------+---------+---------+
9 rows in set (0.00 sec)

mysql> select * from t_order_2;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 565586106611924993 | 21.50 |       2 | SUCCESS |
| 565586106804862977 | 23.50 |       2 | SUCCESS |
| 565586106976829441 | 25.50 |       2 | SUCCESS |
| 565586107152990209 | 27.50 |       2 | SUCCESS |
| 565586107329150977 | 29.50 |       2 | SUCCESS |
| 565586107509506049 | 31.50 |       2 | SUCCESS |
| 565586107635335169 | 33.50 |       2 | SUCCESS |
| 565586107773747201 | 35.50 |       2 | SUCCESS |
| 565586107907964929 | 37.50 |       2 | SUCCESS |
| 565586108063154177 | 39.50 |       2 | SUCCESS |
+--------------------+-------+---------+---------+
10 rows in set (0.00 sec)

// 3307机器
mysql> select * from t_order_1;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 565586058801053696 | 21.50 |       1 | SUCCESS |
| 565586091193663488 | 23.50 |       1 | SUCCESS |
| 565586091365629952 | 25.50 |       1 | SUCCESS |
| 565586091508236288 | 27.50 |       1 | SUCCESS |
| 565586091604705280 | 29.50 |       1 | SUCCESS |
| 565586091692785664 | 31.50 |       1 | SUCCESS |
| 565586091785060352 | 33.50 |       1 | SUCCESS |
| 565586091868946432 | 35.50 |       1 | SUCCESS |
| 565586091936055296 | 37.50 |       1 | SUCCESS |
| 565586092007358464 | 39.50 |       1 | SUCCESS |
+--------------------+-------+---------+---------+
10 rows in set (0.00 sec)

mysql> select * from t_order_2;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 565586091139137537 | 22.50 |       1 | SUCCESS |
| 565586091285938177 | 24.50 |       1 | SUCCESS |
| 565586091432738817 | 26.50 |       1 | SUCCESS |
| 565586091566956545 | 28.50 |       1 | SUCCESS |
| 565586091650842625 | 30.50 |       1 | SUCCESS |
| 565586091726340097 | 32.50 |       1 | SUCCESS |
| 565586091835392001 | 34.50 |       1 | SUCCESS |
| 565586091902500865 | 36.50 |       1 | SUCCESS |
| 565586091969609729 | 38.50 |       1 | SUCCESS |
+--------------------+-------+---------+---------+
9 rows in set (0.01 sec)
```
### (9). 总结
> 在编写algorithm-expression表达式时,要特别小心一点,我在这上面踩坑很久.   