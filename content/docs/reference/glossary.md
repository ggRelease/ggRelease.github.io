---
title: 术语表
linkTitle: 术语表
description: grelease 项目通用语言（Ubiquitous Language）：灰度发布、蓝绿部署、放量速率、错误率等核心概念的统一定义。
weight: 10
---

本页是 grelease 项目的统一术语表，源自主仓库 `CONTEXT.md`。使用指南与维护者指南中出现的术语均以此为准；每个术语标注了应当**避免使用**的近义词，防止团队内产生歧义表达。

### 统一入口（Gateway Ingress）

Spring Cloud Gateway，作为最外层流量入口，负责所有请求的统一下发与灰度决策。

> 避免使用：入口网关、边缘层

### 灰度发布（Canary Release）

受「放量速率」与「错误率」监控的渐进放量方式。放量速率按**请求量步进**控制（单位时间进入新版本的请求量上限）；错误率仅作为**统计与告警信号**，不自动干预流量。灰度阶段的终态是**硬切到蓝绿部署**——即灰度观察完成后，由运维主动将流量整体切换到目标版本的蓝绿槽位，灰度规则随之结束。

> 避免使用：比例路由、按量分流、自动熔断停止

### 蓝绿部署（Blue-Green Deployment）

两套环境整体切换的发布方式，本框架必须支持。与灰度发布是两套独立机制，时间上串行编排：先灰度观察，后蓝绿收尾。

### 灰度决策（Canary Decision）

网关依据请求特征（header/参数/标签）精确匹配命中目标版本的决策过程，不按百分比哈希。

> 避免使用：比例路由、按量分流

### 流量染色（Traffic Tagging）

在统一入口（SCG）内为请求附加灰度特征（如灰度 header）的过程。特征来源为混合模式：既支持客户端主动传入（header/query/cookie 等），也支持网关根据请求属性或 etcd 规则主动染色。网关染色后路由到首个服务。

### 灰度上下文传播（Gray Context Propagation）

应用消费网关染色产生的灰度上下文，依据 etcd 配置在服务调用链中向后传递（类似链路追踪的灰度标签传递），保证一次请求的灰度决策贯穿整条调用链。

### 放量速率（Ramp-Up Rate）

灰度放量的推进方式，按**请求量步进**控制——即单位时间允许进入新版本的请求量上限，逐步上调。

### 错误率（Error Rate）

灰度效果的监控信号。业务失败仅由**后端业务代码返回约定错误码（HTTP 510）**定义；HTTP 200 一律视为正确。网关统计响应中的 510 并按 **gtid 去重**计算错误率。错误率超过阈值时只触发**告警与统计**，不自动停止或回滚流量。

> 避免使用：自动熔断、自动回滚

### 灰度度量（Canary Metrics）

基于 Micrometer 采集的灰度效果统计（请求数、510 数、错误率），按版本分桶、按时间窗口留存历史，存入 etcd，支撑控制台趋势展示与告警。请求计数按 **gtid 去重**。

> 避免使用：Prometheus

### 业务失败码（Business Failure Code）

后端用于声明业务失败的约定 HTTP 状态码，本框架固定为 **510**。HTTP 200 及除 510 外的其他状态码均视为非业务失败，不计入错误率。

### gtid 去重（gtid Deduplication）

灰度效果统计以单个请求为最小粒度，通过 gtid 保证同一请求在窗口内只被统计一次，避免重试或代理层重复计数。

### 配置与存储（etcd）

etcd 承担双重职责——配置中心（发布策略、灰度规则的动态下发）与度量数据存储（灰度效果统计数据落库）。不采用 Apollo。

> 避免使用：Apollo

### 管理控制台（Console）

内嵌于网关模块的管理界面，用于配置发布策略、查看灰度效果、执行切换与回滚。控制台**只写 etcd**，不直接变更网关运行时状态。

> 避免使用：前端控制台（易与 Web 页面混淆）

### 变更链路（Change Propagation）

所有灰度控制动作的单向链路：内嵌控制台 / CLI → HTTP 调 `grelease-web` 管理 API → 写 etcd → 网关监听 → 应用变更。客户端不直接指令网关。

### 灰度工作流（Canary Workflow）

一次发布的完整生命周期载体，只承载一个目标版本：`matchers`（AND 特征匹配）+ `currentVersion` + `targetVersion`。状态机 `CREATED → RUNNING → PROMOTED ⇄ PAUSED → FINISHED`，任意非终态可 `rollback` 至 `ROLLED_BACK`。应用不声明 `isRelease`，正式版本由每次工作流指定的 `releaseVersion` 在 FINISH 时落盘。

> 避免使用：发布单、流水线

### 确认倒计时（State Timeout）

工作流所有非终态的确认超时（默认 3600s）：客户端必须在倒计时内主动推进到下一状态，否则 etcd lease 到期自动回滚，防止客户端断联后工作流悬挂。由 `DistributedLock.lock(name, callable, waitSeconds, releaseSeconds)` 同一套 etcd CAS + lease 原语支撑。

> 避免使用：心跳超时、会话过期

### grelease-cli

命令行管理工具，基于 picocli，编译为 GraalVM native 二进制；通过 HTTP REST 调用 `grelease-web` 管理接口修改灰度配置，不直接访问 etcd。

### 页面灰度（Page Canary）

SCG 托管/反代 Web 静态资源，按同一套特征匹配规则路由到不同版本的页面构建产物，前后端可独立灰度。

### 部署环境

以 Kubernetes 为主，使用 k8s lifecycle（Deployment 滚动 / Service selector / 探针）承载实例的发布与启停。不引入 OpenResty / nginx 独立 Web 层。

> 避免使用：OpenResty 层
