---
layout: post
title: 'COLA源码之StateMachine(三)' 
date: 2022-09-10
author: 李新
tags:  COLA
---

### (1). 概述
前面对State和Transition进行了源码分析,其实,这两个类是数据的载体,属于业务模型的了解,这一小篇主要剖析的是:StateMachine,在进行StateMachine源码剖析之前,还是一样,先看下接口的职责能力范围.  
### (2). StateMachine
```
public interface StateMachine<S, E, C> extends Visitable{
	// **************************************************
	// 触发事件
	// **************************************************
	S fireEvent(S sourceState, E event, C ctx);
	
	String getMachineId();
	
	void showStateMachine();
	
	String generatePlantUML();
}	
```
### (3). StateMachineImpl
```
public class StateMachineImpl<S, E, C> implements StateMachine<S, E, C> {
	 private String machineId;
	
	// 通过Map收集了所有的State,所以,State是不能重复的,因为是key.
	private final Map<S, State<S, E, C>> stateMap;

	private boolean ready;

	public StateMachineImpl(Map<S, State<S, E, C>> stateMap) {
		this.stateMap = stateMap;
	}

	@Override
	public S fireEvent(S sourceStateId, E event, C ctx) {
		// 1. 检查一把
		isReady();
		
		// *************************************************************
		// 2. 根据当前状态(State+Event),找出一个:Transition
		// *************************************************************
		Transition<S, E, C> transition = routeTransition(sourceStateId, event, ctx);

		if (transition == null) {
			Debugger.debug("There is no Transition for " + event);
			return sourceStateId;
		}

		// *************************************************************
        // 3. 执行Transition.transit
		//    在第一步的源码里就剖析了,就是执行action.
		// *************************************************************
		return transition.transit(ctx, false).getId();
	}

	// *************************************************************
	// 2.1 根据当前状态枚举字符串,获得:State对象
	//     在State对象里又拥有这个State所有的Transition,根据Event过滤出相应的Event,遍历符合条件的:Transition,执行用户配置的condition.
	// *************************************************************
	private Transition<S, E, C> routeTransition(S sourceStateId, E event, C ctx) {
		State sourceState = getState(sourceStateId);
		List<Transition<S, E, C>> transitions = sourceState.getEventTransitions(event);
		if (transitions == null || transitions.size() == 0) {
			return null;
		}
		
		Transition<S, E, C> transit = null;
		for (Transition<S, E, C> transition : transitions) {
			if (transition.getCondition() == null) {
				transit = transition;
			} else if (transition.getCondition().isSatisfied(ctx)) { // 执行自定义配置的condition.
				transit = transition;
				break;
			}
		}

		return transit;
	}

    // ***************************************************************
    // 根据当前的状态枚举,获得State对象,因为:State对象里存储着:Transition
	// ***************************************************************
	private State getState(S currentStateId) {
		State state = StateHelper.getState(stateMap, currentStateId);
		if (state == null) {
			showStateMachine();
			throw new StateMachineException(currentStateId + " is not found, please check state machine");
		}
		return state;
	} // end 
}	
```
### (4). 总结
总体来说,阿里开源的COLA相比Spring StateMachine比较简单和易懂,阿里开源一向的习惯是:当你看到模型的时候,相应的接口能力就出来了的,没那么多的花花肠子来着的(能够证明一件事情,阿里开源时,早就做好了类设计). 