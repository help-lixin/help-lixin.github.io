---
layout: post
title: 'Sharding-JDBC SpringBootShardingRuleConfigurationProperties(二)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC源码
---

### (1). 概述
> 在前面剖析了:SpringBootConfiguration在初始化时,会读取配置项(spring.shardingsphere.datasource.names),并构建DataSource.   
> 在这里,主要剖析Sharding-JDBC的规则配置项
### (2). 看下SpringBootShardingRuleConfigurationProperties类结构图
> 通过UML分析,能看到典型的策略模式.

!["SpringBootShardingRuleConfigurationProperties"](/assets/sharding-jdbc/imgs/Sharding-JDBC-SpringBootShardingRuleConfigurationProperties-Class-Diagram.jpg)

### (3). YamlTableRuleConfiguration
```
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingStrategyConfiguration
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=d$->{1..2}.t_order_$->{1..2}


# 分库实际要做的是找到对应上面的数据源名称
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingStrategyConfiguration
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.algorithm-expression=d$->{user_id % 2 + 1}


# 指定逻辑表t_order表的主键(order_id)生成策略为:雪花算法.
# org.apache.shardingsphere.core.yaml.config.sharding.YamlKeyGeneratorConfiguration
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE


# 分表实际要做的找到真实的表名称(t_order_1/t_order_2)
# 指定t_order表的分片策略，分片策略包括分片键和分片算法
# org.apache.shardingsphere.core.yaml.config.sharding.YamlShardingStrategyConfiguration
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order_$->{order_id % 2 + 1}
```
### (4). 总结
> SpringBootShardingRuleConfigurationProperties是Sharding-JDBC的业务模型类,这个类比较重要.它承载着Sharding-JDBC的重要配置信息.     