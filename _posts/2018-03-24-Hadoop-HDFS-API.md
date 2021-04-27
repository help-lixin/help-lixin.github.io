---
layout: post
title: 'Hadoop HDFS API(四)'
date: 2018-03-24
author: 李新
tags: Hadoop
---

### (1). 命令参考地址

> https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html


### (2). HDFS API(pom.xml)
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>help.lixin.hadoop.example</groupId>
    <artifactId>hadoop-example</artifactId>
    <version>1.0-SNAPSHOT</version>

    <repositories>
        <repository>
            <id>central</id>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <layout>default</layout>
            <!-- 是否开启发布版构件下载 -->
            <releases>
                <enabled>true</enabled>
            </releases>
            <!-- 是否开启快照版构件下载 -->
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.7.5</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.7.5</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.7.5</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-mapreduce-client-core</artifactId>
            <version>2.7.5</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <!-- 指定maven编译的jdk版本,如果不指定,maven3默认用jdk 1.5 maven2默认用jdk1.3 -->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <!-- 一般而言，target与source是保持一致的，但是，有时候为了让程序能在其他版本的jdk中运行(对于低版本目标jdk，源代码中不能使用低版本jdk中不支持的语法)，会存在target不同于source的情况 -->
                    <source>1.8</source> <!-- 源代码使用的JDK版本 -->
                    <target>1.8</target> <!-- 需要生成的目标class文件的编译版本 -->
                    <encoding>UTF-8</encoding><!-- 字符集编码 -->
                    <verbose>true</verbose>
                    <showWarnings>true</showWarnings>
                    <fork>true</fork><!-- 要使compilerVersion标签生效，还需要将fork设为true，用于明确表示编译版本配置的可用 -->
                    <executable><!-- path-to-javac --></executable><!-- 使用指定的javac命令，例如：<executable>${JAVA_1_4_HOME}/bin/javac</executable> -->
                    <compilerVersion>1.3</compilerVersion><!-- 指定插件将使用的编译器的版本 -->
                    <meminitial>128m</meminitial><!-- 编译器使用的初始内存 -->
                    <maxmem>512m</maxmem><!-- 编译器使用的最大内存 -->
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <transformers>
                        <!--
                        <transformer
                                implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                            <mainClass>org.demo.App</mainClass>
                        </transformer>
                        -->
                    </transformers>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

        </plugins>
    </build>
</project>
```

### (3). HDFS API
```
package help.lixin.hdfs.example;


import org.apache.commons.io.IOUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.*;
import org.junit.BeforeClass;
import org.junit.Test;

import java.io.File;
import java.io.FileOutputStream;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.URI;
import java.net.URL;

public class HdfsTest {

    private static FileSystem fileSystem = null;

    @BeforeClass
    public static void init() throws Exception {
        fileSystem = FileSystem.get(new URI("hdfs://localhost:9000"), new Configuration());
    }

    public static void close() throws Exception {
        if (null != fileSystem) {
            fileSystem.close();
        }
    }

    @Test
    public void testGetFile() throws Exception {
        // 注册URL工厂
        URL.setURLStreamHandlerFactory(new FsUrlStreamHandlerFactory());

        // 获得输入流
        InputStream inputStream = new URL("hdfs://loclahost:9000/query-log/SogouQ2.txt").openStream();

        OutputStream outputStream = new FileOutputStream(new File("/Users/lixin/IDEAWorkspace/hadoop-example/target/SogouQ2.txt"));

        IOUtils.copy(inputStream, outputStream);
        IOUtils.closeQuietly(inputStream);
        IOUtils.closeQuietly(outputStream);
    }

    @Test
    public void testEachFiles() throws Exception {
        RemoteIterator<LocatedFileStatus> locatedFileStatusRemoteIterator = fileSystem.listFiles(new Path("/"), true);
        while (locatedFileStatusRemoteIterator.hasNext()) {
            LocatedFileStatus next = locatedFileStatusRemoteIterator.next();
            System.out.println(next.getPath().toString());
        }
    }

    @Test
    public void testMkDir() throws Exception {
        boolean mkdirs = fileSystem.mkdirs(new Path("/logs"));
        if (mkdirs) {
            System.out.println("create dir success");
        } else {
            System.out.println("create dir fail");
        }
    }

    @Test
    public void testUploadFile() throws Exception {
        Path logs = new Path("/logs");
        if (!fileSystem.exists(logs)) {
            fileSystem.mkdirs(logs);
        }
        Path src = new Path("/Users/lixin/IDEAWorkspace/hadoop-example/target/SogouQ2.txt");
        fileSystem.copyFromLocalFile(src, logs);
        System.out.println("copy local file to hdfs success");
    }

    @Test
    public void testGetFile2() throws Exception {
        Path src = new Path("/logs/SogouQ2.txt");
        Path dest = new Path("/Users/lixin/IDEAWorkspace/hadoop-example/target/SogouQ3.txt");
        fileSystem.copyToLocalFile(src, dest);

        // 打开文件并转换成流
        FSDataInputStream dataInputStream = fileSystem.open(src);
        FileOutputStream outputStream = new FileOutputStream(new File("/Users/lixin/IDEAWorkspace/hadoop-example/target/SogouQ4.txt"));

        IOUtils.copy(dataInputStream, outputStream);
        IOUtils.closeQuietly(dataInputStream);
        IOUtils.closeQuietly(outputStream);

        System.out.println("copy remote file to local success");
    }

    // 测试文件拷贝之前,先修改权限:  
    // hdfs dfs -chmod 600 /logs/SogouQ2.txt
    @Test
    public void testPermission() throws  Exception {
        FileSystem fileSystem2 = FileSystem.get(
                // HDFS URL
                new URI("hdfs://localhost:9000"),
                // Configuration
                new Configuration(),
                // ********************执行任务时,可以模拟哪个用户去运行*****************
                "test");

        Path src = new Path("/logs/SogouQ2.txt");
        // 打开文件并转换成流
        FSDataInputStream dataInputStream = fileSystem2.open(src);
        // 抛出如下异常
        // org.apache.hadoop.security.AccessControlException: Permission denied: user=test, access=READ, inode="/logs/SogouQ2.txt":lixin:supergroup:-rw-------
        FileOutputStream outputStream = new FileOutputStream(new File("/Users/lixin/IDEAWorkspace/hadoop-example/target/SogouQ4.txt"));

        IOUtils.copy(dataInputStream, outputStream);
        IOUtils.closeQuietly(dataInputStream);
        IOUtils.closeQuietly(outputStream);
        System.out.println("test testPermission copy remote file to local success");
    }
}
```
