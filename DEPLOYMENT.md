# Deployment Guide

> **最后更新：** 2026-08-17  
> **当前状态：** 生产环境运行中。**后端已恢复推送即自动部署**；infra 仅自动产出变更预览；
> **前端尚未接入自动部署**（有会导致线上白屏的前置缺口，见下节）

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

## CI/CD 现状

每个仓库现在有两类互不干扰的 workflow：

| 仓库 | workflow | 触发 | 作用 |
|------|----------|------|------|
| `stoa-backend` | `deploy-production.yml` | `push` → `main` | **部署**：门禁 → 构建 → 更新 Lambda |
| `stoa-backend` | `deploy.yml` | 仅 `workflow_dispatch` | 无凭证的正式发布验证 DAG，不变更任何资源 |
| `stoa-infra` | `cdk-diff.yml` | `push` → `main` | **只读**变更预览，不部署 |
| `stoa-infra` | `deploy.yml` | 仅 `workflow_dispatch` | staging 资格验证，不变更任何资源 |
| `stoa-frontend` | `frontend-ci.yml` / `deploy.yml` | 仅 `workflow_dispatch` | 验证型，**尚未接入部署** |

`docs/security/phase-474-workflow-policy.json` 里的 `production_mutation: NOT RUN` 只描述
`deploy.yml` 那条正式交付 DAG，不是仓库级承诺；生产变更发生在 `deploy-production.yml`。
该文件的门禁由 `tests/test_production_deploy_workflow_contract.py` 约束。

### 后端：推送 main 即部署

`verify` → `deploy` 两段：

1. `verify`（无任何凭证）：`uv sync --frozen --extra dev` → `ruff check .` → `pytest`。
   测试或 lint 不过就不会进入部署。
2. `deploy`（`environment: production`，OIDC 取 `stoa-github-backend-deploy`）：
   构建 Lambda 包 → 校验 provenance → `--dry-run` 预检 → 更新
   `stoa-api` 与 `stoa-weekly-report` → 等待生效。

构建跑在 **`ubuntu-24.04-arm`** 上。这一点是必需的而非优化：包内是 linux **aarch64** wheel，
在 x64 runner 上 `boot_smoke` 无法导入这些二进制，只能靠 `--skip-smoke` 绕过，等于放弃
「上线前证明 handler 可导入」这项检查。arm64 runner 对公开仓库免费，架构也与 Lambda 运行时一致。

> 2026-07-16 那次部署失败的原因是 `pillow==12.3.0` 当时没有 aarch64 wheel（仓库只到 12.2.0）。
> 该 wheel 已发布，且 `build_lambda_dist.py` 现在带 `manylinux_2_28` 往下的平台兼容阶梯。

### 基础设施：只预览，不部署

`cdk-diff.yml` 在推送后检出 infra 与 backend、构建 Lambda dist、执行 `cdk diff --all`，
把变更面写入 run summary 并上传为 `cdk-diff` artifact。它**不含** `cdk deploy`。

infra 上次真实部署为 2026-06-08，此后积压了 Lambda 别名交付、不可变发布指针、
`GSI-ReviewState`、教师申请公开路由等变更。在人工审阅 diff 之前不要开启自动部署。

### 前端：先别接自动部署

当前 `main` 的 `src/main.tsx` 走 `startWebApplication`，启动时必须取到 `/served-release.json`
与 `/runtime-config.json` 两份合法 JSON，任一失败即渲染「应用暂时无法启动」。而：

- `public/` 里只有 `runtime-config.json.template` 与 `served-release.json.template`，
  vite 会把模板原样拷进 `dist/`，所以 `aws s3 sync dist/` 传上去的是模板而非配置。
- 描述文件要求每个对象带 S3 `versionId`，但线上 bucket **未启用版本控制**
  （CDK `FrontendStack` 已写 `versioned=True`，未部署）。
- **生成这两份描述文件的脚本尚不存在**（`verify-release.mjs` 产出的是校验 receipt）。

线上目前正常，是因为它仍在服务 2026-07-27 的旧构建——那版还不需要这两份文件。
curl `/runtime-config.json` 得到 200 是 CloudFront 的 SPA 回退（返回 `index.html`，
`Content-Type: text/html`），不要据此认为配置已就位。

接入顺序：先部署 infra 开启 bucket 版本控制 → 编写描述文件生成器 → 再接 `push` → 部署。

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
