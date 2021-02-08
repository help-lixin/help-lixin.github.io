---
layout: post
title: 'Sharding-JDBC 水平分表之Java配置(二)'
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
│   │   │               ├── config
│   │   │               │   └── ShardingJdbcConfig.java
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
server.port=56081
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

# 用java配置时,一定要配置这个
# 禁止Spring Boot自动加载sharding-jdbc
#spring.shardingsphere.enabled=false
```

### (6). ShardingJdbcConfig
```
package help.lixin.shardingjdbc.config;

import java.sql.SQLException;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

import javax.sql.DataSource;

import org.apache.shardingsphere.api.config.sharding.KeyGeneratorConfiguration;
import org.apache.shardingsphere.api.config.sharding.ShardingRuleConfiguration;
import org.apache.shardingsphere.api.config.sharding.TableRuleConfiguration;
import org.apache.shardingsphere.api.config.sharding.strategy.InlineShardingStrategyConfiguration;
import org.apache.shardingsphere.shardingjdbc.api.ShardingDataSourceFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.alibaba.druid.pool.DruidDataSource;


/**
 * 当Sharding-jdbc的JAR包加载到Spring环境时,会被自动加载.而报错(java.lang.IllegalArgumentException: Data sources cannot be empty.) <br/>
 * 有两种配置方式:<br/>
 * 方式一:  spring.shardingsphere.enabled=false   <br/>  
 * 方式二:   @SpringBootApplication(exclude = org.apache.shardingsphere.shardingjdbc.spring.boot.SpringBootConfiguration.class)  <br/>
 * @author lixin
 */
@Configuration
public class ShardingJdbcConfig {
	// 定义数据源
	private Map<String, DataSource> createDataSourceMap() {
		DruidDataSource dataSource1 = new DruidDataSource();
		dataSource1.setDriverClassName("com.mysql.jdbc.Driver");
		dataSource1.setUrl("jdbc:mysql://localhost:3306/order_db?useUnicode=true");
		dataSource1.setUsername("root");
		dataSource1.setPassword("123456");
		Map<String, DataSource> result = new HashMap<>();
		result.put("d1", dataSource1);
		return result;
	}

	// 定义主键生成策略
	private static KeyGeneratorConfiguration getKeyGeneratorConfiguration() {
		KeyGeneratorConfiguration result = new KeyGeneratorConfiguration("SNOWFLAKE", "order_id");
		return result;
	}

	// 定义t_order表的分片策略
	private TableRuleConfiguration getOrderTableRuleConfiguration() {
		TableRuleConfiguration result = new TableRuleConfiguration("t_order", "d1.t_order_$->{1..2}");
		result.setTableShardingStrategyConfig(
				new InlineShardingStrategyConfiguration("order_id", "t_order_$->{order_id % 2 + 1}"));
		result.setKeyGeneratorConfig(getKeyGeneratorConfiguration());
		return result;
	}

	// 定义sharding-Jdbc数据源
	@Bean
	public DataSource getShardingDataSource() throws SQLException {
		ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
		shardingRuleConfig.getTableRuleConfigs().add(getOrderTableRuleConfiguration());
		Properties properties = new Properties();
		properties.put("sql.show", "true");
		return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), shardingRuleConfig, properties);
	}
}
```
### (7). App
```
package help.lixin.shardingjdbc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// 禁止自动加载sharding-jdbc
@SpringBootApplication(exclude = org.apache.shardingsphere.shardingjdbc.spring.boot.SpringBootConfiguration.class)
public class App {
	public static void main(String[] args) {
		SpringApplication.run(App.class, args);
	}
}
```
### (8). OrderTest
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
		for (long i = 1; i < 11; i++) {
			orderMapper.insertOrder(i, new BigDecimal(20.5 + i), "SUCCESS");
		}
	}
	
	
	@SuppressWarnings({ "deprecation", "rawtypes" })
	@Test
	public void query() {
		// 尝试跨多个表的查询
		List<Long> ids = new ArrayList<Long>();
		ids.add(565225137922637824L);
		ids.add(565225139432587265L);
		ids.add(565225139524861952L);
		ids.add(565225139612942337L);
		
		// select *  from t_order t  where t.order_id in   (   ?   ,  ?   ,  ?   ,  ?   )
		// select *  from t_order_1 t  where t.order_id in   (   ?   ,  ?   ,  ?   ,  ?   ) ::: [565225137922637824, 565225139432587265, 565225139524861952, 565225139612942337]
		// select *  from t_order_2 t  where t.order_id in   (   ?   ,  ?   ,  ?   ,  ?   ) ::: [565225137922637824, 565225139432587265, 565225139524861952, 565225139612942337]
		List<Map> results = orderMapper.selectOrderbyIds(ids);
		Assert.assertNotNull(results);
	}
}
```
### (9). 查看表信息
```
mysql> SELECT * FROM  t_order_1;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 565225137922637824 | 21.50 |       1 | SUCCESS |
| 565225139524861952 | 23.50 |       3 | SUCCESS |
| 565225139701022720 | 25.50 |       5 | SUCCESS |
| 565225139881377792 | 27.50 |       7 | SUCCESS |
| 565225140095287296 | 29.50 |       9 | SUCCESS |
+--------------------+-------+---------+---------+
5 rows in set (0.00 sec)


mysql> SELECT * FROM  t_order_2;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 565225139432587265 | 22.50 |       2 | SUCCESS |
| 565225139612942337 | 24.50 |       4 | SUCCESS |
| 565225139801686017 | 26.50 |       6 | SUCCESS |
| 565225139982041089 | 28.50 |       8 | SUCCESS |
| 565225140191756289 | 30.50 |      10 | SUCCESS |
+--------------------+-------+---------+---------+
5 rows in set (0.00 sec)
```
### (10). 总结
> 如果用java配置时,要稍微注意一点,要禁止Spring Boot自动加载配置.