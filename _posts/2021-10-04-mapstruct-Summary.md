---
layout: post
title: 'mapstruct快速使用' 
date: 2021-10-04
author: 李新
tags:  mapstruct
---

### (1). MapStruct介绍
MapStruct是一个属性映射工具,只需要定义一个Mapper接口,MapStruct就会自动实现这个映射接口,避免了复杂繁琐的映射实现.
在一个JavaWeb工程中会涉及到多种对象,po、vo、dto、entity、do、domain这些定义的对象运用在不同的场景模块中,这种对象与对象之间的互相转换,就需要有一个专门用来解决转换问题的工具.以前是通过反射的方法实现,但是,使用反射的时候都会影响到性能,再后来自己写装换器但是会很浪费时间,而且在添加新的字段的时候也要进行方法的修改.MapSturct是一个生成类型安全,高性能且无依赖的JavaBean映射代码的注解处理器.作为一个工具类,相比于手写,具有便捷,不容易出错的特点. 

### (2). MapStruct的使用
```
 <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <org.mapstruct.version>1.5.0.Beta1</org.mapstruct.version>
</properties>

<dependencies>
	<dependency>
		<groupId>org.mapstruct</groupId>
		<artifactId>mapstruct</artifactId>
		<version>${org.mapstruct.version}</version>
	</dependency>

	<dependency>
		<groupId>org.mapstruct</groupId>
		<artifactId>mapstruct-processor</artifactId>
		<version>${org.mapstruct.version}</version>
		<scope>provided</scope>
	</dependency>
	
</dependencies>

<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>3.8.1</version>
			<configuration>
				<source>1.8</source>
				<target>1.8</target>
				<!-- 指定注解的处理 --> 
				<annotationProcessorPaths>
					<path>
						<groupId>org.mapstruct</groupId>
						<artifactId>mapstruct-processor</artifactId>
						<version>${org.mapstruct.version}</version>
					</path>
				</annotationProcessorPaths>
			</configuration>
		</plugin>
	</plugins>
</build>
```
### (3). 定义DTO
```

package org.mapstruct.example.dto;

import java.util.List;

public class CustomerDto {
    public Long id;
    public String customerName;
    public List<OrderItemDto> orders;
} // end CustomerDto


package org.mapstruct.example.dto;

public class OrderItemDto {

    public String name;
    public Long quantity;
} // end OrderItemDto

```
### (4). 定义DTO
```
package org.mapstruct.example.dto;

import java.util.Collection;

public class Customer {
    private Long id;
    private String name;
    private Collection<OrderItem> orderItems;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Collection<OrderItem> getOrderItems() {
        return orderItems;
    }

    public void setOrderItems(Collection<OrderItem> orderItems) {
        this.orderItems = orderItems;
    }
}// end Customer


package org.mapstruct.example.dto;

public class OrderItem {
    private String name;
    private Long quantity;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Long getQuantity() {
        return quantity;
    }

    public void setQuantity(Long quantity) {
        this.quantity = quantity;
    }
} // end OrderItem
```
### (5). 定义Mapper
```
package org.mapstruct.example.mapper;

import org.mapstruct.InheritInverseConfiguration;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.example.dto.Customer;
import org.mapstruct.example.dto.CustomerDto;
import org.mapstruct.factory.Mappers;

@Mapper(uses = { OrderItemMapper.class })
public interface CustomerMapper {

    CustomerMapper MAPPER = Mappers.getMapper( CustomerMapper.class );

    // 
    @Mapping(source = "orders", target = "orderItems")
    @Mapping(source = "customerName", target = "name")
    Customer toCustomer(CustomerDto customerDto);

    @InheritInverseConfiguration
    CustomerDto fromCustomer(Customer customer);
} // end CustomerMapper


package org.mapstruct.example.mapper;

import org.mapstruct.InheritInverseConfiguration;
import org.mapstruct.Mapper;
import org.mapstruct.example.dto.OrderItem;
import org.mapstruct.example.dto.OrderItemDto;
import org.mapstruct.factory.Mappers;

@Mapper
public interface OrderItemMapper {

    OrderItemMapper MAPPER = Mappers.getMapper(OrderItemMapper.class);

    OrderItem toOrder(OrderItemDto orderItemDto);

    @InheritInverseConfiguration
    OrderItemDto fromOrder(OrderItem orderItem);
} // end OrderItemMapper
```
### (6). CustomerMapperTest
```
public class CustomerMapperTest {

    @Test
    public void testMapDtoToEntity() {

        CustomerDto customerDto = new CustomerDto();
        customerDto.id = 10L;
        customerDto.customerName = "Filip";
        OrderItemDto order1 = new OrderItemDto();
        order1.name = "Table";
        order1.quantity = 2L;
        customerDto.orders = new ArrayList<>( Collections.singleton( order1 ) );

        Customer customer = CustomerMapper.MAPPER.toCustomer( customerDto );

        assertThat( customer.getId() ).isEqualTo( 10 );
        assertThat( customer.getName() ).isEqualTo( "Filip" );
        assertThat( customer.getOrderItems() )
            .extracting( "name", "quantity" )
            .containsExactly( tuple( "Table", 2L ) );
    }
} 	
```
### (7). 总结
优势就是自己不需要一个个的set/get方法的映射了,缺点就是:注解对代码有侵入性.  
