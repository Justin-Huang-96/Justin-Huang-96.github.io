---
layout: post
title: "深入理解InheritableThreadLocal和TransmittableThreadLocal"
categories: [ Java, ThreadLocal ]
description: "详细介绍ThreadLocal的继承版本InheritableThreadLocal以及阿里巴巴开源的TransmittableThreadLocal在线程间数据传递的解决方案"
keywords: [java,threadlocal,inheritablethreadlocal,transmittablethreadlocal,ttl,线程池]
---

# 深入理解InheritableThreadLocal和TransmittableThreadLocal

> 本文将详细探讨Java中的ThreadLocal变量在线程间传递的解决方案，包括JDK自带的InheritableThreadLocal以及阿里巴巴开源的TransmittableThreadLocal，并通过实际示例展示它们的使用场景和局限性。

## 简介

在多线程编程中，ThreadLocal为每个线程提供了变量的本地副本，使得多个线程可以同时访问各自独立的变量实例，避免了线程安全问题。然而，当涉及到父子线程间的数据传递或在线程池场景中，标准的ThreadLocal就显得力不从心了。为了解决这些问题，Java提供了InheritableThreadLocal，而阿里巴巴开源了TransmittableThreadLocal（TTL）。

## ThreadLocal的局限性

在介绍InheritableThreadLocal之前，让我们先看看标准ThreadLocal的局限性。ThreadLocal只能在当前线程中访问，无法传递给子线程：

```java
private static void normal_example() {
    local.set("父线程的值");

    Thread child = new Thread(() -> {
        System.out.println("子线程读取：" + local.get()); // 输出：null
    });

    child.start();
}
```

在上述示例中，子线程无法获取父线程设置的ThreadLocal值。

## InheritableThreadLocal

### 基本概念

InheritableThreadLocal是ThreadLocal的子类，它允许子线程继承父线程的ThreadLocal变量值。当创建一个新的子线程时，该子线程可以访问其父线程中设置的InheritableThreadLocal值。

### 工作原理

在Thread类中有一个私有变量inheritableThreadLocals，在线程创建时会执行init方法，将父线程中的inheritableThreadLocals传递给子线程。

```java
private static void normal_example() {
    local.set("父线程的值");

    Thread child = new Thread(() -> {
        System.out.println("子线程读取：" + local.get()); // 输出：父线程的值
    });

    child.start();
}
```

### 局限性

虽然InheritableThreadLocal解决了父子线程间的数据传递问题，但它在使用线程池时存在明显缺陷。因为线程池中的线程是复用的，只有在线程创建时才会进行父子线程的传递，而线程池中的线程在执行完任务后并不会销毁，而是返回线程池等待下一次使用。

```java
private static void fail_example() throws InterruptedException {
    ExecutorService executor = Executors.newFixedThreadPool(1); // 线程池复用线程

    local.set("主线程设置的值");

    Runnable task = () -> {
        System.out.println("线程池任务中读取：" + local.get());
    };

    executor.submit(task); // 第一次提交
    Thread.sleep(100); // 等一会，确保任务执行完

    local.set("主线程修改后的值");

    executor.submit(task); // 第二次提交
    executor.shutdown();
}
```

在上述示例中，由于线程池中的线程复用了第一次任务时的ThreadLocal值，导致第二次任务执行时获取到的值并不是主线程最新设置的值。

## TransmittableThreadLocal

### 基本概念

TransmittableThreadLocal（TTL）是阿里巴巴开源的一个库，专门用于解决线程池环境下ThreadLocal值传递的问题。它能够在任务执行时自动将主线程的ThreadLocal值传递给执行线程，并在任务执行完成后清除。

### 工作原理

TTL通过装饰器模式，对线程池的执行方法进行增强，在任务提交时捕获当前线程的ThreadLocal值，并在任务执行前将这些值设置到执行线程中，任务执行完成后清理这些值。

### 使用示例

```java
private static void alibaba_ttl_example() {
    // 原始线程池（固定大小）
    ExecutorService rawExecutor = Executors.newFixedThreadPool(1);
    // 包装为TTL支持的线程池
    ExecutorService ttlExecutor = TtlExecutors.getTtlExecutorService(rawExecutor);

    // 设置主线程中的上下文
    context.set("主线程设置的值");

    Runnable task = () -> {
        System.out.println("线程池任务中读取的值：" + context.get());
    };

    // 提交任务时，会自动传递上下文
    ttlExecutor.submit(task);

    // 主线程更新上下文
    context.set("主线程修改后的值");

    ttlExecutor.submit(task);

    ttlExecutor.shutdown();
}
```

在上述示例中，每次任务提交时，都会获取到主线程当前设置的值，解决了线程池环境下ThreadLocal值传递的问题。

### Maven依赖

要使用TransmittableThreadLocal，需要在项目中添加以下依赖：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.12.2</version>
</dependency>
```

## 三种ThreadLocal的对比

| 特性 | ThreadLocal | InheritableThreadLocal | TransmittableThreadLocal |
|------|-------------|------------------------|--------------------------|
| 线程内访问 | 支持 | 支持 | 支持 |
| 父子线程传递 | 不支持 | 支持 | 支持 |
| 线程池环境 | 不支持 | 不支持 | 支持 |
| 性能开销 | 低 | 中等 | 中等偏高 |
| 实现复杂度 | 简单 | 中等 | 复杂 |

## 应用场景

### 日志追踪

在分布式系统中，通常需要追踪一个请求在各个服务间的调用链路。通过TransmittableThreadLocal可以方便地传递traceId等上下文信息。

```java
public class TraceContext {
    private static final TransmittableThreadLocal<String> TRACE_ID = new TransmittableThreadLocal<>();
    
    public static void setTraceId(String traceId) {
        TRACE_ID.set(traceId);
    }
    
    public static String getTraceId() {
        return TRACE_ID.get();
    }
    
    public static void clear() {
        TRACE_ID.remove();
    }
}
```

### 用户身份传递

在多线程处理用户请求时，可以通过TransmittableThreadLocal传递用户身份信息，避免在方法间手动传递。

```java
public class UserContext {
    private static final TransmittableThreadLocal<User> USER = new TransmittableThreadLocal<>();
    
    public static void setUser(User user) {
        USER.set(user);
    }
    
    public static User getCurrentUser() {
        return USER.get();
    }
}
```

## 最佳实践

### 及时清理

使用完ThreadLocal后，应及时调用remove()方法清理数据，避免内存泄漏。

### 合理选择

- 普通单线程场景使用ThreadLocal
- 简单的父子线程场景使用InheritableThreadLocal
- 线程池或复杂异步场景使用TransmittableThreadLocal

### 封装使用

建议将ThreadLocal封装在专门的上下文类中，提供统一的访问接口。

## 结论

ThreadLocal及其扩展版本InheritableThreadLocal和TransmittableThreadLocal为Java多线程编程提供了强大的线程隔离和数据传递能力。在实际开发中，我们应该根据具体场景选择合适的方案，以确保程序的正确性和性能。

## 思考

1. 在实际项目中，如何权衡三种ThreadLocal方案的选择？
2. 使用TransmittableThreadLocal时，如何避免对性能造成过大影响？