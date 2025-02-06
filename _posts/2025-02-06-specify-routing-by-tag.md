---
layout: post  
title: 基于Tag路由请求到指定实例的技术实现详解  
categories: [ microservices, distributed-system ]  
description: 解析如何基于请求头中的 Tag 实现动态路由，将流量精准分发到指定服务实例。  
keywords: tag routing, spring cloud gateway, nacos, load balancing  
mermaid: false  
sequence: false  
flow: false  
mathjax: false  
mindmap: false  
mindmap2: false
---
# 基于Tag路由请求到指定实例的技术实现详解
在现代微服务架构中，动态路由和灰度发布是提升系统灵活性和可靠性的重要手段。本文将深入解析如何基于请求头中的 `Tag` 实现动态路由，将流量精准分发到指定服务实例。以下是实现这一功能的核心技术亮点及实现细节。

## 一、背景与需求
### 1.1 灰度发布与动态路由
灰度发布（金丝雀发布）允许将新版本服务逐步暴露给部分用户，而动态路由则能根据请求特征（如 Header、参数等）将流量导向特定实例。基于 `Tag` 的路由是一种常见的实现方式，例如：

+ 将内测用户的请求路由到新版本服务（`Tag=canary`）。
+ 根据地理位置选择最近的服务实例（`Tag=region`）。

### 1.2 核心需求
1. **请求头匹配**：根据请求头中的 `Tag` 筛选目标实例。
2. **灵活回退**：若无匹配实例，则回退到默认实例。
3. **负载均衡**：支持权重分配，避免单点过载。

---

## 二、技术架构与核心组件
### 2.1 Spring Cloud Gateway
作为 API 网关，Spring Cloud Gateway 提供了全局过滤器机制，能够拦截请求并修改路由逻辑。其响应式编程模型（基于 Reactor）确保了高并发场景下的性能。

### 2.2 Spring Cloud LoadBalancer
Spring Cloud LoadBalancer 是 Spring Cloud 生态中的负载均衡器，支持自定义负载均衡策略。通过实现 `ReactorServiceInstanceLoadBalancer` 接口，可以定制实例选择逻辑。

### 2.3 Nacos
作为注册中心，Nacos 提供了服务实例的元数据管理（如 `Tag`）和权重配置能力。通过 `NacosBalancer`，可以直接使用 Nacos 的权重分配策略。

---

## 三、关键技术实现
### 3.1 基于 Tag 的实例筛选
#### 3.1.1 请求头提取
在 `TagLoadBalancer` 中，从请求的 `HttpHeaders` 中提取 `Tag` 值：

```java
String tag = headers.getFirst("tag");
```

#### 3.1.2 元数据匹配
遍历服务实例的元数据（Metadata），筛选出 Tag 匹配的实例：

```java
List<ServiceInstance> filteredInstances = instances.stream()
    .filter(instance -> tag.equals(instance.getMetadata().get("tag")))
    .toList();
```

#### 3.1.3 回退机制
若未找到匹配实例，则返回全部实例，确保服务可用性：

```java
if (CollUtil.isEmpty(filteredInstances)) {
        return instances; // 回退到所有实例
}
```

### 3.2 权重负载均衡
使用 Nacos 的权重分配策略（`NacosBalancer.getHostByRandomWeight3`），实现随机加权选择：

```java
return new DefaultResponse(NacosBalancer.getHostByRandomWeight3(chooseInstances));
```

### 3.3 网关过滤器集成
在 `TagReactiveLoadBalancerClientFilter` 中，通过全局过滤器拦截请求，调用自定义负载均衡器：

```java
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // 构建负载均衡请求并选择实例
    return choose(lbRequest, serviceId, supportedLifecycleProcessors)
        .doOnNext(response -> handleResponse(...))
        .then(chain.filter(exchange));
}
```

### 3.4 负载均衡生命周期管理
通过 `LoadBalancerLifecycle` 处理负载均衡各阶段事件（如开始、成功、失败），增强可观测性：

```java
processors.forEach(lifecycle -> lifecycle.onStart(lbRequest));
processors.forEach(lifecycle -> lifecycle.onComplete(...));
```


## 四、亮点总结
### 4.1 动态路由能力
+ **灵活匹配**：通过请求头与元数据的匹配，实现精准路由。
+ **无缝回退**：保障在无匹配实例时的服务可用性。

### 4.2 高性能与扩展性
+ **响应式编程**：基于 Reactor 的非阻塞模型，支持高并发场景。
+ **可插拔设计**：通过替换 `NacosBalancer`，可适配其他注册中心（如 Consul、Eureka）。

### 4.3 生产级特性
+ **权重支持**：结合 Nacos 实现流量按权重分配。
+ **全链路监控**：通过生命周期钩子记录负载均衡过程，便于问题排查。


## 五、应用场景
+ **灰度发布**：将部分用户流量导向新版本服务。
+ **多环境隔离**：区分测试环境与生产环境实例。
+ **区域性路由**：根据用户地理位置选择最优实例。


## 六、源码分享
### spring cloud gateway配置文件
```yml
server:
   port: 8180  # 这里是服务启动的端口，多个服务需使用不同端口

spring:
   application:
      name: cloud-gateway  # 应用名称，Nacos 用于服务发现
   config:
      #    nacos:您Nacos配置的dataId[.group]@存在Nacos的配置文件扩展名
      import: optional:nacos:cloud-gateway.yaml
   cloud:
      nacos:
         discovery:
            server-addr: localhost:8848  # Nacos 服务器地址（用于服务注册）
         config:
            server-addr: localhost:8848  # Nacos 服务器地址（用于配置中心）
            file-extension: yaml  # 指定 Nacos 配置文件格式，可选：properties、yaml
            group: DEFAULT_GROUP  # Nacos 配置分组
            namespace: public  # Nacos 命名空间（默认为 public）

      gateway:
         routes:
            - id: cloud-service1-route
              uri: lb://cloud-service1  # 指向 Nacos 或注册中心的服务名
              predicates:
                 - Path=/cloud-service1/**  # 匹配 /api/cloud-service1/xxx
              filters:
                 - RewritePath=/cloud-service1/(?<remaining>.*), /${remaining}  # 去掉 /api/cloud-service1

```
### LoadBalancer
```java
package com.example.cloudgateway.grey;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.*;
import org.springframework.cloud.gateway.config.GatewayLoadBalancerProperties;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.cloud.gateway.filter.ReactiveLoadBalancerClientFilter;
import org.springframework.cloud.gateway.support.DelegatingServiceInstance;
import org.springframework.cloud.gateway.support.NotFoundException;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.net.URI;
import java.util.Map;
import java.util.Set;

import static org.springframework.cloud.gateway.support.ServerWebExchangeUtils.*;

/**
 * <p>
 * 根据请求的 header[tag] 匹配，筛选满足 metadata[tag] 相等的服务实例列表。
 */
@Component
@AllArgsConstructor
@Slf4j
public class TagReactiveLoadBalancerClientFilter implements GlobalFilter, Ordered {

    private final LoadBalancerClientFactory clientFactory;
    private final GatewayLoadBalancerProperties properties;

    @Override
    public int getOrder() {
        return ReactiveLoadBalancerClientFilter.LOAD_BALANCER_CLIENT_FILTER_ORDER;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
//        String schemePrefix = exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR);

//        // 如果不是灰度负载均衡请求，则直接跳过
//        if (url == null || (!"grayLb".equals(url.getScheme()) && !"grayLb".equals(schemePrefix))) {
//            return chain.filter(exchange);
//        }

        // 保留原始 URL
        addOriginalRequestUrl(exchange, url);

        // 获取服务实例名
        URI requestUri = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
        String serviceId = requestUri.getHost();

        // 获取负载均衡生命周期处理器
        Set<LoadBalancerLifecycle> supportedLifecycleProcessors = LoadBalancerLifecycleValidator
                .getSupportedLifecycleProcessors(clientFactory.getInstances(serviceId, LoadBalancerLifecycle.class),
                        RequestDataContext.class, ResponseData.class, ServiceInstance.class);

        // 创建负载均衡请求
        DefaultRequest<RequestDataContext> lbRequest = new DefaultRequest<>(
                new RequestDataContext(new RequestData(exchange.getRequest()), getHint(serviceId)));

        // 选择服务实例并处理响应
        return choose(lbRequest, serviceId, supportedLifecycleProcessors)
                .doOnNext(response -> handleResponse(response, exchange, lbRequest, supportedLifecycleProcessors, url))
                .then(chain.filter(exchange))
                .doOnError(throwable -> handleError(throwable, supportedLifecycleProcessors, lbRequest, exchange))
                .doOnSuccess(aVoid -> handleSuccess(supportedLifecycleProcessors, lbRequest, exchange));
    }

    /**
     * 处理负载均衡响应
     */
    private void handleResponse(Response<ServiceInstance> response, ServerWebExchange exchange,
                                DefaultRequest<RequestDataContext> lbRequest,
                                Set<LoadBalancerLifecycle> processors, URI url) {
        if (!response.hasServer()) {
            processors.forEach(lifecycle -> lifecycle
                    .onComplete(new CompletionContext<>(CompletionContext.Status.DISCARD, lbRequest, response)));
            throw NotFoundException.create(properties.isUse404(), "Unable to find instance for " + url.getHost());
        }

        // 构建服务实例 URI
        ServiceInstance instance = response.getServer();
        String overrideScheme = instance.isSecure() ? "https" : "http";
        if (exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR) != null) {
            overrideScheme = url.getScheme();
        }

        DelegatingServiceInstance serviceInstance = new DelegatingServiceInstance(instance, overrideScheme);
        URI requestUrl = reconstructURI(serviceInstance, exchange.getRequest().getURI());

        // 设置请求属性
        exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
        exchange.getAttributes().put(GATEWAY_LOADBALANCER_RESPONSE_ATTR, response);
        processors.forEach(lifecycle -> lifecycle.onStartRequest(lbRequest, response));
    }

    /**
     * 处理负载均衡错误
     */
    private void handleError(Throwable throwable, Set<LoadBalancerLifecycle> processors,
                             DefaultRequest<RequestDataContext> lbRequest, ServerWebExchange exchange) {
        processors.forEach(lifecycle -> lifecycle
                .onComplete(new CompletionContext<>(CompletionContext.Status.FAILED, throwable, lbRequest,
                        exchange.getAttribute(GATEWAY_LOADBALANCER_RESPONSE_ATTR))));
    }

    /**
     * 处理负载均衡成功
     */
    private void handleSuccess(Set<LoadBalancerLifecycle> processors, DefaultRequest<RequestDataContext> lbRequest,
                               ServerWebExchange exchange) {
        processors.forEach(lifecycle -> lifecycle
                .onComplete(new CompletionContext<>(CompletionContext.Status.SUCCESS, lbRequest,
                        exchange.getAttribute(GATEWAY_LOADBALANCER_RESPONSE_ATTR),
                        new ResponseData(exchange.getResponse(), new RequestData(exchange.getRequest())))));
    }

    /**
     * 选择服务实例
     */
    private Mono<Response<ServiceInstance>> choose(Request<RequestDataContext> lbRequest, String serviceId,
                                                   Set<LoadBalancerLifecycle> processors) {
        TagLoadBalancer loadBalancer = new TagLoadBalancer(
                clientFactory.getLazyProvider(serviceId, ServiceInstanceListSupplier.class), serviceId);
        processors.forEach(lifecycle -> lifecycle.onStart(lbRequest));
        return loadBalancer.choose(lbRequest);
    }

    /**
     * 获取负载均衡提示
     */
    private String getHint(String serviceId) {
        LoadBalancerProperties properties = clientFactory.getProperties(serviceId);
        Map<String, String> hints = properties.getHint();
        return hints.getOrDefault(serviceId, hints.getOrDefault("default", "default"));
    }

    /**
     * 重构 URI
     */
    protected URI reconstructURI(ServiceInstance serviceInstance, URI original) {
        return LoadBalancerUriTools.reconstructURI(serviceInstance, original);
    }
}
```

### 路由策略实现类
```java
package com.example.cloudgateway.grey;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.text.CharSequenceUtil;
import com.alibaba.cloud.nacos.balancer.NacosBalancer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.*;
import org.springframework.cloud.loadbalancer.core.NoopServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.core.ReactorServiceInstanceLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.http.HttpHeaders;
import reactor.core.publisher.Mono;

import java.util.List;

/**
 * 灰度 {@link TagLoadBalancer} 实现类
 * <p>
 * 根据请求的 header[tag] 匹配，筛选满足 metadata[tag] 相等的服务实例列表，然后随机 + 权重进行选择一个。
 * 1. 假如请求的 header[tag] 为空，则不进行筛选，所有服务实例都进行选择。
 * 2. 如果 metadata[tag] 都不相等，则不进行筛选，所有服务实例都进行选择。
 * <p>
 * 注意，权重实现基于 {@link NacosBalancer}，若更换注册中心需调整筛选逻辑。
 */
@RequiredArgsConstructor
@Slf4j
public class TagLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    /**
     * 用于获取 serviceId 对应的服务实例的列表
     */
    private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

    /**
     * 服务实例名，用于日志记录
     */
    private final String serviceId;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        // 获取请求头
        HttpHeaders headers = ((RequestDataContext) request.getContext()).getClientRequest().getHeaders();
        // 获取服务实例列表
        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request).next().map(list -> getInstanceResponse(list, headers));
    }

    /**
     * 根据请求头中的 tag 过滤服务实例
     */
    private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances, HttpHeaders headers) {
        // 如果服务实例为空，则直接返回
        if (CollUtil.isEmpty(instances)) {
            log.warn("[getInstanceResponse][serviceId({}) 服务实例列表为空]", serviceId);
            return new EmptyResponse();
        }

        // 基于 tag 过滤实例列表
        List<ServiceInstance> chooseInstances = filterTagServiceInstances(instances, headers);

        // 随机 + 权重获取实例
        return new DefaultResponse(NacosBalancer.getHostByRandomWeight3(chooseInstances));
    }

    /**
     * 基于 tag 请求头，过滤匹配 tag 的服务实例列表
     */
    private List<ServiceInstance> filterTagServiceInstances(List<ServiceInstance> instances, HttpHeaders headers) {
        // 获取请求头中的 tag
        String tag = headers.getFirst("tag");
        if (CharSequenceUtil.isEmpty(tag)) {
            return instances;
        }

        // 过滤匹配 tag 的实例
        List<ServiceInstance> filteredInstances = instances.stream()
                .filter(instance -> tag.equals(instance.getMetadata().get("tag"))).toList();

        if (CollUtil.isEmpty(filteredInstances)) {
            log.warn("[filterTagServiceInstances][serviceId({}) 没有满足 tag({}) 的服务实例列表，直接使用所有服务实例列表]", serviceId, tag);
            return instances;
        }
        log.info("找到满足tag {} 的实例", instances);
        return filteredInstances;
    }
}
```