---
layout: post
title: 'Spring Security源码之EnableAutoConfiguration(一)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 切入点在哪?
由于是和Spring Security和Spring Boot有做集成,所以,只需要找到:spring.factories即可.  

```
# spring-boot-autoconfigure-2.0.9.RELEASE.jar!/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityRequestMatcherProviderAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\

org.springframework.boot.autoconfigure.security.reactive.ReactiveSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveUserDetailsServiceAutoConfiguration,\

org.springframework.boot.autoconfigure.security.oauth2.client.OAuth2ClientAutoConfiguration
```

### (2). 关注什么?
因为,要看Spring Boot(MVC)的集成,所以,只需要关注以下三个类即可.   

```
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityRequestMatcherProviderAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\
```
### (3). SecurityFilterAutoConfiguration
> SecurityFilterAutoConfiguration的目的在于,启动Servlet容器时,Servlet容器会回调,并传递一个ServletContext,你可以向ServletContext中注册一些信息(比如:Filter/Servlet)  

```
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(SecurityProperties.class)
@ConditionalOnClass({ AbstractSecurityWebApplicationInitializer.class, SessionCreationPolicy.class })
@AutoConfigureAfter(SecurityAutoConfiguration.class)
public class SecurityFilterAutoConfiguration {

	private static final String DEFAULT_FILTER_NAME = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME;

    // ***************************************************************************
	// 向Spring(ServletContext)容器中注册一个Filter. 
	// Filter = org.springframework.web.filter.DelegatingFilterProxy
	// ***************************************************************************
	@Bean
	@ConditionalOnBean(name = DEFAULT_FILTER_NAME)
	public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
			SecurityProperties securityProperties) {
		DelegatingFilterProxyRegistrationBean registration = new DelegatingFilterProxyRegistrationBean(DEFAULT_FILTER_NAME);
		registration.setOrder(securityProperties.getFilter().getOrder());
		registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
		return registration;
	} // end securityFilterChainRegistration
}
```
### (4). SecurityAutoConfiguration
> 认证或者鉴权失败,会通过一个事件去处理(不是重点).   

```
@Configuration
@ConditionalOnClass(DefaultAuthenticationEventPublisher.class)
@EnableConfigurationProperties(SecurityProperties.class)
@Import({ SpringBootWebSecurityConfiguration.class, WebSecurityEnablerConfiguration.class,
		SecurityDataConfiguration.class })
public class SecurityAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(AuthenticationEventPublisher.class)
	public DefaultAuthenticationEventPublisher authenticationEventPublisher(ApplicationEventPublisher publisher) {
		return new DefaultAuthenticationEventPublisher(publisher);
	} // endauthenticationEventPublisher

}
```
### (5). UserDetailsServiceAutoConfiguration
> 该类的主要目的在于,创建UserDetailsManager对象,UserDetailsManager对象持有着登录需要的账号和密码.   

```
@Configuration
@ConditionalOnClass(AuthenticationManager.class)
@ConditionalOnBean(ObjectPostProcessor.class)
@ConditionalOnMissingBean({ AuthenticationManager.class, AuthenticationProvider.class,UserDetailsService.class })
public class UserDetailsServiceAutoConfiguration {

	private static final String NOOP_PASSWORD_PREFIX = "{noop}";

	private static final Pattern PASSWORD_ALGORITHM_PATTERN = Pattern.compile("^\\{.+}.*$");

	private static final Log logger = LogFactory.getLog(UserDetailsServiceAutoConfiguration.class);

    // **************************************************************************************************************
	// InMemoryUserDetailsManager属于:UserDetailsManager的实现类,看名字就知道是一个内存型的账号和密码管理.
	// **************************************************************************************************************
	@Bean
	@ConditionalOnMissingBean(type = "org.springframework.security.oauth2.client.registration.ClientRegistrationRepository")
	@Lazy
	public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties, ObjectProvider<PasswordEncoder> passwordEncoder) {
		
		// spring.security.user.name=zhagnsan
		// 从配置文件中获得user信息
		SecurityProperties.User user = properties.getUser();
		
		// spring.security.user.roles[0]=admin
		// spring.security.user.roles[1]=test
		List<String> roles = user.getRoles();
		
		
		return new InMemoryUserDetailsManager(User.withUsername(user.getName())
				// 针对密码进行处理
				.password(getOrDeducePassword(user, passwordEncoder.getIfAvailable()))
				
				// 对role进行处理.
				// ************************************************************
				// 在构建时,会在角色名称前面加上前缀(ROLE_),例如:ROLE_admin / ROLE_test
				// ************************************************************
				.roles(StringUtils.toStringArray(roles)).build());
	} // end inMemoryUserDetailsManager

    // 处理密码部份
	private String getOrDeducePassword(SecurityProperties.User user,PasswordEncoder encoder) {
		// 从配置文件中获得密码
		// spring.security.user.password=123456
		String password = user.getPassword();
		
		// 判断密码是否为自动生成的,如果是自动生成的,在控制台打印密码.
		if (user.isPasswordGenerated()) {
			logger.info(String.format("%n%nUsing generated security password: %s%n",user.getPassword()));
		}
		
		// 咦,PasswordEncoder没有发挥他的作用哈,在这里只是用正则判断了一下密码,如果正则能match就返回密码,否则,就在密码的前面加上:{noop}
		if (encoder != null || PASSWORD_ALGORITHM_PATTERN.matcher(password).matches()) {
			return password;
		}
		
		return NOOP_PASSWORD_PREFIX + password;
	} // end getOrDeducePassword

}
```
### (6). SecurityRequestMatcherProviderAutoConfiguration
> 向Spring容器中,注册RequestMatcherProvider,内部数据结构,包含着所有的:HandlerMapping.   

```
@Configuration
@ConditionalOnClass({ RequestMatcher.class })
// ******************************************************************************
// 要求当前的web容器,得要是servlet容器,而非:REACTIVE
// ******************************************************************************
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
public class SecurityRequestMatcherProviderAutoConfiguration {

	@Configuration
	@ConditionalOnClass(DispatcherServlet.class)
	@ConditionalOnBean(HandlerMappingIntrospector.class)
	public static class MvcRequestMatcherConfiguration {
		
		
		// *******************************************************************************
		// HandlerMappingIntrospector是一个集合,包含着mvc中所有的:HandlerMapping
		// org.springframework.web.servlet.handler.SimpleUrlHandlerMapping
		// org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
		// org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping
		// org.springframework.web.servlet.handler.SimpleUrlHandlerMapping
		// org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport$EmptyHandlerMapping
		// org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport$EmptyHandlerMapping
		// org.springframework.boot.autoconfigure.web.servlet.WelcomePageHandlerMapping
		// *******************************************************************************
		@Bean
		@ConditionalOnClass(DispatcherServlet.class)
		public RequestMatcherProvider requestMatcherProvider(HandlerMappingIntrospector introspector) {
			return new MvcRequestMatcherProvider(introspector);
		} // end requestMatcherProvider

	} // end MvcRequestMatcherConfiguration
}	
```
### (7). 总结
> 从源码分析来看,Spring Security在初始化的时候,创建了3个Bean,分别是:  
+ DelegatingFilterProxy
+ UserDetailsManager
+ RequestMatcherProvider