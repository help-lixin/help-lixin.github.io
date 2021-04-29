---
layout: post
title: 'Calcite 通过SQL读取CSV案例入门(一)'
date: 2021-04-11
author: 李新
tags:  Calcite
---

### (1). Calcite是什么?
> Apache Calcite是一款开源SQL解析工具,可以将各种SQL语句解析成抽象语法术AST(Abstract Syntax Tree),之后通过操作AST就可以把SQL中所要表达的算法与关系体现在具体代码之中.  

### (2). 以CSV文件为案例
> 在classpath下创建:EMPS.csv,内容如下:

!["calcite-csv文件"](/assets/calcite/imgs/calcite-csv.jpg)

```
EMPNO:int,NAME:string,DEPTNO:int,GENDER:string,CITY:string,EMPID:int,AGE:int,SLACKER:boolean,MANAGER:boolean,JOINEDAT:date
100,"Fred",10,,,30,25,true,false,"1996-08-03"
110,"Eric",20,"M","San Francisco",3,80,,false,"2001-01-01"
110,"John",40,"M","Vancouver",2,,false,true,"2002-05-03"
120,"Wilma",20,"F",,1,5,,true,"2005-09-07"
130,"Alice",40,"F","Vancouver",2,,false,true,"2007-01-01"
```

### (3). 定义schema(model.json)
> 毕竟,csv是不存在所谓的数据库概念,需要一个定义,来定义scchema信息.

```
{
  "version": "1.0",
  "defaultSchema": "SALES",
  "schemas": [
    {
      "name": "SALES",
      "type": "custom",
      "factory": "org.apache.calcite.adapter.csv.CsvSchemaFactory",
      "operand": {
        "directory": "sales"
      }
    }
  ]
}
```
### (4). CsvTest2
```
package org.apache.calcite.test;

import com.sun.javafx.binding.StringFormatter;

import org.junit.jupiter.api.Test;

import java.sql.*;
import java.util.Properties;

public class CsvTest2 {

  @Test
  public void testQuery() throws Exception {
    String sql = "select * from EMPS";
    String model = "/model.json";

    Properties info = new Properties();
    info.put("model", CsvTest2.class.getResource(model).getPath());
    try (Connection connection = DriverManager.getConnection("jdbc:calcite:", info)) {
      Statement statement = connection.createStatement();
      ResultSet resultSet = statement.executeQuery(sql);

      while(resultSet.next()){
        int empno = resultSet.getInt(1);
        String name = resultSet.getString(2);
        int deptno = resultSet.getInt(3);
        String gender = resultSet.getString(4);
        String city = resultSet.getString(5);
        int empid = resultSet.getInt(6);
        int age = resultSet.getInt(7);
        boolean slacker = resultSet.getBoolean(8);
        boolean manager = resultSet.getBoolean(9);
        Date joinDate = resultSet.getDate(10);

        String txt = String.format("empno=%d,name=%s,deptno=%d,gender=%s,city=%s,empid=%d,age=%d,slacker=%b,manager=%b,joinDate=%tF",empno,name,deptno,gender,city,empid,age,slacker,manager,joinDate);
        System.out.println(txt);
      }
    }
  }
}
```
### (5). 查看控制台
```
empno=100,name=Fred,deptno=10,gender=,city=,empid=30,age=25,slacker=true,manager=false,joinDate=1996-08-03
empno=110,name=Eric,deptno=20,gender=M,city=San Francisco,empid=3,age=80,slacker=false,manager=false,joinDate=2001-01-01
empno=110,name=John,deptno=40,gender=M,city=Vancouver,empid=2,age=0,slacker=false,manager=true,joinDate=2002-05-03
empno=120,name=Wilma,deptno=20,gender=F,city=,empid=1,age=5,slacker=false,manager=true,joinDate=2005-09-07
empno=130,name=Alice,deptno=40,gender=F,city=Vancouver,empid=2,age=0,slacker=false,manager=true,joinDate=2007-01-01
```

### (6). pom.xml
```
<dependency>
	<groupId>org.apache.calcite</groupId>
	<artifactId>calcite-core</artifactId>
	<version>1.26.0</version>
</dependency>
<dependency>
	<groupId>org.apache.calcite</groupId>
	<artifactId>calcite-linq4j</artifactId>
	<version>1.26.0</version>
</dependency>
<dependency>
	<groupId>org.apache.calcite</groupId>
	<artifactId>calcite-file</artifactId>
	<version>1.26.0</version>
</dependency>
```
### (7). 总结
> 通过SQL,能直接读取CSV,是不是感觉很神奇?至于原理,后面会分析源码得出结论.  