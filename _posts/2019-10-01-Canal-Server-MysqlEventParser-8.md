---
layout: post
title: 'Canal Server源码之五(MysqlEventParser-8)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述
> 在["Canal Server源码之五(MysqlEventParser-6)"](https://www.lixin.help/2019/10/01/Canal-Server-MysqlEventParser-6.html)剖析了:MysqlMultiStageCoprocessor的大概过程,这一篇主要剖析了:MysqlMultiStageCoprocessor$SimpleParserStage

### (2). 准备工作
> 以一条INSERT语句进行跟踪 

```
INSERT INTO `test2`.`user` (`id`, `nick`, `phone`, `password`, `email`) VALUES ('6', '赵六', '8888888', '88888888', '123@126.com');
```

### (3). SimpleParserStage.onEvent
```
public void onEvent(MessageEvent event, long sequence, boolean endOfBatch) throws Exception {
    try {

        LogEvent logEvent = event.getEvent();
        if (logEvent == null) {
            LogBuffer buffer = event.getBuffer();
            logEvent = decoder.decode(buffer, context);
            event.setEvent(logEvent);
        }

        // INSERT eventType = 30
        int eventType = logEvent.getHeader().getType();
        TableMeta tableMeta = null;
        boolean needDmlParse = false;
        switch (eventType) {
            case LogEvent.WRITE_ROWS_EVENT_V1:
            case LogEvent.WRITE_ROWS_EVENT:
                // **********************************************************
                // 4. 调用:LogEventConvert.parseRowsEventForTableMeta获取表的元数据信息.
                // **********************************************************
                tableMeta = logEventConvert.parseRowsEventForTableMeta((WriteRowsLogEvent) logEvent);
                needDmlParse = true;
                break;
            case LogEvent.UPDATE_ROWS_EVENT_V1:
            case LogEvent.PARTIAL_UPDATE_ROWS_EVENT:
            case LogEvent.UPDATE_ROWS_EVENT:
                // ....
                needDmlParse = true;
                break;
            case LogEvent.DELETE_ROWS_EVENT_V1:
            case LogEvent.DELETE_ROWS_EVENT:
                // ....
                needDmlParse = true;
                break;
            case LogEvent.ROWS_QUERY_LOG_EVENT:
                needDmlParse = true;
                break;
            default:
                CanalEntry.Entry entry = logEventConvert.parse(event.getEvent(), false);
                event.setEntry(entry);
        }

        // 记录一下DML的表结构
        event.setNeedDmlParse(needDmlParse);
        event.setTable(tableMeta);
    } catch (Throwable e) {
        exception = new CanalParseException(e);
        throw exception;
    }
} //end onEvent
```
### (4). LogEventConvert.parseRowsEventForTableMeta
```
public TableMeta parseRowsEventForTableMeta(RowsLogEvent event) {
    // 从协议里获得Table信息
    TableMapLogEvent table = event.getTable();
    if (table == null) {
        // tableId对应的记录不存在
        throw new TableIdNotFoundException("not found tableId:" + event.getTableId());
    }

    // 判断:dbname是否为:test,并且tablename是否为:heartbeat
    boolean isHeartBeat = isAliSQLHeartBeat(table.getDbName(), table.getTableName());
    // 判断:dbname是否为:mysql,并且tablname是否为:ha_health_check
    boolean isRDSHeartBeat = tableMetaCache.isOnRDS() && isRDSHeartBeat(table.getDbName(), table.getTableName());

    // 库名+表名(test2.user)
    String fullname = table.getDbName() + "." + table.getTableName();

    // 定义的filter
    // check name filter
    if (nameFilter != null && !nameFilter.filter(fullname)) {
        return null;
    }
    if (nameBlackFilter != null && nameBlackFilter.filter(fullname)) {
        return null;
    }

    // if (isHeartBeat || isRDSHeartBeat) {
    // // 忽略rds模式的mysql.ha_health_check心跳数据
    // return null;
    // }
    TableMeta tableMeta = null;
    // 是否为RDS heart beat请求
    if (isRDSHeartBeat) {  //false
        // 处理rds模式的mysql.ha_health_check心跳数据
        // 主要RDS的心跳表基本无权限,需要mock一个tableMeta
        FieldMeta idMeta = new FieldMeta("id", "bigint(20)", true, false, "0");
        FieldMeta typeMeta = new FieldMeta("type", "char(1)", false, true, "0");
        tableMeta = new TableMeta(table.getDbName(), table.getTableName(), Arrays.asList(idMeta, typeMeta));
    } else if (isHeartBeat) { //false
         // 是否为HeartBeast请求
        // 处理alisql模式的test.heartbeat心跳数据
        // 心跳表基本无权限,需要mock一个tableMeta
        FieldMeta idMeta = new FieldMeta("id", "smallint(6)", false, true, null);
        FieldMeta typeMeta = new FieldMeta("ts", "int(11)", true, false, null);
        tableMeta = new TableMeta(table.getDbName(), table.getTableName(), Arrays.asList(idMeta, typeMeta));
    }

    // 读取协议头里的信息,并填充到业务模型:EntryPosition
    // logHeader.getWhen() 为Event产生的时间,只能精确到秒.
    EntryPosition position = createPosition(event.getHeader());
    if (tableMetaCache != null && tableMeta == null) {// 入错存在table meta
        // ************************************************************************
        // 5. 从缓存中获取表的元数据信息(TableMetaCache.getTableMeta)
        // ************************************************************************
        tableMeta = getTableMeta(table.getDbName(), table.getTableName(), true, position);
        if (tableMeta == null) {
            if (!filterTableError) {
                throw new CanalParseException("not found [" + fullname + "] in db , pls check!");
            }
        }
    }

    return tableMeta;
} // end parseRowsEventForTableMeta


private boolean isAliSQLHeartBeat(String schema, String table) {
    return "test".equalsIgnoreCase(schema) && "heartbeat".equalsIgnoreCase(table);
} //end isAliSQLHeartBeat

private boolean isRDSHeartBeat(String schema, String table) {
    return "mysql".equalsIgnoreCase(schema) && "ha_health_check".equalsIgnoreCase(table);
}// end isRDSHeartBeat


private EntryPosition createPosition(LogHeader logHeader) {
    return new EntryPosition(logHeader.getLogFileName(), logHeader.getLogPos() - logHeader.getEventLen(), // startPos
        logHeader.getWhen() * 1000L,
        logHeader.getServerId()); // 记录到秒
} //end createPosition
```
### (5). TableMetaCache.getTableMeta
```
private TableMeta getTableMeta(
    // test2
    String dbName, 
    // user
    String tbName, 
    // true
    boolean useCache, 
    // EntryPosition[included=false,journalName=mysql-bin.000023,position=659,serverId=1,timestamp=1606551260000]
    EntryPosition position) {
    try {
        // *************************************************************************
        // 首先从缓存中获得表的元数据信息,获取不到的情况下再从MySqlConnection中获取元据信息
        // 6. TableMetaCache.getTableMeta
        // *************************************************************************
        return tableMetaCache.getTableMeta(dbName, tbName, useCache, position);
    } catch (Throwable e) {
        String message = ExceptionUtils.getRootCauseMessage(e);
        if (filterTableError) {
            if (StringUtils.contains(message, "errorNumber=1146") && StringUtils.contains(message, "doesn't exist")) {
                return null;
            } else if (StringUtils.contains(message, "errorNumber=1142")
                        && StringUtils.contains(message, "command denied")) {
                return null;
            }
        }
        throw new CanalParseException(e);
    } // end catch
}//end getTableMeta
```
### (6). TableMetaCache.getTableMeta
```
public synchronized TableMeta getTableMeta(String schema, String table, boolean useCache, EntryPosition position) {
    TableMeta tableMeta = null;
    if (tableMetaTSDB != null) {  // true
        //  ******************************************************************
        // 7.MemoryTableMeta.find
        //  ******************************************************************
        tableMeta = tableMetaTSDB.find(schema, table);
        if (tableMeta == null) { // false
            // 因为条件变化，可能第一次的tableMeta没取到，需要从db获取一次，并记录到snapshot中
            String fullName = getFullName(schema, table);
            ResultSetPacket packet = null;
            String createDDL = null;
            try {
                try {
                    packet = connection.query("show create table " + fullName);
                } catch (Exception e) {
                    // 尝试做一次retry操作
                    connection.reconnect();
                    packet = connection.query("show create table " + fullName);
                }
                if (packet.getFieldValues().size() > 0) {
                    createDDL = packet.getFieldValues().get(1);
                }
                // 强制覆盖掉内存值
                tableMetaTSDB.apply(position, schema, createDDL, "first");
                tableMeta = tableMetaTSDB.find(schema, table);
            } catch (IOException e) {
                throw new CanalParseException("fetch failed by table meta:" + fullName, e);
            }
        }
        
        return tableMeta;
    } else {
        if (!useCache) {
            tableMetaDB.invalidate(getFullName(schema, table));
        }
        return tableMetaDB.getUnchecked(getFullName(schema, table));
    }
}
```
### (7). MemoryTableMeta.find
```
public TableMeta find(String schema, String table) {
    List<String> keys = Arrays.asList(schema, table);
    // 第一次请tableMeta为空
    TableMeta tableMeta = tableMetas.get(keys);
    if (tableMeta == null) {
        synchronized (this) {
            tableMeta = tableMetas.get(keys);
            if (tableMeta == null) {

                // 获取数据库("test2")
                Schema schemaRep = repository.findSchema(schema);
                if (schemaRep == null) {
                    return null;
                }

                // 获得表信息("user")
                SchemaObject data = schemaRep.findTable(table);
                if (data == null) {
                    return null;
                }
                // 获得表结构语句
                // CREATE TABLE `user` (
	            //  `id` int(11) NOT NULL AUTO_INCREMENT,
	            //  `nick` varchar(20) DEFAULT NULL,
	            //  `phone`  varchar(20) DEFAULT NULL,
	            //  `password` varchar(30) DEFAULT NULL,
	            //  `email`  varchar(30) DEFAULT NULL,
	            //  `account` varchar(30) DEFAULT NULL,
	            //  PRIMARY KEY (`id`)
                // ) ENGINE = InnoDB CHARSET = utf8
                SQLStatement statement = data.getStatement();
                if (statement == null) {
                    return null;
                }

                if (statement instanceof SQLCreateTableStatement) { //true
                    // ***********************************************************
                    // 8. MemoryTableMeta.parse
                    // 解析SQL语句
                    // ***********************************************************
                    tableMeta = parse((SQLCreateTableStatement) statement);
                }
                if (tableMeta != null) {
                    if (table != null) {
                        tableMeta.setTable(table);
                    }
                    if (schema != null) {
                        tableMeta.setSchema(schema);
                    }

                    tableMetas.put(keys, tableMeta);
                }
            }
        }
    }
    return tableMeta;
}// end find
```
### (8). MemoryTableMeta.parse
```
private TableMeta parse(SQLCreateTableStatement statement) {
    // size = 7
    int size = statement.getTableElementList().size();
    if (size > 0) {
        TableMeta tableMeta = new TableMeta();
        for (int i = 0; i < size; ++i) {
            // element 
            // `id` int(11) NOT NULL AUTO_INCREMENT
            SQLTableElement element = statement.getTableElementList().get(i);
            // 针对MySQL中每一个列进行解析,转换成业务模型,处理过程代码忽略.
            processTableElement(element, tableMeta);
        }
        // 返回表的元数据信息.
        return tableMeta;
    }
    return null;
}//end parse
```

### (7). UML时序图
!["MysqlMultiStageCoprocessor$SimpleParserStage时序图"](/assets/canal/imgs/SimpleParserStage-seq.jpg)

### (7). 总结
> 1. 读取协议头内容,判断是否为(INSERT/UPDATE/DELETE),在这里的演示是:<font color='red'>INSERT</font>.     
> 2. 调用:LogEventConvert.parseRowsEventForTableMeta方法,解析表的元数据信息.首先,会从Cache里获取表的元数据,获取不到的情况下,再通过:SchemaRepository去获取表结构语句(CREATE TABLE...),然后,把结构语句解析成:TableMeta对象. 
> 3. 设置MessageEvent.setTable(TableMeta).    
> 4. <font color='red'>设置MessageEvent.setNeedDmlParse(true),这一步的设置,直接影响下一个业务(DmlParserStage)的处理.</font>    
> 下一节,剖析:DmlParserStage内部逻辑.   