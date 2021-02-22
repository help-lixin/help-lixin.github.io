---
layout: post
title: 'Fluent-Validator Validator接口(二)'
date: 2017-06-04
author: 李新
tags: fluent-validator
---

### (1). 概述
> 前面做了一个小小的案例,我们需要自定义验证规则类,这个类需要实现:Validator接口.在这里,主要剖析:Validator.  

### (2). Validator类结构图
!["Validator类结构图"](/assets/fluent-validator/imgs/Validator.jpg)

### (3). Validator
```
package com.baidu.unbiz.fluentvalidator;

/**
 * 验证器接口。
 * <p/>
 * 泛型<t>T</t>表示待验证对象的类型
 *
 * @author zhangxu
 */
public interface Validator<T> {

    /**
     * 判断在该对象上是否接受或者需要验证
     * <p/>
     * 如果返回true，那么则调用{@link #validate(ValidatorContext, Object)}，否则跳过该验证器
     *
     * @param context 验证上下文
     * @param t       待验证对象
     *
     * @return 是否接受验证
     */
    boolean accept(ValidatorContext context, T t);

    /**
     * 执行验证
     * <p/>
     * 如果发生错误内部需要调用{@link ValidatorContext#addErrorMsg(String)}方法，也即<code>context.addErrorMsg(String)
     * </code>来添加错误，该错误会被添加到结果存根{@link Result}的错误消息列表中。
     *
     * @param context 验证上下文
     * @param t       待验证对象
     *
     * @return 是否验证通过
     */
    boolean validate(ValidatorContext context, T t);

    /**
     * 异常回调
     * <p/>
     * 当执行{@link #accept(ValidatorContext, Object)}或者{@link #validate(ValidatorContext, Object)}发生异常时的如何处理
     *
     * @param e       异常
     * @param context 验证上下文
     * @param t       待验证对象
     */
    void onException(Exception e, ValidatorContext context, T t);

}
```
### (4). Composable
```
package com.baidu.unbiz.fluentvalidator;

/**
 * 在Validator中添加额外的验证逻辑，用组合的方式
 *
 * @author zhangxu
 */
public interface Composable<T> {

    /**
     * 切入点，可以织入一些校验逻辑
     *
     * @param current 当前的FluentValidator实例
     * @param context 验证器执行调用中的上下文
     * @param t       待验证的对象
     */
    void compose(FluentValidator current, ValidatorContext context, T t);
}
```
### (5). ValidatorHandler
```
package com.baidu.unbiz.fluentvalidator;

import com.baidu.unbiz.fluentvalidator.annotation.ThreadSafe;

/**
 * 验证器默认实现
 * <p/>
 * 自定义的验证器如果不想实现{@link Validator}所有方法，可以使用这个默认实现，仅覆盖自己需要实现的方法
 *
 * @author zhangxu
 * @see Validator
 */
@ThreadSafe
public class ValidatorHandler<T> implements Validator<T>, Composable<T> {

    @Override
    public boolean accept(ValidatorContext context, T t) {
        return true;
    }

    @Override
    public boolean validate(ValidatorContext context, T t) {
        return true;
    }

    @Override
    public void onException(Exception e, ValidatorContext context, T t) {

    }

    @Override
    public void compose(FluentValidator current, ValidatorContext context, T t) {
        // extension point for clients to add more validators to the current fluent chain
    }

    /**
     * 验证器的名字，用简单类名称表示
     *
     * @return 名字
     */
    @Override
    public String toString() {
        return this.getClass().getSimpleName();
    }

}
```
### (6). 总结
> Validator是开发人员应该要实现的验证规则接口.这也就理解了为什么要实现:Validator还要继承:ValidatorHandler.
> 继承ValidatorHandler是为了让开发根据需要,重写相应的实现方法即可.   