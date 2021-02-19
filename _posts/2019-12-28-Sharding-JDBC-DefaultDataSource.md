---
layout: post
title: 'Sharding-JDBC 默认数据源(七)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC
---

### (1). 目标需求
> 有这样的需求:有些表,不想配置逻辑表,也不想参与分库(分表),只想固定在某个数据源怎么办?   
> 该需求要求的是:一部份表(t_order)分表分库,一部份表(t_user)不参与分表分库.    
### (2). 项目结构如下
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
│   │   │                   ├── OrderMapper.java
│   │   │                   └── UserMapper.java
│   │   └── resources
│   │       └── application.properties
│   └── test
│       └── java
│           └── help
│               └── lixin
│                   └── shardingjdbc
│                       └── AllTest.java
└── target
```

### (3). 表结构如下
```
# d1数据源库和表结构
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


# d2数据源库和表结构
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


# d3数据源库和表结构
CREATE DATABASE `user_db` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';

USE `user_db`;

DROP TABLE IF EXISTS `t_user`;

CREATE TABLE `t_user` (
 `user_id` bigint(20) NOT NULL COMMENT '用户id',
 `fullname` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '用户姓名', `user_type` char(1) DEFAULT NULL COMMENT '用户类型',
 PRIMARY KEY (`user_id`) USING BTREE
 ) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
```

### (4). pom.xml配置
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
spring.shardingsphere.datasource.names=d1,d2,d3

# 水平分库案例,在d1和d2表结构都一样.
# 数据源名称:d1对应的详细信息
spring.shardingsphere.datasource.d1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d1.url=jdbc:mysql://localhost:3306/order_db?useUnicode=true
spring.shardingsphere.datasource.d1.username=root
spring.shardingsphere.datasource.d1.password=123456

# 数据源名称:d2对应的详细信息
spring.shardingsphere.datasource.d2.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d2.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d2.url=jdbc:mysql://localhost:3307/order_db?useUnicode=true
spring.shardingsphere.datasource.d2.username=root
spring.shardingsphere.datasource.d2.password=123456

# 数据源名称:d3对应的详细信息
spring.shardingsphere.datasource.d3.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.d3.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.d3.url=jdbc:mysql://localhost:3308/user_db?useUnicode=true
spring.shardingsphere.datasource.d3.username=root
spring.shardingsphere.datasource.d3.password=123456

############################################逻辑表配置项开始####################################################
# org.apache.shardingsphere.shardingjdbc.spring.boot.sharding.SpringBootShardingRuleConfigurationProperties
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingRuleConfiguration

# 指定逻辑表:t_order的数据节点:[d1/d2].t_order_1,d1.t_order_2
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=d$->{1..2}.t_order_$->{1..2}

# 分库实际要做的是找到对应上面的数据源名称
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.algorithm-expression=d$->{user_id % 2 + 1}

# 设置默认的分库策略,访配置等同于上面两行配置
# spring.shardingsphere.sharding.default-table-strategy.inline.sharding-column=order_id
# spring.shardingsphere.sharding.default-table-strategy.inline.algorithm-expression=t_order_$->{order_id % 2 + 1}


# 注意一点,如果自己在SQL语句中,有指定主键,则配置的ID生成策略就会无效了.
# 指定逻辑表t_order表的主键(order_id)生成策略为:雪花算法.
# org.apache.shardingsphere.core.yaml.config.sharding.YamlKeyGeneratorConfiguration
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE

# 分表实际要做的找到真实的表名称(t_order_1/t_order_2)
# 指定t_order表的分片策略，分片策略包括分片键和分片算法
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingStrategyConfiguration
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order_$->{order_id % 2 + 1}


# 设置默认的分表策略,访配置等同于上面两行配置
# spring.shardingsphere.sharding.default-table-strategy.inline.sharding-column=order_id
# spring.shardingsphere.sharding.default-table-strategy.inline.algorithm-expression=t_order_$->{order_id % 2 + 1}


# 指定主键生成策略为:雪花算法.
# 注意:这里使用的是defaultKeyGenerator
# sharding-jdbc可以做到配置的继承法则
# 注意一点,如果自己在SQL语句中,有指定主键,则配置的ID生成策略就会无效了.
spring.shardingsphere.sharding.default-key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.default-key-generator.column=user_id

# 设置默认的数源名称
spring.shardingsphere.sharding.default-data-source-name=d3
############################################逻辑表配置项结束####################################################

# 打开sharding-jdbc的日志.
spring.shardingsphere.props.sql.show=true
```

### (6). OrderMapper
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
### (7). UserMapper
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
### (8). AllTest
```
package help.lixin.shardingjdbc;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import help.lixin.shardingjdbc.mapper.OrderMapper;
import help.lixin.shardingjdbc.mapper.UserMapper;
import junit.framework.Assert;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = App.class)
public class AllTest {

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
		ids.add(569598419530678272L);
		ids.add(569598420910604288L);
//		ids.add(566773221441929217L);
//		ids.add(566773221924274176L);
		List<Map> results = orderMapper.selectOrderbyIds(ids);
		Assert.assertNotNull(results);
	}
	
	
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

### (9). 测试结果检查
> t_order会按照配置,进行路由.   
> t_user因为,在逻辑表中不存在,所以,会路由到默认的数据源(defaultDataSourceName).    
> 我这里数据乱码,先不管理乱码问题,只要数据能正常路由即可.   

```
lixin-macbook:~ lixin$ mysql -u root -P 3308 -h 127.0.0.1 -p

mysql> use user_db;
Database changed

mysql> SELECT * FROM t_user;
+---------+----------+-----------+
| user_id | fullname | user_type |
+---------+----------+-----------+
|       1 | ??       | NULL      |
|       2 | ??       | NULL      |
|       3 | ??       | NULL      |
|       4 | ??       | NULL      |
|       5 | ??       | NULL      |
+---------+----------+-----------+
5 rows in set (0.00 sec)
```
### (10). 总结
> 通过测试,得出结论:没有匹配到任何逻辑表的情况下,会把SQL语句路由到默认的数据源上.   
> 如果项目场景像上面的需求一样,就可以使用Sharding-JDBC框架,如果,项目场景完全不需要分表分库,建议:不要使用Sharding-JDBC.     
> 毕竟,SQL语句要进行拦截,解析,处理,经历一堆无意义的逻辑没有必要.    
> 当然,你也有可能想说:我想统一整个数据源的配置,那么损失一点点性能,换来统一管理也是可行的.   