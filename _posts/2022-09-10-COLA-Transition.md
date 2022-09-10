---
layout: post
title: 'COLA源码之Transition(一)' 
date: 2022-09-10
author: 李新
tags:  COLA
---

### (1). 概述
COLA的源码相对来说是比较简单的,所以,我不打算从入口开始,我只会抽几个重要的东西来记录下,其中,在我看来比较重要之一的是:Transition.
### (2). Transition
```
// S -- > State(状态)
// E --> Event(事件)
// C --> Condition(条件)
public interface Transition<S, E, C>{
	// Source状态信息配置
	State<S,E,C> getSource();
	void setSource(State<S, E, C> state);
	
	// 事件信息配置
	E getEvent();
	void setEvent(E event);
	
	void setType(TransitionType type);
	
	// Target状态信息配置
	State<S,E,C> getTarget();
	void setTarget(State<S, E, C> state);
	
	// 条件信息配置
	Condition<C> getCondition();
	void setCondition(Condition<C> condition);
	
	// 动作
	Action<S,E,C> getAction();
	void setAction(Action<S, E, C> action);
	
	// ***********************************************************
	// 事件触发
	// ***********************************************************
	State<S, E, C> transit(C ctx, boolean checkCondition);
	
	// 验证
	void verify();
}	
```
### (3). TransitionImpl
```
public class TransitionImpl<S,E,C> implements Transition<S,E,C> {

    private State<S, E, C> source;

    private State<S, E, C> target;

    private E event;

    private Condition<C> condition;

    private Action<S,E,C> action;

    private TransitionType type = TransitionType.EXTERNAL;

    @Override
    public State<S, E, C> getSource() {
        return source;
    }

    @Override
    public void setSource(State<S, E, C> state) {
        this.source = state;
    }

    @Override
    public E getEvent() {
        return this.event;
    }

    @Override
    public void setEvent(E event) {
        this.event = event;
    }

    @Override
    public void setType(TransitionType type) {
        this.type = type;
    }

    @Override
    public State<S, E, C> getTarget() {
        return this.target;
    }

    @Override
    public void setTarget(State<S, E, C> target) {
        this.target = target;
    }

    @Override
    public Condition<C> getCondition() {
        return this.condition;
    }

    @Override
    public void setCondition(Condition<C> condition) {
        this.condition = condition;
    }

    @Override
    public Action<S, E, C> getAction() {
        return this.action;
    }

    @Override
    public void setAction(Action<S, E, C> action) {
        this.action = action;
    }

    @Override
    public State<S, E, C> transit(C ctx, boolean checkCondition) {
        Debugger.debug("Do transition: "+this);
		// 1. 验证
        this.verify();
		// 2. 检查下条件
        if (!checkCondition || condition == null || condition.isSatisfied(ctx)) {
            if(action != null){
				// 3. 执行动作.
                action.execute(source.getId(), target.getId(), event, ctx);
            }
            return target;
        }
        Debugger.debug("Condition is not satisfied, stay at the "+source+" state ");
        return source;
    }

    @Override
    public final String toString() {
        return source + "-[" + event.toString() +", "+type+"]->" + target;
    }

    @Override
    public boolean equals(Object anObject){
        if(anObject instanceof Transition){
            Transition other = (Transition)anObject;
            if(this.event.equals(other.getEvent())
                    && this.source.equals(other.getSource())
                    && this.target.equals(other.getTarget())){
                return true;
            }
        }
        return false;
    }

    @Override
    public void verify() {
		// 如果type是内部,并且source与target状态是一样的情况下,抛出异常.
        if(type== TransitionType.INTERNAL && source != target) {
            throw new StateMachineException(String.format("Internal transition source state '%s' " +
                    "and target state '%s' must be same.", source, target));
        }
    }
}
```
### (4). 总结
为什么,我一开始就讲这个类的源码,原因在于:Transition承载着,Source/Target/Event/Action等相关信息,也就是我们配置状态与事件的关联,以及动作信息来着的.   
