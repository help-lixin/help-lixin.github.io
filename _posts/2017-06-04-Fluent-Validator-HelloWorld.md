---
layout: post
title: 'Fluent Validator Hello World(一)'
date: 2017-06-04
author: 李新
tags: fluent-validator
---

### (1). fluent-validator是什么?
> FluentValidator是百度开源的一个验证框架,适用于以Java语言开发的程序,让开发人员把重点回归到业务逻辑上,使用流式(Fluent Interface)调用风格让验证跑起来很优雅,同时验证器(Validator)可以做到开闭原则,实现最大程度的复用.  

### (2). 使用步骤
> 1. 添加fluent-validator依赖.  
> 2. 定义验证Handler.  
> 3. 定义要验证的Object.   
> 4. 验证.

### (3). pom.xml
```
<dependency>
	<groupId>com.baidu.unbiz</groupId>
	<artifactId>fluent-validator</artifactId>
	<version>1.0.5</version>
	<exclusions>
		<exclusion>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-log4j12</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```
### (4). CarSeatCountValidator
```
class CarSeatCountValidator 
      extends ValidatorHandler<Integer> 
	  implements Validator<Integer> {
		  
	@Override
	public boolean validate(ValidatorContext context, Integer t) {
		if (t < 2) {
			ValidationError error = new ValidationError();
			error.setErrorCode(99999);
			error.setErrorMsg(String.format("Seat count is not valid, invalid value=%s", t));
			context.addError(error);
			return false;
		}
		return true;
	}
}
```
### (5). 定义要验证的业务模型Car
```
class Car {
	private String manufacturer;
	private String licensePlate;
	private int seatCount;
	// ... ... set/get
}
```
### (6). Test
```
public static void main(String[] args) {
	// 1. 要验证的对象
	Car car = new Car();
	
	// 2. 定义验证后的回调函数
	ValidateCallback callback = new ValidateCallback() {
		@Override
		public void onUncaughtException(Validator validator, Exception e, Object target) throws Exception {
		}

		@Override
		public void onSuccess(ValidatorElementList validatorElementList) {
		}

		@Override
		public void onFail(ValidatorElementList validatorElementList, List<ValidationError> errors) {
			System.out.println(errors);
		}
	};
	
	// 开始验证
	FluentValidator
		// 创建:FluentValidator,所以是安全的.
		.checkAll()
		// 验证规则不通过时,立即返回,不再进行后续的验证了
		.failFast().on(car.getSeatCount(), new CarSeatCountValidator())
		// 验证,并回调
		.doValidate(callback);
}
```
### (7). 总结
> 是不是感觉Fluent-Validator相当的简单易用?那么源码结构是什么样的呢?  