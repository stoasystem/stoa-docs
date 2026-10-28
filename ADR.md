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
**Revision (2026-10-28, stoa-backend#136):** 大文件按 S3 分片上传：客户端为每一片向 API 领取一个预签名 PUT URL，签名绑定这一片的 SHA-256，由 S3 校验内容；完成上传时，服务器用 `list_parts` 核对各片。473-08 曾把分片改为经 Lambda 转发，以免 S3 坐标离开服务器；但 5 MiB 的分片经 base64 编码后约 7 MB，超过 Lambda 6 MB 的上限，而 S3 又要求除最后一片外每片至少 5 MiB，分片无法再缩小。因此有意撤销 473-08：预签名 URL 里会出现桶名（含 AWS 账号号）、暂存键和分片的 uploadId。完成、恢复、清理以及审核仍在服务器端进行。

---

## ADR-005: Bedrock Claude Haiku (not Sonnet)

**Status:** Superseded by ADR-011（2026-10-28）  
**Date:** 2026-05-23  
**Context:** AI model selection for tutoring responses.  
**Decision:** Use `claude-haiku` as default model.  
**Rationale:** ~5× cheaper than Claude Sonnet, sufficient quality for structured step-by-step math explanations. Upgrade path to Sonnet available for Premium tier.

---

## ADR-006: 所有 Stack 统一部署在 eu-central-2

**Status:** Accepted  
**Amended by:** ADR-010（AI 回答与图片审核在欧盟处理的例外）  
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

---

## ADR-010: 欧盟处理的例外（修订 ADR-006）

**Status:** Accepted  
**Date:** 2026-10-28  
**Context:** ADR-006 把所有资源部署在 eu-central-2（苏黎世），以符合 nDSG 的数据驻留要求。有两项服务做不到只在苏黎世处理：Bedrock 上的 Claude Sonnet 4.6 在 eu-central-2 不能单区调用，只能走推理配置；Amazon Rekognition 在 eu-central-2 不提供服务。  
**Decision:**
- 存储不变：DynamoDB、S3、日志等所有持久数据只放在 eu-central-2。
- AI 回答通过欧盟推理配置 `eu.anthropic.claude-sonnet-4-6` 调用。请求内容（学生的文字、对话上下文、学习会话里的图片）可能在这些欧盟区域处理：法兰克福、苏黎世、斯德哥尔摩、米兰、西班牙、爱尔兰、巴黎。
- 图片审核调用 eu-central-1（法兰克福）的 Rekognition DetectModerationLabels，只发送图片本身。
- 不得使用 `global.` 推理配置，也不得使用欧盟/EEA 以外的区域；要放开，须另立 ADR。
- 前提：AWS Organizations 的 AI services opt-out 策略对所有服务生效，Bedrock 账户的 `data_retention_mode` 为 `none`（stoa-backend#153）。
- 更换模型或推理配置时，从 eu-central-2 调 `GetInferenceProfile` 核对目标区域列表。

**Rationale:**
- 瑞士联邦委员会认定欧盟/EEA 国家的数据保护水平足够（DSV 附录 1），按 nDSG 向这些国家披露数据不需要额外保障。
- 这些数据只在处理过程中经过欧盟，处理完不落盘：Bedrock 不用输入输出训练模型，不交给模型提供方，且账户已关闭留存；Rekognition 在 opt-out 后不再为改进服务保存图片。
- 欧盟推理配置的目标区域列表是固定的，数据走 AWS 内网传输；CloudTrail 在苏黎世记录每次请求实际在哪个区域处理（`additionalEventData.inferenceRegion`）。
- 隐私说明同步写明处理地点。

---

## ADR-011: 通过欧盟推理配置使用 Claude Sonnet 4.6（取代 ADR-005）

**Status:** Accepted  
**Date:** 2026-10-28  
**Context:** ADR-005 选 Haiku 作默认模型，理由是便宜。实际配置早已是 `eu.anthropic.claude-sonnet-4-6`（stoa-backend `config.py` 的 `bedrock_model_id`）。Ask 要直接看学生的图片（stoa-backend#135），讲题质量是产品的核心。  
**Decision:** 默认模型用 Claude Sonnet 4.6，经欧盟推理配置 `eu.anthropic.claude-sonnet-4-6` 调用，处理地点见 ADR-010。  
**Rationale:**
- 多模态：能直接读作业照片，不必先做文字识别。
- 分步讲解的质量比 Haiku 好。
- eu-central-2 不能单区调用，推理配置是唯一的调用方式。
- 成本高于 Haiku，靠每日额度控制（免费 5 题、标准 30 题、高级 100 题）。
- 换模型前，先用离线评测（卡 078）确认看图和讲题的效果。
