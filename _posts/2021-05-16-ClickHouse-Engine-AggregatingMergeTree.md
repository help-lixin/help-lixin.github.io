---
layout: post
title: 'ClickHouse AggregatingMergeTree表引擎(八)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). AggregatingMergeTree介绍
> AggregatingMergeTree是通过预先定义的聚合函数计算数据并通过二进制的格式存入表内.   

### (2). 测试
```
# 1. 创建库
CREATE DATABASE test1;

USE test1;

# 2. 创建表
CREATE TABLE emp
(
    emp_id     UInt16 COMMENT '员工id',
    name       String COMMENT '员工姓名',
    work_place String COMMENT '工作地点',
    age        UInt8 COMMENT '员工年龄',
    depart     String COMMENT '部门',
    salary     AggregateFunction(sum, Decimal32(2)) COMMENT '工资'
) ENGINE = AggregatingMergeTree() 
  PARTITION BY work_place
  ORDER BY (emp_id, name) 
  PRIMARY KEY emp_id;

# 3. 插入数据(需要注意,在写入数据时,要调用:xxxxState(xxx),而在query时,也要调用相应的:xxxMerge(xxx)  )
INSERT INTO TABLE emp SELECT 1,'tom','上海',25,'信息部',sumState(toDecimal32(10000,2));
INSERT INTO TABLE emp SELECT 1,'tom','上海',25,'信息部',sumState(toDecimal32(20000,2));
INSERT INTO TABLE emp SELECT 1,'tom_2','上海',25,'信息部',sumState(toDecimal32(500,2));

# 4. 检索
SELECT emp_id,name,sumMerge(salary) FROM emp GROUP BY emp_id,name;
┌─emp_id─┬─name──┬─sumMerge(salary)─┐
│      1 │ tom   │         30000.00 │
│      1 │ tom_2 │           500.00 │
└────────┴───────┴──────────────────┘
```
### (3). 总结
> 通过测试:AggregatingMergeTree是根据主键进行汇总的,AggregatingMergeTree适合需要进行汇总的表.   
> 业务场景:每秒QPS,报表...