---
layout: post
title: 'Spring Cloud Bus源码之BusJacksonMessageConverter(四)' 
date: 2021-10-23
author: 李新
tags:  SpringCloudBus
---

### (1). 概述
在前面分析了spring.factories中的一部份,在这里,将分析:spring.factories里的:BusJacksonMessageConverter,一看这个类就知道是运行Jackson对Message进行序列化和反序列化来着的.   

### (2). BusJacksonAutoConfiguration
```
package org.springframework.cloud.bus.jackson;


@Configuration
// 1. 验证:spring.cloud.bus.enabled
@ConditionalOnBusEnabled
@ConditionalOnClass({ RefreshBusEndpoint.class, ObjectMapper.class })
@AutoConfigureBefore({ BusAutoConfiguration.class, JacksonAutoConfiguration.class})
public class BusJacksonAutoConfiguration {

	
	// 2. 创建Bean(org.springframework.messaging.converter.AbstractMessageConverter)
    @Bean
    @ConditionalOnMissingBean(name = "busJsonConverter")
	
	// spging cloud stream流的转换
	// org.springframework.cloud.stream.annotation.StreamMessageConverter
    @StreamMessageConverter
    public AbstractMessageConverter busJsonConverter(@Autowired(required = false) ObjectMapper objectMapper) {
        return new BusJacksonMessageConverter(objectMapper);
    }
}
```
### (3). BusJacksonMessageConverter
```
class BusJacksonMessageConverter extends AbstractMessageConverter implements InitializingBean {

	private static final Log log = LogFactory.getLog(BusJacksonMessageConverter.class);

	private static final String DEFAULT_PACKAGE = ClassUtils.getPackageName(RemoteApplicationEvent.class);

	private final ObjectMapper mapper;
	private final boolean mapperCreated;

	private String[] packagesToScan = new String[] { DEFAULT_PACKAGE };

	public BusJacksonMessageConverter() {
		this(null);
    }

    @Autowired(required = false)
	public BusJacksonMessageConverter(ObjectMapper objectMapper) {
		super(MimeTypeUtils.APPLICATION_JSON);

		if (objectMapper != null) {
			this.mapper = objectMapper;
			this.mapperCreated = false;
		} else {
			this.mapper = new ObjectMapper();
			this.mapperCreated = true;
		}
	}

	public boolean isMapperCreated() {
		return mapperCreated;
	}

	public void setPackagesToScan(String[] packagesToScan) {
		List<String> packages = new ArrayList<>(Arrays.asList(packagesToScan));
		if (!packages.contains(DEFAULT_PACKAGE)) {
			packages.add(DEFAULT_PACKAGE);
		}
		this.packagesToScan = packages.toArray(new String[0]);
	} // end setPackagesToScan

	private Class<?>[] findSubTypes() {
		List<Class<?>> types = new ArrayList<>();
		if (this.packagesToScan != null) {
			for (String pkg : this.packagesToScan) {
				ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(false);
				provider.addIncludeFilter(
						new AssignableTypeFilter(RemoteApplicationEvent.class));

				Set<BeanDefinition> components = provider.findCandidateComponents(pkg);
				for (BeanDefinition component : components) {
					try {
						types.add(Class.forName(component.getBeanClassName()));
					} catch (ClassNotFoundException e) {
						throw new IllegalStateException("Failed to scan classpath for remote event classes", e);
					}
				}
			}
		}
		
		if (log.isDebugEnabled()) {
			log.debug("Found sub types: "+types);
		}
		return types.toArray(new Class<?>[0]);
	} //end findSubTypes

	// *********************************************************************************************
    // 设置支持的类型必须得是:RemoteApplicationEvent
	// *********************************************************************************************
	@Override
	protected boolean supports(Class<?> aClass) {
		return RemoteApplicationEvent.class.isAssignableFrom(aClass);
	}

	@Override
	public Object convertFromInternal(Message<?> message, Class<?> targetClass, Object conversionHint) {
		Object result = null;
		try {
			Object payload = message.getPayload();
			
			// 取出:Message.getPayload() 
			if (payload instanceof byte[]) { // byte[]转换成:targetClass
				try {
					result = this.mapper.readValue((byte[]) payload, targetClass);
				} catch (InvalidTypeIdException e) {
					return new UnknownRemoteApplicationEvent(new Object(), e.getTypeId(), (byte[]) payload);
				}
			} else if (payload instanceof String) {  // String转换成:targetClass
				try {
					result = this.mapper.readValue((String) payload, targetClass);
				} catch (InvalidTypeIdException e) {
					return new UnknownRemoteApplicationEvent(new Object(), e.getTypeId(), ((String) payload).getBytes());
				}
			// workaround for https://github.com/spring-cloud/spring-cloud-stream/issues/1564
			} else if (payload instanceof RemoteApplicationEvent) { // RemoteApplicationEvent
				return payload;
			}
		} catch (Exception e) {
			this.logger.error(e.getMessage(), e);
			return null;
		}
		return result;
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		this.mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
		this.mapper.registerModule(new SubtypeModule(findSubTypes()));
	}
}
```
### (4). 总结
从上面的内容能分析出来,BusJacksonAutoConfiguration配置类的目的:就是创建:AbstractMessageConverter的实现,而:AbstractMessageConverter主要是对Message中的Body进行序列化和反序列化的操作.  