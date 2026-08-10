# STOA 2.0 详细开发计划

> **版本：** v1.1  
> **创建日期：** 2026-08-10  
> **最后核对：** 2026-08-10（逐条比对代码，见 DESIGN_2.0.md §〇）  
> **关联文档：** [DESIGN_2.0.md](./DESIGN_2.0.md) · [PLAN.md](./PLAN.md)  
> **周期：** 每 Sprint 2 周，共 7 个 Sprint（14 周）

---

## 状态总览

**任务完成 14 / 40（35%）** · 部分完成 5 · 未开始 21 · 被决策阻塞 5

| Sprint | 主题 | 周期 | 状态 |
|--------|------|------|------|
| Sprint 0 | 前后端联调 + 清理收尾 | 第 1–2 周 | 🔄 清理已完成，**联调三项未做** |
| Sprint 1 | 学生记忆层 + 多轮追问 | 第 3–4 周 | ✅ 代码完成（未经真实链路验证） |
| Sprint 2 | Claude Vision + 流式 UX + LaTeX | 第 5–6 周 | 🟡 UX/LaTeX 完成；Vision 阻塞 |
| Sprint 3 | 自适应推荐 + WebSocket | 第 7–8 周 | 🟡 推荐完成；WebSocket 未部署 |
| Sprint 4 | 叙述式报告 + 老师 AI 2.0 | 第 9–10 周 | 🟡 报告 2.0 前已实现；老师工具未开始 |
| Sprint 5 | Bedrock Knowledge Base + PWA | 第 11–12 周 | ⛔ 两项均待决策 |
| Sprint 6 | Bedrock Agents（Premium）| 第 13–14 周 | ⏳ 待启动 |

> 🚨 **顺序被打乱了。** Sprint 1–3 已在 Sprint 0 未交付的前提下完成开发，
> 意味着这些 AI 功能只有自动化测试覆盖，从未跑通真实 Cognito → API → UI 链路。
> 建议下一步回到 Sprint 0 的联调、测试账号、端到端三项，再继续 Sprint 4。

---

## Sprint 0：联调 + 收尾（第 1–2 周）

### 目标
消除所有 demo 数据依赖，建立真实的前后端联调链路。

### 已完成 ✅
- [x] 删除 8 个 stub 遗留页面
- [x] 合并双首页（`/` → HomeV2Page）
- [x] 删除 `src/stores/` 死代码
- [x] `.planning/` 加入 `.gitignore`
- [x] 两仓库与 `origin/main` 同步

### 剩余任务

> **S0-01 MSW 已从浏览器包撤出（2026-08-10）**：原方案用 `VITE_ENABLE_MSW` 在
> `main.tsx` / `src/lib/env.ts` 中开关，这违反了"runtime-config.json 是运行时唯一
> 真相源"的架构约束，直接打破 `tests/release/runtime-startup-barrier.test.mjs` 与
> `runtime-env-projection.test.mjs` 两个发布门禁。handlers 保留，改由
> `tests/mswServer.ts`（`msw/node`）在测试侧消费，mock 代码不再进入生产包。

#### S0-01：移除 demo backend，引入 MSW
**Owner**：前端  
**工时**：10h  
**依赖**：无  

**步骤**：
1. 安装 MSW：`npm install msw --save-dev`
2. 初始化：`npx msw init public/`
3. 创建 handler 文件，覆盖所有 demo API 调用：
   ```
   src/mocks/
   ├── browser.ts          # MSW worker 启动入口
   ├── handlers/
   │   ├── auth.ts         # /auth/* mock
   │   ├── questions.ts    # /questions/* mock
   │   ├── conversations.ts
   │   ├── practice.ts
   │   ├── parents.ts
   │   └── index.ts        # 汇总导出
   └── data/
       ├── users.ts        # 测试用户数据
       └── questions.ts    # 示例题目数据
   ```
4. 修改 `src/main.tsx`，在 `VITE_API_MODE=mock` 时启动 MSW worker
5. 删除 `stoa-frontend/backend/` 目录
6. 删除 `package.json` 中的 `demo:backend`、`demo:reset` scripts
7. 删除 `src/services/practice/practiceApi.ts` 中的 `withPracticeDemo` wrapper

**验收标准**：
- `VITE_API_MODE=mock npm run dev` 下，所有页面正常渲染无控制台报错
- `VITE_API_MODE=production npm run dev` 下，请求打到真实 API

---

#### S0-02：Cognito 测试账号创建
**Owner**：后端  
**工时**：2h  
**依赖**：无  

在 AWS Console 中手动创建 dev 环境测试账号：

| 角色 | 邮箱 | 密码 | 备注 |
|------|------|------|------|
| student | `test.student@stoaedu.ch` | `TestPass2026!` | Grade 9，数学 |
| parent | `test.parent@stoaedu.ch` | `TestPass2026!` | 绑定 student |
| tutor | `test.tutor@stoaedu.ch` | `TestPass2026!` | 已激活 |
| admin | `test.admin@stoaedu.ch` | `TestPass2026!` | — |

用后端 `scripts/provision_sandbox_accounts.py` 脚本执行，并在 DynamoDB 中验证 user profile 已创建。

**验收标准**：4 个账号均可在 `app.stoaedu.ch` 登录，跳转到对应角色首页。

---

#### S0-03：前后端全链路联调
**Owner**：全栈  
**工时**：20h  
**依赖**：S0-02  

**联调顺序**（逐步验证，每步通过再继续）：

**阶段 1：认证链路**
- [ ] 注册新账号 → 验证邮件 → 激活
- [ ] 登录 → 获得 accessToken / refreshToken → 存入 localStorage
- [ ] Token 过期自动刷新（`axios` 拦截器）
- [ ] 登出 → token 清除 → 跳转 `/login`

**阶段 2：学生核心流程**
- [ ] 学生登录 → 跳转 `/dashboard`（真实数据）
- [ ] 发起提问（文字）→ AI 回答返回
- [ ] 上传图片 → OCR → 题目识别 → AI 回答
- [ ] 请求老师帮助 → 状态变更为 `teacher_requested`

**阶段 3：老师流程**
- [ ] 老师登录 → 查看任务队列（含上一步的请求）
- [ ] 接管 → AI 静默 → 回复 → 标记解决

**阶段 4：家长流程**
- [ ] 家长登录 → 查看子账号学习摘要
- [ ] 查看周报（需手动触发一次报告生成）

**阶段 5：计费流程**
- [ ] 学生/家长 → 进入计费页 → 点击升级 → Stripe Checkout
- [ ] Webhook 回调 → 订阅状态更新

**修复记录**（边联调边填写）：
| 问题 | 位置 | 修复方式 | 状态 |
|------|------|---------|------|
| | | | |

**验收标准**：以上所有步骤均在生产 API 下跑通，无 mock 数据依赖。

---

#### S0-04：替换轮询为短轮询优化（临时方案）
**Owner**：前端  
**工时**：3h  
**依赖**：S0-03  

在 Sprint 3 的 WebSocket 到来前，先优化现有轮询间隔：

| 文件 | 当前 | 优化后 |
|------|------|--------|
| `useTeacherHelpStatusQuery.ts` | 条件轮询 | 仅在 `teacher_requested` 状态时 5s 轮询，其余停止 |
| `useTeacherAvailabilityQuery.ts` | 60s | 120s（降低频率） |
| `useCheckoutCommandQuery.ts` | 条件轮询 | 仅在 pending 状态时轮询，完成后停止 |

---

## Sprint 1：学生记忆层 + 多轮追问（第 3–4 周）

### 目标
AI 能记住每个学生的薄弱点，学生可以就同一道题追问。

### 依赖
Sprint 0 全部完成（真实 API 联通）

---

### 后端任务

#### S1-BE-01：DynamoDB `student_memory` 实体设计与 Repository
**Owner**：后端  
**工时**：6h  

**DynamoDB 实体设计**：

```
主表条目：
  PK: STUDENT_MEMORY#<student_id>
  SK: CONCEPT#<concept_slug>       # e.g. CONCEPT#quadratic-equations
  entity_type: "student_memory"
  student_id: str
  concept_slug: str                # URL-safe 标识符
  concept_name_de: str             # 德语概念名 "Quadratische Gleichungen"
  subject: str                     # "mathematics"
  grade_range: str                 # "7-9" / "gymnasium"
  error_count: int                 # 累计错误次数（Decimal）
  correct_count: int               # 累计正确次数
  mastery_level: Decimal           # 0.00–1.00
  error_types: list[str]           # ["calculation_error", "sign_confusion"]
  last_error_at: str               # ISO datetime
  last_seen_at: str                # 最近一次相关提问时间
  updated_at: str

GSI（按 student + subject 查询）：
  GSI: student_memory_by_subject
  PK: student_id
  SK: subject#concept_slug
```

**新建文件**：`src/stoa/db/repositories/memory_repo.py`

```python
# 必须实现的方法：
async def upsert_concept(student_id, concept_slug, update_data) -> None
async def get_weak_concepts(student_id, subject, limit=10) -> list[MemoryConcept]
async def get_concept(student_id, concept_slug) -> MemoryConcept | None
async def batch_upsert(student_id, concepts: list[ConceptUpdate]) -> None
```

**新建文件**：`src/stoa/models/memory.py`

```python
class MemoryConcept(BaseModel):
    student_id: str
    concept_slug: str
    concept_name_de: str
    subject: str
    error_count: int
    correct_count: int
    mastery_level: float
    error_types: list[str]
    last_error_at: datetime | None

class ConceptUpdate(BaseModel):
    concept_slug: str
    concept_name_de: str
    subject: str
    is_error: bool
    error_type: str | None = None
```

**测试文件**：`tests/test_memory_repo.py`
- 测试 upsert 幂等性（同一 concept 多次 upsert 不重复创建）
- 测试 mastery_level 计算（errors/(errors+correct)）
- 测试 get_weak_concepts 按 mastery_level 升序返回

**验收标准**：所有 test 通过，DynamoDB 条目格式正确。

---

#### S1-BE-02：`memory_service.py` — 概念提取 + 记忆更新
**Owner**：后端  
**工时**：8h  
**依赖**：S1-BE-01  

**新建文件**：`src/stoa/services/memory_service.py`

**核心逻辑**：

```python
async def extract_and_update_memory(
    student_id: str,
    question_text: str,
    ai_response: dict,           # 现有 ai_service 返回的结构
    student_feedback: int | None  # 1-5 星，None 表示未评分
) -> None:
    """
    从 AI 响应中提取知识点，判断是否出错，更新学生记忆。
    
    逻辑：
    1. ai_response["knowledge_points"] 已包含知识点列表（ai_service 已输出）
    2. suggest_teacher=True 或 feedback <= 2 → 判定为错误，更新 error_count
    3. feedback >= 4 → 判定为正确，更新 correct_count
    4. 重新计算 mastery_level = correct_count / (correct_count + error_count)
    5. batch_upsert 到 DynamoDB
    """

async def get_student_profile_summary(
    student_id: str,
    subject: str,
    top_n: int = 5
) -> str:
    """
    生成用于注入 system prompt 的学生画像摘要（纯文本）。
    
    输出示例：
    "该学生在 [二次方程] 上错误 5 次（主要：符号混淆），掌握度 20%。
     在 [函数图像] 上错误 2 次，掌握度 60%。"
    """
```

**调用时机**（修改 `questions.py`）：
- 题目标记为 `ai_answered` 后，异步调用 `extract_and_update_memory`
- 学生提交 feedback 后，更新对应概念的 correct/error 计数

**测试**：
- Mock ai_response，验证提取的 knowledge_points 正确写入 memory
- 验证 feedback=1 触发 error 更新，feedback=5 触发 correct 更新

---

#### S1-BE-03：`ai_service.py` 注入学生画像
**Owner**：后端  
**工时**：4h  
**依赖**：S1-BE-02  

**修改文件**：`src/stoa/services/ai_service.py`

在现有 `SYSTEM_PROMPT` 末尾增加可选的画像注入块：

```python
SYSTEM_PROMPT_MEMORY_ADDON = """
Student Learning Profile:
{student_profile}

Instructions with profile:
- For weak concepts listed above, add extra cautionary notes at relevant steps.
- Acknowledge known struggle patterns without making the student feel bad.
- If this concept appears in weak areas, suggest the student practice it more.
"""

# 修改现有调用签名，增加 student_id 参数
async def generate_ai_response(
    question: str,
    subject: str,
    grade: str,
    language: str,
    conversation_history: list,
    student_id: str | None = None,   # 新增
) -> AIResponse:
    system = SYSTEM_PROMPT.format(...)
    
    if student_id:
        profile = await memory_service.get_student_profile_summary(student_id, subject)
        if profile:
            system += SYSTEM_PROMPT_MEMORY_ADDON.format(student_profile=profile)
    
    # ... 后续逻辑不变
```

**验收标准**：
- 有 student_memory 记录的学生，AI 响应 system prompt 中包含画像内容
- 无 memory 的新学生，行为与 v1.x 相同（backward compatible）

---

#### S1-BE-04：多轮追问 API 端点
**Owner**：后端  
**工时**：6h  
**依赖**：S1-BE-03  

**修改文件**：`src/stoa/routers/questions.py`

**新增端点**：

```
POST /questions/{question_id}/chat
```

**Request**：
```json
{
  "message": "我还是不理解第二步，为什么符号要变？",
  "message_index": 0   // 可选，指明是对第几步的追问
}
```

**Response**（SSE 流式）：
```
data: {"type":"chunk","content":"这是因为..."}
data: {"type":"chunk","content":"当我们移项时..."}
data: {"type":"done","knowledge_points":["linear-equations"],"suggest_teacher":false}
```

**实现逻辑**：
1. 从 DynamoDB 加载该题目的现有 AI 解答（作为对话历史）
2. 附加学生新追问
3. 调用 `ai_service.generate_ai_response`（携带 conversation history + student_id）
4. 以 SSE 方式流式返回
5. 回答完成后异步更新 memory

**授权**：仅题目归属学生或接管老师可调用。

**新增端点**：

```
GET /questions/{question_id}/thread
```

**Response**：
```json
{
  "question_id": "...",
  "original_answer": {...},
  "follow_ups": [
    {"role": "student", "content": "...", "created_at": "..."},
    {"role": "ai", "content": "...", "created_at": "...", "knowledge_points": [...]}
  ]
}
```

---

### 前端任务

#### S1-FE-01：Dashboard「薄弱知识点」卡片
**Owner**：前端  
**工时**：6h  
**依赖**：S1-BE-02（API）  

**新建文件**：`src/components/dashboard/WeakConceptsCard.tsx`

```typescript
// 展示学生当前 Top 5 薄弱知识点
// 数据来源：GET /students/me/memory?subject=all&limit=5
// 每个知识点：名称 + 掌握度进度条 + 错误次数标签
// 点击跳转 /practice 并预筛选该知识点
```

**新建 Hook**：`src/hooks/student/useStudentMemoryQuery.ts`

```typescript
export function useStudentMemoryQuery(subject?: string) {
  return useQuery({
    queryKey: ['student', 'memory', subject],
    queryFn: () => studentApi.getMemory({ subject }),
  })
}
```

**新建 API**：在 `src/services/student/studentApi.ts` 中新增：

```typescript
getMemory(params: { subject?: string; limit?: number }): Promise<MemoryResponse>
```

**修改文件**：`src/pages/dashboard/StudentDashboardPage.tsx`
- 在顶部 Stats 区域下方插入 `<WeakConceptsCard />`

**验收标准**：
- 新注册学生：卡片显示"暂无学习记录"空状态
- 有提问历史的学生：显示真实薄弱点，掌握度进度条正确

---

> ### ⚠️ 测试方式修正（2026-08-10）
> 早前为这些任务写的"单元测试"把被测逻辑**在测试文件里抄了一遍**，因此无论生产代码
> 如何变化都恒为绿。改成驱动真实代码后，一次性暴露出 5 个此前完全没被发现的缺陷：
>
> | # | 缺陷 | 影响 | 只有真实测试能发现的原因 |
> |---|------|------|------------------------|
> | 1 | 记忆摘要按 snake_case 读取，实际返回 camelCase | 薄弱知识点卡片恒空、AI 个性化恒空 | 镜像测试用的是自己造的数据 |
> | 2 | `actor.user` 属性不存在，每次请求抛 `AttributeError` 被宽 except 吞掉 | **个性化在生产中完全失效** | 镜像测试从不调用真实 `_execute_message_command` |
> | 3 | `memory_context` 绕过 `_sanitise_input` | 学生可构造提问，让注入文本进入 system prompt | 镜像测试不走真实 `get_ai_answer` |
> | 4 | 记忆查询（3 次表读）位于 AI 租约临界区内 | 并发重复请求会超出有界重放等待，返回虚假 `message_in_progress` | 需真实并发测试 |
> | 5 | `MathRenderer` 畸形公式降级分支未转义即拼入 `dangerouslySetInnerHTML` | **AI 输出可注入 DOM 节点（XSS）** | 需真实 jsdom 渲染 |
>
> 前端此前没有任何组件测试运行器，现已引入 vitest + jsdom + Testing Library
> （仅 devDependency，`vitest.config.ts` 与 `vite.config.ts` 分离，不影响生产构建）。
> 命令：`npm test`。当前 44 个组件测试 + 后端 22 个真实 AI 链路测试。

#### S1-FE-02：解答页多轮追问 UI ✅ 已完成（2026-08-10）
> 多轮对话链路（`ChatPage` + `/conversations/{id}/messages`）已存在，实际补齐的是快捷追问入口：
> 新增 `src/components/chat/FollowUpSuggestions.tsx`，在最后一条**已完成**的 AI 消息下渲染 4 个
> 预设追问 chip；流式进行中或消息失败时不显示（`lastCompletedAssistantIndex` 判定）。
> 测试：`tests/unit/followUpAnchor.test.mjs`（8 例，覆盖流式中/失败/仅学生消息/teacher 消息等边界）。

**Owner**：前端  
**工时**：10h  
**依赖**：S1-BE-04（API）、S1-FE-03（流式渲染组件）  

**修改文件**：`src/pages/chat/ChatPage.tsx`（或解答展示页）

**新增 UI 区域**：
```
┌─────────────────────────────────┐
│  原始 AI 解答（已有）               │
├─────────────────────────────────┤
│  追问区域（新增）                   │
│  ┌─ 追问历史 ─────────────────┐   │
│  │  [学生] 第二步为什么要…     │   │
│  │  [AI]  这是因为移项时…      │   │
│  └──────────────────────────┘   │
│  ┌─────────────────────────────┐│
│  │ 还有问题？继续追问…          ││
│  └──────────────────────────── ││
│  [发送]                          │
└─────────────────────────────────┘
```

**新建 Hook**：`src/hooks/chat/useQuestionChatMutation.ts`

```typescript
// 发送追问，返回 ReadableStream
export function useQuestionChatMutation(questionId: string) {
  return useMutation({
    mutationFn: (message: string) => 
      chatApi.sendFollowUp(questionId, message),
  })
}
```

**新建 Hook**：`src/hooks/chat/useQuestionThreadQuery.ts`

```typescript
export function useQuestionThreadQuery(questionId: string) {
  return useQuery({
    queryKey: ['question', questionId, 'thread'],
    queryFn: () => chatApi.getThread(questionId),
    enabled: !!questionId,
  })
}
```

**验收标准**：
- 解答页底部显示追问输入框
- 发送追问 → 流式打字机效果出现回答
- 追问历史折叠展示，最多显示最近 3 轮

---

#### S1-FE-03：新增后端 API 端点
**Owner**：后端  
**工时**：3h  
**依赖**：S1-BE-01  

在 `src/stoa/routers/students.py` 中新增：

```
GET /students/me/memory?subject={subject}&limit={n}
```

**Response**：
```json
{
  "concepts": [
    {
      "concept_slug": "quadratic-equations",
      "concept_name_de": "Quadratische Gleichungen",
      "subject": "mathematics",
      "mastery_level": 0.25,
      "error_count": 5,
      "correct_count": 2,
      "error_types": ["sign_confusion"],
      "last_error_at": "2026-08-09T18:30:00Z"
    }
  ],
  "total": 12,
  "weak_count": 4
}
```

---

## Sprint 2：Claude Vision + 流式 UX + LaTeX（第 5–6 周）

### 目标
图片识别升级到 95% 准确率，AI 回答实现打字机流式效果，数学公式正确渲染。

### 依赖
Sprint 1 完成（memory 层可用）

---

### 后端任务

#### S2-BE-01：Claude Vision 替换 Rekognition OCR
**Owner**：后端  
**工时**：8h  

**修改文件**：`src/stoa/services/ocr_service.py`

**核心变更**：

```python
async def extract_text_from_attachment_v2(
    attachment: dict,
    student_language: str = "de",
) -> OcrResult:
    """
    使用 Claude Haiku vision 直接处理图片，同时完成：
    1. OCR 文字提取
    2. 题目结构理解（题干、已知、求解）
    3. 知识点预识别（传给 memory 层）
    4. 难度评估
    
    相比 Rekognition：
    - 手写体识别率 70% → 95%+
    - 无需单独 OCR 步骤
    - 直接获得题目结构
    """
    image_bytes = await s3.download(attachment["immutable_object_key"])
    
    response = bedrock.converse(
        modelId=settings.bedrock_model_id,  # claude-3-haiku
        messages=[{
            "role": "user",
            "content": [
                {"image": {"format": "jpeg", "source": {"bytes": image_bytes}}},
                {"text": f"""
                请分析这道 {student_language} 数学题目图片，返回 JSON：
                {{
                  "question_text": "完整题目文本",
                  "question_type": "algebra|geometry|calculus|statistics|other",
                  "known_values": ["已知：..."],
                  "what_to_find": "求：...",
                  "knowledge_points": ["知识点1", "知识点2"],
                  "estimated_difficulty": "easy|medium|hard",
                  "has_diagram": false,
                  "confidence": 0.95
                }}
                若无法识别，返回 {{"error": "无法识别", "confidence": 0.0}}
                """}
            ]
        }]
    )
    return parse_ocr_result(response)
```

**Feature Flag**：通过 `settings.enable_vision_ocr = True/False` 控制，默认 True，可快速回滚。

**修改文件**：`src/stoa/config.py` — 新增 `enable_vision_ocr: bool = True`

**测试**：
- 10 道手写数学题测试集（不同清晰度、不同笔迹）
- 验证 95%+ 识别率（vs Rekognition 70%）
- 验证 `has_diagram=True` 情况的处理

---

#### S2-BE-02：AI prompt 增加 LaTeX 输出指令
**Owner**：后端  
**工时**：3h  
**依赖**：无  

**修改文件**：`src/stoa/services/ai_service.py`

在 `SYSTEM_PROMPT` 中增加 LaTeX 格式化要求：

```python
SYSTEM_PROMPT_MATH_FORMAT = """
Mathematical formatting rules:
- ALWAYS use LaTeX for mathematical expressions
- Inline math: $expression$ (e.g., $x^2 + 2x + 1 = 0$)
- Block math: $$expression$$ for standalone equations
- Greek letters: $\\alpha$, $\\beta$, $\\theta$
- Fractions: $\\frac{a}{b}$
- Square root: $\\sqrt{x}$
- Summation: $\\sum_{i=1}^{n}$
- Do NOT use plain text for math (x^2 is wrong, $x^2$ is correct)
"""
```

将此 addon 追加到所有数学 subject 的 system prompt。

**测试**：验证 AI 输出的步骤中包含 `$...$` 格式的 LaTeX。

---

### 前端任务

#### S2-FE-01：KaTeX 数学公式渲染组件
**Owner**：前端  
**工时**：5h  

**安装**：
```bash
npm install katex
npm install -D @types/katex
```

**新建文件**：`src/components/chat/MathRenderer.tsx`

```typescript
import katex from 'katex'
import 'katex/dist/katex.min.css'

interface Props {
  content: string  // 可能包含 $...$ 和 $$...$$ 的混合文本
}

export function MathRenderer({ content }: Props) {
  // 解析文本，将 LaTeX 块替换为 KaTeX 渲染的 HTML
  // 支持 $inline$ 和 $$block$$ 两种格式
  // 错误回退：LaTeX 渲染失败时显示原始文本
}

// 工具函数：在流式输出中安全渲染（等到 $ 闭合后再渲染）
export function renderMathSafe(text: string): string { ... }
```

**修改文件**：`src/components/chat/MessageBubble.tsx`
- 将纯文本渲染替换为 `<MathRenderer content={message.content} />`

**验收标准**：
- `$x^2 + 2x + 1 = 0$` 渲染为格式化公式
- `$$\frac{-b \pm \sqrt{b^2-4ac}}{2a}$$` 渲染为块级公式
- 非数学文本不受影响

---

#### S2-FE-02：流式打字机效果
**Owner**：前端  
**工时**：8h  

**新建文件**：`src/hooks/chat/useStreamingResponse.ts`

```typescript
interface StreamingState {
  content: string
  isStreaming: boolean
  isDone: boolean
  error: Error | null
}

export function useStreamingResponse() {
  const [state, setState] = useState<StreamingState>(initialState)
  
  const startStream = useCallback(async (
    url: string,
    body: object,
    token: string
  ) => {
    setState({ content: '', isStreaming: true, isDone: false, error: null })
    
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`,
        Accept: 'text/event-stream',
      },
      body: JSON.stringify(body),
    })
    
    const reader = response.body?.getReader()
    const decoder = new TextDecoder()
    
    while (true) {
      const { done, value } = await reader!.read()
      if (done) break
      
      const chunk = decoder.decode(value)
      // 解析 SSE: "data: {...}\n\n"
      for (const line of chunk.split('\n')) {
        if (!line.startsWith('data: ')) continue
        const event = JSON.parse(line.slice(6))
        
        if (event.type === 'chunk') {
          setState(prev => ({ ...prev, content: prev.content + event.content }))
        } else if (event.type === 'done') {
          setState(prev => ({ ...prev, isStreaming: false, isDone: true }))
        }
      }
    }
  }, [])
  
  return { ...state, startStream }
}
```

**修改文件**：Chat/解答相关页面
- 替换现有一次性渲染为流式渲染
- 添加光标动画（`|` blinking cursor）在流式过程中
- 流式完成后渲染 KaTeX

**新建文件**：`src/components/chat/StreamingCursor.tsx`
```typescript
// 单一闪烁光标组件，在 isStreaming=true 时显示
```

**验收标准**：
- AI 开始输出 < 1s（首字节延迟）
- 文字逐字出现，有流畅打字机动画
- LaTeX 在流完成后正确渲染（不在流中间渲染，避免 partial LaTeX）

---

#### S2-FE-03：骨架屏优化
**Owner**：前端  
**工时**：3h  
**依赖**：S2-FE-02  

**修改**：解答页、追问区域的加载态

```
等待 AI 响应时（isStreaming=false, 但请求已发送）：
┌───────────────────────────────┐
│  AI 正在思考...                 │
│  ████████████ 30%             │
│  ░░░░░░░░░░░░░░░░░░░░░░░░    │  ← 骨架占位
│  ░░░░░░░░░░░░░░░             │
└───────────────────────────────┘

流式开始后：
┌───────────────────────────────┐
│  步骤 1：首先我们需要...          │
│  ░░░░░░░░░░░░░░░░░░░░░░░░    │  ← 后续骨架
└───────────────────────────────┘
```

---

## Sprint 3：自适应推荐 + WebSocket（第 7–8 周）

### 目标
AI 主动推荐每日练习，实时状态更新替代轮询。

---

### 后端任务

#### S3-BE-01：自适应推荐算法完善
**Owner**：后端  
**工时**：8h  
**依赖**：S1-BE-02（memory）  

**修改文件**：`src/stoa/routers/adaptive.py`（已有）

**完善 `GET /adaptive/recommendations` 的逻辑**：

```python
def compute_recommendation_score(concept: MemoryConcept) -> float:
    """
    基于遗忘曲线 + 错误频率的推荐权重。
    
    score = error_weight × recency_weight × (1 - mastery_level)
    
    error_weight = min(error_count / 10, 1.0)   # 错误越多越优先
    
    days_since_error = (now - last_error_at).days
    recency_weight = exp(-days_since_error / 7)  # 7天半衰期
    
    最终 score ∈ [0, 1]，越高越需要练习
    """

async def get_daily_recommendations(
    student_id: str,
    subject: str | None = None,
    limit: int = 5,
) -> list[RecommendedPractice]:
    """
    返回今日推荐练习题列表：
    1. 从 memory 中获取薄弱概念，按 score 排序
    2. 为每个概念匹配对应的练习题（从 practice 题库）
    3. 添加推荐理由文案（"根据你上周在 X 上的错误…"）
    """
```

**新增 Response 字段**：

```json
{
  "recommendations": [
    {
      "concept_slug": "quadratic-equations",
      "concept_name_de": "Quadratische Gleichungen",
      "reason_de": "Du hast diese Woche 3 Fehler bei quadratischen Gleichungen gemacht.",
      "practice_set_id": "quad-eq-practice-01",
      "estimated_minutes": 10,
      "priority_score": 0.85
    }
  ],
  "generated_at": "2026-08-10T...",
  "next_refresh_at": "2026-08-11T08:00:00Z"
}
```

---

#### S3-BE-02：WebSocket API Gateway 部署
**Owner**：后端 + 基础设施  
**工时**：10h  

**新建文件**：`stoa-infra/stacks/stoa_websocket_stack.py`

```python
from aws_cdk import (
    Stack, Duration,
    aws_apigatewayv2 as apigw,
    aws_lambda as lambda_,
    aws_dynamodb as dynamodb,
)

class StoaWebSocketStack(Stack):
    def __init__(self, scope, id, *, database_table, api_lambda, **kwargs):
        super().__init__(scope, id, **kwargs)
        
        # WebSocket API
        self.ws_api = apigw.WebSocketApi(self, "StoaWsApi",
            connect_route_options=apigw.WebSocketRouteOptions(
                integration=apigw.WebSocketLambdaIntegration(
                    "ConnectIntegration", connect_handler
                )
            ),
            disconnect_route_options=...,
            default_route_options=...,
        )
        
        # connection_id → student_id 映射存入主表
        # PK: WS_CONNECTION#<connection_id>
        # SK: META
        # Attributes: student_id, connected_at, ttl (24h)
```

**新建文件**：`src/stoa/routers/websocket.py`

```python
# Lambda Handlers（通过 main.py 挂载）

async def handle_connect(connection_id: str, student_id: str) -> None:
    """存储 connection_id → student_id 映射，TTL 24h"""

async def handle_disconnect(connection_id: str) -> None:
    """删除连接记录"""

async def push_event(student_id: str, event: dict) -> None:
    """
    通过 API Gateway Management API 推送事件到客户端。
    若连接不存在（GoneException），静默忽略。
    """
    connection_id = await get_connection_id(student_id)
    if not connection_id:
        return
    try:
        await apigw_management.post_to_connection(
            ConnectionId=connection_id,
            Data=json.dumps(event).encode()
        )
    except GoneException:
        await handle_disconnect(connection_id)
```

**事件类型定义**：
```python
class WsEventType(StrEnum):
    ANSWER_READY = "answer_ready"
    TEACHER_TOOK_OVER = "teacher_took_over"  
    TEACHER_REPLIED = "teacher_replied"
    REPORT_READY = "report_ready"
    NOTIFICATION = "notification"
```

**触发点（修改现有代码）**：
- `questions.py`：AI 回答完成后 → `push_event(student_id, {type: ANSWER_READY, ...})`
- `teachers.py`：老师接管/回复后 → `push_event(student_id, {type: TEACHER_REPLIED, ...})`
- `jobs/weekly_reports.py`：报告生成后 → `push_event(student_id, {type: REPORT_READY})`

---

#### S3-BE-03：`/adaptive/feedback` 端点
**Owner**：后端  
**工时**：3h  
**依赖**：S1-BE-02  

**新增端点**：`POST /adaptive/feedback`

```json
// Request
{
  "practice_set_id": "quad-eq-practice-01",
  "concept_slug": "quadratic-equations",
  "result": "correct|incorrect|partial",
  "time_spent_seconds": 180
}

// Response
{
  "memory_updated": true,
  "new_mastery_level": 0.45,
  "encouragement_de": "Gut gemacht! Du machst Fortschritte."
}
```

---

### 前端任务

#### S3-FE-01：WebSocket 连接 Hook
**Owner**：前端  
**工时**：8h  
**依赖**：S3-BE-02  

**新建文件**：`src/hooks/useWebSocket.ts`

```typescript
const WS_BASE_URL = import.meta.env.VITE_WEBSOCKET_BASE_URL

interface WebSocketEvent {
  type: 'answer_ready' | 'teacher_took_over' | 'teacher_replied' | 'report_ready' | 'notification'
  payload: Record<string, unknown>
}

export function useWebSocket() {
  const { accessToken } = useAuthStore()
  const queryClient = useQueryClient()
  const wsRef = useRef<WebSocket | null>(null)
  
  useEffect(() => {
    if (!accessToken || !WS_BASE_URL) return
    
    const ws = new WebSocket(`${WS_BASE_URL}?token=${accessToken}`)
    wsRef.current = ws
    
    ws.onmessage = (event) => {
      const msg: WebSocketEvent = JSON.parse(event.data)
      handleEvent(msg, queryClient)
    }
    
    ws.onclose = () => {
      // 指数退避重连，最多重试 5 次
      scheduleReconnect()
    }
    
    return () => ws.close()
  }, [accessToken])
}

function handleEvent(event: WebSocketEvent, queryClient: QueryClient) {
  switch (event.type) {
    case 'answer_ready':
      queryClient.invalidateQueries({ queryKey: ['question', event.payload.question_id] })
      toast.success('AI 解答已准备好')
      break
    case 'teacher_replied':
      queryClient.invalidateQueries({ queryKey: ['question', event.payload.question_id] })
      toast.info('老师已回复')
      break
    case 'report_ready':
      queryClient.invalidateQueries({ queryKey: ['parent', 'reports'] })
      break
  }
}
```

**修改**：在 `AppProviders.tsx` 中挂载 `<WebSocketProvider />`

**修改（移除轮询）**：
- `useTeacherHelpStatusQuery.ts`：移除 `refetchInterval`，依赖 WS 事件
- `useCheckoutCommandQuery.ts`：保留短暂轮询（计费状态更关键，WS 为辅）

---

#### S3-FE-02：Dashboard「今日推荐练习」卡片 ✅ 已完成（2026-08-10）
> 无需新后端接口：`GET /students/me/memory` 已返回 sequencing 引擎排好序的 `recommendations`。
> 新增 `RecommendedPracticeCard`，取 Top 3，跳转 `/question-bank/:subjectId/:topicId`。
> `useWeakTopicsQuery` 与 `useRecommendationsQuery` 共用同一 query key，只发一次请求。
>
> ⚠️ 同时修复了一个契约 bug：此前前端按 snake_case（`snapshots`/`weak_topics`）读取，
> 而 `_memory_response` 实际返回 camelCase（`weakTopics`/`memorySnapshots`/`recommendations`），
> 导致薄弱知识点卡片恒为空、AI prompt 的记忆注入恒为空。后端同一处也已修正。
> 测试：`tests/unit/memoryMapping.test.mjs`（9 例）、`tests/test_conversation_memory_context.py`（7 例）。

**Owner**：前端  
**工时**：6h  
**依赖**：S3-BE-01、S1-BE-02  

**新建文件**：`src/components/dashboard/DailyRecommendCard.tsx`

```typescript
// 展示今日推荐的 3-5 道练习题
// 每条：知识点名称 + 推荐理由 + 「开始练习」按钮
// 点击「开始练习」跳转 /practice/set/{practice_set_id}
// 底部：「查看全部推荐」链接
```

**修改**：`StudentDashboardPage.tsx` — 在 WeakConceptsCard 下方插入

---

## Sprint 4：叙述式报告 + 老师 AI 2.0（第 9–10 周）

### 目标
家长周报变"能读懂的信"，老师接管效率提升 3 倍。

---

### 后端任务

#### S4-BE-01：AI 叙述式周报生成
**Owner**：后端  
**工时**：8h  
**依赖**：S1-BE-02（memory）  

**修改文件**：`src/stoa/services/report_service.py`

在现有报告生成后，调用 Bedrock 生成叙述段落：

```python
NARRATIVE_PROMPT = """
Du bist ein pädagogischer Assistent von STOA. Schreibe einen kurzen, 
persönlichen Wochenbericht (150–200 Wörter) auf Deutsch für die Eltern von {student_name}.

Daten dieser Woche:
- Gestellte Fragen: {total_questions}
- Von KI gelöst: {ai_resolved} ({ai_rate}%)
- Lehrerbeteiligung: {teacher_sessions} Mal
- Lernzeit: ca. {study_minutes} Minuten

Schwache Bereiche: {weak_concepts}
Fortschritte: {improved_concepts}

Stil: positiv, ermutigend, konkret. Keine Übertreibungen.
Beginne mit dem Vornamen des Schülers.
Ende mit einem kurzen Motivationssatz.
Antworte NUR mit dem Fließtext, ohne Formatierung.
"""

async def generate_narrative_summary(
    student_name: str,
    weekly_stats: WeeklyStats,
    memory_summary: str,
) -> str:
    """生成纯文本的德语叙述摘要，约 150-200 字。"""
```

**修改周报数据模型**（`src/stoa/models/report.py`）：
```python
class WeeklyReport(BaseModel):
    # ... 现有字段
    narrative_summary_de: str | None = None   # 新增
    narrative_generated_at: str | None = None  # 新增
```

---

#### S4-BE-02：老师 AI 草稿 2.0
**Owner**：后端  
**工时**：8h  
**依赖**：S1-BE-02（memory）  

**修改文件**：`src/stoa/services/ai_teacher_tools_service.py`

**增强 `generate_draft_reply` 函数**，注入学生画像：

```python
async def generate_draft_reply_v2(
    question: QuestionRecord,
    teacher_context: str | None,
    student_id: str,
) -> TeacherDraftResult:
    """
    v2 改进：
    1. 获取学生在该题知识点上的历史错误模式
    2. 将错误模式注入 prompt（"该学生历史上总是在符号处出错"）
    3. 生成针对该学生的个性化回复草稿
    4. 同时生成知识点标注和相似题推荐
    """
    student_profile = await memory_service.get_student_profile_summary(
        student_id, question.subject
    )
    
    return TeacherDraftResult(
        draft_reply=...,
        knowledge_points=...,
        similar_exercises=await find_similar_exercises(question),
        error_pattern_hint=extract_error_pattern(student_profile, question),
    )
```

**新增端点**：`POST /teachers/questions/{id}/similar`

```json
// Response
{
  "similar_exercises": [
    {
      "description": "Eine ähnliche quadratische Gleichung mit positiver Diskriminante",
      "latex": "x^2 - 5x + 6 = 0",
      "difficulty": "medium",
      "source": "Lehrplan 21 Mathematik Kl. 9"
    }
  ]
}
```

---

### 前端任务

#### S4-FE-01：周报 AI 摘要段落
**Owner**：前端  
**工时**：5h  
**依赖**：S4-BE-01  

**修改文件**：`src/components/parent/WeeklyReport.tsx`（或对应组件）

在报告顶部增加摘要卡片：

```typescript
// 高亮显示 narrative_summary_de
// 卡片样式：warm paper 背景，左侧竖线装饰，"STOA 本周总结" 标题
// 下方折叠展开完整数据图表（原有内容）
```

---

#### S4-FE-02：老师工作台 AI 面板升级
**Owner**：前端  
**工时**：8h  
**依赖**：S4-BE-02  

**修改文件**：`src/components/tutor/TutorSession.tsx`（或对应组件）

```
┌─ 老师工作台 ─────────────────────────────────────┐
│  题目内容（已有）                                   │
│  ┌─ AI 草稿（已有，升级）────────────────────────┐│
│  │  ✨ 针对该学生个性化草稿（含错误模式提示）      ││
│  │  ────────────────────────────────────────  ││
│  │  学生历史提示：⚠️ 该学生常在符号处出错          ││
│  │  知识点：二次方程，判别式                       ││
│  └────────────────────────────────────────────┘│
│  ┌─ 相似题推荐（新增）────────────────────────────┐│
│  │  推荐布置练习：                                ││
│  │  1. x² - 5x + 6 = 0  [难度:中]               ││
│  │  2. 2x² + 3x - 2 = 0 [难度:中]               ││
│  │  [复制到回复]                                  ││
│  └────────────────────────────────────────────┘│
│  回复框（已有）                                    │
└──────────────────────────────────────────────────┘
```

---

## Sprint 5：Bedrock Knowledge Base + PWA（第 11–12 周）

### 目标
AI 回答与瑞士课程对齐，应用可离线访问。

---

### 知识库内容准备（非技术工作）

**负责**：产品/内容团队  
**工时**：20h（内容工作）  
**并行于技术开发**  

**需要准备的文档**：

| 文档 | 格式 | 内容说明 |
|------|------|---------|
| Lehrplan 21 数学知识点体系（Kl. 7-9） | Markdown | 知识点层级、学习目标 |
| Gymnasium 数学大纲 | Markdown | 知识点 + 考试要求 |
| 各年级典型错误模式 | Markdown | 每个知识点的 Top 3 常见错误 |
| Matura 考试重点（数学） | Markdown | 高频考点、题型分析 |

存储路径：`s3://stoa-{env}-knowledge-base/lehrplan21/`

---

### 后端任务

#### S5-BE-01：Bedrock Knowledge Base 部署
**Owner**：后端 + 基础设施  
**工时**：10h  

**新建文件**：`stoa-infra/stacks/stoa_knowledge_base_stack.py`

```python
from aws_cdk import (
    aws_bedrock as bedrock,
    aws_s3 as s3,
    aws_iam as iam,
)

class StoaKnowledgeBaseStack(Stack):
    def __init__(self, scope, id, *, storage_bucket, **kwargs):
        super().__init__(scope, id, **kwargs)
        
        # Titan Embeddings 模型（eu-central-2 支持）
        self.kb = bedrock.CfnKnowledgeBase(self, "StoaKnowledgeBase",
            name=f"stoa-lehrplan21-{env}",
            role_arn=kb_role.role_arn,
            knowledge_base_configuration=bedrock.CfnKnowledgeBase.KnowledgeBaseConfigurationProperty(
                type="VECTOR",
                vector_knowledge_base_configuration=bedrock.CfnKnowledgeBase.VectorKnowledgeBaseConfigurationProperty(
                    embedding_model_arn=f"arn:aws:bedrock:{region}::foundation-model/amazon.titan-embed-text-v1"
                )
            ),
            storage_configuration=...
        )
        
        # Data Source → S3 bucket
        self.data_source = bedrock.CfnDataSource(self, "KnowledgeBaseS3Source",
            knowledge_base_id=self.kb.attr_knowledge_base_id,
            name="lehrplan21-documents",
            data_source_configuration=bedrock.CfnDataSource.DataSourceConfigurationProperty(
                type="S3",
                s3_configuration=bedrock.CfnDataSource.S3DataSourceConfigurationProperty(
                    bucket_arn=storage_bucket.bucket_arn,
                    inclusion_prefixes=["knowledge-base/"]
                )
            )
        )
```

---

#### S5-BE-02：RAG 检索服务
**Owner**：后端  
**工时**：6h  
**依赖**：S5-BE-01  

**新建文件**：`src/stoa/services/rag_service.py`

```python
async def retrieve_curriculum_context(
    question_text: str,
    subject: str,
    grade: str,
    top_k: int = 3,
) -> str:
    """
    从 Bedrock Knowledge Base 检索与题目相关的课程内容。
    返回格式化文本，注入 system prompt。
    
    示例返回：
    "课程参考（Lehrplan 21，Kl. 9）：
    二次方程应掌握配方法和公式法两种解法。
    常见错误：忘记检验根是否满足原方程。"
    """
    response = bedrock_agent_runtime.retrieve(
        knowledgeBaseId=settings.kb_id,
        retrievalQuery={"text": f"{subject} {grade}: {question_text}"},
        retrievalConfiguration={
            "vectorSearchConfiguration": {"numberOfResults": top_k}
        }
    )
    return format_retrieval_results(response["retrievalResults"])
```

**修改**：`ai_service.py` — 在 Premium 套餐下，提问时追加 RAG 上下文到 system prompt。

**Feature Flag**：`settings.enable_rag = True/False`（按 subscription tier 或全局开关）

---

### 前端任务

#### S5-FE-01：PWA 配置 ⛔ 阻塞 — 需决策后再实施
> **不能按原方案直接接入。** 应用启动时会用 SHA-256 校验 `/runtime-config.json`
> （`src/lib/servedRelease.ts` → `expectedDigest: validated.runtimeConfig.sha256`）。
> `vite-plugin-pwa` 默认的 `generateSW` 会 precache/runtime-cache 该文件与 `index.html`，
> 发版后老用户的 SW 会返回旧副本，摘要比对失败 → 直接卡在启动失败页，属于线上不可用级故障。
>
> 若要实施，必须满足全部三条，且需你确认：
> 1. `runtime-config.json` 与 release manifest 走 **NetworkOnly**，绝不进缓存；
> 2. `index.html` 走 **NetworkFirst**，保证新版本可被拉取；
> 3. precache 仅限带内容哈希的 immutable 资源（`assets/*.js|css|woff2`）。
>
> 另需评估：新增 SW 会改变构建产物形态，`tests/release/verify-release.test.mjs`
> 的产物校验（source map、禁用字符串扫描）需同步确认是否放行。
> 开发环境须 `disable: true`，避免与 MSW 的 `/mockServiceWorker.js` 争夺 SW 注册。

**Owner**：前端  
**工时**：5h  

**安装**：`npm install vite-plugin-pwa`

**修改文件**：`vite.config.ts`

```typescript
import { VitePWA } from 'vite-plugin-pwa'

plugins: [
  VitePWA({
    registerType: 'autoUpdate',
    includeAssets: ['favicon.ico', 'robots.txt'],
    manifest: {
      name: 'STOA Lernassistent',
      short_name: 'STOA',
      description: 'KI-Lernassistent für Schweizer Schüler',
      theme_color: '#8B1A1A',
      background_color: '#FAF7F2',
      display: 'standalone',
      icons: [
        { src: 'pwa-192x192.png', sizes: '192x192', type: 'image/png' },
        { src: 'pwa-512x512.png', sizes: '512x512', type: 'image/png' },
      ]
    },
    workbox: {
      globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
      runtimeCaching: [
        {
          // 历史题目缓存优先（离线可查看）
          urlPattern: /^https:\/\/api\.stoaedu\.ch\/questions/,
          handler: 'CacheFirst',
          options: {
            cacheName: 'questions-cache',
            expiration: { maxEntries: 50, maxAgeSeconds: 7 * 24 * 60 * 60 }
          }
        },
        {
          // 练习题库网络优先（保持最新）
          urlPattern: /^https:\/\/api\.stoaedu\.ch\/practice/,
          handler: 'NetworkFirst',
          options: { cacheName: 'practice-cache' }
        }
      ]
    }
  })
]
```

**准备图标资源**：
- `public/pwa-192x192.png`（STOA logo 192×192）
- `public/pwa-512x512.png`（STOA logo 512×512）

**验收标准**：
- Chrome 显示「安装到主屏幕」提示
- 无网络下可打开历史解答记录
- iOS Safari 添加到主屏幕后图标正确

---

## Sprint 6：Bedrock Agents（Premium）（第 13–14 周）

### 目标
Premium 套餐用户享受多步推理 AI，提升 AI 答疑质量和差异化竞争力。

---

### 后端任务

#### S6-BE-01：Bedrock Agent 设计与部署
**Owner**：后端 + 基础设施  
**工时**：16h  
**依赖**：S5-BE-01（Knowledge Base）  

**Agent Action Groups 设计**：

```
Agent: STOA Premium Learning Assistant
  ├── Action Group: ConceptTools
  │     ├── identify_concepts(question_text) → list[Concept]
  │     └── get_grade_level_context(subject, grade) → str
  │
  ├── Action Group: MemoryTools  
  │     ├── get_student_weaknesses(student_id, subject) → list[Concept]
  │     └── get_student_learning_style(student_id) → str
  │
  ├── Action Group: KnowledgeTools
  │     └── search_curriculum(concept, grade) → str  [→ Knowledge Base]
  │
  └── Action Group: ResponseTools
        └── generate_personalized_steps(context) → str
```

**Lambda Action Executor**：

```python
# 新建文件：src/stoa/services/agent_service.py

async def invoke_premium_agent(
    question: str,
    student_id: str,
    subject: str,
    grade: str,
    session_id: str,
) -> AgentResponse:
    """
    调用 Bedrock Agent。
    仅对 subscription_tier = 'premium' 的学生可用。
    """
    response = bedrock_agent_runtime.invoke_agent(
        agentId=settings.bedrock_agent_id,
        agentAliasId=settings.bedrock_agent_alias_id,
        sessionId=session_id,
        inputText=question,
        sessionState={
            "sessionAttributes": {
                "student_id": student_id,
                "subject": subject,
                "grade": grade,
            }
        }
    )
    return stream_agent_response(response)
```

**修改**：`ai_service.py` — 按 tier 路由：

```python
async def generate_ai_response(..., subscription_tier: str = "free"):
    if subscription_tier == "premium" and settings.enable_premium_agent:
        return await agent_service.invoke_premium_agent(...)
    else:
        return await _invoke_haiku(...)   # 现有逻辑
```

---

#### S6-BE-02：Agent 性能监控与回退
**Owner**：后端  
**工时**：4h  
**依赖**：S6-BE-01  

- Agent 调用超时（> 15s）→ 自动降级为 Haiku 并记录日志
- CloudWatch 指标：`AgentInvocationCount`、`AgentLatencyP95`、`AgentFallbackCount`
- Feature Flag：`settings.enable_premium_agent = True/False`

---

### 前端任务

#### S6-FE-01：Premium 深度分析标记
**Owner**：前端  
**工时**：3h  

**修改**：解答页 — Premium 用户的 AI 解答头部显示：

```
┌─────────────────────────────────────────────┐
│  ✨ Premium 深度分析  ·  知识点：二次方程，判别式  │
│  AI 已分析你的学习历史并个性化本次解答           │
└─────────────────────────────────────────────┘
```

---

## 技术债与持续工作

### 贯穿所有 Sprint 的持续任务

| 任务 | 频率 | Owner |
|------|------|-------|
| E2E 测试补充（每新 API 对应一个 Playwright 测试） | 每 Sprint | 前端 |
| 后端单元测试补充（核心 service 层 70% 覆盖率） | 每 Sprint | 后端 |
| ADR 更新（记录重要架构决策） | 按需 | Tech Lead |
| `stoa-docs/HLD.md` API 清单同步更新 | 每 Sprint 末 | Tech Lead |
| `uv pip audit` + 前端 `npm audit` 安全扫描 | 每 Sprint | 后端 |
| CloudWatch 告警规则更新（新增指标） | 按需 | 后端 |

---

## 完整任务依赖图

```
Sprint 0
  S0-01 (MSW) ────────────────────────────────────────┐
  S0-02 (Cognito 账号) ──→ S0-03 (联调) ──────────────┤
                                                       ↓
Sprint 1                                         [联调完成]
  S1-BE-01 (memory repo) ──→ S1-BE-02 (memory svc)
  S1-BE-02 ──→ S1-BE-03 (prompt 注入)
  S1-BE-03 ──→ S1-BE-04 (追问 API)
  S1-BE-04 ──→ S1-FE-02 (追问 UI)
  S1-BE-02 ──→ S1-FE-01 (薄弱点卡片)

Sprint 2
  S2-BE-01 (Vision OCR) [独立]
  S2-BE-02 (LaTeX prompt) ──→ S2-FE-01 (KaTeX 渲染)
  S2-FE-01 + S2-FE-02 (Stream) ──→ S2-FE-03 (骨架屏)

Sprint 3
  S1-BE-02 ──→ S3-BE-01 (自适应算法)
  S3-BE-02 (WebSocket 基础) ──→ S3-FE-01 (WS Hook)
  S3-BE-01 ──→ S3-FE-02 (推荐卡片)

Sprint 4
  S1-BE-02 ──→ S4-BE-01 (叙述报告)
  S1-BE-02 ──→ S4-BE-02 (老师 AI 2.0)
  S4-BE-01 ──→ S4-FE-01 (周报 UI)
  S4-BE-02 ──→ S4-FE-02 (工作台 UI)

Sprint 5
  [内容准备] ──→ S5-BE-01 (KB 部署)
  S5-BE-01 ──→ S5-BE-02 (RAG 服务)
  S5-FE-01 (PWA) [独立]

Sprint 6
  S5-BE-01 (KB) ──→ S6-BE-01 (Agent)
  S6-BE-01 ──→ S6-BE-02 (监控回退)
  S6-BE-01 ──→ S6-FE-01 (Premium UI)
```

---

## 验收标准汇总（每 Sprint 交付门禁）

| Sprint | 交付门禁 |
|--------|---------|
| Sprint 0 | 无 mock 数据下，完整核心流程可用；E2E 全 26 个 spec 通过 |
| Sprint 1 | 有 2+ 历史提问后，AI 解答包含针对性提示；追问回答正确；memory DynamoDB 数据正确 |
| Sprint 2 | 10 道手写测试题 OCR 准确率 ≥ 90%；数学公式正确渲染；流式首字节 < 1s |
| Sprint 3 | Dashboard 显示个性化推荐；老师回复/AI 完成立即推送无需刷新 |
| Sprint 4 | 周报含 150+ 字德语摘要；老师草稿含学生错误模式提示 |
| Sprint 5 | AI 回答引用 Lehrplan 21 内容；PWA 可安装且离线可查历史 |
| Sprint 6 | Premium 用户 AI 响应质量评分 ≥ 4.5/5；延迟 > 15s 自动降级 |

---

*本文档应在每 Sprint 开始时更新当前进度，Sprint 结束时补充验收记录。*
