---
layout: post
title: 'XXL-Job XxlJobLogger(四)'
date: 2020-02-10
author: 李新
tags: XXL-Job源码
---

### (1). 概述
> 在xxl-job-admin可以查看远程job执行日志,那么这一部份是怎么实现的呢?

### (2). XxlJobLogger打印日志
```
// xxl-job要求打印日志,要求通过这个类来进行输出.
XxlJobLogger.log("XXL-JOB, Hello World.");
```
### (3). XxlJobLogger.log
```
import org.slf4j.helpers.FormattingTuple;
import org.slf4j.helpers.MessageFormatter;

public static void log(String appendLogPattern, Object ... appendLogArguments) {
	// 1. 引用的是:slf4j,刚开始我以为是引用了JDK的MessageFormater
	//    appendLogPattern : 为表达式,例如:log( "hello,[{}]" , "world")
	//    appendLogArguments : 表达式中的占位符
	FormattingTuple ft = MessageFormatter.arrayFormat(appendLogPattern, appendLogArguments);
	// 2. 格式化后的字符串信息.
	String appendLog = ft.getMessage();

	// 3. new Throwable().getStackTrace() 
	//    StackTraceElement返回的是调用栈信息.这里为什么是1,是因为要把当前方法给排除掉.
	StackTraceElement callInfo = new Throwable().getStackTrace()[1];
	logDetail(callInfo, appendLog);
} // end log

private static void logDetail(StackTraceElement callInfo, String appendLog) {
	// 打印日志的格式如下:
	/*// "yyyy-MM-dd HH:mm:ss [ClassName]-[MethodName]-[LineNumber]-[ThreadName] log";
	StackTraceElement[] stackTraceElements = new Throwable().getStackTrace();
	StackTraceElement callInfo = stackTraceElements[1];*/

	StringBuffer stringBuffer = new StringBuffer();
	stringBuffer
		// 时间
	    .append(DateUtil.formatDateTime(new Date()))
	    .append(" ")
		// 类名称+方法名称
		.append("["+ callInfo.getClassName() + "#" + callInfo.getMethodName() +"]").append("-")
		// 代码所在的行数
		.append("["+ callInfo.getLineNumber() +"]").append("-")
		// 线程名称
		.append("["+ Thread.currentThread().getName() +"]").append(" ")
		// 日志详细内容
		.append(appendLog!=null?appendLog:"");
	String formatAppendLog = stringBuffer.toString();

	// 要写入的文件路径,从ThreadLocal中获取,那么,肯定会先在ThreadLocal中保存.
	//  不关注这个内容了.
	// appendlog
	String logFileName = XxlJobFileAppender.contextHolder.get();
	if (logFileName!=null && logFileName.trim().length()>0) {
		// 把日志信息写入到文件中
		XxlJobFileAppender.appendLog(logFileName, formatAppendLog);
	} else {
		logger.info(">>>>>>>>>>> {}", formatAppendLog);
	}
} //end logDetail
```
### (4). 总结
> xxl-job对日志输出这一块,除了日志格式化,其余的内容相当是自己写了一套了.   
