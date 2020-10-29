---
layout: post
title: 'Ribbon源码(LoadBalancerInterceptor)'
date: 2019-06-29
author: 李新
tags: Ribbon
---

### (1).LoadBalancerInterceptor
```
public class LoadBalancerInterceptor 
       implements ClientHttpRequestInterceptor {
    
    private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;
 
    public ClientHttpResponse intercept(
                    final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
            // 获得请求的URL(http://test-provider/hello)
		final URI originalUri = request.getURI();
            // 获得URL上的host(test-provider)
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
           // 1. 委托给:LoadBalancerRequestFactory创建:LoadBalancerRequest
           // 2. 委托给:RibbonLoadBalancerClient执行请求
		return 
                  this.loadBalancer.execute(
                           serviceName, 
                           requestFactory.createRequest(request, body, execution)
                   );
	} // end intercept 
}
```
### (2).LoadBalancerRequestFactory
```
public LoadBalancerRequest<ClientHttpResponse> createRequest(
                    final HttpRequest request,
			final byte[] body, 
                    final ClientHttpRequestExecution execution) {
    return instance -> {
        HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
        if (transformers != null) {
            for (LoadBalancerRequestTransformer transformer : transformers) {
                serviceRequest = transformer.transformRequest(serviceRequest, instance);
            }
        }
        return execution.execute(serviceRequest, body);
    };
}
```
### (3).RibbonLoadBalancerClient
> 请看RibbonLoadBalancerClient源码讲解