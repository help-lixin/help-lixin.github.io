---
layout: post
title: 'CAS源码之ThreadContextMDCServletFilter(五)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
在前面分析了,cas只是简单的创建了一个ClientInfo对象与线程绑定,我们这一篇继续往下分析,在这小篇,主要分析:ThreadContextMDCServletFilter.  
### (2). 看下ThreadContextMDCServletFilter是在什么时机初始化的
```
@Configuration(value = "casLoggingConfiguration", proxyBeanMethods = false)
@EnableConfigurationProperties(CasConfigurationProperties.class)
public class CasLoggingConfiguration {
	
	// CasCookieConfiguration配置中:
	// TicketGrantingCookieRetrievingCookieGenerator
    @Autowired
    @Qualifier("ticketGrantingTicketCookieGenerator")
    private ObjectProvider<CasCookieBuilder> ticketGrantingTicketCookieGenerator;

	// CasCoreTicketsConfiguration配置中:
	// DefaultTicketRegistrySupport
    @Autowired
    @Qualifier("defaultTicketRegistrySupport")
    private ObjectProvider<TicketRegistrySupport> ticketRegistrySupport;

	
	// 1. 创建Filter:ThreadContextMDCServletFilter
	// cas.logging.mdc-enabled=true
    @ConditionalOnBean(value = TicketRegistry.class)
    @ConditionalOnProperty(prefix = "cas.logging", name = "mdc-enabled", havingValue = "true", matchIfMissing = true)
    @Bean
    public FilterRegistrationBean threadContextMDCServletFilter() {
		// 2. 创建ThreadContextMDCServletFilter
        val filter = new ThreadContextMDCServletFilter(ticketRegistrySupport.getObject(), this.ticketGrantingTicketCookieGenerator.getObject());
		
		// 3. 构建参数
        val initParams = new HashMap<String, String>();
        val bean = new FilterRegistrationBean<ThreadContextMDCServletFilter>();
        bean.setFilter(filter);
        bean.setUrlPatterns(CollectionUtils.wrap("/*"));
        bean.setInitParameters(initParams);
        bean.setName("threadContextMDCServletFilter");
        bean.setOrder(Ordered.HIGHEST_PRECEDENCE + 1);
		
        return bean;
    } // end threadContextMDCServletFilter
	
}
```
### (3). ThreadContextMDCServletFilter
```
package org.apereo.cas.logging.web;

// ***********************************************************************
// ... ...
// ***********************************************************************
import org.slf4j.MDC;
// ... ...

public class ThreadContextMDCServletFilter implements Filter {

	// 
    private final TicketRegistrySupport ticketRegistrySupport;
    private final CasCookieBuilder ticketGrantingTicketCookieGenerator;

    private static void addContextAttribute(final String attributeName, final Object value) {
        val result = Optional.ofNullable(value).map(Object::toString).orElse(null);
        if (StringUtils.isNotBlank(result)) {
			// ***********************************************************************
			// 传递变量到MDC是,可以通过配置日志格式化,打印这些信息
			// ***********************************************************************
            MDC.put(attributeName, result);
        }
    }
	
    @Override
    public void init(final FilterConfig filterConfig) {
    }

    @Override
    public void doFilter(final ServletRequest servletRequest, final ServletResponse servletResponse,
                         final FilterChain filterChain) throws IOException, ServletException {
        try {
            val request = (HttpServletRequest) servletRequest;

            addContextAttribute("remoteAddress", request.getRemoteAddr());
            addContextAttribute("remoteUser", request.getRemoteUser());
            addContextAttribute("serverName", request.getServerName());
            addContextAttribute("serverPort", String.valueOf(request.getServerPort()));
            addContextAttribute("locale", request.getLocale().getDisplayName());
            addContextAttribute("contentType", request.getContentType());
            addContextAttribute("contextPath", request.getContextPath());
            addContextAttribute("localAddress", request.getLocalAddr());
            addContextAttribute("localPort", String.valueOf(request.getLocalPort()));
            addContextAttribute("remotePort", String.valueOf(request.getRemotePort()));
            addContextAttribute("pathInfo", request.getPathInfo());
            addContextAttribute("protocol", request.getProtocol());
            addContextAttribute("authType", request.getAuthType());
            addContextAttribute("method", request.getMethod());
            addContextAttribute("queryString", request.getQueryString());
            addContextAttribute("requestUri", request.getRequestURI());
            addContextAttribute("scheme", request.getScheme());
            addContextAttribute("timezone", TimeZone.getDefault().getDisplayName());

            val params = request.getParameterMap();
            params.keySet()
                .stream()
                .filter(k -> !k.equalsIgnoreCase("password"))
                .forEach(k -> {
                    val values = params.get(k);
                    addContextAttribute(k, Arrays.toString(values));
                });

            Collections.list(request.getAttributeNames()).forEach(a -> addContextAttribute(a, request.getAttribute(a)));
            val requestHeaderNames = request.getHeaderNames();
            if (requestHeaderNames != null) {
                Collections.list(requestHeaderNames).forEach(h -> addContextAttribute(h, request.getHeader(h)));
            }

            val cookieValue = this.ticketGrantingTicketCookieGenerator.retrieveCookieValue(request);
            if (StringUtils.isNotBlank(cookieValue)) {
                val p = this.ticketRegistrySupport.getAuthenticatedPrincipalFrom(cookieValue);
                if (p != null) {
                    addContextAttribute("principal", p.getId());
                }
            }
            filterChain.doFilter(servletRequest, servletResponse);
        } finally {
			// ***********************************************************************
			// 清空上下文
			// ***********************************************************************
            MDC.clear();
        }
    }

    /**
     * Does nothing.
     */
    @Override
    public void destroy() {
    }
}
```
### (4). 总结
ThreadContextMDCServletFilter的职责比较简单,配置一些变量到MDC中,可供我们通过配置日志格式时使用.  