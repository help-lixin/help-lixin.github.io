---
layout: post
title: 'Liquibase Hello World(二)'
date: 2021-02-27
author: 李新
tags: Liquibase
---

### (1). 概述
> 在这时,我先搭个Hello World.步骤如下:    
> 1. 编写xml文件(Liquibase内部会根据XML转换成SQL文件).  
> 2. 建库.  
> 3. 让Liquibase在刚建的库里,根据XML创建表.    

### (2). 建库
```
mysql> CREATE DATABASE lbcat;
Query OK, 1 row affected (0.01 sec)

mysql> USE lbcat;
Database changed

mysql> SHOW TABLES;
Empty set (0.00 sec)
```

### (3). pom.xml添加依赖
```
<dependency>
	<groupId>org.liquibase</groupId>
	<artifactId>liquibase-core</artifactId>
	<version>4.3.1</version>
</dependency>
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid</artifactId>
	<version>1.2.4</version>
</dependency>
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<version>5.1.49</version>
</dependency>
```
### (4). root.changelog.xml(changelog编写)
> changelog现在支持:xml/json/yaml/sql文件.  

```
<?xml version="1.0" encoding="UTF-8"?>

<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-2.0.xsd">
    <preConditions>
        <dbms type="mysql"/>
        <!-- 
        <sqlCheck expectedResult="1">select 1</sqlCheck>
        <runningAs username="${loginUser}"/>
         -->
    </preConditions>

    <changeSet id="1" author="nvoxland">
        <comment>
            You can add comments to changeSets.
            They can even be multiple lines if you would like.
            They aren't used to compute the changeSet MD5Sum, so you can update them whenever you want without causing
            problems.
        </comment>
		<!-- 表名称是变量 -->
        <createTable tableName="${person}">
            <column name="id" type="int" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="firstname" type="varchar(50)"/>
            <column name="lastname" type="varchar(50)">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>
    <changeSet id="2" author="nvoxland">
        <comment>Add a username column so we can use "person" for authentication</comment>
        <addColumn tableName="${person}">
            <column name="usernae" type="varchar(8)"/>
        </addColumn>
    </changeSet>
</databaseChangeLog>
```
### (5). LiquibaseTest
```
package help.lixin.liquibase;
import java.sql.SQLException;
import javax.sql.DataSource;
import com.alibaba.druid.pool.DruidDataSource;

import liquibase.Liquibase;
import liquibase.database.Database;
import liquibase.database.core.MySQLDatabase;
import liquibase.database.jvm.JdbcConnection;
import liquibase.exception.LiquibaseException;
import liquibase.integration.commandline.CommandLineResourceAccessor;
import liquibase.resource.FileSystemResourceAccessor;
import liquibase.resource.ResourceAccessor;

public class LiquibaseTest {
	@SuppressWarnings({ "deprecation", "resource" })
	public static void main(String[] args) throws Exception {
		String contexts = "test, context-b";
		String changeLogFile = "root.changelog.xml";
		// ****************************************************
		// 向Liquibase传递表名称(key:逻辑表名称  value:真实表名称)
		// ****************************************************
		// System.getProperties().put("person", "t_person");
		ResourceAccessor resourceAccessor = new CommandLineResourceAccessor(
				Thread.currentThread().getContextClassLoader());
		Database database = buildDatabase();
		Liquibase liquibase = new Liquibase(changeLogFile, resourceAccessor, database);
		try {
			// ****************************************************
			// 向Liquibase传递表名称(key:逻辑表名称  value:真实表名称)
			// ****************************************************
			liquibase.setChangeLogParameter("person", "t_person");
			liquibase.update(contexts);
			System.out.println("");
		} catch (LiquibaseException e) {
			e.printStackTrace();
		}
	}

	// 创建Database,Liquibase对DataSource进行了扩展.
	static Database buildDatabase() throws Exception {
		DataSource dataSource = dataSource();
		JdbcConnection jdbcConnection = new JdbcConnection(dataSource.getConnection());
		Database database = new MySQLDatabase();
		database.setConnection(jdbcConnection);
		return database;
	}

	// 创建数据源(DataSource)
	static DataSource dataSource() {
		String url = "jdbc:mysql://localhost:3306/lbcat?useSSL=false";
		String userName = "root";
		String pwd = "123456";
		DruidDataSource dataSource = new DruidDataSource();
		dataSource.setUrl(url);
		dataSource.setUsername(userName);
		dataSource.setPassword(pwd);
		dataSource.setDefaultAutoCommit(Boolean.FALSE);

		try {
			dataSource.init();
		} catch (SQLException e) {
			e.printStackTrace();
		}
		return dataSource;
	}
}
```
### (6). 查看结果
```
mysql> SHOW TABLES;
+-----------------------+
| Tables_in_lbcat       |
+-----------------------+
| DATABASECHANGELOG     |
| DATABASECHANGELOGLOCK |
| t_person              |
+-----------------------+
3 rows in set (0.00 sec)


mysql> DESC t_person;
+-----------+-------------+------+-----+---------+----------------+
| Field     | Type        | Null | Key | Default | Extra          |
+-----------+-------------+------+-----+---------+----------------+
| id        | int(11)     | NO   | PRI | NULL    | auto_increment |
| firstname | varchar(50) | YES  |     | NULL    |                |
| lastname  | varchar(50) | NO   |     | NULL    |                |
| usernae   | varchar(8)  | YES  |     | NULL    |                |
+-----------+-------------+------+-----+---------+----------------+
4 rows in set (0.00 sec)
```

### (7). 总结
> Liquibase帮我们建了好了表,我这里只使用XML的方式,后面对源剖析,也是以XML为主.  
> 可能就有人会说了,对于分表分库,应该要怎么做呢?比如:sharding-jdbc定义的虚拟表,要如何生成真实的表呢?  
> Shardin-jdbc会对逻辑表转换成真实表,获取这些信息表的信息,以及对应的DataSource.然后封装成Database即可.  
> 肯定又有人要问,能不能在所有的表创建完成之后,再向Eureka进行注册了呢?这个功能是可以的,你只要自己重写(代理):EurekaAutoServiceRegistration即可实现这功能.  
> 比如:所有的表创建完成之后,发布一件事件,当接受到这个事件时,就触发向Eureka进行注册.

### (8). sharding-jdbc伪代码

> Sharding-jdbc与Liquibase结合,如何获取真实的数据源和DataSource.

```
DataSource shadowDataSource = (DataSource) ctx.getBean("shardingDataSource");

if (shadowDataSource instanceof ShardingDataSource) {
	ShardingDataSource shardingDataSource = (ShardingDataSource) shadowDataSource;

	// 所有的数据源集合.
	Map<String, DataSource> dataSourceMap = shardingDataSource.getDataSourceMap();

	// 上下文中获得配置信息.
	ShardingRuntimeContext shardingRuntimeContext = shardingDataSource.getRuntimeContext();
	ShardingRule shardingRule = shardingRuntimeContext.getRule();
	Collection<TableRule> tableRules = shardingRule.getTableRules();
	for (TableRule tableRule : tableRules) {
		// 逻辑表名称
		String logicTable = tableRule.getLogicTable();
		List<DataNode> dataNodes = tableRule.getActualDataNodes();
		dataNodes.forEach(dataNode -> {
			String tableName = dataNode.getTableName();
			String dataSourceName = dataNode.getDataSourceName();
			String formatLine = String.format("logicTable:[%s],dataSource:[%s],tableNmae:[%s]", logicTable,
					dataSourceName, tableName);
					
			// logicTable:[t_order],dataSource:[d1],tableNmae:[t_order_1]
			// logicTable:[t_order],dataSource:[d1],tableNmae:[t_order_2]
			// logicTable:[t_order],dataSource:[d2],tableNmae:[t_order_1]
			// logicTable:[t_order],dataSource:[d2],tableNmae:[t_order_2]
			System.out.println(formatLine);
		});
	}
}
```

### (9). 重写EurekaAutoServiceRegistration伪代码
```
// 1. 自定义:EurekaAutoServiceRegistrationProxy
public class EurekaAutoServiceRegistrationProxy extends EurekaAutoServiceRegistration {

	public EurekaAutoServiceRegistrationProxy(ApplicationContext context, EurekaServiceRegistry serviceRegistry,
			EurekaRegistration registration) {
		super(context, serviceRegistry, registration);
	}

	@Override
	public void start() {
		super.start();
		// 实现你的逻辑
	}

	@Override
	public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
		return WebServerInitializedEvent.class.isAssignableFrom(eventType)
				|| ContextClosedEvent.class.isAssignableFrom(eventType);
	 // 增加新的事件
	}

	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof WebServerInitializedEvent) {
			onApplicationEvent((WebServerInitializedEvent) event);
		} else if (event instanceof ContextClosedEvent) {
			onApplicationEvent((ContextClosedEvent) event);
		}
		// 实现你的逻辑
	}
}
// end EurekaAutoServiceRegistrationProxy


// 2. 感觉Spring针对EurekaAutoServiceRegistration没有Hook解决方案.
//    我们都知道,不论是XML/Annotation/JavaConfig都只是业务模型的表现形式.
//    Spring在实例化Bean之前是先收集Bean信息(BeanDefinition)
//    所以,可以通过:BeanFactoryPostProcessor,把Bean(BeanDefinition)信息给删了,再重新添加新的Bean信息(BeanDefinition).

class TestBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		if(beanFactory instanceof DefaultListableBeanFactory) {
			DefaultListableBeanFactory defaultListableBeanFactory = (DefaultListableBeanFactory)beanFactory;
			//1.定义要删除的Bean名称
			String beanName = "eurekaAutoServiceRegistration";
			boolean res = beanFactory.containsBeanDefinition(beanName);
			if(res) {
				// 2. 删除Bean定义信息,在这时候所有的Bean都还没有实例化的
				defaultListableBeanFactory.removeBeanDefinition(beanName);
				
				// 3. 重新注册新的Bean信息
				BeanDefinition eurekaAutoServiceRegistrationProxyDefinition = new RootBeanDefinition(
						EurekaAutoServiceRegistrationProxy.class);
				((DefaultListableBeanFactory) beanFactory).registerBeanDefinition(beanName, eurekaAutoServiceRegistrationProxyDefinition);
			}
			res = beanFactory.containsBeanDefinition(beanName);
			System.out.println(res);
		}
	}
}// end TestBeanFactoryPostProcessor


// 3. 向Spring注册TestBeanFactoryPostProcessor
//    典型的偷梁换柱法.哈哈哈...
@Bean
public TestBeanFactoryPostProcessor testBeanFactoryPostProcessor() {
	return new TestBeanFactoryPostProcessor();
}
```