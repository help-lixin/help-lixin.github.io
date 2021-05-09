---
layout: post
title: 'Servicecomb Pack之微服务下分布式事务案例(四)'
date: 2021-05-04
author: 李新
tags:  Servicecomb-Pack
---

### (1). 前言

> 业务知识请参考该文章["ServiceComb中的数据最终一致性方案 - part 1"](http://servicecomb.apache.org/cn/docs/distributed_saga_1/)  

### (2). 租车服务(car)
```
// 1. 租车实体对象
package help.lixin.saga.example;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.JsonAutoDetect.Visibility;
import com.fasterxml.jackson.annotation.JsonIgnore;

@JsonAutoDetect(fieldVisibility = Visibility.ANY)
class CarBooking {
  private Integer id;
  private String name;
  private Integer amount;
  private boolean confirmed;
  private boolean cancelled;

  Integer getId() {
    return id;
  }

  void setId(Integer id) {
    this.id = id;
  }

  String getName() {
    return name;
  }

  void setName(String name) {
    this.name = name;
  }

  Integer getAmount() {
    return amount;
  }

  void setAmount(Integer amount) {
    this.amount = amount;
  }

  boolean isConfirmed() {
    return confirmed;
  }

  void confirm() {
    this.confirmed = true;
    this.cancelled = false;
  }

  boolean isCancelled() {
    return cancelled;
  }

  void cancel() {
    this.confirmed = false;
    this.cancelled = true;
  }
}


// 2. CarBookingController
package help.lixin.saga.example;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.CrossOrigin;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

@RestController
public class CarBookingController {
  @Autowired
  private CarBookingService service;

  private final AtomicInteger id = new AtomicInteger(0);

  @CrossOrigin
  @GetMapping("/bookings") List<CarBooking> getAll() {
    return new ArrayList<>(service.getAllBookings());
  }

  @PostMapping("/order/{name}/{cars}")
  CarBooking order(@PathVariable String name, @PathVariable Integer cars) {
    CarBooking booking = new CarBooking();
    booking.setId(id.incrementAndGet());
    booking.setName(name);
    booking.setAmount(cars);
    service.order(booking);
    return booking;
  }

  @DeleteMapping("/bookings")
  void clear() {
    service.clearAllBookings();
    id.set(0);
  }
}


// 3. CarBookingService
package help.lixin.saga.example;

import org.apache.servicecomb.pack.omega.transaction.annotations.Compensable;
import org.springframework.stereotype.Service;

import java.util.Collection;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
class CarBookingService {
	private Map<Integer, CarBooking> bookings = new ConcurrentHashMap<>();

    // *************************************************************************
	// 分支事务,需要使用注解@Compensable,并指定补偿方法.
	// 注意:canal的方法签名要与参与事务的方法签名一样的.
	// *************************************************************************
	@Compensable(compensationMethod = "cancel")
	void order(CarBooking booking) {
		if (booking.getAmount() > 10) {
			throw new IllegalArgumentException("can not order the cars large than ten");
		}
		booking.confirm();
		bookings.put(booking.getId(), booking);
	}

	void cancel(CarBooking booking) {
		Integer id = booking.getId();
		if (bookings.containsKey(id)) {
			bookings.get(id).cancel();
		}
		// Just sleep a while to ensure the Compensated event is after ordering TxAbort
		// event
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			// Just ignore the exception
		}
	}

	Collection<CarBooking> getAllBookings() {
		return bookings.values();
	}

	void clearAllBookings() {
		bookings.clear();
	}
}


// 4. application.properties
spring.application.name=car
server.port=8082
alpha.cluster.address=localhost:8080
```
### (3). 酒店服务(hotel)
```
// 1. HotelBooking实体
package help.lixin.saga.example;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.JsonAutoDetect.Visibility;
import com.fasterxml.jackson.annotation.JsonIgnore;

@JsonAutoDetect(fieldVisibility = Visibility.ANY)
class HotelBooking {
  private Integer id;
  private String name;
  private Integer amount;
  private boolean confirmed;
  private boolean cancelled;

  Integer getId() {
    return id;
  }

  void setId(Integer id) {
    this.id = id;
  }

  String getName() {
    return name;
  }

  void setName(String name) {
    this.name = name;
  }

  Integer getAmount() {
    return amount;
  }

  void setAmount(Integer amount) {
    this.amount = amount;
  }

  boolean isConfirmed() {
    return confirmed;
  }

  void confirm() {
    this.confirmed = true;
    this.cancelled = false;
  }

  boolean isCancelled() {
    return cancelled;
  }

  void cancel() {
    this.confirmed = false;
    this.cancelled = true;
  }
}


// 2. HotelBookingController
package help.lixin.saga.example;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.CrossOrigin;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

@RestController
public class HotelBookingController {
  @Autowired
  private HotelBookingService service;

  private final AtomicInteger id = new AtomicInteger(0);

  @CrossOrigin
  @GetMapping("/bookings")
  List<HotelBooking> getAll() {
    return new ArrayList<>(service.getAllBookings());
  }

  @PostMapping("/order/{name}/{rooms}")
  HotelBooking order(@PathVariable String name, @PathVariable Integer rooms) {
    HotelBooking booking = new HotelBooking();
    booking.setId(id.incrementAndGet());
    booking.setName(name);
    booking.setAmount(rooms);
    service.order(booking);
    return booking;
  }

  @DeleteMapping("/bookings")
  void clear() {
    service.clearAllBookings();
    id.set(0);
  }
}


// 3. HotelBookingService
package help.lixin.saga.example;

import org.apache.servicecomb.pack.omega.transaction.annotations.Compensable;
import org.springframework.stereotype.Service;

import java.util.Collection;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
class HotelBookingService {
  private Map<Integer, HotelBooking> bookings = new ConcurrentHashMap<>();

  // *************************************************************************
  // 分支事务,需要使用注解@Compensable,并指定补偿方法.
  // 注意:canal的方法签名要与参与事务的方法签名一样的.
  // *************************************************************************
  @Compensable(compensationMethod = "cancel")
  void order(HotelBooking booking) {
    if (booking.getAmount() > 2) {
      throw new IllegalArgumentException("can not order the rooms large than two");
    }
    booking.confirm();
    bookings.put(booking.getId(), booking);
  }

  void cancel(HotelBooking booking) {
    Integer id = booking.getId();
    if (bookings.containsKey(id)) {
      bookings.get(id).cancel();
    }
  }

  Collection<HotelBooking> getAllBookings() {
    return bookings.values();
  }

  void clearAllBookings() {
    bookings.clear();
  }
}


// 4. application.properties
spring.application.name=hotel
server.port=8081
alpha.cluster.address=localhost:8080
```
### (4). 预订服务(booking)
```
// 1. BookingController
package help.lixin.saga.example;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.servicecomb.pack.omega.context.annotations.SagaStart;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import java.lang.invoke.MethodHandles;
import java.util.Arrays;
import java.util.List;
import java.util.Map;

@RestController
public class BookingController {

  private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());

  @Value("${car.service.address:http://car.servicecomb.io:8080}")
  private String carServiceUrl;

  @Value("${hotel.service.address:http://hotel.servicecomb.io:8080}")
  private String hotelServiceUrl;

  @Autowired
  private RestTemplate template;

  // ***************************************************************
  // 定义全局事务@SagaStart
  // ***************************************************************
  @SagaStart
  @PostMapping("/booking/{name}/{rooms}/{cars}")
  public String order( // 用户信息
		              @PathVariable String name, 
		              // 预计包房数量
		              @PathVariable Integer rooms,
		              // 租车数量
                      @PathVariable Integer cars) throws Throwable {

    if (cars < 0) {
      throw new Exception("The cars order quantity must be greater than 0");
    }

    // 租车
    template.postForEntity(
        carServiceUrl + "/order/{name}/{cars}",
        null, String.class, name, cars);

    postCarBooking();

    if (rooms < 0) {
      throw new Exception("The rooms order quantity must be greater than 0");
    }

    // 订酒店
    template.postForEntity(
        hotelServiceUrl + "/order/{name}/{rooms}",
        null, String.class, name, rooms);

    postBooking();

    return name + " booking " + rooms + " rooms and " + cars + " cars OK";
  }

  // This method is used by the byteman to inject exception here
  private void postCarBooking() throws Throwable {

  }

  // This method is used by the byteman to inject the faults such as the timeout or the crash
  private void postBooking() throws Throwable {

  }

  // This method is used by the byteman trigger shutdown the master node in the Alpha server cluster
  private void alphaMasterShutdown() {
    String alphaRestAddress = System.getenv("alpha.rest.address");
    LOG.info("alpha.rest.address={}", alphaRestAddress);
    List<String> addresss = Arrays.asList(alphaRestAddress.split(","));

    addresss.stream().filter(address -> {
      // use the actuator alpha endpoint to find the alpha master node
      try {
        ResponseEntity<String> responseEntity = template
            .getForEntity(address + "/actuator/alpha", String.class);
        ObjectMapper mapper = new ObjectMapper();
        if (responseEntity.getStatusCode() == HttpStatus.OK) {
          String json = responseEntity.getBody();
          Map<String, String> map = mapper.readValue(json, Map.class);
          if (map.get("nodeType").equalsIgnoreCase("MASTER")) {
            return true;
          }
        }
      } catch (Exception ex) {
        LOG.error("", ex);
      }
      return false;
    }).forEach(address -> {
      // call shutdown endpoint to shutdown the alpha master node
      HttpHeaders headers = new HttpHeaders();
      headers.setContentType(MediaType.APPLICATION_JSON);
      HttpEntity request = new HttpEntity(headers);
      ResponseEntity<String> responseEntity = template
          .postForEntity(address + "/actuator/shutdown", request, String.class);
      if (responseEntity.getStatusCode() == HttpStatus.OK) {
        LOG.info("Alpah master node {} shutdown", address);
      }
    });
  }
}


// 2. application.properties
spring.application.name=booking
server.port=8083
alpha.cluster.address=localhost:8080
hotel.service.address=http://localhost:8081
car.service.address=http://localhost:8082
```
### (5). pom.xml
```
// 1. parent(pom.xml)定义公共的依赖
<properties>
	<maven.compiler.source>8</maven.compiler.source>
	<maven.compiler.target>8</maven.compiler.target>
	<servicecomb.version>0.6.0</servicecomb.version>
	<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

<modules>
	<module>car</module>
	<module>hotel</module>
	<module>booking</module>
</modules>

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
		<dependency>
			<groupId>org.apache.servicecomb.pack</groupId>
			<artifactId>omega-spring-starter</artifactId>
			<version>${servicecomb.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.servicecomb.pack</groupId>
			<artifactId>omega-transport-resttemplate</artifactId>
			<version>${servicecomb.version}</version>
		</dependency>
	</dependencies>
</dependencyManagement>

// car/hotel/booking 定义依赖.
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.apache.servicecomb.pack</groupId>
		<artifactId>omega-spring-starter</artifactId>
	</dependency>
	<dependency>
		<groupId>org.apache.servicecomb.pack</groupId>
		<artifactId>omega-transport-resttemplate</artifactId>
	</dependency>
</dependencies>
```
### (6). 总结
> 1. 在要开启分布式事务的最外层,只要加入一个注解(@SagaStart)即可.  
> 2. 在分支事务里,只需加入一个注解(@Compensable(compensationMethod = "xxx")),并配置补偿的方法名称(方法签名要事务的方法签名保持一致).   