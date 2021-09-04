---
layout: post
title: 'Liquibase源码之CustomChange(四)' 
date: 2021-09-03
author: 李新
tags:  Liquibase
---

### (1). 概述
在前面,剖析了Liquibase解析XML,并执行SQL,那么,我们能否扩展Change呢?Liquibase提供了扩展类:CustomChange和XML标签.

### (2). 先看下CustomChange接口信息
> 从接口上能看出它有两个实现接口,分别是:CustomTaskChange/CustomSqlChange.   

!["CustomChange接口信息"](/assets/liquibase/imgs/Liquibase-CustomChange.jpg) 

### (3). ChangeSet xml案例
```
<changeSet id="56" author="nvoxland">
	<customChange class="liquibase.change.custom.ExampleCustomSqlChange">
		<param name="tableName" value="person"/>
		<param name="columnName" value="employer_id"/>
		<param name="newValue" value="3"/>
	</customChange>
</changeSet>
```
### (4). ExampleCustomSqlChange
```
package liquibase.change.custom;

import liquibase.database.Database;
import liquibase.exception.RollbackImpossibleException;
import liquibase.exception.SetupException;
import liquibase.exception.ValidationErrors;
import liquibase.resource.ResourceAccessor;
import liquibase.statement.SqlStatement;
import liquibase.statement.core.RawSqlStatement;
import liquibase.structure.core.Column;
import liquibase.structure.core.Table;

public class ExampleCustomSqlChange 
       implements 
	   // 生成正向SQL语句的实现
	   CustomSqlChange, 
	   // 生成逆向Rollback语句的实现
	   CustomSqlRollback {
		   
    private String schemaName;
    private String tableName;
    private String columnName;
    private String newValue;

    @SuppressWarnings("unused")
    private ResourceAccessor resourceAccessor;


    public String getSchemaName() {
        return schemaName;
    }

    public void setSchemaName(String schemaName) {
        this.schemaName = schemaName;
    }

    public String getTableName() {
        return tableName;
    }

    public void setTableName(String tableName) {
        this.tableName = tableName;
    }

    public String getColumnName() {
        return columnName;
    }

    public void setColumnName(String columnName) {
        this.columnName = columnName;
    }

    public String getNewValue() {
        return newValue;
    }

    public void setNewValue(String newValue) {
        this.newValue = newValue;
    }

    // *************************************************************************************
	// 生成正向SQL语句
	// *************************************************************************************
    @Override
    public SqlStatement[] generateStatements(Database database) {
        return new SqlStatement[]{
                new RawSqlStatement("UPDATE "+database.escapeObjectName(null, schemaName, tableName, Table.class)
                        +" SET "+database.escapeObjectName(columnName, Column.class)+" = "+newValue)
        };
    }

	// *************************************************************************************
	// 生成逆向SQL语句
	// *************************************************************************************
    @Override
    public SqlStatement[] generateRollbackStatements(Database database) throws RollbackImpossibleException {
        return new SqlStatement[]{
                new RawSqlStatement("UPDATE "+database.escapeObjectName(null, schemaName, tableName, Table.class)
                        +" SET "+database.escapeObjectName(columnName, Column.class)+" = NULL")
        };
    }

    @Override
    public String getConfirmationMessage() {
        return "Custom class updated "+tableName+"."+columnName;
    }


    @Override
    public void setUp() throws SetupException {
    }

    @Override
    public void setFileOpener(ResourceAccessor resourceAccessor) {
        this.resourceAccessor = resourceAccessor;
    }

    @Override
    public ValidationErrors validate(Database database) {
        return new ValidationErrors();
    }
}
```
### (5). 总结
Liquibase考虑到了扩展问题,早就预留了相应的标签和接口给我们使用,不过,是不允许我们自定义标签的,否则,在DTD校验时就会报错.     