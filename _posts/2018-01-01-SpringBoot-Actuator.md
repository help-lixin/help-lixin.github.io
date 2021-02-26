---
layout: post
title: 'Spring Boot Actuator'
date: 2018-01-01
author: 李新
tags: SpringBoot
---

### (1). Spring Boot Actuator是什么?
>  它是监控系统健康情况的工具.
### (2). 添加依赖
```
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>Greenwich.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-dependencies</artifactId>
			<version>2.1.0.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-netflix</artifactId>
			<version>2.1.1.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-actuator</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```

### (3). spring-boot-actuator-autoconfigure-2.1.0.RELEASE.jar/META-INF/spring.factories
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.actuate.autoconfigure.amqp.RabbitHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.audit.AuditAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.audit.AuditEventsEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.beans.BeansEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.cache.CachesEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.cassandra.CassandraHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.cassandra.CassandraReactiveHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.cloudfoundry.servlet.CloudFoundryActuatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.cloudfoundry.reactive.ReactiveCloudFoundryActuatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.condition.ConditionsReportEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.context.properties.ConfigurationPropertiesReportEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.context.ShutdownEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.couchbase.CouchbaseHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.couchbase.CouchbaseReactiveHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.elasticsearch.ElasticSearchClientHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.elasticsearch.ElasticSearchJestHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.endpoint.EndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.endpoint.jmx.JmxEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.endpoint.web.WebEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.env.EnvironmentEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.flyway.FlywayEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.health.HealthEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.health.HealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.influx.InfluxDbHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.info.InfoContributorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.info.InfoEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.integration.IntegrationGraphEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.jdbc.DataSourceHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.jms.JmsHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.jolokia.JolokiaEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.ldap.LdapHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.liquibase.LiquibaseEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.logging.LogFileWebEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.logging.LoggersEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.mail.MailHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.management.HeapDumpWebEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.management.ThreadDumpEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.CompositeMeterRegistryAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.JvmMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.KafkaMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.Log4J2MetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.LogbackMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.MetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.MetricsEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.SystemMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.amqp.RabbitMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.cache.CacheMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.appoptics.AppOpticsMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.atlas.AtlasMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.datadog.DatadogMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.dynatrace.DynatraceMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.elastic.ElasticMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.ganglia.GangliaMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.graphite.GraphiteMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.humio.HumioMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.influx.InfluxMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.jmx.JmxMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.kairos.KairosMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.newrelic.NewRelicMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.prometheus.PrometheusMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.signalfx.SignalFxMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.simple.SimpleMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.statsd.StatsdMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.export.wavefront.WavefrontMetricsExportAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.jdbc.DataSourcePoolMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.jersey.JerseyServerMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.orm.jpa.HibernateMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.web.client.HttpClientMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.web.jetty.JettyMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.web.reactive.WebFluxMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.web.servlet.WebMvcMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.metrics.web.tomcat.TomcatMetricsAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.mongo.MongoHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.mongo.MongoReactiveHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.neo4j.Neo4jHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.redis.RedisHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.redis.RedisReactiveHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.scheduling.ScheduledTasksEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.security.reactive.ReactiveManagementWebSecurityAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.security.servlet.ManagementWebSecurityAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.session.SessionsEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.solr.SolrHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.system.DiskSpaceHealthIndicatorAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.trace.http.HttpTraceAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.trace.http.HttpTraceEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.web.mappings.MappingsEndpointAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.web.reactive.ReactiveManagementContextAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.web.server.ManagementContextAutoConfiguration,\
org.springframework.boot.actuate.autoconfigure.web.servlet.ServletManagementContextAutoConfiguration
org.springframework.boot.actuate.autoconfigure.web.ManagementContextConfiguration=\
org.springframework.boot.actuate.autoconfigure.endpoint.web.ServletEndpointManagementContextConfiguration,\
org.springframework.boot.actuate.autoconfigure.endpoint.web.reactive.WebFluxEndpointManagementContextConfiguration,\
org.springframework.boot.actuate.autoconfigure.endpoint.web.servlet.WebMvcEndpointManagementContextConfiguration,\
org.springframework.boot.actuate.autoconfigure.endpoint.web.jersey.JerseyWebEndpointManagementContextConfiguration,\
org.springframework.boot.actuate.autoconfigure.web.jersey.JerseyManagementChildContextConfiguration,\
org.springframework.boot.actuate.autoconfigure.web.reactive.ReactiveManagementChildContextConfiguration,\
org.springframework.boot.actuate.autoconfigure.web.servlet.ServletManagementChildContextConfiguration,\
org.springframework.boot.actuate.autoconfigure.web.servlet.WebMvcEndpointChildContextConfiguration

org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.actuate.autoconfigure.metrics.MissingRequiredConfigurationFailureAnalyzer

```

### (4). Actuator入口配置类(WebEndpointProperties)
```
@ConfigurationProperties(prefix = "management.endpoints.web")
public class WebEndpointProperties {

	private final Exposure exposure = new Exposure();

	/**
	 * Base path for Web endpoints. Relative to server.servlet.context-path or
	 * management.server.servlet.context-path if management.server.port is configured.
	 */
	private String basePath = "/actuator";

	/**
	 * Mapping between endpoint IDs and the path that should expose them.
	 */
	private final Map<String, String> pathMapping = new LinkedHashMap<>();

	public Exposure getExposure() {
		return this.exposure;
	}

	public String getBasePath() {
		return this.basePath;
	}

	public void setBasePath(String basePath) {
		Assert.isTrue(basePath.isEmpty() || basePath.startsWith("/"),
				"Base path must start with '/' or be empty");
		this.basePath = cleanBasePath(basePath);
	}

	private String cleanBasePath(String basePath) {
		if (StringUtils.hasText(basePath) && basePath.endsWith("/")) {
			return basePath.substring(0, basePath.length() - 1);
		}
		return basePath;
	}

	public Map<String, String> getPathMapping() {
		return this.pathMapping;
	}

	public static class Exposure {

		/**
		 * Endpoint IDs that should be included or '*' for all.
		 */
		private Set<String> include = new LinkedHashSet<>();

		/**
		 * Endpoint IDs that should be excluded or '*' for all.
		 */
		private Set<String> exclude = new LinkedHashSet<>();

		public Set<String> getInclude() {
			return this.include;
		}

		public void setInclude(Set<String> include) {
			this.include = include;
		}

		public Set<String> getExclude() {
			return this.exclude;
		}

		public void setExclude(Set<String> exclude) {
			this.exclude = exclude;
		}

	}
}

```
### (5). 挑两个简单的一点的来学习
> ShutdownEndpointAutoConfiguration/ShutdownEndpoint.  
> ThreadDumpEndpointAutoConfiguration/ThreadDumpEndpoint

### (6). ShutdownEndpointAutoConfiguration/ShutdownEndpoint.  
```
@Configuration
public class ShutdownEndpointAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean
	// ***************************************************
	// 1. Endpoint必须要加上这个注解
	// ***************************************************
	@ConditionalOnEnabledEndpoint
	public ShutdownEndpoint shutdownEndpoint() {
		return new ShutdownEndpoint();
	}

} // end ShutdownEndpointAutoConfiguration

// ***************************************
// 2. 定义Endpoint的信息,默认是否启动
// ***************************************
@Endpoint(id = "shutdown", enableByDefault = false)
public class ShutdownEndpoint implements ApplicationContextAware {

	private static final Map<String, String> NO_CONTEXT_MESSAGE = Collections
			.unmodifiableMap(
					Collections.singletonMap("message", "No context to shutdown."));

	private static final Map<String, String> SHUTDOWN_MESSAGE = Collections
			.unmodifiableMap(
					Collections.singletonMap("message", "Shutting down, bye..."));

	private ConfigurableApplicationContext context;

	// ********************************************************
	// 3. 这是一个写操作,没有任何的参数
	// ********************************************************
	@WriteOperation
	public Map<String, String> shutdown() {
		if (this.context == null) {
			return NO_CONTEXT_MESSAGE;
		}
		try {
			return SHUTDOWN_MESSAGE;
		}
		finally {
			// *************************************************
			// 4. 所谓的关闭容器,实际就是:ApplicationContext.close方法
			// *************************************************
			Thread thread = new Thread(this::performShutdown);
			thread.setContextClassLoader(getClass().getClassLoader());
			thread.start();
		}
	}

	private void performShutdown() {
		try {
			Thread.sleep(500L);
		}
		catch (InterruptedException ex) {
			Thread.currentThread().interrupt();
		}
		this.context.close();
	}
} // end ShutdownEndpoint

```
### (7). ThreadDumpEndpointAutoConfiguration/ThreadDumpEndpoint
```
@Configuration
public class ThreadDumpEndpointAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean
	// **************************************************
	// 1. Endpoint必须要定义这个注解
	// **************************************************
	@ConditionalOnEnabledEndpoint
	public ThreadDumpEndpoint dumpEndpoint() {
		return new ThreadDumpEndpoint();
	}

} // end ThreadDumpEndpointAutoConfiguration


// *************************************************
// 2. Endpoint的描术信息
// *************************************************
@Endpoint(id = "threaddump")
public class ThreadDumpEndpoint {
	
	// *************************************************
	// 3. 这是一个读操作,不需要任何的参数
	// *************************************************
	@ReadOperation
	public ThreadDumpDescriptor threadDump() {
		// *************************************************
		// 4. 所谓的线程dump调用了:ManagementFactory.getThreadMXBean()
		// *************************************************
		return new ThreadDumpDescriptor(Arrays
				.asList(ManagementFactory.getThreadMXBean().dumpAllThreads(true, true)));
	}

	
	public static final class ThreadDumpDescriptor {

		private final List<ThreadInfo> threads;

		private ThreadDumpDescriptor(List<ThreadInfo> threads) {
			this.threads = threads;
		}

		public List<ThreadInfo> getThreads() {
			return this.threads;
		}

	}
}// end ThreadDumpEndpoint

```
### (8). 总结
> spring.factories定义了大量的AutoConfiguration(Endpoint).若自定义Endpotin有两个很重要的步骤:    
> 1. @ConditionalOnEnabledEndpoint   
> 2. @Endpoint(id = "xxxx")   
> 3. @ReadOperation/@WriteOperation  