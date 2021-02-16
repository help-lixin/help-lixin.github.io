---
layout: post
title: 'Sharding-JDBC TableRule(四)'
date: 2019-12-28
author: 李新
tags: Sharding-JDBC源码
---

### (1). 概述
> 在这里,要对:TableRule进行一个详细的剖析,为什么呢?因为:这个类承载着太多的业务.    
> 比如:我想自定义主键生成策略,想要自定义分表分库策略怎么办?     
> 剖析完:TableRule的内容后,就能找到答案了.   

### (2). TableRule类结构图
> 仍然,参考上一小节剖析的UML图.   

!["TableRule类结构图"](/assets/sharding-jdbc/imgs/ShardingRule-Class-Diagram.jpg)

### (3). TableRule在什么时候进行初始化的?
> TableRule的初始化是在:ShardingRule构建器中进行的.

```
private final Collection<TableRule> tableRules;

public ShardingRule(final ShardingRuleConfiguration shardingRuleConfig, final Collection<String> dataSourceNames) {
	// ... ...
	// ******************************************************
	// 4. 
	//    4.1. 解析:ShardingRuleConfiguration.Collection<TableRuleConfiguration> 
	//    4.2. 转换成:Collection<TableRule>
	// ******************************************************
	tableRules = createTableRules(shardingRuleConfig);
	// ... ...
}
```

### (4). ShardingRule.createTableRules
```
private Collection<TableRule> createTableRules(final ShardingRuleConfiguration shardingRuleConfig) {
	// 4.1 遍历:shardingRuleConfig.getTableRuleConfigs()
	return shardingRuleConfig.getTableRuleConfigs()
		 .stream()
		 // 把TableRuleConfiguration转换成:TableRule
		 .map(each -> new TableRule(each, shardingDataSourceNames, getDefaultGenerateKeyColumn(shardingRuleConfig)))
		 .collect(Collectors.toList());
}
```
### (5). TableRule
> 从上面分析,不难看出:TableRule是在:ShardingRule.createTableRules创建的.   
> 这里主要是分析:TableRule对:ShardingStrategy/ShardingKeyGenerator有没有提供相应的扩展点,以供我们使用.   

```
public TableRule(final TableRuleConfiguration tableRuleConfig, final ShardingDataSourceNames shardingDataSourceNames, final String defaultGenerateKeyColumn) {
	// 逻辑表名称
	logicTable = tableRuleConfig.getLogicTable().toLowerCase();
	// 数据节点(这时,已经对表达式进行了解析)
	List<String> dataNodes = new InlineExpressionParser(tableRuleConfig.getActualDataNodes()).splitAndEvaluate();
	
	dataNodeIndexMap = new HashMap<>(dataNodes.size(), 1);
	actualDataNodes = isEmptyDataNodes(dataNodes)
		? generateDataNodes(tableRuleConfig.getLogicTable(), shardingDataSourceNames.getDataSourceNames()) : generateDataNodes(dataNodes, shardingDataSourceNames.getDataSourceNames());
	actualTables = getActualTables();
	
	// ************************************************************************************
	// 6. 通过ShardingStrategyFactory.newInstance,创建:ShardingStrategy
	//    这是典型的工厂模式
	// ************************************************************************************
	databaseShardingStrategy = 
	            (null == tableRuleConfig.getDatabaseShardingStrategyConfig()) ? 
				null : 
				// 
				ShardingStrategyFactory.newInstance(tableRuleConfig.getDatabaseShardingStrategyConfig());
	tableShardingStrategy = null == tableRuleConfig.getTableShardingStrategyConfig() ? null : ShardingStrategyFactory.newInstance(tableRuleConfig.getTableShardingStrategyConfig());
	final KeyGeneratorConfiguration keyGeneratorConfiguration = tableRuleConfig.getKeyGeneratorConfig();
	generateKeyColumn = null != keyGeneratorConfiguration && !Strings.isNullOrEmpty(keyGeneratorConfiguration.getColumn()) ? keyGeneratorConfiguration.getColumn() : defaultGenerateKeyColumn;
	
	// ************************************************************************************
	// 10. 通过:ShardingKeyGeneratorServiceLoader创建:ShardingKeyGenerator
	//     通过SPI来加载
	// ************************************************************************************
	shardingKeyGenerator = containsKeyGeneratorConfiguration(tableRuleConfig)
			? new ShardingKeyGeneratorServiceLoader().newService(tableRuleConfig.getKeyGeneratorConfig().getType(), tableRuleConfig.getKeyGeneratorConfig().getProperties()) : null;
	checkRule(dataNodes);
}
```
### (6). ShardingStrategyFactory.newInstance
> Shardind-JDBC对分库(分表)策略进行了归类,从现在的代码上来看,它只支持这几类的策略,也就是说:你只能基于这几种进行业务的扩展.    

```
public static ShardingStrategy newInstance(final ShardingStrategyConfiguration shardingStrategyConfig) {
	if (shardingStrategyConfig instanceof StandardShardingStrategyConfiguration) {
		// 精确分片算法
		return new StandardShardingStrategy((StandardShardingStrategyConfiguration) shardingStrategyConfig);
	}
	if (shardingStrategyConfig instanceof InlineShardingStrategyConfiguration) {
		// Inline分片算法
		return new InlineShardingStrategy((InlineShardingStrategyConfiguration) shardingStrategyConfig);
	}
	if (shardingStrategyConfig instanceof ComplexShardingStrategyConfiguration) {
		// 复合分片算法
		return new ComplexShardingStrategy((ComplexShardingStrategyConfiguration) shardingStrategyConfig);
	}
	if (shardingStrategyConfig instanceof HintShardingStrategyConfiguration) {
		// Hint分片算法
		return new HintShardingStrategy((HintShardingStrategyConfiguration) shardingStrategyConfig);
	}
	// 不进行分片
	return new NoneShardingStrategy();
}
```
### (7). ShardingStrategy分片策略详解
> <color='red'>StandardShardingStrategy</color>:   
> StandardShardingStrategy标准分片策略,提供对SQL语句中的=,IN和BETWEEN AND的分片操作支持.   
> StandardShardingStrategy只支持单分片键,提供PreciseShardingAlgorithm和RangeShardingAlgorithm两个分片算法.   
> PreciseShardingAlgorithm是必选的,用于处理=和IN的分片.     
> RangeShardingAlgorithm是可选的,用于处理BETWEEN AND分片,如果不配置RangeShardingAlgorithm,SQL中的BETWEEN AND将按照全库路由处理.    

> <color='red'>ComplexShardingStrategy</color>:    
> 复合分片策略,提供对SQL语句中的=,IN和BETWEEN AND的分片操作支持.    
> ComplexShardingStrategy支持多分片键,由于多分片键之间的关系复杂,因此Sharding-JDBC并未做过多的封装,而是直接将分片键值组合以及分片操作符交于算法接口,完全由应用开发者实现,提供最大的灵活度.   

> <color='red'>InlineShardingStrategy</color>:  
> Inline表达式分片策略,使用Groovy的Inline表达式,提供对SQL语句中的=和IN的分片操作支持. 
> InlineShardingStrategy只支持单分片键,对于简单的分片算法,可以通过简单的配置使用,从而避免繁琐的Java代码开发,如:t_user${user_id % 8},表示t_user表按照user_id按8取模分成8个表,表名称为t_user_0到t_user_7.  

> <color='red'>HintShardingStrategy</color>:   
> 通过Hint而非SQL解析的方式分片的策略

> <color='red'>NoneShardingStrategy</color>:   
> 不分片的策略

### (8). ShardingAlgorithm类结构图
!["ShardingAlgorithm类结构图"](/assets/sharding-jdbc/imgs/ShardingAlgorithm.jpg)

### (9). 如何自定义分片策略呢
> 从代码我们分的出,newInstance是不提供扩展点给我们用的.但是.相应的ShardingStrategy的实现类,是可以支持我们对算法(ShardingAlgorithm)自由配置的.
> 1. 自定义一个ShardingAlgorithm(例如:CustomerComplexKeysShardingAlgorithm),需要实现:ShardingAlgorithm的子接口.      
> 2. 将ShardingStrategy与ShardingAlgorithm过行绑定(注意:InlineShardingStrategy不支持该功能).   
> 3. 下面的代码以:YamlComplexShardingStrategyConfiguration为例,进行分析.      

```
// **************************************YamlComplexShardingStrategyConfiguration**************************************
// 1. 对应的配置信息如下:  
// spring.shardingsphere.sharding.tables.t_order.database-strategy.complex.sharding-columns=user_id,order_id
// spring.shardingsphere.sharding.tables.t_order.database-strategy.complex.algorithm-class-name=help.lixin.CustomerComplexKeysShardingAlgorithm

public final class YamlComplexShardingStrategyConfiguration 
             implements YamlBaseShardingStrategyConfiguration {
	// 分片columns,多个之间用逗号隔开			 
    private String shardingColumns;
	// 分片的算法名称(需要全名称)
    private String algorithmClassName;
} // end YamlComplexShardingStrategyConfiguration


// **************************************// ShardingStrategyConfigurationYamlSwapper**************************************
// 2. help.lixin.CustomerComplexKeysShardingAlgorithm加载时机.
public ShardingStrategyConfiguration swap(final YamlShardingStrategyConfiguration yamlConfiguration) {
	// ... ...
	if (null != yamlConfiguration.getComplex()) {
		shardingStrategyConfigCount++;
		result = new ComplexShardingStrategyConfiguration(
		        yamlConfiguration.getComplex().getShardingColumns(), 
				// *****************************************************************
				// Class.forName(shardingAlgorithmClassName);
				// 通过反射加载自定义的分片算法来着的.
				// *****************************************************************
				ShardingAlgorithmFactory.newInstance(yamlConfiguration.getComplex().getAlgorithmClassName(),ComplexKeysShardingAlgorithm.class));
	}
	// ... ...
	Preconditions.checkArgument(shardingStrategyConfigCount <= 1, "Only allowed 0 or 1 sharding strategy configuration.");
	return result;
}
```
### (10). ShardingKeyGeneratorServiceLoader
```
public final class ShardingKeyGeneratorServiceLoader 
            extends TypeBasedSPIServiceLoader<ShardingKeyGenerator> {
    static {
		// 1. 通过SPI,加载:ShardingKeyGenerator的所有实现
        NewInstanceServiceLoader.register(ShardingKeyGenerator.class);
    }
    
    public ShardingKeyGeneratorServiceLoader() {
		// 2. 定义要加载的类型为:ShardingKeyGenerator
        super(ShardingKeyGenerator.class);
    }
} //end ShardingKeyGeneratorServiceLoader
```
### (11). ShardingKeyGeneratorServiceLoader.newService
> 通过对:ShardingKeyGeneratorServiceLoader.newService进行分析.Sharding-JDBC提供了扩展点,允许自定义:ShardingKeyGenerator.   
> 自定义ShardingKeyGenerator步骤如下:  
> 1. 自定义ShardingKeyGenerator(例如:CustomerShardingKeyGenerator).需要实现:ShardingKeyGenerator接口.  
> 2. 例如:CustomerShardingKeyGenerator.getType() == SNOWFLAKE_NEW.  
> 3. 配置SPI信息:/META-INF/services/org.apache.shardingsphere.spi.keygen.ShardingKeyGenerator(help.lixin.CustomerShardingKeyGenerator).   
> 4. 定义配置(spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE_NEW).   

```
public abstract class TypeBasedSPIServiceLoader<T extends TypeBasedSPI> {
	// ShardingKeyGenerator
    private final Class<T> classType;
	
	public final T newService(final String type, final Properties props) {
		// type = "SNOWFLAKE"
		// 1. loadTypeBasedServices
		Collection<T> typeBasedServices = loadTypeBasedServices(type);
		if (typeBasedServices.isEmpty()) {
			throw new RuntimeException(String.format("Invalid `%s` SPI type `%s`.", classType.getName(), type));
		}
		T result = typeBasedServices.iterator().next();
		result.setProperties(props);
		return result;
	} // end newService
	
	private Collection<T> loadTypeBasedServices(final String type) {
		// 4. 加载:ShardingKeyGenerator的所有实现,并获取实例中:getType() == SNOWFLAKE的ShardingKeyGenerator对象
		return Collections2.filter(
			// 2. 加载:ShardingKeyGenerator的所有实现类,并进行实例化.
			NewInstanceServiceLoader.newServiceInstances(classType), 
			// 3. 所有:ShardingKeyGenerator的实现类,都要实现:getType()方法
			input -> type.equalsIgnoreCase(input.getType()));
	} // end loadTypeBasedServices
}
```
### (12). 总结
> 这一部份,对TableRule进行了深入的分析,因为:它承载的信息太多,最主要是因为:后续可能会有一些业务需要自定义分片策略或主键生成策略.  