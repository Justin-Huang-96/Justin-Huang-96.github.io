---
layout: post
title: 设计模式
categories: [ Tools ]
description: 浅谈设计模式
keywords: design-pattern
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 浅谈设计模式

先从Spring这个框架开始学习，看看它用了哪些优秀的设计模式

### **1. 工厂模式（Factory Pattern）**

**核心概念：**
工厂模式通过一个工厂类负责创建对象，而不是在代码中直接实例化对象。

为什么？

1. **解耦对象创建与使用**：使用者不关心具体对象如何创建，只关心获取的接口或类。
2. **便于扩展**：可以通过修改工厂类或 `FactoryBean` 实现动态切换不同的 Bean 实现类。
3. **统一管理**：所有 Bean 的创建和生命周期由 Spring 容器统一管理，**便于集中修改和优化**。



> `BeanFactory` 是 Spring 最基础的 IOC 容器接口，它负责管理和创建 Bean 实例。
>
> `ApplicationContext` 是 `BeanFactory` 的子接口，提供更多高级功能，比如事件发布、国际化等。
>
> 





