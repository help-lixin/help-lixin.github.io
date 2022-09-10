---
layout: post
title: 'COLA源码之State(二)' 
date: 2022-09-10
author: 李新
tags:  COLA
---

### (1). 概述
前面对Transition的源码进行了分析,在这一小篇将对:State进行部析,在剖析之前,我们行看一下State接口,因为,接口代表了它的大概的职责能力范围. 

### (2). State
```
// ********************************************************
// State你可以理解为:状态
// ********************************************************
public interface State<S,E,C> extends Visitable {
	S getId();
	
	// 添加当前状态(State)与目标状态(State)关联的事件(Event).
	Transition<S,E,C> addTransition(E event, State<S, E, C> target, TransitionType transitionType);
	
	// 返回当前状态(State)所有的:Transition
	List<Transition<S,E,C>> getEventTransitions(E event);
	
	Collection<Transition<S,E,C>> getAllTransitions();
}	
```
### (3). StateImpl
```
public class StateImpl<S,E,C> implements State<S,E,C> {
    protected final S stateId;
	
	// 
    private EventTransitions eventTransitions = new EventTransitions();

    StateImpl(S stateId){
        this.stateId = stateId;
    }

    // *********************************************************************
	// 1. 添加状态转换相应的处理
	// *********************************************************************
    @Override
    public Transition<S, E, C> addTransition(E event, State<S,E,C> target, TransitionType transitionType) {
		
        Transition<S, E, C> newTransition = new TransitionImpl<>();
		// *********************************************************************
		// 2. 配置:原始状态(State) / 目标状态(State) / 事件(Event)
		//        注意:原始状态就是当前这个类
		// *********************************************************************
        newTransition.setSource(this);
        newTransition.setTarget(target);
        newTransition.setEvent(event);
        newTransition.setType(transitionType);

        Debugger.debug("Begin to add new transition: "+ newTransition);

        // *********************************************************************
		// 看得出来:eventTransitions是一个MAP对象,Key是事件(Event),Value:是Transition
		// *********************************************************************
        eventTransitions.put(event, newTransition);
        return newTransition;
    }

    @Override
    public List<Transition<S, E, C>> getEventTransitions(E event) {
        return eventTransitions.get(event);
    }

    @Override
    public Collection<Transition<S, E, C>> getAllTransitions() {
        return eventTransitions.allTransitions();
    }

    @Override
    public S getId() {
        return stateId;
    }

    @Override
    public String accept(Visitor visitor) {
        String entry = visitor.visitOnEntry(this);
        String exit = visitor.visitOnExit(this);
        return entry + exit;
    }
}
```
### (4). EventTransitions
```
public class EventTransitions<S,E,C> {
    private HashMap<E, List<Transition<S,E,C>>> eventTransitions;

    public EventTransitions(){
        eventTransitions = new HashMap<>();
    }

    public void put(E event, Transition<S, E, C> transition){
        if(eventTransitions.get(event) == null){
            List<Transition<S,E,C>> transitions = new ArrayList<>();
            transitions.add(transition);
            eventTransitions.put(event, transitions);
        }
        else{
            List existingTransitions = eventTransitions.get(event);
            verify(existingTransitions, transition);
            existingTransitions.add(transition);
        }
    }

    private void verify(List<Transition<S,E,C>> existingTransitions, Transition<S,E,C> newTransition) {
        for (Transition transition : existingTransitions) {
            if (transition.equals(newTransition)) {
                throw new StateMachineException(transition + " already Exist, you can not add another one");
            }
        }
    }

    public List<Transition<S,E,C>> get(E event){
        return eventTransitions.get(event);
    }

    public List<Transition<S,E,C>> allTransitions(){
        List<Transition<S,E,C>> allTransitions = new ArrayList<>();
        for(List<Transition<S,E,C>> transitions : eventTransitions.values()){
            allTransitions.addAll(transitions);
        }
        return allTransitions;
    }
}
```
### (5). 总结
State代表状态,一个State内部持有一个EventTransitions,key是Event,而value是Transition对象的集合.  