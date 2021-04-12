---
layout: post
title: 'JavaCC 加法运算(二)'
date: 2021-04-11
author: 李新
tags:  JavaCC
---

### (1). 前言
> 这一小节,主要是通过一个加法运算来入门.   

### (2). 步骤
> 需求是这样的,用户输入一个表达式,对表达式的内容进行求和,例如:(1+2+3).   
> 1. 编写*.jj文件.   
> 2. 编译*.jj到*.java.   
> 3. 运行Java程序.
> 4. 稍微剖析下原理.   

### (3). 编写add.jj
```
options {
    STATIC = false ;
}

PARSER_BEGIN(Adder)
    class Adder {
        public static void main( String[] args ) throws ParseException, TokenMgrError, NumberFormatException {
            Adder parser = new Adder( System.in );
            int val = parser.start();
            System.out.println(val);
        }
    }
PARSER_END(Adder)

SKIP : { " "}
SKIP : { "\n" | "\r" | "\r\n" }
TOKEN : { < PLUS : "+" > }
TOKEN : { < DECIMAL : (["0"-"9"])+ > }

int start() throws NumberFormatException :
{
    int i ;
    int value ;
}
{
    value = toInteger()
    (
        <PLUS>
        i = toInteger()
        { value += i ; }
    )*
    <EOF>
    { return value ; }
}

int toInteger() throws NumberFormatException :
{
    Token t ;
}
{
    t=<DECIMAL>
    { return Integer.parseInt( t.image ) ; }
}
```
### (4). 编译*.jj
```
# 编译adder.jj
MacBook-Pro:adder lixin$ javacc adder.jj
Java Compiler Compiler Version 7.0.9 (Parser Generator)
(type "javacc" with no arguments for help)
Reading from file adder.jj . . .
File "TokenMgrError.java" does not exist.  Will create one.
File "ParseException.java" does not exist.  Will create one.
File "Token.java" does not exist.  Will create one.
File "SimpleCharStream.java" does not exist.  Will create one.
Parser generated successfully.
```
### (5). 查看编译后的内容
```
MacBook-Pro:adder lixin$ ll
-rw-r--r--   1 lixin  staff   5896  4 12 16:56 Adder.java
-rw-r--r--   1 lixin  staff    557  4 12 16:56 AdderConstants.java
-rw-r--r--   1 lixin  staff   8353  4 12 16:56 AdderTokenManager.java
-rw-r--r--   1 lixin  staff   6176  4 12 16:56 ParseException.java
-rw-r--r--   1 lixin  staff  11828  4 12 16:56 SimpleCharStream.java
-rw-r--r--   1 lixin  staff   4070  4 12 16:56 Token.java
-rw-r--r--   1 lixin  staff   4429  4 12 16:56 TokenMgrError.java

-rw-r--r--   1 lixin  staff    804  4 12 16:41 adder.jj
```
### (6). 编译*.java文件
```
MacBook-Pro:adder lixin$ javac *.java
```
### (7). 运行测试
```
# 1. 创建要运算的内容
MacBook-Pro:adder lixin$ echo "1 + 2 + 3 " > input.txt

# 2. 运行显示结果为:6
MacBook-Pro:adder lixin$ java Adder < input.txt
6
```
### (8). 部份原理剖析
```
# 生成后的内容.
public static void main(String[] args) throws ParseException, TokenMgrError, NumberFormatException {
	Adder parser = new Adder( System.in );
	int val = parser.start();
	System.out.println(val);
}

# 1. 我们编写的BNF
final public int start() throws ParseException, NumberFormatException {
	
	
	// 2. 声明变量,为临时变量.
	int i;
	// 3. 声明变量,为返回结果.
	int value;
	
	// 上面的内容,对应jj描述的内容
	// int start() throws NumberFormatException :
	// {
	// int i ;
	// int value ;
	// }
	
	
	// 读取一个Token,并转换成为:value
	value = toInteger();
	
	label_1: while (true) {
		switch ((jj_ntk == -1) ? jj_ntk_f() : jj_ntk) {
		case PLUS: {
			;
			break;
		}
		default:
			jj_la1[0] = jj_gen;
			break label_1;
		}
		
		jj_consume_token(PLUS);
		i = toInteger();
		value += i;
	}
	
	jj_consume_token(0);
	{
		if ("" != null)
			return value;
	}
	throw new Error("Missing return statement in function");
}


//int toInteger() throws NumberFormatException :
// {
//    Token t ;
// }
// {
//    t=<DECIMAL>   // 读取Token
//    把token的内容,转换成:int
//    { return Integer.parseInt( t.image ) ; }
// }

final public int toInteger() throws ParseException, NumberFormatException {
	Token t;
	t = jj_consume_token(DECIMAL);
	{
		if ("" != null)
			return Integer.parseInt(t.image);
	}
	throw new Error("Missing return statement in function");
}
```
### (9). 总结
> 1. JavaCC通过.jj文件来定义词法和语法解析(有点类似于:DSL).  
> 2. 对文本进行切割的过程就是词法解析,并根据规则(SKIP),过滤掉一些不要的,最后留下来的内容为:Token.   
> 3. 对上一步产生Tokens进行语法解析,和业务逻辑的执行.    
> 4. 为什么非要看JavaCC,因为,SQL的解析是用JavaCC来做的,当然,也能更加有利于用知道SQL执行原理哈.      