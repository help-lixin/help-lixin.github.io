---
layout: post
title: '独立租户部署ERP系统,成本节约解决方案'
date: 2021-03-21
author: 李新
tags:  解决方案
---

### (1). 需求
> 随着微服务的流行,我们会根据业务逻辑的关系,对微服务进行拆分,最终一套Saas系统可能会存在上百个微服务进程,物理机器的数量可能达到了几十台,甚至可能上百台.     
> 如果Saas系统中某个租户,需要一套独立的部署,应该怎么办?有两个点是要考虑的:    
> 1. 机器数量代表着企业成本,动辄上百台机器,租户肯定是不能承受的.    
> 2. 独立部署的用户是随着时间而增长的,刚部署的机器与用户的承载数是不成正比的,是否存在浪费机器的可能性.  

### (2). 解决思路
> 带着上面的那些问题,所以,才会存在这么一套解决方案.解决的思路大致如下:  
> 1. 独立租户在部署时,会把所有的微服务,全部合在一个大的"容器"里,但是,每个微服务都有自己空间,各自不相干扰.  
> 2. 随着独立租户的用户基线增长,可以,随时,让某个微服务从"容器"中分离出来,独立成一个新的微服务进程.     
> 3. 这套解决方案,不能影响开发人员,对开发人员要尽可能的透明.  

### (3). 解决方案
> 既然如此,那市面上是否有相应的解决方案呢?  
> 1. OSGI,OSGI确实不错,但是,需要开发学习OSGI的知识,对开发影响比较大,而且,我的需求比较简单,用上OSGI有点大重了.  
> 2. Jarslink,Jarslink是蚂蚁金服研发出来的一套模块化框架(也是嫌充OSGI太重),原理实则:就是扩展了ClassLoader,让每一个微服务都在自己的ClassLoader时运行.     
> 3. 我这边的方案是对Jarslink进行扩展开发.  

### (4). 业务模块集成步骤
> 1. 业务模块只需要按照下面的模板,改动SpringApplication即可.  
> 2. 业务模块打包成jar(比如:hello-service-1.0.0-SNAPSHOT.jar).
> 3. 在独立部署用户时,为每个业务模块配置xml,例如:<module name="hello-service" version="1.0.0" desc="hello service" enabled="true" class="help.lixin.hello.Application" properties-file="application.properties"/>  
> 4. 在boot项目的plungins目录下,新建一个文件夹,这个文件夹的名称要求必须是微服务的名称(比如:hello-service)  
> 5. 把业务模块jar包和xml(*.properties)拷贝到${boot}/plugins/hello-service目录下.

>  SpringApplication模板代码如下:

```
package help.lixin.hello;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.Properties;
import java.util.stream.Stream;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;

@SpringBootApplication
public class Application {
	public static void main(String[] args) throws Exception {
		new Application().start(args, null);
	}

	private List<Process> process = new ArrayList<Process>();

    // 1. boot项目会通过xml配置中的信息,反射实例化该类
	public Application() {
		process.add(new PropertiesProcess());
	}

    // 2. 通过实例化之后,反射调用该方法.这样写的目的是为了能做扩展(请看后面我预留的案例).
    //    Object... contexts 可以由用户自定义扩展.
    //    String[] args    是启动时传递的参数(比如:--spring.config.location=classpath:/application.properties  --logging.config=/logback.xml).
	public ApplicationContext start(String[] args, Object... contexts) {
		SpringApplication application = new SpringApplication(Application.class);
		Optional.ofNullable(contexts).ifPresent(items -> {
			Stream.of(items).forEach(context -> {
				process.stream().forEach(process -> {
					if (process.support(context)) {
						process.process(application, context);
					}
				});
			});
		});
		return application.run(args);
	}
}

// 3. 定义ContextProcess
interface Process {
	boolean support(Object obj);

	void process(SpringApplication springApplication, Object obj);
}

// 4. 针对Properties的处理
class PropertiesProcess implements Process {

	@Override
	public boolean support(Object obj) {
		// 4.1 会先对object进行判断,是否支持
		return null != obj && obj instanceof Properties;
	}

	@Override
	public void process(SpringApplication application, Object obj) {
		// *******************************************
		// 4.2 再次调用:process进行配置
		// *******************************************
		application.setDefaultProperties((Properties) obj);
	}
}
```

> 模块目录结构如下:

```
plugins/
├── common-lib               #所有项目公共的jar
│   ├── classmate-1.4.0.jar
│   ├── commons-collections-3.2.2.jar
│   ├── commons-lang-2.6.jar
│   ├── commons-lang3-3.8.1.jar
│   ├── commons-logging-1.1.3.jar
│   ├── guava-17.0.jar
│   ├── hibernate-validator-6.0.13.Final.jar
│   ├── jackson-annotations-2.9.0.jar
│   ├── jackson-core-2.9.7.jar
│   ├── jackson-databind-2.9.7.jar
│   ├── jackson-datatype-jdk8-2.9.7.jar
│   ├── jackson-datatype-jsr310-2.9.7.jar
│   ├── jackson-module-parameter-names-2.9.7.jar
│   ├── jarslink-api-1.6.1.20180301.jar
│   ├── javax.annotation-api-1.3.2.jar
│   ├── jboss-logging-3.3.2.Final.jar
│   ├── jul-to-slf4j-1.7.25.jar
│   ├── log4j-api-2.11.1.jar
│   ├── log4j-to-slf4j-2.11.1.jar
│   ├── logback-classic-1.2.3.jar
│   ├── logback-core-1.2.3.jar
│   ├── slf4j-api-1.7.25.jar
│   ├── snakeyaml-1.23.jar
│   ├── spring-aop-5.1.2.RELEASE.jar
│   ├── spring-beans-5.1.2.RELEASE.jar
│   ├── spring-boot-2.1.0.RELEASE.jar
│   ├── spring-boot-autoconfigure-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-json-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-logging-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-tomcat-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-web-2.1.0.RELEASE.jar
│   ├── spring-context-5.1.2.RELEASE.jar
│   ├── spring-core-5.1.2.RELEASE.jar
│   ├── spring-expression-5.1.2.RELEASE.jar
│   ├── spring-jcl-5.1.2.RELEASE.jar
│   ├── spring-web-5.1.2.RELEASE.jar
│   ├── spring-webmvc-5.1.2.RELEASE.jar
│   ├── tomcat-embed-core-9.0.12.jar
│   ├── tomcat-embed-el-9.0.12.jar
│   ├── tomcat-embed-websocket-9.0.12.jar
│   └── validation-api-2.0.2.Final.jar
└── hello-service                                         # 模块名称(微服务名称)
    ├── application.properties                            # 微服务的配置文件
    ├── hello-service-1.0.0-SNAPSHOT.jar                  # 业务模块jar
    ├── lib                                               # 微服务自身的jar包(同时,还会继承common-lib)
    └── module.xml                                        # 模块配置信息
```

### (5). boo模块集成步骤
> 1. boot是一个"容器",它是所有微服务的聚合.       
> 2. boot会加载plugins下的所有模块(没有module.xml)的模块是:公共模块(公共模块是所有的业务模块都要依赖的jar),特别要注意:公共模块只允许有一个.   
> 3. boot会监听plugins的变化,针对变化进行热部署.  

### (6). 总结
> 项目部份代码,以及架构图,后续会进行详解.  