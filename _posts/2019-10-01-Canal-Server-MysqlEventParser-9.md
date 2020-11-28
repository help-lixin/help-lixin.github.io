---
layout: post
title: 'Canal Server源码之五(MysqlEventParser-9)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 概述
> 在["Canal Server源码之五(MysqlEventParser-6)"](https://www.lixin.help/2019/10/01/Canal-Server-MysqlEventParser-6.html)剖析了:MysqlMultiStageCoprocessor的大概过程,这一篇主要剖析了:MysqlMultiStageCoprocessor$DmlParserStage

### (2). 准备工作
> 以一条INSERT语句进行跟踪 

```
INSERT INTO `test2`.`user` (`id`, `nick`, `phone`, `password`, `email`) VALUES ('6', '赵六', '8888888', '88888888', '123@126.com');
```

### (3). DmlParserStage.onEvent
```
public void onEvent(MessageEvent event) throws Exception {
    try {

        // 只有前一阶段(SimpleParserStage),设置为:true,这个阶段才会生效.
        if (event.isNeedDmlParse()) { 
            // 读取协议头里的事件类型
            // eventType = 30
            int eventType = event.getEvent().getHeader().getType();
            CanalEntry.Entry entry = null;
            switch (eventType) {
                case LogEvent.ROWS_QUERY_LOG_EVENT:
                    // ... ...
                    break;
                default:
                    // *************************************************************
                    // 4. 调用:LogEventConvert.parseRowsEvent 解析数据
                    // *************************************************************
                    entry = logEventConvert.parseRowsEvent((RowsLogEvent) event.getEvent(), event.getTable());
            }

            event.setEntry(entry);
        }
    } catch (Throwable e) {
        exception = new CanalParseException(e);
        throw exception;
    }
} // end onEvent
```
### (4). LogEventConvert.parseRowsEvent
```
public Entry parseRowsEvent(RowsLogEvent event, TableMeta tableMeta) {
    if (filterRows) { // flase
        return null;
    }

    try {
        // false
        if (tableMeta == null) { // 如果没有外部指定
            tableMeta = parseRowsEventForTableMeta(event);
        }

        if (tableMeta == null) { // false
            // 拿不到表结构,执行忽略
            return null;
        }


        // 获得事件的类型,我这里是INSERT
        EventType eventType = null;
        int type = event.getHeader().getType();
        if (LogEvent.WRITE_ROWS_EVENT_V1 == type || LogEvent.WRITE_ROWS_EVENT == type) {
            // 我这里以INSERT为案例.
            eventType = EventType.INSERT;
        } else if (LogEvent.UPDATE_ROWS_EVENT_V1 == type || LogEvent.UPDATE_ROWS_EVENT == type
                    || LogEvent.PARTIAL_UPDATE_ROWS_EVENT == type) {
            eventType = EventType.UPDATE;
        } else if (LogEvent.DELETE_ROWS_EVENT_V1 == type || LogEvent.DELETE_ROWS_EVENT == type) {
            eventType = EventType.DELETE;
        } else {
            throw new CanalParseException("unsupport event type :" + event.getHeader().getType());
        }

        // 构建者模式
        RowChange.Builder rowChangeBuider = RowChange.newBuilder();
        // tableId = 228
        rowChangeBuider.setTableId(event.getTableId());
        // false
        rowChangeBuider.setIsDdl(false);

        // eventType = INSERT
        rowChangeBuider.setEventType(eventType);
        
        // 从event中获取所有行的信息.
        RowsLogBuffer buffer = event.getRowsBuf(charset.name());

        // 获得所有的列:{0, 1, 2, 3, 4, 5, 6, 7}
        BitSet columns = event.getColumns();
        // 获得所有更新的列:{0, 1, 2, 3, 4, 5, 6, 7}
        BitSet changeColumns = event.getChangeColumns();

        boolean tableError = false;
        int rowsCount = 0;

        // columns = {0, 1, 2, 3, 4, 5, 6, 7}
        // 循环读取每一列的数据
        while (buffer.nextOneRow(columns, false)) {
            // 处理row记录
            RowData.Builder rowDataBuilder = RowData.newBuilder();
            if (EventType.INSERT == eventType) {
                // *****************************************************************
                // insert的记录放在before字段中
                // 5. parseOneRow 解析单行数据
                // *****************************************************************
                tableError |= parseOneRow(rowDataBuilder, event, buffer, columns, true, tableMeta);
            } else if (EventType.DELETE == eventType) {
                // delete的记录放在before字段中
                tableError |= parseOneRow(rowDataBuilder, event, buffer, columns, false, tableMeta);
            } else {
                // update需要处理before/after
                tableError |= parseOneRow(rowDataBuilder, event, buffer, columns, false, tableMeta);
                if (!buffer.nextOneRow(changeColumns, true)) {
                    rowChangeBuider.addRowDatas(rowDataBuilder.build());
                    break;
                }

                tableError |= parseOneRow(rowDataBuilder, event, buffer, changeColumns, true, tableMeta);
            }

            rowsCount++;
            rowChangeBuider.addRowDatas(rowDataBuilder.build());
        }

        // ************************************************************************
        // rowChangeBuider解析后的内容:
        // ************************************************************************
        // tableId: 228
        // eventType: INSERT
        // isDdl: false
        // rowDatas {
        // afterColumns {
        //    index: 0
        //    sqlType: 4
        //    name: "id"
        //    isKey: true
        //    updated: true
        //    isNull: false
        //    value: "5"
        //    mysqlType: "int(11)"
        //  }
        // afterColumns {
        //    index: 1
        //    sqlType: 12
        //    name: "nick"
        //    isKey: false
        //    updated: true
        //    isNull: false
        //    value: "\350\265\265\345\205\255"
        //    mysqlType: "varchar(20)"
        //  }
        //  afterColumns {
        //    index: 2
        //    sqlType: 12
        //    name: "phone"
        //   isKey: false
        //    updated: true
        //    isNull: false
        //    value: "8888888"
        //    mysqlType: "varchar(20)"
        //  }
        //  afterColumns {
        //    index: 3
        //    sqlType: 12
        //    name: "password"
        //    isKey: false
        //    updated: true
        //    isNull: false
        //    value: "88888888"
        //    mysqlType: "varchar(30)"
        //  }
        //  afterColumns {
        //    index: 4
        //    sqlType: 12
        //    name: "email"
        //    isKey: false
        //    updated: true
        //    isNull: false
        //    value: "123@126.com"
        //   mysqlType: "varchar(30)"
        //  }
        //  afterColumns {
        //    index: 5
        //    sqlType: 12
        //    name: "account"
        //    isKey: false
        //    updated: true
        //    isNull: true
        //    mysqlType: "varchar(30)"
        //  }
        //}

        
        TableMapLogEvent table = event.getTable();

        // createHeader的业务:
        // 构建Header,内容如下(logfileName/position/schema...):
        // version: 1
        // logfileName: "mysql-bin.000026"
        // logfileOffset: 353
        // serverId: 1
        // serverenCode: "UTF-8"
        // executeTime: 1606553694000
        // sourceType: MYSQL
        // schemaName: "test2"
        // tableName: "user"
        // eventLength: 76
        // eventType: INSERT
        // props {
        //   key: "rowsCount"
        //   value: "1"
        // }

        Header header = createHeader(event.getHeader(),
            table.getDbName(),
            table.getTableName(),
            eventType,
            rowsCount);

        // 对rowChangeBuider内容进行build,返回:RowChange
        RowChange rowChange = rowChangeBuider.build();

        // tableError解析rows是查看是否有错误
        if (tableError) {
            Entry entry = createEntry(header, EntryType.ROWDATA, ByteString.EMPTY);
            logger.warn("table parser error : {}storeValue: {}", entry.toString(), rowChange.toString());
            return null;
        } else { // true
            // 
            Entry entry = createEntry(header, EntryType.ROWDATA, rowChange.toByteString());
            return entry;
        }
    } catch (Exception e) {
        throw new CanalParseException("parse row data failed.", e);
    }
} // end parseRowsEvent
```
### (5). parseOneRow
```
private boolean parseOneRow(
    RowData.Builder rowDataBuilder, 
    RowsLogEvent event, 
    RowsLogBuffer buffer, 
    BitSet cols,
    boolean isAfter, 
    TableMeta tableMeta) throws UnsupportedEncodingException {
    //  columnCnt = 6   
    int columnCnt = event.getTable().getColumnCnt();


    // [ColumnInfo [type=3, meta=0, name=null, unsigned=false, pk=false, set_enum_values=null, charset=0, geoType=0, nullable=false]
    //  ColumnInfo [type=15, meta=60, name=null, unsigned=false, pk=false, set_enum_values=null, charset=0, geoType=0, nullable=true]
    // ColumnInfo [type=15, meta=60, name=null, unsigned=false, pk=false, set_enum_values=null, charset=0, geoType=0, nullable=true]
    // ColumnInfo [type=15, meta=90, name=null, unsigned=false, pk=false, set_enum_values=null, charset=0, geoType=0, nullable=true]
    // ColumnInfo [type=15, meta=90, name=null, unsigned=false, pk=false, set_enum_values=null, charset=0, geoType=0, nullable=true]
    // ColumnInfo [type=15, meta=90, name=null, unsigned=false, pk=false, set_enum_values=null, charset=0, geoType=0, nullable=true]
    
    ColumnInfo[] columnInfo = event.getTable().getColumnInfo();


    // mysql8.0针对set @@global.binlog_row_metadata='FULL' 可以记录部分的metadata信息
    boolean existOptionalMetaData = event.getTable().isExistOptionalMetaData();
    boolean tableError = false;
    // check table fileds count，只能处理加字段
    boolean existRDSNoPrimaryKey = false;
    //获取字段过滤条件
    List<String> fieldList = null;
    List<String> blackFieldList = null;
    
    if (tableMeta != null) {
        // null
        fieldList = fieldFilterMap.get(tableMeta.getFullName().toUpperCase());
        // null
        blackFieldList = fieldBlackFilterMap.get(tableMeta.getFullName().toUpperCase());
    }
    
    if (tableMeta != null && columnInfo.length > tableMeta.getFields().size()) { //false
        if (tableMetaCache.isOnRDS()) {
            // 特殊处理下RDS的场景
            List<FieldMeta> primaryKeys = tableMeta.getPrimaryFields();
            if (primaryKeys == null || primaryKeys.isEmpty()) {
                if (columnInfo.length == tableMeta.getFields().size() + 1
                    && columnInfo[columnInfo.length - 1].type == LogEvent.MYSQL_TYPE_LONGLONG) {
                    existRDSNoPrimaryKey = true;
                }
            }
        }

        EntryPosition position = createPosition(event.getHeader());
        if (!existRDSNoPrimaryKey) {
            // online ddl增加字段操作步骤：
            // 1. 新增一张临时表，将需要做ddl表的数据全量导入
            // 2. 在老表上建立I/U/D的trigger，增量的将数据插入到临时表
            // 3. 锁住应用请求，将临时表rename为老表的名字，完成增加字段的操作
            // 尝试做一次reload，可能因为ddl没有正确解析，或者使用了类似online ddl的操作
            // 因为online ddl没有对应表名的alter语法，所以不会有clear cache的操作
            tableMeta = getTableMeta(event.getTable().getDbName(), event.getTable().getTableName(), false, position);// 强制重新获取一次
            if (tableMeta == null) {
                tableError = true;
                if (!filterTableError) {
                    throw new CanalParseException("not found [" + event.getTable().getDbName() + "."
                                                    + event.getTable().getTableName() + "] in db , pls check!");
                }
            }

            // 在做一次判断
            if (tableMeta != null && columnInfo.length > tableMeta.getFields().size()) {
                tableError = true;
                if (!filterTableError) {
                    throw new CanalParseException("column size is not match for table:" + tableMeta.getFullName()
                                                    + "," + columnInfo.length + " vs " + tableMeta.getFields().size());
                }
            }
            // } else {
            // logger.warn("[" + event.getTable().getDbName() + "." +
            // event.getTable().getTableName()
            // + "] is no primary key , skip alibaba_rds_row_id column");
        }
    } //end if

    for (int i = 0; i < columnCnt; i++) {
        ColumnInfo info = columnInfo[i];
        // mysql 5.6开始支持nolob/mininal类型,并不一定记录所有的列,需要进行判断
        if (!cols.get(i)) {
            continue;
        }

        // rds日志不解析
        if (existRDSNoPrimaryKey && i == columnCnt - 1 && info.type == LogEvent.MYSQL_TYPE_LONGLONG) { // false
            // 不解析最后一列
            String rdsRowIdColumnName = "#alibaba_rds_row_id#";
            buffer.nextValue(rdsRowIdColumnName, i, info.type, info.meta, false);
            Column.Builder columnBuilder = Column.newBuilder();
            columnBuilder.setName(rdsRowIdColumnName);
            columnBuilder.setIsKey(true);
            columnBuilder.setMysqlType("bigint");
            columnBuilder.setIndex(i);
            columnBuilder.setIsNull(false);
            Serializable value = buffer.getValue();
            columnBuilder.setValue(value.toString());
            columnBuilder.setSqlType(Types.BIGINT);
            columnBuilder.setUpdated(false);

            if (needField(fieldList, blackFieldList, columnBuilder.getName())) {
                if (isAfter) {
                    rowDataBuilder.addAfterColumns(columnBuilder.build());
                } else {
                    rowDataBuilder.addBeforeColumns(columnBuilder.build());
                }
            }
            continue;
        }

        FieldMeta fieldMeta = null;
        if (tableMeta != null && !tableError) { // true
            // 处理file meta
            // 从缓存中获得元业务据信息
            fieldMeta = tableMeta.getFields().get(i);
        }

        if (fieldMeta != null && existOptionalMetaData && tableMetaCache.isOnTSDB()) { // false
            // check column info
            boolean check = StringUtils.equalsIgnoreCase(fieldMeta.getColumnName(), info.name);
            check &= (fieldMeta.isUnsigned() == info.unsigned);
            check &= (fieldMeta.isNullable() == info.nullable);

            if (!check) {
                throw new CanalParseException("MySQL8.0 unmatch column metadata & pls submit issue , table : "
                                                + tableMeta.getFullName() + ", db fieldMeta : "
                                                + fieldMeta.toString() + " , binlog fieldMeta : " + info.toString()
                                                + " , on : " + event.getHeader().getLogFileName() + ":"
                                                + (event.getHeader().getLogPos() - event.getHeader().getEventLen()));
            }
        }

        Column.Builder columnBuilder = Column.newBuilder();
        if (fieldMeta != null) {
            // 设置列的名称(id)
            columnBuilder.setName(fieldMeta.getColumnName());
            // 是否主键
            columnBuilder.setIsKey(fieldMeta.isKey());

            // 增加mysql type类型,issue 73
            // mysqlType: "int(11)"
            columnBuilder.setMysqlType(fieldMeta.getColumnType());
        } else if (existOptionalMetaData) {
            columnBuilder.setName(info.name);
            columnBuilder.setIsKey(info.pk);
            // mysql8.0里没有mysql type类型
            // columnBuilder.setMysqlType(fieldMeta.getColumnType());
        }
        // 设置索引
        columnBuilder.setIndex(i);
        columnBuilder.setIsNull(false);

        // fixed issue
        // https://github.com/alibaba/canal/issues/66，特殊处理binary/varbinary，不能做编码处理
        boolean isBinary = false;
        boolean isSingleBit = false;
        if (fieldMeta != null) { // true
            if (StringUtils.containsIgnoreCase(fieldMeta.getColumnType(), "VARBINARY")) { // false
                isBinary = true;
            } else if (StringUtils.containsIgnoreCase(fieldMeta.getColumnType(), "BINARY")) { // false
                isBinary = true;
            } else if (StringUtils.containsIgnoreCase(fieldMeta.getColumnType(), "TINYINT(1)")) { // false
                isSingleBit = true;
            }
        }

        buffer.nextValue(columnBuilder.getName(), i, info.type, info.meta, isBinary);
        int javaType = buffer.getJavaType();
        if (buffer.isNull()) {
            columnBuilder.setIsNull(true);
        } else {
            final Serializable value = buffer.getValue();
            // 处理各种类型
            switch (javaType) {
                case Types.INTEGER:
                case Types.TINYINT:
                case Types.SMALLINT:
                case Types.BIGINT:
                    // 处理unsigned类型
                    Number number = (Number) value;
                    boolean isUnsigned = (fieldMeta != null ? fieldMeta.isUnsigned() : (existOptionalMetaData ? info.unsigned : false));
                    if (isUnsigned && number.longValue() < 0) {
                        switch (buffer.getLength()) {
                            case 1: /* MYSQL_TYPE_TINY */
                                columnBuilder.setValue(String.valueOf(Integer.valueOf(TINYINT_MAX_VALUE
                                                                                        + number.intValue())));
                                javaType = Types.SMALLINT; // 往上加一个量级
                                break;

                            case 2: /* MYSQL_TYPE_SHORT */
                                columnBuilder.setValue(String.valueOf(Integer.valueOf(SMALLINT_MAX_VALUE
                                                                                        + number.intValue())));
                                javaType = Types.INTEGER; // 往上加一个量级
                                break;

                            case 3: /* MYSQL_TYPE_INT24 */
                                columnBuilder.setValue(String.valueOf(Integer.valueOf(MEDIUMINT_MAX_VALUE
                                                                                        + number.intValue())));
                                javaType = Types.INTEGER; // 往上加一个量级
                                break;

                            case 4: /* MYSQL_TYPE_LONG */
                                columnBuilder.setValue(String.valueOf(Long.valueOf(INTEGER_MAX_VALUE
                                                                                    + number.longValue())));
                                javaType = Types.BIGINT; // 往上加一个量级
                                break;

                            case 8: /* MYSQL_TYPE_LONGLONG */
                                columnBuilder.setValue(BIGINT_MAX_VALUE.add(BigInteger.valueOf(number.longValue()))
                                    .toString());
                                javaType = Types.DECIMAL; // 往上加一个量级，避免执行出错
                                break;
                        }
                    } else {
                        // 对象为number类型，直接valueof即可
                        columnBuilder.setValue(String.valueOf(value));
                    }

                    if (isSingleBit && javaType == Types.TINYINT) {
                        javaType = Types.BIT;
                    }
                    break;
                case Types.REAL: // float
                case Types.DOUBLE: // double
                    // 对象为number类型，直接valueof即可
                    columnBuilder.setValue(String.valueOf(value));
                    break;
                case Types.BIT:// bit
                    // 对象为number类型
                    columnBuilder.setValue(String.valueOf(value));
                    break;
                case Types.DECIMAL:
                    columnBuilder.setValue(((BigDecimal) value).toPlainString());
                    break;
                case Types.TIMESTAMP:
                    // 修复时间边界值
                    // String v = value.toString();
                    // v = v.substring(0, v.length() - 2);
                    // columnBuilder.setValue(v);
                    // break;
                case Types.TIME:
                case Types.DATE:
                    // 需要处理year
                    columnBuilder.setValue(value.toString());
                    break;
                case Types.BINARY:
                case Types.VARBINARY:
                case Types.LONGVARBINARY:
                    // fixed text encoding
                    // https://github.com/AlibabaTech/canal/issues/18
                    // mysql binlog中blob/text都处理为blob类型，需要反查table
                    // meta，按编码解析text
                    if (fieldMeta != null && isText(fieldMeta.getColumnType())) {
                        columnBuilder.setValue(new String((byte[]) value, charset));
                        javaType = Types.CLOB;
                    } else {
                        // byte数组，直接使用iso-8859-1保留对应编码，浪费内存
                        columnBuilder.setValue(new String((byte[]) value, ISO_8859_1));
                        // columnBuilder.setValueBytes(ByteString.copyFrom((byte[])
                        // value));
                        javaType = Types.BLOB;
                    }
                    break;
                case Types.CHAR:
                case Types.VARCHAR:
                    columnBuilder.setValue(value.toString());
                    break;
                default:
                    columnBuilder.setValue(value.toString());
            }
        }

        columnBuilder.setSqlType(javaType);
        // 设置是否update的标记位
        columnBuilder.setUpdated(isAfter
                                    && isUpdate(rowDataBuilder.getBeforeColumnsList(),
                                        columnBuilder.getIsNull() ? null : columnBuilder.getValue(),
                                        i));
        if (needField(fieldList, blackFieldList, columnBuilder.getName())) {
            if (isAfter) {
                rowDataBuilder.addAfterColumns(columnBuilder.build());
            } else {
                rowDataBuilder.addBeforeColumns(columnBuilder.build());
            }
        }
    }
    return tableError;
}
```
### (6). UML时序图
!["DmlParserStage"](/assets/canal/imgs/DmlParserStage-seq.jpg)


### (7). 总结
> DmlParserStage阶段最主要是对报文的内容解码成:Entry对象.   
> 所以,自然这个阶段要配置多线程进行解析. 