---
layout: post
title: 'GRPC HelloWorld入门' 
date: 2022-09-19
author: 李新
tags:  GRPC
---


### (1). 概述
在这一小篇我们用GRPC构建一个简单的应用,以促进我们对GRPC有一个印象.

### (2). 项目结构如下
```
lixin-macbook:grpc-demo-parent lixin$ tree -L 1
.
├── grpc-demo-client
├── grpc-demo-proto
├── grpc-demo-server
└── pom.xml
```
### (3). grpc-demo-proto项目结构
```
lixin-macbook:grpc-demo-parent lixin$ tree grpc-demo-proto/
grpc-demo-proto/
├── pom.xml
└── src
    ├── main
    ├── java
    └── proto
    └── helloworld.proto
```
### (4). grpc-demo-proto/pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>grpc-demo-parent</artifactId>
        <groupId>help.lixin.grpc</groupId>
        <version>1.1.0</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>grpc-demo-proto</artifactId>
    <name>grpc-demo-proto</name>

    <dependencies>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>1.43.1</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>1.43.1</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>1.43.1</version>
        </dependency>
        <dependency> <!-- necessary for Java 9+ -->
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
                <version>1.6.2</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:3.19.1:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.43.1:exe:${os.detected.classifier}</pluginArtifact>
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
### (5). grpc-demo-proto/src/main/proto/helloworld.proto
```
syntax = "proto3";

option java_multiple_files = true;
option java_package = "help.lixin.helloworld";
option java_outer_classname = "HelloWorldProto";
option objc_class_prefix = "HLW";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```
### (6). grpc-demo-server项目结构如下
```
lixin-macbook:grpc-demo-parent lixin$ tree grpc-demo-server/
grpc-demo-server/
├── pom.xml
└── src
    ├── main
    ├── java
    │   └── help
    │       └── lixin
    │           └── helloworld
    │               ├── GreeterGrpc.java
    │               ├── HelloReply.java
    │               ├── HelloReplyOrBuilder.java
    │               ├── HelloRequest.java
    │               ├── HelloRequestOrBuilder.java
    │               ├── HelloWorldProto.java
    │               └── HelloWorldService.java
    └── resources
```
### (7). grpc-demo-server/pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>grpc-demo-parent</artifactId>
        <groupId>help.lixin.grpc</groupId>
        <version>1.1.0</version>
    </parent>
    <packaging>jar</packaging>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>grpc-demo-server</artifactId>

    <dependencies>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>1.43.1</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>1.43.1</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>1.43.1</version>
        </dependency>
        <dependency> <!-- necessary for Java 9+ -->
            <groupId>org.apache.tomcat</groupId>
            <artifactId>annotations-api</artifactId>
            <version>6.0.53</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```
### (8). 编写服务端业务代码实现(HelloWorldService)
```
package help.lixin.helloworld;

import io.grpc.stub.StreamObserver;

public class HelloWorldService extends GreeterGrpc.GreeterImplBase {
    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloReply> responseObserver) {
        try {
            System.out.println(request.getName());
            // ... ...
            responseObserver.onNext(HelloReply.newBuilder().setMessage("Process Hello World,Result HelloWorld! ").build());
        } finally {
            responseObserver.onCompleted();
        }
    }
}
```
### (9). 编写服务端启动代码
```
package help.lixin;

import help.lixin.helloworld.HelloWorldService;
import io.grpc.Server;
import io.grpc.ServerBuilder;

public class HelloServer {
    public static void main(String[] args) throws Exception {
        Server server = ServerBuilder
                .forPort(8888)
                .addService(new HelloWorldService())
                .build();
        server.start();
        System.out.println("sever start....");
        server.awaitTermination();
    }
}
```
### (10). grpc-demo-client项目结构
```
lixin-macbook:grpc-demo-parent lixin$ tree grpc-demo-client/
grpc-demo-client/
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── help
    │   │       └── lixin
    │   │           └── helloworld
    │   │               ├── GreeterGrpc.java
    │   │               ├── HelloReply.java
    │   │               ├── HelloReplyOrBuilder.java
    │   │               ├── HelloRequest.java
    │   │               ├── HelloRequestOrBuilder.java
    │   │               └── HelloWorldProto.java
    │   └── resources
    └── test
        └── java
            └── help
                └── lixin
                    └── helloworld
                        └── HelloClient.java
```
### (11). grpc-demo-client/pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>grpc-demo-parent</artifactId>
        <groupId>help.lixin.grpc</groupId>
        <version>1.1.0</version>
    </parent>
    <packaging>jar</packaging>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>grpc-demo-client</artifactId>

    <dependencies>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>1.43.1</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>1.43.1</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>1.43.1</version>
        </dependency>
        <dependency> <!-- necessary for Java 9+ -->
            <groupId>org.apache.tomcat</groupId>
            <artifactId>annotations-api</artifactId>
            <version>6.0.53</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```
### (12). 编写业务端代码(HelloClient)
```
package help.lixin.helloworld;

import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;

public class HelloClient {
    public static void main(String[] args) {
        ManagedChannel channel = ManagedChannelBuilder.forAddress("127.0.0.1", 8888).usePlaintext().build();
        try {
            GreeterGrpc.GreeterBlockingStub helloWorldServiceStube = GreeterGrpc.newBlockingStub(channel);
            HelloReply helloResponse = helloWorldServiceStube.sayHello(HelloRequest.newBuilder().setName("Client Hello World!!!").build());
            System.out.println(helloResponse);
        } finally {
            channel.shutdownNow();
        }
    }
}
```
### (13). proto文件解析
> 以下代码是通过protobuf-maven-plugin插件读取:proto文件,反向解析出来的.   

+ HelloWorldProto
+ HelloRequestOrBuilder
+ HelloRequest
+ HelloReplyOrBuilder
+ HelloReply
+ GreeterGrpc

### (14). 总结
>  GRPC编码过程如下:  
+ 编写proto文件. 
+ 通过插件:protobuf-maven-plugin解析proto文件,并生成java代码. 
+ 服务端编写处理业务逻辑代码. 
+ 服务端编写main启动. 
+ 测试客户端调用服务端提供的服务. 
