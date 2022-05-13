---
layout: post
title: 'Spring Boot是如何加载日志之LoggingApplicationListener' 
date: 2022-05-13
author: 李新
tags:  SpringBoot Slf4j Logback
---

### (1). 概述
最近要对MDC进行扩展,需要看下Spring是如何加载logback的.

### (2). 先看下logback的xml配置文件(logback-spring.xml)
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration scanPeriod="10 seconds">
    <property name="log.path" value="${APPLICATION_LOG_DIR}"/>
    <property name="serverName" value="${spring.application.name}"/>
    <springProperty scope="context"  name="service-name" source="spring.application.name"/>

    <property name="STDOUT_PATTEN"
              value='[%d{yyyy-MM-dd HH:mm:ss.SSS}] %highlight([%level]) [${PID:- }] [%thread] [${service-name}] [%X{traceId:-0}] [%X{tenantId:-0}] [%logger{5}\:%line#%M] %highlight([%msg]) %n'/>
    <!--    [时间][日志等级][进程PID][线程名][服务名称][TraceID][租户ID][包首字母.类名][日志信息]-->
    <property name="FILE_PATTEN"
              value='[%d{yyyy-MM-dd HH:mm:ss.SSS}][%level][${PID:-0}][%thread][${service-name}][%X{traceId:-0}][%X{tenantId:-0}][%logger{5}\:%line#%M][%msg]%n'/>
    <!--    [时间][日志等级][进程PID][线程名][服务名称][TraceID][租户ID][包首字母.类名][日志信息]-->

    <!-- 彩色日志 -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
    <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
    <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />

    <!-- 时间滚动输出 level为 DEBUG 日志 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/${service-name}.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <Pattern>${FILE_PATTEN}</Pattern>
            <charset>UTF-8</charset> <!-- 设置字符集 -->
            <immediateFlush>false</immediateFlush>
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 日志归档 -->
            <fileNamePattern>${log.path}/log-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
    </appender>

    <!--为了防止进程退出时，内存中的数据丢失，请加上此选项-->
    <shutdownHook class="ch.qos.logback.core.hook.DelayingShutdownHook"/>
    <!-- 可用来获取StatusManager中的状态 -->
    <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener"/>

    <!-- 解决debug模式下循环发送的问题 -->
    <logger name="org.apache.http.impl.conn.Wire" level="WARN"/>

    <root level="info">
        <appender-ref ref="FILE"/>
    </root>
	
    <logger name="org.springframework" level="info"/>
    <logger name="org.apache" level="info"/>
</configuration>
```
### (3). 加载日志的入口在哪?
> 答案: spring-boot-2.4.2.jar/META-INF/spring.factories 

```
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.context.logging.LoggingApplicationListener

# Logging Systems
org.springframework.boot.logging.LoggingSystemFactory=\
org.springframework.boot.logging.logback.LogbackLoggingSystem.Factory,\
org.springframework.boot.logging.log4j2.Log4J2LoggingSystem.Factory,\
org.springframework.boot.logging.java.JavaLoggingSystem.Factory
```
### (4). LoggingApplicationListener
```
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationStartingEvent) {
		// 1. 启动事件
		onApplicationStartingEvent((ApplicationStartingEvent) event);
	}
	else if (event instanceof ApplicationEnvironmentPreparedEvent) {
		// 2. 事件准备,会加载logback.xml文件
		onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
	}
	else if (event instanceof ApplicationPreparedEvent) {
		onApplicationPreparedEvent((ApplicationPreparedEvent) event);
	}
	else if (event instanceof ContextClosedEvent
			&& ((ContextClosedEvent) event).getApplicationContext().getParent() == null) {
		onContextClosedEvent();
	}
	else if (event instanceof ApplicationFailedEvent) {
		onApplicationFailedEvent();
	}
} // end onApplicationEvent

```
### (5). 创建LogbackLoggingSystem
```
// 1.1  通过LoggingSystemFactory.fromSpringFactories()加载:LoggingSystemFactory的实现,Spring提供的SPI.
private void onApplicationStartingEvent(ApplicationStartingEvent event) {
	// 我这里以:LogbackLoggingSystem为例
	this.loggingSystem = LoggingSystem.get(event.getSpringApplication().getClassLoader());
	// 1.1.1 触发初始化
	this.loggingSystem.beforeInitialize();
} // end onApplicationStartingEvent
```
### (6). Env准备事件
```
// 2.1 触发准备事件,即加载classpath下的logback.xml
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
	if (this.loggingSystem == null) {
		this.loggingSystem = LoggingSystem.get(event.getSpringApplication().getClassLoader());
	}
	// 初始化
	initialize(event.getEnvironment(), event.getSpringApplication().getClassLoader());
} // end onApplicationEnvironmentPreparedEvent

```
### (7). 准备加载xml
```
protected void initialize(ConfigurableEnvironment environment, ClassLoader classLoader) {
	getLoggingSystemProperties(environment).apply();
	this.logFile = LogFile.get(environment);
	if (this.logFile != null) {
		this.logFile.applyToSystemProperties();
	}
	this.loggerGroups = new LoggerGroups(DEFAULT_GROUP_LOGGERS);
	initializeEarlyLoggingLevel(environment);
	// 初始化,加载xml文件
	initializeSystem(environment, this.loggingSystem, this.logFile);
	initializeFinalLoggingLevels(environment, this.loggingSystem);
	registerShutdownHookIfNecessary(environment, this.loggingSystem);
} // end initialize

```
### (8). 最终委托给LogbackLoggingSystem加载xml
```
private void initializeSystem(ConfigurableEnvironment environment, LoggingSystem system, LogFile logFile) {
	String logConfig = StringUtils.trimWhitespace(environment.getProperty(CONFIG_PROPERTY));
	try {
		LoggingInitializationContext initializationContext = new LoggingInitializationContext(environment);
		if (ignoreLogConfig(logConfig)) {
			system.initialize(initializationContext, null, logFile);
		}
		else {
			system.initialize(initializationContext, logConfig, logFile);
		}
	}
	catch (Exception ex) {
		Throwable exceptionToReport = ex;
		while (exceptionToReport != null && !(exceptionToReport instanceof FileNotFoundException)) {
			exceptionToReport = exceptionToReport.getCause();
		}
		exceptionToReport = (exceptionToReport != null) ? exceptionToReport : ex;
		// NOTE: We can't use the logger here to report the problem
		System.err.println("Logging system failed to initialize using configuration from '" + logConfig + "'");
		exceptionToReport.printStackTrace(System.err);
		throw new IllegalStateException(ex);
	}
} // end initializeSystem
```
### (9). 总结
SpringBoot加载日志的入口为:LoggingApplicationListener,通过SPI加载LoggingSystemFactory的不同实现来实现日志的加载.  