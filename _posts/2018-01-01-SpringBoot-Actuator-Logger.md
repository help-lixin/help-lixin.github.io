---
layout: post
title: 'Spring Boot Actuator'
date: 2018-01-01
author: 李新
tags: SpringBoot
---

### (1). Actuator允许动态调整Log级别
```
//查看所有的日志信息
curl -X GET http://localhost:8080/actuator/loggers

// 修改某个类或者某个包(help.lixin.actuator.controller)的日志级别
curl -X POST -H "Content-Type: application/json" -d '{"configuredLevel": "INFO"}' http://localhost:8080/actuator/loggers/help.lixin.actuator.controller
```
### (2). 看下(LoggersEndpointAutoConfiguration)源码
```
@Configuration
public class LoggersEndpointAutoConfiguration {

	@Bean
	@ConditionalOnBean(LoggingSystem.class)
	@Conditional(OnEnabledLoggingSystemCondition.class)
	@ConditionalOnMissingBean
	@ConditionalOnEnabledEndpoint
	public LoggersEndpoint loggersEndpoint(LoggingSystem loggingSystem) {
		return new LoggersEndpoint(loggingSystem);
	}

	static class OnEnabledLoggingSystemCondition extends SpringBootCondition {

		@Override
		public ConditionOutcome getMatchOutcome(ConditionContext context,
				AnnotatedTypeMetadata metadata) {
			// 判断是否有配置:org.springframework.boot.logging.LoggingSystem =none
			// 如果有配置则返回:false
			ConditionMessage.Builder message = ConditionMessage
					.forCondition("Logging System");
			// org.springframework.boot.logging.LoggingSystem 
			String loggingSystem = System.getProperty(LoggingSystem.SYSTEM_PROPERTY);
			
			// LoggingSystem.NONE = "none"
			if (LoggingSystem.NONE.equals(loggingSystem)) {
				return ConditionOutcome.noMatch(message.because("system property "
						+ LoggingSystem.SYSTEM_PROPERTY + " is set to none"));
			}
			return ConditionOutcome.match(message.because("enabled"));
		}

	}
}
```
### (3). LoggersEndpoint源码
```
// *****************************************************
// 2. http://ip:port/actuator/loggers
// *****************************************************
@Endpoint(id = "loggers")
public class LoggersEndpoint {

	private final LoggingSystem loggingSystem;

	// 1. 注入:LoggingSystem
	public LoggersEndpoint(LoggingSystem loggingSystem) {
		Assert.notNull(loggingSystem, "LoggingSystem must not be null");
		this.loggingSystem = loggingSystem;
	}

	// GET http://ip:port/actuator/loggers
	@ReadOperation
	public Map<String, Object> loggers() {
		Collection<LoggerConfiguration> configurations = this.loggingSystem
				.getLoggerConfigurations();
		if (configurations == null) {
			return Collections.emptyMap();
		}
		Map<String, Object> result = new LinkedHashMap<>();
		result.put("levels", getLevels());
		result.put("loggers", getLoggers(configurations));
		return result;
	}

	// GET  http://ip:port/actuator/loggers/help.lixin 
	@ReadOperation
	public LoggerLevels loggerLevels(@Selector String name) {
		Assert.notNull(name, "Name must not be null");
		LoggerConfiguration configuration = this.loggingSystem
				.getLoggerConfiguration(name);
		return (configuration != null) ? new LoggerLevels(configuration) : null;
	}

	// POST  http://ip:port/actuator/loggers/help.lixin  {configuredLevel:xxx,effectiveLevel:xxx}
	@WriteOperation
	public void configureLogLevel(@Selector String name,
			@Nullable LogLevel configuredLevel) {
		Assert.notNull(name, "Name must not be empty");
		this.loggingSystem.setLogLevel(name, configuredLevel);
	}

	private NavigableSet<LogLevel> getLevels() {
		Set<LogLevel> levels = this.loggingSystem.getSupportedLogLevels();
		return new TreeSet<>(levels).descendingSet();
	}

	private Map<String, LoggerLevels> getLoggers(
			Collection<LoggerConfiguration> configurations) {
		Map<String, LoggerLevels> loggers = new LinkedHashMap<>(configurations.size());
		for (LoggerConfiguration configuration : configurations) {
			loggers.put(configuration.getName(), new LoggerLevels(configuration));
		}
		return loggers;
	}
}
```
### (4). 总结
> Actuator针对日志这一块的操作是不是so easy...