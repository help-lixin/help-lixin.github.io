---
layout: post
title: 'EcmaScript6 定义Class'
date: 2020-04-10
author: 李新
tags: EcmaScript6
---

### (1). ES6之前定义类与继承关系
```
// 声明父类
function Person(name,age){
  this.name = name;
  this.age = age;
}
// 为父类配置方法.
Person.prototype.say=function(){
  console.log(this.name +" -- " + this.age)
}

// 声明子类
function Teacher(name,age,school){
  Person.call(this,name,age);
  this.school = school;
}

// 配置子类继承
Teacher.prototype = new Person();
Teacher.prototype.constructor = Teacher;
// 为子类配置方法
Teacher.prototype.hello=function(){
  console.log(this.name +" -- " + this.age + " -- " + this.school);
}

// 创建子类实例
let zhangsan = new Teacher("zhangsan",28,"shengzheng");
zhangsan.hello();
zhangsan.say();
```
### (2). ES6之后定义类与继承关系
```
// 声明父类
class Person {
  constructor(name,age) {
      this.name = name;
      this.age = age;
  }

  say(){
    console.log(this.name + "  " + this.age);
  }
}

// 声明子类,继承父类
class Teacher extends Person {
  constructor(name,age,school) {
      // 调用父类
      super(name,age);
      this.school = school;
  }
  
  hello(){
    console.log(this.name +" -- " + this.age + " -- " + this.school);
  }
}

// 创建:Teacher实例
let teacher = new Teacher("zhgnsan",28,"shenzheng");
teacher.hello();
teacher.say();
```
