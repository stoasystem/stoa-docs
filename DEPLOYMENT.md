# Deployment Guide

> **最后更新：** 2026-05-28  
> **当前状态：** 生产环境已部署，CI/CD 已配置

---

## 生产环境现状

| 资源 | 值 |
|------|-----|
| **前端 URL** | https://app.stoaedu.ch |
| **API URL** | https://api.stoaedu.ch |
| **AWS Region** | eu-central-2（苏黎世） |
| **AWS Account** | 562923011260 |
| **Lambda 函数** | `stoa-api` |
| **S3 前端 Bucket** | `stoa-frontend-562923011260` |
| **CloudFront（前端）** | `E27CVAMQHDMW80` → `d1hq27o785lonk.cloudfront.net` |
| **API Gateway** | `vkuxk2gbue` → `https://vkuxk2gbue.execute-api.eu-central-2.amazonaws.com` |
| **Cognito User Pool** | `eu-central-2_Ss93YQzjJ` |
| **DynamoDB 表** | `stoa-table` |

---

## CI/CD 自动部署（推荐）

所有仓库已配置 **GitHub Actions + AWS OIDC**，推送到 `main` 分支即自动部署：

| 仓库 | 触发 | 流程 | 耗时 |
|------|------|------|------|
| `stoa-frontend` | push → main | lint → build → S3 sync → CloudFront invalidation | ~1 min |
| `stoa-backend` | push → main | pip install (linux arm64) → lambda update-function-code | ~1 min |
| `stoa-infra` | push → main | CDK diff → CDK deploy --all | ~6 min |
| `stoa-infra` | PR → main | CDK diff（结果贴在 PR 评论） | ~3 min |

**无需手动配置 AWS 凭证**，GitHub Actions 通过 OIDC 临时 Token 认证，IAM 角色：
- `stoa-github-frontend-deploy`（S3 + CloudFront 只读写权限）
- `stoa-github-backend-deploy`（Lambda update 权限）
- `stoa-github-infra-deploy`（CDK PowerUser 权限）

---

## 手动部署（本地调试用）

### Prerequisites

```bash
# 必须工具
aws --version          # AWS CLI v2
uv --version           # Python 包管理
node --version         # Node.js 20+
npx cdk --version      # CDK CLI（无需全局安装）
```

### 1. 验证 AWS 身份

```bash
aws sts get-caller-identity
# 应返回 Account: 562923011260, Region: eu-central-2
```

### 2. 手动部署前端

```bash
cd stoa-frontend
npm install
npm run build   # 读取 .env.production（已配置 VITE_API_URL=https://api.stoaedu.ch）

# 上传到 S3
aws s3 sync dist/ s3://stoa-frontend-562923011260/ \
  --region eu-central-2 --delete \
  --exclude "index.html" \
  --cache-control "public,max-age=31536000,immutable"

aws s3 cp dist/index.html s3://stoa-frontend-562923011260/index.html \
  --region eu-central-2 \
  --cache-control "no-cache,no-store,must-revalidate"

# 清除 CloudFront 缓存
aws cloudfront create-invalidation \
  --distribution-id E27CVAMQHDMW80 --paths "/*"
```

### 3. 手动部署后端

```bash
cd stoa-backend

# 重建 Lambda 包（必须用 linux arm64 wheels）
rm -rf dist
pip install \
  --platform manylinux2014_aarch64 \
  --implementation cp \
  --python-version 3.12 \
  --only-binary :all: \
  --target dist \
  -r requirements.txt
cp -r src/stoa dist/stoa

# 打包并更新 Lambda
cd dist && zip -r ../lambda.zip . -q && cd ..
aws lambda update-function-code \
  --function-name stoa-api \
  --zip-file fileb://lambda.zip \
  --region eu-central-2

aws lambda wait function-updated --function-name stoa-api --region eu-central-2
```

### 4. 手动部署基础设施（CDK）

```bash
cd stoa-infra
uv sync

# 查看变更预览
npx cdk diff --all

# 部署全部 Stack
npx cdk deploy --all --require-approval never
```

CDK Stack 部署顺序（依赖关系自动处理）：
```
StoaAuthStack
  ↓
StoaDatabaseStack + StoaStorageStack（并行）
  ↓
StoaNotificationStack
  ↓
StoaApiStack
  ↓
StoaFrontendStack
```

### 5. 验证

```bash
# API 健康检查
curl https://api.stoaedu.ch/health
# → {"status":"ok","version":"0.1.0"}

# 前端可访问
curl -sI https://app.stoaedu.ch | grep HTTP
# → HTTP/2 200
```

---

## 从零搭建（新环境）

> 仅在需要全新 AWS 账号部署时参考。

```bash
# 1. CDK Bootstrap
cd stoa-infra
uv sync
aws sts get-caller-identity
npx cdk bootstrap aws://ACCOUNT_ID/eu-central-2

# 2. 部署所有 Stack
npx cdk deploy --all --require-approval never

# 3. 申请 *.stoaedu.ch 通配符证书
#    us-east-1（CloudFront 用）
aws acm request-certificate \
  --domain-name "*.stoaedu.ch" \
  --subject-alternative-names "stoaedu.ch" \
  --validation-method DNS --region us-east-1

#    eu-central-2（API GW 用）
aws acm request-certificate \
  --domain-name "*.stoaedu.ch" \
  --subject-alternative-names "stoaedu.ch" \
  --validation-method DNS --region eu-central-2

# 4. 添加 Route 53 DNS 验证 CNAME，等待证书 ISSUED

# 5. 创建 API GW 自定义域名
aws apigatewayv2 create-domain-name \
  --domain-name api.stoaedu.ch \
  --domain-name-configurations \
    CertificateArn=ACM_ARN_EU,EndpointType=REGIONAL,SecurityPolicy=TLS_1_2 \
  --region eu-central-2

aws apigatewayv2 create-api-mapping \
  --domain-name api.stoaedu.ch \
  --api-id API_GW_ID --stage '$default' --region eu-central-2

# 6. 更新 FrontendStack 的 WILDCARD_CERT_ARN_US，重新 cdk deploy StoaFrontendStack

# 7. 添加 Route 53 ALIAS 记录（app/api 两条）
```

---

## 环境变量参考

### 后端（Lambda 环境变量）

| 变量 | 说明 |
|------|------|
| `COGNITO_USER_POOL_ID` | Cognito User Pool ID |
| `COGNITO_STUDENT_CLIENT_ID` | 学生 App Client |
| `COGNITO_PARENT_CLIENT_ID` | 家长 App Client |
| `COGNITO_TEACHER_CLIENT_ID` | 老师 App Client |
| `COGNITO_ADMIN_CLIENT_ID` | 管理员 App Client |
| `DYNAMODB_TABLE_NAME` | DynamoDB 表名 |
| `S3_IMAGES_BUCKET` | 图片上传 Bucket |
| `S3_REPORTS_BUCKET` | 报告存储 Bucket |
| `SQS_TEACHER_QUEUE_URL` | 老师升级队列 URL |
| `BEDROCK_MODEL_ID` | Bedrock 模型 ID |

### 前端（Vite 环境变量，`.env.production`）

| 变量 | 当前值 |
|------|--------|
| `VITE_API_URL` | `https://api.stoaedu.ch` |
| `VITE_COGNITO_USER_POOL_ID` | `eu-central-2_Ss93YQzjJ` |
| `VITE_COGNITO_CLIENT_ID` | `5cq57qrgo94ivseotslhajrlh3`（student client） |
| `VITE_APP_ENV` | `production` |

> `.env.production` 不提交到 Git（.gitignore）。CI/CD 直接将变量硬编码在 workflow env 中。
