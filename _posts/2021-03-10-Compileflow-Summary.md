---
layout: post
title: 'Compileflow 简介与Ktv案例(一)'
date: 2021-02-27
author: 李新
tags: Compileflow
---

### (1). Compileflow是什么?
> Compileflow Process引擎是淘宝工作流TBBPM引擎之一,是专注于纯内存执行,无状态的流程引擎,通过将流程文件转换生成java代码编译执行,简洁高效.  
> Compileflow会识别XML,并生成一个java类,再编译成class,加载内存后反射生存对象缓存起来(有点Jsp转换成Servlet的感觉).等引擎需要执行指定XML的流程时,执行了被创建好的对象类. 

### (2). 案例
> 案例代码来源:https://github.com/alibaba/compileflow.git    
> demo描述:N个人去ktv唱歌,每人唱首歌,ktv消费原价为30元/人,如果总价超过300打九折,小于300按原价付款. 

### (3). ktvExample.bpmn20
> 注意:这个流程设计文档(ktvExample.bpmn20)必须要在(classpath:bpmn20/ktv/目录下).  

```
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns:cf="http://compileflow.alibaba.com"
             xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
             typeLanguage="http://www.w3.org/2001/XMLSchema"
             expressionLanguage="http://www.w3.org/1999/XPath"
             targetNamespace="http://compileflow.alibaba.com">

    <process id="ktv" name="ktv" isExecutable="true">
        <extensionElements>
            <cf:var name="price" description="支付价格" dataType="java.lang.Integer" inOutType="return"/>
            <cf:var name="totalPrice" description="实付价" dataType="java.lang.Integer" inOutType="inner"/>
            <cf:var name="pList" description="人员" dataType="java.util.List&lt;java.lang.String&gt;"
                    inOutType="param"/>
        </extensionElements>

        <startEvent id="start"/>

        <sequenceFlow id="flow1" sourceRef="start" targetRef="singLoop"/>

        <subProcess id="singLoop" name="sing loop">
            <standardLoopCharacteristics cf:collection="pList" cf:elementVar="p"
                                         cf:indexVar="i" cf:elementVarClass="java.lang.String">
            </standardLoopCharacteristics>
            <startEvent id="singStart"/>
            <sequenceFlow id="subFlow1" sourceRef="singStart" targetRef="sing"/>
            <serviceTask id="sing" name="sing task" cf:type="spring-bean"
                         cf:bean="ktvService"
                         cf:class="com.allibaba.compileflow.test.mock.KtvService"
                         cf:method="sing">
                <extensionElements>
                    <cf:var name="p1" description="" dataType="java.lang.String" contextVarName="p" defaultValue=""
                            inOutType="param"/>
                </extensionElements>
            </serviceTask>
            <sequenceFlow id="subFlow2" sourceRef="sing" targetRef="singEnd"/>
            <endEvent id="singEnd"/>
        </subProcess>

        <sequenceFlow id="flow2" sourceRef="singLoop" targetRef="calPrice"/>

        <serviceTask id="calPrice" name="calPrice" cf:type="java"
                     cf:class="com.allibaba.compileflow.test.mock.MockJavaClazz"
                     cf:method="calPrice">
            <extensionElements>
                <cf:var name="p1" description="人数" dataType="java.lang.Integer" contextVarName="pList.size()"
                        defaultValue=""
                        inOutType="param"/>
                <cf:var name="p2" description="价格" dataType="java.lang.Integer" contextVarName="totalPrice"
                        defaultValue=""
                        inOutType="return"/>
            </extensionElements>
        </serviceTask>

        <sequenceFlow id="flow3" sourceRef="calPrice" targetRef="payDecision"/>

        <exclusiveGateway id="payDecision"/>

        <sequenceFlow id="flow4" sourceRef="payDecision" targetRef="originalPrice">
            <conditionExpression cf:type="java">
                <![CDATA[totalPrice>=300]]>
            </conditionExpression>
        </sequenceFlow>
        <sequenceFlow id="flow5" sourceRef="payDecision" targetRef="promotionPrice"/>

        <scriptTask id="originalPrice" name="original price" scriptFormat="ql">
            <extensionElements>
                <cf:var name="price" description="价格" dataType="java.lang.Integer" contextVarName="totalPrice"
                        defaultValue="" inOutType="param"/>
                <cf:var name="price" description="价格" dataType="java.lang.Integer" contextVarName="price"
                        defaultValue=""
                        inOutType="return"/>
            </extensionElements>
            <script><![CDATA[(round(price*0.9,0)).intValue()]]></script>
        </scriptTask>

        <scriptTask id="promotionPrice" name="promotion task" scriptFormat="ql">
            <extensionElements>
                <cf:var name="price" description="价格" dataType="java.lang.Integer" contextVarName="totalPrice"
                        defaultValue="" inOutType="param"/>
                <cf:var name="price" description="价格" dataType="java.lang.Integer" contextVarName="price"
                        defaultValue=""
                        inOutType="return"/>
            </extensionElements>
            <script><![CDATA[(round(price*0.9,0)).intValue()]]></script>
        </scriptTask>

        <sequenceFlow id="flow6" sourceRef="originalPrice" targetRef="pay"/>
        <sequenceFlow id="flow7" sourceRef="promotionPrice" targetRef="pay"/>

        <serviceTask id="pay" name="pay" cf:type="java"
                     cf:class="com.allibaba.compileflow.test.mock.KtvService"
                     cf:method="payMoney">
            <extensionElements>
                <cf:var name="p1" description="价格" dataType="java.lang.Integer"
                        contextVarName="price" defaultValue="" inOutType="param"/>
            </extensionElements>
        </serviceTask>

        <sequenceFlow id="flow8" sourceRef="pay" targetRef="end"/>

        <endEvent id="end"/>
    </process>
</definitions>
```
### (4). ktv.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" default-autowire="byName"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
 http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">

    <bean id="ktvService" class="com.allibaba.compileflow.test.mock.KtvService"/>

</beans>
```
### (5). common.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" default-autowire="byName"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
 http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">

    <bean id="applicationContextProvider"
          class="com.alibaba.compileflow.engine.process.preruntime.generator.bean.SpringApplicationContextProvider"/>

</beans>
```
### (6). orderFulfillment.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
 http://www.springframework.org/schema/beans/spring-beans-2.5.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <context:component-scan base-package="com.allibaba.compileflow.test.om"/>
</beans>
```
### (7). KtvService
```
package com.allibaba.compileflow.test.mock;

import java.io.PrintStream;

/**
 * @author pin
 */
public class KtvService {

    public void sing(String name) {
        System.out.println(name + " is singing");
    }

    public void payMoney(int price) {
        System.out.println("actually paid money: " + price);
    }

}
```
### (8). MockJavaClazz
```
package com.allibaba.compileflow.test.mock;

public class MockJavaClazz {
	
    public int calPrice(int num) {
        System.out.println("total price: " + 30 * num);
        return 30 * num;
    }
}

```
### (9). ProcessEngineTest
```
package com.allibaba.compileflow.test;

import com.alibaba.compileflow.engine.ProcessEngine;
import com.alibaba.compileflow.engine.ProcessEngineFactory;
import com.alibaba.compileflow.engine.common.constants.FlowModelType;
import com.alibaba.compileflow.engine.definition.tbbpm.TbbpmModel;
import com.alibaba.compileflow.engine.process.preruntime.converter.impl.TbbpmModelConverter;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import java.io.OutputStream;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * @author yusu
 */
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {
    "classpath:bean/common.xml", "classpath:bean/ktv.xml", "classpath:bean/orderFulfillment.xml"
})
public class ProcessEngineTest {
	
	@Test
	public void testProcessEngineBpmn20() {
		final String code = "bpmn20.ktv.ktvExample";

		final Map<String, Object> context = new HashMap<>();
		List<String> pList = new ArrayList<>();
		pList.add("wuxiang");
		pList.add("yusu");
		context.put("pList", pList);

		final ProcessEngine processEngine = ProcessEngineFactory.getStatelessProcessEngine(FlowModelType.BPMN);
		System.out.println(processEngine.start(code, context));
	}
}
```
### (10). 运行结果
!["compileflow 运行后产生class文件"](/assets/compileflow/imgs/compileflow.jpg)


```
package compileflow;

import java.util.Map;
import java.lang.Integer;
import com.ql.util.express.DefaultContext;
import java.util.List;
import com.alibaba.compileflow.engine.process.preruntime.generator.script.ScriptExecutorProvider;
import com.allibaba.compileflow.test.mock.KtvService;
import java.lang.String;
import java.util.HashMap;
import com.alibaba.compileflow.engine.runtime.instance.ProcessInstance;
import com.alibaba.compileflow.engine.common.utils.DataType;
import com.allibaba.compileflow.test.mock.MockJavaClazz;
import com.ql.util.express.IExpressContext;
import com.alibaba.compileflow.engine.common.utils.ObjectFactory;
import com.alibaba.compileflow.engine.ProcessEngineFactory;
import com.alibaba.compileflow.engine.process.preruntime.generator.bean.BeanProvider;

public class KtvFlow implements ProcessInstance {

    private java.util.List<java.lang.String> pList = null;
    private java.lang.Integer price = null;
    private java.lang.Integer totalPrice = null;

    public Map<String, Object> execute(Map<String, Object> _pContext) throws Exception {
        pList = (List)DataType.transfer(_pContext.get("pList"), List.class);
        Map<String, Object> _pResult = new HashMap<>();
        int i = -1;
        for (String p : pList) {
            i++;
            //SubProcess
            {
                //ServiceTask
                ((KtvService)BeanProvider.getBean("ktvService")).sing((String)DataType.transfer(p, String.class));
            }
        }
        //ServiceTask
        totalPrice = ((MockJavaClazz)ObjectFactory.getInstance("com.allibaba.compileflow.test.mock.MockJavaClazz")).calPrice((Integer)DataType.transfer(pList.size(), Integer.class));
        exclusiveGatewayPayDecision();
        //ServiceTask
        ((KtvService)ObjectFactory.getInstance("com.allibaba.compileflow.test.mock.KtvService")).payMoney(price);
        _pResult.put("price", price);
        return _pResult;
    }

    private void exclusiveGatewayPayDecision() {
        //ExclusiveGateway
        if (totalPrice>=300) {
            IExpressContext<String, Object> nfScriptContext = new DefaultContext<>();
            nfScriptContext.put("price", totalPrice);
            price = (java.lang.Integer)ScriptExecutorProvider.getInstance().getScriptExecutor("QL").execute("(round(price*0.9,0)).intValue()", nfScriptContext);
        } else {
            IExpressContext<String, Object> nfScriptContext = new DefaultContext<>();
            nfScriptContext.put("price", totalPrice);
            price = (java.lang.Integer)ScriptExecutorProvider.getInstance().getScriptExecutor("QL").execute("(round(price*0.9,0)).intValue()", nfScriptContext);
        }
    }
}
```

### (11). 总结
> compileflow无非不过就是把流程图(XML)转换成了Java代码,从项目的命名(compileFlow)就能看出来.    
> 为什么要编译成Java代码,优势和缺点是什么?    
> 优势:相比那种责任链的模式的实现,这样做,性能更优.     
> 缺点:CompileFlow底层肯定是自定义了ClassLoader,如果,流程图的更改(哪怕是个小小的变量),就会触发ClassLoader重新加载.    
> 下一小节,剖析生成Java代码和ClassLoader加载,用来验证我的观点.     