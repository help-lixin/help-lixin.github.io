---
layout: post
title: 'Sharding-JDBC SpringBootConfiguration(一)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC源码
---

### (1). Sharding-JDBC 源码该如何跟踪?
> 通过:sharding-jdbc-spring-boot-starter-4.1.1.jar/META-INF/spring.factories找到Spring Boot的入口.  

### (2). application.properties

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
# 配置ds1的master为:d1,slave为:s1
# org.apache.shardingsphere.core.yaml.config.masterslave.YamlMasterSlaveRuleConfiguration
spring.shardingsphere.sharding.master‐slave‐rules.ds1.master‐data‐source‐name=d1
spring.shardingsphere.sharding.master‐slave‐rules.ds1.slave‐data‐source‐names=s1

# 配置ds2的master为:d2,slave为:s2
spring.shardingsphere.sharding.master‐slave‐rules.ds2.master‐data‐source‐name=d2
spring.shardingsphere.sharding.master‐slave‐rules.ds2.slave‐data‐source‐names=s2

# 指定逻辑表:t_order的数据节点:[d1/d2].t_order_1,d1.t_order_2
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds$->{1..2}.t_order_$->{1..2}

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
### (3). SpringBootConfiguration.shardingDataSource
> 1. 初始化真实的DataSource.  
> 2. 创建事务AOP.  
> 3. 创建ShardingDataSource,代理多个:DataSource.   

```
package org.apache.shardingsphere.shardingjdbc.spring.boot;


@Configuration
@ComponentScan("org.apache.shardingsphere.spring.boot.converter")
@EnableConfigurationProperties({
        SpringBootShardingRuleConfigurationProperties.class,
        SpringBootMasterSlaveRuleConfigurationProperties.class, SpringBootEncryptRuleConfigurationProperties.class,
        SpringBootPropertiesConfigurationProperties.class, SpringBootShadowRuleConfigurationProperties.class})
@ConditionalOnProperty(prefix = "spring.shardingsphere", name = "enabled", havingValue = "true", matchIfMissing = true)
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
@RequiredArgsConstructor
public class SpringBootConfiguration 
		// **********************************************************
	   // 2. 由于实现了:EnvironmentAware,所以Spring会回调该方法
	   // **********************************************************
       implements EnvironmentAware {
	
	// **********************************************************
	// 1. 由于@EnableConfigurationProperties({})
	//     Spring会读取配置文件,并映射到实体模型上.
	// **********************************************************
    private final SpringBootShardingRuleConfigurationProperties shardingRule;
	private final SpringBootMasterSlaveRuleConfigurationProperties masterSlaveRule;
	private final SpringBootEncryptRuleConfigurationProperties encryptRule;
	private final SpringBootShadowRuleConfigurationProperties shadowRule;
	private final SpringBootPropertiesConfigurationProperties props;
	private final Map<String, DataSource> dataSourceMap = new LinkedHashMap<>();
	private final String jndiName = "jndi-name";
	

	// ***************************************************************
	// 3. 创建事务扫描器(@ShardingTransactionType)
	// ***************************************************************
	@Bean
	public ShardingTransactionTypeScanner shardingTransactionTypeScanner() {
		return new ShardingTransactionTypeScanner();
	}

	// ***************************************************************
	// 4. 基于前面创建的DataSource,封装成一个新的:DataSource,即:ShardingDataSource类
	// ***************************************************************
	@Bean
	@Conditional(ShardingRuleCondition.class)
	public DataSource shardingDataSource() throws SQLException {
		return ShardingDataSourceFactory.createDataSource(dataSourceMap, new ShardingRuleConfigurationYamlSwapper().swap(shardingRule), props.getProps());
	}
	
	
	// **********************************************************
	// 2. 由于实现了:EnvironmentAware,所以Spring会回调该方法
	// **********************************************************
	public final void setEnvironment(final Environment environment) {
		String prefix = "spring.shardingsphere.datasource.";
		// 2.1 getDataSourceNames
		for (String each : getDataSourceNames(environment, prefix)) {
			try {
				// 2.3 getDataSource
				// 初始化数据源,通过:SpringBootConfiguration.dataSourceMap
				// Hold住所有用户定义的数据源.
				dataSourceMap.put(each, getDataSource(environment, prefix, each));
			} catch (final ReflectiveOperationException ex) {
				throw new ShardingSphereException("Can't find datasource type!", ex);
			} catch (final NamingException namingEx) {
				throw new ShardingSphereException("Can't find JNDI datasource!", namingEx);
			}
		}
	}// end setEnvironment
	
	// 2.2 getDataSourceNames
	private List<String> getDataSourceNames(final Environment environment, final String prefix) {
		StandardEnvironment standardEnv = (StandardEnvironment) environment;
		standardEnv.setIgnoreUnresolvableNestedPlaceholders(true);
		// spring.shardingsphere.datasource.name
		// spring.shardingsphere.datasource.names
		return null == standardEnv.getProperty(prefix + "name")
				? new InlineExpressionParser(standardEnv.getProperty(prefix + "names")).splitAndEvaluate() : Collections.singletonList(standardEnv.getProperty(prefix + "name"));
	} // end getDataSourceNames
	
	// 2.3 getDataSource
	private DataSource getDataSource(final Environment environment, final String prefix, final String dataSourceName) throws ReflectiveOperationException, NamingException {
		Map<String, Object> dataSourceProps = PropertyUtil.handle(environment, prefix + dataSourceName.trim(), Map.class);
		Preconditions.checkState(!dataSourceProps.isEmpty(), "Wrong datasource properties!");
		if (dataSourceProps.containsKey(jndiName)) {
			return getJndiDataSource(dataSourceProps.get(jndiName).toString());
		}
		DataSource result = DataSourceUtil.getDataSource(dataSourceProps.get("type").toString(), dataSourceProps);
		DataSourcePropertiesSetterHolder.getDataSourcePropertiesSetterByType(dataSourceProps.get("type").toString()).ifPresent(
			dataSourcePropertiesSetter -> dataSourcePropertiesSetter.propertiesSet(environment, prefix, dataSourceName, result));
		return result;
	} // end getDataSource
}

```
### (4). 总结
> Sharding-JDBC对DataSource(ShardingDataSource)进行了扩展,应用程序获取Connection时,将会是调用:ShardingDataSource.getConnection方法.   
> 所以:Sharding-JDBC代码的入口是:ShardingDataSource类.   