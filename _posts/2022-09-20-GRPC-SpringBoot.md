---
layout: post
title: 'GRPC与SpringBoot整合' 
date: 2022-09-19
author: 李新
tags:  GRPC
---


### (1). 概述
在这一小篇学习GRPC与SpringBoot的整合.

### (2). 项目结构
```
lixin-macbook:grpc-springboot-demo-parent lixin$ tree -L 1
.
├── client                       # 客户端
├── pom.xml
├── proto                        # PB定义的协议
└── server                       # 服务端
```
### (3). grpc-springboot-demo-parent/pom.xml配置
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>help.lixin.grpc.demo</groupId>
    <artifactId>grpc-springboot-demo-parent</artifactId>
    <packaging>pom</packaging>
    <version>1.1.0</version>
    <name>grpc-springboot-demo-parent ${project.version}</name>

    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <spring-boot.version>2.4.1</spring-boot.version>
    </properties>

    <repositories>
        <repository>
            <id>alimaven</id>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
        </repository>
    </repositories>

    <modules>
        <module>server</module>
        <module>client</module>
        <module>proto</module>
    </modules>
</project>
```
### (4). proto项目结构
```
lixin-macbook:grpc-springboot-demo-parent lixin$ tree proto
proto
├── pom.xml
├── src
│   └── main
│       ├── java
│       │   └── help
│       │       └── lixin
│       │           └── proto
│       │               ├── HelloRequest.java
│       │               ├── HelloRequestOrBuilder.java
│       │               ├── HelloResponse.java
│       │               ├── HelloResponseOrBuilder.java
│       │               ├── HelloServiceGrpc.java
│       │               └── HelloServiceProto.java
│       ├── proto
│       │   └── HelloService.proto
│       └── resources
└── target
```
### (5). proto/pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>help.lixin.grpc.demo</groupId>
        <artifactId>grpc-springboot-demo-parent</artifactId>
        <version>1.1.0</version>
    </parent>
    <artifactId>grpc-springboot-demo-proto</artifactId>
    <packaging>jar</packaging>
    <version>1.1.0</version>

    <!-- 源码编译,需要指定以下依赖,并配置scope -->
    <dependencies>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>1.42.2</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>1.42.2</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>1.42.2</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.tomcat</groupId>
            <artifactId>annotations-api</artifactId>
            <version>6.0.53</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.7.0</version>
            </extension>
        </extensions>

        <resources>
            <resource>
                <directory>src/main/proto</directory>
                <includes>
                    <include>**/*.proto</include>
                </includes>
            </resource>
        </resources>

        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:3.19.1:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.42.1:exe:${os.detected.classifier}</pluginArtifact>
                    <!-- proto文件放置的目录 -->
                    <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
                    <!-- 生成文件的目录 -->
                    <outputDirectory>${project.basedir}/src/main/java</outputDirectory>
                    <!-- 生成文件前是否把目标目录清空-->
                    <clearOutputDirectory>false</clearOutputDirectory>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
### (6). proto/src/main/proto/HelloService.proto
```
syntax = "proto3";

option java_multiple_files = true;
option java_package = "help.lixin.proto";
option java_outer_classname = "HelloServiceProto";
option objc_class_prefix = "HLW";

package help.lixin.proto;

service HelloService {
  rpc sayHello (HelloRequest) returns (HelloResponse) {}
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```
### (7). server项目结构
```
lixin-macbook:grpc-springboot-demo-parent lixin$ tree server
server
├── pom.xml
├── src
│   └── main
│       ├── java
│       │   └── help
│       │       └── lixin
│       │           └── demo
│       │               ├── Application.java
│       │               └── service
│       │                   └── HelloService.java
│       └── resources
│           └── application.yml
└── target
```
### (8). server/pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>help.lixin.grpc.demo</groupId>
        <artifactId>grpc-springboot-demo-parent</artifactId>
        <version>1.1.0</version>
    </parent>
    <artifactId>grpc-springboot-demo-server</artifactId>
    <packaging>jar</packaging>
    <version>1.1.0</version>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
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

        <!-- grpc依赖添加 -->
        <dependency>
            <groupId>net.devh</groupId>
            <artifactId>grpc-server-spring-boot-starter</artifactId>
            <version>2.13.1.RELEASE</version>
        </dependency>

        <!-- 添加协议依赖 -->
        <dependency>
            <groupId>help.lixin.grpc.demo</groupId>
            <artifactId>grpc-springboot-demo-proto</artifactId>
            <version>1.1.0</version>
        </dependency>
    </dependencies>
</project>
```
### (9). server/src/main/java/help/lixin/demo/service/HelloService.java
```
package help.lixin.demo.service;

import help.lixin.proto.HelloRequest;
import help.lixin.proto.HelloResponse;
import help.lixin.proto.HelloServiceGrpc;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;

// ************************************************************
// 自定义业务实现,并指定注解.
// ************************************************************
@GrpcService
public class HelloService extends HelloServiceGrpc.HelloServiceImplBase {
    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {
        try {
            String name = request.getName();
            responseObserver.onNext(HelloResponse.newBuilder().setMessage(name + " 你好哈!").build());
        } finally {
            responseObserver.onCompleted();
        }
    }
}
```
### (10). server/src/main/java/help/lixin/demo/Application.java
```
package help.lixin.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }
}
``` 
### (11). server/src/main/resources/application.yml
```
spring:
  application:
    name: demo-server
server:
  port: 8080

grpc:
  server:
    port: 8888
```
### (12). client项目结构
```
lixin-macbook:grpc-springboot-demo-parent lixin$ tree client
client
├── pom.xml
├── src
│   └── main
│       ├── java
│       │   └── help
│       │       └── lixin
│       │           └── demo
│       │               ├── GrpcClientApplication.java
│       │               └── controller
│       │                   └── HelloController.java
│       └── resources
│           └── application.yml
└── target
```
### (13). client/pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>help.lixin.grpc.demo</groupId>
        <artifactId>grpc-springboot-demo-parent</artifactId>
        <version>1.1.0</version>
    </parent>
    <artifactId>grpc-springboot-demo-client</artifactId>
    <packaging>jar</packaging>
    <version>1.1.0</version>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
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

        <!-- grpc依赖添加 -->
        <dependency>
            <groupId>net.devh</groupId>
            <artifactId>grpc-client-spring-boot-starter</artifactId>
            <version>2.13.1.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>help.lixin.grpc.demo</groupId>
            <artifactId>grpc-springboot-demo-proto</artifactId>
            <version>1.1.0</version>
        </dependency>
    </dependencies>
</project>
```
### (14). client/src/main/java/help/lixin/demo/controller/HelloController.java
```
package help.lixin.demo.controller;

import help.lixin.proto.HelloRequest;
import help.lixin.proto.HelloResponse;
import help.lixin.proto.HelloServiceGrpc;
import net.devh.boot.grpc.client.inject.GrpcClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    // **************************************************************
	// GRPC客户端.
	// **************************************************************
    @GrpcClient("demo-server")
    private HelloServiceGrpc.HelloServiceBlockingStub helloService;

    @GetMapping("/hello")
    public String sayHello(String name) {
        HelloResponse helloResponse = helloService.sayHello(HelloRequest.newBuilder().setName(name).build());
        return helloResponse.getMessage();
    }
}
```
### (15). client/src/main/java/help/lixin/demo/GrpcClientApplication.java
```
package help.lixin.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GrpcClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(GrpcClientApplication.class);
    }
}
```
### (16). client/src/main/resources/application.yml
```
spring:
  application:
    name: demo-client
server:
  port: 7070

grpc:
  client:
    demo-server: # 定义微服务的名称
      # 使用静态ip地址的方式访问
      address: 'static://127.0.0.1:8888'
      #
      negotiationType: plaintext
```
### (17). 测试访问
```
lixin-macbook:~ lixin$ curl http://localhost:7070/hello?name=lixin
lixin 你好哈!
```
### (18). 总结
GRPC自身并没有提供与SpringBoot的集成,上面的依赖是来自于社区,从注解配置信息上能看得出来,这个社区对GRPC进行了扩展,允许负载均衡和服务发现来着的.  