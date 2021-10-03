---
layout: post
title: 'Spring Cloud Stream集成Kafka源码之KafkaBinderConfiguration(十一)'
date: 2021-10-01
author: 李新
tags: SpringCloudStream Kafka
---

### (1). 概述
在前面,把Spring Cloud Stream与Kafka进行了集成,在这里,开始要深入了解下,集成后有哪些配置项是可以配置的,也就是要找到业务模型. 

### (2). 源码切入点在哪?
一年前有把Spring Cloud Stream与RocketMQ进行集成,稍微的分析了下原理,在这里也稍微的提一下,切入点代码在:spring-cloud-stream工程里,它会读取classpath下所有的:spring.binders,并初始化.   

```
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
@EnableConfigurationProperties({ BindingServiceProperties.class })
@Import(ContentTypeConfiguration.class)
public class BinderFactoryAutoConfiguration {
	@Bean
	public BinderTypeRegistry binderTypeRegistry(
			ConfigurableApplicationContext configurableApplicationContext) {
		Map<String, BinderType> binderTypes = new HashMap<>();
		ClassLoader classLoader = configurableApplicationContext.getClassLoader();
		try {
			// **********************************************************************
			// spring cloud stream在初始化的时候,会去读取:META-INF/spring.binders文件,
			// **********************************************************************
			Enumeration<URL> resources = classLoader.getResources("META-INF/spring.binders");
			
			// ... ...

			if (binderTypes.isEmpty() && !Boolean.valueOf(this.selfContained)
					&& (resources == null || !resources.hasMoreElements())) {
				this.logger.debug(
						"Failed to locate 'META-INF/spring.binders' resources on the classpath."
								+ " Assuming standard boot 'META-INF/spring.factories' configuration is used");
			} else {
				// ********************************************************************
				// 解析:META-INF/spring.binders的内容,并,保存到:Map容器里.
				// ********************************************************************
				while (resources.hasMoreElements()) {
					URL url = resources.nextElement();
					UrlResource resource = new UrlResource(url);
					for (BinderType binderType : parseBinderConfigurations(classLoader, resource)) {
						binderTypes.put(binderType.getDefaultName(), binderType);
					}
				}
			} //end else
		} catch (IOException | ClassNotFoundException e) {
			throw new BeanCreationException("Cannot create binder factory:", e);
		}
		return new DefaultBinderTypeRegistry(binderTypes);
	} // end binderTypeRegistry
}
```
### (3). 查看kafka的配置(spring.binders)
```
# spring-cloud-stream-binder-kafka-2.1.0.RELEASE.jar/META-INF/spring.binders

kafka:\
org.springframework.cloud.stream.binder.kafka.config.KafkaBinderConfiguration
```
### (4). KafkaBinderConfiguration
```
@Bean
KafkaBinderConfigurationProperties configurationProperties(KafkaProperties kafkaProperties) {
	return new KafkaBinderConfigurationProperties(kafkaProperties);
}
```
### (5). KafkaBinderConfigurationProperties
> KafkaBinderConfigurationProperties主要负责Kafka的相关信息配置,那么,另一部份配置又在哪呢(spring.cloud.stream.bindings)? 

```
// *******************************************************************************
// KafkaBinderConfigurationProperties主要负责如下配置,从这些配置能看出,主要是配置MQ的相关信息.
// # spring.cloud.stream.kafka.binder.brokers=127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094
// # spring.cloud.stream.kafka.binder.zk-nodes=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
// # spring.cloud.stream.kafka.binder.auto-create-topics=true
// # spring.cloud.stream.kafka.binder.required-acks=-1
// *******************************************************************************
@ConfigurationProperties(prefix = "spring.cloud.stream.kafka.binder")
public class KafkaBinderConfigurationProperties {
	// kafka brokder地址
	private static final String DEFAULT_KAFKA_CONNECTION_STRING = "localhost:9092";
	
	// kafka支持事务管理
	private final Transaction transaction = new Transaction();
	
	// 内部再持有一个:org.springframework.boot.autoconfigure.kafka.KafkaProperties
	// 会从KafkaProperties中解析出来一些信息,填充到当前模型里.
	private final KafkaProperties kafkaProperties;
	
	// zk地址
	private String[] zkNodes = new String[] { "localhost" };
	
	// 应用于生产者和消费者的属性
	private Map<String, String> configuration = new HashMap<>();
	
	// consumer属性配置
	private Map<String, String> consumerProperties = new HashMap<>();

	// producer属性配置
	private Map<String, String> producerProperties = new HashMap<>();
	
	// zk端口
	private String defaultZkPort = "2181";
	
	// kafka broker地址
	private String[] brokers = new String[] { "localhost" };
	
	// brokder port
	private String defaultBrokerPort = "9092";

	private String[] headers = new String[] {};
	
	// 更新offset时间窗口
	private int offsetUpdateTimeWindow = 10000;
	
	// 
	private int offsetUpdateCount;

	private int offsetUpdateShutdownTimeout = 2000;
	
	// 
	private int maxWait = 100;
	
	// 自动创建topic
	private boolean autoCreateTopics = true;
	
	// 自动添加praition
	private boolean autoAddPartitions;

	// socket缓冲区
	private int socketBufferSize = 2097152;

	// zk会话超时时间
	private int zkSessionTimeout = 10000;

	 // zk连接超时时间
	private int zkConnectionTimeout = 10000;
	
	// product ack模式(0/1/-1)
	private String requiredAcks = "1";
	
	// 副本数
	private short replicationFactor = 1;
	
	// 每次fetch大小
	private int fetchSize = 1024 * 1024;

	// 最小分区数
	private int minPartitionCount = 1;
	
	// 
	private int queueSize = 8192;

	// 心跳
	private int healthTimeout = 60;
	private JaasLoginModuleConfiguration jaas;
	private String headerMapperBeanName;
}
```
### (6). BindingServiceProperties
> BindingServiceProperties属于spring colud stream的代码,它主要负责:spring.cloud.stream.bindings的配置.   

```
@ConfigurationProperties("spring.cloud.stream")
@JsonInclude(Include.NON_DEFAULT)
public class BindingServiceProperties
	implements ApplicationContextAware, InitializingBean {

    // 重试间隔
	private static final int DEFAULT_BINDING_RETRY_INTERVAL = 30;
	
	private String source;

	@Value("${INSTANCE_INDEX:${CF_INSTANCE_INDEX:0}}")
	private int instanceIndex;
	
	private List<Integer> instanceIndexList = new ArrayList<>();

	private int instanceCount = 1;

	// **********************************************************************
	// 
	// spring.cloud.stream.bindings.output.destination=hello-world
    // spring.cloud.stream.bindings.output.content-type=text/plain
	// **********************************************************************
	private Map<String, BindingProperties> bindings = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);

	
	private Map<String, BinderProperties> binders = new HashMap<>();

	
	private String defaultBinder;

	
	private String[] dynamicDestinations = new String[0];

	
	private int dynamicDestinationCacheSize = 10;
	
	// 绑定重试间隔
	private int bindingRetryInterval = DEFAULT_BINDING_RETRY_INTERVAL;

	private ConfigurableApplicationContext applicationContext = new GenericApplicationContext();

	private ConversionService conversionService;
}		
```
### (7). BindingProperties
```
package org.springframework.cloud.stream.config;

@JsonInclude(Include.NON_DEFAULT)
@Validated
public class BindingProperties  {
	// 
	public static final MimeType DEFAULT_CONTENT_TYPE = MimeTypeUtils.APPLICATION_JSON;
	private static final String COMMA = ",";
	
	// *************************************************************
	
	// *************************************************************
	private String destination;
	
	// 所属组
	private String group;
	
	// 持久化时的数据类型
	private String contentType = DEFAULT_CONTENT_TYPE.toString();
	
	private String binder;
		
    // *************************************************************
	// 消费者信息指定
	//  spring.cloud.stream.bindings.XXX.consumer.concurrency=2
	// *************************************************************
	private ConsumerProperties consumer;
	
	// *************************************************************
	// 生产者信息指定
	// *************************************************************
	private ProducerProperties producer;
}
```
### (8). 总结
通过源码分析,我们能知道,Spring Cloud Stream与Kafka集成后,支持哪些属性了.   
