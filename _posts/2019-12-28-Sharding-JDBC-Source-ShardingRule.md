---
layout: post
title: 'Sharding-JDBC ShardingRule(三)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC源码
---

### (1). 概述
> 在这一小节,主要剖析:ShardingDataSource的创建过程.  

### (2). SpringBootConfiguration.shardingDataSource
```
@Bean
@Conditional(ShardingRuleCondition.class)
public DataSource shardingDataSource() throws SQLException {
	// 4. 通过:ShardingDataSourceFactory的工厂方法创建:DataSource
	return ShardingDataSourceFactory
	       .createDataSource(
		        dataSourceMap, 
				// 3. 把SpringBootShardingRuleConfigurationProperties配置,转换成规则配置模型:ShardingRuleConfiguration.  
				new ShardingRuleConfigurationYamlSwapper().swap(shardingRule), 
				props.getProps()
			); // end createDataSource
}
```
### (3). ShardingRuleConfigurationYamlSwapper.swap
> 这一步做法是把SpringBootShardingRuleConfigurationProperties转换成:ShardingRuleConfiguration.   
> 可以理解为配置 => 业务模型的转换.   

```
public ShardingRuleConfiguration swap(final YamlShardingRuleConfiguration yamlConfiguration) {
	ShardingRuleConfiguration result = new ShardingRuleConfiguration();
	
	for (Entry<String, YamlTableRuleConfiguration> entry : yamlConfiguration.getTables().entrySet()) {
		YamlTableRuleConfiguration tableRuleConfig = entry.getValue();
		tableRuleConfig.setLogicTable(entry.getKey());
		result.getTableRuleConfigs().add(tableRuleConfigurationYamlSwapper.swap(tableRuleConfig));
	}
	
	result.setDefaultDataSourceName(yamlConfiguration.getDefaultDataSourceName());
	result.getBindingTableGroups().addAll(yamlConfiguration.getBindingTables());
	result.getBroadcastTables().addAll(yamlConfiguration.getBroadcastTables());
	if (null != yamlConfiguration.getDefaultDatabaseStrategy()) {
		result.setDefaultDatabaseShardingStrategyConfig(shardingStrategyConfigurationYamlSwapper.swap(yamlConfiguration.getDefaultDatabaseStrategy()));
	}
	if (null != yamlConfiguration.getDefaultTableStrategy()) {
		result.setDefaultTableShardingStrategyConfig(shardingStrategyConfigurationYamlSwapper.swap(yamlConfiguration.getDefaultTableStrategy()));
	}
	if (null != yamlConfiguration.getDefaultKeyGenerator()) {
		result.setDefaultKeyGeneratorConfig(keyGeneratorConfigurationYamlSwapper.swap(yamlConfiguration.getDefaultKeyGenerator()));
	}
	Collection<MasterSlaveRuleConfiguration> masterSlaveRuleConfigs = new LinkedList<>();
	for (Entry<String, YamlMasterSlaveRuleConfiguration> entry : yamlConfiguration.getMasterSlaveRules().entrySet()) {
		YamlMasterSlaveRuleConfiguration each = entry.getValue();
		each.setName(entry.getKey());
		masterSlaveRuleConfigs.add(masterSlaveRuleConfigurationYamlSwapper.swap(entry.getValue()));
	}
	result.setMasterSlaveRuleConfigs(masterSlaveRuleConfigs);
	if (null != yamlConfiguration.getEncryptRule()) {
		result.setEncryptRuleConfig(encryptRuleConfigurationYamlSwapper.swap(yamlConfiguration.getEncryptRule()));
	}
	return result;
}
```
### (4). ShardingDataSourceFactory.createDataSource 
```
public static DataSource createDataSource(
		// 所有的真实数据源
		final Map<String, DataSource> dataSourceMap, 
		// 在第3步中,把:SpringBootShardingRuleConfigurationProperties转换成:ShardingRuleConfiguration.   
		final ShardingRuleConfiguration shardingRuleConfig, 
		// 其它的属性配置信息.
		final Properties props) throws SQLException {
		
	return new ShardingDataSource(  
	     dataSourceMap, 
		 // 5. 将:ShardingRuleConfiguration再次进行解析,转换成:ShardingRule
		 new ShardingRule(shardingRuleConfig, dataSourceMap.keySet()), 
		 props);
}
```
### (5). ShardingRule
> Sharding-JDBC对于业务模型有两次转换,要稍微的注意下.   
> 1. SpringBootShardingRuleConfigurationProperties 转换成  ShardingRuleConfiguration.    
> 2. ShardingRuleConfiguration 转换成  ShardingRule.     

```
public class ShardingRule implements BaseRule {
    // 规则配置
    private final ShardingRuleConfiguration ruleConfiguration;
    // 分片的数据源名称
    private final ShardingDataSourceNames shardingDataSourceNames;
    
	// 表规则
    private final Collection<TableRule> tableRules;
    
	// 绑定表
    private final Collection<BindingTableRule> bindingTableRules;
    
	// 公共表
    private final Collection<String> broadcastTables;
    
	// 默认的数据源分片策略
    private final ShardingStrategy defaultDatabaseShardingStrategy;
    
	// 默认的表分片策略
    private final ShardingStrategy defaultTableShardingStrategy;
    
	// 默认的主键生成
    private final ShardingKeyGenerator defaultShardingKeyGenerator;
    
	// 主从(读写分离)规则
    private final Collection<MasterSlaveRule> masterSlaveRules;
    
	// 加密解密规则
    private final EncryptRule encryptRule;
    
    public ShardingRule(final ShardingRuleConfiguration shardingRuleConfig, final Collection<String> dataSourceNames) {
        Preconditions.checkArgument(null != shardingRuleConfig, "ShardingRuleConfig cannot be null.");
        Preconditions.checkArgument(null != dataSourceNames && !dataSourceNames.isEmpty(), "Data sources cannot be empty.");
		
        this.ruleConfiguration = shardingRuleConfig;
        shardingDataSourceNames = new ShardingDataSourceNames(shardingRuleConfig, dataSourceNames);
		
		// 1. 把 Collection<TableRuleConfiguration> 转换成 Collection<TableRule> 
		//    TableRule是含有:分库/分表的规则的
        tableRules = createTableRules(shardingRuleConfig);
		// 2. 公共表
        broadcastTables = shardingRuleConfig.getBroadcastTables();
		// 绑定表
        bindingTableRules = createBindingTableRules(shardingRuleConfig.getBindingTableGroups());
		
		// 把 ShardingStrategyConfiguration(defaultDatabaseShardingStrategyConfig) 转换成  ShardingStrategy
        defaultDatabaseShardingStrategy = createDefaultShardingStrategy(shardingRuleConfig.getDefaultDatabaseShardingStrategyConfig());
		
		// 把 ShardingStrategyConfiguration(defaultTableShardingStrategyConfig) 转换成  ShardingStrategy
        defaultTableShardingStrategy = createDefaultShardingStrategy(shardingRuleConfig.getDefaultTableShardingStrategyConfig());
		
		// 把 KeyGeneratorConfiguration 转换成 ShardingKeyGenerator
        defaultShardingKeyGenerator = createDefaultKeyGenerator(shardingRuleConfig.getDefaultKeyGeneratorConfig());
		// 把 Collection<MasterSlaveRuleConfiguration>  转换成 Collection<MasterSlaveRule>
        masterSlaveRules = createMasterSlaveRules(shardingRuleConfig.getMasterSlaveRuleConfigs());
		
		把 EncryptRuleConfiguration 转换成  EncryptRule
        encryptRule = createEncryptRule(shardingRuleConfig.getEncryptRuleConfig());
    } // end 构造器
}
```
### (6). 看下ShardingRule的类结构图
!["ShardingRule类结构图"](/assets/sharding-jdbc/imgs/ShardingRule-Class-Diagram.jpg)

### (7). 总结
> Sharding-JDBC经过两次模型的转换,最终业务模型为:ShardingRule,后续所有的业务都会与这个类打交道.
> 在这里,先只对ShardingRule进行了一大概的了解.   