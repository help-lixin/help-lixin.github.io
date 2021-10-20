---
layout: post
title: 'Spring Security简单入门案例' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
在这里先对Spring Security进行一个简单的入门,入门案例的要求是,判断用户是否具备有URL访问的权限.

### (2). 项目目录结构如下
```
lixin@lixin GitWorkspace % tree spring-security-example 
spring-security-example
├── README.md
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── security
│   │   │               └── example
│   │   │                   ├── Application.java
│   │   │                   ├── access
│   │   │                   │   └── PermissionVaidator.java
│   │   │                   ├── config
│   │   │                   │   └── WebSecurityConfig.java
│   │   │                   ├── controller
│   │   │                   │   └── HelloController.java
│   │   │                   └── userdetails
│   │   │                       └── DefaultUserDetailsService.java
│   │   └── resources
│   │       └── static
│   │           └── login.html
│   └── test
└── target
```
### (3). WebSecurityConfig
```
package help.lixin.security.example.config;

import help.lixin.security.example.userdetails.DefaultUserDetailsService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.annotation.web.configurers.ExpressionUrlAuthorizationConfigurer;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();


        // 注意:当你发现调用不了UserDetailsService的时候,要注意关闭csrf
        http.formLogin()
                // 登录成功后,必须要是POST请求
                // .successForwardUrl("/main.html")
                // 登录页面
                .loginPage("/login.html")
                // 登录处理URL(Filter[UsernamePasswordAuthenticationFilter])
                .loginProcessingUrl("/login");


        ExpressionUrlAuthorizationConfigurer<HttpSecurity>.ExpressionInterceptUrlRegistry expressionInterceptUrlRegistry = http.authorizeRequests();
        expressionInterceptUrlRegistry
                // 允许/login.html不需要认证.
                .antMatchers("/login.html").permitAll()
                // .antMatchers("/hello").hasRole("ADMIN")
                // 所有请求都要经过access验证,才能访问
                .anyRequest().access("@permissionVaidator.hasPermission(request,authentication)");
    }

    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder passwordEncoder) {
        UserDetailsService userDetailsService = new DefaultUserDetailsService(passwordEncoder);
        return userDetailsService;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```
### (4). DefaultUserDetailsService
```
package help.lixin.security.example.userdetails;

import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;

// 加载用户信息.
public class DefaultUserDetailsService implements UserDetailsService {

    private PasswordEncoder passwordEncoder;

    public DefaultUserDetailsService(PasswordEncoder passwordEncoder) {
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // TOOD lixin 去Repository中Query,如果,查找不到,则抛出异常:UsernameNotFoundException

        UserDetails user = User.withUsername("lixin")
                .password(passwordEncoder.encode("123456"))
                // 用户具有哪些资源(URI)的权限
                .authorities("/hello", "/test")
                .build();
        return user;
    }
}
```
### (5). PermissionVaidator
```
package help.lixin.security.example.access;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import java.util.Collection;

/**
 * 自定义权限验证
 */
@Component
public class PermissionVaidator {

    public boolean hasPermission(HttpServletRequest request, Authentication authentication) {
        // 请求的URL
        String requestURI = request.getRequestURI();

        // 用户登录后的权限
        Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
        for (GrantedAuthority grantedAuthority : authorities) {
            // 建议这里用Ant表达式对URL进行判断,准确性会更好一些.
            if (grantedAuthority.getAuthority().equals(requestURI)) {
                return true;
            }
        }
        return false;
    }
}
```
### (6). HelloController
```
package help.lixin.security.example.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }
}
```
### (7). Application
```
package help.lixin.security.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
### (8). login.html
```
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <title>user login</title>
</head>
<body>
    <form action="/login" method="post">
        用户名:<input type="text" name="username">    <br/>
        密码:<input type="password" name="password">  <br/>
        <input type="submit" value="登录">
    </form>
</body>
</html>
```
### (9). pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>help.lixin.security.example</groupId>
    <artifactId>spring-security-example</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.9.RELEASE</version>
        <relativePath />
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
	
</project>
```
### (10). 总结
通过Spring Security的简单配置,就可以实现对URL资源的认证和鉴权操作.   