---
layout: post
title: k8s工作负载简介
categories: [ k8s,workload ]
description: simple talk about k8s-workload
keywords: k8s,workload
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---
# Kubernetes 工作负载总结

## 核心工作负载类型
1. **Deployment**
    - **用途**：部署无状态应用（如微服务）。
    - **特点**：
        - 支持多副本管理，关注副本数量和分布。
        - 适用于无状态场景，数据通常存储于外部中间件（如Redis、MySQL）。
        - Pod 重启后 IP 可能变化，无持久化存储。

2. **StatefulSet**
    - **用途**：部署有状态应用（如 Redis、MySQL 等数据中间件）。
    - **特点**：
        - 提供稳定的网络标识（固定访问地址）和持久化存储。
        - Pod 重启后能挂载原有数据，适合需要数据持久化的场景。

3. **DaemonSet**
    - **用途**：部署守护进程（如日志收集器）。
    - **特点**：
        - 每个节点（Node）运行且仅运行一个 Pod。
        - 适用于需全局驻留服务的场景（如监控、日志收集）。

4. **Job / CronJob**
    - **用途**：运行一次性任务或定时任务（如垃圾清理）。
    - **特点**：
        - `Job`：执行一次性任务，完成后 Pod 终止。
        - `CronJob`：基于时间调度（如每天凌晨运行）。

---

## 核心原则
- **不直接部署 Pod**：  
  Pod 是应用的载体，但应通过工作负载（Deployment/StatefulSet 等）管理，以增强功能（如副本控制、稳定性、调度策略）。

- **选择依据**：
    - 无状态服务 → **Deployment**
    - 有状态服务 → **StatefulSet**
    - 节点级守护进程 → **DaemonSet**
    - 任务调度 → **Job / CronJob**

---

## 补充说明
- **网络访问问题**：
    - 当前部署的应用仅在内网可通过 Pod IP 访问（如 `curl <Pod-IP>`）。
    - 外网访问需通过后续网络配置（如 Service、Ingress），将在后续课程探讨。

- **参考资源**：
    - [Kubernetes 官方文档](https://kubernetes.io/docs/concepts/workloads/)
