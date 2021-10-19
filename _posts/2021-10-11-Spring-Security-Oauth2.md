---
layout: post
title: 'Spring Security与Oauth2整合(一)' 
date: 2021-10-11
author: 李新
tags:  SpringSecurity
---

### (1). 概述
Oauth2一直都比较流行,在这里稍微的拿个小小的Demo来入门,后面,会对源码进行剖析.     

### (2). 项目结构
```
infinova@lixin Workspace % tree spring-security-oauth-example 
spring-security-oauth-example
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── oauth
│   │   │               ├── Application.java
│   │   │               ├── config
│   │   │               │   ├── AuthorizeConfig.java
│   │   │               │   ├── JwtTokenStoreConfig.java
│   │   │               │   ├── ResourceConfig.java
│   │   │               │   └── SecurityConfig.java
│   │   │               ├── controller
│   │   │               │   └── UserController.java
│   │   │               └── service
│   │   │                   └── UserService.java
│   │   └── resources
│   └── test
│       └── java
└── target
```
### (3). UserService
```
package help.lixin.oauth.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class UserService implements UserDetailsService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    public void setPasswordEncoder(PasswordEncoder passwordEncoder) {
        this.passwordEncoder = passwordEncoder;
    }

    public PasswordEncoder getPasswordEncoder() {
        return passwordEncoder;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 登录的用户和密码
        return User.withUsername("admin")
                .password(passwordEncoder.encode("111111"))
                .authorities("/add", "/test")
                .build();
    }
}
```
### (4). SecurityConfig
```
package help.lixin.oauth.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 关闭csrf防护
        http.csrf().disable();
        http.authorizeRequests()
                // 允许部份请求可以访问
                .antMatchers("/oauth/**", "/login/**", "/logout/**").permitAll()
                // 其余所有的请求,都要认证后才能访问
                .anyRequest().authenticated()
                .and()
                // 允许表单登录的访问
                .formLogin().permitAll();
    }
}
```
### (5). AuthorizeConfig
```
package help.lixin.oauth.config;

import help.lixin.oauth.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.oauth2.config.annotation.configurers.ClientDetailsServiceConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerEndpointsConfigurer;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;

/**
 * 授权服务器配置
 */
@Configuration
// 1. 开启Authorization注解
@EnableAuthorizationServer
public class AuthorizeConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private UserService userService;

    @Autowired
    private JwtTokenStore jwtTokenStore;

    @Autowired
    private JwtAccessTokenConverter jwtAccessTokenConverter;

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        // 密码模式下,需要配置:AuthenticationManager和UserDetailsService
        endpoints
                .userDetailsService(userService)
                .tokenStore(jwtTokenStore)
                .accessTokenConverter(jwtAccessTokenConverter);
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        // 暂时放在内存里
        clients.inMemory()
                .withClient("test")
                .secret(passwordEncoder.encode("test"))
                // .accessTokenValiditySeconds(3600)
                .redirectUris("http://www.baidu.com")
                .scopes("all")
                // 授权码模式/以及refresh_token
                .authorizedGrantTypes("authorization_code", "refresh_token");
    }
}
```
### (6). ResourceConfig
```
package help.lixin.oauth.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;
import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;

/**
 * 资源服务器配置
 */
@Configuration
// 1. 开启注解(@EnableResourceServer)
@EnableResourceServer
public class ResourceConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .requestMatchers().antMatchers("/user/**");
    }
}
```
### (7). JwtTokenStoreConfig
```
package help.lixin.oauth.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;

/**
 * jwt token配置
 */
@Configuration
public class JwtTokenStoreConfig {

    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        // 配置jwt使用的密钥
        jwtAccessTokenConverter.setSigningKey("hello");
        return jwtAccessTokenConverter;
    }

    @Bean
    public JwtTokenStore jwtTokenStore(JwtAccessTokenConverter jwtAccessTokenConverter) {
        return new JwtTokenStore(jwtAccessTokenConverter);
    }
}
```
### (8). UserController
```
package help.lixin.oauth.controller;

import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.Serializable;
import java.util.Objects;

@RestController
@RequestMapping("/user")
public class UserController {

    /**
     * 获取当前用户
     *
     * @param authentication
     * @return
     */
    @GetMapping("/get")
    public User user(Authentication authentication) {
        Object principal = authentication.getPrincipal();
        User user = new User();
        user.setId("100");
        user.setName("zhangsan");
        return user;
    }
}

class User implements Serializable {
    private String id;
    private String name;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(id, user.id) && Objects.equals(name, user.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }

    @Override
    public String toString() {
        return "User{" +
                "id='" + id + '\'' +
                ", name='" + name + '\'' +
                '}';
    }
}
```
### (9).Application
```
package help.lixin.oauth;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
### (10). 引导用户至登录页面
```
http://localhost:8080/oauth/authorize?response_type=code&client_id=test&redirect_uri=http://www.baidu.com&scope=all
```

!["用户至登录页面"](/assets/spring-security/imgs/spring-security-login-page.png)
### (12). 用户授权登录
!["用户授权登录"](/assets/spring-security/imgs/spring-security-approval.png)
### (13). 重定向到客户端,并返回code
!["code"](/assets/spring-security/imgs/spring-security-code.png)
### (14). 通过code换access_token
> 设置client/secret/code.   


!["设置client和secret"](/assets/spring-security/imgs/spring-security-access-token-1.png)
!["设置code"](/assets/spring-security/imgs/spring-security-access-token-2.png)
### (14). 使用access_token请求资源服务器
!["通过access_token请求资源服务器"](/assets/spring-security/imgs/spring-security-resource.png)
### (15). 使用refresh_token重新生成access_token
!["refresh_token重新生成access_token"](/assets/spring-security/imgs/spring-security-refresh-token.png)