---
layout: post
title: 'Guice 案例(三)'
date: 2020-12-12
author: 李新
tags: Guice
---

### (1). 定义接口
```
package help.lixin.guice.service;

public interface UserService {
    void sayHello(String name, int age);
}
```
### (2). 定义实现类(UserServiceOneImpl)
```
package help.lixin.guice.service.impl;

import javax.inject.Singleton;
import help.lixin.guice.service.UserService;

@Singleton
public class UserServiceOneImpl implements UserService {
    public void sayHello(String name, int age) {
        System.err.println("UserServiceOneImpl Hello :" + name + " age:" + age);
    }
}
```
### (3). 定义实现类(UserServiceTwoImpl)
```
package help.lixin.guice.service.impl;

import javax.inject.Singleton;
import help.lixin.guice.service.UserService;

@Singleton
public class UserServiceTwoImpl implements UserService {
    public void sayHello(String name, int age) {
        System.err.println("UserServiceTwoImpl Hello :" + name + " age:" + age);
    }
}
```
### (4). 定义配置文件(Module)
```
package help.lixin.guice.config;
import com.google.inject.AbstractModule;
import com.google.inject.name.Names;

import help.lixin.guice.service.UserService;
import help.lixin.guice.service.impl.UserServiceOneImpl;
import help.lixin.guice.service.impl.UserServiceTwoImpl;

public class BeanConfig extends AbstractModule {
    @Override
    protected void configure() {			
        bind(UserService.class).annotatedWith(Names.named("userOne")).to(UserServiceOneImpl.class);		

        bind(UserService.class).annotatedWith(Names.named("userTwo")).to(UserServiceTwoImpl.class);
    } //end configure

    @Provides
    public List<UserService> userList() {
        List<UserService> users = new ArrayList<UserService>(2);
        users.add(new UserServiceOneImpl());
        users.add(new UserServiceTwoImpl());
        return users;
    } // end userList
}

```
### (5). 测试
```
package help.lixin.guice;

import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.Key;
import com.google.inject.name.Names;

import help.lixin.guice.config.BeanConfig;
import help.lixin.guice.service.UserService;

public class App {
    public static void main(String[] args) {

        Injector injector = Guice.createInjector(new BeanConfig());
        UserService userOne = injector.getInstance(Key.get(UserService.class, Names.named("userOne")));
        UserService userTwo = injector.getInstance(Key.get(UserService.class, Names.named("userTwo")));

        userOne.sayHello("张三", 25);
        userTwo.sayHello("李四", 40);


        Type type = new ParameterizedType() {
        @Override
        public Type getRawType() {
        return List.class;
        }

        @Override
        public Type getOwnerType() {
        return null;
        }

        @Override
        public Type[] getActualTypeArguments() {
        return new Type[]{UserService.class};
        }
        };

        // 根据提供者类型来获得提供者:        
        Provider p1 = injector.getProvider(Key.get(type));
        System.out.println(p1);

        // 获得提供者
        Provider p = injector.getProvider(
            Key.get( new TypeToken<List<UserService>>() {}.getType()));
        // List
        System.out.println(p.get());

        // 获得对象		
        List<UserService>  list = (List)injector.getInstance(
        Key.get(new TypeToken<List<UserService>>() {}.getType()));
        System.out.println(list);
    }
}
```
