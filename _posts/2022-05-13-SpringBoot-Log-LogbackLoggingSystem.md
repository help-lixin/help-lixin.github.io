---
layout: post
title: 'Spring Boot是如何加载日志之LogbackLoggingSystem' 
date: 2022-05-13
author: 李新
tags:  SpringBoot Slf4j Logback
---

### (1). 概述
在前分析了,Spring Boot是通过(SPI)LoggingSystemFactory工厂加载:LoggingSystem(以LogbackLoggingSystem为例),然后,调用LoggingSystem的相关方法加载XML.   

### (2). Logback加载xml的顺序
```
protected String[] getStandardConfigLocations() {
	// ********************************************************************
	// logback-test.groovy
	// logback-test.xml
	// logback.groovy
	// logback.xml
	// ********************************************************************
	return new String[] { "logback-test.groovy", "logback-test.xml", "logback.groovy", "logback.xml" };
}
```
### (3). Logback加载xml(spring)的顺序
```
protected String[] getSpringConfigLocations() {
	String[] locations = getStandardConfigLocations();
	for (int i = 0; i < locations.length; i++) {
		// ********************************************************************
		String extension = StringUtils.getFilenameExtension(locations[i]);
		// logback-test-spring.groovy
		// logback-test-spring.xml
		// logback-spring.groovy
		// logback-spring.xml
		// ********************************************************************
		locations[i] = locations[i].substring(0, locations[i].length() - extension.length() - 1) + "-spring."
				+ extension;
	}
	return locations;
}
```
### (4). LogbackLoggingSystem.initialize
```
public void initialize(LoggingInitializationContext initializationContext, String configLocation, LogFile logFile) {
	LoggerContext loggerContext = getLoggerContext();
	if (isAlreadyInitialized(loggerContext)) {
		return;
	}
	// ********************************************************************
	// 调用父类AbstractLoggingSystem.initialize
	// ********************************************************************
	super.initialize(initializationContext, configLocation, logFile);
	loggerContext.getTurboFilterList().remove(FILTER);
	markAsInitialized(loggerContext);
	if (StringUtils.hasText(System.getProperty(CONFIGURATION_FILE_PROPERTY))) {
		getLogger(LogbackLoggingSystem.class.getName()).warn("Ignoring '" + CONFIGURATION_FILE_PROPERTY
				+ "' system property. Please use 'logging.config' instead.");
	}
}
```
### (5). AbstractLoggingSystem.initialize
```
public void initialize(LoggingInitializationContext initializationContext, String configLocation, LogFile logFile) {
	if (StringUtils.hasLength(configLocation)) {
		initializeWithSpecificConfig(initializationContext, configLocation, logFile);
		return;
	}
	
	// ********************************************************************
	// ********************************************************************
	initializeWithConventions(initializationContext, logFile);
} // end initialize

private void initializeWithConventions(LoggingInitializationContext initializationContext, LogFile logFile) {
	
	// ********************************************************************
	// 1. 读取如下配置
	// logback-test.groovy
	// logback-test.xml
	// logback.groovy
	// logback.xml
	// ********************************************************************
	String config = getSelfInitializationConfig();
	if (config != null && logFile == null) {
		// self initialization has occurred, reinitialize in case of property changes
		reinitialize(initializationContext);
		return;
	}
	
	// 2. 读取spring的配置
	if (config == null) {
		// ********************************************************************
		// logback-test-spring.groovy
		// logback-test-spring.xml
		// logback-spring.groovy
		// logback-spring.xml
		// ********************************************************************
		config = getSpringInitializationConfig();
	}
	
	// 3. 加载配置文件
	if (config != null) {
		// ********************************************************************
		// 委托给子类:LogbackLoggingSystem.loadConfiguration
		// ********************************************************************
		loadConfiguration(initializationContext, config, logFile);
		return;
	}
	loadDefaults(initializationContext, logFile);
} // end 
```
### (6). LogbackLoggingSystem.loadConfiguration
```
protected void loadConfiguration(LoggingInitializationContext initializationContext, String location,
			LogFile logFile) {
	super.loadConfiguration(initializationContext, location, logFile);
	LoggerContext loggerContext = getLoggerContext();
	stopAndReset(loggerContext);
	try {
		// ********************************************************************
		// 加载xml
		// ********************************************************************
		configureByResourceUrl(initializationContext, loggerContext, ResourceUtils.getURL(location));
	}
	catch (Exception ex) {
		throw new IllegalStateException("Could not initialize Logback logging from " + location, ex);
	}
	List<Status> statuses = loggerContext.getStatusManager().getCopyOfStatusList();
	StringBuilder errors = new StringBuilder();
	for (Status status : statuses) {
		if (status.getLevel() == Status.ERROR) {
			errors.append((errors.length() > 0) ? String.format("%n") : "");
			errors.append(status.toString());
		}
	}
	if (errors.length() > 0) {
		throw new IllegalStateException(String.format("Logback configuration error detected: %n%s", errors));
	}
}
```
### (7).  LogbackLoggingSystem.configureByResourceUrl
```
private void configureByResourceUrl(LoggingInitializationContext initializationContext, LoggerContext loggerContext,
			URL url) throws JoranException {
	if (XML_ENABLED && url.toString().endsWith("xml")) {
		// ********************************************************************
		// SpringBootJoranConfigurator是对xml标签的扩展
		// JoranConfigurator是加载logback.xml的入口.
		// ********************************************************************
		JoranConfigurator configurator = new SpringBootJoranConfigurator(initializationContext);
		configurator.setContext(loggerContext);
		configurator.doConfigure(url);
	}
	else {
		new ContextInitializer(loggerContext).configureByResource(url);
	}
}
```
### (8). Logback类结构图
> 类图对照XML查看,效果更明显

!["Logback类结构图"](/assets/logback/imgs/logback.jpg)

### (9). 总结
LogbackLoggingSystem主要是通过加载xml进行初始化. 
