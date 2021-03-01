---
layout: post
title: 'Liquibase Hello World(二)'
date: 2021-02-27
author: 李新
tags: Liquibase
---

### (1). 概述
> 在这里,先搭个Hello World.步骤如下:    
> 1. 编写xml文件(Liquibase内部会根据XML转换成SQL文件).  
> 2. 建库.  
> 3. 让Liquibase在刚建的库里,根据XML创建表.    
> 4. 注意事项:Liquibase内部没有解决线程安全问题,所以,不能并发执行:liquibase.update(contexts).  

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
> 肯定又有人要问,能不能在所有的表创建完成之后,再向Eureka进行注册了呢?  
> 带着这些问题,请看后面的内容.   