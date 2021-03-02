---
layout: post
title: 'Liquibase Erueka延迟注册(四)'
date: 2021-02-27
author: 李新
tags: Liquibase 解决方案
---

### (1). 概述
> 能不能在Liquibase执行完成之后,再让我们的程序向注册中心(Eureka)进行注册呢(即服务正确初始化之后,再提供服务)?  
> 答案是可以的,这里提供大概思路:
> 1. 扩展EurekaAutoServiceRegistration.    
> 2. 接受到自定义Event后,才真正的触发向注册中心注册.  

### (2). 自定义事件(RegisterServiceStartEvent)
```
public class RegisterServiceStartEvent extends ApplicationEvent {
    public RegisterServiceStartEvent(Object source) {
        super(source);
    }
}
```
### (3). 禁用Spring的EurekaAutoServiceRegistration
```
package help.lixin.framework.eureka.serviceregistry;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.beans.factory.support.RootBeanDefinition;

/**
 * 禁用Spring自带的:EurekaAutoServiceRegistration,添加自定义的.
 */
public class DisableEurekaAutoServiceRegistrationProcessor implements BeanFactoryPostProcessor {
    private Logger logger = LoggerFactory.getLogger(DisableEurekaAutoServiceRegistrationProcessor.class);

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        if (beanFactory instanceof DefaultListableBeanFactory) {
            DefaultListableBeanFactory defaultListableBeanFactory = (DefaultListableBeanFactory) beanFactory;
            String beanName = "eurekaAutoServiceRegistration";
            boolean res = beanFactory.containsBeanDefinition(beanName);
            if (res) {
                defaultListableBeanFactory.removeBeanDefinition(beanName);
                BeanDefinition eurekaAutoServiceRegistrationOverrideDefinition = new RootBeanDefinition(
                        EurekaAutoServiceRegistrationOverride.class);
                ((DefaultListableBeanFactory) beanFactory).registerBeanDefinition(beanName, eurekaAutoServiceRegistrationOverrideDefinition);
                logger.info("Override EurekaAutoServiceRegistration To EurekaAutoServiceRegistrationOverride SUCCESS");
            } else {
                logger.warn("Override EurekaAutoServiceRegistration To EurekaAutoServiceRegistrationOverride FAIL,description = applicationContext NOT FOUND beanName:[{}]", beanName);
            }
            res = beanFactory.containsBeanDefinition(beanName);
            logger.info("Override EurekaAutoServiceRegistration To EurekaAutoServiceRegistrationOverride SUCCESS");
        } else {
            logger.warn("Override EurekaAutoServiceRegistration To EurekaAutoServiceRegistrationOverride FAIL,description = (ConfigurableListableBeanFactory !instanceof DefaultListableBeanFactory)");
        }
    }
}
```
### (4). 扩展EurekaAutoServiceRegistration(EurekaAutoServiceRegistrationOverride)
```
public class EurekaAutoServiceRegistrationOverride extends EurekaAutoServiceRegistration {
	private Logger logger = LoggerFactory.getLogger(EurekaAutoServiceRegistrationOverride.class);

	private ApplicationContext context;

	private EurekaRegistration registration;

	private EurekaServiceRegistry serviceRegistry;

	private AtomicInteger port = new AtomicInteger(0);

	private AtomicBoolean running = new AtomicBoolean(false);

	private AtomicBoolean register = new AtomicBoolean(false);

	private Set<String> expectedEventTriggerRegister = new HashSet<>();

	public EurekaAutoServiceRegistrationOverride(ApplicationContext context, EurekaServiceRegistry serviceRegistry,
												 EurekaRegistration registration, EurekaLazyRegisterProperties eurekaLazyRegisterProperties) {
		super(context, serviceRegistry, registration);
		this.context = context;
		this.serviceRegistry = serviceRegistry;
		this.registration = registration;

		// clone
		Set<String> clone = new HashSet<>();
		clone.addAll(eurekaLazyRegisterProperties.getExpectedEventTriggerRegister());
		expectedEventTriggerRegister = clone;
	}

	@Override
	public void start() {
		this.running.set(true);
		logger.warn(
				"Enable EurekaAutoServiceRegistration Override,Wait RegisterServiceStartEvent , Delay Register Service...");
	}

	public boolean isRunning() {
		return running.get();
	}

	@Override
	public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
		return WebServerInitializedEvent.class.isAssignableFrom(eventType)
				|| ContextClosedEvent.class.isAssignableFrom(eventType)
				|| RegisterServiceStartEvent.class.isAssignableFrom(eventType);
	}

	public void onApplicationEvent(RegisterServiceStartEvent event) {
		logger.info("revice RegisterServiceStartEvent,ready Register Service.");
		// 获得事件源名称.
		String eventName = (String) event.getSource();
		// 移除事件
		expectedEventTriggerRegister.remove(eventName);
		// 移除事件之后,发现还存在要期待的事件,则返回,直到:expectedEventTriggerRegister里不存在事件,则进行注册.
		if (!expectedEventTriggerRegister.isEmpty()) {
			return;
		}

		if (this.port.get() != 0) {
			if (this.registration.getNonSecurePort() == 0) {
				this.registration.setNonSecurePort(this.port.get());
			}

			if (this.registration.getSecurePort() == 0 && this.registration.isSecure()) {
				this.registration.setSecurePort(this.port.get());
			}
		}

		if (this.running.get() && this.registration.getNonSecurePort() > 0) {
			this.serviceRegistry.register(this.registration);
			this.context.publishEvent(new InstanceRegisteredEvent<>(this, this.registration.getInstanceConfig()));
			this.running.set(true);
			// 标记注册完成
			this.register.set(true);
		}
	}

	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof WebServerInitializedEvent) {
			onApplicationEvent((WebServerInitializedEvent) event);
		} else if (event instanceof ContextClosedEvent) {
			onApplicationEvent((ContextClosedEvent) event);
		} else if (event instanceof RegisterServiceStartEvent) {
			onApplicationEvent((RegisterServiceStartEvent) event);
		}
	}

	public Set<String> getExpectedEventTriggerRegister() {
		return expectedEventTriggerRegister;
	}

	public AtomicBoolean getRegister() {
		return register;
	}
}
```
### (5). 配置监听
> 监听的目的,主要是防止一直未向注册中心注册,需要打日志提醒.  

```
public class EurekaAutoServiceRegistrationOverrideListener {
    private Logger logger = LoggerFactory.getLogger(EurekaAutoServiceRegistrationOverrideListener.class);
    private EurekaAutoServiceRegistration eurekaAutoServiceRegistration;
    // 间隔多少秒执行一次.
    private final long delay;
    private volatile boolean isRunning = false;

    public EurekaAutoServiceRegistrationOverrideListener(EurekaAutoServiceRegistration eurekaAutoServiceRegistration, long delay) {
        this.delay = delay;
        this.eurekaAutoServiceRegistration = eurekaAutoServiceRegistration;
        Thread checkThread = new Thread(runnable());
        checkThread.setDaemon(true);
        checkThread.setName("EurekaAutoServiceRegistrationOverrideListener");
        checkThread.start();
    }

    public Runnable runnable() {
        return () -> {
            while (!isRunning) {
                if (eurekaAutoServiceRegistration instanceof EurekaAutoServiceRegistrationOverride) {
                    EurekaAutoServiceRegistrationOverride eurekaAutoServiceRegistrationOverride = (EurekaAutoServiceRegistrationOverride) eurekaAutoServiceRegistration;
                    Set<String> expectedEvents = eurekaAutoServiceRegistrationOverride.getExpectedEventTriggerRegister();
                    if (!eurekaAutoServiceRegistrationOverride.getRegister().get()) {
                        logger.warn("EurekaAutoServiceRegistration Register Service FAIL,Wait Events:[{}]", expectedEvents);
                    } else {
                        // 注册成功的情况下,当前的线程不要再执行了,不要浪费线程做无意义的事情.
                        isRunning = true;
                    }
                }
                try {
                    TimeUnit.SECONDS.sleep(delay);
                } catch (InterruptedException ignore) {
                }
            }
        };
    }
}
```
### (6). 发布事件,触发向注册中心注册
```
applicationContext.publishEvent(new RegisterServiceStartEvent("TriggerRegister"));
```
### (7). 总结
> 由于Spring没有提供Hook的方式,对EurekaAutoServiceRegistration进行扩展,所以,只能偷梁换柱的先删除定义信息,再添加自定义的信息.   
