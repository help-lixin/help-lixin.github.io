---
layout: post
title: 'MyBatis Generator 扩展'
date: 2021-02-27
author: 李新
tags: MyBatis 解决方案
---

### (1). 概述
> 随着企业标准化的需求,开发期望,浏览数据库,选择表,可以一键生成整个项目工程.   
> MyBatis Generator可以一键生成:XXMapper.xml/XXMapper.java/XXXDomain.java,
> 但,还不太符合开发的要求,开发期望的是生成整个工程项目,包括:pom.xml/Controller/Service/Mapper...  
> 那么,能否对:MyBatis Generator进行扩展,来实现这个功能呢?  
> 答案:是可以的!  

### (2). 我先说下MyBatis Generator原理
> 其实,这样的功能,也可以完全自己做,用模板引擎来套就好了,没看源码之前,我也以为是用模板引擎来做的,看过MyBatis的源码后,才发现,人家不是那么做的.     
> MyBatis的开发人员,在这方面设计还是不错的,人家早就留出了接口给我们做扩展.      
> 以下几个接口和类是扩展的重点类:  
> 1. org.mybatis.generator.api.Plugin     
> 2. org.mybatis.generator.api.GeneratedJavaFile    
> 3. org.mybatis.generator.api.dom.java.CompilationUnit     
> 4. org.mybatis.generator.api.JavaFormatter   

### (3). 实现步骤
> 1. 实现Plugin接口(一般是继续它的适配类:PluginAdapter)典型适配模式的责任链模式.  
> 2. 重写contextGenerateAdditionalJavaFiles和contextGenerateAdditionalXmlFiles方法.  
> 3. 顾名思意,这两个方法就是生成java文件和xml文件的.   
> 4. 这两个方法要求返回的是:GeneratedJavaFile/GeneratedXmlFile,我这里只介绍:GeneratedJavaFile   
> 5. GeneratedJavaFile的构造器,需要一个重要的对象:CompilationUnit,这个对象就是:承载着我们要产生Java文件的对象.  
> 6. 在xml中配置启用插件
> 7. 我这里只给出大概代码和思路,不会粘出所有的代码的.  

### (4). 扩展Plugin
```
public class ServicePlugin extends PluginAdapter {
	public List<GeneratedJavaFile> contextGenerateAdditionalJavaFiles(IntrospectedTable introspectedTable) {
		List<GeneratedJavaFile> generatedJavaFiles = new ArrayList<GeneratedJavaFile>(2);
		// 生成的项目路径
		String targetProject = getProperties().getProperty("targetProject", null);
		// package
		String targetPackage = getProperties().getProperty("targetPackage", null);
		if (null == targetProject || null == targetPackage) {
			return generatedJavaFiles;
		}
		// 生成接口
		generatedJavaFiles.add(buildInterface(introspectedTable, targetProject, targetPackage));
		// 生成接口的实现类
		generatedJavaFiles.add(buildInterfaceImpl(introspectedTable, targetProject, targetPackage));
		return generatedJavaFiles;
	} // end contextGenerateAdditionalJavaFiles
	
	
	private GeneratedJavaFile buildInterface(IntrospectedTable introspectedTable, String targetProject,
				String targetPackage) {
		// help.lixin.entity.Order
		String entityFullName = getEntityFullName(introspectedTable);
		// Order
		String entityShortName = introspectedTable.getTableConfiguration().getDomainObjectName();
		// order
		String lowEntityShortName = lowerFirst(entityShortName);

		// 把Order转换成:help.lixin.service.IOrderService
		String serviceInterface = getServiceFullName(I, targetPackage, entityShortName);
		
		// ******************************************************************
		// Interface是:CompilationUnit的实现类
		//  在这里就开始构建:CompilationUnit
		// ******************************************************************
		Interface compilationUnit = new Interface(new FullyQualifiedJavaType(serviceInterface));
		// 设置接口的可见性
		compilationUnit.setVisibility(JavaVisibility.PUBLIC);
		// 为接口设置:import xxxx;
		importDefault(compilationUnit);
		// 为接口设置:import xxxx;
		compilationUnit.addImportedType(new FullyQualifiedJavaType(entityFullName));

		// 给接口添加方法
		Method insert = buildInsert(entityFullName, lowEntityShortName);
		insert.setAbstract(true);
		compilationUnit.addMethod(insert);

		// 给接口添加方法
		Method selectAll = buildSelectAll(entityFullName);
		selectAll.setAbstract(true);
		compilationUnit.addMethod(selectAll);

		// 给接口添加方法
		Method updateByPrimaryKey = buildUpdateByPrimaryKey(entityFullName, lowEntityShortName);
		updateByPrimaryKey.setAbstract(true);
		compilationUnit.addMethod(updateByPrimaryKey);

		// 判断表是否有主键
		IntrospectedColumn pkColumn = pkColumn(introspectedTable);
		if (null != pkColumn) {
			// 给接口添加方法
			Method deleteByPrimaryKey = buildDeleteByPrimaryKey(pkColumn);
			deleteByPrimaryKey.setAbstract(true);
			compilationUnit.addMethod(deleteByPrimaryKey);
		}
		
		// **************************************************************
		// JavaFormatter会把CompilationUnit里所有数据,转换成字符串的.
		// **************************************************************
		JavaFormatter javaFormatter = new DefaultJavaFormatter();
		GeneratedJavaFile generatedJavaFile = new GeneratedJavaFile(compilationUnit, targetProject, javaFormatter);
		return generatedJavaFile;
	}// end buildInterface
}
```
### (5). 配置xml(generatorConfig.xml)
```
<!DOCTYPE generatorConfiguration PUBLIC
 "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
 "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
  <classPathEntry location="/Users/lixin/Workspace/code-gen/src/main/resources/lib/mysql-connector-java-5.1.49.jar" />
  
  <context id="simple" targetRuntime="MyBatis3Simple">
  	<!-- 添加自定义插件 -->
  	<plugin type="help.lixin.framework.codegen.plugins.ServicePlugin">
  		<property name="targetProject" value="/Users/lixin/Workspace/code-gen/target/src/main/java"/>
  		<property name="targetPackage" value="example.service"/>
  	</plugin>
    <jdbcConnection driverClass="com.mysql.jdbc.Driver"
        connectionURL="jdbc:mysql://localhost:3306/order_db" userId="root" password="123456"/>
    <javaModelGenerator targetPackage="example.model" targetProject="/Users/lixin/Workspace/code-gen/target/src/main/java">
    	<property name="enableSubPackages" value="true"/>
    	<property name="trimStrings" value="true"/>
    </javaModelGenerator>
    <sqlMapGenerator targetPackage="mappers" targetProject="/Users/lixin/Workspace/code-gen/target/src/main/resources"/>
    <javaClientGenerator type="XMLMAPPER" targetPackage="example.mapper" targetProject="/Users/lixin/Workspace/code-gen/target/src/main/java"/>
    <table tableName="t_order_1" schema="t_order" domainObjectName="Order" mapperName="OrderMapper" modelType="hierarchical"/>
  </context>
</generatorConfiguration>
```
### (6). 运行测试
```
package help.lixin.framework.codegen;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

import org.mybatis.generator.api.MyBatisGenerator;
import org.mybatis.generator.config.Configuration;
import org.mybatis.generator.config.xml.ConfigurationParser;
import org.mybatis.generator.internal.DefaultShellCallback;

public class App {
	public static void main(String[] args) throws Exception {
		List<String> warnings = new ArrayList<String>();
		boolean overwrite = true;

		File configFile = new File("/Users/lixin/Workspace/code-gen/src/main/resources/generatorConfig.xml");

		ConfigurationParser cp = new ConfigurationParser(warnings);
		Configuration config = cp.parseConfiguration(configFile);
		DefaultShellCallback callback = new DefaultShellCallback(overwrite);
		MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config, callback, warnings);
		myBatisGenerator.generate(null);
	}
}
```
### (7). 总结
> 通过对Plugin进行扩展,就可以做到不需要改动MyBatis.即可轻巧的实现代码生成.   