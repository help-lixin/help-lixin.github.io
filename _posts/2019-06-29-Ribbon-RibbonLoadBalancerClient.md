---
layout: post
title: 'Ribbon源码(RibbonLoadBalancerClient)'
date: 2019-06-29
author: 李新
tags: Ribbon
---

### (1).RibbonLoadBalancerClient
```
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    // 根据负载均衡,获取一个服务信息
    Server server = getServer(loadBalancer, hint);
    // ...
 }

protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    if (loadBalancer == null) {
        return null;
    }
    // 委托给:ZoneAwareLoadBalancer处理
    return loadBalancer.chooseServer(hint != null ? hint : "default");
}
```
### (2).ZoneAwareLoadBalancer
```
public Server chooseServer(Object key) {
    // 
    if (!ENABLED.get() || 
         // 根据:test-provider统计可用的zone.当zone少于一个时,进入该代码块
         getLoadBalancerStats().getAvailableZones().size() <= 1) 
    { // true
        logger.debug("Zone aware logic disabled or there is only one zone");
        // 调用父类的chooseServer
        return super.chooseServer(key);
    }
    
    Server server = null;
    try {
        LoadBalancerStats lbStats = getLoadBalancerStats();
        Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
        logger.debug("Zone snapshots: {}", zoneSnapshot);
        if (triggeringLoad == null) {
            triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                    "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
        } //end if

        if (triggeringBlackoutPercentage == null) {
            triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                    "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
        } //end if
        
        Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
        logger.debug("Available zones: {}", availableZones);
        if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
            String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
            logger.debug("Zone chosen: {}", zone);
            if (zone != null) {
                BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                server = zoneLoadBalancer.chooseServer(key);
            }
        } //end if
    } catch (Exception e) {
        logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
    } //end try
    
    
    if (server != null) {
        return server;
    } else {
        logger.debug("Zone avoidance logic is not invoked.");
        return super.chooseServer(key);
    } //end else
}
```
### (3).BaseLoadBalancer
```
public Server chooseServer(Object key) {
    // counter = BasicCounter
    if (counter == null) { // false
        counter = createCounter();
    }
    counter.increment();
    // rule = com.netflix.loadbalancer.ZoneAvoidanceRule
    if (rule == null) { // false
        return null;
    } else {
        try {
            // 委托给:ZoneAvoidanceRule进行调用
            return rule.choose(key);
        } catch (Exception e) {
            logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
            return null;
        }
    }
}
```
### (4).ZoneAvoidanceRule
```
public class ZoneAvoidanceRule 
       extends PredicateBasedRule {
    
}

public abstract class PredicateBasedRule 
       extends ClientConfigEnabledRoundRobinRule {
           
    // IRule才是真正进行服务的选择
    // key == "default"
    public Server choose(Object key) {
        // lb = ZoneAwareLoadBalancer
        // 因为,ZoneAwareLoadBalancer在初始化时:
        // rule.setLoadBalancer(this);
        
        ILoadBalancer lb = getLoadBalancer();
        // lb.getAllServers() 获取所有的服务列表
        // 委托给:AbstractServerPredicate选择服务
        Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
        if (server.isPresent()) {
            return server.get();
        } else {
            return null;
        }       
    }    
}
```
### (5).AbstractServerPredicate
```
public Optional<Server> chooseRoundRobinAfterFiltering(
          // []
          List<Server> servers, 
          // "default"
          Object loadBalancerKey) {
    // 委托给:CompositePredicate
    List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
    
    if (eligible.size() == 0) {
        return Optional.absent();
    }
    // 获取一个Server
    return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
}
```
### (6).CompositePredicate
```
public List<Server> getEligibleServers(
           // [172.17.9.197:8080] -- test-provder服务列表
           List<Server> servers, 
           Object loadBalancerKey  // default
           ) {
    
    // result = [172.17.9.197:8080]
    List<Server> result = super.getEligibleServers(servers, loadBalancerKey);
    // fallbacks = 
    // com.netflix.loadbalancer.AvailabilityPredicate
    // com.netflix.loadbalancer.AbstractServerPredicate$2
    Iterator<AbstractServerPredicate> i = fallbacks.iterator();
    // result.size == 1
    // minimalFilteredServers = 1
    // minimalFilteredPercentage = 0.0
    while (!(result.size() >= minimalFilteredServers && result.size() > (int) (servers.size() * minimalFilteredPercentage))
            && i.hasNext()) { // false
        AbstractServerPredicate predicate = i.next();
        result = predicate.getEligibleServers(servers, loadBalancerKey);
    }
    return result;
}
```
