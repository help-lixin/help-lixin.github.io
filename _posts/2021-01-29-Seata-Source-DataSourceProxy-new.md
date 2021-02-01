---
layout: post
title: 'Seata 分支事务处理之DataSourceProxy(一)'
date: 2021-01-29
author: 李新
tags: Seata源码
---

### (1). 先看下DataSourceProxy的类图
> RM的类图有点多,不过,对于Java开发人员都很清楚的几个类(DataSource/Connection/Statement/PreparedStatement).  
> RM就是以DataSource为切入点.   

!["RM类结构图解"](/assets/seata/imgs/seata-rm-DataSourceProxy.jpg)

### (2). 先看几个Seata开发的拦截器
```
// SeataRestTemplateInterceptor
// 给RestTemplate增加拦截器,当ThreadLocal(RootContext)中有xid时
// 把xid通过Http Header的方法进行传递(key=TX_XID , value=xxx)
public class SeataRestTemplateInterceptor implements ClientHttpRequestInterceptor {
	@Override
	public ClientHttpResponse intercept(HttpRequest httpRequest, byte[] bytes,
			ClientHttpRequestExecution clientHttpRequestExecution) throws IOException {
			
		HttpRequestWrapper requestWrapper = new HttpRequestWrapper(httpRequest);
		String xid = RootContext.getXID();
		if (!StringUtils.isEmpty(xid)) {
			requestWrapper.getHeaders().add(RootContext.KEY_XID, xid);
		}
		return clientHttpRequestExecution.execute(requestWrapper, bytes);
	}
} // end SeataRestTemplateInterceptor


//  SeataHandlerInterceptor
// 了解Spring MVC的应该知道:HandlerInterceptor是Spring MVC的拦截器
// 它的主要目的是:
// 1. 进入Controller方法之前(执行业务方法之前),如果Http Header中有:TX_XID,就把TX_XID的值,绑定到ThreadLocal(RootContext).
// 2. 离开Controller方法之后(执行业务方法之后),如果Http Header中有:TX_XID,就把ThreadLocal(RootContext)进行解绑.
public class SeataHandlerInterceptor implements HandlerInterceptor {

	private static final Logger log = LoggerFactory
			.getLogger(SeataHandlerInterceptor.class);

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
			Object handler) {
		String xid = RootContext.getXID();
		
		// 从请求头中,获得:TX_XID
		String rpcXid = request.getHeader(RootContext.KEY_XID);
		
		if (log.isDebugEnabled()) {
			log.debug("xid in RootContext {} xid in RpcContext {}", xid, rpcXid);
		}

		if (xid == null && rpcXid != null) {
			// 绑定
			RootContext.bind(rpcXid);
			if (log.isDebugEnabled()) {
				log.debug("bind {} to RootContext", rpcXid);
			}
		}
		return true;
	}

	@Override
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
			Object handler, Exception e) {

		// 从请求头中,获得:TX_XID
		String rpcXid = request.getHeader(RootContext.KEY_XID);

		if (StringUtils.isEmpty(rpcXid)) {
			return;
		}

		// 解绑
		String unbindXid = RootContext.unbind();
		if (log.isDebugEnabled()) {
			log.debug("unbind {} from RootContext", unbindXid);
		}
		
		if (!rpcXid.equalsIgnoreCase(unbindXid)) { // 如果解绑后与Http请求头的中的不同
			log.warn("xid in change during RPC from {} to {}", rpcXid, unbindXid);
			if (unbindXid != null) {
				// 重新绑定.
				RootContext.bind(unbindXid);
				log.warn("bind {} back to RootContext", unbindXid);
			}
		} // end 
	}
}// end
```

### (3). 看下业务是如何配置RM的

```
package io.seata.sample;

@SpringBootApplication
@EnableEurekaClient
public class StorageApplication {

    public static void main(String[] args) {
        SpringApplication.run(StorageApplication.class, args);
    }

    // 1. 创建需要代理的DataSource
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource() {
        DruidDataSource druidDataSource = new DruidDataSource();
        return druidDataSource;
    }

	// 2. 创建:DataSourceProxy,它代理真实的DataSource(DruidDataSource)
    @Primary
    @Bean("dataSourceProxy")
    public DataSourceProxy dataSourceProxy(DataSource dataSource) {
        return new DataSourceProxy(dataSource);
    }

	// 3. 为jdbcTemplate配置数据源为代理的DataSource
    @Bean("jdbcTemplate")
    @ConditionalOnBean(DataSourceProxy.class)
    public JdbcTemplate jdbcTemplate(DataSourceProxy dataSourceProxy) {
        return new JdbcTemplate(dataSourceProxy);
    }
}
```
### (4). new DataSourceProxy
```
// **********************************************************************
// domain
private static final String DEFAULT_RESOURCE_GROUP_ID = "DEFAULT";
private String resourceGroupId;
private String jdbcUrl;
private String dbType;
private String userName;

// client.rm.tableMetaCheckEnable = false
private static boolean ENABLE_TABLE_META_CHECKER_ENABLE = ConfigurationFactory.getInstance().getBoolean(
	ConfigurationKeys.CLIENT_TABLE_META_CHECK_ENABLE, DEFAULT_CLIENT_TABLE_META_CHECK_ENABLE);

// 表结构元数据检查间隔
private static final long TABLE_META_CHECKER_INTERVAL = 60000L;

// 定时任务
private final ScheduledExecutorService tableMetaExcutor = new ScheduledThreadPoolExecutor(1,
	new NamedThreadFactory("tableMetaChecker", 1, true));


// **********************************************************************
// 1. 
public DataSourceProxy(DataSource targetDataSource) {
	this(targetDataSource, DEFAULT_RESOURCE_GROUP_ID);
} //end DataSourceProxy

// 2. 
public DataSourceProxy(DataSource targetDataSource, String resourceGroupId) {
	// 检查targetDataSource不能是:SeataDataSourceProxy
	if (targetDataSource instanceof SeataDataSourceProxy) {
		LOGGER.info("Unwrap the target data source, because the type is: {}", targetDataSource.getClass().getName());
		targetDataSource = ((SeataDataSourceProxy) targetDataSource).getTargetDataSource();
	}
	

	this.targetDataSource = targetDataSource;
	
	// *************************************
	// 初始化
	// *************************************
	init(targetDataSource, resourceGroupId);
} //end DataSourceProxy


// 3.
private void init(DataSource dataSource, String resourceGroupId) {
	this.resourceGroupId = resourceGroupId;
	
	// 尝试的拿一个Connection
	try (Connection connection = dataSource.getConnection()) {
		// 从Connection中获得:jdbcurl
		jdbcUrl = connection.getMetaData().getURL();
		// 从Connection中获得:dbType
		// dbType
		dbType = JdbcUtils.getDbType(jdbcUrl);
		// 如果是Oracle获得:userName
		if (JdbcConstants.ORACLE.equals(dbType)) {
			userName = connection.getMetaData().getUserName();
		}
	} catch (SQLException e) {
		throw new IllegalStateException("can not init dataSource", e);
	}

	//	********************************************************************************
	// DefaultResourceManager.get().registerResource这一部份主要是把RM的信息向TC汇报.进行资源的注册.
	// 这一部份的内容,我留到下一节讲.
	// 这是TC的日志: 
	// RM register success,message:RegisterRMRequest{resourceIds='jdbc:mysql://127.0.0.1:3306/fescar', applicationId='storage-service', transactionServiceGroup='my_test_tx_group'},channel:[id: 0x34606fb9, L:/172.17.12.122:8091 - R:/172.17.12.122:49749],client version:1.4.0
	//	********************************************************************************
	DefaultResourceManager.get().registerResource(this);
	
	
	if (ENABLE_TABLE_META_CHECKER_ENABLE) { // false
		//  开启了表元数据检查,则创建定时任务去做处理.
		tableMetaExcutor.scheduleAtFixedRate(() -> {
			try (Connection connection = dataSource.getConnection()) {
				TableMetaCacheFactory.getTableMetaCache(DataSourceProxy.this.getDbType())
					.refresh(connection, DataSourceProxy.this.getResourceId());
			} catch (Exception ignore) {
			}
		}, 0, TABLE_META_CHECKER_INTERVAL, TimeUnit.MILLISECONDS);
	}

	// 设置ThreadLocal(RootContext)中的BranchType为:AT
	//Set the default branch type to 'AT' in the RootContext.
	RootContext.setDefaultBranchType(this.getBranchType());
} //end 


public BranchType getBranchType() {
	return BranchType.AT;
}// end getBranchType
```

### (5). 总结
> 1. DataSourceProxy需要包裹真实DataSource对象.  
> 2. DataSourceProxy在构建时,会向TC进行资源的注册(这部份专门用一小节来讲,会辅助UML进行分解).  
> 3. 可配置开启了表元数据检查.    