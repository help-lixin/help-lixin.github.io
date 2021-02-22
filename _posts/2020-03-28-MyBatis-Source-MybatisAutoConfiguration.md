---
layout: post
title: 'MyBatis源码之MybatisAutoConfiguration(一)'
date: 2020-03-28
author: 李新
tags: MyBatis
---

### (1). 目的
> 研究mybatis-spring-boot-starter,看下有哪些配置项以及可扩展项.   

### (2). 添加mybatis starter依赖
```
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<version>2.1.4</version>
</dependency>
```
### (3). 寻找mybatis-spring-boot-starter入口
> 发现:mybatis-spring-boot-starter是一个jar,但是没有相应的SPI(EnableAutoConfiguration).   
> 那么,就看下POM依赖信息吧,期望在相应的POM依赖里有SPI.   

```
<dependencies>
	<dependency>
	  <groupId>org.springframework.boot</groupId>
	  <artifactId>spring-boot-starter</artifactId>
	</dependency>
	<dependency>
	  <groupId>org.springframework.boot</groupId>
	  <artifactId>spring-boot-starter-jdbc</artifactId>
	</dependency>
	<!-- 重点:看这个名字就知道是SPI -->
	<dependency>
	  <groupId>org.mybatis.spring.boot</groupId>
	  <artifactId>mybatis-spring-boot-autoconfigure</artifactId>
	</dependency>
	<dependency>
	  <groupId>org.mybatis</groupId>
	  <artifactId>mybatis</artifactId>
	</dependency>
	<dependency>
	  <groupId>org.mybatis</groupId>
	  <artifactId>mybatis-spring</artifactId>
	</dependency>
</dependencies>
```
### (4). mybatis-spring-boot-autoconfigure.jar/META-INF/spring.factories
```
# Auto Configure
# 我们关注:MybatisAutoConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisLanguageDriverAutoConfiguration,\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```
### (5). MybatisAutoConfiguration
```
// 1. Spring认识这是一个配置类
@org.springframework.context.annotation.Configuration
// 2. CllassPath下有:SqlSessionFactory/SqlSessionFactoryBean这个配置类才能生效. 
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
// 3. Spring容器中要有一个:DataSource
@ConditionalOnSingleCandidate(DataSource.class)
// 4. 启用配置类(MybatisProperties)
@EnableConfigurationProperties(MybatisProperties.class)
// 5. 在加载MybatisAutoConfiguration之前,要先加载:DataSourceAutoConfiguration/MybatisLanguageDriverAutoConfiguration
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, MybatisLanguageDriverAutoConfiguration.class })
public class MybatisAutoConfiguration implements InitializingBean {
	// 配置信息	
	private final MybatisProperties properties;
	// 拦截器
	private final Interceptor[] interceptors;
	// 类型处理器
	private final TypeHandler[] typeHandlers;
	// MyBatis 3.2提供的(Pluggable Scripting Languages For Dynamic SQL)
	private final LanguageDriver[] languageDrivers;
	// 资源加载器
	private final ResourceLoader resourceLoader;
	private final DatabaseIdProvider databaseIdProvider;
	// **********************************************************
	// 如果你想对:Configuration对象进行扩展配置,可以实现:ConfigurationCustomizer.customize(Configuration configuration)
	// **********************************************************
	private final List<ConfigurationCustomizer> configurationCustomizers;

	// 6. 构造器
	public MybatisAutoConfiguration(MybatisProperties properties, ObjectProvider<Interceptor[]> interceptorsProvider,
	  ObjectProvider<TypeHandler[]> typeHandlersProvider, ObjectProvider<LanguageDriver[]> languageDriversProvider,
	  ResourceLoader resourceLoader, ObjectProvider<DatabaseIdProvider> databaseIdProvider,
	  ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider) {
		this.properties = properties;
		this.interceptors = interceptorsProvider.getIfAvailable();
		this.typeHandlers = typeHandlersProvider.getIfAvailable();
		this.languageDrivers = languageDriversProvider.getIfAvailable();
		this.resourceLoader = resourceLoader;
		this.databaseIdProvider = databaseIdProvider.getIfAvailable();
		this.configurationCustomizers = configurationCustomizersProvider.getIfAvailable();
	} //end 
  
	// 7. 由于该类实现了:InitializingBean,所以Spring会回调该方法.
	public void afterPropertiesSet() {
		//委托给:checkConfigFileExists
		checkConfigFileExists();
	} // end checkConfigFileExists

	private void checkConfigFileExists() {
	  // mybatis.check-config-location = false
	  // mybatis.config-location = null
	  // mybatis.config-location: 是我们平时配置mybatis的入口配置文件,而非*Mapper.xml
	  if (this.properties.isCheckConfigLocation() && StringUtils.hasText(this.properties.getConfigLocation())) {
		Resource resource = this.resourceLoader.getResource(this.properties.getConfigLocation());
		Assert.state(resource.exists(),
			"Cannot find config location: " + resource + " (please add config file or check your Mybatis configuration)");
	  }
	} // end checkConfigFileExists
	
	
	// 8. 这一步是构建:SqlSessionFactory
	@Bean
	@ConditionalOnMissingBean
	public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
		SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
		factory.setDataSource(dataSource);
		factory.setVfs(SpringBootVFS.class);
		
		// mybatis.config-location
		if (StringUtils.hasText(this.properties.getConfigLocation())) {
		  factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
		}
		
		// 1. 如果配置项:mybatis.configuration为空
		// 1.1 创建一个:Configuration对象
		// 1.2 获取所有ConfigurationCustomizer的实现类,并挨个调用:ConfigurationCustomizer.customize(Configuration)
		//     上一步主要是留个Hook给开发,允许开发人员对:Configuration对象进行配置.
		applyConfiguration(factory);
		
		// mybatis.configuration-properties
		if (this.properties.getConfigurationProperties() != null) {
		  factory.setConfigurationProperties(this.properties.getConfigurationProperties());
		}
		
		// 配置拦截器:interceptors
		if (!ObjectUtils.isEmpty(this.interceptors)) {
		  factory.setPlugins(this.interceptors);
		}
		
		// 配置:databaseIdProvider
		if (this.databaseIdProvider != null) {
		  factory.setDatabaseIdProvider(this.databaseIdProvider);
		}
		
		// 实体所在的package名称,多package之间用逗号分隔
		// mybatis.type-aliases-package
		if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
		  factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
		}
		
		// mybatis.type-aliases-super-type
		if (this.properties.getTypeAliasesSuperType() != null) {
		  factory.setTypeAliasesSuperType(this.properties.getTypeAliasesSuperType());
		}
		
		// 可以自定义类型处理器(比较:BigDecimal),多个之间用逗号分隔
		// mybatis.type-handlers-package
		if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
		  factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
		}
		
		// 
		if (!ObjectUtils.isEmpty(this.typeHandlers)) {
		  factory.setTypeHandlers(this.typeHandlers);
		}
		
		// 指定*Mapper.xml文件
		// mybatis.mapper-locations = []
		if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
		  factory.setMapperLocations(this.properties.resolveMapperLocations());
		}
		
		// 下面这几个留着以后分析MyBatis新的特性再详解.
		Set<String> factoryPropertyNames = Stream
			.of(new BeanWrapperImpl(SqlSessionFactoryBean.class).getPropertyDescriptors()).map(PropertyDescriptor::getName)
			.collect(Collectors.toSet());
		Class<? extends LanguageDriver> defaultLanguageDriver = this.properties.getDefaultScriptingLanguageDriver();
		if (factoryPropertyNames.contains("scriptingLanguageDrivers") && !ObjectUtils.isEmpty(this.languageDrivers)) {
		  // Need to mybatis-spring 2.0.2+
		  factory.setScriptingLanguageDrivers(this.languageDrivers);
		  if (defaultLanguageDriver == null && this.languageDrivers.length == 1) {
			defaultLanguageDriver = this.languageDrivers[0].getClass();
		  }
		}
		if (factoryPropertyNames.contains("defaultScriptingLanguageDriver")) {
		  // Need to mybatis-spring 2.0.2+
		  factory.setDefaultScriptingLanguageDriver(defaultLanguageDriver);
		}
		return factory.getObject();
	} // end sqlSessionFactory
	
	
	// 9. 创建:SqlSessionTemplate
	@Bean
	@ConditionalOnMissingBean
	public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
		// mybatis.executorType
		// 根据不同的:ExecutorType创建:Executor
		ExecutorType executorType = this.properties.getExecutorType();
		if (executorType != null) {
		  return new SqlSessionTemplate(sqlSessionFactory, executorType);
		} else {
		  return new SqlSessionTemplate(sqlSessionFactory);
		}
	} // end sqlSessionTemplate
}
```
### (6). MybatisAutoConfiguration$AutoConfiguredMapperScannerRegistrar
> 向Spring容器中注册一个Bean(MapperScannerConfigurer),MapperScannerConfigurer的主要职责是把IMapper接口与IMapper.xml进行关联(动态代理)  

```
public static class AutoConfiguredMapperScannerRegistrar 
              implements BeanFactoryAware, 
			  ImportBeanDefinitionRegistrar {
	
	private BeanFactory beanFactory;

	// 2. 因为实现了:ImportBeanDefinitionRegistrar
	//    Spring会回调访方法
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	  if (!AutoConfigurationPackages.has(this.beanFactory)) {
		logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.");
		return;
	  }
	  logger.debug("Searching for mappers annotated with @Mapper");
	  List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
	  if (logger.isDebugEnabled()) {
		packages.forEach(pkg -> logger.debug("Using auto-configuration base package '{}'", pkg));
	  }

	  // *********************************************************
	  // 向Spring中注册一个Bean:MapperScannerConfigurer
	  // *********************************************************
	  BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
	  // MapperScannerConfigurer.setProcessPropertyPlaceHolders
	  builder.addPropertyValue("processPropertyPlaceHolders", true);
	  // MapperScannerConfigurer.setAnnotationClass
	  builder.addPropertyValue("annotationClass", Mapper.class);
	  // MapperScannerConfigurer.setBasePackage
	  builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(packages));
	  
	  BeanWrapper beanWrapper = new BeanWrapperImpl(MapperScannerConfigurer.class);
	  // 配置属性
	  Set<String> propertyNames = Stream.of(beanWrapper.getPropertyDescriptors()).map(PropertyDescriptor::getName)
		  .collect(Collectors.toSet());
	  if (propertyNames.contains("lazyInitialization")) {
		// Need to mybatis-spring 2.0.2+
		builder.addPropertyValue("lazyInitialization", "${mybatis.lazy-initialization:false}");
	  }
	  if (propertyNames.contains("defaultScope")) {
		// Need to mybatis-spring 2.0.6+
		builder.addPropertyValue("defaultScope", "${mybatis.mapper-default-scope:}");
	  }
	  
	  // *************************************************
	  // 向Spring中注册Bean(MapperScannerConfigurer)
	  // *************************************************
	  registry.registerBeanDefinition(MapperScannerConfigurer.class.getName(), builder.getBeanDefinition());
	} // end registerBeanDefinitions

	// 1.因为实现了:BeanFactoryAware
	// 所以,Spring会回调该方法,并设置:BeanFactory(ApplicationContext)
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
	  this.beanFactory = beanFactory;
	} // end setBeanFactory
} // end 
```
### (7). 总结
> mybatis-spring-boot-starter的核心配置类是:MybatisAutoConfiguration,它的职责如下:  
> 1. 注入开发人员自定义的ConfigurationCustomizer对象(ConfigurationCustomizer可以对Configuration进行配置).  
> 2. 创建SqlSessionFactory.  
> 3. 创建SqlSessionTemplate.  
> 4. 创建MapperScannerConfigurer,它主要负责把*Mapper.java进行动态代理(与*Mapper.xml进行关联).   
> 5. MapperScannerConfigurer类属于MyBatis与Spring的整合,在这里我不剖析了.   
