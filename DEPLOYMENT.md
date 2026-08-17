# Deployment Guide

> **最后更新：** 2026-08-17  
> **当前状态：** 生产环境运行中，但 **CI 不再自动部署**，发布必须手动执行（见下节）

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
| **DynamoDB 表** | `stoa-main`（另有 `stoa-sandbox`） |

---

## CI/CD 现状：只验证，不部署

> 本节于 2026-08-17 依据 workflow 文件与 GitHub Actions 运行记录核实重写。
> 此前版本声称「推送到 main 即自动部署」，该说法已不成立，照此操作会误以为代码已上线。

三个仓库的 workflow 在 2026-07-20 与 2026-07-31 两次改造后，已从「部署流水线」变成
「发布资格验证流水线」：

| 仓库 | workflow | 触发方式 | 是否部署 |
|------|----------|----------|----------|
| `stoa-backend` | Formal Release Verification | 仅 `workflow_dispatch` | 否 |
| `stoa-frontend` | Formal Release Verification / Frontend Staging Eligibility | 仅 `workflow_dispatch` | 否 |
| `stoa-infra` | Infrastructure Staging Eligibility | 仅 `workflow_dispatch` | 否 |

关键事实：

- **没有 `push` / `pull_request` 触发器**，推送 `main` 不会启动任何 workflow。
- staging 相关 job 只校验产物摘要并执行 `release_environment.py --help`，**不调用 AWS 变更 API**。
- 流水线末尾显式输出 `production-infrastructure/deploy/smoke/rollback = NOT RUN`。
- 后端最后一次真实部署尝试为 2026-07-16（旧版含 `Update Lambda function code` 的 workflow），
  **在构建 Lambda 包步骤失败**，未完成部署；此后 workflow 即被替换为验证型。

因此当前**任何发布都必须按下节手动执行**。OIDC 角色（`stoa-github-*-deploy`）仍然存在，
但已无 workflow 使用它们。

### 本地验证的环境要求

Lambda 构建与 CDK synth 都硬性要求 **Python 3.12**（CI 用 3.12.13）。在 3.13 下
`scripts/build_lambda_dist.py` 会直接失败，并连带使 `test_lambda_dist_build.py` 全部报错：

```bash
uv python install 3.12
UV_PROJECT_ENVIRONMENT=.venv312 uv run --python 3.12 --extra dev pytest -q
```

在 macOS 上构建属交叉编译（dist 内为 linux aarch64 wheel），本机无法导入，需加
`--skip-smoke`；也因此 **CDK synth 无法在 macOS 上完成**（provenance 守卫会运行 boot smoke）。

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

# 重建 Lambda 包。必须走此脚本而不是裸 pip：它固定 linux aarch64 wheel、
# 产出 .stoa-build-manifest.json，而 CDK synth 的 provenance 守卫会校验该 manifest。
# macOS 上属交叉编译，本机无法导入 linux 二进制，故需 --skip-smoke（CI always runs smoke）。
UV_PROJECT_ENVIRONMENT=.venv312 uv run --python 3.12 \
  python scripts/build_lambda_dist.py --repo-root . --dist dist --skip-smoke

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

## 教师邀请制上线前置条件

> 核实时间 2026-08-17。前后端与 infra 代码均已入库，但下列外部条件未满足前，
> 邀请制在生产环境无法走通。

代码侧（已提交，**待部署**）：

| 项 | 仓库 | 说明 |
|----|------|------|
| `claim` / 审核队列 / 状态查询 / 重发邀请 | `stoa-backend` | approve 之后的激活闭环 |
| `GSI-ReviewState` | `stoa-infra` | 审核队列查询依赖；生产表当前**尚无**此索引 |
| 三个免授权网关路由 | `stoa-infra` | 见下 |

网关路由：候选人在申请、查状态、领取邀请时都还没有身份，这三个请求必须免 JWT。
生产网关此前只有 `/auth/*` 与 `/health` 免授权，其余落入 `ANY /{proxy+}`（JWT），
因此这三个请求都会得到 `401`：

- `POST /teacher-applications`
- `GET /teacher-applications/{application_id}/status`
- `POST /teacher-applications/activation/claim`

`POST /teacher-applications/activation/consume` 由已登录用户调用，留在 JWT 之后即可。
注意 `GET /teacher-applications` 是审核队列，**必须保持在 JWT 之后**，
所以这三条路由是按路径单独声明方法的，不能沿用 `/auth/*` 那种 POST+GET 批量写法。

运维侧（需人工处理，代码无法覆盖）：

1. **SES 仍在沙箱**：`ProductionAccessEnabled = false`，配额 200/日、1/秒。
   域名 `stoaedu.ch` 已验证（发件地址 `noreply@stoaedu.ch` 可用），但沙箱状态下
   **只能发给已验证地址**，候选人邮箱（如 gmail）会被拒。邀请信发送失败时
   审核响应返回 `invitationDelivered: false`，候选人拿不到链接。
   → 需向 AWS 申请解除沙箱限制。
2. **无人持有 `teacher_identity_reviewer`**：生产表中现存的 capability 授权只有一条
   `platform_operations_reader`。没有该权限则无法审核申请，也无法重发邀请。
   通过 `POST /admin/privileged-identities/{id}/capabilities` 授予需要调用方先持有
   `admin_identity_manager`，而当前同样无人持有 → 首次授权需要直接写库引导。
3. **`app_base_url`**：默认 `https://app.stoaedu.ch`，邀请链接形如
   `{app_base_url}/teacher-activate?token=...`，需与前端实际域名一致。

Lambda 执行角色权限已具备，无需改动：`cognito-idp:AdminCreateUser` /
`AdminSetUserPassword` / `AdminGetUser` / `AdminAddUserToGroup`、`ses:SendEmail` /
`SendRawEmail`。

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
