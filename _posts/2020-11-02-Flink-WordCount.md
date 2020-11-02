---
layout: post
title: 'Flink WordCount'
date: 2020-12-02
author: 李新
tags: Flink
---

### (1).JDK版本
```
lixin-macbook:~ lixin$ java -version
    java version "1.8.0_251"
    Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
    Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
```
### (2).Scala版本
```
lixin-macbook:~ lixin$ scala -version
    Scala code runner version 2.11.12 -- Copyright 2002-2017, LAMP/EPFL
```
### (3).pom.xml配置
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>help.lixin.wc</groupId>
    <artifactId>flink-wordcount</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <flink.version>1.11.2</flink.version>
        <scala.version>2.11.12</scala.version>
        <scala.binary.version>2.11</scala.binary.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-scala_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.7</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
            <scope>runtime</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- We use the maven-shade plugin to create a fat jar that contains all necessary dependencies. -->
            <!-- Change the value of <mainClass>...</mainClass> if your program entry point changes. -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.0.0</version>
                <executions>
                    <!-- Run shade goal on package phase -->
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <artifactSet>
                                <excludes>
                                    <exclude>org.apache.flink:force-shading</exclude>
                                    <exclude>com.google.code.findbugs:jsr305</exclude>
                                    <exclude>org.slf4j:*</exclude>
                                    <exclude>log4j:*</exclude>
                                </excludes>
                            </artifactSet>
                            <filters>
                                <filter>
                                    <!-- Do not copy the signatures in the META-INF folder.
                                    Otherwise, this might cause SecurityExceptions when using the JAR. -->
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>groupId.StreamingJob</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <!-- Java Compiler -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>

            <!-- Scala Compiler -->
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>3.2.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- Eclipse Scala Integration -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-eclipse-plugin</artifactId>
                <version>2.8</version>
                <configuration>
                    <downloadSources>true</downloadSources>
                    <projectnatures>
                        <projectnature>org.scala-ide.sdt.core.scalanature</projectnature>
                        <projectnature>org.eclipse.jdt.core.javanature</projectnature>
                    </projectnatures>
                    <buildcommands>
                        <buildcommand>org.scala-ide.sdt.core.scalabuilder</buildcommand>
                    </buildcommands>
                    <classpathContainers>
                        <classpathContainer>org.scala-ide.sdt.launching.SCALA_CONTAINER</classpathContainer>
                        <classpathContainer>org.eclipse.jdt.launching.JRE_CONTAINER</classpathContainer>
                    </classpathContainers>
                    <excludes>
                        <exclude>org.scala-lang:scala-library</exclude>
                        <exclude>org.scala-lang:scala-compiler</exclude>
                    </excludes>
                    <sourceIncludes>
                        <sourceInclude>**/*.scala</sourceInclude>
                        <sourceInclude>**/*.java</sourceInclude>
                    </sourceIncludes>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>build-helper-maven-plugin</artifactId>
                <version>1.7</version>
                <executions>
                    <!-- Add src/main/scala to eclipse build path -->
                    <execution>
                        <id>add-source</id>
                        <phase>generate-sources</phase>
                        <goals>
                            <goal>add-source</goal>
                        </goals>
                        <configuration>
                            <sources>
                                <source>src/main/scala</source>
                            </sources>
                        </configuration>
                    </execution>
                    <!-- Add src/test/scala to eclipse build path -->
                    <execution>
                        <id>add-test-source</id>
                        <phase>generate-test-sources</phase>
                        <goals>
                            <goal>add-test-source</goal>
                        </goals>
                        <configuration>
                            <sources>
                                <source>src/test/scala</source>
                            </sources>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```
### (4).WorkCount.scala
```

import org.apache.flink.api.scala._

object WordCount {
  def main(args: Array[String]): Unit = {
    // 创建一个批处理执行环境
    val environment = ExecutionEnvironment.getExecutionEnvironment
    val text = environment.fromElements(
      "hello world",
      "hello scala",
      "hello eclipse"
    )
    val counts = text.flatMap { _.toLowerCase.split(" ") }
      .map { (_, 1) }
      .groupBy(0)
      .sum(1)
    counts.print()
  }
}

```
### (5).java.lang.ClassNotFoundException: org.apache.flink.api.scala.typeutils.CaseClassTypeInfo
> 因为加入的依赖生命周期为:provided.

!["org.apache.flink.api.scala.typeutils.CaseClassTypeInfo解决方案"](/assets/flink/imgs/flink-hello-world.jpg)

### (6).运行查看结果
```
(eclipse,1)
(scala,1)
(world,1)
(hello,3)
```