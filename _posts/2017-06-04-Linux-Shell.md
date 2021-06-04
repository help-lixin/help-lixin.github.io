---
layout: post
title: 'Shell基础入门'
date: 2017-06-04
author: 李新
tags: Linux
---

### (1). Shell变量定义
```
# #!是一个约定的标记,它告诉系统这个脚本需要什么解释器来执行,即使用哪一种Shell.  
#!/bin/bash

# 定义变量(注意:变量名称前后不能有空格)
app_name=order-service

# 给变量赋值
app_name=tomcat-app-service

# 字符串拼接
welcome="welcome use "${app_name}""

# 使用变量(只需要在变量名称前面加上美元符号+花括号即可)
echo ${app_name}
echo ${welcome}
# 获得变量的长度
echo "var app_name length:${#app_name}"
```
### (2). Shell数组定义
```
#!/bin/bash

# 声明数组,并给数组赋值
# 注意,数组元素之间是用空格来区分来着的
array=(1 2 3 4 5 6 7 8 9)

# 给数据指定下标赋值
array[9]=10

# 获得数组的长度
echo "array length:${#array[*]}"

# 获取数组指定下标的内容
echo "array[0]=${array[0]}"
echo "array[9]=${array[9]}"
```
### (3). Shell传递参数
```
# $0 : 执行的脚本名称.   
# $# : 传递到脚本的参数个数.  
# $? : 显示最后命令的退出状态,0表示没有错误,其他任何值表明有错误. 
# $$ : 获得当前脚本运行的当前进程ID号.  

#!/bin/bash
# ./test 1 2 3 4
echo "当前shell,运行的过程id:$$"

nohup echo "hello world" > /tmp/a.txt &

# 可以用于:产生pid文件
# $! : 获得后台运行的最后一个进程的ID号
echo "后台运行进程id:$!" 
```
### (4). 基本运算符
```

#   关系运算符
-eq   : 检测两个数是否相等(equals)  
-ne   : 检测两个数是否不相等(!=)  
-gt   : 检测左边的数是否大于右边的(>)  
-lt   : 检测左边的数是否小于右边的(<)   
-ge   : 检测左边的数是否大于等于右边的(>=)   
-le   : 检测左边的数是否小于等于右边的(<=)   


#  布尔运算符
!     : 非运算
-o    : 或运算(||)
-a    : 与运算(&&)


#   逻辑运算符
&&   : 逻辑AND
||   : 逻辑OR
```
### (5). 文件运算符
```
操作符	说明	                                        举例
-b file	检测文件是否是块设备文件，如果是，则返回 true。	[ -b $file ] 返回 false。
-c file	检测文件是否是字符设备文件，如果是，则返回 true。	[ -c $file ] 返回 false。
-d file	检测文件是否是目录，如果是，则返回 true。	[ -d $file ] 返回 false。
-f file	检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。	[ -f $file ] 返回 true。
-g file	检测文件是否设置了 SGID 位，如果是，则返回 true。	[ -g $file ] 返回 false。
-k file	检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。	[ -k $file ] 返回 false。
-p file	检测文件是否是有名管道，如果是，则返回 true。	[ -p $file ] 返回 false。
-u file	检测文件是否设置了 SUID 位，如果是，则返回 true。	[ -u $file ] 返回 false。
-r file	检测文件是否可读，如果是，则返回 true。	[ -r $file ] 返回 true。
-w file	检测文件是否可写，如果是，则返回 true。	[ -w $file ] 返回 true。
-x file	检测文件是否可执行，如果是，则返回 true。	[ -x $file ] 返回 true。
-s file	检测文件是否为空（文件大小是否大于0），不为空返回 true。	[ -s $file ] 返回 true。
-e file	检测文件（包括目录）是否存在，如果是，则返回 true。	[ -e $file ] 返回 true。
```
### (6). if流程控制
```
#!/bin/bash
shell_path=/Users/lixin/Desktop/shell/var.sh
if test -e ${shell_path} && test -r ${shell_path} && test -w ${shell_path} && test -x ${shell_path}
then
    echo "file:${shell_path} rwx"
else
    echo "file:${shell_path} 没有任何权限"
fi
```
### (7). for循环流程控制
```
# 查找mysql进程,并kill掉
#!/bin/bash
pid_array=(`ps aux|grep mysql|grep -v grep|awk '{print $2}'`)

# ${pid_array[*]} 为数组内容
for item in ${pid_array[*]}
do
    kill ${item}
done
```
### (8). do循环流程控制
```
# 查找mysql进程,并kill掉
#!/bin/bash
pid_array=(`ps aux|grep mysql|grep -v grep|awk '{print $2}'`)
len=${#pid_array[*]}
index=0;

while(($index < $len))
do
    pid=${pid_array[$index]}
    kill $pid
    let "index++"
done
```
### (9). case分支控制
```
# 注意:两个分号代表break
#!/bin/bash
enum=$1
case $enum in
1|2|3|4|5)
    echo $enum
    ;;
6|7|8|9|10)
    echo $enum
    ;;
*)
    echo "input error"
    ;;
esac
```
### (10). function
```
# shell中的函数,必须要先声明,才能被调用,也就是说,建议函数认在最顶上.
#!/bin/bash

start(){
   echo "start"
}

stop(){
   echo "stop"
}

restart(){
    stop
    start
}

case "$1" in
"start")
    start
    ;;
"stop")
    stop
    ;;
"restart")
    restart
    ;;
*)
    echo "input error"
    ;;
esac
```