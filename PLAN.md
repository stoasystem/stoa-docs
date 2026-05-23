# STOA 开发计划

> **版本：** v1.0  
> **日期：** 2026-05-23  
> **作者：** STOA Team  
> **状态：** 草稿

---

## 目录

1. [项目概览](#一项目概览)
2. [开发原则](#二开发原则)
3. [技术栈总览](#三技术栈总览)
4. [阶段规划](#四阶段规划)
5. [MVP 详细 Sprint 计划](#五mvp-详细-sprint-计划)
6. [模块任务拆解](#六模块任务拆解)
7. [团队分工](#七团队分工)
8. [里程碑与交付物](#八里程碑与交付物)
9. [测试策略](#九测试策略)
10. [上线 Checklist](#十上线-checklist)
11. [风险评估与应对](#十一风险评估与应对)
12. [规范与约定](#十二规范与约定)

---

## 一、项目概览

### 1.1 产品定位

STOA 是一个面向**瑞士德语区（Zürich）中小学生**的本地化课后学习支持平台。核心模式：

```
学生课后提问
    → 受控 AI 分步解答（Bedrock Claude Haiku）
    → 学生仍不理解 → 真人老师接管
    → 全程学习记录 → 家长可见的周报
```

STOA **不替代**传统家教，而是补足课后无法实时覆盖的即时答疑场景。

### 1.2 目标用户

| 角色 | 描述 | 核心诉求 |
|------|------|----------|
| 学生 | Sekundarschule / Gymnasium 学生 | 课后遇题立即获得帮助，不被卡住 |
| 家长 | 瑞士中产家庭，愿意付费教育 | 透明、可追踪、有老师兜底 |
| 老师 | STOA 合作家教老师 | 高效处理复杂问题，减少重复答疑 |
| 管理员 | STOA 运营团队 | 平台质量、内容安全、商业数据 |

### 1.3 商业模式

| 套餐 | 价格 | 内容 |
|------|------|------|
| Free | CHF 0 | 每日 5 次 AI 答疑，无老师服务 |
| Standard | CHF 39–49/月 | 30 次/日 AI 答疑 + 老师文字答疑额度 + 周报 |
| Premium | CHF 79–99/月 | 无限 AI + 优先老师响应 + 详细报告 + 线下衔接 |

---

## 二、开发原则

### 2.1 核心原则

1. **MVP 优先**：先验证商业假设（家长愿意付费），再扩展功能
2. **Monolith First**：FastAPI 单 Lambda，复杂后再拆微服务
3. **AWS Managed 优先**：能用托管服务就不自建（Cognito > 自建 Auth，DynamoDB > 自建 DB）
4. **开源补位**：AWS 不覆盖的场景优先用开源（React、FastAPI、Zustand）
5. **安全第一**：AI 内容受控，所有 S3 无公开访问，WAF 防护

### 2.2 代码规范

- Python：`ruff` 格式化，`mypy` 类型检查，行宽 100
- TypeScript：`eslint` + `prettier`，严格模式
- Git：`main` 保护分支，功能分支 `feat/xxx`，bugfix 分支 `fix/xxx`
- Commit 格式：`feat: ...` / `fix: ...` / `chore: ...` / `docs: ...`
- PR 必须至少 1 人 review 后合并

### 2.3 Definition of Done

每个任务完成的标准：
- [ ] 代码通过 lint / type check
- [ ] 单元测试覆盖率 ≥ 70%（核心 service 层）
- [ ] 已在本地 / dev 环境验证
- [ ] PR 已 review 并合并
- [ ] 相关文档已更新

---

## 三、技术栈总览

| 层次 | 技术 | 说明 |
|------|------|------|
| 后端语言 | Python 3.12 | |
| 后端框架 | FastAPI | 高性能，原生 async |
| Lambda 适配 | Mangum | FastAPI → Lambda handler |
| 包管理 | uv | 替代 pip/poetry，极速依赖管理 |
| 配置管理 | pydantic-settings | 环境变量类型安全 |
| 前端框架 | React 19 + TypeScript | |
| 前端构建 | Vite | 极速 HMR |
| 状态管理 | Zustand | 轻量，无 Redux 样板 |
| 服务端状态 | TanStack Query | 缓存 + 自动刷新 |
| 认证客户端 | AWS Amplify JS | Cognito 集成 |
| HTTP 客户端 | Axios | JWT 拦截器 |
| 基础设施 | AWS CDK v2 (Python) | 8 个 Stack |
| 数据库 | DynamoDB On-Demand | 单表设计，4 GSI |
| 认证 | Amazon Cognito | User Pool，4 App Client |
| AI | Amazon Bedrock (Claude Haiku) | 受控 prompt |
| OCR | Amazon Rekognition | 图片题目提取 |
| 存储 | Amazon S3 | 图片 + 前端 + 报告 |
| CDN | Amazon CloudFront | SPA + OAC |
| 队列 | Amazon SQS FIFO | 老师升级队列 |
| 邮件 | Amazon SES | 家长周报 |
| 定时任务 | EventBridge Scheduler | 每周周报生成 |
| 监控 | CloudWatch + WAF | 日志、告警、速率限制 |
| Region | eu-central-2（苏黎世） | 所有 Stack 统一 |

---

## 四、阶段规划

### Phase 1 — MVP（第 1–8 周）

**目标**：验证核心商业假设，上线最小可用产品

**范围**：
- 地区：Zürich 德语区
- 学段：Sekundarschule + Gymnasium
- 学科：**仅数学**
- 形式：文字 + 图片提问
- AI：受控分步解释
- 老师：文字接管
- 家长：学习摘要 + 周报
- 订阅：手动开通（不做完整支付系统）

**验证指标**：
- 10+ 付费家庭（Standard 或 Premium）
- 学生 7 日留存率 > 40%
- AI 答疑满意度 > 3.5/5
- 老师接管率 < 30%（AI 解决大部分问题）
- 月 AWS 费用 < CHF 150

---

### Phase 2 — 成长期（第 9–20 周）

**目标**：扩展学科、完善体验、建立付费体系

**新增功能**：
- Stripe 订阅支付系统
- 多学科支持（物理、德语、英语）
- 个性化学习记忆（学生画像）
- AI 老师辅助工具（自动总结、生成练习题）
- 完整家长周报（趋势图、知识点变化）
- WebSocket 实时通知（替代轮询）
- 移动端响应式优化

---

### Phase 3 — 扩张期（第 21 周起）

**目标**：B2B 合作，覆盖机构

**方向**：
- 其他辅导学校 SaaS 版本
- 私立学校 / 国际学校合作
- Matura 备考专项产品
- 多语言支持（法语区）

---

## 五、MVP 详细 Sprint 计划

> **Sprint 周期**：2 周  
> **总时长**：8 周（4 个 Sprint）  
> **每周工作日**：5 天

---

### Sprint 1（第 1–2 周）：基础设施 + 认证

**Sprint 目标**：AWS 环境就绪，用户可以注册登录

#### 基础设施（stoa-infra）

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| CDK Bootstrap（eu-central-2） | P0 | 1h | 后端 |
| 部署 AuthStack（Cognito User Pool + 4 App Client） | P0 | 4h | 后端 |
| 部署 DatabaseStack（DynamoDB 单表 + 4 GSI） | P0 | 3h | 后端 |
| 部署 StorageStack（S3 images/reports/logs bucket） | P0 | 2h | 后端 |
| 部署 FrontendStack（S3 + CloudFront + OAC） | P0 | 3h | 后端 |
| 配置 dev / prod 两套环境 context | P1 | 2h | 后端 |
| 配置 CloudWatch 基础 Dashboard | P2 | 2h | 后端 |

#### 后端（stoa-backend）

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| uv 环境初始化，本地开发配置 | P0 | 1h | 后端 |
| FastAPI app 骨架 + Mangum handler | P0 | 2h | 后端 |
| pydantic-settings 配置（.env 支持） | P0 | 1h | 后端 |
| DynamoDB client 封装（db/dynamodb.py） | P0 | 2h | 后端 |
| User 数据模型 + user_repo.py | P0 | 3h | 后端 |
| POST /auth/register（Cognito Admin API 创建用户） | P0 | 4h | 后端 |
| POST /auth/login（Cognito InitiateAuth） | P0 | 3h | 后端 |
| POST /auth/refresh（RefreshToken flow） | P0 | 2h | 后端 |
| POST /auth/logout（RevokeToken） | P1 | 1h | 后端 |
| JWT 依赖注入（deps.py get_current_user） | P0 | 3h | 后端 |
| GET /health endpoint | P0 | 0.5h | 后端 |
| 部署 ApiStack（Lambda + API Gateway HTTP API + WAF） | P0 | 3h | 后端 |
| 单元测试：auth router（moto mock Cognito） | P1 | 4h | 后端 |

#### 前端（stoa-frontend）

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| Vite + React 19 + TypeScript 初始化 | P0 | 1h | 前端 |
| Amplify JS 配置（amplify.ts） | P0 | 2h | 前端 |
| Axios 实例 + JWT 拦截器（api.ts） | P0 | 2h | 前端 |
| Zustand authStore（user 状态） | P0 | 1h | 前端 |
| 登录页面 UI | P0 | 4h | 前端 |
| 注册页面 UI（角色选择 + 年级） | P0 | 4h | 前端 |
| React Router 路由配置 + 权限守卫 | P0 | 3h | 前端 |
| 首次部署前端到 S3 + CloudFront | P0 | 1h | 前端 |

**Sprint 1 交付物**：
- ✅ 用户可以注册（学生/家长/老师角色）
- ✅ 用户可以登录，获得 JWT Token
- ✅ AWS 基础设施全部 up
- ✅ API `/health` 可访问

---

### Sprint 2（第 3–4 周）：核心问答流程（AI 答疑）

**Sprint 目标**：学生可以提交问题，AI 返回分步解答

#### 后端

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| Question 数据模型 + question_repo.py | P0 | 4h | 后端 |
| POST /files/presign（S3 预签名 URL） | P0 | 2h | 后端 |
| Rekognition OCR 封装（ocr_service.py） | P0 | 3h | 后端 |
| POST /questions（提交题目，触发 OCR + AI） | P0 | 6h | 后端 |
| Bedrock AI service（ai_service.py + prompt 模板） | P0 | 8h | 后端 |
| GET /questions/{id}（查询题目 + AI 回复） | P0 | 2h | 后端 |
| 问题状态机（pending→ai_answered） | P0 | 3h | 后端 |
| Free 用户每日次数限制（DynamoDB 计数） | P1 | 3h | 后端 |
| GET /students/{id}/questions（分页历史） | P1 | 2h | 后端 |
| 单元测试：AI service（mock Bedrock response） | P1 | 4h | 后端 |

#### 前端

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| 学生首页 Home.tsx（学习工作台） | P0 | 4h | 前端 |
| 提问页 Ask.tsx（文字输入 + 图片上传） | P0 | 6h | 前端 |
| 图片上传组件 ImageUpload.tsx（S3 预签名 URL 直传） | P0 | 5h | 前端 |
| AI 分步解答页 Answer.tsx（StepAnswer 组件） | P0 | 8h | 前端 |
| 分步解答渲染组件 StepAnswer.tsx | P0 | 5h | 前端 |
| 问题状态轮询（TanStack Query refetchInterval） | P0 | 2h | 前端 |
| 问题历史页 History.tsx | P1 | 4h | 前端 |

**Sprint 2 交付物**：
- ✅ 学生可以输入文字题目
- ✅ 学生可以上传作业图片（OCR 自动提取文字）
- ✅ AI 返回结构化分步解答
- ✅ 解答页面分步渲染（非直接给答案）
- ✅ Free 用户每日 5 次限制

---

### Sprint 3（第 5–6 周）：老师接管 + 家长面板

**Sprint 目标**：老师可以接管复杂问题，家长可以查看学习摘要

#### 后端

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| SQS FIFO 队列推送（notify_service.py） | P0 | 3h | 后端 |
| POST /questions/{id}/request-teacher（升级 + 入队） | P0 | 3h | 后端 |
| GET /teachers/queue（老师待处理队列） | P0 | 2h | 后端 |
| POST /teachers/questions/{id}/takeover（锁定题目） | P0 | 3h | 后端 |
| POST /teachers/questions/{id}/reply（老师回复） | P0 | 2h | 后端 |
| PUT /teachers/questions/{id}/resolve（标记解决） | P0 | 1h | 后端 |
| TeacherSession 数据模型 + 存储 | P0 | 3h | 后端 |
| AI 自动静默逻辑（teacher_active 状态） | P0 | 2h | 后端 |
| GET /students/{id}/summary（薄弱知识点聚合） | P1 | 4h | 后端 |
| GET /parents/{id}/children（绑定学生列表） | P1 | 2h | 后端 |
| GET /parents/{id}/reports/{week}（查询周报） | P1 | 2h | 后端 |
| WeeklyReport 数据模型 + report_repo.py | P1 | 3h | 后端 |
| POST /questions/{id}/feedback（学生评分） | P1 | 2h | 后端 |

#### 前端

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| 答疑页「请求老师帮助」按钮 + 状态切换 | P0 | 3h | 前端 |
| 老师接管状态 UI（「老师正在帮助你」提示） | P0 | 3h | 前端 |
| 老师工作台 Queue.tsx（待处理队列列表） | P0 | 5h | 前端 |
| 老师答疑页 Session.tsx（含 AI 上下文、回复框） | P0 | 6h | 前端 |
| 家长首页 Dashboard.tsx（摘要卡片） | P1 | 5h | 前端 |
| 家长周报页 Report.tsx | P1 | 5h | 前端 |
| 学习趋势图组件 WeeklyChart.tsx | P1 | 4h | 前端 |

**Sprint 3 交付物**：
- ✅ 学生可以请求老师接管
- ✅ 老师看到待处理队列
- ✅ 老师接管后可回复，AI 自动静默
- ✅ 家长可查看孩子学习摘要
- ✅ 家长周报页面可用

---

### Sprint 4（第 7–8 周）：完善 + 上线

**Sprint 目标**：管理后台、周报定时任务、全面测试、正式上线

#### 后端

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| EventBridge 周报定时任务 Lambda | P0 | 5h | 后端 |
| report_service.py（聚合数据 + Bedrock 生成摘要） | P0 | 6h | 后端 |
| SES 邮件发送（家长周报 HTML 模板） | P0 | 4h | 后端 |
| GET /admin/users（分页用户列表） | P1 | 2h | 后端 |
| PATCH /admin/users/{id}（更新订阅等级） | P1 | 2h | 后端 |
| GET /admin/stats（平台指标） | P1 | 3h | 后端 |
| WAF 规则调优（速率限制 IP 阈值） | P1 | 2h | 后端 |
| CloudWatch 告警配置（错误率 + P99 延迟） | P1 | 2h | 后端 |
| 集成测试（完整问答链路） | P0 | 8h | 后端 |
| Lambda 冷启动优化（依赖精简） | P1 | 4h | 后端 |

#### 前端

| 任务 | 优先级 | 预估工时 | 负责人 |
|------|--------|----------|--------|
| 管理员后台 Dashboard.tsx（用户列表 + 统计） | P1 | 6h | 前端 |
| 错误页面（404 / 500） | P1 | 2h | 前端 |
| 加载态 / 骨架屏优化 | P1 | 3h | 前端 |
| 响应式布局适配（平板 + 手机） | P1 | 6h | 前端 |
| 多语言框架接入（de 为主，预留 en/fr） | P2 | 4h | 前端 |
| E2E 测试（Playwright，注册→提问→解答流程） | P1 | 8h | 前端 |

#### 上线准备

| 任务 | 优先级 | 预估工时 |
|------|--------|----------|
| 生产环境 CDK deploy（prod context） | P0 | 3h |
| SES 域名验证（stoa.ch） + DNS 配置 | P0 | 2h |
| Cognito 邮件模板定制（德语） | P1 | 2h |
| CloudFront 自定义域名配置 | P0 | 2h |
| 安全审查（IAM 最小权限、S3 公开访问检查） | P0 | 3h |
| 内测（5–10 个真实家庭试用） | P0 | 1 周 |
| Bug 修复迭代 | P0 | 10h |

**Sprint 4 交付物**：
- ✅ 每周一自动生成并发送家长周报
- ✅ 管理员后台可查看用户和统计
- ✅ 全链路集成测试通过
- ✅ 生产环境部署完成
- ✅ 内测 5+ 真实用户

---

## 六、模块任务拆解

### 6.1 stoa-backend 模块清单

#### routers/（API 路由层）

| 文件 | 包含端点 | Sprint |
|------|---------|--------|
| `auth.py` | register, login, refresh, logout | S1 |
| `files.py` | presign | S2 |
| `questions.py` | submit, get, request-teacher, feedback | S2–S3 |
| `students.py` | summary, questions | S2–S3 |
| `teachers.py` | queue, takeover, reply, resolve | S3 |
| `parents.py` | children, reports | S3 |
| `admin.py` | users, stats | S4 |

#### services/（业务逻辑层）

| 文件 | 功能 | 外部依赖 | Sprint |
|------|------|---------|--------|
| `ai_service.py` | Bedrock 调用 + prompt 模板 | Bedrock | S2 |
| `ocr_service.py` | 图片文字提取 | Rekognition | S2 |
| `auth_service.py` | Cognito JWT 校验 | Cognito | S1 |
| `notify_service.py` | SQS 入队 + SES 邮件 | SQS, SES | S3 |
| `report_service.py` | 周报聚合 + AI 摘要 | DynamoDB, Bedrock | S4 |

#### db/repositories/（数据访问层）

| 文件 | 主要方法 | Sprint |
|------|---------|--------|
| `user_repo.py` | put_user, get_user, get_by_email | S1 |
| `question_repo.py` | put, get, list_by_student, update_status | S2 |
| `report_repo.py` | put_report, get_by_week | S3–S4 |

### 6.2 stoa-infra CDK Stack 部署顺序

```
AuthStack
    ↓
DatabaseStack + StorageStack（并行）
    ↓
NotificationStack
    ↓
ApiStack（依赖 Auth + Database + Storage + Notification）
    ↓
AiStack + MonitoringStack（并行）
    ↓
FrontendStack（eu-central-2）
```

### 6.3 stoa-frontend 页面优先级

| 页面 | 路由 | 优先级 | Sprint |
|------|------|--------|--------|
| 登录 | /login | P0 | S1 |
| 注册 | /register | P0 | S1 |
| 学生首页 | /student | P0 | S2 |
| 提问页 | /student/ask | P0 | S2 |
| 解答页 | /student/answer/:id | P0 | S2 |
| 历史页 | /student/history | P1 | S2 |
| 老师队列 | /teacher/queue | P0 | S3 |
| 老师答疑 | /teacher/session/:id | P0 | S3 |
| 家长首页 | /parent | P1 | S3 |
| 家长周报 | /parent/report/:week | P1 | S3 |
| 管理后台 | /admin | P2 | S4 |

---

## 七、团队分工

### 7.1 角色职责

| 角色 | 主要职责 | 仓库权限 |
|------|---------|---------|
| 后端开发 | FastAPI、DynamoDB、Bedrock、CDK | stoa-backend, stoa-infra |
| 前端开发 | React、UI 组件、Amplify 认证 | stoa-frontend |
| 全栈（Tech Lead） | 架构决策、CR、CI/CD、跨端联调 | 全部 |
| 产品 / 运营 | PRD 维护、内测协调、用户反馈 | stoa-docs |

### 7.2 工作流

```
功能开发
  1. 从 main 创建 feat/xxx 分支
  2. 本地开发 + 单元测试
  3. Push → 发起 PR
  4. 至少 1 人 Review
  5. CI 通过（lint + test）
  6. Squash merge 到 main
  7. 自动部署到 dev 环境

上线流程
  1. main 打 tag（v0.1.0）
  2. 手动触发 prod deploy
  3. 验证 /health + 核心流程
  4. 发布通知
```

---

## 八、里程碑与交付物

| 里程碑 | 时间 | 交付物 | 验收标准 |
|--------|------|--------|---------|
| M1：基础设施就绪 | 第 2 周末 | AWS 环境 + 认证系统 | 用户可注册/登录，API 可访问 |
| M2：AI 答疑上线 | 第 4 周末 | 完整问答流程 | 学生提交问题，AI 30s 内返回分步解答 |
| M3：老师接管上线 | 第 6 周末 | 老师工作台 + 家长面板 | 老师可接管，家长可看摘要 |
| M4：MVP 正式上线 | 第 8 周末 | 完整 MVP + 内测 | 5+ 付费用户，系统稳定运行 7 天 |
| M5：Phase 2 启动 | 第 12 周 | Stripe 支付 + 多学科 | 订阅系统上线，新增物理/德语 |

---

## 九、测试策略

### 9.1 测试层次

```
E2E Tests（Playwright）
    ↑ 核心用户流程
Integration Tests（pytest + moto）
    ↑ API → DB → AWS 服务 mock
Unit Tests（pytest）
    ↑ service / repository 单元
```

### 9.2 测试重点

| 模块 | 测试类型 | 重点场景 |
|------|---------|---------|
| auth_service | 单元 | Token 校验、角色权限、过期处理 |
| ai_service | 单元 | prompt 模板正确性、response 解析、错误处理 |
| question_repo | 单元 + 集成 | DynamoDB put/get/update，GSI 查询 |
| POST /questions | 集成 | OCR → AI → DynamoDB 完整链路 |
| 老师接管流程 | E2E | 学生请求 → 老师接管 → AI 静默 → 解决 |
| 家长周报生成 | 集成 | 数据聚合 + Bedrock 摘要 + SES 发送 |

### 9.3 Mock 策略

- **本地开发**：`moto` mock DynamoDB / S3 / SQS
- **Bedrock**：固定 fixture JSON 返回
- **Cognito**：moto mock，或直接用 dev User Pool
- **前端**：MSW（Mock Service Worker）mock API

---

## 十、上线 Checklist

### 基础设施

- [ ] CDK bootstrap 完成（eu-central-2）
- [ ] 所有 Stack 在 prod 环境 deploy 成功
- [ ] S3 所有 Bucket 确认无公开访问
- [ ] WAF 已关联 API Gateway
- [ ] CloudWatch 告警已配置（错误率 + 延迟 + 费用预算）
- [ ] DynamoDB 时间点恢复已开启

### 安全

- [ ] IAM Role 最小权限原则验证
- [ ] Secrets Manager 中无明文密钥
- [ ] Cognito MFA 已配置（管理员账号）
- [ ] CloudTrail 已开启
- [ ] S3 访问日志已开启

### 后端

- [ ] `/health` endpoint 返回 200
- [ ] 所有路由 JWT 保护正常
- [ ] Bedrock prompt 在所有学科/年级组合下返回正确格式
- [ ] Free 用户限额逻辑测试通过
- [ ] SQS 老师队列消息可正常入队/消费

### 前端

- [ ] CloudFront 自定义域名配置完成
- [ ] HTTPS 强制跳转
- [ ] SPA 404 fallback 正常（CloudFront 错误页配置）
- [ ] 移动端布局验证（iOS Safari + Android Chrome）

### 运营

- [ ] SES 域名验证（stoa.ch）完成
- [ ] 邮件发送测试（家长周报 + 注册确认）
- [ ] Cognito 邮件模板已更新为德语
- [ ] 至少 1 名老师账号已创建并测试
- [ ] 监控仪表盘可正常查看

---

## 十一、风险评估与应对

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|---------|
| Bedrock Claude Haiku 回答质量不达标 | 中 | 高 | 准备 prompt 优化迭代方案；预留切换 Sonnet 的开关 |
| Rekognition OCR 对手写题目识别率低 | 高 | 中 | 允许学生手动修正 OCR 结果；Phase 2 切 Textract |
| DynamoDB 单表设计出现访问瓶颈 | 低 | 中 | GSI 按需扩展；热点 Key 用 Hash 分散 |
| Lambda 冷启动影响体验 | 中 | 低 | 关键路由预置并发（约 $3/月）；监控 P99 延迟 |
| AWS Bedrock eu-central-2 不支持 Claude Haiku | 中 | 高 | 切换 Amazon Nova Lite（同 Region）；或申请 Bedrock 模型访问权限 |
| 家长付费意愿不及预期 | 中 | 高 | 内测阶段免费赠送，收集真实反馈，快速迭代 |
| 老师接管响应慢（> 4小时） | 中 | 高 | SLA 承诺；多个老师轮值；SQS 消息可见性超时自动重分配 |
| 内容安全（学生提交不当内容） | 低 | 高 | Bedrock Guardrails 过滤；管理员内容审核工具 |

---

## 十二、规范与约定

### 12.1 分支命名

```
feat/s1-auth-register        # Sprint 1，认证注册功能
feat/s2-ai-answer            # Sprint 2，AI 答疑
fix/question-status-bug      # Bug 修复
chore/update-dependencies    # 依赖更新
docs/update-plan             # 文档更新
```

### 12.2 DynamoDB Key 规范

```
User          PK: USER#<user_id>      SK: PROFILE
Question      PK: QUESTION#<id>       SK: META
WeeklyReport  PK: REPORT#<id>         SK: SUMMARY
TeacherSession PK: SESSION#<id>       SK: META
```

### 12.3 环境变量规范

- 本地开发：`.env`（不提交，`.gitignore` 已配置）
- dev/prod：CDK 注入 Lambda 环境变量，敏感值走 Secrets Manager
- 前端：`VITE_` 前缀，`.env.local` 本地，CI/CD 注入生产值

### 12.4 API 响应格式

```json
// 成功
{"data": {...}, "message": "ok"}

// 错误
{"detail": "错误描述", "code": "ERROR_CODE"}
```

### 12.5 日志规范

- 结构化 JSON 日志（FastAPI middleware 统一输出）
- 必须包含：`request_id`, `user_id`, `path`, `method`, `status_code`, `duration_ms`
- 敏感字段（password, token）不得出现在日志中

---

*文档持续更新中。最新版本见 [stoa-docs/PLAN.md](./PLAN.md)*
