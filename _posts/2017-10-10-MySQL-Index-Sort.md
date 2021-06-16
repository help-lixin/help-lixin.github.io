---
layout: post
title: 'MySQL 排序优化'
date: 2017-10-10
author: 李新
tags: MySQL
---

### (1). 前言
> ["本案例的SQL脚本来源于:https://github.com/help-lixin/test_db"](https://github.com/help-lixin/test_db)

### (2). 查看表结构
```
# 1. 查看表结构
#    存在主键联合索引(`emp_no`,`title`,`from_date`)
#    普通索引titles` (`title`)
#    外键约束titles_ibfk_1
mysql> SHOW CREATE TABLE titles\G
*************************** 1. row ***************************
       Table: titles
Create Table: CREATE TABLE `titles` (
  `emp_no` int(11) NOT NULL,
  `title` varchar(50) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date DEFAULT NULL,
  PRIMARY KEY (`emp_no`,`title`,`from_date`),
  KEY `titles` (`title`),
  CONSTRAINT `titles_ibfk_1` FOREIGN KEY (`emp_no`) REFERENCES `employees` (`emp_no`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

### (3). 排序案例
```
# ********************************************************************
# 尽量避免:Using filesort,让排序和查询都遵循最左原则.
# 通过这样的SQL代码,在Extra就能避免:Using filesort
# ********************************************************************
mysql> EXPLAIN SELECT *  FROM employees.titles WHERE emp_no = 10026 ORDER BY title,from_date\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 2
     filtered: 100.00
        Extra: Using where		
```


```
# 把排序字段顺序调整(不符合最左元则),立即出现: Using filesort
mysql> EXPLAIN SELECT *  FROM employees.titles WHERE emp_no = 10026 ORDER BY from_date,title\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 2
     filtered: 100.00
        Extra: Using where; Using filesort
```

```
# 命中索引(emp_no和title),并且,不会使用: Using filesort
# 虽然:ORDER BY from_date,title,实际上:MySQL优化器,忽略掉了:title的排序
mysql> \W
mysql> EXPLAIN SELECT *  FROM employees.titles WHERE emp_no = 10026 AND title='Engineer'  ORDER BY from_date,title\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: ref
possible_keys: PRIMARY,titles
          key: PRIMARY
      key_len: 156
          ref: const,const
         rows: 1
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

## ************************************************
##  MySQL直接忽略掉了,无用的排序
## ************************************************
Note (Code 1003): /* select#1 */ 
select 
    `employees`.`titles`.`emp_no` AS `emp_no`,
	`employees`.`titles`.`title` AS `title`,
	`employees`.`titles`.`from_date` AS `from_date`,
	`employees`.`titles`.`to_date` AS `to_date` 
from `employees`.`titles` 
where ((`employees`.`titles`.`emp_no` = 10026) and (`employees`.`titles`.`title` = 'Engineer')) 
order by `employees`.`titles`.`from_date`
```

```
# 符合最左元则法,所以,不会使用: Using filesort
# ORDER BY title,from_date desc
mysql> EXPLAIN SELECT *  FROM employees.titles WHERE emp_no = 10026 AND title='Engineer'  ORDER BY title,from_date desc\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: ref
possible_keys: PRIMARY,titles
          key: PRIMARY
      key_len: 156
          ref: const,const
         rows: 1
     filtered: 100.00
        Extra: Using where
```

```
# 语法IN,会造成:Using filesort
mysql> EXPLAIN SELECT *  FROM employees.titles WHERE emp_no IN( 10026,10025 ) ORDER BY title,from_date desc\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 3
     filtered: 100.00
        Extra: Using where; Using filesort
```

```
# 符合最左元则
mysql> EXPLAIN SELECT *  FROM employees.titles WHERE emp_no > 10026 ORDER BY emp_no  \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 221659
     filtered: 100.00
        Extra: Using where
```
### (4). 结论
```
查询条件+排序条件结合起来,符合最左元则,则可以利用索引的优势,让MySQL省去:Using filesort
如果Using filesort无法避免,就尽量的使用覆盖索引.
```