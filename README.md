# grelease-docs

[grelease](https://github.com/huyiyu/grelease) 的文档站点源码，基于 [OINK](https://oink.pgsty.com/)（Hugo 文档框架）构建。

## 信息架构

- **使用指南**（`content/docs/guide/`）：面向接入 grelease 做灰度发布/蓝绿部署的团队
- **维护者指南**（`content/docs/maintain/`）：面向 grelease 自身的贡献者，含 ADR 归档
- **术语参考**（`content/docs/reference/`）：两侧共用的统一术语表

内容随主仓库 `docs/`、`CONTEXT.md`、`README.md` 逐步迁移，当前为骨架 + 打样阶段（快速开始 1 篇、ADR-0002 1 篇、术语表 1 篇）。

## 本地预览

```bash
hugo server
```

首次运行会拉取 OINK Hugo Module。需要 Git、Go 1.27+、Hugo Extended 0.160.1+，不需要 Node.js/npm。

## 构建

```bash
hugo --gc --minify
```

产物在 `public/`，其中包含站点级 `llms.txt` 与逐页 Markdown 镜像，供 AI 助手直接读取。

## 部署

尚未接入 CI/上线部署（见主仓库 ADR-0014），后续按需启用 GitHub Pages 或 Cloudflare Pages 工作流。
