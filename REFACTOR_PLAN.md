# STOA 产品重构计划

> 制定时间：2026-08-24
> 范围：stoa-frontend / stoa-backend / stoa-infra
> 产品目标：收敛为「以对话为唯一主界面、以真人老师实时答疑为核心卖点」的青少年学习产品

---

## 一、现状基线

盘点自 2026-08-24，作为后续所有「减少了多少」的对照基准。

| 指标 | 前端 | 后端 |
|------|------|------|
| 源码规模 | 596 文件 / 48,281 行 | 96,781 行 |
| 对外接口 | 92 条路由 | 237 个端点 |
| 最大单文件 | `ReportOperationsPage.tsx` 3,123 行 | `production_pilot_service.py` 6,359 行 |
| 测试 | 48 个单测 | 148 文件 / 77,441 行 / 1,835 个测试函数 |

`admin.py` 单文件承载 111 个端点、4,471 行，占后端对外面的 47%。

### 核心能力真实状态

真人老师链路的业务状态机（升级 → 派单 → 认领 → 接管 → 回复 → 结单）**已完整实现且并发安全**，使用 DynamoDB 条件事务 + 版本 CAS + 账户围栏，派单排名考虑负载比、SLA 罚分与近期派单惩罚。这部分是可复用资产。

但存在四个阻断性缺口：

1. **实时能力不存在。** `websocket_service.py`（479 行）与 `websocket_repo.py`（351 行）实现完整，但 `register_connection` / `disconnect_connection` 在 `src/` 内无任何调用方；没有 `$connect` / `$disconnect` handler；`stoa-infra` 九个 Stack 中 grep `websocket` 零命中。学生只能轮询获知老师回复。
2. **SQS 升级队列名存实亡。** 队列、DLQ、消费者代码 `jobs/teacher_escalation.py` 均存在，但该 Lambda 从未在 CDK 中部署。实际派单是 API 请求内同步 best-effort；10 分钟接受超时后的重派需人工调用管理端接口，无调度器。
3. **老师候选查询为全表扫描。** `list_teacher_profiles` 对 DynamoDB 做 `SK = "PROFILE"` 全表 scan 后过滤角色，无 GSI。
4. **升级路径重复且有纯桩。** question 与 conversation 各有一套升级实现；`practice.py` 的 `request_teacher_help` 仅返回写死的德语文案，不升级、不派单，老师端不可见。

### 流式与测试的真实状态

- `POST /conversations/{id}/messages/stream` 是**伪流式**：同步取回完整 Bedrock 答案后切成 100 字符块一次性下发。代码注释已自认。根因是 API Gateway 缓冲响应，非参数问题。
- 1,835 个测试函数**全部禁网运行**（conftest 中 `disable_socket`），以 fake 替换全部仓储与 provider。它们证明函数逻辑正确，不证明接口可用。这是审计判定「0/9 关键流程完成」的直接原因。

---

## 二、已确认的产品决策

| # | 决策 | 影响 |
|---|------|------|
| 1 | 计费保留，保证逻辑通，暂不实际启用 | 不删除计费相关约 1.2–1.4 万行；Stripe 维持关闭态 |
| 2 | 真人老师准入保留原策略（免费档不可用） | `teacher_support_allowance_service` 门控不变；测试需手动改档绕过 |
| 3 | 练习 / 自适应 / 作业自动化全部保留 | 删减范围收窄至流程产物 |
| 4 | 线上环境即测试环境，系统尚未正式投入使用 | 允许破坏性变更，无需数据迁移与向后兼容 |

### 决策 4 的连带事实

- 无真实用户，DynamoDB 遗留孤儿档案与 sandbox 数据可安全清理。
- 部署流水线名为 "Deploy to Production"、Lambda 别名为 `production`，而审计记录生产发布控制面为 `DEFERRED_OUT_OF_SCOPE`。命名与实际用途不一致，需在后续阶段正名。

### 决策 2 的连带事实

免费档账号无法验证核心卖点，而 Stripe 处于关闭态（api key 为空、live charges 关闭），无法走正常付费流程。后端已具备手动改档能力（`manualOverrideAt` / `manualOverrideBy` / `manualOverrideSource` 字段与管理端订阅审批流），因此测试通路无需改动业务代码。

---

## 三、阶段一：清场与打通测试通路

目标：移除流程产物、清理死代码、建立可验证核心功能的通路与测试基线。

### P1-0 让聊天求助真正派单给老师 ✅ 已完成

执行 P1-1 时发现的阻断缺陷，优先级高于全部删减工作，故插入为 P1-0。

**问题**：会话升级只在会话记录上写 `escalated` / `escalation_status` 标记，从不调用派单。`conversations.py` 虽引入 `teacher_dispatch_service`，但仅用于查询在线人数。而 `GET /teachers/me/help-requests` 只返回已分配给该老师的请求，因此聊天求助对任何老师都不可见。生产表中一条 2026-05-30 的求助已挂起三个月。

**修复**：新增 `teacher_dispatch_service.dispatch_conversation()`，复用 question 通道相同的候选排名与教师围栏/档位绑定条件，将分配写入会话记录。派单为 best-effort，避免计划器故障导致已扣减周额度的升级被拒。

**结果**：学生求助立即返回被分配的老师姓名，老师队列可见该请求。

### P1-1 打通真人老师测试通路 ✅ 已完成

过程中发现并修复了一个真实生产缺陷，详见第五节。

**实际执行记录**：

原以为只需管理端改档，实际遇到两层阻断。

第一层，档位语义与预期不符：`student` 档的 `teacher_support_cases` 为 0，只有 `teacher_supported`（2 次/周）与 `family`（10 次/周）含真人老师。且 `PAID_GRANT#` 授权仅由 Stripe 激活路径写入，管理端审批只更新家长 profile 的 `subscription_tier`，不产生授权，因此手动改档无法解锁真人老师。

第二层，一个真实生产缺陷：DynamoDB 资源层将所有数字返回为 `Decimal`，而 `paid_entitlement_service` 与 `teacher_support_allowance_service` 的 `_positive_integer` 只接受 `int`。`_relationship_proof` 因此拒绝每一个存储的版本号与围栏代次，`get_active_beneficiary_grant` 恒返回 `None`。**即使接通 Stripe 且用户真实付费，真人老师同样会返回 403。** 同仓库的 `checkout_command_repo`、`question_submission_repo` 均已处理 Decimal，此路径遗漏；因 Stripe 从未启用，该代码从未对真实表执行过，而 hermetic 夹具使用 Python int，故全量测试长期通过。

**修复**：两处 `_positive_integer` 按仓库既有约定接受整数值 Decimal，并补充以 Decimal 构造授权的回归测试。新增 `scripts/enable_teacher_support_testing.py` 为测试身份开通权益与教师派单资格（教师需 `availability_status` 与至少一个学科才会被计为可派单）。

### 阶段一后端删减实际结果

| 指标 | 起点 | 现在 |
|------|------|------|
| 已注册端点 | 237 | 202 |
| `/admin` 端点 | 111 | 76 |
| 删除代码 | — | 约 22,000 行 |

已删除的服务：`production_pilot_service`、`enterprise_stability_service`、`external_activation_service`、`bi_observability_service`、`core_smoke_service`、`report_audit_retention_service`、`report_artifact_edit_service`、`report_edit_service`、`report_recovery_evidence_service`、`support_destination_service`、`support_sla_service`、`support_handoff_service`、`customer_lifecycle_service`。

删除过程中的两个教训，供后续阶段参考：

1. **按文本块删路由会连带吞掉辅助函数。** 以「装饰器到下一个装饰器」切分会把定义在路由之间的模块级函数一并删除，导致 `_get_report_or_404` 消失、大量保留路由 500。必须用 AST 按函数节点边界删除。
2. **autouse fixture 会跨删除边界供给保留测试。** `_fenced_artifact_edit_compat` 同时为已删的编辑服务和保留的 `report_recovery_service` 打补丁，删除后 resend 链路失去围栏与邮件桩而 502。删测试前需确认 fixture 的服务对象范围。

两个守卫会强制清单同步，删完必须执行：`python scripts/generate_route_authorization_inventory.py`，并同步 `test_admin_authorization.py` 中的 admin 路由计数断言。

### P1-2 删除生产试点与证据生成器

**目标**：移除与产品功能无关的流程产物。

**涉及**：`production_pilot_service.py`（6,359 行）、`external_activation_service`（709）、`bi_observability_service`（648）、`enterprise_stability_service`（341）、`release_evidence_service`（300）、`core_smoke_service`（153），以及 `admin.py` 中对应路由（`/external-activation/*`、`/bi/*`、`/core-smoke`）与其测试文件。

**预计**：约 8,500 行，纠缠度极低。

**验收**：`ruff check` 与全量 `pytest` 通过；`/health` 正常。

### P1-3 删除报告恢复 / 审计留存 / 法务保全

**目标**：移除对零用户系统而言过早建设的合规运维机器。

**涉及**：`report_audit_retention_service`（1,519）、`report_recovery_job_service`（988）、`report_artifact_edit_service`（957）、`report_recovery_service`（347）、`report_edit_service`（313）、`report_artifact_service`（301）、`report_recovery_evidence_service`（241），以及 `admin.py` 中 `/reports/**` 段约 45 条路由。

**保留**：`report_service`（912）与 `jobs/weekly_reports` — 家长周报是产品功能，仅删除其外围恢复 / 留存 / 法务机器。

**预计**：约 4,700 行服务 + 1,900 行路由。

**验收**：家长周报生成与读取端点仍可用；全量测试通过。

### P1-4 删除 support 交接 / SLA / 客户生命周期

**涉及**：`support_destination_service`（919）、`customer_lifecycle_service`（468）、`support_sla_service`（304）、`support_handoff_service`（288）及 `admin.py` 对应路由。

**预计**：约 2,000 行。

**验收**：全量测试通过。

### 阶段一前端删减实际结果

| 指标 | 起点 | 现在 |
|------|------|------|
| 路由 | 92 | 79 |
| 文件 | 596 | 约 540 |
| 行数 | 48,281 | 约 40,000 |
| 运行时依赖 | 29 | 26 |

**判定标准**：把前端 184 个 `httpClient` 调用逐一比对后端路由清单，凡是调用后端不存在端点的界面即为假界面。据此发现 21 个 service 在调不存在的接口，且这些界面无一例外已被标记 `status: demo`。该标准比「看起来像不像演示」客观，建议后续继续沿用。

据此保留了 `learningProfileApi`（`/students/{id}/learning-profile` 真实存在），删除了组织后台、导师排班板、弱点诊断、课程图谱、家长月报、高级分析与留存页。

`ReportOperationsPage.tsx`（3,123 行，全前端最大文件、移动端最差界面）整体删除：其多数功能随后端报告治理路由一同消失，保留一个被掏空一半的控制台不符合目标。后端仍保留报告列表、重发、重试与恢复任务路由，将来可配一个小得多的界面。

**已知遗留**：`initialMessage` 参数会使 `POST /conversations` 返回 500，属既有缺陷，尚未定位。练习求助因此不携带该参数。

### P1-5 前端删除组织后台与 phase12 mock

**目标**：移除全部为演示数据支撑、无真实后端的表面。

**涉及**：11 条 `organization` 路由（全部 status=demo）、`src/pages/organization/`、`phase12MockData.ts`（450 行）及其 7 个消费 service（`organizationApi`、`learningProfileApi`、`curriculumGraphApi`、`diagnosisApi`、`parentMonthlyReportApi`、`tutorAssignmentApi`、`advancedAnalyticsApi`）。

**验收**：`tsc -b` 与 `eslint` 零错误；`vitest` 通过。

### P1-6 前端删除死依赖与死代码

**涉及**：
- `aws-amplify`（重包，唯一引用者 `src/lib/api.ts` 自身零引用者）
- `react-hook-form`、`@hookform/resolvers`（`src/` 内零引用）
- `src/hooks/useMockChat.ts`（79 行，无引用者）
- `src/services/demo/demoFallback.ts` 的 `withDemoFallback`（被 19 个 service 包裹，但 `allowDemoFallback` 硬编码 `false`，兜底数据打包却不可达）

**验收**：构建产物体积下降；`tsc -b`、`eslint`、`vitest` 通过。

### P1-7 修复 practice 的 teacher-help 纯桩

**目标**：使练习场景的求助真正进入升级链路。

**现状**：`practice.py` 的 `request_teacher_help` 仅生成 request_id、记一条 usage 事件、返回写死的德语文案。

**做法**：接入与 question 路径一致的升级与派单流程，复用 allowance 准入。

**验收**：练习内求助后，老师队列可见该请求。

### 阶段一测试基线实际结果 ✅

补上了两层此前完全缺失的证据。

**一、真实环境冒烟：`stoa-backend/scripts/smoke_live_flows.py`**

不能放在 `tests/` 下——conftest 用会话级 fixture 全程禁用 socket。它以真实身份驱动真实 HTTP 面：健康检查、四角色登录（含两个特权客户端）、学生与家长旅程，以及需显式开启的真人老师升级（每次消耗一个周配额）。

```
STOA_SMOKE_PASSWORD='...' python scripts/smoke_live_flows.py --include-escalation
→ 12/12 checks passed
```

首次运行即发现测试学生缺 `primary_subjects`，导致 `GET /students/me/profile` 返回 503。注册路径三个分支都会写入该字段，故属预置脚本的缺口，已补齐。

**二、接口契约守卫：`stoa-frontend/scripts/check-api-contract.mjs`**

把「前端调用比对后端路由清单」自动化，`npm run check:api-contract`。它捕获到一个真实缺陷：记忆摘要调 `/students/me/memory`，实际服务于 `/adaptive/students/me/memory`。

**仍待决策：25 条无后端路由的调用**，分布在 14 个文件。每条需决定「补后端」还是「删前端」：

| 领域 | 调用 |
|------|------|
| 支持工单 | `/support/tickets` 系列、`/admin/support/tickets` 系列 |
| 文件上传 | `POST /files`、`GET /files/{id}`、`POST /files/tutor-credentials` |
| 管理端 | `/admin/analytics/overview`、`/admin/usage-summary`、`/admin/system-status`、`/admin/help-requests`、`/admin/feedback`、`/admin/support-requests`、`/admin/billing-interest` |
| 表单提交 | `POST /contact/requests`、`POST /feedback`、`POST /support/requests`、`POST /partnership/interests` |
| 其他 | `GET /referrals/me`、`GET /students/me/learning-history`、`GET /teachers/me/profile`、`GET /teacher-help/request/{id}`、`GET /parents/me/children/{id}/practice-summary`、`POST /teachers/ai-tools/drafts/{id}/{action}` |

其中文件上传值得注意：后端有完整的 `/files/intents`、分块上传与 `complete` 流程，前端却在调不存在的 `POST /files`，属契约不匹配而非功能缺失。

### P1-9 文件上传：四个缺陷叠加导致从未成功过 ✅

按契约守卫的清单先处理文件上传（拍照提问是核心输入）。前端原本调不存在的 `POST /files`，改为实现后端真实的三段式流程后，逐层暴露出四个独立缺陷——**上传在生产环境从来没有成功过一次**。

| # | 缺陷 | 位置 |
|---|------|------|
| 1 | `claim_upload_part` 用 `isinstance(x, int)` 校验存储的 `expires_at`，拒绝 DynamoDB 返回的 Decimal | `attachment_repo.py` |
| 2 | `_require_positive_integer` 同样不接受 Decimal，导致分块列表读取失败 | `attachment_repo.py` |
| 3 | `_transition` 声明了从不使用的占位符 `:one`，DynamoDB 以 ValidationException 拒绝整个请求 | `attachment_repo.py` |
| 4 | 分块合并未回传各分块校验值，而上传创建时声明了 SHA256，S3 以 InvalidRequest 拒绝 | `attachment_service.py` |
| 5 | 图片桶未启用版本控制，而代码要求 S3 返回 `VersionId`（另三个桶都已启用） | `stoa-infra/stacks/storage_stack.py` |

诊断过程中还发现所有异常都被一个**不打日志的 catch-all** 收敛成 `upload_service_unavailable`，无法区分依赖故障与契约错误。已为其加上错误类别与 provider 错误码的记录（不输出对象键等私有坐标），这是定位缺陷 4 的关键。

**共同教训**：这五个缺陷 2,944 个测试全部无法发现。前三个是夹具用 Python int 而真实表返回 Decimal；第三个是 fake 表不像 DynamoDB 那样校验表达式；第四、五个是 fake 不模拟 S3 的校验与版本语义。已补三个回归测试，其中一个用「像 DynamoDB 一样拒绝未使用占位符」的表替身，可捕获整类问题。

**仍待处理的同类风险**：全仓库有 36 个文件、90 处 `isinstance(x, int)` 形式的存储值校验，其中多数文件在别处已处理 Decimal 却在这些位置遗漏。建议收敛为统一的规范化辅助函数。（已由 P1-10 处理。）

### P1-10 Decimal 收敛：真实风险面远小于初估 ✅

初步统计的「90 处待改」是高估。逐条追溯每个 `isinstance(x, int)` 校验所守护的值的来源后，全仓 104 处中只有 3 处守护的是真正从 DynamoDB 读回的值；其余守护的是调用方传入的时间戳、分页 limit、以及 Stripe 事件载荷等 JSON 解析结果——这些地方要求严格 `int` 是正确的，改动反而会削弱校验。因此本任务不引入全局辅助函数，只修这 3 处，并按各文件既有约定就地规范化。

| # | 缺陷 | 生产后果 |
|---|---|---|
| 1 | `activate_retention_fence` 校验从表读回的 `generation` | 每一次调用都抛 `conditional_conflict`，附件保留围栏无法启用 |
| 2 | `_optional_positive_int` 读取存储的 `attempt` 计数 | 归一化为 0，会话消息租约续期后返回 `upload_service_unavailable` |
| 3 | `get_teacher_curriculum_assignment` 校验存储的 `version` | 静默返回 `None`，教师课纲分配形同不存在 |

已实测确认真实表返回 `Decimal('1')`。三处均补回归测试，并用「改回旧写法后测试必须失败」的方式验证守卫确实有效。

**冒烟套件扩展**：将文件上传（intent → chunk → complete）纳入 `smoke_live_flows.py`，使 P1-9 的验证从一次性人工确认变为持续守卫；同时修正教师队列检查的时序缺陷——该查询是 scan，最终一致，刚写入的升级需要有界重试才稳定可见。

### P1-11 断链接口逐条决策（进行中）

守卫先修了一处误报：调用方插值的是 accept/reject/archive 三个字面量，后端各有独立路由，改为逐段匹配后不再误报。真实断链从 23 条降到 20 条，其中两条核心的已补：

**学生看不到老师是否接单**。聊天轮询的状态路由从未实现，等待指示器永远停在「刚提交」。升级状态本就存在会话记录上，因此新路由以客户端已持有的 conversationId 为键，无需为反查 requestId 增加索引。修复过程中又发现两个缺陷：一是绑定写在 `dispatched_teacher_id`，而路由读的是无人写入的 `current_teacher`；二是存储的 `escalation_status` 在教师打开案件前一直是 pending，故当已有教师绑定时以 assigned 上报。冒烟新增「学生看到请求被接走」一项。

**教师看到的是编造的自己**。`/teachers/me/profile` 从未实现，前端用一份硬编码档案兜底：打码的 IBAN、结算周期、通过的背景审查——没有任何系统记录这些。已按存储中真实存在的字段实现该路由（姓名、邮箱、账号与可用状态、派单学科、每周时段、并发上限），并删除前端的伪造数据与展示这些数据的整块界面。

**剩余 20 条的处置方向**：支持工单系统（7 条）、管理后台看板（7 条）、推荐/合作/联系/反馈表单（4 条）、学习历史与家长练习摘要（2 条）。前三类均无后端且远离学习核心，倾向删除；后两类与学习相关，需判断能否指向已有的 learning-profile 与 children/summary 路由。

### P1-13 学习核心的两条断链 ✅

**学生看不到自己的学习历史**。该页面等三个查询都有数据才渲染，因此这一条 404 会让整页报错。已实现 `/students/me/learning-history`。

实现过程中发现一个更严重的问题：`question_repo.list_by_student` 查的是以 `student_id` 为键的 GSI，而会话、消息、用量台账、上传意图、周报**都带这个字段**。整页取回等于把杂项全当成提问。首次上线时该接口把 100 条台账和系统消息标成了「Question asked」；同样的读法也存在于 `/students/{id}/summary`，导致一个从未提过问的学生被统计为 127 次提问。现按记录本身的类型筛选：历史包含提问与会话，统计只算提问。修复后历史从 100 条降为 36 条真实会话，统计从 127 变为 0。回归测试直接喂入真实索引会返回的混合行，改回整页取回即失败。

**学生档案里的监护人和信用卡是编造的**。前端把一份硬编码档案合并到 API 响应上，每个学生看到的都是同一个虚构监护人、生日、电话，以及「Visa 尾号 4242」——没有任何系统记录这些，而暗示已绑卡比不显示更糟。后端补齐了实际存储的字段（邮箱、作答语言、是否已绑定家长账号），前端删除伪造数据与展示它们的板块，套餐信息改为完全取自订阅查询。

**家长练习摘要**（`/parents/me/children/{id}/practice-summary`）没有任何页面使用，连同其 hook、两个组件、类型与 mock 生成器一并删除。

### P1-14 删除前端仓库内的演示后端 ✅

前端仓库里还跟踪着一套完整的 FastAPI 应用（`stoa-frontend/backend/`，8 个文件、磁盘 30MB），由 `demo:backend` / `demo:reset` 驱动，为演示提供假数据。前端现已接真实 API，没有任何代码引用或测试运行它，已删除。

同时修正 README：它记载的测试账号是 `*@test.com / password123`，在任何环境都不存在——这正是本次工作最初查询登录信息时被误导的源头。README 开头改为列出部署环境中真实存在的四个账号。

### P1-12 删除无后端支撑的外围界面 ✅

剩余 18 条断链分四类，均无后端且远离学习核心，全部删除：支持工单系统（4 页 + 表单）、管理后台看板（用量/反馈/求助/分析共 4 页）、推荐与合作页、联系/反馈/支持三个表单、教师资质上传。共删除 40 个文件、约 2,076 行。

判断依据是：一个提交即 404 的表单比没有表单更糟——它接收了用户输入然后丢弃。承载真实内容的页面则保留：联系页仍列出邮箱与电话，支持页仍有 FAQ 并链向联系页，只移除提交表单。

**教师资质上传值得单独说明**。后端 `TeacherApplicationRequest` 用 `extra="forbid"` 明确拒绝资质文档，并专门抛出 `credential_documents_deferred`；而前端 `toSnakeApplication` 实际只发送五个允许字段，收集到的文件 ID 从未被发送。也就是说这一步既上传失败、又无人接收，却在向申请者索要文档。删除它与后端的既有决策一致。

**结果**：前端 137 个调用全部对应真实后端路由，断链从最初的 25 条降为 0。

**加上守卫**：契约检查此前只是本地工具，这正是 25 条断链得以长期累积的原因。现已加入生产部署流水线，在 lint 与类型检查之后运行，拉取后端发布的路由清单进行比对。已验证：制造一条断链时 CI 退出码为 1。

**顺带修正的悬空链接**：类型检查无法覆盖字符串路由。管理后台首页有三张卡片、两个合作落地页的 CTA 指向已删除路径，均已修复（合作 CTA 改指联系页）。另写了一段一次性脚本比对全部 `<Link to>` 与路由表，确认无残留。

### 本轮新发现的待办

- **家长周报生成失败**：周报记录 `status=generation_failed`，错误为调用 `ListObjectVersions` 时 `AccessDenied`。需要为报告生成角色补该权限。
- **前端类型检查此前形同虚设**：项目用 `tsc -b`（带 project references），而我先前用 `npx tsc --noEmit` 校验，实际不检查任何文件、恒为通过。已改用 `npm run typecheck`，并据此发现并修复了一批真实类型错误。
- **`/students/{id}/summary` 的语义**：现在诚实地只统计拍照提问实体，而该学生的全部学习发生在会话中，故显示为 0。是否应把会话计入「提问」属产品决策，待定。
- **部署地址易混淆**：应用在 `app.stoaedu.ch`，而根域名 `stoaedu.ch` 是一套独立的静态营销站（Bootstrap 模板）。根域名的深链接返回 404 是该站自身没有对应页面，与本应用无关；应用域名的 SPA 回退配置正常。上一轮写入 README 的地址曾误为根域名，已更正。
- **功能开关残留**：`feedback`、`referrals`、`supportTickets` 三个运行时开关对应的功能已删除，但开关本身仍缠绕在发布脚本、e2e 与 i18n 中。清理它们涉及发布工具链，未纳入本轮。

### P1-8 建立真实接口冒烟测试基线

**目标**：补上从未存在的 L2 层证据 — 证明接口在真实环境可用，而非函数逻辑正确。

**做法**：新增一层打真实 sandbox 环境的契约冒烟测试，覆盖四角色登录、学生提问、升级真人老师、老师接单与回复、家长查看孩子。与现有 hermetic 单测分离，独立执行。

**优先补空白**：`teacher_applications.py`（8 条路由仅 1 个测试文件）、`adaptive.py`（15 条路由仅 2 个测试文件）。

**验收**：冒烟套件可对 sandbox 环境一键执行并全绿。

---

## 四、阶段二：把实时做成真的

目标：补齐核心卖点所依赖的实时能力。这是整个重构的重心。

### P2-1 WebSocket 基础设施

新增 API Gateway WebSocket API、`$connect` / `$disconnect` / `$default` Lambda handler、CDK 实时栈，接入既有 `WS_CONN#` 连接表。

三端代码已就绪，仅缺中间管道：后端 fanout 服务 479 行、仓储 351 行、前端客户端 `useRealtimeNotifications.ts` 181 行（含心跳、指数退避重连、离线降级）。

同一通道承载：老师回复推送、白板协作同步、老师上线 / 接单提示。

### P2-2 修复派单可靠性

- 部署既有 SQS 消费者 Lambda `jobs/teacher_escalation.py`
- 新增 EventBridge 定时器自动重派超时任务，替代人工调用
- 为老师候选查询新增 GSI，替换全表扫描

### P2-3 合并双升级路径

统一 question 与 conversation 两套重复实现为单一链路。

### P2-4 真流式

将伪流式改为真实逐字输出。需改变传输层（Lambda Function URL response streaming 或 WebSocket），并改用 Bedrock 的流式调用。

### P2-5 白板与数学输入

- **Excalidraw**（MIT，可商用嵌入，`@excalidraw/excalidraw` React 组件）
- **Yjs** CRDT 协作，走 P2-1 的同一条 WebSocket
- **MathLive** 数学公式输入（自带移动端虚拟数学键盘、语音朗读、ARIA）

**明确排除 tldraw**：其 SDK 自 4.0（2025-09）起为自定义 source-available 授权，生产环境商用需付费并配置 license key。大量资料仍错标为 MIT。

GeoGebra 如需引入，须先单独确认商用授权。

### P2-6 前端信息架构收敛

功能保留，入口收敛。学生顶层入口从十余个降为两个：

```
/chat                 唯一主界面
  ├─ 顶部卡片：今日复习 + 打卡状态
  ├─ 侧栏：历史会话
  └─ 对话内：升级真人老师 / 打开白板 / 插入公式
/learn                二级容器（题库 / 练习 / 错题本）
/teacher/queue        老师端
/parent               家长端
/admin                精简后的管理端
```

### P2-7 移动端补齐

现状：外壳已就绪（`AppLayout` 有独立底部导航、安全区适配、viewport 正确），短板在内容页 — 88 个页面中 29 个零响应式，全项目零触摸事件处理。

阶段一删除管理端页面后压力大幅缓解，此处补齐剩余内容页。

---

## 五、阶段三：题库互动化

### P3-1 FSRS 调度底座

引入 `py-fsrs`（后端）与 `ts-fsrs`（前端），均为 MIT。相同记忆保持率下比传统 SM-2 减少 20–30% 复习量，为 Anki 自 23.10 起的默认算法。

错题本从「越堆越长的列表」变为「今天该复习的 N 题」。

### P3-2 打卡机制

- 打卡判定绑定「完成当日到期复习」，而非「打开应用」或「做任意一题」，以规避 goal drift
- 补签卡上限 2 张（无上限的补签会使打卡失去意义）
- 里程碑设于 7 / 30 / 100 / 365 天
- 可选同伴打卡（对日完成率有显著正向作用）

### P3-3 与真人老师形成闭环

```
连续 2 次答不上来 → 提示找老师 → 老师讲解
→ 题目连同讲解注入 FSRS 队列 → 7 天后回访
```

使真人答疑从一次性消费转为长期记忆闭环。

---

## 六、目标产出

| 指标 | 现状 | 目标 |
|------|------|------|
| 后端端点 | 237 | ~170（阶段一后）→ 按需再收敛 |
| 后端行数 | 96,781 | ~80,000（阶段一后） |
| 前端路由 | 92 | ~70（阶段一后），顶层入口 2 个 |
| 学生感知老师回复 | 轮询 | 推送 |
| 流式 | 伪流式 | 真流式 |
| 接口真实验证 | 无 | 核心流程全覆盖 |

注：因决策 1、3 保留了计费与学习模块，删减幅度小于早期设想，收敛重心转为入口与导航层级。

---

## 七、风险

1. **删除牵连测试**：后端删除服务时须同步删除其测试文件，否则全量 pytest 会红。每步删除后立即跑 `ruff check` 与 `pytest`。
2. **allowance 耦合**：`teacher_support_allowance_service` 门控两条升级路径，P1-7 修桩时须复用同一准入器，不得绕过。
3. **account fence 织入面广**：`account_fence_generation` 织入几乎所有写路径（含教师派单条件），属合规基建，任何阶段均不得删除。
4. **WebSocket 成本模型**：P2-1 前需确认白板是全量开放还是限定场景，这决定并发规模。
5. **题库内容缺口**：`mockQuestionBank.ts` 为 657 行假数据。FSRS 算法接入约 2–3 天，真实内容生产可能需数月，两者不可混为一谈。
