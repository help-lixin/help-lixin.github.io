---
layout: post
title: 'Nacos Client简单使用(三)'
date: 2019-05-01
author: 李新
tags: Nacos
---

### (1). 添加依赖
```
<dependency>
	<groupId>com.alibaba.nacos</groupId>
	<artifactId>nacos-client</artifactId>
	<version>1.3.0</version>
</dependency>
```
### (2). App.java
```
package help.lixin.nacos;

import java.util.Properties;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Executor;
import java.util.concurrent.TimeUnit;

import com.alibaba.nacos.api.NacosFactory;
import com.alibaba.nacos.api.PropertyKeyConst;
import com.alibaba.nacos.api.config.ConfigService;
import com.alibaba.nacos.api.config.listener.Listener;
import com.alibaba.nacos.api.exception.NacosException;

public class App {
	public static void main(String[] args) throws NacosException, InterruptedException {
		// 需要先创建namespace,获得:命名空间ID
		// 在配置列表有TAB可以切换不同的namespace,新建配置
		String namespace = "45a6112b-a866-4e92-a8d6-9e440bcdebbe";
		String serverAddr = "localhost:8848";
		String dataId = "application.properties";
		String group = "erp:sso-service";

		Properties properties = new Properties();
		properties.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
		properties.put(PropertyKeyConst.NAMESPACE, namespace);
		// 通过NacosFactory工厂,创建ConfigService
		ConfigService configService = NacosFactory.createConfigService(properties);

		// 5000 超时时间
		String content = configService.getConfig(dataId, group, 5000);
		
		// 监听配置的变化,当有配置变化时,推送给:Listener
		configService.addListener(dataId, group, new Listener() {
			@Override
			public void receiveConfigInfo(String configInfo) {
				System.out.println("recieve :" + configInfo);
			}

			@Override
			public Executor getExecutor() {
				return null;
			}
		});
		
		System.out.println(content);
		CountDownLatch latch = new CountDownLatch(1);
		latch.await(5, TimeUnit.MINUTES);
	}
}

```

### (3). 测试运行结果
```
user.login.url=http://www.lixin.help
user.image.url=http://image.lixin.help/logo.jpg

recieve : user.login.url=http://www.lixin.help/test
user.image.url=http://image.lixin.help/logo.jpg
```