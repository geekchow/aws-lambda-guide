# AWS Lambda — A Mental-Model Guide / 心智模型学习指南

A learning guide to AWS (Amazon Web Services) Lambda, built from first principles: why Lambda exists, how one invocation travels through the service, and how **versions**, **aliases**, and **provisioned concurrency** fit in. It follows one path (Why → What → Concept map → Running example → Deep dives → Full walkthrough), and one running example ties it together: the `checkout` function of an online shop, ShopFast, during Black Friday week.

一份从第一性原理出发的 AWS（Amazon Web Services，亚马逊云服务）Lambda 学习指南：Lambda 为什么存在、一次调用如何在服务内部流转，以及**版本**、**别名**和**预置并发**如何嵌入其中。全篇沿"为什么 → 是什么 → 概念地图 → 贯穿示例 → 组件深入 → 完整走查"展开，并以网店 ShopFast 的 `checkout` 函数在黑五一周的经历贯穿始终。

| Language | Start here |
|---|---|
| English | [aws-lambda-guide/00-overview.md](aws-lambda-guide/00-overview.md) |
| 中文 | [aws-lambda-guide-cn/00-overview.md](aws-lambda-guide-cn/00-overview.md) |

## Contents / 目录

| # | English | 中文 |
|---|---|---|
| 00 | [Overview & mindmap](aws-lambda-guide/00-overview.md) | [总览与思维导图](aws-lambda-guide-cn/00-overview.md) |
| 01 | [Why](aws-lambda-guide/01-why.md) | [为什么](aws-lambda-guide-cn/01-why.md) |
| 02 | [What](aws-lambda-guide/02-what.md) | [是什么](aws-lambda-guide-cn/02-what.md) |
| 03 | [Concept map](aws-lambda-guide/03-concept-map.md) | [概念地图](aws-lambda-guide-cn/03-concept-map.md) |
| 04 | [Running example](aws-lambda-guide/04-running-example.md) | [贯穿示例](aws-lambda-guide-cn/04-running-example.md) |
| 05.1 | [Versions](aws-lambda-guide/05-deep-dives/01-versions.md) | [版本](aws-lambda-guide-cn/05-deep-dives/01-versions.md) |
| 05.2 | [Aliases](aws-lambda-guide/05-deep-dives/02-aliases.md) | [别名](aws-lambda-guide-cn/05-deep-dives/02-aliases.md) |
| 05.3 | [Provisioned concurrency](aws-lambda-guide/05-deep-dives/03-provisioned-concurrency.md) | [预置并发](aws-lambda-guide-cn/05-deep-dives/03-provisioned-concurrency.md) |
| 05.4 | [Invoke Frontend & concurrency](aws-lambda-guide/05-deep-dives/04-invoke-frontend-and-concurrency.md) | [调用前端与并发](aws-lambda-guide-cn/05-deep-dives/04-invoke-frontend-and-concurrency.md) |
| 05.5 | [Assignment & Placement](aws-lambda-guide/05-deep-dives/05-assignment-and-placement.md) | [分配与放置](aws-lambda-guide-cn/05-deep-dives/05-assignment-and-placement.md) |
| 05.6 | [Execution environment](aws-lambda-guide/05-deep-dives/06-execution-environment.md) | [执行环境](aws-lambda-guide-cn/05-deep-dives/06-execution-environment.md) |
| 06 | [Full walkthrough](aws-lambda-guide/06-walkthrough.md) | [完整走查](aws-lambda-guide-cn/06-walkthrough.md) |
| 07 | [Next steps & lab](aws-lambda-guide/07-next-steps.md) | [自测与动手实验](aws-lambda-guide-cn/07-next-steps.md) |

Diagrams render natively on GitHub (Mermaid). Architecture swim-lane sources: `diagrams/lambda-architecture.drawio` in each language folder (open with [diagrams.net](https://app.diagrams.net)).

## Notes / 说明

- Internal service names (Invoke Frontend, Counting, Assignment, Placement) come from AWS's public re:Invent architecture talks — a mental model, not a public API contract.
- Prices in the provisioned-concurrency chapter are illustrative `us-east-1` figures; check the [AWS Lambda pricing page](https://aws.amazon.com/lambda/pricing/).
- 内部服务名称（调用前端、计数、分配、放置）来自 AWS 公开的 re:Invent 架构演讲，是心智模型而非公开 API 契约；预置并发章节中的价格仅作示意，请以 AWS 定价页为准。
