---
title: 快速开始
linkTitle: 快速开始
description: grelease 是什么、解决什么问题、核心架构与模块划分一览。
weight: 10
---

grelease 是一个**灰度发布前后端一站式框架**，以 Spring Cloud Gateway 为统一流量入口，提供灰度发布与蓝绿部署的规则配置、执行与度量，覆盖页面与 API 两种流量。

## 特性

- **统一入口**：Spring Cloud Gateway（WebMVC 变体）作为最外层单一决策点，页面与 API 统一下发
- **灰度发布**：基于请求特征（header/参数/标签）匹配目标版本，放量速率按请求量步进控制
- **蓝绿部署**：与灰度串行编排（先灰度观察，后蓝绿收尾），一键切换与回滚
- **页面灰度**：SCG 直接托管 Web 静态资源，前后端可独立灰度
- **灰度上下文传播**：网关染色后，应用依据配置沿服务调用链传递灰度上下文
- **健康保护**：后端返回约定错误码（如 510）即业务失败，错误率超阈值全自动停止放量
- **内嵌控制台**：配置发布策略、查看灰度效果、执行切换与回滚
- **CLI**：通过命令修改灰度配置（picocli + x509/mTLS 认证）
{.cards}

## 架构

```text
客户端
  │
  ▼
grelease-web ── SCG(WebMVC) ──特征匹配灰度决策──► 后端服务 v1 / v2
  │   │            │                          └─► 页面构建产物 v1 / v2
  │   │            ▼
  │   │   grelease-core（染色 / 放量 / 错误率判定）
  │   │            │
  │   ▼            ▼
  │  grelease-config-loader      grelease-metrics（Micrometer 采集）
  │   │   (ConfigDataLoader)          │
  │   │     ▲ 监听/推送               │
  │   │     │        ┌───────────────┘
  │   ▼     ▼        ▼
  │   └──► etcd ◄── 度量数据落库
  │
  │  写 etcd 侧（与控制台同链路，不直接调网关）：
  │    内嵌管理控制台 ─(写)─► etcd ─(监听)─► 网关应用变更
  │    grelease-cli（picocli + x509/mTLS）─(写)─► etcd
  │    读侧：控制台 / CLI ◄── etcd 灰度效果 / 趋势
```

> [!IMPORTANT]
> 控制台与 CLI 只写 etcd，网关只监听 etcd，二者之间无直接调用——这是保证网关与配置解耦、变更可审计、多实例一致的关键约束。详见[维护者指南](/docs/maintain/)。

## 模块

| 模块 | 职责 |
| --- | --- |
| `grelease-core` | 灰度发布核心：路由装配、灰度决策、染色、放量速率控制、错误率判定 |
| `grelease-config-loader` | etcd 配置加载：Spring Boot `ConfigDataLoader` 接入 `Environment` + etcd 读写/监听 client API |
| `grelease-metrics` | 灰度度量：Micrometer 采集，按版本分桶、时间窗口留存，写入 etcd |
| `grelease-web` | Web 装配：SCG WebMVC 启动装配、全局染色 Servlet Filter、内嵌管理控制台 |
| `grelease-cli` | 命令行管理工具：picocli + x509/mTLS 认证，命令方式修改灰度配置 |

## 技术栈

- Java 25（虚拟线程） · Gradle 多模块（Version Catalog + platform BOM）
- Spring Boot 4.0.x · Spring Cloud 2025.1.x (Oakwood) · Spring Cloud Gateway 5.0.x (WebMVC)
- etcd（配置 + 度量存储） · Micrometer · picocli
- 部署：Kubernetes（k8s lifecycle）

## 下一步

- 命令行怎么用 → CLI 章节（编写中）
- 控制台怎么操作 → Console 章节（编写中）
- Admin API 有哪些接口 → API 契约章节（编写中）
- 怎么部署到 k8s → 部署章节（编写中）
- 不熟悉某个术语 → [术语参考](/docs/reference/glossary/)

源码与最新进展见 [huyiyu/grelease](https://github.com/huyiyu/grelease)。
