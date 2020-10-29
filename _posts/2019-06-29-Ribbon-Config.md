---
layout: post
title: 'Ribbon源码(RibbonAutoConfiguration)'
date: 2019-06-29
author: 李新
tags: Ribbon
---

### (1).Ribbon配置(@LoadBalanced)
```
@EnableDiscoveryClient
@EnableEurekaClient
@SpringBootApplication
public class ConsumerApplication {

	@Bean
    // 启用负载均衡
	@LoadBalanced
	RestTemplate restTemplate() {
		ClientHttpRequestInterceptor basicAuthenticationInterceptor = new BasicAuthenticationInterceptor("lixin", "123456");
		List<ClientHttpRequestInterceptor> interceptors = new ArrayList<ClientHttpRequestInterceptor>();
		interceptors.add(basicAuthenticationInterceptor);
		RestTemplate restTemplate = new RestTemplate();
		restTemplate.setInterceptors(interceptors);
		return restTemplate;
	}

	public static void main(String[] args) {
		SpringApplication.run(ConsumerApplication.class, args);
	}
}
```
### (2).RibbonAutoConfiguration
```
@Configuration
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)
// **************************Ribbon的配置类*********************************
// @RibbonClient会找到所有的@RibbonClients注解信息,并注入到Spring的Bean中
//  在SpringClientFactory.createContext会用到这些配置信息
// @Import(RibbonClientConfigurationRegistrar.class)
@RibbonClients
// **************************Ribbon的配置类*********************************

// 在EurekaClientAutoConfiguration这个配置文件之后初始化当前配置
@AutoConfigureAfter(name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
// *************************************************************************
// 在实例化当前配置之前,先执行:LoadBalancerAutoConfiguration/AsyncLoadBalancerAutoConfiguration
@AutoConfigureBefore({LoadBalancerAutoConfiguration.class, AsyncLoadBalancerAutoConfiguration.class})
// *************************************************************************

// 启用两个配置文件: RibbonEagerLoadProperties/ServerIntrospectorProperties
@EnableConfigurationProperties({RibbonEagerLoadProperties.class, ServerIntrospectorProperties.class})
public class RibbonAutoConfiguration {

	@Autowired(required = false)
	private List<RibbonClientSpecification> configurations = new ArrayList<>();

	@Autowired
	private RibbonEagerLoadProperties ribbonEagerLoadProperties;

	@Bean
	public HasFeatures ribbonFeature() {
		return HasFeatures.namedFeature("Ribbon", Ribbon.class);
	}

   // 1. 创建SpringClientFactory(包含有两个配置类: RibbonAutoConfiguration |  RibbonEurekaAutoConfiguration )
	@Bean
	public SpringClientFactory springClientFactory() {
		SpringClientFactory factory = new SpringClientFactory();
           // RibbonAutoConfiguration |  RibbonEurekaAutoConfiguration
           // RibbonClientSpecification{name='default.org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration', configuration=[]}
           // RibbonClientSpecification{name='default.org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration', configuration=[class org.springframework.cloud.netflix.ribbon.eureka.EurekaRibbonClientConfiguration]}
		factory.setConfigurations(this.configurations);
		return factory;
	}

    // 2. 创建负载均衡Client.并包含着SprginClientFactory
	@Bean
	@ConditionalOnMissingBean(LoadBalancerClient.class)
	public LoadBalancerClient loadBalancerClient() {
		return new RibbonLoadBalancerClient(springClientFactory());
	}

	@Bean
	@ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
	@ConditionalOnMissingBean
	public LoadBalancedRetryFactory loadBalancedRetryPolicyFactory(final SpringClientFactory clientFactory) {
		return new RibbonLoadBalancedRetryFactory(clientFactory);
	}

   // 3. 创建PropertiesFactory工厂
	@Bean
	@ConditionalOnMissingBean
	public PropertiesFactory propertiesFactory() {
		return new PropertiesFactory();
	}

	@Bean
	@ConditionalOnProperty(value = "ribbon.eager-load.enabled")
	public RibbonApplicationContextInitializer ribbonApplicationContextInitializer() {
		return new RibbonApplicationContextInitializer(springClientFactory(),
				ribbonEagerLoadProperties.getClients());
	}

	@Configuration
	@ConditionalOnClass(HttpRequest.class)
	@ConditionalOnRibbonRestClient
	protected static class RibbonClientHttpRequestFactoryConfiguration {

		@Autowired
		private SpringClientFactory springClientFactory;

		@Bean
		public RestTemplateCustomizer restTemplateCustomizer(
				final RibbonClientHttpRequestFactory ribbonClientHttpRequestFactory) {
			return restTemplate -> restTemplate.setRequestFactory(ribbonClientHttpRequestFactory);
		}

		@Bean
		public RibbonClientHttpRequestFactory ribbonClientHttpRequestFactory() {
			return new RibbonClientHttpRequestFactory(this.springClientFactory);
		}
	}



	@Target({ ElementType.TYPE, ElementType.METHOD })
	@Retention(RetentionPolicy.RUNTIME)
	@Documented
	@Conditional(OnRibbonRestClientCondition.class)
	@interface ConditionalOnRibbonRestClient { }

	private static class OnRibbonRestClientCondition extends AnyNestedCondition {
		public OnRibbonRestClientCondition() {
			super(ConfigurationPhase.REGISTER_BEAN);
		}

		@Deprecated //remove in Edgware"
		@ConditionalOnProperty("ribbon.http.client.enabled")
		static class ZuulProperty {}

		@ConditionalOnProperty("ribbon.restclient.enabled")
		static class RibbonProperty {}
	}


	static class RibbonClassesConditions extends AllNestedConditions {

		RibbonClassesConditions() {
			super(ConfigurationPhase.PARSE_CONFIGURATION);
		}

		@ConditionalOnClass(IClient.class)
		static class IClientPresent {

		}

		@ConditionalOnClass(RestTemplate.class)
		static class RestTemplatePresent {

		}

		@ConditionalOnClass(AsyncRestTemplate.class)
		static class AsyncRestTemplatePresent {

		}

		@ConditionalOnClass(Ribbon.class)
		static class RibbonPresent {

		}

}
```
### (3).AsyncLoadBalancerAutoConfiguration
```
@Configuration
@ConditionalOnBean(LoadBalancerClient.class)
@ConditionalOnClass(AsyncRestTemplate.class)
public class AsyncLoadBalancerAutoConfiguration {


	@Configuration
	static class AsyncRestTemplateCustomizerConfig {
		@LoadBalanced
		@Autowired(required = false)
		private List<AsyncRestTemplate> restTemplates = Collections.emptyList();

           // 6.为AsyncRestTemplate配置拦截器
		@Bean
		public SmartInitializingSingleton loadBalancedAsyncRestTemplateInitializer(
				final List<AsyncRestTemplateCustomizer> customizers) {
			return new SmartInitializingSingleton() {
				@Override
				public void afterSingletonsInstantiated() {
					for (AsyncRestTemplate restTemplate : AsyncRestTemplateCustomizerConfig.this.restTemplates) {
						for (AsyncRestTemplateCustomizer customizer : customizers) {
							customizer.customize(restTemplate);
						}
					}
				}
			};
		}
	}

	@Configuration
	static class LoadBalancerInterceptorConfig {
           // 4.创建:AsyncLoadBalancerInterceptor
		@Bean
		public AsyncLoadBalancerInterceptor asyncLoadBalancerInterceptor(LoadBalancerClient loadBalancerClient) {
			return new AsyncLoadBalancerInterceptor(loadBalancerClient);
		}

           // 5. 对AsyncLoadBalancerInterceptor进行配置
		@Bean
		public AsyncRestTemplateCustomizer asyncRestTemplateCustomizer(
				final AsyncLoadBalancerInterceptor loadBalancerInterceptor) {
			return new AsyncRestTemplateCustomizer() {
				@Override
				public void customize(AsyncRestTemplate restTemplate) {
					List<AsyncClientHttpRequestInterceptor> list = new ArrayList<>(
							restTemplate.getInterceptors());
					list.add(loadBalancerInterceptor);
					restTemplate.setInterceptors(list);
				}
			};
		}
	}
}
```
### (4).LoadBalancerAutoConfiguration
```
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

   // 10. 为RestTemplate配置拦截器
	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer.customize(restTemplate);
                }
            }
        });
	}

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

   // 7.创建负载均衡请求工厂
	@Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
           // transformers = []
		return new LoadBalancerRequestFactory(loadBalancerClient, transformers);
	}

   
	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
     
           // 8.创建负载均衡拦截器(ClientHttpRequestInterceptor)
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

           // 9. 创建:RestTemplateCustomizer包含有所有的:ClientHttpRequestInterceptor集合
		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}

	@Configuration
	@ConditionalOnClass(RetryTemplate.class)
	public static class RetryAutoConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public LoadBalancedRetryFactory loadBalancedRetryFactory() {
			return new LoadBalancedRetryFactory() {};
		}
	}

	@Configuration
	@ConditionalOnClass(RetryTemplate.class)
	public static class RetryInterceptorAutoConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public RetryLoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient, LoadBalancerRetryProperties properties,
				LoadBalancerRequestFactory requestFactory,
				LoadBalancedRetryFactory loadBalancedRetryFactory) {
			return new RetryLoadBalancerInterceptor(loadBalancerClient, properties,
					requestFactory, loadBalancedRetryFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final RetryLoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}
}
```
