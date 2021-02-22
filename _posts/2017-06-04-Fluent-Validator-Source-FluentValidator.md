---
layout: post
title: 'Fluent-Validator FluentValidator类(三)'
date: 2017-06-04
author: 李新
tags: fluent-validator
---

### (1). 概述
> 在这里主要对FluentValidator进行剖析.

### (2). 看下FluentValidator类图
!["FluentValidator类图"](/assets/fluent-validator/imgs/FluentValidator-Class-Diagram.jpg)

### (3). FluentValidator使用
```
FluentValidator 
	// *****************************************************
	// 4. 创建FluentValidator对象
	// *****************************************************
	.checkAll()
	// *****************************************************
	// 5. FluentValidator进行验证时,验证不通过直接退出,不再继续对其它对象进行验证了
	// *****************************************************
	// 验证规则不通过时,立即返回,不再进行后续的验证了
	.failFast()
	// *****************************************************
	// 6. 配置验证对象与验证规则的关系
	// *****************************************************
	.on(car.getSeatCount(), new CarSeatCountValidator()) //
	// *****************************************************
	// 7. 开始验证,并回调
	// *****************************************************
	.doValidate(callback);
```
### (4). FluentValidator.checkAll
```
public static FluentValidator checkAll() {
	return checkAll(null);
} 

public static FluentValidator checkAll(Class... groups) {
	// 创建了一个:FluentValidator对象
	return new FluentValidator().setGroups(groups);
}
```
### (5). FluentValidator.failFast
```
public FluentValidator failFast() {
	this.isFailFast = true;
	return this;
}
```
### (6). FluentValidator.on
```
public <T> FluentValidator on(T t, Validator<T> v) {
	Preconditions.checkNotNull(v, "Validator should not be NULL");
	// 1. 如果Validator属于:ValidatorHandler,则调用:ValidatorHandler.compose
	composeIfPossible(v, t);
	// 2. 创建:ValidatorElement Hold住:验证规则(Validator)和验证对象(属性)
	// 3. 添加到:validatorElementList里
	doAdd(new ValidatorElement(t, v));
	lastAddCount = 1;
	return this;
} // end on


private <T>  void composeIfPossible(Validator<T> v, T t) {
	final FluentValidator self = this;
	if (v instanceof ValidatorHandler) {
		((ValidatorHandler) v).compose(self, context, t);
	}
} // end composeIfPossible
```
### (7). FluentValidator.doValidate
```
public FluentValidator doValidate(ValidateCallback cb) {
	Preconditions.checkNotNull(cb, "ValidateCallback should not be NULL");
	// 1. 如果待验证的列表为空,则直接返回
	if (validatorElementList.isEmpty()) {
		LOGGER.debug("Nothing to validate");
		return this;
	}
	
	// 2. 给上下文设置一个默认的:result
	context.setResult(result);

	// 打印日志
	LOGGER.debug("Start to validate through " + validatorElementList);
	long start = System.currentTimeMillis();
	try {
		// ThreadLocal设置groups
		GroupingHolder.setGrouping(groups);
		// ************************************************************
		// 3. 遍历所有的验证规则集合
		// ************************************************************
		for (ValidatorElement element : validatorElementList.getAllValidatorElements()) {
			// 4. 要验证的对象
			Object target = element.getTarget();
			// 5. 验证的规则
			Validator v = element.getValidator();
			try {
				// *******************************************************
				// 6. 先调用:Validator.accept,只有为:true时,才调用:validate
				// *******************************************************
				if (v.accept(context, target)) {
					// *******************************************************
					// 7. 再调用:Validator.validate
					// *******************************************************
					if (!v.validate(context, target)) { //验证不通过时
						result.setIsSuccess(false); // 设置验证不通过
						
						if (isFailFast) { // 如果:isFailFast为true,跳出循环体,不再进行余下的验证了
							break;
						}// end if
					} // end validate
				} // end accept
			} catch (Exception e) {
				try {
					// *****************************************************
					// 8. 有异常时,调用:Validator.onException
					// *****************************************************
					v.onException(e, context, target);
					// 9. 回调:ValidateCallback
					cb.onUncaughtException(v, e, target);
				} catch (Exception e1) {
					if (LOGGER.isDebugEnabled()) {
						LOGGER.error(v + " onException or onUncaughtException throws exception due to " + e1
								.getMessage(), e1);
					}
					throw new RuntimeValidateException(e1);
				}
				if (LOGGER.isDebugEnabled()) {
					LOGGER.error(v + " failed due to " + e.getMessage(), e);
				}
				throw new RuntimeValidateException(e);
			}
		}

		if (result.isSuccess()) {  // 验证成功,回调:ValidateCallback.onSuccess
			cb.onSuccess(validatorElementList);
		} else { // 验证失败,回调:ValidateCallback.onFail
			cb.onFail(validatorElementList, result.getErrors());
		}
	} finally {
		// 清除ThreadLocal信息
		GroupingHolder.clean();
		// 计算验证的成本
		int timeElapsed = (int) (System.currentTimeMillis() - start);
		LOGGER.debug("End to validate through" + validatorElementList + " costing " + timeElapsed + "ms with "
				+ "isSuccess=" + result.isSuccess());
		result.setTimeElapsed(timeElapsed);
	}
	return this;
}
```
### (8). 总结
> 好像没什么要总结的,FluentValidator源码相当的简单!    