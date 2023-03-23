---
layout: post
title: 'Java容器之间隔离方案' 
date: 2023-03-23
author: 李新
tags:  Spring
---

### (1). 概述

最近在做一些东西,想要实现对象容器之间的隔离,在看Feign时有看到这段代码,所以,特意做一个简单的实现.    

### (2). NamedContextFactory
```
package org.springframework.cloud.context.named;

import java.util.Collection;
import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactoryUtils;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.core.ResolvableType;
import org.springframework.core.env.MapPropertySource;

/**
 * Creates a set of child contexts that allows a set of Specifications to define the beans
 * in each child context.
 *
 * Ported from spring-cloud-netflix FeignClientFactory and SpringClientFactory
 *
 * @param <C> specification
 * @author Spencer Gibb
 * @author Dave Syer
 */
// TODO: add javadoc
public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
        implements DisposableBean, ApplicationContextAware {

    private final String propertySourceName;

    private final String propertyName;

    private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();

    private Map<String, C> configurations = new ConcurrentHashMap<>();

    private ApplicationContext parent;

    private Class<?> defaultConfigType;

    public NamedContextFactory(Class<?> defaultConfigType, String propertySourceName, String propertyName) {
        this.defaultConfigType = defaultConfigType;
        this.propertySourceName = propertySourceName;
        this.propertyName = propertyName;
    }

    @Override
    public void setApplicationContext(ApplicationContext parent) throws BeansException {
        this.parent = parent;
    }

    public void setConfigurations(List<C> configurations) {
        for (C client : configurations) {
            this.configurations.put(client.getName(), client);
        }
    }

    public Set<String> getContextNames() {
        return new HashSet<>(this.contexts.keySet());
    }

    @Override
    public void destroy() {
        Collection<AnnotationConfigApplicationContext> values = this.contexts.values();
        for (AnnotationConfigApplicationContext context : values) {
            // This can fail, but it never throws an exception (you see stack traces
            // logged as WARN).
            context.close();
        }
        this.contexts.clear();
    }

    protected AnnotationConfigApplicationContext getContext(String name) {
        if (!this.contexts.containsKey(name)) {
            synchronized (this.contexts) {
                if (!this.contexts.containsKey(name)) {
                    this.contexts.put(name, createContext(name));
                }
            }
        }
        return this.contexts.get(name);
    }

    protected AnnotationConfigApplicationContext createContext(String name) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        if (this.configurations.containsKey(name)) {
            for (Class<?> configuration : this.configurations.get(name).getConfiguration()) {
                context.register(configuration);
            }
        }
        for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
            if (entry.getKey().startsWith("default.")) {
                for (Class<?> configuration : entry.getValue().getConfiguration()) {
                    context.register(configuration);
                }
            }
        }
        context.register(PropertyPlaceholderAutoConfiguration.class, this.defaultConfigType);
        context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(this.propertySourceName,
                Collections.<String, Object>singletonMap(this.propertyName, name)));
        if (this.parent != null) {
            // Uses Environment from parent as well as beans
            context.setParent(this.parent);
            // jdk11 issue
            // https://github.com/spring-cloud/spring-cloud-netflix/issues/3101
            context.setClassLoader(this.parent.getClassLoader());
        }
        context.setDisplayName(generateDisplayName(name));
        context.refresh();
        return context;
    }

    protected String generateDisplayName(String name) {
        return this.getClass().getSimpleName() + "-" + name;
    }

    public <T> T getInstance(String name, Class<T> type) {
        AnnotationConfigApplicationContext context = getContext(name);
        try {
            return context.getBean(type);
        }
        catch (NoSuchBeanDefinitionException e) {
            // ignore
        }
        return null;
    }

    public <T> ObjectProvider<T> getLazyProvider(String name, Class<T> type) {
        return new ClientFactoryObjectProvider<>(this, name, type);
    }

    public <T> ObjectProvider<T> getProvider(String name, Class<T> type) {
        AnnotationConfigApplicationContext context = getContext(name);
        return context.getBeanProvider(type);
    }

    public <T> T getInstance(String name, Class<?> clazz, Class<?>... generics) {
        ResolvableType type = ResolvableType.forClassWithGenerics(clazz, generics);
        return getInstance(name, type);
    }

    @SuppressWarnings("unchecked")
    public <T> T getInstance(String name, ResolvableType type) {
        AnnotationConfigApplicationContext context = getContext(name);
        String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context, type);
        if (beanNames.length > 0) {
            for (String beanName : beanNames) {
                if (context.isTypeMatch(beanName, type)) {
                    return (T) context.getBean(beanName);
                }
            }
        }
        return null;
    }

    public <T> Map<String, T> getInstances(String name, Class<T> type) {
        AnnotationConfigApplicationContext context = getContext(name);

        return BeanFactoryUtils.beansOfTypeIncludingAncestors(context, type);
    }

    /**
     * Specification with name and configuration.
     */
    public interface Specification {

        String getName();

        Class<?>[] getConfiguration();

    }

}
```
### (3). ClientFactoryObjectProvider
```
/*
 * Copyright 2012-2020 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.cloud.context.named;

import java.util.Iterator;
import java.util.Spliterator;
import java.util.function.Consumer;
import java.util.function.Supplier;
import java.util.stream.Stream;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.lang.Nullable;

/**
 * Special ObjectProvider that allows the actual ObjectProvider to be resolved later
 * because of the creation of the named child context.
 *
 * @param <T> - type of the provided object
 */
class ClientFactoryObjectProvider<T> implements ObjectProvider<T> {

    private final NamedContextFactory<?> clientFactory;

    private final String name;

    private final Class<T> type;

    private ObjectProvider<T> provider;

    ClientFactoryObjectProvider(NamedContextFactory<?> clientFactory, String name, Class<T> type) {
        this.clientFactory = clientFactory;
        this.name = name;
        this.type = type;
    }

    @Override
    public T getObject(Object... args) throws BeansException {
        return delegate().getObject(args);
    }

    @Override
    @Nullable
    public T getIfAvailable() throws BeansException {
        return delegate().getIfAvailable();
    }

    @Override
    public T getIfAvailable(Supplier<T> defaultSupplier) throws BeansException {
        return delegate().getIfAvailable(defaultSupplier);
    }

    @Override
    public void ifAvailable(Consumer<T> dependencyConsumer) throws BeansException {
        delegate().ifAvailable(dependencyConsumer);
    }

    @Override
    @Nullable
    public T getIfUnique() throws BeansException {
        return delegate().getIfUnique();
    }

    @Override
    public T getIfUnique(Supplier<T> defaultSupplier) throws BeansException {
        return delegate().getIfUnique(defaultSupplier);
    }

    @Override
    public void ifUnique(Consumer<T> dependencyConsumer) throws BeansException {
        delegate().ifUnique(dependencyConsumer);
    }

    @Override
    public Iterator<T> iterator() {
        return delegate().iterator();
    }

    @Override
    public Stream<T> stream() {
        return delegate().stream();
    }

    @Override
    public T getObject() throws BeansException {
        return delegate().getObject();
    }

    @Override
    public void forEach(Consumer<? super T> action) {
        delegate().forEach(action);
    }

    @Override
    public Spliterator<T> spliterator() {
        return delegate().spliterator();
    }

    private ObjectProvider<T> delegate() {
        if (this.provider == null) {
            this.provider = this.clientFactory.getProvider(this.name, this.type);
        }
        return this.provider;
    }

}
```
### (4). ActionSpecification
```
package help.lixin.camunda.demo.module;

import java.util.Arrays;
import java.util.Objects;

import org.springframework.cloud.context.named.NamedContextFactory;


public class ActionSpecification implements NamedContextFactory.Specification {

    private String name;

    private Class<?>[] configuration;

    public ActionSpecification() {
    }

    public ActionSpecification(String name, Class<?>[] configuration) {
        this.name = name;
        this.configuration = configuration;
    }

    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Class<?>[] getConfiguration() {
        return this.configuration;
    }

    public void setConfiguration(Class<?>[] configuration) {
        this.configuration = configuration;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        ActionSpecification that = (ActionSpecification) o;
        return Objects.equals(this.name, that.name) && Arrays.equals(this.configuration, that.configuration);
    }

    @Override
    public int hashCode() {
        return Objects.hash(this.name, this.configuration);
    }

    @Override
    public String toString() {
        return new StringBuilder("ActionSpecification{").append("name='").append(this.name).append("', ").append("configuration=").append(Arrays.toString(this.configuration)).append("}").toString();
    }

}
```
### (5). ActionContextFactory
```
package help.lixin.camunda.demo.module;

import lixin.help.config.ActionConfig;
import org.springframework.cloud.context.named.NamedContextFactory;

public class ActionContextFactory extends NamedContextFactory<ActionSpecification> {

    public ActionContextFactory() {
        //  propertySourceName: 为map定义一个名称
        //  propertyName: map中的key
        super(ActionConfig.class, "actionProperties", "tenant");
    }
}
```
### (6). ActionConfig
```
package lixin.help.config;

import lixin.help.service.IHelloService;
import lixin.help.service.impl.HelloService;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ActionConfig {

    @Value("${tenant}")
    private String tenant;


    @Bean
    public IHelloService helloService() {
        System.out.println(tenant);
        return new HelloService();
    }
}
```
### (7). ActionModuleConfig
```
package help.lixin.camunda.demo.module;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ActionModuleConfig {
    @Bean
    public ActionContextFactory actionContextFactory() {
        return new ActionContextFactory();
    }
}
```
### (8). Application
```
package help.lixin.camunda.demo;

import help.lixin.camunda.demo.module.ActionContextFactory;
import lixin.help.service.IHelloService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
public class Application {
    public static void main(String... args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(Application.class, args);
        ActionContextFactory actionContextFactory = ctx.getBean(ActionContextFactory.class);
        // 此处的name可以是随意的,只是会根据这个name创建一个容器而已.
        IHelloService helloService = actionContextFactory.getInstance("00007", IHelloService.class);
        helloService.print();
    }
}
```

### (9). 总结
NamedContextFactory适用于根据固定的类,来生成对象,然后,这些在ApplicationContext内.  