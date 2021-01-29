---
layout: post
title: 'Seata SeataAutoConfiguration(一)'
date: 2021-01-28
author: 李新
tags: Seata源码
---
### (1). seata-spring-boot-starter(spring.factories)
> 查看seata是如何与Spring Boot结合的(找到入口文件):    
> seata-spring-boot-starter/META-INF/spring.factories   

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
io.seata.spring.boot.autoconfigure.SeataAutoConfiguration,\
io.seata.spring.boot.autoconfigure.HttpAutoConfiguration
```

### (2). HttpAutoConfiguration
> HttpAutoConfiguration类的主要职责:   
> 1. 它实现了Spring MVC的Interceptor:org.springframework.web.servlet.handler.HandlerInterceptorAdapter.   
> 2. preHandle方法,拦截Http请求头(TX_XID),并设置到当前的线程上下文中(ThreadLocal).  
> 2. postHandle方法,清除ThreadLocal中的(TX_XID).   

```
@Configuration
@ConditionalOnWebApplication
public class HttpAutoConfiguration extends WebMvcConfigurerAdapter {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new TransactionPropagationIntercepter());
    }

    @Override
    public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> exceptionResolvers) {
        exceptionResolvers.add(new HttpHandlerExceptionResolver());
    }

}
```
### (3). SeataProperties
> SeataProperties是Seata的配置信息载体.其中,有几个方法需要特别注意:  
> 1. setApplicationId/getApplicationId     
> 2. setTxServiceGroup/getTxServiceGroup    
> 3. 注意看下这两个的实现类:SpringCloudAlibabaConfiguration

```
// SEATA_PREFIX = "seata"
@ConfigurationProperties(prefix = SEATA_PREFIX)
@EnableConfigurationProperties(SpringCloudAlibabaConfiguration.class)
public class SeataProperties {
	private boolean enabled = true;
	private String applicationId;
	private String txServiceGroup;
	private boolean enableAutoDataSourceProxy = true;
	private String dataSourceProxyMode = DefaultValues.DEFAULT_DATA_SOURCE_PROXY_MODE;
	private boolean useJdkProxy = false;
	private String[] excludesForAutoProxying = {};
	
	
	@Autowired
	private SpringCloudAlibabaConfiguration springCloudAlibabaConfiguration;
	
	public String getApplicationId() {
		if (applicationId == null) {
			applicationId = springCloudAlibabaConfiguration.getApplicationId();
		}
		return applicationId;
	}

	public SeataProperties setApplicationId(String applicationId) {
		this.applicationId = applicationId;
		return this;
	}

	// ****************************************************************
	// 调用了:SpringCloudAlibabaConfiguration
	// ****************************************************************
	public String getTxServiceGroup() {
		if (txServiceGroup == null) {
			txServiceGroup = springCloudAlibabaConfiguration.getTxServiceGroup();
		}
		return txServiceGroup;
	}

	public SeataProperties setTxServiceGroup(String txServiceGroup) {
		this.txServiceGroup = txServiceGroup;
		return this;
	}

}
```
### (4). SpringCloudAlibabaConfiguration
```
@ConfigurationProperties(prefix = StarterConstants.SEATA_SPRING_CLOUD_ALIBABA_PREFIX)
public class SpringCloudAlibabaConfiguration implements ApplicationContextAware {

    private static final Logger LOGGER = LoggerFactory.getLogger(SpringCloudAlibabaConfiguration.class);
    private static final String SPRING_APPLICATION_NAME_KEY = "spring.application.name";
    private static final String DEFAULT_SPRING_CLOUD_SERVICE_GROUP_POSTFIX = "-seata-service-group";
    private String applicationId;
    private String txServiceGroup;
    private ApplicationContext applicationContext;


    public String getApplicationId() {
		// 如果用户没有自定义的配置:spring.cloud.alibaba.seata.applicationId
        if (applicationId == null) {
			// 则读取环境变量:spring.application.name
            applicationId = applicationContext.getEnvironment().getProperty(SPRING_APPLICATION_NAME_KEY);
        }
        return applicationId;
    }

    public String getTxServiceGroup() {
		// 如果用户没有自定义配置:spring.cloud.alibaba.seata.tx-service-group
        if (txServiceGroup == null) {
            String applicationId = getApplicationId();
            if (applicationId == null) {
                LOGGER.warn("{} is null, please set its value", SPRING_APPLICATION_NAME_KEY);
            }
			// 比如:account-seata-service-group
            txServiceGroup = applicationId + DEFAULT_SPRING_CLOUD_SERVICE_GROUP_POSTFIX;
        }
        return txServiceGroup;
    }

    public void setTxServiceGroup(String txServiceGroup) {
        this.txServiceGroup = txServiceGroup;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```
### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 
