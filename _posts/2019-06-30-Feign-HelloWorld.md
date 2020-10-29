---
layout: post
title: 'Sprig Cloud Feign HelloWorld'
date: 2019-06-30
author: 李新
tags: Feign
---

### (1).引入feign
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
### (2).启用Feign 
```
package help.lixin.samples;

import java.util.ArrayList;
import java.util.List;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Bean;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.support.BasicAuthenticationInterceptor;
import org.springframework.web.client.RestTemplate;

@EnableFeignClients
@EnableDiscoveryClient
@EnableEurekaClient
@SpringBootApplication
public class ConsumerApplication {

	@Bean
	@LoadBalanced
	RestTemplate restTemplate() {
		ClientHttpRequestInterceptor basicAuthenticationInterceptor = new BasicAuthenticationInterceptor("lixin", "123456");
		List<ClientHttpRequestInterceptor> interceptors = new ArrayList<ClientHttpRequestInterceptor>();
		interceptors.add(basicAuthenticationInterceptor);
		RestTemplate restTemplate = new RestTemplate();
		restTemplate.setInterceptors(interceptors);
		return restTemplate;
	}

	public static void main(String[] args) {
		SpringApplication.run(ConsumerApplication.class, args);
	}
}
```
### (3).HelloService
```
package help.lixin.samples.service;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient("TEST-PROVIDER")
public interface HelloService {

	@GetMapping("/hello")
	String hello();

}
```
### (4).ConsumerController
```
package help.lixin.samples.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import help.lixin.samples.service.HelloService;

@RestController
public class ConsumerController {
	private Logger logger = LoggerFactory.getLogger(ConsumerController.class);

	@Autowired
	private RestTemplate restTemplate;
	
	@Autowired
	private HelloService helloService;

	@GetMapping("/consumer")
	public String index() {
		logger.info("====================trace1====================");
		String localStr = "consumer...";
		String url = "http://test-provider/hello";
//		String url = "http://localhost:8080/hello";
//		String result = restTemplate.getForEntity(url, String.class).getBody();
		String result = helloService.hello();
		return localStr + result;
	}
}
```
