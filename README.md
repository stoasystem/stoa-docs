# stoa-docs

STOA 学习平台设计与开发文档中心。

## 🌐 线上地址

| 地址 | 说明 |
|------|------|
| **https://app.stoaedu.ch** | 前端 React SPA（生产环境） |
| **https://api.stoaedu.ch** | 后端 FastAPI（生产环境） |
| **https://api.stoaedu.ch/health** | API 健康检查 |

## 文档目录

| 文件 | 描述 | 最后更新 |
|------|------|---------|
| [PRD.md](./PRD.md) | 产品需求文档 — 用户角色、功能需求、成功指标 | 2026-05-23 |
| [HLD.md](./HLD.md) | 高层设计文档 — 架构、API、数据模型、AWS 费用 | 2026-05-23 |
| [PLAN.md](./PLAN.md) | 开发计划 — Sprint 规划、当前进度快照、任务拆解 | 2026-05-28 |
| [ADR.md](./ADR.md) | 架构决策记录（9 条） | 2026-05-28 |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | AWS 部署指南（CI/CD + 手动部署） | 2026-05-28 |

## 代码仓库

| 仓库 | 说明 | CI/CD |
|------|------|-------|
| [stoasystem/stoa-backend](https://github.com/stoasystem/stoa-backend) | Python FastAPI + Lambda | push → Lambda update |
| [stoasystem/stoa-frontend](https://github.com/stoasystem/stoa-frontend) | React 19 + TypeScript | push → S3 + CloudFront |
| [stoasystem/stoa-infra](https://github.com/stoasystem/stoa-infra) | AWS CDK v2 (Python) | push → CDK deploy |

## 技术栈

```
前端:  React 19 · TypeScript · Vite · Zustand · TanStack Query · AWS Amplify JS
后端:  Python 3.12 · FastAPI · Mangum · pydantic-settings · boto3
基础设施: AWS CDK v2 · Lambda · API Gateway HTTP API · DynamoDB · S3 · CloudFront
         Cognito · Bedrock (Claude Haiku) · Rekognition · SQS FIFO · SES
Region: eu-central-2（苏黎世）
```

## 在线文档（Lark）

- [PRD（原始）](https://netoceanai.sg.larksuite.com/docx/GemMduD6moFtd0xSQtbl2JYJgBb)
- [HLD（Lark 版）](https://netoceanai.sg.larksuite.com/docx/GXuQdO9UEoRRPmxJuYllhEWRg1J)

## 快速开始

1. 阅读 [PRD.md](./PRD.md) 了解产品背景和需求
2. 阅读 [HLD.md](./HLD.md) 了解技术架构
3. 阅读 [PLAN.md](./PLAN.md) 了解当前进度和下一步任务
4. 参考 [DEPLOYMENT.md](./DEPLOYMENT.md) 了解 CI/CD 自动部署流程
