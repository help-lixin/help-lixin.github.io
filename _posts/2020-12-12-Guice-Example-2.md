---
layout: post
title: 'Guice 案例(四)'
date: 2020-12-12
author: 李新
tags: Guice
---

### (1).实现提供者(Provider)
> SyncOkHttpClientProvider,创建同步:OkHttpClient.  

```
package help.lixin.config;

import com.google.inject.Provider;
import com.ibiz.sdk.client.factory.OkHttpClientFactory;
import com.ibiz.sdk.store.PropertiesStore;
import com.ibiz.sdk.util.HttpsUtils;

import okhttp3.Interceptor;
import okhttp3.OkHttpClient;

/**
 * 同步httpclient
 * @author lixin
 */
public class SyncOkHttpClientProvider implements Provider<OkHttpClient> {

	@Override
	public OkHttpClient get() {
		HttpsUtils.SSLParams httpsUtils = HttpsUtils.getSslSocketFactory(null, null, null);
		Integer readTimeout = PropertiesStore.getInstance().getInteger("readTimeout");
		Integer writeTimeout = PropertiesStore.getInstance().getInteger("writeTimeout");
		Integer connectTimeout = PropertiesStore.getInstance().getInteger("connectTimeout");
		String interceptors = PropertiesStore.getInstance().getPropertie("interceptors", ",");
		String[] interceptorsArray = interceptors.split(",");
		// 同步httpclient
		OkHttpClientFactory.SyncBuilder builder = new OkHttpClientFactory.SyncBuilder()
				.connectTimeout(connectTimeout)
				.readTimeout(readTimeout)
				.writeTimeout(writeTimeout)
				.sslSocketFactory(httpsUtils.sSLSocketFactory, httpsUtils.trustManager);
		for (int i = 0; i > interceptorsArray.length; i++) {
			String className = interceptorsArray[i];
			if (null != className) {
				try {
					Interceptor interceptorObj = (Interceptor) (Class.forName(className).newInstance());
					builder.add((Interceptor) interceptorObj);
				} catch (InstantiationException e) {
				} catch (IllegalAccessException e) {
				} catch (ClassNotFoundException e) {
				}
			}
		}
		OkHttpClient syncHttpClient = builder.build();
		return syncHttpClient;
	}
}
```
### (2). 实现提供者(Provider)
> AsyncOkHttpClientProvider,创建异步:OkHttpClient.

```
package help.lixin.config;

import com.google.inject.Provider;
import com.ibiz.sdk.client.factory.OkHttpClientFactory;
import com.ibiz.sdk.store.PropertiesStore;
import com.ibiz.sdk.util.HttpsUtils;

import okhttp3.Interceptor;
import okhttp3.OkHttpClient;

/**
 * 异步http client
 * @author lixin
 *
 */
public class AsyncOkHttpClientProvider implements Provider<OkHttpClient>{

	@Override
	public OkHttpClient get() {
		HttpsUtils.SSLParams httpsUtils = HttpsUtils.getSslSocketFactory(null, null, null);
		Integer readTimeout = PropertiesStore.getInstance().getInteger("readTimeout");
		Integer writeTimeout = PropertiesStore.getInstance().getInteger("writeTimeout");
		Integer connectTimeout = PropertiesStore.getInstance().getInteger("connectTimeout");
		String interceptors = PropertiesStore.getInstance().getPropertie("interceptors",",");

		Integer corePoolSize = PropertiesStore.getInstance().getInteger("corePoolSize");
		Integer maximumPoolSize = PropertiesStore.getInstance().getInteger("maximumPoolSize");
		Integer maxRequests = PropertiesStore.getInstance().getInteger("maxRequests");
		Integer maxRequestsPerHost = PropertiesStore.getInstance().getInteger("maxRequestsPerHost");

		String[] interceptorsArray = interceptors.split(",");
		// 同步httpclient
		OkHttpClientFactory.AsyncBuilder builder = new OkHttpClientFactory.AsyncBuilder()
				.connectTimeout(connectTimeout)
				.readTimeout(readTimeout)
				.writeTimeout(writeTimeout)
				.corePoolSize(corePoolSize)
				.maximumPoolSize(maximumPoolSize)
				.maxRequests(maxRequests)
				.maxRequestsPerHost(maxRequestsPerHost)
				.sslSocketFactory(httpsUtils.sSLSocketFactory, httpsUtils.trustManager);
		for (int i = 0; i > interceptorsArray.length; i++) {
			String className = interceptorsArray[i];
			if (null != className) {
				try {
					Interceptor interceptorObj = (Interceptor) (Class.forName(className).newInstance());
					builder.add((Interceptor) interceptorObj);
				} catch (InstantiationException e) {
				} catch (IllegalAccessException e) {
				} catch (ClassNotFoundException e) {
				}
			}
		}
		OkHttpClient syncHttpClient = builder.build();
		return syncHttpClient;
	}
}
```
### (3). 定义配置(Module)
```
package help.lixin.config;

import com.google.inject.AbstractModule;
import com.google.inject.Scopes;
import com.google.inject.name.Names;

import okhttp3.OkHttpClient;

public class Module extends AbstractModule {

	/**
	 * 如果有指定名字的,只能通过Class类型+名字去获得对象.
	 */
	@Override
	protected void configure() {
		bind(BodyBuilder.class).annotatedWith(Names.named("formJsonBodyBuilder")).to(FormJsonBodyBuilder.class)
				.in(Scopes.SINGLETON);
		bind(BodyBuilder.class).annotatedWith(Names.named("formBodyBuilder")).to(FormBodyBuilder.class)
				.in(Scopes.SINGLETON);
		// 构建同步的http client
		bind(OkHttpClient.class).annotatedWith(Names.named("syncHttpClient")).toProvider(SyncOkHttpClientProvider.class)
				.in(Scopes.SINGLETON);
		// 判断是否开启了异步回调开关
		String asyncSwtich = PropertiesStore.getInstance().getPropertie("async.swtich", "false");
		if ("TRUE".equalsIgnoreCase(asyncSwtich)) {
			// 构建异步的http client
			bind(OkHttpClient.class).annotatedWith(Names.named("asyncHttpClient"))
					.toProvider(AsyncOkHttpClientProvider.class).in(Scopes.SINGLETON);
		}
		// 构建请求对象
		bind(RequestService.class).to(RequestServiceImpl.class).in(Scopes.SINGLETON);
		// 处理返回值
		bind(HandlerReturnValue.class).to(FatJsonHandlerReturnValueImpl.class).in(Scopes.SINGLETON);
	}
}
```
### (4). 定义Client使用
```
package help.lixin.container;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.Key;
import com.google.inject.name.Names;
import help.lixin.config.Module;

public class Container {
	private static Injector INJECTOR = null;

	private Container() {
	}

	static class ContainerHolder {
		private static final Container CONTAINER = new Container();
	}

	public static Container getInstnace() {
		return ContainerHolder.CONTAINER;
	}

	public void init(com.google.inject.Module... modules) {
		List<com.google.inject.Module> moduleList = new ArrayList<com.google.inject.Module>();
		moduleList.add(new Module());
		if (null != modules) {
			List<com.google.inject.Module> list = Arrays.asList(modules);
			moduleList.addAll(list);
		}
		if (null == INJECTOR) {
			synchronized (Container.class) {
				if (null == INJECTOR) {
					INJECTOR = Guice.createInjector(moduleList);
				}
			}
		}
	}

	public Object getBean(Class<?> clazz) {
		return INJECTOR.getInstance(clazz);
	}

	public Object getBean(Class<?> clazz, String name) {
		return INJECTOR.getInstance(Key.get(clazz, Names.named(name)));
	}
}
```
