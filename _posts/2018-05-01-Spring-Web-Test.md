---
layout: post
title: 'SpringBoot Web单元测试'
date: 2018-05-01
author: 李新
tags: SpringBoot
---

### (1). 需求
> 随着微服务的流行,前后端分离,这时候,后端不再像以前一样,可以有一界面进行测试了,随之开发人员的代码质量也出现了问题.  
> 那么,如何解决呢?  
> 解决方案是,开发人员应该写单元测试,单元测试的目的是让测试用例尽可能的覆盖到分支上,然后,在测试时要关闭类与类之间的依赖关系.   

### (2). User
```
package help.lixin.test.model;

import java.io.Serializable;

public class User implements Serializable {
	private static final long serialVersionUID = 4953463227600668630L;
	private String name;
	private int age;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	@Override
	public String toString() {
		return "User [name=" + name + ", age=" + age + "]";
	}
}
```
### (3). HelloController
```
package help.lixin.test.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import help.lixin.test.model.User;
import help.lixin.test.service.IHelloService;

@RestController
public class HelloController {
	@Autowired
	private IHelloService helloService;

	@GetMapping("/")
	public String home() {
		System.err.println("===========/home===================");
		return "Hello World!";
	}

	@GetMapping("/var1")
	public String var1(@RequestParam("name") String name) {
		System.err.println("===========/var1===================");
		return "Hello World VAR1!";
	}

	@PostMapping("/var2")
	public String var2(@RequestParam("name") String name) {
		System.err.println("===========/var2===================");
		return "Hello World VAR2!";
	}

	@PostMapping("/json")
	public User json(@RequestBody User user) {
		// 调了业务接口
		helloService.sayhello();
		System.err.println("===========/var json===================");
		User result = new User();
		result.setName("LIXIN");
		result.setAge(user.getAge());
		return result;
	}
}
```
### (4). IHelloService
```
package help.lixin.test.service;

public interface IHelloService {
	public void sayhello();
}
```
### (5). HelloService
```
package help.lixin.test.service.impl;

import org.springframework.stereotype.Service;

import help.lixin.test.service.IHelloService;

@Service
public class HelloService implements IHelloService {

	public void sayhello() {
		System.out.println("=========================hello world=========================");
	}
}
```

### (6). LogFilter
```
package help.lixin.test.filter;

import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

public class LogFilter implements Filter {

	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		System.out.println("**********************LogFilter start******************************");
		chain.doFilter(request, response);
		System.out.println("**********************LogFilter end******************************");
	}
}
```

### (7). TestConfig
```
package help.lixin.test;

import javax.servlet.Filter;

import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import help.lixin.test.filter.LogFilter;

@Configuration
public class TestConfig {
	
	@Bean
	public Filter logFilter() {
		return new LogFilter();
	}
	
	@Bean
    public FilterRegistrationBean filterRegistrationBean(Filter logFilter){
        FilterRegistrationBean bean = new FilterRegistrationBean();
        bean.setFilter(logFilter);
        bean.addUrlPatterns("/*");
        return bean;
    }
}
```
### (8). WebMvcTest
```
package help.lixin.test;


import static org.hamcrest.Matchers.containsString;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import static org.mockito.Mockito.*;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;

import com.fasterxml.jackson.databind.ObjectMapper;

import help.lixin.test.controller.HelloController;
import help.lixin.test.model.User;
import help.lixin.test.service.IHelloService;

// @RequestBody时需要加入:@EnableWebMvc,否则出现405请求
@EnableWebMvc
@RunWith(SpringRunner.class)
// 配置仅仅针对哪个类进行测试
@SpringBootTest(classes = { HelloController.class ,TestConfig.class })
@AutoConfigureMockMvc
public class WebMvcTest {

	@Autowired
	private MockMvc mockMvc;

	@MockBean
	// 创建一个虚拟的Bean
	private IHelloService helloService;

	private ObjectMapper objectMapper;

	@Before
	public void init() {
		// 如果自动注入,则需要配合:@AutoConfigureMockMvc注解
		// 否则,需要像下面这样手写.
		// mockMvc = MockMvcBuilders.standaloneSetup(new HelloController()).build();
		objectMapper = new ObjectMapper();
	}

	@Test
	public void testHome() throws Exception {
		mockMvc.perform(get("/")).andDo(print()).andExpect(status().isOk())
				.andExpect(content().string(containsString("Hello World!")));
	}

	@Test
	public void tetsVar1() throws Exception {
		mockMvc.perform(get("/var1?name=", "zhangsan")).andDo(print()).andExpect(status().isOk())
				.andExpect(content().string(containsString("Hello World VAR1!")));
	}

	@Test
	public void tetsVar2() throws Exception {
		mockMvc.perform(post("/var2").param("name", "zhansan")).andDo(print()).andExpect(status().isOk())
				.andExpect(content().string(containsString("Hello World VAR2!")));
	}

	@Test
	public void testJson() throws Exception {
		// 无参无返回值处理方法的Mock
		doNothing().when(helloService).sayhello();
		
		User param = new User();
		param.setAge(34);
		param.setName("ZHANGSAN");
		String requestString = objectMapper.writeValueAsString(param);
		// JSON请求时,需要加上注解:@EnableWebMvc
		mockMvc.perform(post("/json").contentType(MediaType.APPLICATION_JSON_UTF8_VALUE).content(requestString))
				.andDo(print()).andExpect(status().isOk())
				.andExpect(MockMvcResultMatchers.jsonPath("$.name").value("LIXIN"));
	}
}
```
### (7). pom.xml
```
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>Greenwich.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-dependencies</artifactId>
			<version>2.1.0.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-netflix</artifactId>
			<version>2.1.1.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
	<dependency>
		<groupId>junit</groupId>
		<artifactId>junit</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```
### (8). 总结
> 通过MockMvc可以解决对Controller测试(因为依赖容器嘛),而:@MockBean可以解决类与类之间的关系.    
> ["mockito学习参考"](https://www.letianbiji.com/java-mockito/mockito-mock.html)