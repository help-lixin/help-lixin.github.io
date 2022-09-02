---
layout: post
title: 'Spring Plugin源码之PluginRegistry(三)' 
date: 2022-09-02
author: 李新
tags:  SpringPlugin
---

### (1). 概述
前面介绍到了OrderAwarePluginRegistry,它是PluginRegistry的实现类,我们稍微看一下:PluginRegistry的接口定义,大概就能知道,它拥有哪功能了.

### (2). Plugin
> 先看一下这个接口,因为,这个接口是所有Plugin都要求实现的接口.  

```
package org.springframework.plugin.core;

public interface Plugin<S> {
	
	boolean supports(S delimiter);
}	
```
### (3). PluginRegistry
```
package org.springframework.plugin.core;

import java.util.Arrays;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;
import java.util.Optional;
import java.util.function.Supplier;

import org.springframework.util.Assert;

public interface PluginRegistry<T extends Plugin<S>, S> extends Iterable<T> {

	public static <S, T extends Plugin<S>> PluginRegistry<T, S> empty() {
		return of(Collections.emptyList());
	}

	public static <S, T extends Plugin<S>> PluginRegistry<T, S> of(Comparator<? super T> comparator) {

		Assert.notNull(comparator, "Comparator must not be null!");

		return of(Collections.emptyList(), comparator);
	}

	@SafeVarargs
	public static <S, T extends Plugin<S>> PluginRegistry<T, S> of(T... plugins) {
		return of(Arrays.asList(plugins), OrderAwarePluginRegistry.DEFAULT_COMPARATOR);
	}

	public static <S, T extends Plugin<S>> PluginRegistry<T, S> of(List<? extends T> plugins) {
		return of(plugins, OrderAwarePluginRegistry.DEFAULT_COMPARATOR);
	}

	public static <S, T extends Plugin<S>> PluginRegistry<T, S> of(List<? extends T> plugins,
			Comparator<? super T> comparator) {

		Assert.notNull(plugins, "Plugins must not be null!");
		Assert.notNull(comparator, "Comparator must not be null!");

		return OrderAwarePluginRegistry.of(plugins, comparator);
	}

	Optional<T> getPluginFor(S delimiter);

	T getRequiredPluginFor(S delimiter) throws IllegalArgumentException;

	T getRequiredPluginFor(S delimiter, Supplier<String> message) throws IllegalArgumentException;

	List<T> getPluginsFor(S delimiter);

	<E extends Exception> T getPluginFor(S delimiter, Supplier<E> ex) throws E;
	
	<E extends Exception> List<T> getPluginsFor(S delimiter, Supplier<E> ex) throws E;

	T getPluginOrDefaultFor(S delimiter, T plugin);

	T getPluginOrDefaultFor(S delimiter, Supplier<T> defaultSupplier);

	List<T> getPluginsFor(S delimiter, List<? extends T> plugins);

	int countPlugins();

	boolean contains(T plugin);

	boolean hasPluginFor(S delimiter);

	List<T> getPlugins();
}
```
### (4). 总结
OrderAwarePluginRegistry内部因为Hold住所有的Plugin集合(List),PluginRegistry的方法无非不过是基于这些集合做一些操作而已.  