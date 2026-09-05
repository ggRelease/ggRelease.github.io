---
title: "ADR-0002：Spring Cloud Gateway WebMVC 作为统一入口的单一决策点"
linkTitle: "0002 · SCG WebMVC 单一决策点"
description: >-
  为什么用 Spring Cloud Gateway WebMVC 变体作为统一流量入口的唯一决策点，
  而不是 OpenResty/nginx 分层或 WebFlux 响应式栈。
weight: 20
---

> [!NOTE]
> **状态**：已接受　**日期**：2026-09-01

## 背景

`grelease` 作为灰度发布框架，需要一个统一流量入口。曾有多个候选：引入 OpenResty/nginx 分层处理前端流量、网关用 WebFlux 响应式栈、自建 Netty 网关。

## 决策

- **Spring Cloud Gateway（WebMVC 变体，`spring-cloud-starter-gateway-server-webmvc`）作为最外层统一入口**，承担页面与 API 的统一下发与灰度决策，是唯一决策点。
- **不引入 OpenResty/nginx 独立 Web 层**：页面静态资源由 SCG 直接托管，按同一套特征匹配规则做页面灰度。
- **不引入 WebFlux/响应式**：灰度决策、etcd 读写、510 统计均为阻塞式业务逻辑，WebMVC + Java 25 虚拟线程已提供接近 WebFlux 的性能（官方基准差距 <5%），且团队无响应式经验。

## 备选方案

| 方案 | 放弃原因 |
| --- | --- |
| OpenResty/Lua 分层（入口管前端、SCG 管 API） | 引入额外组件；灰度决策全在网关做，Lua 染色能力用不上 |
| SCG WebFlux + Netty | 响应式学习成本高；本项目的决策/存储逻辑是阻塞式，反而易阻塞事件循环 |
| 自建 Netty 网关 | 需要重造路由、反代、filter 链，复杂度远超收益 |

## 后果

- **优点**：单一决策点、前后端统一灰度、部署简单（一个 Java 应用）、与 Java 25 虚拟线程契合。
- **代价**：WebMVC 变体**不支持 `default-filters` 配置**（详见主仓库 CLAUDE.md），全局染色需在 Servlet Filter 层实现；高并发纯 I/O 转发场景下吞吐略低于 WebFlux（本项目目标为内部/中小流量，可接受）。
