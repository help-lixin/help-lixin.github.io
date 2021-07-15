---
layout: post
title: 'Apollo 部署与集成(二)'
date: 2021-07-15
author: 李新
tags:  Apollo
---

### (1). 概述
前面稍看了下Apollo架构,在这一小节,主要搭建把Apollo搭建起来.   

### (2). 源码下载
```
lixin-macbook:~ lixin$ cd ~/GitRepository/
lixin-macbook:GitRepository lixin$ git clone https://github.com/ctripcorp/apollo.git
lixin-macbook:GitRepository lixin$ cd apollo/
```
### (3). 执行SQL脚本
```
# 1. 导入SQL脚本,创建数据库
mysql> SOURCE /Users/lixin/GitRepository/apollo/scripts/sql/apolloconfigdb.sql
mysql> SOURCE /Users/lixin/GitRepository/apollo/scripts/sql/apolloportaldb.sql

# 2. 检查ApolloPortalDB
mysql> select `Id`, `Key`, `Value`, `Comment` from `ApolloPortalDB`.`ServerConfig` limit 1;
+----+--------------------+-------+--------------------------+
| Id | Key                | Value | Comment                  |
+----+--------------------+-------+--------------------------+
|  1 | apollo.portal.envs | dev   | 可支持的环境列表         |
+----+--------------------+-------+--------------------------+

# 3. 检查ApolloConfigDB
mysql> select `Id`, `Key`, `Value`, `Comment` from `ApolloConfigDB`.`ServerConfig` limit 1;
+----+--------------------+-------------------------------+------------------------------------------------------+
| Id | Key                | Value                         | Comment                                              |
+----+--------------------+-------------------------------+------------------------------------------------------+
|  1 | eureka.service.url | http://localhost:8080/eureka/ | Eureka服务Url，多个service以英文逗号分隔             |
+----+--------------------+-------------------------------+------------------------------------------------------+
```
### (4). 配置(/Users/lixin/GitRepository/apollo/scripts/build.sh)
```
# apollo config db info
# apollo-configservice/apollo-adminservice共享一个db
apollo_config_db_url='jdbc:mysql://127.0.0.1:3306/ApolloConfigDB?characterEncoding=utf8'
apollo_config_db_username=root
apollo_config_db_password=123456

# apollo portal db info
# portal 独有db
apollo_portal_db_url='jdbc:mysql://127.0.0.1:3306/ApolloPortalDB?characterEncoding=utf8'
apollo_portal_db_username=root
apollo_portal_db_password=123456


# meta server url, different environments should have different meta server addresses
dev_meta=http://localhost:8080
fat_meta=http://fill-in-fat-meta-server:8080
uat_meta=http://fill-in-uat-meta-server:8080
pro_meta=http://fill-in-pro-meta-server:8080
```
### (5). 源码编译
```
# 1. 要进入这个目录下,执行编译,要不会报错来着
lixin-macbook:apollo lixin$ cd scripts/
lixin-macbook:scripts lixin$ ./build.sh
```
### (6). 准备运行目录
```
# 1. 创建项目运行目录
lixin-macbook:scripts lixin$ mkdir -p  ~/Developer/apollo/{apollo-configservice,apollo-adminservice,apollo-portal}

# 2. 查看下目录内容
lixin-macbook:scripts lixin$ ll ~/Developer/apollo/
drwxr-xr-x   2 lixin  staff    64  7 15 14:50 apollo-adminservice/
drwxr-xr-x   2 lixin  staff    64  7 15 14:50 apollo-configservice/
drwxr-xr-x   2 lixin  staff    64  7 15 14:50 apollo-portal/

# 3. 拷贝构建物
lixin-macbook:scripts lixin$ cp ../apollo-configservice/target/apollo-configservice-1.9.0-SNAPSHOT-github.zip ~/Developer/apollo/apollo-configservice/
lixin-macbook:scripts lixin$ cp ../apollo-adminservice/target/apollo-adminservice-1.9.0-SNAPSHOT-github.zip ~/Developer/apollo/apollo-adminservice/
lixin-macbook:scripts lixin$ cp ../apollo-portal/target/apollo-portal-1.9.0-SNAPSHOT-github.zip ~/Developer/apollo/apollo-portal/
```
### (7). apollo-configservice
```
# 1. 开始配置apollo-configservice,监听端口为:8080
lixin-macbook:~ lixin$ cd ~/Developer/apollo/apollo-configservice/

# 2. 解压zip
lixin-macbook:apollo-configservice lixin$ unzip apollo-configservice-1.9.0-SNAPSHOT-github.zip

# 3. 查看目录内容
lixin-macbook:apollo-configservice lixin$ ll
-rwxr-xr-x  1 lixin  staff     49944  7 15 14:46 apollo-configservice-1.9.0-SNAPSHOT-sources.jar*
-rwxr-xr-x  1 lixin  staff  71999313  7 15 14:46 apollo-configservice-1.9.0-SNAPSHOT.jar*
-rw-r--r--  1 lixin  staff       223  4  8 14:45 apollo-configservice.conf
drwxr-xr-x  4 lixin  staff       128  7 15 14:54 config/
drwxr-xr-x  4 lixin  staff       128  7 15 14:45 scripts/

# 4. 配置日志输出目录(mkdir: /opt/logs/100003171: Permission denied)
lixin-macbook:apollo-configservice lixin$ vi scripts/startup.sh
  ## 修改该配置项
  LOG_DIR=/tmp/logs/100003171/


# 5. 启动:apollo-configservice
lixin-macbook:apollo-configservice lixin$ ./scripts/startup.sh
2021年 7月15日 星期四 15时01分52秒 CST ==== Starting ====
Started [10235]
Waiting for server startup.....
2021年 7月15日 星期四 15时02分19秒 CST Server started in 25 seconds!

# 6. 测试是否启动成功(能看到:Eureka代表成功了)
lixin-macbook:apollo-configservice lixin$ curl http://localhost:8080
<!doctype html>
  <head>
    <base href="/">
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>Eureka</title>
```
### (8). apollo-adminservice
```
# 1. 开始配置apollo-adminservice(监听端口为:8090)
lixin-macbook:~ lixin$ cd ~/Developer/apollo/apollo-adminservice/

# 2. 解压
lixin-macbook:apollo-adminservice lixin$ unzip apollo-adminservice-1.9.0-SNAPSHOT-github.zip

# 3. 查看目录结构
lixin-macbook:apollo-adminservice lixin$ ll
-rwxr-xr-x  1 lixin  staff     30203  7 15 14:46 apollo-adminservice-1.9.0-SNAPSHOT-sources.jar*
-rwxr-xr-x  1 lixin  staff  67470962  7 15 14:47 apollo-adminservice-1.9.0-SNAPSHOT.jar*
-rw-r--r--  1 lixin  staff       220  7 15 15:07 apollo-adminservice.conf
drwxr-xr-x  4 lixin  staff       128  7 15 15:06 config/
drwxr-xr-x  4 lixin  staff       128  7 15 15:08 scripts/

# 4. 配置日志输出
lixin-macbook:apollo-adminservice lixin$ vi scripts/startup.sh
	## 配置这一行即可
	LOG_DIR=/tmp/logs/100003172/

# 5. 启动:apollo-adminservice
lixin-macbook:apollo-adminservice lixin$ ./scripts/startup.sh
2021年 7月15日 星期四 15时10分03秒 CST ==== Starting ====
Started [10583]
Waiting for server startup......
2021年 7月15日 星期四 15时10分35秒 CST Server started in 30 seconds!

# 6. 测试访问
lixin-macbook:apollo-adminservice lixin$ curl http://localhost:8090
apollo-adminservice
```
### (9). apollo-portal
```
# 1. 开始配置apollo-portal,监听端口为8070
lixin-macbook:~ lixin$ cd ~/Developer/apollo/apollo-portal/

# 2. 解压
lixin-macbook:apollo-portal lixin$ unzip apollo-portal-1.9.0-SNAPSHOT-github.zip

# 3. 查看目录结构
lixin-macbook:apollo-portal lixin$ ll
-rwxr-xr-x  1 lixin  staff   1227525  7 15 14:47 apollo-portal-1.9.0-SNAPSHOT-sources.jar*
-rwxr-xr-x  1 lixin  staff  58042927  7 15 14:47 apollo-portal-1.9.0-SNAPSHOT.jar*
-rw-r--r--  1 lixin  staff       223  4  8 14:45 apollo-portal.conf
drwxr-xr-x  5 lixin  staff       160  7 15 14:47 config/
drwxr-xr-x  4 lixin  staff       128  7 15 15:12 scripts/

# 4. 配置日志目录
lixin-macbook:apollo-portal lixin$ vi scripts/startup.sh
	# 修改这一行即可
	LOG_DIR=/tmp/logs/100003173

# 5. 启动:apollo-portal
lixin-macbook:apollo-portal lixin$ ./scripts/startup.sh
2021年 7月15日 星期四 15时16分27秒 CST ==== Starting ====
Started [10867]
Waiting for server startup....
2021年 7月15日 星期四 15时16分48秒 CST Server started in 20 seconds!

# 6. 测试访问
lixin-macbook:apollo-portal lixin$ curl -vvv http://localhost:8070
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8070 (#0)
> GET / HTTP/1.1
> Host: localhost:8070
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 302
< Set-Cookie: JSESSIONID=D4E6DB6ED11A85A41728C2F1C743BDF5; Path=/; HttpOnly
```
### (10). UI访问(http://localhost:8070)
> 账号密码为(apollo/admin)

!["Apollo Portal"](/assets/apollo/imgs/apollo-portal.png)

### (11). 创建项目
创建项目:  
部门:   样例部门1(TEST1)   
Appid: 7BBB492B-62F8-453F-B50B-0D568308E87A   
应用程序名称: spring-cloud-apollo-example   

!["创建项目"](/assets/apollo/imgs/apollo-create-project.jpg)

### (12). 创建namespace并发布配置
namespace:在前面说过namespace是多个配置文件的集合.

!["创建namespace"](/assets/apollo/imgs/apollo-create-namespace.jpg)
!["namespace添加配置项"](/assets/apollo/imgs/apollo-namespace-add-config.png)
!["namespace home"](/assets/apollo/imgs/apollo-app-home.png)
!["deploy"](/assets/apollo/imgs/apollo-deploy.png)

### (12). Java集成
```
# 1. 添加依赖
<dependency>
	<groupId>com.ctrip.framework.apollo</groupId>
	<artifactId>apollo-client</artifactId>
	<version>1.9.0-SNAPSHOT</version>
</dependency>

# 2.java代码
package help.lixin.apollo;

import com.ctrip.framework.apollo.Config;
import com.ctrip.framework.apollo.ConfigService;
 
public class ApolloClientTest {
	public static void main(String[] args) {
		System.getProperties().put("env", "DEV");
		System.getProperties().put("app.id", "7BBB492B-62F8-453F-B50B-0D568308E87A");
		System.getProperties().put("apollo.meta", "http://127.0.0.1:8080");
		
		Config config = ConfigService.getConfig("TEST1.jdbc");
		String key = "jdbc.url";
		String defaultValue = "default value";
		String value = config.getProperty(key, defaultValue);
		System.out.println("value: " + value);
		
	}
}

# 3. 查看输出结果
value: jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&characterEncoding=UTF-8
```
### (13). Spring Boot集成
@RefreshScope : 会动态的刷新配置项,我在另一篇文章里有深入研究(每次刷新都会剔除Bean),但是,不知道为什么,这次在Apollo里,每次刷新Bean还是同一个,是自己弄错了吗?    

```
# 1. 添加依赖
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>com.ctrip.framework.apollo</groupId>
	<artifactId>apollo-client</artifactId>
	<version>1.9.0-SNAPSHOT</version>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>

# 2. 配置属性bean
package help.lixin.properties;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConfigurationProperties(prefix = "jdbc")
public class JdbcProperties {

	private String url;

	public String getUrl() {
		return url;
	}

	public void setUrl(String url) {
		this.url = url;
	}
}


# 3. 启用注解:@EnableApolloConfig
package help.lixin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import com.ctrip.framework.apollo.spring.annotation.EnableApolloConfig;

@SpringBootApplication
//指定namespace(TEST1是部门,jdbc是namespace)
@EnableApolloConfig({ "TEST1.jdbc", "application" })
public class App {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(App.class, args);
	}
}


# 4. Controller
// *************************************************************
// 1. 只有加了这个注解:@RefreshScope,配置变动时,才会被刷新.
// *************************************************************
@RefreshScope
@RestController
public class HelloController {
	private Logger logger = LoggerFactory.getLogger(HelloController.class);

	@Autowired
	private IHelloService helloService;

    // 2. 通过@Value配置
	@Value("${jdbc.url}")
	private String url;

    // 3. 通过属性来配置的,发现不会发生变化
	@Autowired
	private JdbcProperties jdbcProperties;

	@GetMapping("/test")
	public String test() {
		System.out.println(this + " url:" + url);
		return jdbcProperties.getUrl();
	}
}

# 5. application.properties
server.port=9999
spring.application.name=test-provider

env=DEV
app.id=7BBB492B-62F8-453F-B50B-0D568308E87A
apollo.meta=http://127.0.0.1:8080
apollo.cacheDir=/tmp/data/
#apollo.accesskey.secret=1cf998c4e2ad4704b45a98a509d15719
```
### (14). 总结
在这里,对Apollo进行了一个简单的入门,也留了一个疑惑(上次深入:@RefreshScope发现,Bean是会销毁的,这次用Apollo发现Bean还是同一个.)