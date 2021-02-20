---
layout: post
title: 'XXL-Job XxlJobConfig(二)'
date: 2021-02-21
author: 李新
tags: XXL-Job源码
---

### (1). 找到XxlJobSpringExecutor的入口
> XxlJobSpringExecutor是由:XxlJobConfig(Spring会自动扫描该类)类进行配置的.   

```
package com.xxl.job.executor.core.config;

import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class XxlJobConfig {
    private Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);
	
	// xxl-job-admin的地址
    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;

	// 通过accessToken增加安全管理
    @Value("${xxl.job.accessToken}")
    private String accessToken;

	// 执行器的名字(会向xxl-job-admin注册访名称).
    @Value("${xxl.job.executor.appname}")
    private String appname;

    @Value("${xxl.job.executor.address}")
    private String address;

	// 执行器的IP地址(本机[内/外]IP地址)
    @Value("${xxl.job.executor.ip}")
    private String ip;

	// 执行器要开启的端口(注意:要规划好端口)
    @Value("${xxl.job.executor.port}")
    private int port;

	// 执行器的日志path
    @Value("${xxl.job.executor.logpath}")
    private String logPath;

	// 以天为单位,对执行器的日志进行轮询处理.
    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;

	// ************************************************************
	// 2. 创建:XxlJobExecutor
	// ************************************************************
    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppname(appname);
        xxlJobSpringExecutor.setAddress(address);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

        return xxlJobSpringExecutor;
    }
}
```
### (2). 看下XxlJobSpringExecutor类的继承关系图
!["XxlJobSpringExecutor"](/assets/xxl-job/imgs/XxlJobSpringExecutor.jpg)

> SmartInitializingSingleton : XxlJobSpringExecutor实现了该接口,在Spring初始化完之后,回调:afterSingletonsInstantiated方法.
### (3). 总结
> XxlJobSpringExecutor顾名意思:即任务执行器,它的职责如下:  
> 1. 向xxl-job-admin进行注册.   
> 2. 通过Netty开启HTTP监听.   
> 3. 等待xxl-job-admin分派任务,并执行任务.   