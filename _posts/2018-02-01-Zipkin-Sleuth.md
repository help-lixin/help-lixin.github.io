---
layout: post
title: 'Zipkin spring-cloud-starter-sleuth整合(四)'
date: 2018-02-01
author: 李新
tags: Zipkin
---

### (1). 概述
> spring-cloud-starter-sleuth对brave进行了整合,把依赖加进来,增加配置即可:实现跨度汇报.  

### (2). 在Spring中基本都是规定了,我们要找到相应的EnableAutoConfiguration
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
		<dependency>
			<groupId>io.zipkin.reporter2</groupId>
			<artifactId>zipkin-reporter-bom</artifactId>
			<version>2.16.1</version>
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
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-sleuth</artifactId>
	</dependency>
</dependencies>
```
### (3). spring-cloud-sleuth-core-2.1.0.RELEASE.jar/META-INF/spring.factories
> 重点就是TraceAutoConfiguration.

```
# Auto Configuration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.sleuth.annotation.SleuthAnnotationAutoConfiguration,\
org.springframework.cloud.sleuth.sampler.SamplerAutoConfiguration,\
org.springframework.cloud.sleuth.autoconfig.TraceAutoConfiguration,\
org.springframework.cloud.sleuth.log.SleuthLogAutoConfiguration,\
org.springframework.cloud.sleuth.propagation.SleuthTagPropagationAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.web.TraceHttpAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.web.TraceWebAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.web.TraceWebServletAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.web.client.TraceWebClientAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.web.client.TraceWebAsyncClientAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.async.AsyncAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.async.AsyncCustomAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.async.AsyncDefaultAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.scheduling.TraceSchedulingAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.web.client.feign.TraceFeignClientAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.hystrix.SleuthHystrixAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.rxjava.RxJavaAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.reactor.TraceReactorAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.web.TraceWebFluxAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.zuul.TraceZuulAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.grpc.TraceGrpcAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.messaging.TraceMessagingAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.messaging.TraceSpringIntegrationAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.messaging.websocket.TraceWebSocketAutoConfiguration,\
org.springframework.cloud.sleuth.instrument.opentracing.OpentracingAutoConfiguration

# Environment Post Processor
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.cloud.sleuth.autoconfig.TraceEnvironmentPostProcessor
```

### (4). TraceAutoConfiguration
```
public class TraceAutoConfiguration {

	/**
	 * Tracer bean name. Name of the bean matters for some instrumentations.
	 */
	public static final String TRACER_BEAN_NAME = "tracer";

	/**
	 * Default value used for service name if none provided.
	 */
	public static final String DEFAULT_SERVICE_NAME = "default";

	@Autowired(required = false)
	List<Reporter<zipkin2.Span>> spanReporters = new ArrayList<>();

	@Autowired(required = false)
	List<SpanAdjuster> spanAdjusters = new ArrayList<>();

	@Autowired(required = false)
	List<FinishedSpanHandler> finishedSpanHandlers = new ArrayList<>();

	@Autowired(required = false)
	List<CurrentTraceContext.ScopeDecorator> scopeDecorators = new ArrayList<>();

	@Autowired(required = false)
	ExtraFieldPropagation.FactoryBuilder extraFieldPropagationFactoryBuilder;

	// **************************************************************
	// 看到这个类,我相信你应该能看懂了.
	// 要注意:Sampler,否则你会发现半天都不进行汇报逻辑.
	// 那么到这时候,你肯定会好奇,那Reporter来自于哪?
	// 只要增加spring-cloud-starter-zipkin依赖进来,并配置即会启用相应的Reporter实现类.
	// 当然,你也可以自定义:Reporter,我这里自定义了一个测试.
	// **************************************************************
	@Bean
	@ConditionalOnMissingBean
	// NOTE: stable bean name as might be used outside sleuth
	Tracing tracing(
			@Value("${spring.zipkin.service.name:${spring.application.name:default}}") String serviceName,
			Propagation.Factory factory, CurrentTraceContext currentTraceContext,
			Sampler sampler, ErrorParser errorParser, SleuthProperties sleuthProperties) {
		Tracing.Builder builder = Tracing.newBuilder().sampler(sampler)
				.errorParser(errorParser)
				.localServiceName(StringUtils.isEmpty(serviceName) ? DEFAULT_SERVICE_NAME
						: serviceName)
				.propagationFactory(factory).currentTraceContext(currentTraceContext)
				.spanReporter(
						new CompositeReporter(this.spanAdjusters, this.spanReporters))
				.traceId128Bit(sleuthProperties.isTraceId128())
				.supportsJoin(sleuthProperties.isSupportsJoin());
		for (FinishedSpanHandler finishedSpanHandlerFactory : this.finishedSpanHandlers) {
			builder.addFinishedSpanHandler(finishedSpanHandlerFactory);
		}
		return builder.build();
	}

	// 典型的组合模式
	private static class CompositeReporter implements Reporter<zipkin2.Span> {

		private static final Log log = LogFactory.getLog(CompositeReporter.class);

		private final List<SpanAdjuster> spanAdjusters;

		private final Reporter<zipkin2.Span> spanReporter;

		private CompositeReporter(List<SpanAdjuster> spanAdjusters,
				List<Reporter<Span>> spanReporters) {
			this.spanAdjusters = spanAdjusters;
			this.spanReporter = spanReporters.size() == 1 ? spanReporters.get(0)
					: new ListReporter(spanReporters);
		}

		private static class ListReporter implements Reporter<zipkin2.Span> {

			private final List<Reporter<Span>> spanReporters;

			private ListReporter(List<Reporter<Span>> spanReporters) {
				this.spanReporters = spanReporters;
			}

			@Override
			public void report(Span span) {
				// 当有多个:Reporter,每个Reporter都进行汇报
				for (Reporter<zipkin2.Span> spanReporter : this.spanReporters) {
					try {
						spanReporter.report(span);
					}
					catch (Exception ex) {
						log.warn("Exception occurred while trying to report the span "
								+ span, ex);
					}
				}
			}

			@Override
			public String toString() {
				return "ListReporter{" + "spanReporters=" + this.spanReporters + '}';
			}

		}

		@Override
		public void report(Span span) {
			Span spanToAdjust = span;
			for (SpanAdjuster spanAdjuster : this.spanAdjusters) {
				spanToAdjust = spanAdjuster.adjust(spanToAdjust);
			}
			// 单个Reporter进行汇报
			this.spanReporter.report(spanToAdjust);
		}

		@Override
		public String toString() {
			return "CompositeReporter{" + "spanAdjusters=" + this.spanAdjusters
					+ ", spanReporters=" + this.spanReporter + '}';
		}

	}

	@Bean(name = TRACER_BEAN_NAME)
	@ConditionalOnMissingBean
	Tracer tracer(Tracing tracing) {
		return tracing.tracer();
	}

	// ********************************************************
	// 采集器,只有采集器允许的情况下,才会进行采集,
	// 要注意下,你可以自定义(建议定义,因为,默认是不采集)
	// Sampler.NEVER_SAMPLE : 代表不采集
	// ********************************************************
	@Bean
	@ConditionalOnMissingBean
	Sampler sleuthTraceSampler() {
		return Sampler.NEVER_SAMPLE;
	}
	
	@Bean
	@ConditionalOnMissingBean
	SpanNamer sleuthSpanNamer() {
		return new DefaultSpanNamer();
	}

	@Bean
	@ConditionalOnMissingBean
	Propagation.Factory sleuthPropagation(SleuthProperties sleuthProperties) {
		if (sleuthProperties.getBaggageKeys().isEmpty()
				&& sleuthProperties.getPropagationKeys().isEmpty()) {
			return B3Propagation.FACTORY;
		}
		ExtraFieldPropagation.FactoryBuilder factoryBuilder;
		if (this.extraFieldPropagationFactoryBuilder != null) {
			factoryBuilder = this.extraFieldPropagationFactoryBuilder;
		}
		else {
			factoryBuilder = ExtraFieldPropagation
					.newFactoryBuilder(B3Propagation.FACTORY);
		}
		if (!sleuthProperties.getBaggageKeys().isEmpty()) {
			factoryBuilder = factoryBuilder
					// for HTTP
					.addPrefixedFields("baggage-", sleuthProperties.getBaggageKeys())
					// for messaging
					.addPrefixedFields("baggage_", sleuthProperties.getBaggageKeys());
		}
		if (!sleuthProperties.getPropagationKeys().isEmpty()) {
			for (String key : sleuthProperties.getPropagationKeys()) {
				factoryBuilder = factoryBuilder.addField(key);
			}
		}
		return factoryBuilder.build();
	}

	@Bean
	CurrentTraceContext sleuthCurrentTraceContext(CurrentTraceContext.Builder builder) {
		for (CurrentTraceContext.ScopeDecorator scopeDecorator : this.scopeDecorators) {
			builder.addScopeDecorator(scopeDecorator);
		}
		return builder.build();
	}

	@Bean
	@ConditionalOnMissingBean
	CurrentTraceContext.Builder sleuthCurrentTraceContextBuilder() {
		return ThreadLocalCurrentTraceContext.newBuilder();
	}

	@Bean
	@ConditionalOnMissingBean
	ReporterMetrics sleuthReporterMetrics() {
		return new InMemoryReporterMetrics();
	}

	@Bean
	@ConditionalOnMissingBean
	Reporter<zipkin2.Span> noOpSpanReporter() {
		return Reporter.NOOP;
	}

	@Bean
	@ConditionalOnMissingBean
	ErrorParser errorParser() {
		return new ErrorParser();
	}

	@Bean
	@ConditionalOnMissingBean
	// NOTE: stable bean name as might be used outside sleuth
	CurrentSpanCustomizer spanCustomizer(Tracing tracing) {
		return CurrentSpanCustomizer.create(tracing);
	}

}
```
### (5). ProviderApplication
```
package help.lixin.zipkin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;

import brave.sampler.CountingSampler;
import brave.sampler.Sampler;
import zipkin2.Span;
import zipkin2.reporter.Reporter;

@SpringBootApplication
public class ProviderApplication {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(ProviderApplication.class, args);
	}
	
	// 我重写了:Sampler
	@Bean
	@ConditionalOnMissingBean
	Sampler sleuthTraceSampler() {
		return CountingSampler.create(0.08f);
	}
	
	// 自定义Reporter
	@Bean
	public Reporter<zipkin2.Span> reporter(){
		return new Reporter<Span>() {
			public void report(Span span) {
				// 打印span
				// {"traceId":"971caad7ee3fb517","id":"971caad7ee3fb517","kind":"SERVER","name":"get /hello","timestamp":1614243859694274,"duration":71699,"localEndpoint":{"serviceName":"test-provider","ipv4":"172.17.0.253"},"remoteEndpoint":{"ipv6":"::1","port":51671},"tags":{"http.method":"GET","http.path":"/hello","mvc.controller.class":"HelloController","mvc.controller.method":"hello"}}
				System.out.println(span.toString());
			}
		};
	}
}

```
### (6). 总结
> spring-cloud-starter-sleuth对brave进行了整合,只要把依赖加入就可以实现数据汇报了.   
> Reporter的实现在相应的jar包里(spring-cloud-starter-zipkin).   
> 也可以自己实现:Reporter要注意一点,这个方法是同步的,建议用异步(AsyncReporter$BoundedAsyncReporter)和自定义Sender结合.  