---
layout: post
title: 'Sharding-JDBC 读写分离(六)'
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
CREATE DATABASE `order_db_1` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';

USE `order_db_1`;

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



CREATE DATABASE `order_db_2` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';

USE `order_db_2`;

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
spring.shardingsphere.datasource.names=d1,d2,s1,s2

############################################Master配置####################################################
# 水平分库案例,在d1和d2表结构都一样.
# 数据源名称:d1对应的详细信息
spring.shardingsphere.datasource.d1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d1.url=jdbc:mysql://localhost:3307/order_db_1?useUnicode=true
spring.shardingsphere.datasource.d1.username=root
spring.shardingsphere.datasource.d1.password=123456

# 数据源名称:d2对应的详细信息
spring.shardingsphere.datasource.d2.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d2.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d2.url=jdbc:mysql://localhost:3307/order_db_2?useUnicode=true
spring.shardingsphere.datasource.d2.username=root
spring.shardingsphere.datasource.d2.password=123456

############################################Slave配置####################################################

spring.shardingsphere.datasource.s1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.s1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.s1.url=jdbc:mysql://localhost:3308/order_db_1?useUnicode=true
spring.shardingsphere.datasource.s1.username=root
spring.shardingsphere.datasource.s1.password=123456

# 数据源名称:d2对应的详细信息
spring.shardingsphere.datasource.s2.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.s2.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.s2.url=jdbc:mysql://localhost:3308/order_db_2?useUnicode=true
spring.shardingsphere.datasource.s2.username=root
spring.shardingsphere.datasource.s2.password=123456

############################################逻辑表配置项开始####################################################
# org.apache.shardingsphere.shardingjdbc.spring.boot.sharding.SpringBootShardingRuleConfigurationProperties
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingRuleConfiguration

# 主从配置
# 声明配置ds1的master为:d1,slave为:s1
# org.apache.shardingsphere.core.yaml.config.masterslave.YamlMasterSlaveRuleConfiguration
spring.shardingsphere.sharding.master‐slave‐rules.ds1.master‐data‐source‐name=d1
spring.shardingsphere.sharding.master‐slave‐rules.ds1.slave‐data‐source‐names=s1

# 声明配置ds2的master为:d2,slave为:s2
spring.shardingsphere.sharding.master‐slave‐rules.ds2.master‐data‐source‐name=d2
spring.shardingsphere.sharding.master‐slave‐rules.ds2.slave‐data‐source‐names=s2


# 重点注意:
# actual-data-nodes 需要配置成master的名称(ds1/ds2)
# 指定逻辑表:t_order的数据节点:[ds1/ds2].t_order_1,d1.t_order_2
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds$->{1..2}.t_order_$->{1..2}


# 重点注意:
# database-strategy.inline.algorithm-expression 需要配置成master的名称(ds1/ds2)
# 分库实际要做的是找到对应上面的数据源名称
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.algorithm-expression=ds$->{user_id % 2 + 1}


# 指定逻辑表t_order表的主键(order_id)生成策略为:雪花算法.
# org.apache.shardingsphere.core.yaml.config.sharding.YamlKeyGeneratorConfiguration
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE

# 分表实际要做的找到真实的表名称(t_order_1/t_order_2)
# 指定t_order表的分片策略，分片策略包括分片键和分片算法
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingStrategyConfiguration
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order_$->{order_id % 2 + 1}
############################################逻辑表配置项结束####################################################

# 打开sharding-jdbc的日志.
spring.shardingsphere.props.sql.show=true
```

### (5). OrderMapper
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
### (6). OrderTest
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
	public void batchSave() throws Exception {
		// 测试ds2.t_order插入数据
		for (long i = 1; i < 20; i++) {
			orderMapper.insertOrder(1L, new BigDecimal(20.5 + i), "SUCCESS");
		}
		
		// 测试ds1.t_order插入数据
		for (long i = 1; i < 20; i++) {
			orderMapper.insertOrder(2L, new BigDecimal(20.5 + i), "SUCCESS");
		}
	}

	@SuppressWarnings({ "deprecation", "rawtypes" })
	@Test
	public void query() {
		List<Long> ids = new ArrayList<Long>();
		ids.add(566773221035081728L);
		ids.add(566773221223825408L);
		ids.add(566773221441929217L);
		ids.add(566773221924274176L);
		List<Map> results = orderMapper.selectOrderbyIds(ids);
		Assert.assertNotNull(results);
	}
}
```
### (7). 查看日志
```
# 查询的SQL语句所指向的是:s1和s2而不是:d1和d2,这就证明,查询语句,所使用的是slave数据源
Actual SQL: s1 ::: select *  from t_order_1 t  where t.order_id in   (   ?   ,  ?   ,  ?   ,  ?   ) ::: [566773221035081728, 566773221223825408, 566773221441929217, 566773221924274176]
Actual SQL: s1 ::: select *  from t_order_2 t  where t.order_id in   (   ?   ,  ?   ,  ?   ,  ?   ) ::: [566773221035081728, 566773221223825408, 566773221441929217, 566773221924274176]
Actual SQL: s2 ::: select *  from t_order_1 t  where t.order_id in   (   ?   ,  ?   ,  ?   ,  ?   ) ::: [566773221035081728, 566773221223825408, 566773221441929217, 566773221924274176]
Actual SQL: s2 ::: select *  from t_order_2 t  where t.order_id in   (   ?   ,  ?   ,  ?   ,  ?   ) ::: [566773221035081728, 566773221223825408, 566773221441929217, 566773221924274176]
```
### (8). 总结
> Sharding-JDBC开始主从配置后,需要注意一点:actual-data-nodes和database-strategy.inline.algorithm-expression要配置成定义的主从数据源.而不再是原来的数据源了.   