---
layout: post
title: 'JavaCC 安装(一)'
date: 2021-04-11
author: 李新
tags:  JavaCC
---

### (1). JavaCC介绍
> 在看Calcite时,有看到使用了JavaCC,所以,得先放下手里的Caclite,那么,什么是JavaCC呢?
> JavaCC是一个能生成词法和语法的分析器的生成程序.  
> 词法分析器,就是对一串文本进行拆分,转换成一系列的Token.  
> 语法分析器,就是对词法分析器产生的Token,进行校验.  
> 给我感觉:MyBatis的动态SQL有这个功能来着的.

### (2). JavaCC安装
```
lixin-macbook:~ lixin$ wget https://github.com/javacc/javacc/archive/javacc-7.0.9.tar.gz
lixin-macbook:~ lixin$ tar -zxvf javacc-7.0.9.tar.gz
lixin-macbook:~ lixin$ mv javacc-javacc-7.0.9 javacc-7.0.9
lixin-macbook:~ lixin$ mv javacc-7.0.9 ~/Developer/
lixin-macbook:~ lixin$ cd ~/Developer/javacc-7.0.9/
lixin-macbook:javacc-7.0.9 lixin$ ant
lixin-macbook:javacc-7.0.9 lixin$ chmod 755 ./scripts/javacc
```
### (3). 设置环境变量
```
lixin-macbook:~ lixin$ vi ~/.bash_profile
JAVACC_HOME=/Users/lixin/Developer/javacc-7.0.9
PATH=$PATH:$JAVACC_HOME/scripts:
```
### (4). 环境变量生效
```
lixin-macbook:~ lixin$ source ~/.bash_profile
```
### (5). 运行javacc
```
lixin-macbook:~ lixin$ javacc
Java Compiler Compiler Version 7.0.9 (Parser Generator)

Usage:
    javacc option-settings inputfile

"option-settings" is a sequence of settings separated by spaces.
Each option setting must be of one of the following forms:

    -optionname=value (e.g., -STATIC=false)
    -optionname:value (e.g., -STATIC:false)
    -optionname       (equivalent to -optionname=true.  e.g., -STATIC)
    -NOoptionname     (equivalent to -optionname=false. e.g., -NOSTATIC)

Option settings are not case-sensitive, so one can say "-nOsTaTiC" instead
of "-NOSTATIC".  Option values must be appropriate for the corresponding
option, and must be either an integer, a boolean, or a string value.

The integer valued options are:

    CHOICE_AMBIGUITY_CHECK (default : 2)
    DEPTH_LIMIT            (default : 0)
    LOOKAHEAD              (default : 1)
    OTHER_AMBIGUITY_CHECK  (default : 1)

The boolean valued options are:

    BUILD_PARSER                    (default : true)
    BUILD_TOKEN_MANAGER             (default : true)
    CACHE_TOKENS                    (default : false)
    COMMON_TOKEN_ACTION             (default : false)
    DEBUG_LOOKAHEAD                 (default : false)
    DEBUG_PARSER                    (default : false)
    DEBUG_TOKEN_MANAGER             (default : false)
    ERROR_REPORTING                 (default : true)
    FORCE_LA_CHECK                  (default : false)
    GENERATE_ANNOTATIONS            (default : false)
    GENERATE_BOILERPLATE            (default : true)
    GENERATE_CHAINED_EXCEPTION      (default : false)
    GENERATE_GENERICS               (default : false)
    GENERATE_STRING_BUILDER         (default : false)
    IGNORE_ACTIONS                  (default : false)
    IGNORE_CASE                     (default : false)
    JAVA_UNICODE_ESCAPE             (default : false)
    KEEP_LINE_COLUMN                (default : true)
    NO_DFA                          (default : false)
    SANITY_CHECK                    (default : true)
    STATIC                          (default : true)
    STOP_ON_FIRST_ERROR             (default : false)
    SUPPORT_CLASS_VISIBILITY_PUBLIC (default : true)
    TOKEN_MANAGER_USES_PARSER       (default : false)
    UNICODE_INPUT                   (default : false)
    USER_CHAR_STREAM                (default : false)
    USER_TOKEN_MANAGER              (default : false)

The string valued options are:

    GRAMMAR_ENCODING             (default : <<empty>>)
    JAVA_TEMPLATE_TYPE           (default : classic)
    JDK_VERSION                  (default : 1.5)
    NAMESPACE                    (default : <<empty>>)
    OUTPUT_DIRECTORY             (default : .)
    OUTPUT_LANGUAGE              (default : java)
    PARSER_CODE_GENERATOR        (default : <<empty>>)
    PARSER_INCLUDE               (default : <<empty>>)
    PARSER_SUPER_CLASS
    STACK_LIMIT                  (default : <<empty>>)
    TOKEN_EXTENDS                (default : <<empty>>)
    TOKEN_FACTORY                (default : <<empty>>)
    TOKEN_INCLUDE                (default : <<empty>>)
    TOKEN_MANAGER_CODE_GENERATOR (default : <<empty>>)
    TOKEN_MANAGER_INCLUDE        (default : <<empty>>)
    TOKEN_MANAGER_SUPER_CLASS
    TOKEN_SUPER_CLASS

EXAMPLE:
    javacc -STATIC=false -LOOKAHEAD:2 -debug_parser mygrammar.jj
```
### (6). 总结
> 能显示上面的内容,代表javacc安装成功.  