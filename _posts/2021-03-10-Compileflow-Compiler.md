---
layout: post
title: 'Compileflow Compiler生成Java源码(二)'
date: 2021-02-27
author: 李新
tags: Compileflow ClassLoader
---

### (0). 概述
> 前面对Compileflow有了一个入门,也说到它的核心原理,以及优缺点,这一小节,对优缺点进行难证(验证:流程图的改变,会触发ClassLoader进行热加载).  

### (1). AbstractProcessRuntime.compile
```
// 0. 定义生成java源代码类
private static final Compiler COMPILER = new CompilerImpl();

// 1. 编译源码
public void compile() {
	compiledClassCache.computeIfAbsent(code, c -> compileJavaCode(getJavaCode(code)));
}

// 2. 获取源码信息
// <process id="ktv" name="ktv" isExecutable="true">
// code = ktv
private String getJavaCode(String code) { 
	return javaCodeCache.computeIfAbsent(code, c -> generateJavaCode());
}

// 3. 生成java代码
public String generateJavaCode() {
	classTarget.addSuperInterface(ClassWrapper.of(ProcessInstance.class));
	// ... ...
	// 这部份内容忽略不计
	generateFlowMethod("execute", this::generateExecuteMethodBody);
	return classTarget.generateCode();
}

// 4. 编译java代码,并通过ClassLoader加载
private Class<?> compileJavaCode(String source) {
	return COMPILER.compileJavaCode(classTarget.getFullName(), source);
}
```
### (2). CompilerImpl
```
package com.alibaba.compileflow.engine.process.preruntime.compiler.impl;

import com.alibaba.compileflow.engine.common.CompileFlowException;
import com.alibaba.compileflow.engine.common.utils.FileUtils;
import com.alibaba.compileflow.engine.process.preruntime.compiler.Compiler;
import com.alibaba.compileflow.engine.process.preruntime.compiler.*;
import com.alibaba.compileflow.engine.process.preruntime.compiler.impl.support.EcJavaCompiler;

import java.io.File;
import java.io.FileOutputStream;
import java.io.PrintWriter;


public class CompilerImpl implements Compiler {
	
	// 0. 定义编译类
    private static final JavaCompiler JAVA_COMPILER = new EcJavaCompiler();

    @Override
    public Class<?> compileJavaCode(
					//  compileflow.KtvFlow
					String fullClassName, 
					// package compileflow; ... ...
					String sourceCode) {

		
        try {
			// 1. 创建产生Java文件的目录(用户目录/.flowclasses)
			// public static final String FLOW_COMPILE_CLASS_DIR = System.getProperty("user.dir") + File.separator
        + ".flowclasses" + File.separator;
				
            String dirPath = CompileConstants.FLOW_COMPILE_CLASS_DIR;
            File dirFile = new File(dirPath);
            if (!dirFile.exists()) {
                if (!dirFile.mkdir()) {
                    throw new RuntimeException(
                        "Output directory for process class can't be created, directory is " + dirPath);
                }
            }

			// 2. 创建:KtvFlow.java文件
            File javaSourceFile = writeJavaFile(dirFile, fullClassName, sourceCode);
			
			// 3. Wrapper成JavaSource对象
            JavaSource javaSource = JavaSource.of(javaSourceFile, sourceCode, fullClassName);
			
			// ********************************************************
			// 4. 委托给:EcJavaCompiler进行处理(实际底层又委托给了:org.eclipse.jdt.internal.compiler.Compiler)
			//    先不关注这个.只需要知道这一步会产生:KtvFlow.class文件,因为,在这里我主要是验证:
			//    当你对流程图中的变量进行改变时,是会触发ClassLoader重新加载的.
			// ********************************************************
            JAVA_COMPILER.compile(javaSource, new File(dirPath), new CompileOption());
			
			// 5. 读取:KtvFlow.class文件
            File classFile = new File(dirFile, fullClassName.replace('.', File.separatorChar) + ".class");
            byte[] classBytes = FileUtils.readFileToByteArray(classFile);
			
			// ********************************************************
			// 6. 通过自定义的ClassLoader(FlowClassLoader)加载KtvFlow.class
			// ********************************************************
            return FlowClassLoader.getInstance().defineClass(fullClassName, classBytes);
        } catch (CompileFlowException e) {
            throw e;
        } catch (Exception e) {
            throw new CompileFlowException(e.getMessage(), e);
        }
    } // end compileJavaCode
}

```
### (3). FlowClassLoader
```
package com.alibaba.compileflow.engine.process.preruntime.compiler.impl;

import com.alibaba.compileflow.engine.common.CompileFlowException;
import com.alibaba.compileflow.engine.process.preruntime.compiler.CompileConstants;

import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;


public class FlowClassLoader extends URLClassLoader {

    private static volatile FlowClassLoader instance = null;

    public FlowClassLoader(URL[] urls, ClassLoader parent) {
        super(urls, parent);
    }

	
    public static FlowClassLoader getInstance() {
        if (instance == null) {
            synchronized (FlowClassLoader.class) {
                if (instance == null) {
                    try {
						// 定义要加载class的目录
                        URL url = new URL("file:///" + CompileConstants.FLOW_COMPILE_CLASS_DIR);
                        instance = new FlowClassLoader(new URL[] {url},
                            FlowClassLoader.class.getClassLoader());
                    } catch (MalformedURLException e) {
                        throw new CompileFlowException(e);
                    }
                }
            }
        }
        return instance;
    }

	// ********************************************************************
	// 看到没有,流程图的改变,是会造成ClassLoader进行热加载.  
	// ********************************************************************
    public void clearCache() {
        instance = null;
    }

	// ********************************************************************
	// 把字节码交给父类(URLClassLoader)进行加载.
	// ********************************************************************
    public Class<?> defineClass(String name, byte[] classBytes) {
        return defineClass(name, classBytes, 0, classBytes.length);
    }

}
```
### (4). 总结
> Compileflow相比责任链的实现,有它的优势,但是,缺点也很明确,其实,通过分析这个,你自己也能实现Class的热加载.  