# STOA — 高层设计文档（HLD）

> **版本：** v1.0  
> **日期：** 2026-05-23  
> **原始文档：** [Lark HLD](https://netoceanai.sg.larksuite.com/docx/GXuQdO9UEoRRPmxJuYllhEWRg1J)  
> **状态：** 已评审

---

## 目录

1. [系统架构概览](#一系统架构概览)
2. [功能模块](#二功能模块)
3. [API 接口清单](#三api-接口清单)
4. [数据模型](#四数据模型)
5. [AWS 服务及费用估算](#五aws-服务及费用估算)
6. [后端架构](#六后端架构)
7. [前端架构](#七前端架构)
8. [CDK 基础设施架构](#八cdk-基础设施架构)
9. [文件夹结构](#九文件夹结构)

---

## 一、系统架构概览

```
┌─────────────┐    ┌──────────────────────────────────────────────────────┐
│   学生/家长  │    │                    AWS Cloud (eu-central-2)           │
│   老师/管理  │    │                                                      │
│   [Browser] │    │  CloudFront ──→ S3 (SPA)                             │
└──────┬──────┘    │                                                      │
       │ HTTPS      │  API Gateway (HTTP API)                              │
       │            │    │                                                │
       │            │    ↓                                                │
       │            │  Lambda (FastAPI/Mangum)                            │
       │            │    │                                                │
       │            │    ├──→ Cognito (Auth)                              │
       │            │    ├──→ DynamoDB (Data)                             │
       │            │    ├──→ S3 (Images/Reports)                         │
       │            │    ├──→ Bedrock (AI - Claude Haiku)                 │
       │            │    ├──→ Rekognition (OCR)                           │
       │            │    ├──→ SQS FIFO (Teacher Queue)                    │
       │            │    └──→ SES (Email)                                 │
       │            │                                                      │
       │            │  EventBridge Scheduler → Weekly Report Lambda        │
       │            └──────────────────────────────────────────────────────┘
       └─────────────────────────────→ CloudFront → S3 (SPA)
```

### 核心设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 后端部署 | 单 Lambda（Monolith） | MVP 阶段，简单易维护；可按需拆分 |
| API 网关 | HTTP API（非 REST API） | 成本低 70%，性能更好，功能足够 |
| 数据库 | DynamoDB 单表 | Serverless，按需付费，适合 MVP 规模 |
| AI 模型 | Claude Haiku via Bedrock | 速度快，成本低，瑞士合规（eu-central-2） |
| 认证 | Cognito | 无需自建，GDPR 合规，JWT 集成简单 |
| 图片存储 | S3 + 预签名 URL | 前端直传，减轻 Lambda 负担 |

---

## 二、功能模块

### 2.1 MVP 功能（Phase 1）

#### 认证与用户管理
- 邮箱注册（学生/家长/老师/管理员四角色）
- 邮件验证 + Cognito User Pool 管理
- JWT Access/Refresh Token（Access: 1h，Refresh: 30d）
- 家长-学生账号关联

#### 学生 AI 答疑
- 文字 + 图片提问（支持 LaTeX）
- Rekognition OCR 自动提取题目
- Claude Haiku 分步引导式解答（prompt 工程确保不直接给答案）
- 问题状态机：`pending → ai_answering → ai_answered → teacher_requested → teacher_active → resolved`
- 每日提问限额（Free: 5，Standard: 30，Premium: ∞）
- 解答反馈评分（1–5 星）

#### 老师接管
- SQS FIFO 队列管理老师升级请求
- 老师工作台（题目队列 + 锁定机制）
- 接管后 AI 自动静默
- 老师富文本回复（含数学公式）
- 响应时间 SLA 追踪

#### 家长面板
- 学生学习摘要（知识点薄弱分析）
- 自动周报（Bedrock 生成摘要 + SES 发送）
- 订阅管理入口

#### 管理后台
- 用户管理（手动修改订阅等级）
- 平台统计（DAU、AI 使用量、老师效率）

---

## 三、API 接口清单

### 认证 `/auth`

| Method | Path | 描述 | Auth |
|--------|------|------|------|
| POST | /auth/register | 注册新用户 | 无 |
| POST | /auth/login | 邮箱密码登录 | 无 |
| POST | /auth/refresh | 刷新 AccessToken | 无 |
| POST | /auth/logout | 撤销 RefreshToken | JWT |

### 文件 `/files`

| Method | Path | 描述 | Auth |
|--------|------|------|------|
| POST | /files/presign | 获取 S3 图片上传预签名 URL | JWT |

### 题目 `/questions`

| Method | Path | 描述 | Auth |
|--------|------|------|------|
| POST | /questions | 提交新题目（触发 OCR + AI） | JWT(Student) |
| GET | /questions/{id} | 查询题目详情 + AI 回复 | JWT |
| POST | /questions/{id}/request-teacher | 请求老师接管 | JWT(Student) |
| POST | /questions/{id}/feedback | 对 AI 解答评分 | JWT(Student) |

### 学生 `/students`

| Method | Path | 描述 | Auth |
|--------|------|------|------|
| GET | /students/{id}/questions | 获取历史题目（分页） | JWT(Student/Parent) |
| GET | /students/{id}/summary | 获取学习摘要（薄弱知识点） | JWT(Student/Parent) |

### 老师 `/teachers`

| Method | Path | 描述 | Auth |
|--------|------|------|------|
| GET | /teachers/queue | 获取待处理题目队列 | JWT(Teacher) |
| POST | /teachers/questions/{id}/takeover | 锁定题目，开始接管 | JWT(Teacher) |
| POST | /teachers/questions/{id}/reply | 提交老师回复 | JWT(Teacher) |
| PUT | /teachers/questions/{id}/resolve | 标记题目已解决 | JWT(Teacher) |

### 家长 `/parents`

| Method | Path | 描述 | Auth |
|--------|------|------|------|
| GET | /parents/{id}/children | 获取绑定的学生列表 | JWT(Parent) |
| GET | /parents/{id}/reports/{week} | 查询指定周的学习报告 | JWT(Parent) |

### 管理 `/admin`

| Method | Path | 描述 | Auth |
|--------|------|------|------|
| GET | /admin/users | 分页查询用户列表 | JWT(Admin) |
| PATCH | /admin/users/{id} | 更新用户信息/订阅等级 | JWT(Admin) |
| GET | /admin/stats | 平台统计数据 | JWT(Admin) |

### 通用

| Method | Path | 描述 | Auth |
|--------|------|------|------|
| GET | /health | 健康检查 | 无 |

---

## 四、数据模型

### DynamoDB 单表设计

**表名**：`stoa-{env}`  
**Billing**：On-Demand（按需）

| 实体 | PK | SK | GSI |
|------|----|----|-----|
| User | `USER#<user_id>` | `PROFILE` | GSI1: email_index |
| Question | `QUESTION#<question_id>` | `META` | GSI2: student_index (student_id + created_at) |
| WeeklyReport | `REPORT#<report_id>` | `SUMMARY` | GSI3: parent_index (parent_id + week_start) |
| TeacherSession | `SESSION#<session_id>` | `META` | GSI4: teacher_index (teacher_id + status) |

### User 模型

```python
class User:
    user_id: str          # UUID
    email: str
    role: str             # student | parent | teacher | admin
    name: str
    grade: str | None     # 学生年级（7–13）
    parent_id: str | None # 学生关联的家长 ID
    children: list[str]   # 家长关联的学生 ID 列表
    subscription_tier: str  # free | standard | premium
    daily_question_count: int  # 当天已提问次数
    daily_reset_date: str     # 最后重置日期（YYYY-MM-DD）
    created_at: str       # ISO 8601
    updated_at: str
```

### Question 模型

```python
class Question:
    question_id: str      # UUID
    student_id: str
    subject: str          # math | physics | german | english（MVP: math）
    grade: str            # 7 | 8 | ... | 13
    text: str             # 题目文字（原始输入或 OCR 结果）
    image_key: str | None # S3 key（如有图片）
    ocr_text: str | None  # OCR 提取的原始文字
    status: str           # pending | ai_answering | ai_answered | teacher_requested | teacher_active | resolved
    ai_steps: list[dict]  # AI 分步解答 [{"step": 1, "explanation": "..."}]
    teacher_reply: str | None
    teacher_id: str | None
    feedback_score: int | None  # 1–5
    ai_tokens_used: int   # Bedrock 计费用量
    created_at: str
    updated_at: str
    resolved_at: str | None
```

### WeeklyReport 模型

```python
class WeeklyReport:
    report_id: str        # UUID
    student_id: str
    parent_id: str
    week_start: str       # ISO date（周一）
    total_questions: int
    ai_resolved: int
    teacher_resolved: int
    avg_feedback_score: float
    weak_topics: list[str]   # 薄弱知识点 Top 5
    ai_summary: str       # Bedrock 生成的自然语言摘要（德语）
    created_at: str
```

### TeacherSession 模型

```python
class TeacherSession:
    session_id: str       # UUID
    question_id: str
    teacher_id: str
    status: str           # active | resolved
    takeover_at: str
    resolved_at: str | None
    response_time_seconds: int | None
```

---

## 五、AWS 服务及费用估算

> 基准：100 个活跃学生，每人每天 10 次 AI 请求，1000 次/天总量

| 服务 | 规格 | 月费用估算（CHF） |
|------|------|----------------|
| Lambda | 1000次/天 × 30天 = 30,000次，平均 3s，512MB | ~CHF 1 |
| API Gateway | 30,000 次请求/月，HTTP API | ~CHF 1 |
| DynamoDB | On-Demand，30K reads + 30K writes/月 | ~CHF 3 |
| S3 | 10GB 图片存储 + 5GB 前端 + 传输 | ~CHF 3 |
| CloudFront | 50GB 流量/月 | ~CHF 5 |
| Bedrock (Claude Haiku) | 30K 请求 × 平均 800 tokens = 24M tokens | ~CHF 30–50 |
| Rekognition | 1,000 图片/月 × $0.001 | ~CHF 1 |
| Cognito | < 50,000 MAU 免费 | CHF 0 |
| SQS | < 1M 消息免费 | CHF 0 |
| SES | 1,000 邮件/月 × $0.0001 | ~CHF 0.1 |
| EventBridge | 少量规则，几乎免费 | ~CHF 0 |
| CloudWatch | 基础监控免费，日志 5GB/月 | ~CHF 2 |
| WAF | $5/规则/月，3 条规则 | ~CHF 15 |
| Secrets Manager | 5 个 Secret | ~CHF 2 |
| ACM | 免费 | CHF 0 |
| **合计** | | **~CHF 65–90/月** |

**成本优化策略**：
- Bedrock 是最大成本，优化 prompt 长度是关键
- Lambda 冷启动：仅对关键路由启用 Provisioned Concurrency（可选）
- CloudFront 缓存策略优化静态资源命中率
- Phase 2 可考虑 Bedrock Batch Inference 降低批量任务成本

---

## 六、后端架构

### 6.1 目录结构

```
stoa-backend/
├── src/
│   └── stoa/
│       ├── main.py           # FastAPI app + Mangum handler
│       ├── config.py         # pydantic-settings 配置
│       ├── deps.py           # 依赖注入（get_current_user 等）
│       ├── routers/
│       │   ├── auth.py
│       │   ├── questions.py
│       │   ├── students.py
│       │   ├── teachers.py
│       │   ├── parents.py
│       │   ├── files.py
│       │   └── admin.py
│       ├── services/
│       │   ├── ai_service.py      # Bedrock Claude Haiku
│       │   ├── ocr_service.py     # Rekognition
│       │   ├── auth_service.py    # Cognito JWT 校验
│       │   ├── notify_service.py  # SQS + SES
│       │   └── report_service.py  # 周报生成
│       ├── db/
│       │   ├── dynamodb.py        # DynamoDB client 封装
│       │   └── repositories/
│       │       ├── user_repo.py
│       │       ├── question_repo.py
│       │       └── report_repo.py
│       └── models/
│           ├── user.py
│           ├── question.py
│           └── report.py
├── tests/
│   ├── unit/
│   └── integration/
├── pyproject.toml
└── .env.example
```

### 6.2 请求处理流程

```
HTTP Request
    → API Gateway (HTTP API)
    → Lambda invoke
    → Mangum adapter
    → FastAPI middleware（日志、CORS、异常处理）
    → JWT 依赖注入（deps.py）
    → Router handler
    → Service 层
    → Repository / AWS SDK
    → Response
```

### 6.3 AI 答疑流程（核心链路）

```
POST /questions
    1. OCR（如有图片）：Rekognition → 提取文字
    2. 构建 Prompt：
       - System: 你是瑞士德语区初高中数学老师，用分步方式引导学生理解，不直接给最终答案
       - User: 年级 + 学科 + 题目文字
    3. Bedrock invoke_model (claude-haiku-20240307)
    4. 解析 JSON 格式的分步解答
    5. 存储到 DynamoDB（status: ai_answered）
    6. 返回 question_id 给前端
```

---

## 七、前端架构

### 7.1 目录结构

```
stoa-frontend/
├── src/
│   ├── main.tsx
│   ├── App.tsx               # 路由配置
│   ├── lib/
│   │   ├── api.ts            # Axios 实例 + JWT 拦截器
│   │   └── amplify.ts        # Amplify 配置
│   ├── store/
│   │   ├── authStore.ts      # Zustand 认证状态
│   │   └── questionStore.ts  # 当前题目状态
│   ├── pages/
│   │   ├── auth/
│   │   │   ├── Login.tsx
│   │   │   └── Register.tsx
│   │   ├── student/
│   │   │   ├── Home.tsx      # 学习工作台
│   │   │   ├── Ask.tsx       # 提问页
│   │   │   ├── Answer.tsx    # 解答页
│   │   │   └── History.tsx   # 历史记录
│   │   ├── teacher/
│   │   │   ├── Queue.tsx     # 待处理队列
│   │   │   └── Session.tsx   # 答疑工作台
│   │   ├── parent/
│   │   │   ├── Dashboard.tsx # 学习摘要
│   │   │   └── Report.tsx    # 周报详情
│   │   └── admin/
│   │       └── Dashboard.tsx # 管理后台
│   └── components/
│       ├── ImageUpload.tsx
│       ├── StepAnswer.tsx    # 分步解答渲染
│       └── WeeklyChart.tsx   # 学习趋势图
├── package.json
└── vite.config.ts
```

### 7.2 状态管理策略

| 状态类型 | 工具 | 场景 |
|---------|------|------|
| 服务端数据（缓存） | TanStack Query | 题目列表、周报、摘要 |
| 客户端状态 | Zustand | 用户信息、当前题目 ID |
| 认证状态 | Zustand + Amplify | JWT Token、用户角色 |
| 表单状态 | React State | 提问表单、回复框 |

### 7.3 路由与权限守卫

```
/                  → 登录页（未认证重定向）
/login             → 公开
/register          → 公开
/student/*         → 仅 role=student
/teacher/*         → 仅 role=teacher
/parent/*          → 仅 role=parent
/admin/*           → 仅 role=admin
```

---

## 八、CDK 基础设施架构

### 8.1 Stack 划分

```
stoa-infra/
├── stacks/
│   ├── auth_stack.py        # Cognito User Pool + 4 App Clients
│   ├── database_stack.py    # DynamoDB 单表 + 4 GSI
│   ├── storage_stack.py     # S3: images / reports / logs
│   ├── api_stack.py         # Lambda + API Gateway + WAF
│   ├── ai_stack.py          # Bedrock IAM 权限 + EventBridge 周报调度
│   ├── notification_stack.py # SQS FIFO + SES 域名验证
│   ├── monitoring_stack.py  # CloudWatch Dashboard + Alarms
│   └── frontend_stack.py    # S3 SPA + CloudFront + OAC (eu-central-2)
```

### 8.2 AuthStack

```python
# 关键配置
user_pool = cognito.UserPool(
    self, "StoaUserPool",
    self_sign_up_enabled=True,
    auto_verify={"email": True},
    password_policy={"min_length": 8, "require_uppercase": True},
    account_recovery=cognito.AccountRecovery.EMAIL_ONLY,
)

# 4 个 App Client：student / parent / teacher / admin
# 每个 client 有独立的 auth flow + 自定义 attribute
```

### 8.3 ApiStack

```python
# Lambda
fn = lambda_.Function(
    self, "StoaApi",
    runtime=lambda_.Runtime.PYTHON_3_12,
    handler="stoa.main.handler",
    memory_size=512,
    timeout=Duration.seconds(30),  # AI 请求最多 30s
    environment={
        "DYNAMODB_TABLE": table.table_name,
        "S3_BUCKET": bucket.bucket_name,
        "USER_POOL_ID": user_pool.user_pool_id,
    }
)

# HTTP API Gateway
api = apigw.HttpApi(
    self, "StoaHttpApi",
    cors_preflight={...},
    default_authorizer=JwtAuthorizer(user_pool),
)
```

### 8.4 部署环境

| 环境 | 用途 | AWS Account |
|------|------|-------------|
| dev | 开发测试 | 同一账号，前缀 dev- |
| prod | 生产 | 同一账号，前缀 prod- |

CDK Context 区分环境：
```json
// cdk.json context
{
  "dev": {"domainPrefix": "dev-stoa", "logLevel": "DEBUG"},
  "prod": {"domainPrefix": "stoa", "logLevel": "INFO", "alarmEnabled": true}
}
```

---

## 九、文件夹结构

### 9.1 GitHub 仓库结构

```
github.com/stoasystem/
├── stoa-backend/     # Python FastAPI
├── stoa-frontend/    # React TypeScript
├── stoa-infra/       # AWS CDK Python
└── stoa-docs/        # 文档
    ├── README.md
    ├── PRD.md        # 本文档（产品需求）
    ├── HLD.md        # 本文档（高层设计）
    ├── PLAN.md       # 开发计划
    ├── ADR.md        # 架构决策记录
    └── DEPLOYMENT.md # 部署指南
```

### 9.2 完整文件夹树

```
stoa-backend/
├── src/stoa/
│   ├── main.py
│   ├── config.py
│   ├── deps.py
│   ├── routers/         # 7 个 router 文件
│   ├── services/        # 5 个 service 文件
│   ├── db/
│   │   ├── dynamodb.py
│   │   └── repositories/  # 3 个 repo 文件
│   └── models/          # 3 个 model 文件
├── tests/
│   ├── unit/
│   └── integration/
├── pyproject.toml       # uv 管理
└── .env.example

stoa-frontend/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── lib/             # api.ts, amplify.ts
│   ├── store/           # authStore, questionStore
│   ├── pages/           # 4 个角色页面目录
│   └── components/      # 公共组件
├── public/
├── package.json
└── vite.config.ts

stoa-infra/
├── stacks/              # 8 个 CDK Stack
├── app.py               # CDK App 入口
├── cdk.json
└── pyproject.toml

stoa-docs/
├── README.md
├── PRD.md
├── HLD.md
├── PLAN.md
├── ADR.md
└── DEPLOYMENT.md
```

---

*文档持续更新中。最新版本见 [stoa-docs/HLD.md](./HLD.md)*
