---
layout: post
title: 'Servicecomb Pack之全局事务是如何传播的(六)'
date: 2021-05-04
author: 李新
tags:  Servicecomb-Pack
---

### (1). 前言
> 前面剖析了全局事务的启动过程,可是分支事务和全局事务在不同的进程里,分支事务与全局事务是如何关联的呢?
> 核心逻辑就在:omega-transport工程里(其实,你会发现跟ZipKin好像.)    

```
# 整个传输层,是一工程层抽象,可以根据项目的要求:依赖不同的实现(dubbo/feig/resttemplate...).
# 我这里依赖的是:omega-transport-resttemplate
lixin-macbook:omega lixin$ tree omega-transport -L 1
omega-transport
├── omega-transport-dubbo
├── omega-transport-feign
├── omega-transport-hystrix
├── omega-transport-resttemplate
├── omega-transport-servicecomb
├── omega-transport.iml
├── pom.xml
└── target
```
### (2). 如何入手?
> 查看omega-transport-resttemplate工程信息,我们知道,这是一个Spring Boot的工程,Spring Boot在启动时,会通过SPI加载:spring.factories里的内容,并实例化配置的类.     
> <font color='red'>RestTemplateConfig用于传播全局事务ID到分支事务里.</font>  
> <font color='red'>WebConfig用于从Http协议头,获取全局事务ID,绑定到当前线程里.</font>  

!["omega-transport-resttemplate"](/assets/servicecomb-pack/imgs/saga-omega-transport-resttemplate.png)

### (3). RestTemplateConfig
> 创建:RestTemplate实例,并为它配置拦截器.  

```
@Configuration
public class RestTemplateConfig {

  // 1. 创建:RestTemplate实例
  @Bean(name = "omegaRestTemplate")
  public RestTemplate omegaRestTemplate(
          // 注入:OmegaContext
          @Autowired(required = false) OmegaContext context,
		  RestTemplateBuilder restTemplateBuilder) {
    return restTemplateBuilder
	     // 添加拦截器:TransactionClientHttpRequestInterceptor
        .additionalInterceptors(new TransactionClientHttpRequestInterceptor(context))
        .build();
  }// end 
}
```
### (4). TransactionClientHttpRequestInterceptor
> 拦截RestTemplate的所有请求,当线程上下文中存在全局事务ID(globalTxId),则,把全局事务ID(globalTxId)和本地事务ID(localTxId),添加到HTTP协议头里.   

```
class TransactionClientHttpRequestInterceptor implements org.springframework.http.client.ClientHttpRequestInterceptor {
  private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());
  private final OmegaContext omegaContext;

  TransactionClientHttpRequestInterceptor(OmegaContext omegaContext) {
    this.omegaContext = omegaContext;
  }

  @Override
  public ClientHttpResponse intercept(
                            HttpRequest request, byte[] body,
							ClientHttpRequestExecution execution) throws IOException {
	
	//线程上下文(OmegaContext)里的全局事务ID,如果,存在全局事务ID的话
    // 为HttpRequest添加协议头(X-Pack-Global-Transaction-Id/X-Pack-Local-Transaction-Id)
    if (omegaContext!= null && omegaContext.globalTxId() != null) {
      request.getHeaders().add(GLOBAL_TX_ID_KEY, omegaContext.globalTxId());
      request.getHeaders().add(LOCAL_TX_ID_KEY, omegaContext.localTxId());
      
      LOG.debug("Added {} {} and {} {} to request header",
          GLOBAL_TX_ID_KEY,
          omegaContext.globalTxId(),
          LOCAL_TX_ID_KEY,
          omegaContext.localTxId());
    } else {
      LOG.debug("Cannot inject transaction ID, as the OmegaContext is null or cannot get the globalTxId.");
    }
	
    return execution.execute(request, body);
  }
  
}
```
### (5). WebConfig
> 为Spring MVC项目,添加拦截器.

```
@Configuration
@EnableWebMvc
// 1. 给Spring的项目添加:Interceptor
//    为什么不直接添加Filter,如果集成的项目是Struts或者原生Servlet,是否有做考虑?  
public class WebConfig extends WebMvcConfigurerAdapter {

  private final OmegaContext omegaContext;

  private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());


  @Autowired
  public WebConfig(@Autowired(required=false) OmegaContext omegaContext) {
    this.omegaContext = omegaContext;
  }

  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    if (omegaContext == null) {
      LOG.info("The OmegaContext is not injected, The transaction handler is disabled");
    }
	// ************************************************************
	// 为Spring的项目,添加拦截器.
	// ************************************************************
    registry.addInterceptor(new TransactionHandlerInterceptor(omegaContext));
  }
}
```
### (6). TransactionHandlerInterceptor
> TransactionHandlerInterceptor实际是在分支事务侧来着的,它会拦截所有的MVC请求,然后,获得全局事务ID和分支事务ID绑定到当前线程里.  

```
// HandlerInterceptor为Spring的拦截器
class TransactionHandlerInterceptor implements org.springframework.web.servlet.HandlerInterceptor {

  private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());

  private final OmegaContext omegaContext;

  TransactionHandlerInterceptor(OmegaContext omegaContext) {
    this.omegaContext = omegaContext;
  }

  //  请求Halder之前,走该方法
  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
	// 从Http的协议头(X-Pack-Global-Transaction-Id)里,获得全局事务ID,如果全局事务ID存在,则绑定到线程上下文里(OmegaContext)
    if (omegaContext != null) {
      String globalTxId = request.getHeader(GLOBAL_TX_ID_KEY);
      if (globalTxId == null) {
        LOG.debug("Cannot inject transaction ID, no such header: {}", GLOBAL_TX_ID_KEY);
      } else {
		// 绑定全局事务ID和分支事务ID到线程上下文里.
        omegaContext.setGlobalTxId(globalTxId);
        omegaContext.setLocalTxId(request.getHeader(LOCAL_TX_ID_KEY));
      }
    } else {
      LOG.debug("Cannot inject transaction ID, as the OmegaContext is null.");
    }
    return true;
  }

  @Override
  public void postHandle(HttpServletRequest request, HttpServletResponse response, Object o, ModelAndView mv) {
  }

  // 请求目标方法结束以后,会走该方法
  @Override
  public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object o, Exception e) {
    if (omegaContext != null) {
		// **********************************************
		// 方法结束后,清场
		// **********************************************
      omegaContext.clear();
    }
  }
}
```
### (7). 总结
> <font color='red'>TransactionClientHttpRequestInterceptor负责把全局事务和本地事务,传播(添加HTTP协议头)到分支事务里.</font>  
> <font color='red'>TransactionHandlerInterceptor负责从HTTP协议头里取出,上面传递的全局事务和本地事务,绑定到当前线程上下文里.</font>  