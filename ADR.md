# Architecture Decision Records

## ADR-001: HTTP API over REST API (API Gateway)

**Status:** Accepted  
**Date:** 2026-05-23  
**Context:** Choose API Gateway type for the backend.  
**Decision:** Use HTTP API (not REST API).  
**Rationale:** 70% cheaper, lower latency, native Cognito JWT authorizer support. REST API features (request transformation, Usage Plans) not needed for MVP.

---

## ADR-002: FastAPI Monolith on Single Lambda (Mangum)

**Status:** Accepted  
**Date:** 2026-05-23  
**Context:** Deployment model for the Python backend.  
**Decision:** Single Lambda function with FastAPI + Mangum adapter.  
**Rationale:** Simpler deployment, centralized logging, easier local dev. Split to per-route Lambdas in Phase 2 only if specific routes exceed timeout limits.

---

## ADR-003: DynamoDB Single-Table Design

**Status:** Accepted  
**Date:** 2026-05-23  
**Context:** Database choice and schema design.  
**Decision:** DynamoDB with single-table design, 4 GSIs, On-Demand billing.  
**Rationale:** No idle cost, auto-scales, covers all access patterns via PK/SK + GSIs. Relational DB (RDS) adds unnecessary complexity and fixed cost for MVP scale.

---

## ADR-004: S3 Presigned URL for Image Upload

**Status:** Accepted  
**Date:** 2026-05-23  
**Context:** Homework image upload from students.  
**Decision:** Client obtains presigned PUT URL from API, uploads directly to S3.  
**Rationale:** Bypasses Lambda 6 MB payload limit and 29s timeout. Reduces Lambda cost and latency.

---

## ADR-005: Bedrock Claude Haiku (not Sonnet)

**Status:** Accepted  
**Date:** 2026-05-23  
**Context:** AI model selection for tutoring responses.  
**Decision:** Use `claude-haiku` as default model.  
**Rationale:** ~5× cheaper than Claude Sonnet, sufficient quality for structured step-by-step math explanations. Upgrade path to Sonnet available for Premium tier.

---

## ADR-006: 所有 Stack 统一部署在 eu-central-2

**Status:** Accepted  
**Date:** 2026-05-23  
**Context:** CloudFront + S3 FrontendStack 的部署 Region 选择。  
**Decision:** FrontendStack 与其他所有 Stack 统一部署在 eu-central-2（苏黎世）。  
**Rationale:** 所有资源集中在同一 Region，减少运维复杂度，符合 nDSG 数据驻留要求。CloudFront 作为全球边缘网络，S3 源站在 eu-central-2 对全球访问速度影响可忽略。ACM 证书在 us-east-1 单独创建并在 CloudFront 中引用，无需迁移整个 Stack。

---

## ADR-007: 自定义域名通过 Route 53 + ACM 通配符证书实现

**Status:** Accepted  
**Date:** 2026-05-28  
**Context:** 需要通过 `app.stoaedu.ch` / `api.stoaedu.ch` 自定义域名访问前端和 API，而非 CloudFront/API GW 默认域名。  
**Decision:** 申请 `*.stoaedu.ch` 通配符 ACM 证书（us-east-1 用于 CloudFront，eu-central-2 用于 API GW），在 Route 53 添加 ALIAS 记录指向 CloudFront 和 API GW Regional 端点。  
**Rationale:**
- 通配符证书一次申请覆盖所有子域名，无需为每个子域名单独申请
- stoaedu.ch 的 NS 已迁移至 AWS Route 53，DNS 管理完全在 AWS 内完成
- Route 53 ALIAS 记录（非 CNAME）可用于 Apex 域名，且不产生额外 DNS 查询延迟
- 现有 Route 53 已有 DNS 验证 CNAME，新通配符证书秒级完成验证

---

## ADR-008: GitHub Actions + AWS OIDC 无密钥 CI/CD

**Status:** Accepted  
**Date:** 2026-05-28  
**Context:** 需要自动化三个仓库（stoa-frontend / stoa-backend / stoa-infra）的 AWS 部署流程。  
**Decision:** 使用 GitHub Actions + AWS IAM OIDC（Web Identity Federation），不存储长期 AWS 访问密钥。每个仓库对应一个最小权限 IAM 角色。  
**Rationale:**
- OIDC 临时 Token 自动轮换，无需手动管理密钥过期
- 各角色权限最小化：frontend 只能写 S3 + 创建 CloudFront invalidation，backend 只能更新 Lambda，infra 才有 CDK 所需的 PowerUser 权限
- AWS IAM OIDC Provider `token.actions.githubusercontent.com` 已存在于账号中
- 相比 IAM User + Access Key，减少了密钥泄露的攻击面

**实施细节：**
- Trust Policy 限制 `sub` 为 `repo:stoasystem/*:ref:refs/heads/main`，仅 main 分支可触发部署
- infra workflow 同时克隆 stoa-backend 以构建 Lambda dist（CDK 引用 `../stoa-backend/dist`）
- PR 触发时只运行 `cdk diff` 并将结果贴在 PR 评论，push to main 才真正执行 `cdk deploy`

---

## ADR-009: Lambda 部署包采用 manylinux2014_aarch64 预编译 Wheels

**Status:** Accepted  
**Date:** 2026-05-28  
**Context:** Lambda 运行环境为 Linux ARM64（Amazon Linux 2023），本地开发环境为 macOS。`pydantic-core`、`cryptography` 等含 C 扩展的包在 macOS 编译后无法在 Lambda 运行。  
**Decision:** 在打包 Lambda dist 时，使用 `pip install --platform manylinux2014_aarch64 --implementation cp --python-version 3.12 --only-binary :all:` 强制拉取 Linux ARM64 预编译 wheel。  
**Rationale:**
- 避免依赖 Docker 进行交叉编译（Docker 在 CI 和本地均增加复杂度）
- manylinux2014 是 AWS Lambda 支持的标准 Linux ABI，兼容性好
- `--only-binary :all:` 确保不会回退到源码编译，若无对应 wheel 则报错（快速失败，而非静默错误）
