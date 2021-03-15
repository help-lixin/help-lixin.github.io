---
layout: post
title: 'DDD 值对象(五)'
date: 2020-06-01
author: 李新
tags: DDD
---

### (1). 引子
> DDD的战略设计知识,来源于:极客时间<DDD实战课>和(https://www.cnblogs.com/sheng-jie/p/6931646.html)            
> DDD的战术设计知识来源于:https://github.com/citerus/dddsample-core.git    
### (2). DDD中的值对象
> 看一下<实现领域驱动设计>一书中对值对象的定义:  
> 通过对象属性值来识别的对象,它将多个相关属性组合为一个概念整体.在DDD中用来描述领域的特定方面,并且是一个没有标识符的对象,叫作值对象.   
> 在领域建模的过程中,值对象可以保证属性归类的清晰和概念的完整性,避免属性零碎.  
> 值对象是若干个属性的集合,只有数据初始化操作和有限的不涉及修改数据的行为,基本不包含业务逻辑.值对象的属性集虽然在物理上独立出来了,但在逻辑上它仍然是实体属性的一部分,用于描述实体的特征.  
### (3). 案例分析
> 人员实体原本包括:姓名、年龄、性别以及人员所在的省、市、县和街道等属性.  
> 这样显示地址相关的属性就很零碎了对不对?现在,我们可以将“省、市、县和街道等属性",拿出来构成一个"地址属性集合",这个集合就是值对象了.

!["Value Object"](https://static001.geekbang.org/resource/image/13/f6/136512ac4c65b3f2ed4b2898b40965f6.jpg)
### (4). Address值对象
```

public class Person implements Serializable {
	private static final long serialVersionUID = 5588448113938976398L;
	private String id;
	private String name;
	private int age;
	// 地址
	private Address address;

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public Address getAddress() {
		return address;
	}

	public void setAddress(Address address) {
		this.address = address;
	}
} // end person


public class Address implements Serializable {
	private static final long serialVersionUID = 422726171482528378L;
	private String province;
	private String city;
	private String county;
	private String street;
	
	// 注意:值对象,一旦创建后就不允许对其修改,若要修改:整个值对象替换
	// 所以:只生成了get方法
	public Address(String province, String city, String county, String street) {
		this.province = province;
		this.city = city;
		this.county = county;
		this.street = street;
	}

	public String getProvince() {
		return province;
	}

	public String getCity() {
		return city;
	}

	public String getCounty() {
		return county;
	}

	public String getStreet() {
		return street;
	}
	
	// get/set/toString/equals/hashCode...
}//end address
```
### (5). 值对象的问题
> 值对象该如何存储呢?  
> 1. (单个值对象),值对象不会孤立存在,所以我们可以将值对象中的属性作为所属实体/聚合根的数据列来存储(比如:我们可以将收货地址的属性映射到客户实体中).这样做就会导致数据表列数增多,但是能够优化查询性能,因为不需要联表查询.  
> 2. (多个值对像序列化到单个列),若一个客户有多个收货地址,这个时候,我们可以把多个地址序列化成大对象的模式,把一个集合序列化后塞到外层实体表的某一列中,是有点匪夷所思.而且数据库的列宽是有限制的,且不方便查询.但似乎也带来一个好处,大大简化了系统的设计.  
> 3. (使用数据库实体保存多个值对像),为每个值对象委派唯一标识,以数据库实体形式保存值对象.  

### (6). 总结
> 值对象 = 值 + 对象 = 将一个值用对象的方式进行表述,来表达一个具体的固定不变的概念.