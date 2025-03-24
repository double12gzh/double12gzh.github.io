---
layout: post 
title: Azure Fleet 功能分析
categories: [k8s,multi-cluster]
description: 分析介绍 kubeFleet 的功能.
keywords: k8s, multi-cluster
excerpt: 摘要：介绍 kubeFleet 的功能.
---

## 目录

* TOC
{:toc}

## 0. 项目概述

**项目概述**

KubeFleet 是一个开源的 Kubernetes 多集群管理解决方案，其核心目标是实现将 Kubernetes 集群视为"牛"（cattle）而不是"宠物"（pets）的理念，即实现集群的标准化管理和自动化运维。

**核心概念**
- Fleet（舰队）：用于管理多个 Kubernetes 集群的解决方案
- Hub Cluster（中心集群）：作为控制平面的 Kubernetes 集群
- Member Cluster（成员集群）：舰队中的成员 Kubernetes 集群
- Fleet-system Namespace：在所有集群中预留的命名空间，用于运行 Fleet 网络控制器和内部资源

**主要功能模块**

- 集群加入/离开机制
- 允许成员集群通过注册自定义资源到中心集群来加入或离开舰队
- 提供集群身份验证和授权机制
- 工作负载编排
- 允许用户在中心集群创建资源
- 支持将资源选择性地传播到舰队中的指定成员集群

## 1. 项目分析
Fleet 是 Kubernetes 社区的一个项目，旨在解决多集群管理中的挑战。它提供了一种标准化的方式来管理多个 Kubernetes 集群，使得集群的加入、离开、身份验证和授权等操作更加简单和可控。

**技术栈**

- 使用 Go 语言开发
- 基于 Kubernetes 原生 API 和自定义资源定义（CRD）
- 使用 Helm 进行部署
- 支持 Docker 容器化

**项目目标和发展方向**

- 集群属性支持
  - 支持节点级别的 SKU 属性（CPU、GPU、内存等）
  - 支持网络拓扑属性

- 调度功能增强
  - 支持命名空间资源的调度
  - 实现动态调度
  - 支持工作负载的亲和性/反亲和性
  - 
- 健康检查
  - 支持自定义工作负载健康检查
  
- 扩展模式
  - 支持工作负载的扩展模式
  - 实现集群间的负载均衡

**使用场景**

- 多集群管理
- 工作负载分发
- 集群资源优化
- 高可用部署
- 跨集群服务编排

这个项目的主要价值在于提供了一个统一的平台来管理多个 Kubernetes 集群，使得集群管理更加自动化和标准化。它特别适合需要管理多个 Kubernetes 集群的企业级应用场景，比如混合云环境、多区域部署等。

## 2. 代码结构分析

基于代码结构分析，我来详细说明 KubeFleet 各个模块之间的协作关系：

1. 核心控制层
   
- 集群管理模块
  - membercluster 控制器：负责成员集群的生命周期管理
  - internalmembercluster 控制器：处理内部成员集群状态
  - memberclusterplacement 控制器：处理集群放置策略
  - clusterinventory 控制器：维护集群资源清单

- 资源编排模块
  - clusterresourceplacement 控制器：负责资源放置决策
  - workgenerator 控制器：生成具体的工作负载
  - workapplier 控制器：应用工作负载到目标集群
  - workv1alpha1 控制器：处理工作负载版本兼容性

2. 调度层

- scheduler 模块：负责资源调度决策
- propertyprovider 模块：提供集群属性信息
- clusterschedulingpolicysnapshot 控制器：维护调度策略快照

3. 监控和变更追踪

- resourcewatcher 模块：监控资源变化
- clusterresourceplacementwatcher 控制器：监控资源放置变化
- clusterresourcebindingwatcher 控制器：监控资源绑定状态
- metrics 模块：收集和暴露指标
  
4. 安全层

- webhook 模块：处理准入控制
- authtoken 模块：处理认证令牌

**模块间协作流程**

1. 集群加入流程
  
```bash
用户请求 -> webhook验证 -> membercluster控制器 -> internalmembercluster控制器
-> clusterinventory更新 -> 集群状态同步
```

2. 资源部署流程

```bash
用户创建资源 -> clusterresourceplacement控制器 -> scheduler决策
-> workgenerator生成工作负载 -> workapplier应用负载 -> resourcewatcher监控状态
```

3. 调度决策流程

```bash
propertyprovider提供集群属性 -> clusterschedulingpolicysnapshot更新
-> scheduler进行调度决策 -> memberclusterplacement执行放置
```

4. 状态同步流程

```bash
resourcewatcher监控变化 -> clusterresourcebindingwatcher更新绑定
-> metrics收集指标 -> 状态反馈给用户
```

**关键交互点**

1. 控制器协调：
- 各个控制器通过 Kubernetes 的 informer 机制监听资源变化
- 控制器之间通过自定义资源（CR）进行状态传递
- 使用 finalizer 确保资源清理的完整性

2. 调度决策：
- propertyprovider 提供集群属性信息
- scheduler 基于属性和策略进行调度决策
- memberclusterplacement 执行具体的放置操作

3. 工作负载管理：
- workgenerator 生成具体的工作负载
- workapplier 负责将工作负载应用到目标集群
- resourcewatcher 监控工作负载状态

4. 状态同步：
- 各个 watcher 持续监控资源状态
- metrics 模块收集性能指标
- 状态变化通过 CR 更新反馈

**数据流向**

1. 配置数据流：

```bash
用户配置 -> webhook验证 -> 控制器处理 -> 资源创建/更新
```

2. 状态数据流：
```bash
集群状态 -> watcher监控 -> 控制器处理 -> 状态更新
```

3. 指标数据流：

```bash
系统运行 -> metrics收集 -> 指标暴露 -> 监控系统
```

这种模块化的设计使得系统具有良好的可扩展性和可维护性，每个模块都有明确的职责，通过标准的 Kubernetes 机制进行协作。同时，模块间的松耦合设计也使得系统更容易进行功能扩展和问题定位。

### 2.1 membercluster 控制器 与 internalmembercluster 控制器

1. 职责定位不同

**MemberCluster 控制器：**

- 运行在 Hub 集群（中心集群）上
- 负责处理用户创建的 MemberCluster 资源
- 主要职责：
  - 管理成员集群的加入/离开流程
  - 创建和管理必要的 RBAC 资源
  - 创建和管理命名空间
  - 创建和管理 InternalMemberCluster 资源
  - 处理成员集群的生命周期事件

**InternalMemberCluster 控制器：**

- 运行在成员集群上
- 负责处理成员集群内部的协调工作
- 主要职责：
  - 管理成员集群的健康状态
  - 收集和报告集群属性
  - 管理成员集群上的代理（agents）
  - 维护心跳信息
  - 处理资源统计信息

2. 功能特点不同

**MemberCluster 控制器：**

- 提供集群加入/离开的完整流程
- 管理集群级别的权限和访问控制
- 处理集群级别的资源清理
- 维护集群加入状态
- 提供集群级别的条件（conditions）管理

**InternalMemberCluster 控制器：**

- 提供集群属性收集机制
- 管理集群健康检查
- 处理集群资源统计
- 维护代理状态
- 提供集群心跳机制

3. 交互对象不同

**MemberCluster 控制器：**

- 与 Kubernetes API 服务器交互
- 创建和管理 RBAC 资源
- 创建和管理命名空间
- 管理 `InternalMemberCluster` 资源

**InternalMemberCluster 控制器：**

- 与成员集群的 API 服务器交互
- 与属性提供者（PropertyProvider）交互
- 与成员集群上的代理交互
- 管理集群资源统计

4. 状态管理不同

**MemberCluster 控制器：**

- 管理集群加入状态（ReadyToJoin, Joined, Left 等）
- 处理集群级别的条件
- 管理集群级别的资源状态

**InternalMemberCluster 控制器：**

- 管理集群健康状态
- 管理集群属性状态
- 管理代理状态
- 维护心跳状态

5. 资源管理不同
  
**MemberCluster 控制器：**

- 管理集群级别的资源（命名空间、RBAC 等）
- 处理集群资源的创建和清理
- 管理集群级别的权限

**InternalMemberCluster 控制器：**

- 收集和报告集群资源统计
- 管理集群内部代理资源
- 处理集群属性资源

6. 工作流程不同

**MemberCluster 控制器：**

```bash
用户创建 MemberCluster -> 创建命名空间和 RBAC -> 创建 InternalMemberCluster
-> 管理集群加入状态 -> 处理集群生命周期事件
```

**InternalMemberCluster 控制器：**

```bash
启动代理 -> 收集集群属性 -> 更新健康状态 -> 维护心跳
-> 报告资源统计 -> 管理代理状态
```

**总结**
这两个控制器是 KubeFleet 中集群管理的两个关键组件，它们协同工作但职责不同：
- `MemberCluster 控制器`负责集群级别的管理和协调
- `InternalMemberCluster 控制器`负责成员集群内部的运行状态管理

它们通过 Kubernetes 的 API 和自定义资源进行交互，共同确保集群管理的完整性和可靠性。这种设计使得集群管理更加模块化和可维护，同时也提供了更好的扩展性。

### 2.2 ClusterInventory 控制器

**ClusterInventory 控制器主要负责：**

- 为每个 `MemberCluster` 创建和管理对应的 `ClusterProfile`
- 同步集群状态和健康信息
- 处理集群配置文件的清理工作

**工作流程：**

1. 获取 MemberCluster
- 检索 MemberCluster 对象
- 处理对象不存在的情况

2. 处理删除操作
- 检查是否标记为删除
- 清理相关的 ClusterProfile
- 移除清理 finalizer

3. 管理 Finalizer
- 检查并添加清理 finalizer
- 确保资源正确清理

4. 创建/更新 ClusterProfile
- 创建或更新对应的 ClusterProfile
- 设置必要的标签和显示名称
- 同步集群状态

**状态同步：**

```go
func (r *Reconciler) syncClusterProfileCondition(mc *clusterv1beta1.MemberCluster, cp *clusterinventory.ClusterProfile)
```

状态同步逻辑：

1. 健康状态检查
- 检查成员代理状态
- 验证心跳信息
- 评估 API 服务器健康状态

2. 状态条件设置
- 设置控制平面健康状态
- 更新观察到的生成版本
- 提供详细的状态消息

状态同步条件

```go
const (
    clusterNoStatusReason      = "MemberAgentReportedNoStatus"
    clusterHeartbeatLostReason = "MemberAgentHeartbeatLost"
    clusterHealthUnknownReason = "MemberAgentReportedNoHealthInfo"
    clusterUnHealthyReason     = "MemberClusterAPIServerUnhealthy"
    clusterHealthyReason       = "MemberClusterAPIServerHealthy"
)
```

健康状态评估

```go
switch {
case memberAgentStatus == nil:
    // 处理未报告状态的情况
case mcHealthCond == nil:
    // 处理缺少健康条件的情况
case memberAgentLastHeartbeat == nil || time.Since(memberAgentLastHeartbeat.Time) > r.ClusterUnhealthyThreshold:
    // 处理心跳丢失的情况
case mcHealthCond.Status == metav1.ConditionUnknown:
    // 处理未知状态的情况
case mcHealthCond.Status == metav1.ConditionFalse:
    // 处理不健康状态的情况
default:
    // 处理健康状态的情况
}

```

**使用场景**

1. 集群健康监控
- 监控集群 API 服务器状态
- 跟踪心跳信息
- 评估集群健康状态

2. 集群配置管理
- 维护集群配置文件
- 同步集群状态
- 管理集群元数据

3. 资源生命周期管理
- 处理集群加入/离开
- 管理资源清理
- 维护资源状态
