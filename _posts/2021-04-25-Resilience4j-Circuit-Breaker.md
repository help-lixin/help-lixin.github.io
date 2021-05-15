---
layout: post
title: 'Resilience4j Circuit Breaker使用案例(二)'
date: 2021-04-25
author: 李新
tags:  Resilience4j
---

### (1). 简介
> Circuit Breaker通过具有三种正常状态的有限状态机来实现:CLOSED,OPEN和HALF_OPEN以及两个特殊状态:DISABLED和FORCED_OPEN.   
> 当熔断器关闭(close)时,所有的请求都会通过熔断器.  
> 如果失败率超过设定的阈值,熔断器就会从关闭状态转换到打开状态(open),这时所有的请求都会被拒绝.  
> 当经过一段时间后,熔断器会从打开状态转换到半开状态(half_open),这时仅有一定数量的请求会被放入,并重新计算失败率.     
> 如果失败率超过阈值,则变为打开状态(open),如果失败率低于阈值,则变为关闭状态(close).   

### (2). CircuitBreaker配置

|  配置属性                                      | 默认值       | 描述  |
|  ----                                         | ----        | ----  |
| failureRateThreshold                          | 50          | 失败请求百分比,超过这个比例,CircuitBreaker就会变成OPEN状态 |
| slowCallDurationThreshold                     | 60000(ms)   | 慢调用时间,当一个调用慢于这个时间时，会被记录为慢调用 |
| slowCallRateThreshold                         | 100         | 当慢调用达到这个百分比的时候,CircuitBreaker就会变成OPEN状态 |
| permittedNumberOfCallsInHalfOpenState         | 10          | 当CircuitBreaker处于HALF_OPEN状态的时候,允许通过的请求数量 |
| slidingWindowType	                            | COUNT_BASED | 滑动窗口类型:COUNT_BASED代表是基于计数的滑动窗口,TIME_BASED代表是基于计时的滑动窗口 |
| slidingWindowSize                             | 100         | 滑动窗口大小,如果配置COUNT_BASED默认值100就代表是最近100个请求,如果配置TIME_BASED代表记录:最近100s的请求. |
| minimumNumberOfCalls                          | 100         | 最小请求个数.只有在滑动窗口内,请求个数达到这个个数,才会触发CircuitBreaker对于是否打开断路器的判断 |
| waitDurationInOpenState                       | 60000(ms)   | 从OPEN状态变成HALF_OPEN状态需要的等待时间 |
| automaticTransitionFromOpenToHalfOpenEnabled  | false       | 如果设置为true代表:是否自动从OPEN状态变成HALF_OPEN,即使没有请求过来. |
| recordExceptions                              | empty       | 需要记录为失败的异常列表 |
| ignoreExceptions                              | empty       | 需要忽略的异常列表 |


### (3). CircuitBreaker配置案例

```
// 1. 所有Exception以及其子类都认为是失败.
// 2. 滑动窗口采用基于计时的,并且记录最近10秒的请求.
// 3. 触发断路器判断必须在10秒内至少有5个请求,在失败比例达到30%以上之后,断路器变为:OPEN.  
// 4. 断路器OPEN之后,在2秒后自动转化为HALF_OPEN.
// 5. 断路器在HALF_OPEN之后,允许通过的请求数量为:3个
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
		// 滑动窗口类型(TIME_BASED:时间 / COUNT_BASED:计数器 )
		.slidingWindowType(SlidingWindowType.TIME_BASED)
		// 滑动窗口大小(记录最近10秒的请求)
		.slidingWindowSize(10)
		// 最小请求个数.只有在滑动窗口内,请求个数达到这个个数,才会触发CircuitBreaker对于是否打开断路器的判断
		.minimumNumberOfCalls(5)
		// 当CircuitBreaker处于HALF_OPEN状态的时候,允许通过的请求数量
		.permittedNumberOfCallsInHalfOpenState(3)
		// 自动从OPEN状态变成HALF_OPEN,即使没有请求过来.
		.automaticTransitionFromOpenToHalfOpenEnabled(true)
		// 从OPEN状态变成HALF_OPEN状态需要的等待2秒
		.waitDurationInOpenState(Duration.ofSeconds(2))
		// 失败率达到30%,CircuitBreaker就会变成OPEN状态
		.failureRateThreshold(30)
		// 所有Exception异常会统计为失败.
		.recordExceptions(Exception.class)
		// 
		.build();
```
### (4). pom.xml
```
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>Greenwich.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-dependencies</artifactId>
			<version>2.1.0.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-netflix</artifactId>
			<version>2.1.1.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

<dependencies>
	 <dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency> 
		<groupId>io.github.resilience4j</groupId>
		<artifactId>resilience4j-spring</artifactId>
		<version>1.7.0</version>
	</dependency>
	<dependency>
		<groupId>junit</groupId>
		<artifactId>junit</artifactId>
		<scope>test</scope>
	</dependency>
	
	<dependency>
		<groupId>org.assertj</groupId>
		<artifactId>assertj-core</artifactId>
	</dependency>
	<dependency>
		<groupId>org.mockito</groupId>
		<artifactId>mockito-core</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```
### (5). HelloWorldService
> HelloWorldService实际为业务代码

```
package help.lixin.resilience4j;

import io.vavr.control.Either;
import io.vavr.control.Try;

import java.io.IOException;
import java.util.concurrent.Future;

public interface HelloWorldService {
	
	String returnHelloWorld();

    Future<String> returnHelloWorldFuture();

    Either<HelloWorldException, String> returnEither();

    Try<String> returnTry();

    String returnHelloWorldWithException() throws IOException;

    String returnHelloWorldWithName(String name);

    String returnHelloWorldWithNameWithException(String name) throws IOException;

    void sayHelloWorld();

    void sayHelloWorldWithException() throws IOException;

    void sayHelloWorldWithName(String name);

    void sayHelloWorldWithNameWithException(String name) throws IOException;
}
```
### (6). HelloWorldException
>  HelloWorldException为自定义的业务异常.  

```
package help.lixin.resilience4j;

public class HelloWorldException extends RuntimeException {

    public HelloWorldException() {
        super("BAM!");
    }

    public HelloWorldException(String message) {
        super(message);
    }
}
```
### (7). CircuitBreakerTest
> CircuitBreakerTest单元测试

```
@Test
public void test() {
	// 1. 所有Exception以及其子类都认为是失败.
	// 2. 滑动窗口采用基于统计的,并且记录最近10个的请求.
	// 3. 触发断路器判断至少有5个请求,在失败比例达到20%以上之后,断路器变为:OPEN.
	// 4. 断路器OPEN之后,在2秒后自动转化为HALF_OPEN.
	// 5. 断路器在HALF_OPEN之后,允许通过的请求数量为:3个
	CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
			// 滑动窗口类型(TIME_BASED:时间 / COUNT_BASED:计数器 )
			.slidingWindowType(SlidingWindowType.COUNT_BASED)
			// 滑动窗口大小(统计最近的10个请求)
			.slidingWindowSize(10)
			// 最小请求个数.只有在滑动窗口内,请求个数达到这个个数,才会触发CircuitBreaker对于是否打开断路器的判断
			.minimumNumberOfCalls(5)
			// 当CircuitBreaker处于HALF_OPEN状态的时候,允许通过的请求数量
			.permittedNumberOfCallsInHalfOpenState(3)
			// 自动从OPEN状态变成HALF_OPEN,即使没有请求过来.
			.automaticTransitionFromOpenToHalfOpenEnabled(true)
			// 从OPEN状态变成HALF_OPEN状态需要的等待2秒
			.waitDurationInOpenState(Duration.ofSeconds(2))
			// 失败率达到30%,CircuitBreaker就会变成OPEN状态
			.failureRateThreshold(20)
			// 所有Exception异常会统计为失败.
			.recordExceptions(Exception.class)
			//
			.build();
	
		CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(circuitBreakerConfig);
		CircuitBreaker circuitBreaker = registry.circuitBreaker("test");

		// 业务代码
		HelloService helloService = new HelloService();
		for (int i = 1; i <= 12; i++) {
			int index = i;
			Supplier<String> supplier = circuitBreaker.decorateSupplier(() -> {
				return helloService.sayHello(index);
			});

			if (i == 6) {
				try {
					TimeUnit.SECONDS.sleep(3);
					System.out.println("****************************************************************");
				} catch (InterruptedException e1) {
				}
			}

			Try.ofSupplier(supplier) //
					.onSuccess(s -> {
						System.out.println(new Date() + "  " + s + "  " + circuitBreaker.getState());
					}).onFailure(e -> {
						System.err.println(new Date() + "  " + e.getMessage() + "  " + circuitBreaker.getState());
					});
		} // end  for
}// end test


// HelloService
class HelloService {
	public String sayHello(int index) {
		if (index < 6 && index % 2 == 0) {
			throw new RuntimeException("fail index->" + index);
		}
		return "success index : " + index;
	}
}
```

### (8). 日志验证
```
# 请求成功(CircuitBreaker状态为:CLOSE)
Sat May 15 21:49:29 CST 2021  success index : 1  CLOSED
# 请求失败(CircuitBreaker状态为:CLOSE)
Sat May 15 21:49:29 CST 2021  fail index->2  CLOSED
# 请求成功(CircuitBreaker状态为:CLOSE)
Sat May 15 21:49:29 CST 2021  success index : 3  CLOSED
# 请求失败(CircuitBreaker状态为:CLOSE)
Sat May 15 21:49:29 CST 2021  fail index->4  CLOSED

# ************************************************************
# 从第5个请求开始计算失败率CircuitBreaker状态为:OPEN)
Sat May 15 21:49:29 CST 2021  success index : 5  OPEN

# ************************************************************
# 在第6个请求的时候,我让程序休眠了3秒,因为,状态一旦OPEN之后,所有的请求都是会被拒绝的,只有等待2秒后,回归到:HALF_OPEN,才开始接受一部份请求.
# OPEN的时间为:29秒,HALF_OPEN的时间为:32秒
Sat May 15 21:49:32 CST 2021  success index : 6  HALF_OPEN
Sat May 15 21:49:32 CST 2021  success index : 7  HALF_OPEN

# 开始正确接受请求(CircuitBreaker状态为:CLOSE)
Sat May 15 21:49:32 CST 2021  success index : 8  CLOSED
Sat May 15 21:49:32 CST 2021  success index : 9  CLOSED
Sat May 15 21:49:32 CST 2021  success index : 10  CLOSED
Sat May 15 21:49:32 CST 2021  success index : 11  CLOSED
Sat May 15 21:49:32 CST 2021  success index : 12  CLOSED
```
### (8). 总结
> 熔断与限流的区别在于:熔断之后,还存在半打开状态,同时,熔断会进行降级处理,而限流是直接拒绝请求了,缺少一个智能的半打开.    