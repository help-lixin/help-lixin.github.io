---
layout: post
title: 'mapstruct与Spring整合' 
date: 2021-10-04
author: 李新
tags:  mapstruct
---

### (1). 概述
前面对mapstruct有一个简单的入门,在这里,将能过Spring与mapstruct进行整合.

### (2). 项目结构如下
```
lixin-macbook:mapstruct-demo lixin$ tree 
.
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── help
│   │   │       └── lixin
│   │   │           └── mapstruct
│   │   │               ├── Application.java
│   │   │               ├── controller
│   │   │               │   └── UserController.java
│   │   │               ├── convert
│   │   │               │   └── UserConvert.java
│   │   │               ├── dto
│   │   │               │   └── UserDTO.java
│   │   │               ├── entity
│   │   │               │   └── User.java
│   │   │               ├── service
│   │   │               │   ├── IUserService.java
│   │   │               │   └── impl
│   │   │               │       └── UserService.java
│   │   │               └── vo
│   │   └── resources
│   └── test
│       └── java
└── target
```
### (3). UserController
```
package help.lixin.mapstruct.controller;

import help.lixin.mapstruct.dto.UserDTO;
import help.lixin.mapstruct.service.IUserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
public class UserController {

    @Autowired
    private IUserService userService;


    @GetMapping("/all")
    public List<UserDTO> all() {
        return userService.queryAll();
    }
}
```
### (4). IUserService
```
package help.lixin.mapstruct.service;

import help.lixin.mapstruct.dto.UserDTO;

import java.util.List;

public interface IUserService {
    List<UserDTO> queryAll();
}
```
### (5). UserService
```
package help.lixin.mapstruct.service.impl;

import help.lixin.mapstruct.convert.UserConvert;
import help.lixin.mapstruct.dto.UserDTO;
import help.lixin.mapstruct.entity.User;
import help.lixin.mapstruct.service.IUserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Service
public class UserService implements IUserService {

    @Autowired
    private UserConvert userConvert;

    @Override
    public List<UserDTO> queryAll() {
        // 1. 模拟从repository中获取数据
        List<User> users = mockRepositoryQuery();

        List<UserDTO> userDTOS = userConvert.toUserDTO(users);
        return userDTOS;
    }

    protected List<User> mockRepositoryQuery() {
        List<User> users = new ArrayList<>();

        User user1 = new User();
        user1.setId(1l);
        user1.setName("张三");
        user1.setAge(15);


        User user2 = new User();
        user2.setId(1l);
        user2.setName("张三");
        user2.setAge(15);

        users.add(user1);
        users.add(user2);
        return users;
    }
}
```
### (6). User
```
package help.lixin.mapstruct.entity;

import java.io.Serializable;

public class User implements Serializable {
    private Long id;
    private String name;
    private Integer age;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```
### (7). UserDTO
```
package help.lixin.mapstruct.dto;

import java.io.Serializable;

public class UserDTO implements Serializable {
    private Long uesrId;
    private String userName;
    private Integer age;

    public Long getUesrId() {
        return uesrId;
    }

    public void setUesrId(Long uesrId) {
        this.uesrId = uesrId;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "UserDTO{" +
                "uesrId=" + uesrId +
                ", userName='" + userName + '\'' +
                ", age=" + age +
                '}';
    }
}
```
### (8). UserConvert
```
package help.lixin.mapstruct.convert;

import help.lixin.mapstruct.dto.UserDTO;
import help.lixin.mapstruct.entity.User;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;

import java.util.List;

// ****************************************************************
// 指定componentModel
// 实际就是在生成的源码里,添加不同的注解,来标识这个类.
// ****************************************************************
@Mapper(componentModel = "spring")
public interface UserConvert {

    @Mappings({
            @Mapping(source = "id", target = "uesrId"),
            @Mapping(source = "name", target = "userName")
    })
    UserDTO toUserDTO(User user);


    @Mappings({
            @Mapping(source = "users.id", target = "uesrId"),
            @Mapping(source = "users.name", target = "userName")
    })
    List<UserDTO> toUserDTO(List<User> users);
}
```
### (9). Application
```
package help.lixin.mapstruct;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
### (10).  pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>help.lixin.mapstruct</groupId>
    <artifactId>mapstruct-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.9.RELEASE</version>
        <relativePath />
    </parent>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <spring-cloud.version>Finchley.SR4</spring-cloud.version>
        <spring-boot.version>2.0.9.RELEASE</spring-boot.version>
        <org.mapstruct.version>1.5.0.Beta1</org.mapstruct.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${org.mapstruct.version}</version>
        </dependency>

        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct-processor</artifactId>
            <version>${org.mapstruct.version}</version>
            <scope>provided</scope>
        </dependency>
        <!-- Springboot依赖 ,包含日志依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <version>${spring-boot.version}</version>
        </dependency>

        <!-- Springboot-web依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>${spring-boot.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <!-- 指定注解的处理 -->
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${org.mapstruct.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```
### (11). mapstruct原理
mapstruct没有使用反射,而是在,编译期生成了一个对应的类,来辅助我们做set/get.

```
# 1. 编译
lixin-macbook:mapstruct-demo lixin$ mvn compile

# 2. 会在UserConvert接口同级目录下创建一个实现类:UserConvertImpl
lixin-macbook:mapstruct-demo lixin$ tree target/classes/help/lixin/mapstruct/convert/
target/classes/help/lixin/mapstruct/convert/
├── UserConvert.class
└── UserConvertImpl.class

# 3. 反编译:UserConvertImpl

package help.lixin.mapstruct.convert;

import help.lixin.mapstruct.dto.UserDTO;
import help.lixin.mapstruct.entity.User;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import org.springframework.stereotype.Component;

// ****************************************************************
// Spring注解
// ****************************************************************
@Component
public class UserConvertImpl implements UserConvert {
    public UserConvertImpl() {
    }

    public UserDTO toUserDTO(User user) {
        if (user == null) {
            return null;
        } else {
            UserDTO userDTO = new UserDTO();
            userDTO.setUesrId(user.getId());
            userDTO.setUserName(user.getName());
            userDTO.setAge(user.getAge());
            return userDTO;
        }
    }

    public List<UserDTO> toUserDTO(List<User> users) {
        if (users == null) {
            return null;
        } else {
            List<UserDTO> list = new ArrayList(users.size());
            Iterator var3 = users.iterator();

            while(var3.hasNext()) {
                User user = (User)var3.next();
                list.add(this.toUserDTO(user));
            }

            return list;
        }
    }
}
```
### (12). 总结
mapstruct底层实际是通过标识的注解(@Mapper),来帮我们动态生成类,并把类标注为Spring的组件(@Component).   
