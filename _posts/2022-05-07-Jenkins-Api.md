---
layout: post
title: 'Jenkins API' 
date: 2022-05-07
author: 李新
tags:  Jenkins
---

### (1). 概述
几年前看过普元科技的DevOps,对Jenkins做了一些研究,却一直未做一些代码的公开.   
近来工作很忙,趁着有一点时间,把Jenkins的一些API操作记录下. 

### (2). Jenkins API
Jenkins Api实际就是Jenkins提供了一些API的能力,可以通过API操作Jenkins创建项目,触发构建,查看结果.  

### (3). BasicTest
```
package help.lixin.test;

import com.cdancy.jenkins.rest.JenkinsClient;
import com.cdancy.jenkins.rest.domain.common.IntegerResponse;
import com.cdancy.jenkins.rest.domain.common.RequestStatus;
import com.cdancy.jenkins.rest.domain.crumb.Crumb;
import com.cdancy.jenkins.rest.domain.job.JobList;
import com.cdancy.jenkins.rest.domain.queue.QueueItem;
import com.cdancy.jenkins.rest.features.CrumbIssuerApi;
import com.cdancy.jenkins.rest.features.JobsApi;
import com.cdancy.jenkins.rest.features.QueueApi;
import org.apache.commons.io.IOUtils;

import java.io.FileInputStream;
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class BasicTest {
    public static void main(String[] args) throws Exception {
        JenkinsClient client = JenkinsClient.builder() //
                .endPoint("http://172.30.50.7:8080") // Optional. Defaults to
                .credentials("lixin:lixin") // Optional.
                .build();
        crumb(client);
        // 查询
        JobList jobList = jobList(client);
        System.out.println(jobList);
        // 创建任务
        RequestStatus job = createJob(client);
        System.out.println(job);

        IntegerResponse response = build(client);
        System.out.println(response);

        queue(client, response.value());
        System.out.println();
    }


    public static void queue(JenkinsClient client, Integer queueId) {
        QueueApi queueApi = client.api().queueApi();
        QueueItem item = queueApi.queueItem(queueId);
        System.out.println(item);
    }


    public static IntegerResponse build(JenkinsClient client) {
        JobsApi jobsApi = client.api().jobsApi();
        Map<String, List<String>> properties = new HashMap<>();
        properties.put("env", Arrays.asList("dev"));
        properties.put("BRANCH", Arrays.asList("master"));

        IntegerResponse integerResponse = jobsApi.buildWithParameters(null, "tmp-setting-service", properties);
        System.out.println(integerResponse);
        return integerResponse;
    }


    /**
     * 创建任务
     *
     * @param client
     * @return
     * @throws Exception
     */
    public static RequestStatus createJob(JenkinsClient client) throws Exception {
        String path = "/Users/lixin/Workspace/jenkins-demo/src/main/resources/tmp-setting-service.xml";
        String xmlFile = IOUtils.toString(new FileInputStream(path));
        JobsApi jobsApi = client.api().jobsApi();
        RequestStatus requestStatus = jobsApi.create(null, "tmp-setting-service", xmlFile);
        return requestStatus;
    }

    /**
     * 获取所有的任务列表
     *
     * @param client
     * @return
     */
    public static JobList jobList(JenkinsClient client) {
        JobsApi jobsApi = client.api().jobsApi();
        JobList jobList = jobsApi.jobList("");
        return jobList;
    }


    /**
     * crumb登录
     *
     * @param client
     */
    public static void crumb(JenkinsClient client) {
        CrumbIssuerApi crumbIssuerApi = client.api().crumbIssuerApi();
        Crumb crumb = crumbIssuerApi.crumb();
    }
}
```
### (4). 添加pom依赖
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>help.lixin</groupId>
    <artifactId>jenkins-demo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <repositories>
        <repository>
            <id>nexus-aliyun</id>
            <name>Nexus aliyun</name>
            <url>https://maven.aliyun.com/repository/jcenter</url>
        </repository>
    </repositories>


    <dependencies>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-io</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>23.6-jre</version>
        </dependency>
        <dependency>
            <groupId>org.apache.jclouds</groupId>
            <artifactId>jclouds-core</artifactId>
            <version>2.1.2</version>
        </dependency>
        <dependency>
            <groupId>com.google.auto.service</groupId>
            <artifactId>auto-service</artifactId>
            <version>1.0-rc6</version>
        </dependency>
        <dependency>
            <groupId>com.google.auto.value</groupId>
            <artifactId>auto-value-annotations</artifactId>
            <version>1.6.6</version>
        </dependency>
        <dependency>
            <groupId>com.google.auto.value</groupId>
            <artifactId>auto-value</artifactId>
            <version>1.6.6</version>
        </dependency>
        <dependency>
            <groupId>com.cdancy</groupId>
            <artifactId>jenkins-rest</artifactId>
            <version>0.0.29</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpcore</artifactId>
            <version>4.4.8</version>
        </dependency>
    </dependencies>
</project>
```
### (5). tmp-setting-service.xml
```
<?xml version='1.1' encoding='UTF-8'?>
<flow-definition plugin="workflow-job@1167.v8fe861b_09ef9">
    <actions>
        <org.jenkinsci.plugins.pipeline.modeldefinition.actions.DeclarativeJobAction plugin="pipeline-model-definition@1.9.3"/>
        <org.jenkinsci.plugins.pipeline.modeldefinition.actions.DeclarativeJobPropertyTrackerAction plugin="pipeline-model-definition@1.9.3">
            <jobProperties/>
            <triggers/>
            <parameters>
                <string>BRANCH</string>
                <string>env</string>
            </parameters>
            <options/>
        </org.jenkinsci.plugins.pipeline.modeldefinition.actions.DeclarativeJobPropertyTrackerAction>
    </actions>
    <description></description>
    <keepDependencies>false</keepDependencies>
    <properties>
        <org.jenkinsci.plugins.workflow.job.properties.PipelineTriggersJobProperty>
            <triggers/>
        </org.jenkinsci.plugins.workflow.job.properties.PipelineTriggersJobProperty>
        <hudson.model.ParametersDefinitionProperty>
            <parameterDefinitions>
                <hudson.model.ChoiceParameterDefinition>
                    <name>env</name>
                    <description>请选择环境</description>
                    <choices>
                        <string>dev</string>
                        <string>test</string>
						<string>pro</string>
                    </choices>
                </hudson.model.ChoiceParameterDefinition>
                <net.uaznia.lukanus.hudson.plugins.gitparameter.GitParameterDefinition plugin="git-parameter@0.9.15">
                    <name>BRANCH</name>
                    <uuid>a98ba383-70e9-448c-85b3-3d5e6d14bfa9</uuid>
                    <type>PT_BRANCH</type>
                    <tagFilter>*</tagFilter>
                    <branchFilter>origin/(.*)</branchFilter>
                    <defaultValue>master</defaultValue>
                    <listSize>5</listSize>
                    <requiredParameter>false</requiredParameter>
                </net.uaznia.lukanus.hudson.plugins.gitparameter.GitParameterDefinition>
            </parameterDefinitions>
        </hudson.model.ParametersDefinitionProperty>
    </properties>
    <definition class="org.jenkinsci.plugins.workflow.cps.CpsScmFlowDefinition" plugin="workflow-cps@2640.v00e79c8113de">
        <scm class="hudson.plugins.git.GitSCM" plugin="git@4.10.3">
            <configVersion>2</configVersion>
            <userRemoteConfigs>
                <hudson.plugins.git.UserRemoteConfig>
				    <!-- git地址(http) -->
                    <url>xxxxx</url>
                    <credentialsId>lixin-gitlab</credentialsId>
                </hudson.plugins.git.UserRemoteConfig>
            </userRemoteConfigs>
            <branches>
                <hudson.plugins.git.BranchSpec>
                    <name>*/master</name>
                </hudson.plugins.git.BranchSpec>
            </branches>
            <doGenerateSubmoduleConfigurations>false</doGenerateSubmoduleConfigurations>
            <submoduleCfg class="empty-list"/>
            <extensions/>
        </scm>
        <scriptPath>Jenkinsfile</scriptPath>
        <lightweight>true</lightweight>
    </definition>
    <triggers/>
    <disabled>false</disabled>
</flow-definition>
```
### (6). 总结
如果自己想实现自己的一整套DevOps(Jenkins+Docker+K8S),则可以通过Jenkins API + K8S API实现DevOps.  