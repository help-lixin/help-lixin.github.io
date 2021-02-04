---
layout: post
title: 'Seata  TCC分支事务之ActionInterceptorHandler(二)'
date: 2021-01-29
author: 李新
tags: Seata-TCC源码
---

### (1). ActionInterceptorHandler.proceed
```
public Map<String, Object> proceed(
						Method method, 
						Object[] arguments, 
						String xid, 
						TwoPhaseBusinessAction businessAction,
						Callback<Object> targetCallback) throws Throwable {
	// 构建返回的参数对象
	Map<String, Object> ret = new HashMap<>(4);

	//TCC name
	// 获得@TwoPhaseBusinessAction(namt="xxxx")注解上name的内容,这个名称要确保唯一
	String actionName = businessAction.name();
	
	// 构建一个Context
	BusinessActionContext actionContext = new BusinessActionContext();
	// 设置全局事务xid
	actionContext.setXid(xid);
	// 设置action名称
	//set action name
	actionContext.setActionName(actionName);

	// *************************************************************************************
	// 2. 向TC注册分支事务的开启,ActionInterceptorHandler.doTccActionLogStore
	// *************************************************************************************
	//Creating Branch Record
	String branchId = doTccActionLogStore(method, arguments, businessAction, actionContext);
	// 设置分支事务ID
	actionContext.setBranchId(branchId);

	// 获得:BusinessActionContext在方法列表中的第几个索引.
	//set the parameter whose type is BusinessActionContext
	// 获得方法的入参列表
	Class<?>[] types = method.getParameterTypes();
	int argIndex = 0;
	for (Class<?> cls : types) { //遍历入参列表
		// 判断参数是否为:BusinessActionContext
		// 如果是:BusinessActionContext,则给:arguments[N]赋值为:actionContext(BusinessActionContext)
		if (cls.getName().equals(BusinessActionContext.class.getName())) {
			arguments[argIndex] = actionContext;
			break;
		}
		argIndex++;
	}
	
	// arguments = {xxxx}
	//the final parameters of the try method
	ret.put(Constants.TCC_METHOD_ARGUMENTS, arguments);
	
	// ******************************************************************
	// targetCallback.execute() 触发业务代码的执行.
	// the final result
	// result = {xxxx}
	// ******************************************************************
	ret.put(Constants.TCC_METHOD_RESULT, targetCallback.execute());
	return ret;
}
```
### (2). ActionInterceptorHandler.doTccActionLogStore
```
protected String doTccActionLogStore(
		Method method, 
		Object[] arguments, 
		TwoPhaseBusinessAction businessAction,
		BusinessActionContext actionContext) {
			
	String actionName = actionContext.getActionName();
	String xid = actionContext.getXid();
	
	// *******************************************************
	// 3.读取方法上的注解@BusinessActionContextParameter和参数,转换成:Map
	// ActionInterceptorHandler.fetchActionRequestContext
	// key = @BusinessActionContextParameter(paramName="xxxx")
	// value = arguments[N]
	// *******************************************************
	
	// 我的参数如下: 
	// accountNo = A
	// amount = 10.0
	Map<String, Object> context = fetchActionRequestContext(method, arguments);
	
	// action-start-time = 1612429992664
	context.put(Constants.ACTION_START_TIME, System.currentTimeMillis());

	// 设置:
	// sys::prepare = prepareMinus
	// sys::commit = commit
	// sys::rollback = rollback
	// actionName = firstTccAction
	//init business context
	initBusinessContext(context, method, businessAction);
	
	// 设置:
	// host-name = 172.17.0.240
	//Init running environment context
	initFrameworkContext(context);
	
	
	actionContext.setActionContext(context);

	// 创建一个Map来包裹所有的变量信息
	//init applicationData
	Map<String, Object> applicationContext = new HashMap<>(4);
	// actionContext = { xxx }
	applicationContext.put(Constants.TCC_ACTION_CONTEXT, context);
	
	// ************************************************************************
	// 把applicationContext序列化之后的字符串如下:
	// {"actionContext":{"amount":10.0,"action-start-time":1612429992664,"sys::prepare":"prepareMinus","accountNo":"A","sys::rollback":"rollback","sys::commit":"commit","host-name":"172.17.0.240","actionName":"firstTccAction"}}
	// ************************************************************************
	String applicationContextStr = JSON.toJSONString(applicationContext);
	try {
		//registry branch record
		// 注册分支事务.
		Long branchId = DefaultResourceManager.get()
		               .branchRegister(BranchType.TCC, actionName, null, xid, applicationContextStr, null);
        // 返回分支事务的ID
		return String.valueOf(branchId);
	} catch (Throwable t) {
		// 出现了异常,本地事务会rollback
		// 全局分支事务也会要起进行canal
		String msg = String.format("TCC branch Register error, xid: %s", xid);
		LOGGER.error(msg, t);
		throw new FrameworkException(t, msg);
	}
}
```
### (3). ActionInterceptorHandler.fetchActionRequestContext
> 这方法没什么好讲的了吧!读取注解和参数,转换成Map对象

```
protected Map<String, Object> fetchActionRequestContext(Method method, Object[] arguments) {
	Map<String, Object> context = new HashMap<>(8);

	Annotation[][] parameterAnnotations = method.getParameterAnnotations();
	for (int i = 0; i < parameterAnnotations.length; i++) {
		for (int j = 0; j < parameterAnnotations[i].length; j++) {
			if (parameterAnnotations[i][j] instanceof BusinessActionContextParameter) {
				BusinessActionContextParameter param = (BusinessActionContextParameter)parameterAnnotations[i][j];
				if (arguments[i] == null) {
					throw new IllegalArgumentException("@BusinessActionContextParameter 's params can not null");
				}
				Object paramObject = arguments[i];
				int index = param.index();
				//List, get by index
				if (index >= 0) {
					@SuppressWarnings("unchecked")
					Object targetParam = ((List<Object>)paramObject).get(index);
					if (param.isParamInProperty()) {
						context.putAll(ActionContextUtil.fetchContextFromObject(targetParam));
					} else {
						context.put(param.paramName(), targetParam);
					}
				} else {
					if (param.isParamInProperty()) {
						context.putAll(ActionContextUtil.fetchContextFromObject(paramObject));
					} else {
						context.put(param.paramName(), paramObject);
					}
				}
			}
		}
	}
	return context;
}
```
### (4). 总结
> ActionInterceptorHandler的主要职责:  
> 1. 根据调用方法上的注解@TwoPhaseBusinessAction和@BusinessActionContextParameter转换成业务模型:Map.        
> 2. 构建上下文对象(BusinessActionContext),承载着上一步的Map信息.   
> 3. 把上一步的BusinessActionContext转换成字符串,向TC发起分支事务的注册(commit/rollback时,需要这些参数的).   
> 4. 执行业务操作.   
> 5. 如果try出现异常会怎么办?try异常的情况下,本地事务rollback,全局事务会向TC汇报,TC会驱动所有的分支事务rollback,分支事务做好幂等性即可.     