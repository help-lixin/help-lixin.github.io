---
layout: post
title: 'MySQL Explain执行计划'
date: 2017-10-10
author: 李新
tags: MySQL
---

### (1). 前言
> ["本案例的SQL脚本来源于:https://github.com/help-lixin/test_db"](https://github.com/help-lixin/test_db)

### (2). 如何进行MySQL性能优化
> 1. 打开MySQL的慢SQL功能,把查询超过N秒的语句定义为慢SQL.  
> 2. 把抓取的慢SQL,通过Explain对SQL语句的性能进行判断.   

### (3). 打开慢SQL方式
```
# OFF : 关闭
# ON  : 启用
mysql> show variables like 'slow_query%';
+---------------------+----------------------------------------------+
| Variable_name       | Value                                        |
+---------------------+----------------------------------------------+
| slow_query_log      | OFF                                          |
| slow_query_log_file | /usr/local/mysql/data/lixin-macbook-slow.log |
+---------------------+----------------------------------------------+

mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+

# 1. 打开慢SQL方式一
mysql> SET GLOBAL slow_query_log='ON'; 
mysql> SET GLOBAL slow_query_log_file='/usr/local/mysql/data/lixin-macbook-slow.log';
mysql> SET GLOBAL long_query_time=5;

# 2. 打开慢SQL方式二
[mysqld]
slow_query_log = ON
slow_query_log_file = /usr/local/mysql/data/lixin-macbook-slow.log
long_query_time = 5
```
### (4). Explain简单使用
```
# \G   :  将查询结果进行按列打印.
# \W   :  显示所有的warning信息
mysql> EXPLAIN SELECT * FROM employees WHERE emp_no = 10070\G\W
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL

# 显示警告信息
mysql> SHOW WARNINGS\G
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: 
    /* select#1 */ 
	/* 索引查询,直接从B+TREE里直接取出 */
	select 
	      '10070' AS `emp_no`,
	      '1955-08-20' AS `birth_date`,
		  'Reuven' AS `first_name`,
		  'Garigliano' AS `last_name`,
		  'M' AS `gender`,
		  '1985-10-14' AS `hire_date` 
	from `employees`.`employees` 
	where 1
```
### (5). Explain id列
在多条SQL语句中,ID越大的越先执行,如果ID相同的,在上面的先执行.

### (6). Explain select_type列详解
select_type : 代表查询类型
1. SIMPLE(简单的SELECT语句)  
2. PRIMARY(查询中若包含任何复杂的子部分,最外层查询则被标记为Primary)   
3. DERIVED(衍生表,在FROM(不在WHERE)中包含的子查询被标记为DERIVED)   
4. SUBQUERY(在SELECT或WHERE列表中包含了子查询)   
5. UNION(联合查询)  

```
# 1. SIMPLE 简单的SELECT语句(不包括UNION操作或子查询操作)
mysql> EXPLAIN SELECT * FROM employees WHERE emp_no = 10070\G\W
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE


# 2. PRIMARY (外部的主查询):查询中若包含任何复杂的子部分,最外层查询则被标记为Primary
#    SUBQUERY(在SELECT或WHERE列表中包含了子查询)
mysql> EXPLAIN SELECT * FROM employees.employees  WHERE birth_date >= ( SELECT MAX(birth_date) FROM employees.employees )\G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
*************************** 2. row ***************************
           id: 2
  select_type: SUBQUERY


# 3. DERIVED  衍生表,在FROM中包含(不在WHERE)的子查询被标记为DERIVED
mysql> SET SESSION optimizer_switch="derived_merge=off";
mysql> EXPLAIN SELECT t.*  
FROM ( SELECT * FROM employees.employees )  AS t 
WHERE t.emp_no = 10034 \G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
  table: <derived2>
*************************** 2. row ***************************
           id: 2
  select_type: DERIVED
  
  
# 4. UNION(联合查询)   
mysql> EXPLAIN 
SELECT *
FROM employees.employees WHERE emp_no = 10034
UNION
SELECT *
FROM employees.employees WHERE emp_no = 10035 \G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: employees
*************************** 2. row ***************************
           id: 2
  select_type: UNION
        table: employees
*************************** 3. row ***************************
           id: NULL
  select_type: UNION RESULT
        table: <union1,2>  
```

### (7). Explain type列详解
<font color='red'>type : 可以直观的判断出当前SQL语句的性能,从最好到最差依次是: null > system > const > eq_ref > ref > range > index > ALL.</font>        
对于SQL优化来说:要尽可能保证type列的值属于range及以上级别.

```
# 1. type:null,性能是最好的,一般是使用了聚合(MIN/MAX...)函数操作索引列,结果直接从索引树获取即可,因此,性能是最好的.
mysql> EXPLAIN 
SELECT MIN(emp_no)
FROM employees.employees  \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: NULL
   partitions: NULL
         type: NULL

# 2. const   使用索引(emp_no)与常量(10034)进行比较.
#    system  
mysql> EXPLAIN SELECT t.*  
FROM ( SELECT * FROM employees.employees WHERE emp_no = 10034 )  AS t  \G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
         type: system
*************************** 2. row ***************************
           id: 2
  select_type: DERIVED
        table: employees
         type: const


# 3. eq_ref : 在进行连接查询时,连接查询的条件中使用了主键(唯一索引)进行条件(WHERE)进行过滤,因此,这种类型的SQL就是:eq_ref
mysql> EXPLAIN SELECT e.emp_no
    -> FROM departments d , dept_emp e
    -> WHERE  d.dept_no = e.dept_no
    -> AND e.emp_no = '10126' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: e
   partitions: NULL
         type: ref
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: d
   partitions: NULL
         type: eq_ref

# 4. ref: 当使用普通索引进行查询时,type的类型就是:ref
mysql> EXPLAIN
    -> SELECT *
    -> FROM dept_emp
    -> WHERE dept_no = 'd005'  \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
   partitions: NULL
         type: ref

# 4. ref: 在复杂查询的情况下,type=ref
mysql> EXPLAIN 
SELECT 
   e.emp_no, d.dept_no
FROM dept_emp d, employees e
WHERE d.emp_no = e.emp_no \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: e
   partitions: NULL
         type: index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: d
   partitions: NULL
         type: ref


# 5. range,使用了索引,并进行范围查询
mysql> EXPLAIN SELECT * FROM employees.employees WHERE emp_no > 10004 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
   partitions: NULL
         type: range


# 6. index,查询的所有记录,可以从索引树中直接获取(索引覆盖)
# 在表:dept_emp中:emp_no,dept_no为联合索引,所以,直接从索引树中获得数据,不需要再去PRIMARY B+ TREE中获取数据了
mysql> SHOW INDEX FROM dept_emp \G;
mysql> EXPLAIN SELECT emp_no,dept_no FROM dept_emp \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
         type: index


# 7. ALL,全表扫描.
# dept_emp表中:emp_no,dept_no是联合索引,但是,to_date并没有索引,所以,需要回到PRIARY B+ TREE树里检索.
mysql> EXPLAIN SELECT emp_no,dept_no,to_date FROM dept_emp \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
         type: ALL

# 8. index_merge,对多个索引分别进行条件扫描,然后将它们各自的结果进行合并
# 在MySQL5.0之前,一个表一次只能使用一个索引,无法同时使用多个索引分别进行条件扫描.
# 但是从5.1开始,引入了index merge优化技术,对同一个表可以使用多个索引分别进行条件扫描.  
mysql> EXPLAIN SELECT *  FROM dept_emp WHERE emp_no = 10014 OR  dept_no = 'd005' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
   partitions: NULL
         type: index_merge
possible_keys: PRIMARY,dept_no
          key: PRIMARY,dept_no     # 同时,使用了主键索引和普通索引(dept_no)
      key_len: 4,12                # int=4 char=(4*3)
          ref: NULL
         rows: 148055
     filtered: 100.00
        Extra: Using union(PRIMARY,dept_no); Using where
```
### (8). Explain possible_keys列详解
possible_keys : 这一次的查询,可能会使用哪些索引,MySQL内部优化器会进行判断,如果此次查询走索引性能比全表扫描的性能要差,那么,内部优化器让此次查询进行全表扫描.

```
# dept_emp表总共有:331603数据,而查询可能命中:331143行,所以,优化器让其全表扫描了.
mysql> EXPLAIN SELECT * FROM dept_emp WHERE dept_no LIKE 'd005%' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
   partitions: NULL
         type: ALL
possible_keys: dept_no      # 因为WHERE是LIKE,所以,并没有使用索引.
          key: NULL         # NULL实际上没有用到索引.
      key_len: NULL
          ref: NULL
         rows: 331143       # 查询命中的数据:331143
     filtered: 44.71
        Extra: Using where
```
### (9). Explain key列详解
key : 此次SQL查询,所使用的索引.  

### (10). Explain rows列详解
rows : 该SQL语句,可能要查询的数据条数,该值越小越好.

### (11). Explain key_len列详解
key_len : 表示本次查询中,所选择的索引长度有多少字节,通常我们可借此判断联合索引有多少列被选择了.  

```
# 1. 如果:字段允许NULL的会多出1 bytes来记录,而NOT NULL则不需要.
# 2. 总的bytes数是与字符集是有关系的.

# 字符集所占用的bytes:
# character set        : utf8=3 , gbk=2 , latin1=1
# 
#  typeint             : 1 bytes
#  smallint            : 2 bytes
#  middleint           : 3 bytes
#  int                 : 4 bytes
#  bigint              : 8 bytes
#  
#  date                : 3 bytes
#  timestamp           : 4 bytes
#  datetime            : 8 bytes
#  
#  char(N)  NOT NULL   : (N * character set)
#  char(N)  NULL       : (N * character set) + 1 bytes
#  
#  varchar(N) NOT NULL : (N * character set) + 2 bytes
#  varchar(N) NULL     : (N * character set) + 1 bytes + 2 bytes
#  
#  比如: id INT             -->   ken_len = 4
#  比如: id INT NOT NULL    -->   ken_len = 4 + 1 


# 1. 查看表的字符集
mysql> show table status from employees LIKE 'dept_emp'\G
*************************** 1. row ***************************
           Name: dept_emp
         Engine: InnoDB
      Collation: utf8_general_ci


# 2. 查看下dept_emp表的结构
mysql> desc dept_emp;
+-----------+---------+------+-----+---------+-------+
| Field     | Type    | Null | Key | Default | Extra |
+-----------+---------+------+-----+---------+-------+
| emp_no    | int(11) | NO   | PRI | NULL    |       |
| dept_no   | char(4) | NO   | PRI | NULL    |       |
| from_date | date    | NO   |     | NULL    |       |
| to_date   | date    | NO   |     | NULL    |       |
+-----------+---------+------+-----+---------+-------+

# 3. dept_emp表中,存在联合索引(PRIMARY)
# keylen = emp_no(4) + dept_no(4*3) = 16
mysql> EXPLAIN SELECT *  FROM dept_emp WHERE emp_no = 10014 AND dept_no = 'd005' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
   partitions: NULL
         type: const
possible_keys: PRIMARY,dept_no
          key: PRIMARY
      key_len: 16
	  
# 4. dept_emp表中,存在联合索引(PRIMARY)
# keylen = emp_no(4) = 4
mysql> EXPLAIN SELECT *  FROM dept_emp WHERE emp_no = 10014  \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
   partitions: NULL
         type: ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
```
### (12). Explain Extra列详解
Extra : 提供了额外的信息,可以帮助我们判断当前SQL是否使用了覆盖索引,文件排序,使用索引进行查询条件等等信息.

```
# ********************************************************************
# Using Index   :  使用了索引覆盖(查询的结果列是索引的情况下,无须再回到PRIMARY B+ TREE进行取数据,直接在索引树里可获取数据)
#                  使用索引覆盖是SQL优化的重点.
# ********************************************************************
mysql> EXPLAIN SELECT emp_no  FROM dept_emp WHERE emp_no = 10014  \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
         type: ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
        Extra: Using index

# ********************************************************************
# Using Where : 使用了Where条件,但是,没有使用到索引,这种性能是不行的.
# ********************************************************************
mysql> EXPLAIN SELECT * FROM employees.employees WHERE first_name = 'Mary' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 299025
     filtered: 10.00
        Extra: Using where

# ********************************************************************
# Using Index Condition  : 查询的结果(因为SELECT是*)没有使用索引覆盖(建议使用索引覆盖),同时,WHERE只使用到了普通索引
# ********************************************************************
# 添加一个普通索引
mysql> ALTER TABLE employees.salaries ADD  INDEX idx_salary (salary);
mysql> EXPLAIN SELECT * FROM salaries WHERE salary > 108220 \G
# 优化:使用索引覆盖来实现优化
# mysql> EXPLAIN SELECT emp_no FROM salaries WHERE salary > 108220 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: salaries
   partitions: NULL
         type: range
possible_keys: idx_salary
          key: idx_salary
      key_len: 4
          ref: NULL
         rows: 75504
     filtered: 100.00
        Extra: Using index condition

# ********************************************************************
# Using Temporary : 在没有索引的列上使用了临时表去执行去重,性能不是很好,建议添加索引来达到优化.
# ********************************************************************
mysql> EXPLAIN SELECT DISTINCT title,to_date  FROM titles \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 443318
     filtered: 100.00
        Extra: Using temporary

# 在有索引的列上去执行去重
mysql> ALTER TABLE titles ADD INDEX titles(title);
mysql> EXPLAIN SELECT DISTINCT title  FROM titles \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: range
possible_keys: PRIMARY,titles
          key: titles
      key_len: 152
          ref: NULL
         rows: 7
     filtered: 100.00
        Extra: Using index for group-by   # 给要去重的列添加索引后,立即变成了使用覆盖索引了.

# ********************************************************************
# Using filesort   : 使用文件排序,会使用磁盘+内存的方式进行文件排序(单路排序,双路排序).
# ********************************************************************
mysql> EXPLAIN SELECT * FROM titles ORDER BY title \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 443318
     filtered: 100.00
        Extra: Using filesort

# ********************************************************************
# Select tables optimized away : 直接在索引列上使用了聚合函数,不需要操作表
# ********************************************************************
mysql> EXPLAIN SELECT MIN(emp_no) FROM titles \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: NULL
   partitions: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: Select tables optimized away
```

### (13). 案例
```
# 1. 当前字符集为UTF8
# 2. 查看表结构(titles)
mysql> desc titles;
+-----------+-------------+------+-----+---------+-------+
| Field     | Type        | Null | Key | Default | Extra |
+-----------+-------------+------+-----+---------+-------+
| emp_no    | int(11)     | NO   | PRI | NULL    |       |
| title     | varchar(50) | NO   | PRI | NULL    |       |
| from_date | date        | NO   | PRI | NULL    |       |
| to_date   | date        | YES  |     | NULL    |       |
+-----------+-------------+------+-----+---------+-------+

# 3.查看表(titles)有哪些索引
# PRIMARY(emp_no,title,from_date)
mysql> SHOW INDEX FROM titles\G;
*************************** 1. row ***************************
        Table: titles
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: emp_no
    Collation: A
  Cardinality: 299628
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
*************************** 2. row ***************************
        Table: titles
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 2
  Column_name: title
    Collation: A
  Cardinality: 442843
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
*************************** 3. row ***************************
        Table: titles
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 3
  Column_name: from_date
    Collation: A
  Cardinality: 443318
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:

# ***********************************************************
# 4. 命中部份索引(emp_no)
# ken_len = emp_no(4) 
mysql> EXPLAIN SELECT * FROM employees.titles  WHERE emp_no = 10007 \G
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
        Extra: NULL

# 5.命中部份索引(emp_no和title)
# key_len = emp_no(4) + title(50*3+2) = 156
mysql> EXPLAIN SELECT * FROM employees.titles  WHERE emp_no = 10007 AND title > 'Staff'  \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: range
possible_keys: PRIMARY,titles
          key: PRIMARY
      key_len: 156
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using where

# 6.命中全部索引(emp_no,title,from_date)
# 注意:条件全都是等于.
# key_len = emp_no(4) + title(50*3+2) + from_date(3) = 159
mysql> EXPLAIN SELECT * FROM employees.titles  WHERE emp_no = 10007 AND title = 'Staff' AND from_date = STR_TO_DATE('1989-02-10' , '%Y-%m-%d')  \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: const
possible_keys: PRIMARY,titles
          key: PRIMARY
      key_len: 159
          ref: const,const,const
         rows: 1
     filtered: 100.00
        Extra: NULL

# **********************************************************************************
# 7.命中部份索引(emp_no,title)
# key_len = emp_no(3) + title(50*3+2) = 156
# 注意:title条件是大于.
# 因为:是区间查询,与from_date是排拆的,所以,from_date没有命中索引,需要回表检索.
# **********************************************************************************
mysql> EXPLAIN SELECT * FROM employees.titles  WHERE emp_no = 10007 AND title > 'Staff' AND from_date = STR_TO_DATE('1989-02-10' , '%Y-%m-%d')  \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: titles
   partitions: NULL
         type: range
possible_keys: PRIMARY,titles
          key: PRIMARY
      key_len: 156
          ref: NULL
         rows: 1
     filtered: 10.00
        Extra: Using where

# 7. 联合索引使用总结
# 7.1 当使用联合索引时,如果使用了大于号(>),则会造成部份索引命中失效. 
# 7.2 对索引列,使用了函数,会造成索引命中失效.  
# 7.3 使用不等于(!=/<>)会导致全表扫描
# 7.4 使用IS NULL,IS NOT NULL,会造成全表扫描.  
# 7.5 使用LIKE时,通配符(%)只能在后面,否则会造成全表扫描.   
# 7.6 索引列与查询常量类型不相同,会造成全表扫描.  
# 7.7 少用OR/IN,MySQL内部优化器可能不走索引.  
# 7.8 范围查询时,数据尽可能的少,如果命中数据量太大了,优化器可能不走索引. 
```
### (14). 总结
```
# 查看EXPLAIN的顺序
# 1. 看ID,ID越大的,越优先执行,ID相同的,从上至下执行.  
# 2. 看type,如果type:ALL,需要先解决全表扫描.  
# 3. 看key,如果key:NULL,则需要添加索引.
# 4. 看rows,rows的数量尽可能的少.
# 5. 对于联合索引,是否索引覆盖查看:ken_len,根据ken_len进行计算.   
# 6. 看Extra,尽可能是:Using index(使用覆盖索引).
```