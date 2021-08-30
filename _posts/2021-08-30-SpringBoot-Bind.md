---
layout: post
title: 'Binder详解' 
date: 2021-08-30
author: 李新
tags:  SpringBoot
---

### (1). 背景
Gateway与Apollo进行了整合,每次新增一个微服务,都需要重启Gateway,而这类型的应用,尽可能的越少岩机越好,所以:能否做到配置的热更新呢?   
GatewayProperties是Gateway的数据载体,最终承载着所有的数据,每次Apollo刷新时,能否读取Environment对象,把配置文件信息转换成业务模型呢?而不是手动去new对象呢?

### (2). application.properties
```
user.name=zhgnsan
user.age=25
user.address[0].addr=gd
user.address[1].addr=bj
```
### (3). Address
```
package help.lixin.samples.model;

import java.io.Serializable;

public class Address implements Serializable {
	private static final long serialVersionUID = -5417183600602968242L;
	private String addr;

	public String getAddr() {
		return addr;
	}

	public void setAddr(String addr) {
		this.addr = addr;
	}
}
```
### (4). User
```
package help.lixin.samples.model;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;

public class User implements Serializable {
	private static final long serialVersionUID = -3758881566559902069L;
	private String name;
	private int age;
	private List<Address> address = new ArrayList<Address>(0);

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

	public List<Address> getAddress() {
		return address;
	}

	public void setAddress(List<Address> address) {
		this.address = address;
	}
}
```
### (5). HelloController
```
package help.lixin.samples.controller;

import java.util.List;

import org.springframework.boot.context.properties.bind.Bindable;
import org.springframework.boot.context.properties.bind.Binder;
import org.springframework.context.EnvironmentAware;
import org.springframework.core.env.Environment;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import help.lixin.samples.model.Address;
import help.lixin.samples.model.User;

@RestController
public class HelloController implements EnvironmentAware {

	private Environment environment;

	@GetMapping("/test")
	public String hello() {
		// 1. 绑定对象
		User user = Binder.get(environment)
		                  .bind("user", User.class)
		                  .get();
		System.out.println(user);
		
		// 2. 绑定集合
		List<Address>  address = Binder.get(environment)
				                       .bind("user.address", Bindable.listOf(Address.class))
				                       .get();
		System.out.println(address);
		return "Hello World!!!";
	}

	public void setEnvironment(Environment environment) {
		this.environment = environment;
	}
}

```
### (6). 总结
通过Bind可以轻松的解决,配置文件到业务模型的转换,省去了手工new对象的过程.  
