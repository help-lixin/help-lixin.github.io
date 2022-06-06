---
layout: post
title: 'Atomix源码之AtomixRegistry(一)' 
date: 2022-06-01
author: 李新
tags:  Atomix
---

### (1). 概述

### (2). builder
```
Atomix atomix1 = Atomix.builder();
```
### (3). Atomix.builder
```
public static AtomixBuilder builder() {
	return builder(Thread.currentThread().getContextClassLoader());
}
```
### (4). Atomix.builder
```
public static AtomixBuilder builder(ClassLoader classLoader) {
	// ********************************************************************
	// 遍历ClassLoader里所有的类,通过反射创建如下接口的实现类.
	// io.atomix.primitive.partition.PartitionGroup$Type
	// io.atomix.primitive.PrimitiveType
	// io.atomix.primitive.protocol.PrimitiveProtocol$Type
	// io.atomix.core.profile.Profile$Type
	// io.atomix.cluster.discovery.NodeDiscoveryProvider$Type
	// ********************************************************************
	AtomixRegistry registry = AtomixRegistry.registry(classLoader);
	return new AtomixBuilder(config(classLoader, null, registry), registry);
}
```
### (5). AtomixRegistry.registry
```
static AtomixRegistry registry(ClassLoader classLoader) {
	return ClasspathScanningRegistry.builder().withClassLoader(classLoader).build();
}
```
### (6). ClasspathScanningRegistry
```
public class ClasspathScanningRegistry implements AtomixRegistry {

  public static Builder builder() {
    return new Builder();
  } // end builder

  public Builder withClassLoader(ClassLoader classLoader) {
	this.classLoader = checkNotNull(classLoader, "classLoader cannot be null");
	return this;
  } // end withClassLoader

  
  @Override
  public AtomixRegistry build() {
	return new ClasspathScanningRegistry(
		classLoader,
		Arrays.asList(
			PartitionGroup.Type.class,
			PrimitiveType.class,
			PrimitiveProtocol.Type.class,
			Profile.Type.class,
			NodeDiscoveryProvider.Type.class),
		whitelistPackages);
  } // end build
}  
```
### (7). ClasspathScanningRegistry构建器
```
private ClasspathScanningRegistry(ClassLoader classLoader, Collection<Class<? extends NamedType>> types, Set<String> whitelistPackages) {
    final Map<CacheKey, Map<Class<? extends NamedType>, Map<String, NamedType>>> mappings = CACHE.computeIfAbsent(classLoader, cl -> new ConcurrentHashMap<>());
    final Map<Class<? extends NamedType>, Map<String, NamedType>> registrations = mappings.computeIfAbsent(new CacheKey(types.toArray(new Class[0])), cacheKey -> {
          final ClassGraph classGraph = !whitelistPackages.isEmpty() ?
              new ClassGraph().enableClassInfo().whitelistPackages(whitelistPackages.toArray(new String[0])).addClassLoader(classLoader) :
              new ClassGraph().enableClassInfo().addClassLoader(classLoader);
          
          try (final ScanResult scanResult = classGraph.scan()) {
            final Map<Class<? extends NamedType>, Map<String, NamedType>> result = new ConcurrentHashMap<>();
			// **********************************************************************
			// 遍历ClassLoader里所有的类,对类型这行验证.
			// **********************************************************************
            for (Class<? extends NamedType> type : cacheKey.types) {
              final Map<String, NamedType> tmp = new ConcurrentHashMap<>();
			  
              scanResult.getClassesImplementing(type.getName())
			  
			  .forEach(classInfo -> {
                if (classInfo.isInterface() || classInfo.isAbstract() || Modifier.isPrivate(classInfo.getModifiers())) {
                  return;
                }
				
				// **********************************************************************
				// 通过反射,new出这些对象.
				// io.atomix.primitive.partition.PartitionGroup$Type
				// io.atomix.primitive.PrimitiveType
				// io.atomix.primitive.protocol.PrimitiveProtocol$Type
				// io.atomix.core.profile.Profile$Type
				// io.atomix.cluster.discovery.NodeDiscoveryProvider$Type
				// **********************************************************************
                final NamedType instance = newInstance(classInfo.loadClass());
                final NamedType oldInstance = tmp.put(instance.name(), instance);
                if (oldInstance != null) {
                  LOGGER.warn("Found multiple types with name={}, classes=[{}, {}]", instance.name(),
                      oldInstance.getClass().getName(), instance.getClass().getName());
                }
              });
              result.put(type, Collections.unmodifiableMap(tmp));
            }
            return result;
          }
        });
    this.registrations.putAll(registrations);
} 
```
### (8). 看下AtomixRegistry接口能力
```
package io.atomix.core;

import io.atomix.core.registry.ClasspathScanningRegistry;
import io.atomix.utils.NamedType;
import java.util.Collection;


public interface AtomixRegistry {

  /**
   * Creates a new registry.
   *
   * @return the registry instance
   */
  static AtomixRegistry registry() {
    return registry(Thread.currentThread().getContextClassLoader());
  }

  /**
   * Creates a new registry instance using the given class loader.
   *
   * @param classLoader the registry class loader
   * @return the registry instance
   */
  static AtomixRegistry registry(ClassLoader classLoader) {
    return ClasspathScanningRegistry.builder().withClassLoader(classLoader).build();
  }

  /**
   * Returns the collection of registrations for the given type.
   * 返回给定类型的集合实现类
   * @param type the type for which to return registrations
   * @param <T>  the type for which to return registrations
   * @return a collection of registrations for the given type
   */
  <T extends NamedType> Collection<T> getTypes(Class<T> type);

  /**
   * Returns a named registration by type.
   * 根据类型,返回指类型的实现类
   * @param type the registration type
   * @param name the registration name
   * @param <T>  the registration type
   * @return the registration instance
   */
  <T extends NamedType> T getType(Class<T> type, String name);

}
```
### (9). 总结
AtomixRegistry最主要的作用是扫描ClassLoader,如果属于以下这些类型的子类(PartitionGroup$Type/PrimitiveType/PrimitiveProtocol$Type/Profile$Type/NodeDiscoveryProvider$Type),则通过反射new出来.   