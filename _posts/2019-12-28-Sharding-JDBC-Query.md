---
layout: post
title: 'Sharding-JDBC 未指定分片查询剖析(八)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC
---

### (1). 前言
如果查询的SQL语句,并没有带上分片(sharding-column)的字段,那么结果是怎么样的呢?

### (2). Sharding JDBC分片配置如下
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
#spring.shardingsphere.sharding.default-data-source-name=d3
############################################逻辑表配置项结束####################################################

# 打开sharding-jdbc的日志.
spring.shardingsphere.props.sql.show=true
```
### (3). t_order分片图解
> 1. 根据user_id分库(选择数据源).  
> 2. 根据user_id水平分表.  

!["t_order分片图解"](/assets/sharding-jdbc/imgs/sharding-jdbc-order.png)

### (4). OrderMapper
```
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

    @Select("<script>" +
            "SELECT * FROM t_order " +
    		"LIMIT #{page} , #{pageSize}"
    		+ "</script>")
    List<Map> queryAll(@Param("page") long page,@Param("pageSize") long pageSize);

    @Select("<script>" +
            "SELECT * FROM t_order t" +
            " where t.user_id in " +
            " <foreach collection='userIds' open='(' separator=',' close=')' item='id'>" +
            " #{id} " +
            " </foreach>" + 
    		"LIMIT #{page} , #{pageSize}"
    		+ "</script>")
    List<Map> queryInUserId(@Param("userIds")  List<Long> userIds, @Param("page") long page,@Param("pageSize") long pageSize);
    
    
    @Select("<script>" +
            "SELECT COUNT(*) FROM t_order " 
    		+ "</script>")
    long count();
}
```
### (5). QueryNoRouteTest
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

@RunWith(SpringRunner.class)
@SpringBootTest(classes = App.class)
public class QueryNoRouteTest {
	
	@Autowired
	private OrderMapper orderMapper;

    // 1. 测试保存数据
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

   // 2. 测试没有路由情况下查询
	@Test
	public void queryAll() {
		List<Map> results = orderMapper.queryAll(0, 4);
		System.out.println(results);
		results = orderMapper.queryAll(4, 4);
		System.out.println(results);
		results = orderMapper.queryAll(8, 4);
		System.out.println(results);
		results = orderMapper.queryAll(12, 4);
		System.out.println(results);
		results = orderMapper.queryAll(16, 4);
		System.out.println(results);
		results = orderMapper.queryAll(20, 4);
		System.out.println(results);
		results = orderMapper.queryAll(24, 4);
		System.out.println(results);
		results = orderMapper.queryAll(28, 4);
		System.out.println(results);
		results = orderMapper.queryAll(32, 4);
		System.out.println(results);
	}
	
	
	// 3. 测试有分库的路由情况下
	@Test
	public void queryInUserId() {
		List<Long> userIds = new ArrayList<Long>();
		userIds.add(1L);
		userIds.add(2L);
		
		List<Map> results = orderMapper.queryInUserId(userIds, 0, 4);
		System.out.println(results);
		results = orderMapper.queryInUserId(userIds,4, 4);
		System.out.println(results);
		results = orderMapper.queryInUserId(userIds,8, 4);
		System.out.println(results);
		results = orderMapper.queryInUserId(userIds,12, 4);
		System.out.println(results);
		results = orderMapper.queryInUserId(userIds,16, 4);
		System.out.println(results);
		results = orderMapper.queryInUserId(userIds,20, 4);
		System.out.println(results);
		results = orderMapper.queryInUserId(userIds,24, 4);
		System.out.println(results);
		results = orderMapper.queryInUserId(userIds,28, 4);
		System.out.println(results);
		results = orderMapper.queryInUserId(userIds,32, 4);
		System.out.println(results);
	}

	// 4. 测试count
	@Test
	public void count() {
		long count = orderMapper.count();
		System.out.println(count);
	}
}
```
### (6). 测试批量保存(QueryNoRouteTest.batchSave)
```
# ****************************************3306****************************************
lixin-macbook:sharding-jdbc-demo lixin$ mysql -u root -P 3306 -p
mysql> use order_db;
Database changed
mysql> SELECT * FROM t_order_1;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 620244932191322112 | 22.50 |       2 | SUCCESS |
| 620244932388454400 | 24.50 |       2 | SUCCESS |
| 620244932535255040 | 26.50 |       2 | SUCCESS |
| 620244932744970240 | 28.50 |       2 | SUCCESS |
| 620244932891770880 | 30.50 |       2 | SUCCESS |
| 620244933034377216 | 32.50 |       2 | SUCCESS |
| 620244933168594944 | 34.50 |       2 | SUCCESS |
| 620244933298618368 | 36.50 |       2 | SUCCESS |
| 620244933478973440 | 38.50 |       2 | SUCCESS |
+--------------------+-------+---------+---------+
9 rows in set (0.00 sec)

mysql> SELECT * FROM t_order_2;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 620244931998384129 | 21.50 |       2 | SUCCESS |
| 620244932296179713 | 23.50 |       2 | SUCCESS |
| 620244932468146177 | 25.50 |       2 | SUCCESS |
| 620244932635918337 | 27.50 |       2 | SUCCESS |
| 620244932812079105 | 29.50 |       2 | SUCCESS |
| 620244932971462657 | 31.50 |       2 | SUCCESS |
| 620244933101486081 | 33.50 |       2 | SUCCESS |
| 620244933235703809 | 35.50 |       2 | SUCCESS |
| 620244933369921537 | 37.50 |       2 | SUCCESS |
| 620244933688688641 | 39.50 |       2 | SUCCESS |
+--------------------+-------+---------+---------+
10 rows in set (0.00 sec)


# ****************************************3307****************************************
lixin-macbook:~ lixin$ mysql -h 127.0.0.1 -u root -P3307 -p
mysql> use order_db;
Database changed
mysql> SELECT * FROM t_order_1;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 620244929406304256 | 21.50 |       1 | SUCCESS |
| 620244930639429632 | 23.50 |       1 | SUCCESS |
| 620244930790424576 | 25.50 |       1 | SUCCESS |
| 620244930924642304 | 27.50 |       1 | SUCCESS |
| 620244931071442944 | 29.50 |       1 | SUCCESS |
| 620244931230826496 | 31.50 |       1 | SUCCESS |
| 620244931360849920 | 33.50 |       1 | SUCCESS |
| 620244931537010688 | 35.50 |       1 | SUCCESS |
| 620244931654451200 | 37.50 |       1 | SUCCESS |
| 620244931901915136 | 39.50 |       1 | SUCCESS |
+--------------------+-------+---------+---------+
10 rows in set (0.00 sec)

mysql> SELECT * FROM t_order_2;
+--------------------+-------+---------+---------+
| order_id           | price | user_id | status  |
+--------------------+-------+---------+---------+
| 620244930555543553 | 22.50 |       1 | SUCCESS |
| 620244930719121409 | 24.50 |       1 | SUCCESS |
| 620244930849144833 | 26.50 |       1 | SUCCESS |
| 620244930995945473 | 28.50 |       1 | SUCCESS |
| 620244931134357505 | 30.50 |       1 | SUCCESS |
| 620244931302129665 | 32.50 |       1 | SUCCESS |
| 620244931486679041 | 34.50 |       1 | SUCCESS |
| 620244931583148033 | 36.50 |       1 | SUCCESS |
| 620244931788668929 | 38.50 |       1 | SUCCESS |
+--------------------+-------+---------+---------+
9 rows in set (0.00 sec)
```
### (7). 测试没有路由的情况(QueryNoRouteTest.queryAll)
从查询结果分析,当没有路由键的情况下,会线性的输出结果(但是SQL还是会发送到不同的分片).
输出结果依次是这样:  d1.t_order_1/d1._t_order_2/d2.t_order_1/d2.t_order_2
```
2021-07-09 14:06:49.582  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.582  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@3cb49121), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@3cb49121, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@227b9277, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@4c4215d7, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@b56d8a7, containsSubquery=false)
2021-07-09 14:06:49.582  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 4]
2021-07-09 14:06:49.582  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 4]
2021-07-09 14:06:49.583  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 4]
2021-07-09 14:06:49.583  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 4]
[{user_id=2, price=22.50, order_id=620244932191322112, status=SUCCESS}, {user_id=2, price=24.50, order_id=620244932388454400, status=SUCCESS}, {user_id=2, price=26.50, order_id=620244932535255040, status=SUCCESS}, {user_id=2, price=28.50, order_id=620244932744970240, status=SUCCESS}]
2021-07-09 14:06:49.722  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.723  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@36330be8), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@36330be8, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@38ba8b45, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@41f23499, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@31dbf5bb, containsSubquery=false)
2021-07-09 14:06:49.724  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 8]
2021-07-09 14:06:49.724  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 8]
2021-07-09 14:06:49.724  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 8]
2021-07-09 14:06:49.724  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 8]
[{user_id=2, price=30.50, order_id=620244932891770880, status=SUCCESS}, {user_id=2, price=32.50, order_id=620244933034377216, status=SUCCESS}, {user_id=2, price=34.50, order_id=620244933168594944, status=SUCCESS}, {user_id=2, price=36.50, order_id=620244933298618368, status=SUCCESS}]
2021-07-09 14:06:49.771  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.772  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@6338afe2), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@6338afe2, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@68360fb9, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@1c787389, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@67b3960b, containsSubquery=false)
2021-07-09 14:06:49.774  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 12]
2021-07-09 14:06:49.776  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 12]
2021-07-09 14:06:49.776  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 12]
2021-07-09 14:06:49.776  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 12]
[{user_id=2, price=38.50, order_id=620244933478973440, status=SUCCESS}, {user_id=2, price=21.50, order_id=620244931998384129, status=SUCCESS}, {user_id=2, price=23.50, order_id=620244932296179713, status=SUCCESS}, {user_id=2, price=25.50, order_id=620244932468146177, status=SUCCESS}]
2021-07-09 14:06:49.797  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.797  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@6ec98ccc), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@6ec98ccc, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@441aa7ae, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@1534bdc6, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@53079ae6, containsSubquery=false)
2021-07-09 14:06:49.797  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 16]
2021-07-09 14:06:49.798  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 16]
2021-07-09 14:06:49.798  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 16]
2021-07-09 14:06:49.798  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 16]
[{user_id=2, price=27.50, order_id=620244932635918337, status=SUCCESS}, {user_id=2, price=29.50, order_id=620244932812079105, status=SUCCESS}, {user_id=2, price=31.50, order_id=620244932971462657, status=SUCCESS}, {user_id=2, price=33.50, order_id=620244933101486081, status=SUCCESS}]
2021-07-09 14:06:49.813  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.813  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@5a06eeef), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@5a06eeef, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@1c0cf193, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@3dd66ff5, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@24258b54, containsSubquery=false)
2021-07-09 14:06:49.813  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 20]
2021-07-09 14:06:49.813  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 20]
2021-07-09 14:06:49.813  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 20]
2021-07-09 14:06:49.813  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 20]
[{user_id=2, price=35.50, order_id=620244933235703809, status=SUCCESS}, {user_id=2, price=37.50, order_id=620244933369921537, status=SUCCESS}, {user_id=2, price=39.50, order_id=620244933688688641, status=SUCCESS}, {user_id=1, price=21.50, order_id=620244929406304256, status=SUCCESS}]
2021-07-09 14:06:49.825  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.826  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@2211e731), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@2211e731, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@73e399cc, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@3dd591b9, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@538905d2, containsSubquery=false)
2021-07-09 14:06:49.826  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 24]
2021-07-09 14:06:49.826  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 24]
2021-07-09 14:06:49.826  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 24]
2021-07-09 14:06:49.826  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 24]
[{user_id=1, price=23.50, order_id=620244930639429632, status=SUCCESS}, {user_id=1, price=25.50, order_id=620244930790424576, status=SUCCESS}, {user_id=1, price=27.50, order_id=620244930924642304, status=SUCCESS}, {user_id=1, price=29.50, order_id=620244931071442944, status=SUCCESS}]
2021-07-09 14:06:49.838  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.839  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@5e62ca19), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@5e62ca19, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@188bf4d8, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@7dd7ec56, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@6528d339, containsSubquery=false)
2021-07-09 14:06:49.839  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 28]
2021-07-09 14:06:49.839  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 28]
2021-07-09 14:06:49.839  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 28]
2021-07-09 14:06:49.839  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 28]
[{user_id=1, price=31.50, order_id=620244931230826496, status=SUCCESS}, {user_id=1, price=33.50, order_id=620244931360849920, status=SUCCESS}, {user_id=1, price=35.50, order_id=620244931537010688, status=SUCCESS}, {user_id=1, price=37.50, order_id=620244931654451200, status=SUCCESS}]
2021-07-09 14:06:49.853  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.853  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@22f80e36), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@22f80e36, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@3c98981e, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@6dcee890, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@713e49c3, containsSubquery=false)
2021-07-09 14:06:49.853  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 32]
2021-07-09 14:06:49.853  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 32]
2021-07-09 14:06:49.853  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 32]
2021-07-09 14:06:49.853  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 32]
[{user_id=1, price=39.50, order_id=620244931901915136, status=SUCCESS}, {user_id=1, price=22.50, order_id=620244930555543553, status=SUCCESS}, {user_id=1, price=24.50, order_id=620244930719121409, status=SUCCESS}, {user_id=1, price=26.50, order_id=620244930849144833, status=SUCCESS}]
2021-07-09 14:06:49.867  INFO 3153 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order LIMIT ? , ?
2021-07-09 14:06:49.867  INFO 3153 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@24ac6fef, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@756200d1), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@756200d1, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@390a07a0, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@674e4c82, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@572b4072, containsSubquery=false)
2021-07-09 14:06:49.867  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 36]
2021-07-09 14:06:49.868  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 36]
2021-07-09 14:06:49.871  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 LIMIT ? , ? ::: [0, 36]
2021-07-09 14:06:49.871  INFO 3153 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 LIMIT ? , ? ::: [0, 36]
[{user_id=1, price=28.50, order_id=620244930995945473, status=SUCCESS}, {user_id=1, price=30.50, order_id=620244931134357505, status=SUCCESS}, {user_id=1, price=32.50, order_id=620244931302129665, status=SUCCESS}, {user_id=1, price=34.50, order_id=620244931486679041, status=SUCCESS}]
2021-07-09 14:06:49.906  INFO 3153 --- [       Thread-2] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closing ...
2021-07-09 14:06:49.926  INFO 3153 --- [       Thread-2] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
2021-07-09 14:06:49.942  INFO 3153 --- [       Thread-2] com.alibaba.druid.pool.DruidDataSource   : {dataSource-2} closing ...
2021-07-09 14:06:49.951  INFO 3153 --- [       Thread-2] com.alibaba.druid.pool.DruidDataSource   : {dataSource-2} closed
2021-07-09 14:06:49.954  INFO 3153 --- [       Thread-2] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'
```

### (8). 测试带有路由分库的键(QueryNoRouteTest.queryInUserId)
> 由于路由分库的键,在所有的库上都有,所以,会查询所有的表,但是,查询依然是线性的返回结果,不会是多张表混合的结果. 

```
2021-07-09 14:16:31.363  INFO 3400 --- [           main] h.lixin.shardingjdbc.QueryNoRouteTest    : Started QueryNoRouteTest in 7.101 seconds (JVM running for 9.43)
2021-07-09 14:16:32.952  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:32.952  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@50122012), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@50122012, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@569348e1, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@7db5b890, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@407b8435, containsSubquery=false)
2021-07-09 14:16:32.953  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 4]
2021-07-09 14:16:32.953  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 4]
2021-07-09 14:16:32.953  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 4]
2021-07-09 14:16:32.953  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 4]
[{user_id=2, price=22.50, order_id=620244932191322112, status=SUCCESS}, {user_id=2, price=24.50, order_id=620244932388454400, status=SUCCESS}, {user_id=2, price=26.50, order_id=620244932535255040, status=SUCCESS}, {user_id=2, price=28.50, order_id=620244932744970240, status=SUCCESS}]
2021-07-09 14:16:33.047  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:33.047  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@6c5ca0b6), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@6c5ca0b6, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@37b01ce2, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@1a88c4f5, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@1894fa9f, containsSubquery=false)
2021-07-09 14:16:33.047  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 8]
2021-07-09 14:16:33.047  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 8]
2021-07-09 14:16:33.047  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 8]
2021-07-09 14:16:33.047  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 8]
[{user_id=2, price=30.50, order_id=620244932891770880, status=SUCCESS}, {user_id=2, price=32.50, order_id=620244933034377216, status=SUCCESS}, {user_id=2, price=34.50, order_id=620244933168594944, status=SUCCESS}, {user_id=2, price=36.50, order_id=620244933298618368, status=SUCCESS}]
2021-07-09 14:16:33.070  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:33.070  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@33d230ce), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@33d230ce, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@35e74e08, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@a316f6b, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@63f9ddf9, containsSubquery=false)
2021-07-09 14:16:33.070  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 12]
2021-07-09 14:16:33.070  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 12]
2021-07-09 14:16:33.070  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 12]
2021-07-09 14:16:33.070  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 12]
[{user_id=2, price=38.50, order_id=620244933478973440, status=SUCCESS}, {user_id=2, price=21.50, order_id=620244931998384129, status=SUCCESS}, {user_id=2, price=23.50, order_id=620244932296179713, status=SUCCESS}, {user_id=2, price=25.50, order_id=620244932468146177, status=SUCCESS}]
2021-07-09 14:16:33.085  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:33.086  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@7caf1e5), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@7caf1e5, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@5c234920, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@7ddeb27f, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@38087342, containsSubquery=false)
2021-07-09 14:16:33.086  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 16]
2021-07-09 14:16:33.086  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 16]
2021-07-09 14:16:33.086  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 16]
2021-07-09 14:16:33.086  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 16]
[{user_id=2, price=27.50, order_id=620244932635918337, status=SUCCESS}, {user_id=2, price=29.50, order_id=620244932812079105, status=SUCCESS}, {user_id=2, price=31.50, order_id=620244932971462657, status=SUCCESS}, {user_id=2, price=33.50, order_id=620244933101486081, status=SUCCESS}]
2021-07-09 14:16:33.106  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:33.106  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@4e974b9e), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@4e974b9e, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@370c7cc5, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@61b838f2, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@2a04ab05, containsSubquery=false)
2021-07-09 14:16:33.108  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 20]
2021-07-09 14:16:33.109  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 20]
2021-07-09 14:16:33.109  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 20]
2021-07-09 14:16:33.109  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 20]
[{user_id=2, price=35.50, order_id=620244933235703809, status=SUCCESS}, {user_id=2, price=37.50, order_id=620244933369921537, status=SUCCESS}, {user_id=2, price=39.50, order_id=620244933688688641, status=SUCCESS}, {user_id=1, price=21.50, order_id=620244929406304256, status=SUCCESS}]
2021-07-09 14:16:33.125  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:33.126  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@7bf018dd), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@7bf018dd, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@15a8cebd, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@3f6c2763, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@5c82031b, containsSubquery=false)
2021-07-09 14:16:33.126  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 24]
2021-07-09 14:16:33.126  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 24]
2021-07-09 14:16:33.126  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 24]
2021-07-09 14:16:33.126  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 24]
[{user_id=1, price=23.50, order_id=620244930639429632, status=SUCCESS}, {user_id=1, price=25.50, order_id=620244930790424576, status=SUCCESS}, {user_id=1, price=27.50, order_id=620244930924642304, status=SUCCESS}, {user_id=1, price=29.50, order_id=620244931071442944, status=SUCCESS}]
2021-07-09 14:16:33.145  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:33.145  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@2ad7bd26), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@2ad7bd26, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@3cc3f13e, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@69b3886f, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@53d30d23, containsSubquery=false)
2021-07-09 14:16:33.145  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 28]
2021-07-09 14:16:33.145  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 28]
2021-07-09 14:16:33.146  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 28]
2021-07-09 14:16:33.146  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 28]
[{user_id=1, price=31.50, order_id=620244931230826496, status=SUCCESS}, {user_id=1, price=33.50, order_id=620244931360849920, status=SUCCESS}, {user_id=1, price=35.50, order_id=620244931537010688, status=SUCCESS}, {user_id=1, price=37.50, order_id=620244931654451200, status=SUCCESS}]
2021-07-09 14:16:33.166  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:33.167  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@5514579e), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@5514579e, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@58c42c8c, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@3af236d0, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@23564dd2, containsSubquery=false)
2021-07-09 14:16:33.167  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 32]
2021-07-09 14:16:33.167  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 32]
2021-07-09 14:16:33.167  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 32]
2021-07-09 14:16:33.167  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 32]
[{user_id=1, price=39.50, order_id=620244931901915136, status=SUCCESS}, {user_id=1, price=22.50, order_id=620244930555543553, status=SUCCESS}, {user_id=1, price=24.50, order_id=620244930719121409, status=SUCCESS}, {user_id=1, price=26.50, order_id=620244930849144833, status=SUCCESS}]
2021-07-09 14:16:33.186  INFO 3400 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ?
2021-07-09 14:16:33.187  INFO 3400 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@37314843, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@589fb74d), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@589fb74d, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@200d1a3d, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@7de147e9, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@12567179, containsSubquery=false)
2021-07-09 14:16:33.187  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 36]
2021-07-09 14:16:33.187  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 36]
2021-07-09 14:16:33.187  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 36]
2021-07-09 14:16:33.187  INFO 3400 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ,  ?   ) LIMIT ? , ? ::: [1, 2, 0, 36]
[{user_id=1, price=28.50, order_id=620244930995945473, status=SUCCESS}, {user_id=1, price=30.50, order_id=620244931134357505, status=SUCCESS}, {user_id=1, price=32.50, order_id=620244931302129665, status=SUCCESS}, {user_id=1, price=34.50, order_id=620244931486679041, status=SUCCESS}]
```

### (9). 测试带有路由分库的键(QueryNoRouteTest.queryInUserId)
> 这一次的Query只查user_id=2的数据,所以,路由到库时,只会在指定的库里查询结果,但是,结果的返回,依然是性线的结果,不会是多张表混合的结果.  

```
2021-07-09 14:17:36.312  INFO 3427 --- [           main] h.lixin.shardingjdbc.QueryNoRouteTest    : Started QueryNoRouteTest in 6.884 seconds (JVM running for 7.917)
2021-07-09 14:17:37.905  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:37.905  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@4dd2ef54), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@4dd2ef54, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@795b66d, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@359ceb13, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@1068176, containsSubquery=false)
2021-07-09 14:17:37.906  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 4]
2021-07-09 14:17:37.906  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 4]
[{user_id=2, price=22.50, order_id=620244932191322112, status=SUCCESS}, {user_id=2, price=24.50, order_id=620244932388454400, status=SUCCESS}, {user_id=2, price=26.50, order_id=620244932535255040, status=SUCCESS}, {user_id=2, price=28.50, order_id=620244932744970240, status=SUCCESS}]
2021-07-09 14:17:37.996  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:37.997  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@3c33fcf8), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@3c33fcf8, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@dada335, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@716f94c1, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@53feeac9, containsSubquery=false)
2021-07-09 14:17:37.997  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 8]
2021-07-09 14:17:37.997  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 8]
[{user_id=2, price=30.50, order_id=620244932891770880, status=SUCCESS}, {user_id=2, price=32.50, order_id=620244933034377216, status=SUCCESS}, {user_id=2, price=34.50, order_id=620244933168594944, status=SUCCESS}, {user_id=2, price=36.50, order_id=620244933298618368, status=SUCCESS}]
2021-07-09 14:17:38.004  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:38.004  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@3915e7c3), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@3915e7c3, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@167a21b, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@7c0df4ab, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@2e362407, containsSubquery=false)
2021-07-09 14:17:38.005  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 12]
2021-07-09 14:17:38.005  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 12]
[{user_id=2, price=38.50, order_id=620244933478973440, status=SUCCESS}, {user_id=2, price=21.50, order_id=620244931998384129, status=SUCCESS}, {user_id=2, price=23.50, order_id=620244932296179713, status=SUCCESS}, {user_id=2, price=25.50, order_id=620244932468146177, status=SUCCESS}]
2021-07-09 14:17:38.014  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:38.014  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@6f315b08), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@6f315b08, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@6c8efde4, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@24e5389c, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@3b170235, containsSubquery=false)
2021-07-09 14:17:38.014  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 16]
2021-07-09 14:17:38.014  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 16]
[{user_id=2, price=27.50, order_id=620244932635918337, status=SUCCESS}, {user_id=2, price=29.50, order_id=620244932812079105, status=SUCCESS}, {user_id=2, price=31.50, order_id=620244932971462657, status=SUCCESS}, {user_id=2, price=33.50, order_id=620244933101486081, status=SUCCESS}]
2021-07-09 14:17:38.030  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:38.031  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@4ae8fb2a), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@4ae8fb2a, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@54326e9, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@20216016, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@2b441e56, containsSubquery=false)
2021-07-09 14:17:38.031  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 20]
2021-07-09 14:17:38.031  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 20]
[{user_id=2, price=35.50, order_id=620244933235703809, status=SUCCESS}, {user_id=2, price=37.50, order_id=620244933369921537, status=SUCCESS}, {user_id=2, price=39.50, order_id=620244933688688641, status=SUCCESS}]
2021-07-09 14:17:38.042  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:38.043  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@1a2773a8), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@1a2773a8, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@78b0ec3a, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@46612bfc, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@4f213a2, containsSubquery=false)
2021-07-09 14:17:38.043  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 24]
2021-07-09 14:17:38.043  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 24]
[]
2021-07-09 14:17:38.052  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:38.053  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@1cde374), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@1cde374, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@6818fd48, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@9263c54, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@28daf506, containsSubquery=false)
2021-07-09 14:17:38.053  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 28]
2021-07-09 14:17:38.053  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 28]
[]
2021-07-09 14:17:38.071  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:38.071  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@799f354a), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@799f354a, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@33bdd01, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@159ac15f, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@7ac48f05, containsSubquery=false)
2021-07-09 14:17:38.071  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 32]
2021-07-09 14:17:38.072  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 32]
[]
2021-07-09 14:17:38.085  INFO 3427 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT * FROM t_order t where t.user_id in   (   ?   ) LIMIT ? , ?
2021-07-09 14:17:38.086  INFO 3427 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@433ef204, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@17216605), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@17216605, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=7, distinctRow=false, projections=[ShorthandProjection(owner=Optional.empty, actualColumns=[ColumnProjection(owner=null, name=order_id, alias=Optional.empty), ColumnProjection(owner=null, name=price, alias=Optional.empty), ColumnProjection(owner=null, name=user_id, alias=Optional.empty), ColumnProjection(owner=null, name=status, alias=Optional.empty)])]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@10a907ec, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@59b492ec, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@55c1ced9, containsSubquery=false)
2021-07-09 14:17:38.086  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_1 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 36]
2021-07-09 14:17:38.086  INFO 3427 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT * FROM t_order_2 t where t.user_id in   (   ?   ) LIMIT ? , ? ::: [2, 0, 36]
[]
2021-07-09 14:17:38.181  INFO 3427 --- [       Thread-2] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closing ...
2021-07-09 14:17:38.186  INFO 3427 --- [       Thread-2] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
2021-07-09 14:17:38.187  INFO 3427 --- [       Thread-2] com.alibaba.druid.pool.DruidDataSource   : {dataSource-2} closing ...
2021-07-09 14:17:38.193  INFO 3427 --- [       Thread-2] com.alibaba.druid.pool.DruidDataSource   : {dataSource-2} closed
2021-07-09 14:17:38.198  INFO 3427 --- [       Thread-2] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'
```
### (10). 统计结果(QueryNoRouteTest.count)
```
2021-07-09 14:21:36.611  INFO 3527 --- [           main] h.lixin.shardingjdbc.QueryNoRouteTest    : Started QueryNoRouteTest in 8.186 seconds (JVM running for 9.73)
2021-07-09 14:21:38.253  INFO 3527 --- [           main] ShardingSphere-SQL                       : Logic SQL: SELECT COUNT(*) FROM t_order
2021-07-09 14:21:38.253  INFO 3527 --- [           main] ShardingSphere-SQL                       : SQLStatement: SelectStatementContext(super=CommonSQLStatementContext(sqlStatement=org.apache.shardingsphere.sql.parser.sql.statement.dml.SelectStatement@6c8f4bc7, tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@714e861f), tablesContext=org.apache.shardingsphere.sql.parser.binder.segment.table.TablesContext@714e861f, projectionsContext=ProjectionsContext(startIndex=7, stopIndex=14, distinctRow=false, projections=[AggregationProjection(type=COUNT, innerExpression=(*), alias=Optional.empty, derivedAggregationProjections=[], index=-1)]), groupByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.groupby.GroupByContext@28989415, orderByContext=org.apache.shardingsphere.sql.parser.binder.segment.select.orderby.OrderByContext@6eda012b, paginationContext=org.apache.shardingsphere.sql.parser.binder.segment.select.pagination.PaginationContext@781dbe44, containsSubquery=false)
2021-07-09 14:21:38.254  INFO 3527 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT COUNT(*) FROM t_order_1
2021-07-09 14:21:38.254  INFO 3527 --- [           main] ShardingSphere-SQL                       : Actual SQL: d1 ::: SELECT COUNT(*) FROM t_order_2
2021-07-09 14:21:38.254  INFO 3527 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT COUNT(*) FROM t_order_1
2021-07-09 14:21:38.254  INFO 3527 --- [           main] ShardingSphere-SQL                       : Actual SQL: d2 ::: SELECT COUNT(*) FROM t_order_2
38
```

### (10). 总结
Sharding JDBC在Query时,当查询条件中没有sharding-column(分片的列)时,虽然SQL语句会分发到所有的分片.   
<font color='red'>但是,返回的结果不会是多张表的混合结果,它会取第一个分片的结果,直到第一个分片的结果取完,再取第二个分片的结果,依次类推.</font>  
  