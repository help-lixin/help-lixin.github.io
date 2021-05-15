---
layout: post
title: 'Resilience4j Circuit Breaker使用案例(二)'
date: 2021-04-25
author: 李新
tags:  Resilience4j
---

### (1). 简介
> Circuit Breaker通过具有三种正常状态的有限状态机实现:CLOSED,OPEN和HALF_OPEN以及两个特殊状态:DISABLED和FORCED_OPEN.   
> 当熔断器关闭(close)时,所有的请求都会通过熔断器.  
> 如果失败率超过设定的阈值,熔断器就会从关闭状态转换到打开状态(open),这时所有的请求都会被拒绝.  
> 当经过一段时间后,熔断器会从打开状态转换到半开状态(half_open),这时仅有一定数量的请求会被放入,并重新计算失败率.     
> 如果失败率超过阈值,则变为打开状态(open),如果失败率低于阈值,则变为关闭状态(close).   

### (2). pom.xml
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
### (3). HelloWorldService
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
### (4). HelloWorldException
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
### (5). CircuitBreakerTest
> CircuitBreakerTest单元测试

```
package help.lixin.resilience4j;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.BDDMockito.given;
import static org.mockito.BDDMockito.then;
import static org.mockito.Mockito.*;

import java.io.IOException;
import java.net.SocketTimeoutException;
import java.time.Duration;
import java.util.concurrent.Callable;
import java.util.concurrent.CancellationException;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionException;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

import org.junit.Before;
import org.junit.Test;

import io.github.resilience4j.circuitbreaker.CallNotPermittedException;
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.vavr.CheckedConsumer;
import io.vavr.CheckedFunction0;
import io.vavr.CheckedFunction1;
import io.vavr.CheckedRunnable;
import io.vavr.control.Either;
import io.vavr.control.Try;

public class CircuitBreakerTest {
	private HelloWorldService helloWorldService;

	@Before
	public void init() {
		helloWorldService = mock(HelloWorldService.class);
	}

	@Test
	public void shouldDecorateSupplierAndReturnWithSuccess() {
		CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
		CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
		assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
		
		given(helloWorldService.returnHelloWorld()).willReturn("Hello world");

		// decorateSupplier
		Supplier<String> supplier = circuitBreaker.decorateSupplier(helloWorldService::returnHelloWorld);

		String result = supplier.get();

		assertThat(result).isEqualTo("Hello world");
		assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
		assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
		assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
		then(helloWorldService).should().returnHelloWorld();
	}
	
	
	@Test
    public void shouldExecuteSupplierAndReturnWithSuccess() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        
        given(helloWorldService.returnHelloWorld()).willReturn("Hello world");

        // executeSupplier
        String result = circuitBreaker.executeSupplier(helloWorldService::returnHelloWorld);

        assertThat(result).isEqualTo("Hello world");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnHelloWorld();
    }
	
	@Test
    public void shouldDecorateSupplierAndReturnWithException() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorld()).willThrow(new RuntimeException("BAM!"));
        // decorateSupplier
        Supplier<String> supplier = circuitBreaker.decorateSupplier(helloWorldService::returnHelloWorld);
        
        // ofSupplier
        Try<String> result = Try.ofSupplier(supplier);

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorld();
    }
	
	@Test
    public void shouldDecorateCheckedSupplierAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorldWithException()).willReturn("Hello world");
        
        CheckedFunction0<String> checkedSupplier = circuitBreaker.decorateCheckedSupplier(helloWorldService::returnHelloWorldWithException);
        // apply
        String result = checkedSupplier.apply();

        assertThat(result).isEqualTo("Hello world");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnHelloWorldWithException();
    }
	
	@Test
    public void shouldDecorateCheckedSupplierAndReturnWithException() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        
        given(helloWorldService.returnHelloWorldWithException()).willThrow(new RuntimeException("BAM!"));
        // decorateCheckedSupplier
        CheckedFunction0<String> checkedSupplier = circuitBreaker.decorateCheckedSupplier(helloWorldService::returnHelloWorldWithException);
        
        // 
        Try<String> result = Try.of(checkedSupplier);

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorldWithException();
    }
	
	@Test
    public void shouldDecorateCallableAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorldWithException()).willReturn("Hello world");
        
        // callable
        Callable<String> callable = circuitBreaker.decorateCallable(helloWorldService::returnHelloWorldWithException);

        String result = callable.call();

        assertThat(result).isEqualTo("Hello world");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnHelloWorldWithException();
    }
	
	@Test
    public void shouldExecuteCallableAndReturnWithSuccess() throws Throwable {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorldWithException()).willReturn("Hello world");

        String result = circuitBreaker.executeCallable(helloWorldService::returnHelloWorldWithException);

        assertThat(result).isEqualTo("Hello world");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnHelloWorldWithException();
    }
	
	@Test
    public void shouldDecorateCallableAndReturnWithException() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorldWithException())
            .willThrow(new RuntimeException("BAM!"));
        // 
        Callable<String> callable = circuitBreaker.decorateCallable(helloWorldService::returnHelloWorldWithException);

        Try<String> result = Try.of(callable::call);

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorldWithException();
    }
	
	@Test
    public void shouldDecorateCheckedRunnableAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        CheckedRunnable checkedRunnable = circuitBreaker
            .decorateCheckedRunnable(helloWorldService::sayHelloWorldWithException);

        checkedRunnable.run();

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().sayHelloWorldWithException();
    }

    @Test
    public void shouldDecorateCheckedRunnableAndReturnWithException() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        CheckedRunnable checkedRunnable = circuitBreaker.decorateCheckedRunnable(() -> {
            throw new RuntimeException("BAM!");
        });

        Try<Void> result = Try.run(checkedRunnable);

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
    }

    @Test
    public void shouldDecorateRunnableAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        Runnable runnable = circuitBreaker.decorateRunnable(helloWorldService::sayHelloWorld);

        runnable.run();

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().sayHelloWorld();
    }

    @Test
    public void shouldExecuteRunnableAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        circuitBreaker.executeRunnable(helloWorldService::sayHelloWorld);

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().sayHelloWorld();
    }


    @Test
    public void shouldDecorateRunnableAndReturnWithException() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        Runnable runnable = circuitBreaker.decorateRunnable(() -> {
            throw new RuntimeException("BAM!");
        });

        Try<Void> result = Try.run(runnable::run);

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
    }

    @Test
    public void shouldDecorateConsumerAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        Consumer<String> consumer = circuitBreaker
            .decorateConsumer(helloWorldService::sayHelloWorldWithName);

        consumer.accept("Tom");

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().sayHelloWorldWithName("Tom");
    }

    @Test
    public void shouldDecorateConsumerAndReturnWithException() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        Consumer<String> consumer = circuitBreaker.decorateConsumer((value) -> {
            throw new RuntimeException("BAM!");
        });

        Try<Void> result = Try.run(() -> consumer.accept("Tom"));

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
    }

    @Test
    public void shouldDecorateCheckedConsumerAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        CheckedConsumer<String> checkedConsumer = circuitBreaker
            .decorateCheckedConsumer(helloWorldService::sayHelloWorldWithNameWithException);

        checkedConsumer.accept("Tom");

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().sayHelloWorldWithNameWithException("Tom");
    }

    @Test
    public void shouldDecorateCheckedConsumerAndReturnWithException() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        int numberOfBufferedCallsBefore = metrics.getNumberOfBufferedCalls();
        CheckedConsumer<String> checkedConsumer = circuitBreaker
            .decorateCheckedConsumer((value) -> {
                throw new RuntimeException("BAM!");
            });

        Try<Void> result = Try.run(() -> checkedConsumer.accept("Tom"));

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(numberOfBufferedCallsBefore).isEqualTo(0);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
    }

    @Test
    public void shouldDecorateFunctionAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorldWithName("Tom")).willReturn("Hello world Tom");
        Function<String, String> function = CircuitBreaker
            .decorateFunction(circuitBreaker, helloWorldService::returnHelloWorldWithName);

        String result = function.apply("Tom");

        assertThat(result).isEqualTo("Hello world Tom");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnHelloWorldWithName("Tom");
    }

    @Test
    public void shouldDecorateFunctionAndReturnWithException() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorldWithName("Tom"))
            .willThrow(new RuntimeException("BAM!"));
        Function<String, String> function = CircuitBreaker
            .decorateFunction(circuitBreaker, helloWorldService::returnHelloWorldWithName);

        Try<String> result = Try.of(() -> function.apply("Tom"));

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
    }

    @Test
    public void shouldDecorateCheckedFunctionAndReturnWithSuccess() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorldWithNameWithException("Tom"))
            .willReturn("Hello world Tom");
        CheckedFunction1<String, String> function = CircuitBreaker
            .decorateCheckedFunction(circuitBreaker,
                helloWorldService::returnHelloWorldWithNameWithException);

        String result = function.apply("Tom");

        assertThat(result).isEqualTo("Hello world Tom");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnHelloWorldWithNameWithException("Tom");
    }


    @Test
    public void shouldDecorateCheckedFunctionAndReturnWithException() throws Throwable {
        CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnHelloWorldWithNameWithException("Tom"))
            .willThrow(new RuntimeException("BAM!"));
        CheckedFunction1<String, String> function = CircuitBreaker
            .decorateCheckedFunction(circuitBreaker,
                helloWorldService::returnHelloWorldWithNameWithException);

        Try<String> result = Try.of(() -> function.apply("Tom"));

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
    }

    @Test
    public void shouldReturnFailureWithCircuitBreakerOpenException() {
        // Given
        CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
            .slidingWindowSize(2)
            .permittedNumberOfCallsInHalfOpenState(2)
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofMillis(1000))
            .build();
        CircuitBreaker circuitBreaker = CircuitBreaker.of("testName", circuitBreakerConfig);
        CheckedRunnable checkedRunnable = circuitBreaker.decorateCheckedRunnable(() -> {
            throw new RuntimeException("BAM!");
        });

        // When
        circuitBreaker.onError(0, TimeUnit.NANOSECONDS, new RuntimeException());
        circuitBreaker.onError(0, TimeUnit.NANOSECONDS, new RuntimeException());

        // Then
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(2);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(2);

        // When
        Try result = Try.run(checkedRunnable);

        // Then
        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(CallNotPermittedException.class);
    }

    @Test
    public void shouldReturnFailureWithRuntimeException() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        CheckedRunnable checkedRunnable = circuitBreaker.decorateCheckedRunnable(() -> {
            throw new RuntimeException("BAM!");
        });

        Try result = Try.run(checkedRunnable);

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
    }

    @Test
    public void shouldNotRecordIOExceptionAsAFailure() {
        // tag::shouldNotRecordIOExceptionAsAFailure[]
        CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
            .slidingWindowSize(2)
            .permittedNumberOfCallsInHalfOpenState(2)
            .waitDurationInOpenState(Duration.ofMillis(1000))
            .ignoreExceptions(IOException.class)
            .build();
        CircuitBreaker circuitBreaker = CircuitBreaker.of("testName", circuitBreakerConfig);
        // Simulate a failure attempt
        circuitBreaker.onError(0, TimeUnit.NANOSECONDS, new HelloWorldException());
        // CircuitBreaker is still CLOSED, because 1 failure is allowed
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        CheckedRunnable checkedRunnable = circuitBreaker.decorateCheckedRunnable(() -> {
            throw new SocketTimeoutException("BAM!");
        });

        Try result = Try.run(checkedRunnable);

        assertThat(result.isFailure()).isTrue();
        // CircuitBreaker is still CLOSED, because SocketTimeoutException has not been recorded as a failure
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        assertThat(result.failed().get()).isInstanceOf(IOException.class);
        // end::shouldNotRecordIOExceptionAsAFailure[]
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
    }

    @Test
    public void shouldInvokeRecoverFunction() {
        // tag::shouldInvokeRecoverFunction[]
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        // When I decorate my function and invoke the decorated function
        CheckedFunction0<String> checkedSupplier = circuitBreaker.decorateCheckedSupplier(() -> {
            throw new RuntimeException("BAM!");
        });

        Try<String> result = Try.of(checkedSupplier)
            .recover(throwable -> "Hello Recovery");

        // Then the function should be a success, because the exception could be recovered
        assertThat(result.isSuccess()).isTrue();
        // and the result must match the result of the recovery function.
        assertThat(result.get()).isEqualTo("Hello Recovery");
        // end::shouldInvokeRecoverFunction[]
    }

    @Test
    public void shouldInvokeMap() {
        // tag::shouldInvokeMap[]
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        // When I decorate my function
        CheckedFunction0<String> decoratedSupplier = CircuitBreaker
            .decorateCheckedSupplier(circuitBreaker,
                () -> "This can be any method which returns: 'Hello");

        // and chain an other function with map
        Try<String> result = Try.of(decoratedSupplier)
            .map(value -> value + " world'");

        // Then the Try Monad returns a Success<String>, if all functions ran successfully.
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.get()).isEqualTo("This can be any method which returns: 'Hello world'");
        // end::shouldInvokeMap[]
    }

    @Test
    public void shouldThrowCircuitBreakerOpenException() {
        // tag::shouldThrowCircuitBreakerOpenException[]
        CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
            .slidingWindowSize(2)
            .waitDurationInOpenState(Duration.ofMillis(1000))
            .build();
        CircuitBreaker circuitBreaker = CircuitBreaker.of("testName", circuitBreakerConfig);
        // Simulate a failure attempt
        circuitBreaker.onError(0, TimeUnit.NANOSECONDS, new RuntimeException());
        // CircuitBreaker is still CLOSED, because 1 failure is allowed
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        // Simulate a failure attempt
        circuitBreaker.onError(0, TimeUnit.NANOSECONDS, new RuntimeException());
        // CircuitBreaker is OPEN, because the failure rate is above 50%
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);

        // When I decorate my function and invoke the decorated function
        Try<String> result = Try.of(circuitBreaker.decorateCheckedSupplier(() -> "Hello"))
            .map(value -> value + " world");

        // Then the call fails, because CircuitBreaker is OPEN
        assertThat(result.isFailure()).isTrue();
        // Exception is CallNotPermittedException
        assertThat(result.failed().get()).isInstanceOf(CallNotPermittedException.class);
        // end::shouldThrowCircuitBreakerOpenException[]
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(2);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(2);
    }

    @Test
    public void shouldInvokeAsyncApply() throws ExecutionException, InterruptedException {
        // tag::shouldInvokeAsyncApply[]
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        Supplier<String> decoratedSupplier = circuitBreaker
            .decorateSupplier(() -> "This can be any method which returns: 'Hello");

        CompletableFuture<String> future = CompletableFuture.supplyAsync(decoratedSupplier)
            .thenApply(value -> value + " world'");

        assertThat(future.get()).isEqualTo("This can be any method which returns: 'Hello world'");
        // end::shouldInvokeAsyncApply[]
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
    }

    @Test
    public void shouldDecorateCompletionStageAndReturnWithSuccess()
        throws ExecutionException, InterruptedException {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        given(helloWorldService.returnHelloWorld()).willReturn("Hello");
        Supplier<CompletionStage<String>> completionStageSupplier =
            () -> CompletableFuture.supplyAsync(helloWorldService::returnHelloWorld);
        Supplier<CompletionStage<String>> decoratedCompletionStageSupplier =
            circuitBreaker.decorateCompletionStage(completionStageSupplier);

        CompletionStage<String> decoratedCompletionStage = decoratedCompletionStageSupplier
            .get()
            .thenApply(value -> value + " world");

        assertThat(decoratedCompletionStage.toCompletableFuture().get()).isEqualTo("Hello world");
        then(helloWorldService).should().returnHelloWorld();
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
    }


    @Test
    public void shouldExecuteCompletionStageAndReturnWithSuccess()
        throws ExecutionException, InterruptedException {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        given(helloWorldService.returnHelloWorld()).willReturn("Hello");

        CompletionStage<String> decoratedCompletionStage = circuitBreaker
            .executeCompletionStage(
                () -> CompletableFuture.supplyAsync(helloWorldService::returnHelloWorld))
            .thenApply(value -> value + " world");

        assertThat(decoratedCompletionStage.toCompletableFuture().get()).isEqualTo("Hello world");
        then(helloWorldService).should().returnHelloWorld();
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
    }

    @Test
    public void shouldExecuteVoidCompletionStageAndReturnWithSuccess()
        throws ExecutionException, InterruptedException {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        CompletionStage<Void> decoratedCompletionStage = circuitBreaker
            .executeCompletionStage(
                () -> CompletableFuture.runAsync(helloWorldService::sayHelloWorld));

        decoratedCompletionStage.toCompletableFuture().get();

        then(helloWorldService).should().sayHelloWorld();
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
    }

    @Test
    public void shouldDecorateCompletionStageAndReturnWithExceptionAtSyncStage() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        Supplier<CompletionStage<String>> completionStageSupplier = () -> {
            throw new CompletionException(new RuntimeException("BAM! At sync stage"));
        };
        Supplier<CompletionStage<String>> decoratedCompletionStageSupplier =
            circuitBreaker.decorateCompletionStage(completionStageSupplier);

        CompletionStage<String> decoratedCompletionStage = decoratedCompletionStageSupplier.get();

        assertThatThrownBy(decoratedCompletionStage.toCompletableFuture()::get)
            .isInstanceOf(ExecutionException.class)
            .hasCause(new RuntimeException("BAM! At sync stage"));
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
    }

    @Test
    public void shouldDecorateCompletionStageAndReturnWithExceptionAtAsyncStage() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        given(helloWorldService.returnHelloWorld())
            .willThrow(new RuntimeException("BAM! At async stage"));
        Supplier<CompletionStage<String>> completionStageSupplier =
            () -> CompletableFuture.supplyAsync(helloWorldService::returnHelloWorld);
        Supplier<CompletionStage<String>> decoratedCompletionStageSupplier =
            circuitBreaker.decorateCompletionStage(completionStageSupplier);

        CompletionStage<String> decoratedCompletionStage = decoratedCompletionStageSupplier.get();

        assertThatThrownBy(decoratedCompletionStage.toCompletableFuture()::get)
            .isInstanceOf(ExecutionException.class)
            .hasCause(new RuntimeException("BAM! At async stage"));
        then(helloWorldService).should().returnHelloWorld();
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
    }

    @Test
    public void shouldDecorateCompletionStageAndIgnoreHelloWorldException() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .ignoreExceptions(HelloWorldException.class)
            .build();
        CircuitBreaker circuitBreaker = CircuitBreaker.of("backendName", config);
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        given(helloWorldService.returnHelloWorld()).willThrow(new HelloWorldException());
        Supplier<CompletionStage<String>> completionStageSupplier =
            () -> CompletableFuture.supplyAsync(helloWorldService::returnHelloWorld);

        CompletionStage<String> stringCompletionStage = circuitBreaker
            .executeCompletionStage(completionStageSupplier);

        assertThatThrownBy(stringCompletionStage.toCompletableFuture()::get)
            .isInstanceOf(ExecutionException.class).hasCause(new HelloWorldException());
        then(helloWorldService).should().returnHelloWorld();
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
    }

    @Test
    public void shouldChainDecoratedFunctions() {
        // tag::shouldChainDecoratedFunctions[]
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker anotherCircuitBreaker = CircuitBreaker.ofDefaults("anotherTestName");
        // When I create a Supplier and a Function which are decorated by different CircuitBreakers
        CheckedFunction0<String> decoratedSupplier = CircuitBreaker
            .decorateCheckedSupplier(circuitBreaker, () -> "Hello");
        CheckedFunction1<String, String> decoratedFunction = CircuitBreaker
            .decorateCheckedFunction(anotherCircuitBreaker, (input) -> input + " world");

        // and I chain a function with map
        Try<String> result = Try.of(decoratedSupplier)
            .mapTry(decoratedFunction);

        assertThat(result.isSuccess()).isTrue();
        assertThat(result.get()).isEqualTo("Hello world");
        // end::shouldChainDecoratedFunctions[]
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        CircuitBreaker.Metrics metrics2 = anotherCircuitBreaker.getMetrics();
        assertThat(metrics2.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics2.getNumberOfFailedCalls()).isEqualTo(0);
    }

    @Test
    public void testCreateWithNullConfig() {
        assertThatThrownBy(() -> CircuitBreaker.of("test", (CircuitBreakerConfig) null))
            .isInstanceOf(NullPointerException.class).hasMessage("Config must not be null");
    }

    @Test
    public void shouldNotMeasureErrorsAsFailures() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        given(helloWorldService.returnHelloWorld()).willThrow(new StackOverflowError("BAM!"));
        Supplier<String> supplier = circuitBreaker
            .decorateSupplier(helloWorldService::returnHelloWorld);

        assertThatThrownBy(supplier::get).isInstanceOf(StackOverflowError.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorld();

    }

    @Test
    public void shouldExecuteTrySupplierAndReturnWithSuccess() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnTry()).willReturn(Try.success("Hello world"));

        Try<String> result = circuitBreaker.executeTrySupplier(helloWorldService::returnTry);

        assertThat(result).contains("Hello world");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnTry();
    }

    @Test
    public void shouldExecuteEitherSupplierAndReturnWithSuccess() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnEither()).willReturn(Either.right("Hello world"));

        Either<Exception, String> result = circuitBreaker
            .executeEitherSupplier(helloWorldService::returnEither);

        assertThat(result).contains("Hello world");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnEither();
    }

    @Test
    public void shouldExecuteTrySupplierAndReturnWithFailure() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnTry()).willReturn(Try.failure(new RuntimeException("BAM!")));

        Try<String> result = circuitBreaker.executeTrySupplier(helloWorldService::returnTry);

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnTry();
    }

    @Test
    public void shouldExecuteTrySupplierAndReturnWithCallNotPermittedException() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        circuitBreaker.transitionToOpenState();

        Try<String> result = circuitBreaker.executeTrySupplier(helloWorldService::returnTry);

        assertThat(result.isFailure()).isTrue();
        assertThat(result.failed().get()).isInstanceOf(CallNotPermittedException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfNotPermittedCalls()).isEqualTo(1);
        then(helloWorldService).should(never()).returnTry();
    }


    @Test
    public void shouldExecuteEitherSupplierAndReturnWithFailure() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        given(helloWorldService.returnEither()).willReturn(Either.left(new HelloWorldException()));

        Either<Exception, String> result = circuitBreaker
            .executeEitherSupplier(helloWorldService::returnEither);

        assertThat(result.isLeft()).isTrue();
        assertThat(result.getLeft()).isInstanceOf(RuntimeException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnEither();
    }

    @Test
    public void shouldExecuteEitherSupplierAndReturnWithCallNotPermittedException() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        circuitBreaker.transitionToOpenState();

        Either<Exception, String> result = circuitBreaker
            .executeEitherSupplier(helloWorldService::returnEither);

        assertThat(result.isLeft()).isTrue();
        assertThat(result.getLeft()).isInstanceOf(CallNotPermittedException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfNotPermittedCalls()).isEqualTo(1);
        then(helloWorldService).should(never()).returnEither();
    }

    @Test
    public void shouldDecorateFutureSupplierAndReturnSuccess() throws Exception {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        Future<String> future = mock(Future.class);
        given(future.get()).willReturn("Hello World");
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        Supplier<Future<String>> supplier = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        String value = supplier.get().get();

        assertThat(value).isEqualTo("Hello World");
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should().get();
    }

    @Test
    public void shouldDecorateFutureSupplierAndCBLogicEvalOnlyOnceSuccess() throws Exception {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        Future<String> future = mock(Future.class);
        given(future.get()).willReturn("Hello World");
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        Supplier<Future<String>> supplier = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        //fetching value multiple time should evaluate CB logic only once
        Future<String> decoratedFuture = supplier.get();
        decoratedFuture.get();
        decoratedFuture.get();
        decoratedFuture.get();

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(1);
        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should(times(3)).get();
    }

    @Test
    public void shouldDecorateFutureSupplierAndCBLogicEvalOnlyOnceWithException() throws Exception {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        //create a Future
        Future<String> future = mock(Future.class);
        given(future.get()).willThrow(new ExecutionException(new RuntimeException("BAM!")));
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        Supplier<Future<String>> supplier = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        //fetching value multiple time should evaluate CB logic only once
        Future<String> decoratedFuture = supplier.get();

        //evaluate future three times
        Throwable thrown = catchThrowable(() -> decoratedFuture.get());
        catchThrowable(() -> decoratedFuture.get());
        catchThrowable(() -> decoratedFuture.get());

        assertThat(thrown).isInstanceOf(ExecutionException.class)
            .hasCauseInstanceOf(RuntimeException.class);
        assertThat(thrown.getCause().getMessage()).isEqualTo("BAM!");

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should(times(3)).get();
    }

    @Test
    public void shouldDecorateFutureSupplierAndReturnWithExceptionEvenBeforeFutureIsCreated() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        given(helloWorldService.returnHelloWorldFuture()).willThrow(new RuntimeException("BAM!"));

        Supplier<Future<String>> supplier = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        Throwable thrown = catchThrowable(() -> supplier.get());

        assertThat(thrown).isInstanceOf(RuntimeException.class)
            .hasMessage("BAM!");

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorldFuture();
    }

    @Test
    public void shouldDecorateFutureSupplierAndFutureReturnException() throws Exception {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        //create a Future
        Future<String> future = mock(Future.class);
        given(future.get()).willThrow(new ExecutionException(new RuntimeException("BAM!")));
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        Supplier<Future<String>> supplier = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        Throwable thrown = catchThrowable(() -> supplier.get().get());

        assertThat(thrown).isInstanceOf(ExecutionException.class)
            .hasCauseInstanceOf(RuntimeException.class);
        assertThat(thrown.getCause().getMessage()).isEqualTo("BAM!");

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should().get();
    }

    @Test
    public void shouldDecorateFutureWithSupplierAndCallCancel() throws Exception {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        Future<String> future = mock(Future.class);
        given(future.get()).willThrow(new CancellationException());
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        Supplier<Future<String>> supplier = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        Throwable thrown = catchThrowable(() -> supplier.get().get());

        assertThat(thrown).isInstanceOf(CancellationException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should().get();
    }

    @Test
    public void shouldDecorateFutureSupplierAndInterruptByTaskThread() throws Exception {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        Future<String> future = mock(Future.class);
        given(future.get()).willThrow(new ExecutionException(new InterruptedException()));
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        // If interrupt is called on the Task thread than InterruptedException is thrown wrapped in
        // ExecutionException where as if current thread gets interrupted it throws
        // InterruptedException directly.
        Supplier<Future<String>> supplier = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        Throwable thrown = catchThrowable(() -> supplier.get().get());

        assertThat(thrown).isInstanceOf(ExecutionException.class);
        assertThat(thrown).hasCauseInstanceOf(InterruptedException.class);

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should().get();
    }

    @Test
    public void shouldDecorateFutureSupplierAndInterruptedByCallingThread() throws Exception {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        Future<String> future = mock(Future.class);
        given(future.get()).willThrow(new InterruptedException());
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        Supplier<Future<String>> decoratedFuture = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        Throwable thrown = catchThrowable(() -> decoratedFuture.get().get());

        assertThat(thrown).isInstanceOf(InterruptedException.class);
        assertThat(circuitBreaker.getMetrics().getNumberOfBufferedCalls()).isEqualTo(0);
        assertThat(circuitBreaker.getMetrics().getNumberOfFailedCalls()).isEqualTo(0);
        assertThat(circuitBreaker.getMetrics().getNumberOfSuccessfulCalls()).isEqualTo(0);
        assertThat(circuitBreaker.getMetrics().getNumberOfNotPermittedCalls()).isEqualTo(0);
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should().get();
    }

    @Test
    public void shouldDecorateFutureSupplierTimeout() throws Exception {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);

        Future<String> future = mock(Future.class);
        given(future.get(anyLong(), any(TimeUnit.class))).willThrow(new TimeoutException());
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        //Decorating future likely to throw timeout exception
        Supplier<Future<String>> supplier = circuitBreaker.decorateFuture(
            helloWorldService::returnHelloWorldFuture);

        Throwable thrown = catchThrowable(() -> supplier.get().get(5, TimeUnit.SECONDS));

        assertThat(thrown).isInstanceOf(TimeoutException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);
        assertThat(metrics.getNumberOfSuccessfulCalls()).isEqualTo(0);
        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should().get(anyLong(), any(TimeUnit.class));
    }

    @Test
    public void shouldReturnFailureWithCircuitBreakerOpenExceptionWithFutures() throws Exception {
        // Given
        CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
            .slidingWindowSize(2)
            .permittedNumberOfCallsInHalfOpenState(2)
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofMillis(1000))
            .build();
        CircuitBreaker circuitBreaker = CircuitBreaker.of("testName", circuitBreakerConfig);

        Future<String> future = mock(Future.class);
        given(future.get()).willReturn("Hello World");
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);

        Supplier<Future<String>> futureSupplier = circuitBreaker
            .decorateFuture(helloWorldService::returnHelloWorldFuture);

        // When
        circuitBreaker.onError(0, TimeUnit.NANOSECONDS, new RuntimeException());
        circuitBreaker.onError(0, TimeUnit.NANOSECONDS, new RuntimeException());

        // Then
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(2);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(2);

        // When
        Throwable thrown = catchThrowable(() -> futureSupplier.get().get());

        // Then
        assertThat(thrown)
            .isInstanceOf(ExecutionException.class)
            .hasCauseInstanceOf(CallNotPermittedException.class);
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(2);
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(2);
        assertThat(metrics.getNumberOfNotPermittedCalls()).isEqualTo(1);

        then(helloWorldService).shouldHaveZeroInteractions();
        then(future).shouldHaveZeroInteractions();
    }

    @Test
    public void shouldNotRecordIOExceptionAsAFailureWithFuture() throws Exception {
        // tag::shouldNotRecordIOExceptionAsAFailure[]
        CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
            .slidingWindowSize(2)
            .permittedNumberOfCallsInHalfOpenState(2)
            .waitDurationInOpenState(Duration.ofMillis(1000))
            .ignoreExceptions(IOException.class)
            .build();
        CircuitBreaker circuitBreaker = CircuitBreaker.of("testName", circuitBreakerConfig);
        // Simulate a failure attempt
        circuitBreaker.onError(0, TimeUnit.NANOSECONDS, new HelloWorldException());
        // CircuitBreaker is still CLOSED, because 1 failure is allowed
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);

        Future<String> future = mock(Future.class);
        given(future.get()).willThrow(new ExecutionException(new SocketTimeoutException("BAM!")));
        given(helloWorldService.returnHelloWorldFuture()).willReturn(future);
        Supplier<Future<String>> futureSupplier = circuitBreaker
            .decorateFuture(helloWorldService::returnHelloWorldFuture);

        // When
        Throwable thrown = catchThrowable(() -> futureSupplier.get().get());

        //Then
        // CircuitBreaker is still CLOSED, because SocketTimeoutException has not been recorded as a failure
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        // end::shouldNotRecordIOExceptionAsAFailure[]
        assertThat(thrown).isInstanceOf(ExecutionException.class);
        assertThat(thrown).hasCauseInstanceOf(IOException.class);

        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(1);
        //this failure is because of HelloWorldException thrown earlier
        assertThat(metrics.getNumberOfFailedCalls()).isEqualTo(1);

        then(helloWorldService).should().returnHelloWorldFuture();
        then(future).should().get();
    }

    @Test
    public void shouldDecorateFutureAndReturnWithCallNotPermittedException() {

        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("testName");
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        circuitBreaker.transitionToOpenState();

        //If interrupt is called on the Task thread than InterruptedException is thrown wrapped in
        // ExecutionException where as if current thread gets interrupted it throws
        // InterruptedException directly.
        Supplier<Future<String>> futureSupplier = circuitBreaker
            .decorateFuture(helloWorldService::returnHelloWorldFuture);

        Throwable thrown = catchThrowable(() -> futureSupplier.get().get());
        assertThat(thrown)
            .isInstanceOf(ExecutionException.class)
            .hasCauseInstanceOf(CallNotPermittedException.class);

        assertThat(metrics.getNumberOfBufferedCalls()).isEqualTo(0);
        assertThat(metrics.getNumberOfNotPermittedCalls()).isEqualTo(1);
    }
}
```
### (6). 总结
> 在这里先把单元测试案例,跑起来,后面会对:熔断的配置信息进行详解.   