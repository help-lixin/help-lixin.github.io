---
layout: post
title: 'Spring Security源码之DefaultLoginPageGeneratingFilter(六)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
比较好奇一件事情,就是Spring Security是如何生成登录页面的呢?

### (2). WebSecurityConfigurerAdapter
```
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
		
		// ***************************************************************************
		// FormLoginConfigurer是核心类
		// ***************************************************************************
        FormLoginConfigurer formLoginConfigurer = http.formLogin();
	}
}		
```
### (3). FormLoginConfigurer
```
public final class FormLoginConfigurer<H extends HttpSecurityBuilder<H>> extends
		AbstractAuthenticationFilterConfigurer<H, FormLoginConfigurer<H>, UsernamePasswordAuthenticationFilter> {

	public FormLoginConfigurer() {
		// *******************************************************************************
		// 1. 配置了验证:账号和密码登录Filter.暂时,咱们先不管理这个.
		// *******************************************************************************
		super(new UsernamePasswordAuthenticationFilter(), null);
		usernameParameter("username");
		passwordParameter("password");
	}
	
	// 我们知道,init方法最终是会被HttpSecurity统一回调的
	public void init(H http) throws Exception {
		super.init(http);
		// *******************************************************************************
		// 2. 初始化默认登录页面Filter
		// *******************************************************************************
		initDefaultLoginFilter(http);
	}

	private void initDefaultLoginFilter(H http) {
		// *******************************************************************************
		// 3. 从Spring容器的上下文中获得一个Filter:DefaultLoginPageGeneratingFilter
		// *******************************************************************************
		DefaultLoginPageGeneratingFilter loginPageGeneratingFilter = http.getSharedObject(DefaultLoginPageGeneratingFilter.class);
		if (loginPageGeneratingFilter != null && !isCustomLoginPage()) {
			// 对DefaultLoginPageGeneratingFilter进行配置
			loginPageGeneratingFilter.setFormLoginEnabled(true);
			loginPageGeneratingFilter.setUsernameParameter(getUsernameParameter());
			loginPageGeneratingFilter.setPasswordParameter(getPasswordParameter());
			loginPageGeneratingFilter.setLoginPageUrl(getLoginPage());
			loginPageGeneratingFilter.setFailureUrl(getFailureUrl());
			loginPageGeneratingFilter.setAuthenticationUrl(getLoginProcessingUrl());
		}
	} // end initDefaultLoginFilter
}
```
### (4). DefaultLoginPageGeneratingFilter
```
public class DefaultLoginPageGeneratingFilter extends GenericFilterBean {
	
	public static final String DEFAULT_LOGIN_PAGE_URL = "/login";
	public static final String ERROR_PARAMETER_NAME = "error";
	private String loginPageUrl;
	private String logoutSuccessUrl;
	private String failureUrl;
	private boolean formLoginEnabled;
	private boolean openIdEnabled;
	private boolean oauth2LoginEnabled;
	private String authenticationUrl;
	private String usernameParameter;
	private String passwordParameter;
	private String rememberMeParameter;
	private String openIDauthenticationUrl;
	private String openIDusernameParameter;
	private String openIDrememberMeParameter;
	private Map<String, String> oauth2AuthenticationUrlToClientName;
	private Function<HttpServletRequest, Map<String, String>> resolveHiddenInputs = request -> Collections.emptyMap();
	
	// ************************************************************************************
	// GenericFilterBean实现了Filter,所以,重点关注:filter方法.
	// ************************************************************************************
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
				throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		boolean loginError = isErrorPage(request);
		boolean logoutSuccess = isLogoutSuccess(request);
		// 如果是登录页面请求/错误请求/退出请求
		if (isLoginUrlRequest(request) || loginError || logoutSuccess) {
			// *****************************************************************************
			// 生成登录页面
			// *****************************************************************************
			String loginPageHtml = generateLoginPageHtml(request, loginError, logoutSuccess);
			response.setContentType("text/html;charset=UTF-8");
			response.setContentLength(loginPageHtml.getBytes(StandardCharsets.UTF_8).length);
			response.getWriter().write(loginPageHtml);

			return;
		}
		chain.doFilter(request, response);
	} // end doFilter
	
	
	// 居然直接把要生成的页面HTML,写在代码里了.
	private String generateLoginPageHtml(HttpServletRequest request, boolean loginError, boolean logoutSuccess) {
		String errorMsg = "none";

		if (loginError) {
			HttpSession session = request.getSession(false);

			if (session != null) {
				AuthenticationException ex = (AuthenticationException) session.getAttribute(WebAttributes.AUTHENTICATION_EXCEPTION);
				errorMsg = ex != null ? ex.getMessage() : "none";
			}
		}

		StringBuilder sb = new StringBuilder();

		sb.append("<html><head><title>Login Page</title></head>");

		if (formLoginEnabled) {
			sb.append("<body onload='document.f.").append(usernameParameter).append(".focus();'>\n");
		}

		if (loginError) {
			sb.append("<p style='color:red;'>Your login attempt was not successful, try again.<br/><br/>Reason: ");
			sb.append(errorMsg);
			sb.append("</p>");
		}

		if (logoutSuccess) {
			sb.append("<p style='color:green;'>You have been logged out</p>");
		}

		if (formLoginEnabled) {
			sb.append("<h3>Login with Username and Password</h3>");
			sb.append("<form name='f' action='").append(request.getContextPath())
					.append(authenticationUrl).append("' method='POST'>\n");
			sb.append("<table>\n");
			sb.append("	<tr><td>User:</td><td><input type='text' name='");
			sb.append(usernameParameter).append("' value='").append("'></td></tr>\n");
			sb.append("	<tr><td>Password:</td><td><input type='password' name='").append(passwordParameter).append("'/></td></tr>\n");

			if (rememberMeParameter != null) {
				sb.append("	<tr><td><input type='checkbox' name='").append(rememberMeParameter).append("'/></td><td>Remember me on this computer.</td></tr>\n");
			}

			sb.append("	<tr><td colspan='2'><input name=\"submit\" type=\"submit\" value=\"Login\"/></td></tr>\n");
			renderHiddenInputs(sb, request);
			sb.append("</table>\n");
			sb.append("</form>");
		}

		if (openIdEnabled) {
			sb.append("<h3>Login with OpenID Identity</h3>");
			sb.append("<form name='oidf' action='").append(request.getContextPath()).append(openIDauthenticationUrl).append("' method='POST'>\n");
			sb.append("<table>\n");
			sb.append("	<tr><td>Identity:</td><td><input type='text' size='30' name='");
			sb.append(openIDusernameParameter).append("'/></td></tr>\n");

			if (openIDrememberMeParameter != null) {
				sb.append("	<tr><td><input type='checkbox' name='")
                  .append(openIDrememberMeParameter)
				  .append("'></td><td>Remember me on this computer.</td></tr>\n");
			}

			sb.append("	<tr><td colspan='2'><input name=\"submit\" type=\"submit\" value=\"Login\"/></td></tr>\n");
			sb.append("</table>\n");
			renderHiddenInputs(sb, request);
			sb.append("</form>");
		}

		if (oauth2LoginEnabled) {
			sb.append("<h3>Login with OAuth 2.0</h3>");
			sb.append("<table>\n");
			for (Map.Entry<String, String> clientAuthenticationUrlToClientName : oauth2AuthenticationUrlToClientName.entrySet()) {
				sb.append(" <tr><td>");
				sb.append("<a href=\"").append(request.getContextPath()).append(clientAuthenticationUrlToClientName.getKey()).append("\">");
				sb.append(HtmlUtils.htmlEscape(clientAuthenticationUrlToClientName.getValue(), "UTF-8"));
				sb.append("</a>");
				sb.append("</td></tr>\n");
			}
			sb.append("</table>\n");
		}

		sb.append("</body></html>");

		return sb.toString();
	} // end generateLoginPageHtml
	
	
	// 创建隐藏域
	private void renderHiddenInputs(StringBuilder sb, HttpServletRequest request) {
		for(Map.Entry<String, String> input : this.resolveHiddenInputs.apply(request).entrySet()) {
			sb.append("	<input name=\"" + input.getKey()
					+ "\" type=\"hidden\" value=\"" + input.getValue() + "\" />\n");
		}
	} // end renderHiddenInputs
	
}	
```
### (5). 总结
在这里没有对UsernamePasswordAuthenticationFilter进行剖析,而是,先剖析了生成登录页面的Filter(DefaultLoginPageGeneratingFilter)进行了剖析.   