---
layout: post
title: 'CAS源码之ClientInfoThreadLocalFilter(四)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
在前一篇,分析CAS自定义了一堆Filter(注意:Filter是有顺序的),在这一篇开始按照顺序对Filter源码的剖析,在这一小篇,主要剖析:ClientInfoThreadLocalFilter

### (2). 看下ClientInfoThreadLocalFilter是在什么时机初始化的
```
@Configuration("casCoreAuditConfiguration")
@EnableAspectJAutoProxy(proxyTargetClass = true)
@EnableConfigurationProperties(CasConfigurationProperties.class)
@Slf4j
public class CasCoreAuditConfiguration {
	
	
	// **************************************************************************************
	// FilterRegistrationBean是Spring提供给开发使用,主要用来注册Filter的,Web容器(比如:Tomcat)会回调这些方法.
	// **************************************************************************************
	@Bean
	public FilterRegistrationBean casClientInfoLoggingFilter() {
		
		// cas.audit.engine
		val audit = casProperties.getAudit().getEngine();
		val bean = new FilterRegistrationBean<ClientInfoThreadLocalFilter>();
		// Filter
		bean.setFilter(new ClientInfoThreadLocalFilter());
		// 过滤的请求
		bean.setUrlPatterns(CollectionUtils.wrap("/*"));
		bean.setName("CAS Client Info Logging Filter");
		// 是支持异步
		bean.setAsyncSupported(true);
		// 优先级是最高的
		bean.setOrder(Ordered.HIGHEST_PRECEDENCE + 1);
		
		// 以下这些都是构建Filter需要的参数
		// 定义从request中获取client的ip地址的协议头名称
		// cas.audit.engine.alternateClientAddrHeaderName
		val initParams = new HashMap<String, String>();
		if (StringUtils.isNotBlank(audit.getAlternateClientAddrHeaderName())) {
			initParams.put(ClientInfoThreadLocalFilter.CONST_IP_ADDRESS_HEADER, audit.getAlternateClientAddrHeaderName());
		}
		
		// 定义从request中获取server的ip地址的协议头名称(与useServerHostAddress是排它的)
		// cas.audit.engine.alternateServerAddrHeaderName
		if (StringUtils.isNotBlank(audit.getAlternateServerAddrHeaderName())) {
			initParams.put(ClientInfoThreadLocalFilter.CONST_SERVER_IP_ADDRESS_HEADER, audit.getAlternateServerAddrHeaderName());
		}
		
		// 是否使用服务器的IP地址信息
		// cas.audit.engine.useServerHostAddress
		initParams.put(ClientInfoThreadLocalFilter.CONST_USE_SERVER_HOST_ADDRESS, String.valueOf(audit.isUseServerHostAddress()));
		bean.setInitParameters(initParams);
		return bean;
	} // end casClientInfoLoggingFilter
	
}
```
### (3). ClientInfoThreadLocalFilter
```
// 1. 实现:Filter
public class ClientInfoThreadLocalFilter implements Filter {

    public static final String CONST_IP_ADDRESS_HEADER = "alternativeIpAddressHeader";
    public static final String CONST_SERVER_IP_ADDRESS_HEADER = "alternateServerAddrHeaderName";
    public static final String CONST_USE_SERVER_HOST_ADDRESS = "useServerHostAddress";

    private String alternateLocalAddrHeaderName;
    private boolean useServerHostAddress;
    private String alternateServerAddrHeaderName;
    
    @Override
    public void destroy() {
        // nothing to do here
    }

    @Override
    public void doFilter(final ServletRequest request, final ServletResponse response, final FilterChain filterChain) throws IOException, ServletException {
        try {
			// 3. 创建ClientInfo对象
            final ClientInfo clientInfo =
                    new ClientInfo((HttpServletRequest) request,
                            this.alternateServerAddrHeaderName,
                            this.alternateLocalAddrHeaderName,
                            this.useServerHostAddress);
            
			// ***********************************************************************
			// 4. 把ClientInfo与线程上下文进行绑定
			// ***********************************************************************
			ClientInfoHolder.setClientInfo(clientInfo);
			
			// 5. 继续执行过滤器
            filterChain.doFilter(request, response);
        } finally {
			// ***********************************************************************
			// 6. 清除线程上下文信息.
			// ***********************************************************************// ***********************************************************************
            ClientInfoHolder.clear();
        }
    }


	//  ******************************************************************************
	// 初始时,会被调用,就是上面配置的参数
	//  ******************************************************************************
    @Override
    public void init(final FilterConfig filterConfig) throws ServletException {
        this.alternateLocalAddrHeaderName = filterConfig.getInitParameter(CONST_IP_ADDRESS_HEADER);
        this.alternateServerAddrHeaderName = filterConfig.getInitParameter(CONST_SERVER_IP_ADDRESS_HEADER);
        String useServerHostAddr = filterConfig.getInitParameter(CONST_USE_SERVER_HOST_ADDRESS);
        if (useServerHostAddr != null && !useServerHostAddr.isEmpty()) {
            this.useServerHostAddress = Boolean.valueOf(useServerHostAddr);
        }
    }
}
```
### (4). ClientInfo
```
package org.apereo.inspektr.common.web;

import javax.servlet.http.HttpServletRequest;
import java.net.Inet4Address;

public class ClientInfo {
    public static ClientInfo EMPTY_CLIENT_INFO = new ClientInfo();

	// 服务器的ip地址信息
    /** IP Address of the server (local). */
    private final String serverIpAddress;

	// 请求时,client的ip地址信息
    /** IP Address of the client (Remote) */
    private final String clientIpAddress;
	
	
    private final String geoLocation;

    private final String userAgent;
    
    private ClientInfo() {
        this(null);
    }

    public ClientInfo(final HttpServletRequest request) {
        this(request, null, null, false);
    }
    
	// *************************************************************************************
	// 读取request中部份信息,转换到当前模型上.
	// *************************************************************************************
    public ClientInfo(final HttpServletRequest request,
                      final String alternateServerAddrHeaderName,
                      final String alternateLocalAddrHeaderName,
                      final boolean useServerHostAddress) {

        try {
            String serverIpAddress = request != null ? request.getLocalAddr() : null;
            String clientIpAddress = request != null ? request.getRemoteAddr() : null;
			
            if (request == null) {
                this.geoLocation = "unknown";
                this.userAgent = "unknown";
            } else {
				// *************************************************************************************
				// 1. cas.audit.engine.useServerHostAddress  与  cas.audit.engine.alternateServerAddrHeaderName 是排它的
				// *************************************************************************************
                if (useServerHostAddress) {
					// 直接使用服务器本地的ip地址信息
                    serverIpAddress = Inet4Address.getLocalHost().getHostAddress();
                } else if (alternateServerAddrHeaderName != null && !alternateServerAddrHeaderName.isEmpty()) {
					// 从协议头获得服务器的ip的ip地址信息,如果取不到的话从request中获得localAddr
                    serverIpAddress = request.getHeader(alternateServerAddrHeaderName) != null ? request.getHeader(alternateServerAddrHeaderName) : request.getLocalAddr();
                }
				
				// *************************************************************************************
				// 2. 读取协议头里client的ip地址信息,读取不到的情况下,获取:RemoteAddr
				// *************************************************************************************
                if (alternateLocalAddrHeaderName != null && !alternateLocalAddrHeaderName.isEmpty()) {
                    clientIpAddress = request.getHeader(alternateLocalAddrHeaderName) != null ? request.getHeader(alternateLocalAddrHeaderName) : request.getRemoteAddr();
                }
				
				// *************************************************************************************
				// 3. 从请求头,获得:user-agent,如果,没有user-agent,则返回:unknown
				// *************************************************************************************
                String header = request.getHeader("user-agent");
                this.userAgent = header == null ? "unknown" : header;
				
				// *************************************************************************************
				// 4. 从请求的参数中,获得:geolocation,如果,参数中不存在,则从协议头中获取.
				// *************************************************************************************
                String geo = request.getParameter("geolocation");
                if (geo == null) {
                    geo = request.getHeader("geolocation");
                }
                this.geoLocation = geo == null ? "unknown" : geo;
				
            }
			
			// 最终赋值服务端的ip地址 和 client端的ip地址.
            this.serverIpAddress = serverIpAddress == null ? "unknown" : serverIpAddress;
            this.clientIpAddress = clientIpAddress == null ? "unknown" : clientIpAddress;
            
        } catch (final Exception e) {
            throw new RuntimeException(e);
        }
    }

    // ... ...
}

```
### (5). 总结
ClientInfoThreadLocalFilter的职责比较简单:   
+ 读取Request信息转换成:ClientInfo.  
+ 把ClientInfo与ThreadLocal进行绑定. 
+ 通过责任链,继续往下执行.  
+ 把ClientInfo与ThreadLocal进行解绑. 