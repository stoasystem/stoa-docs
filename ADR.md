# Architecture Decision Records

## ADR-001: HTTP API over REST API (API Gateway)

**Status:** Accepted  
**Context:** Choose API Gateway type for the backend.  
**Decision:** Use HTTP API (not REST API).  
**Rationale:** 70% cheaper, lower latency, native Cognito JWT authorizer support. REST API features (request transformation, Usage Plans) not needed for MVP.

## ADR-002: FastAPI Monolith on Single Lambda (Mangum)

**Status:** Accepted  
**Context:** Deployment model for the Python backend.  
**Decision:** Single Lambda function with FastAPI + Mangum adapter.  
**Rationale:** Simpler deployment, centralized logging, easier local dev. Split to per-route Lambdas in Phase 2 only if specific routes exceed timeout limits.

## ADR-003: DynamoDB Single-Table Design

**Status:** Accepted  
**Context:** Database choice and schema design.  
**Decision:** DynamoDB with single-table design, 4 GSIs, On-Demand billing.  
**Rationale:** No idle cost, auto-scales, covers all access patterns via PK/SK + GSIs. Relational DB (RDS) adds unnecessary complexity and fixed cost for MVP scale.

## ADR-004: S3 Presigned URL for Image Upload

**Status:** Accepted  
**Context:** Homework image upload from students.  
**Decision:** Client obtains presigned PUT URL from API, uploads directly to S3.  
**Rationale:** Bypasses Lambda 6 MB payload limit and 29s timeout. Reduces Lambda cost and latency.

## ADR-005: Bedrock Claude Haiku (not Sonnet)

**Status:** Accepted  
**Context:** AI model selection for tutoring responses.  
**Decision:** Use `claude-haiku` as default model.  
**Rationale:** ~5× cheaper than Claude Sonnet, sufficient quality for structured step-by-step math explanations. Upgrade path to Sonnet available for Premium tier.

## ADR-006: 所有 Stack 统一部署在 eu-central-2

**Status:** Accepted  
**Context:** CloudFront + S3 FrontendStack 的部署 Region 选择。  
**Decision:** FrontendStack 与其他所有 Stack 统一部署在 eu-central-2（苏黎世）。  
**Rationale:** 所有资源集中在同一 Region，减少运维复杂度，符合 nDSG 数据驻留要求。CloudFront 作为全球边缘网络，S3 源站在 eu-central-2 对全球访问速度影响可忽略。如未来需要绑定自定义域名（stoa.ch）+ ACM 证书，可在 us-east-1 单独创建一张证书并在 CloudFront 中引用，无需迁移整个 Stack。
