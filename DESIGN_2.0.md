# STOA 开发设计 2.0

> **版本：** v2.0  
> **创建日期：** 2026-08-04  
> **状态：** 部分实施中 — Sprint 任务完成 14/40（35%）  
> **最后核对：** 2026-08-10（逐条比对代码实际状态，非文档勾选）  
> **前置文档：** [PLAN.md](./PLAN.md) · [PRD.md](./PRD.md) · [ADR.md](./ADR.md) · [PROJECT_SLIM_PLAN.md](./PROJECT_SLIM_PLAN.md) · [SPRINT_PLAN_2.0.md](./SPRINT_PLAN_2.0.md)

---

## 〇、实施状态快照（2026-08-10）

> 本节由代码核查生成。全文各节的状态标注以本节为准。

### 总览

| 维度 | 数值 |
|------|------|
| Sprint 任务完成 | 14 / 40（35%） |
| 部分完成 | 5 |
| 未开始 | 21 |
| 被决策阻塞 | 5 项 |

### 🚨 首要问题：Sprint 0 未完成，Sprint 1–3 已在其上叠加

本文档 §2 写明「不在 demo 数据上开发 2.0 功能，所有新 AI 特性必须基于真实生产 API」。
实际情况是 Sprint 0 的**前后端全链路联调、真实测试账号、端到端跑通**三项均未执行，
而 Sprint 1–3 的功能已经实现。这些 AI 功能目前**仅有自动化测试覆盖，从未在真实
Cognito → API → UI 链路上验证**。后续任何功能开发前应先补齐此项。

### 功能模块（§3.3）

| 功能 | 状态 | 实际情况 |
|------|------|---------|
| A 学生记忆层 | 🟡 部分 | 记忆持久化为 `LEARNING_MEMORY#` 快照（**与本文档设计的 `STUDENT_MEMORY#/CONCEPT#` 模型不一致**，无 `mastery_level`）。prompt 注入 2026-08-10 修复后才真正生效。解答内高亮、家长热力图未做 |
| B 对话式多轮问答 | ✅ 已完成 | 走 `/conversations`（非本文档设计的 `/questions/{id}/chat`）。多轮链路 2.0 前已存在，本次补齐快捷追问 chips |
| C 自适应练习推荐 | 🟡 部分 | 排序算法与 recommendations 2.0 前已存在，本次接入 Dashboard 卡片。`POST /adaptive/feedback` 未实现 |
| D 老师 AI 工具 2.0 | ⬜ 未开始 | `ai-draft` 端点存在但为字符串模板，**未调 Bedrock、未传学生画像**。similar / error-pattern 端点及前端面板均无 |
| E 家长 AI 叙述报告 | ✅ 已完成 | **2.0 前已实现**：`report_service.py:270-296` 调 Bedrock 生成叙述，失败回退统计模板 |
| F 流式响应优化 | ✅ 已完成 | 打字机光标、Thinking 占位、KaTeX、骨架屏 + lazy 路由 |
| G Claude Vision 多模态 | ⛔ 阻塞 | 仍走 Rekognition `detect_text`，Bedrock 只收 OCR 后文本。需确认模型 ID |

### 新技术引入（§3.4）

| 技术 | 状态 | 实际情况 |
|------|------|---------|
| 1 Bedrock Agents | ⬜ 未开始 | 无 `agent_service.py`，无 `InvokeAgent`，无 tier 路由 |
| 2 Bedrock Knowledge Base | ⛔ 阻塞 | 无 `rag_service.py`，无 Titan Embeddings。需确认 DynamoDB 表名对齐 |
| 3 WebSocket 替代轮询 | 🟡 部分 | 服务层与前端通知中心已连通，但 `websocket_live_deployed=False` **未部署**；设计的四个事件名多数未 emit；前端仍剩 3 处 `refetchInterval` |
| 4 LaTeX / KaTeX | ✅ 已完成 | prompt 指令 + `MathRenderer`，并修复降级分支 XSS |
| 5 PWA + 离线 | ⛔ 阻塞 | 缓存 `runtime-config.json` 会导致启动 SHA-256 校验失败，属线上不可用级风险 |

### 等待决策的事项

| 决策项 | 解锁 | 说明 |
|--------|------|------|
| Bedrock 模型 ID | 功能 G、Sprint 6 | 涉及模型名变更 |
| DynamoDB 表名对齐 | 引入 2 | KB 需与现有单表设计对齐 |
| PWA 缓存策略 | 引入 5 | `runtime-config` 必须 NetworkOnly |
| demo backend 删除 | Sprint 0 收尾 | `stoa-frontend/backend/` 及相关脚本 |
| 记忆查询缓存 | AI 热路径性能 | 每条消息多付 3 次 DynamoDB 读（约 50–100ms） |

---

## 一、2.0 定位与背景

### 当前状态（v1.x 盘点）

截至 2026-08 月，STOA 已完成：

- ✅ AWS 基础设施全量上线（eu-central-2）
- ✅ 完整认证体系（Cognito + JWT + 四角色）
- ✅ Stripe 订阅计费 + 幂等对账
- ✅ 四语言 UI（DE/EN/FR/IT）
- ✅ 老师工作台 + SLA 追踪
- ✅ 家长周报 + 账号运维
- ✅ CI/CD 三仓库自动部署

但同时积累了：

- ❌ 前后端未真正联调（仍大量 demo 数据）
- ❌ 双首页、双 Mistakes 页、废弃 Store、遗留页面
- ❌ 3050+ 规划文件污染工作区
- ❌ AI 层仍是基础 Bedrock Claude Haiku，无上下文记忆、无个性化

**2.0 目标**：在完成代码瘦身的前提下，以 **AI 能力深化** 为核心驱动，将 STOA 从"AI 答题工具"升级为"有记忆、能成长、懂学生"的**智能学习伙伴**。

---

## 二、瘦身先行：2.0 启动前置条件

> 详细执行步骤见 [PROJECT_SLIM_PLAN.md](./PROJECT_SLIM_PLAN.md)

### ✅ 已完成（P0）

| 操作 | 完成时间 |
|------|---------|
| 删除 `src/stores/`（死代码） | 2026-08-04 |
| 删除 `src/store/uiStore.ts`（死代码） | 2026-08-04 |
| `.planning/` 加入前后端 `.gitignore` | 2026-08-04 |
| `evidence/` 加入后端 `.gitignore` | 2026-08-04 |

### 📋 2.0 开发前必须完成（P1）

| 操作 | 负责人 | 预计工时 | 状态（2026-08-10） |
|------|--------|---------|-------------------|
| 删除 8 个未挂路由的遗留页面 | 前端 | 2h | ✅ 已完成 |
| 产品确认保留哪个首页版本并合并 | 产品 + 前端 | 4h | ✅ 已完成（旧 home 页面树整体删除） |
| 移除 `stoa-frontend/backend/` demo backend | 前端 | 3h | ⛔ 待你确认后删除 |
| 用 MSW 替代 demo backend 完成前端离线开发 | 前端 | 8h | 🟡 改道：MSW 无法进浏览器包（见下），handlers 现由 `tests/mswServer.ts` 在测试侧消费 |
| 完成前后端联调（Cognito → API → UI 全链路） | 全栈 | 16h | ⬜ **未开始 — 阻塞后续所有 Sprint** |

> ⚠️ **MSW 方案变更（2026-08-10）**：原计划用 `VITE_ENABLE_MSW` 在 `main.tsx` / `src/lib/env.ts`
> 中开关，这违反了「`runtime-config.json` 是运行时唯一真相源」的架构约束，直接打破
> `tests/release/runtime-startup-barrier.test.mjs` 与 `runtime-env-projection.test.mjs`
> 两个发布门禁。mock 代码已从生产包移除，仅供测试使用。前端离线开发能力因此**未达成**，
> 需依赖 Sprint 0 的真实环境联调。

> 🚨 **核心原则**：不在 demo 数据上开发 2.0 功能。2.0 所有新 AI 特性必须基于真实生产 API。
> **当前未遵守** —— 详见 §〇 首要问题。

---

## 三、AI 能力深化路线图（核心）

### 3.1 现有 AI 架构的局限

```
当前（v1.x）
学生提问 → Bedrock Claude Haiku → 单次无状态回答 → 老师接管（可选）
```

**问题**：
- 每次提问独立，AI 不记得学生之前犯过什么错
- 所有学生收到相同风格的解答，无个性化
- AI 无法主动感知学生的学习薄弱点并调整策略
- 老师接管和 AI 解答是割裂的两条线，没有协同

### 3.2 2.0 AI 架构目标

```
目标（v2.0）
学生画像（记忆层）
    ↓
提问上下文 + 历史错误 + 薄弱点
    ↓
AI 编排层（多步推理 + 策略选择）
    ↓
个性化分步解答 + 自适应难度
    ↓
老师辅助工具（AI 草稿 + 知识图谱提示）
    ↓
家长洞察报告（AI 叙述式摘要）
```

---

### 3.3 功能模块详细设计

#### 功能 A：学生记忆层（Student Memory） 🟡 部分完成

> **实现与本节设计不一致。** 实际持久化的是 `LEARNING_MEMORY#<student_id>` /
> `SUBJECT#<subject>` 快照（`adaptive_learning_repo.py:241-277`），而非下方设计的
> `STUDENT_MEMORY#/CONCEPT#`，也没有 `mastery_level` / `error_count` / `error_types` 字段。
> 若要按本节改造属于模型变更，需先决策。
>
> - ✅ 知识点抽取与画像累积（2.0 前已存在）
> - ✅ prompt 注入画像摘要 —— 但**直到 2026-08-10 才真正生效**：此前调用
>   `actor.user`（属性不存在）抛 `AttributeError`，被宽 `except` 吞掉，个性化恒为空
> - ✅ 前端「薄弱知识点」卡片
> - ⬜ 解答中高亮与历史错误相关的步骤
> - ⬜ 家长侧知识点掌握度热力图
> - ⚠️ 性能：记忆查询为 3 次 DynamoDB 读，位于每条消息的热路径（已移出 AI 租约临界区）

**问题**：当前 AI 每次都从零开始，不知道学生的历史。  
**目标**：AI 能记住学生在哪些知识点反复犯错，并在解答中主动适配。

**实现方案**：

```
技术选型：Amazon Bedrock Knowledge Base + DynamoDB
存储结构：
  - 每次提问/回答 → 提取"知识点标签 + 错误类型"
  - 存入 student_memory 表（DynamoDB）
  - 累积后形成"学生知识画像"
  - 每次提问时附加画像摘要到 system prompt
```

**DynamoDB 新增实体**：

```
PK: STUDENT_MEMORY#<student_id>
SK: CONCEPT#<concept_id>
Attributes:
  - concept_name: str          # "二次方程"
  - error_count: int           # 累计错误次数
  - last_error_at: datetime
  - error_types: list[str]     # ["计算错误", "符号混淆"]
  - mastery_level: float       # 0.0–1.0（根据历史计算）
  - updated_at: datetime
```

**Prompt 增强示例**：

```python
system_prompt = f"""
你是 STOA 的学习助手，正在帮助一位{student.grade}年级学生解答数学题。

【该学生的学习画像】
- 在"二次方程"知识点上已错误 5 次，主要错误类型：符号混淆
- 在"函数图像"上掌握良好（近期无错误）
- 学习风格：需要更多步骤分解，不要跳步

【解答原则】
1. 分步引导，不直接给答案
2. 针对该学生的薄弱点，在相关步骤额外提醒
3. 使用德语解答
"""
```

**前端变化**：
- 学生侧：展示"你的薄弱知识点"卡片（来自 memory）
- 解答中：高亮与历史错误相关的步骤
- 家长侧：知识点掌握度热力图

---

#### 功能 B：对话式多轮问答（Conversational AI） ✅ 已完成

> 落地端点为 `/conversations`（含幂等命令、AI 租约、并发收敛），非本节设计的
> `/questions/{id}/chat`。多轮链路 2.0 前已存在，本次新增前端快捷追问 chips
> （`FollowUpSuggestions.tsx`，仅在最后一条已完成的助手消息下渲染）。

**问题**：当前是单次 Q&A，学生无法追问。  
**目标**：学生可以在同一道题上持续追问，AI 保持上下文。

**实现方案**：

```
现有 conversations 路由已有流式支持（stream endpoint）
扩展：
  - 每道题（question）关联一个 conversation_thread
  - thread 存储多轮消息历史（DynamoDB list item）
  - 前端 Chat UI 允许追问（已有 /chat 页面，需联调）
  - token 窗口控制：保留最近 10 轮 + 学生画像摘要
```

**新增 API 端点**：

```
POST /questions/{id}/chat        # 在题目内追问
GET  /questions/{id}/thread      # 获取该题对话历史
```

**前端变化**：
- 解答页底部增加"继续追问"输入框
- 对话历史折叠展示（避免过长）

---

#### 功能 C：自适应练习推荐（Adaptive Practice） 🟡 部分完成

> 排序算法与 `recommendations` 输出 2.0 前已存在；本次接入 Dashboard
> `RecommendedPracticeCard`。缺 `POST /adaptive/feedback` 反馈回路，
> 因此推荐质量目前无法基于「做对/做错」自我修正。

**问题**：当前 Practice 路径是固定课程顺序，不感知学生状态。  
**目标**：根据学生的历史表现，动态推荐最需要练习的知识点。

**实现方案**：

```
算法：基于"遗忘曲线 + 错误频率"的简单加权推荐
  score = error_rate × recency_weight × difficulty_weight

实现栈：
  - 后端：adaptive router（已有 /adaptive 路由框架）
  - 计算：Lambda 函数（可复用现有 jobs 框架）
  - 存储：DynamoDB adaptive_assignment 实体（已有）
```

**新增 API 端点**：

```
GET  /adaptive/recommendations     # 今日推荐练习（已有，需完善逻辑）
POST /adaptive/feedback            # 练习结果反馈（更新记忆层）
```

**前端变化**：
- Dashboard 顶部「今日推荐」卡片（个性化，非静态）
- 推荐说明：「根据你上周在二次方程的错误，推荐练习这 3 题」

---

#### 功能 D：AI 辅助老师工作台（Teacher AI Tools 2.0） ⬜ 未开始

> `ai-draft` 端点已存在但只是字符串模板拼接，**未调用 Bedrock、未传入学生画像**。
> 相似题推荐、错误模式分析的后端端点与前端面板均无。

**问题**：现有老师 AI 工具（draft）功能单一。  
**目标**：让老师的接管效率提升 3 倍。

**新增 AI 能力**：

| 工具 | 描述 | 技术实现 |
|------|------|---------|
| **智能草稿 2.0** | 基于题目 + 学生画像生成针对性回复草稿 | Bedrock + student memory |
| **知识点标注** | 自动识别题目涉及的知识点并标注 | Claude structured output |
| **相似题推荐** | 推荐 2–3 道相似练习题作为跟进 | Bedrock Embedding + Knowledge Base |
| **错误归因分析** | 分析该学生在此类题上的错误模式 | 学生画像 + 统计聚合 |

**API 变化**：

```
POST /teachers/questions/{id}/ai-draft     # 增强：传入学生画像
POST /teachers/questions/{id}/similar      # 新增：相似题推荐
GET  /teachers/questions/{id}/error-pattern # 新增：错误模式分析
```

---

#### 功能 E：家长 AI 叙述式报告（Narrative Report） ✅ 已完成

> **本节前提有误。** 核查发现该能力在 2.0 立项前就已实现，周报并非"数据堆砌"：
> `report_service.py:55-69` 定义了要求生成 summary/strengths 的 system prompt，
> `270-296` 调用 `bedrock-runtime.invoke_model` 生成叙述，仅在异常时才回退到
> `302-350` 的确定性统计模板。定时任务链路见 `jobs/weekly_reports.py:102-103`。
> 前端 `ChildReportPage.tsx:84-86` 直接渲染 `report.summary`。
>
> 仍可优化（非本节原目标）：前端无专用 AI 摘要组件，`ParentReportSummaryCard.tsx`
> 已存在但**未被任何页面引用**，属死代码。

**问题**：当前周报是数据堆砌，家长难以快速理解。  
**目标**：用自然语言叙述学生本周学习状况，像"给家长的一封信"。

**实现方案**：

```python
# report_service.py 增强
def generate_narrative_report(student_id, week):
    stats = aggregate_weekly_stats(student_id, week)
    memory = get_student_memory(student_id)
    
    prompt = f"""
    请用温和、专业的德语，写一段 150-200 字的学习周报，
    发给{student.name}的家长。内容包括：
    
    数据：本周提问{stats.total}次，AI 解决{stats.ai_resolved}次，
    老师接管{stats.teacher_sessions}次。
    
    薄弱点：{memory.top_weak_concepts}
    进步点：{memory.recent_improvements}
    
    语气：积极正面，指出具体改进点，结尾一句鼓励。
    """
    
    return bedrock.invoke(prompt)
```

**前端变化**：
- 周报顶部增加"AI 摘要段落"（叙述式，高亮展示）
- 数据图表保留在下方作为支撑

---

#### 功能 F：流式响应优化（Streaming UX） ✅ 已完成

> `useStreamingChat` 2.0 前已存在；本次补齐打字机光标（`StreamingCursor`）、
> 首 token 前的 "Thinking…" 占位、KaTeX 渲染，以及骨架屏与 `React.Suspense`
> lazy 路由的衔接。

**问题**：当前流式 API 已有，但前端 UX 还未充分利用（等待感强）。  
**目标**：AI 解答做到"打字机效果"，首字节延迟感知 < 1 秒。

**技术改进**：

```typescript
// 前端：使用 ReadableStream 实时渲染
const streamAnswer = async (questionId: string) => {
  const response = await fetch(`/api/conversations/${threadId}/stream`, {
    headers: { Authorization: `Bearer ${token}` }
  });
  
  const reader = response.body?.getReader();
  const decoder = new TextDecoder();
  
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    
    // 实时追加到 UI，支持 Markdown 渐进渲染
    appendStreamChunk(decoder.decode(value));
  }
};
```

**新增**：
- 流式渲染时支持 LaTeX 数学公式（`KaTeX` 延迟渲染）
- 流式过程中显示"AI 正在思考..."动画（骨架屏）
- 错误重试机制（网络中断自动续传）

---

#### 功能 G：多模态输入增强（Multimodal Input） ⛔ 阻塞

> 现状仍是 Rekognition `detect_text` → 纯文本 → Bedrock，图片本身从未进入模型。
> 改用 Claude Vision 涉及模型 ID 变更，需你先决策。

**问题**：当前 OCR 用 Rekognition，对手写体识别率不足 70%（PRD 中已标注风险）。  
**目标**：提升图片题目识别准确率至 90%+。

**技术升级方案**：

```
方案 1（推荐）：Claude Vision 直接处理图片
  - 将图片 base64 编码直接传给 Claude（支持 vision）
  - 让 Claude 同时完成 OCR + 理解题意
  - 无需单独 Rekognition 步骤
  - 成本：约 $0.002/张（Claude Haiku vision）
  - 优势：手写体识别率 95%+，数学符号识别更准

方案 2（备选）：Amazon Textract（替代 Rekognition）
  - 专为文档 OCR 优化
  - 支持表格、公式结构
  - 成本略高于 Rekognition
```

**实现变化**（方案 1）：

```python
# 当前：Rekognition OCR → 文本 → Bedrock
# 改为：图片直接发送 Claude Vision

async def process_question_image(image_s3_key: str) -> QuestionContext:
    image_bytes = await s3.get_object(image_s3_key)
    
    response = bedrock.converse(
        modelId="anthropic.claude-3-haiku-20240307-v1:0",
        messages=[{
            "role": "user",
            "content": [
                {
                    "image": {
                        "format": "jpeg",
                        "source": {"bytes": image_bytes}
                    }
                },
                {"text": "请提取并理解这道数学题，返回：题目文本、涉及知识点、难度估计"}
            ]
        }]
    )
    return parse_question_context(response)
```

---

### 3.4 新技术引入建议

#### 引入 1：Bedrock Agents（自主 AI 工作流） ⬜ 未开始

> 无 `agent_service.py`，代码库中无 `InvokeAgent` 调用，无按订阅 tier 的模型路由。

**场景**：当学生的问题涉及多个知识点时，当前 AI 只能给一个线性解答。  
**目标**：构建 AI Agent，能够"分解问题 → 检索相关知识 → 多步推理 → 整合答案"。

```
Bedrock Agent 工作流：
学生问题
  → Action: identify_concepts（识别知识点）
  → Action: retrieve_knowledge（从 Knowledge Base 检索相关知识）
  → Action: check_student_memory（检查该学生的薄弱点）
  → Action: generate_steps（生成个性化步骤）
  → 最终回答
```

**适用场景**：Premium 套餐用户（"优先 AI"服务），Standard 用户仍用简单 Haiku。

---

#### 引入 2：Bedrock Knowledge Base（RAG 知识库） ⛔ 阻塞

> 无 `rag_service.py`，无 Titan Embeddings 调用，无 KB Stack。
> 部署前需确认与现有 DynamoDB 单表设计的表名对齐方案。

**场景**：AI 回答基于通用知识，缺乏瑞士课程体系针对性（Lehrplan 21）。  
**目标**：构建 STOA 专属课程知识库，让 AI 回答契合瑞士学校课程标准。

```
知识库内容：
  - 瑞士 Lehrplan 21 数学知识点体系
  - 各年级典型题目与标准解法
  - 常见错误模式与纠正策略
  - 考试重点（Matura 备考）

技术实现：
  - S3 存储课程 PDF/文档
  - Bedrock Knowledge Base 自动向量化（使用 Amazon Titan Embeddings）
  - 每次提问时 RAG 检索相关课程内容作为上下文
```

**预期效果**：AI 解答更贴合瑞士课程，减少"解法正确但不符合学校教法"的投诉。

---

#### 引入 3：WebSocket 实时通知（替代轮询） 🟡 部分完成

> 服务层与前端通知中心已连通，但配置项 `websocket_live_deployed=False`，
> **API Gateway WebSocket API 实际未部署**。本节设计的四个事件名多数没有对应的
> emit 点。前端仍保留 3 处 `refetchInterval`；本次仅优化了教师协助相关的两处
> 轮询节奏（活跃 5s / 等待 15s / 其余停止，且后台不轮询）。

**场景**：当前学生等待 AI 回答时用轮询（`refetchInterval`），体验差。  
**目标**：实时推送 AI 回答完成、老师接管等状态变化。

```
技术选型：API Gateway WebSocket API（已有 SQS FIFO 基础设施可复用）

事件类型：
  - answer_ready: AI 解答完成
  - teacher_took_over: 老师已接管
  - teacher_replied: 老师回复
  - report_ready: 周报生成完成
```

**后端实现**：

```python
# 现有 EventBridge + SQS → 扩展为同时推 WebSocket
async def on_question_answered(question_id: str, student_id: str):
    await notify_service.push_websocket(
        connection_id=get_student_connection(student_id),
        event={"type": "answer_ready", "question_id": question_id}
    )
```

**前端变化**：
- 移除所有 `refetchInterval` 轮询
- 接入 WebSocket，通过事件驱动更新 UI

---

#### 引入 4：LaTeX / 数学公式渲染 ✅ 已完成

> prompt 侧输出指令 + 前端 `MathRenderer`（KaTeX）。
> ⚠️ 实现过程中发现并修复一个 XSS：LaTeX 解析失败的降级分支直接把原始表达式塞进
> `dangerouslySetInnerHTML`，而该内容来自 AI 输出，可被注入 HTML。现已 HTML 转义。

**场景**：当前 AI 输出数学公式为纯文本（如 `x^2 + 2x + 1 = 0`），渲染体验差。  
**目标**：AI 输出 LaTeX，前端使用 KaTeX 实时渲染。

```typescript
// 安装
npm install katex react-katex

// 使用
import { InlineMath, BlockMath } from 'react-katex';
import 'katex/dist/katex.min.css';

// AI 回答中自动检测并渲染
const renderWithMath = (text: string) => {
  // 检测 $...$ 和 $$...$$ 并替换为 KaTeX 渲染
  return text.replace(/\$\$(.*?)\$\$/g, (_, math) => 
    renderToString(<BlockMath math={math} />)
  );
};
```

**后端变化**：
- 修改 AI prompt，要求数学公式输出为 LaTeX 格式（`$...$`）
- 不影响已有 API 接口

---

#### 引入 5：PWA + 离线能力 ⛔ 阻塞

> **与现有启动校验直接冲突。** 应用启动时会拉取 `runtime-config.json` 并校验
> SHA-256；Service Worker 一旦缓存该文件，配置轮换后哈希不匹配，应用将无法启动
> ——属线上不可用级风险。需先决策缓存策略（至少 `runtime-config.json` 必须
> NetworkOnly 且不参与预缓存清单）。

**场景**：学生在地铁/回家路上想查看历史解答，无网络。  
**目标**：核心内容可离线访问，支持「添加到主屏幕」。

```typescript
// vite.config.ts 引入 vite-plugin-pwa
import { VitePWA } from 'vite-plugin-pwa';

plugins: [
  VitePWA({
    registerType: 'autoUpdate',
    workbox: {
      globPatterns: ['**/*.{js,css,html,ico,png,svg}'],
      runtimeCaching: [{
        urlPattern: /^https:\/\/api\.stoaedu\.ch\/questions/,
        handler: 'CacheFirst',         // 历史问答缓存优先
        options: { cacheName: 'questions-cache', expiration: { maxEntries: 50 } }
      }]
    },
    manifest: {
      name: 'STOA Lernassistent',
      short_name: 'STOA',
      theme_color: '#8B1A1A',          // 品牌 burgundy
      icons: [/* ... */]
    }
  })
]
```

---

## 四、2.0 Sprint 计划

> **前提**：瘦身 P1 工作（联调、demo 移除、首页合并）在 Sprint 0 完成。  
> **Sprint 周期**：2 周

### Sprint 0（立即开始，2 周）：瘦身 + 联调

**目标**：完成代码清理，打通真实前后端链路

| 任务 | 工时 | 状态 |
|------|------|------|
| 删除 8 个遗留页面 | 2h | ✅ |
| 合并双首页（产品确认后） | 4h | ✅ |
| 移除 demo backend，引入 MSW | 8h | 🟡 MSW 仅测试侧；demo backend 待确认删除 |
| 前后端全链路联调（Cognito → API → UI） | 16h | ⬜ |
| 创建真实测试账号（学生/家长/老师各 1 个） | 2h | ⬜ |
| 端到端跑通：注册 → 提问 → AI 回答 → 老师接管 | 8h | ⬜ |

**交付物**：真实环境下完整核心流程可用

> 🚨 **本 Sprint 未交付，但 Sprint 1–3 已在其上开发。** 所有 2.0 AI 功能目前只有
> 自动化测试覆盖，未经真实链路验证。恢复开发前应优先补齐后三项。

---

### Sprint 1（第 3–4 周）：学生记忆层 + 对话式追问

**目标**：AI 能记住学生，学生可以追问

| 任务 | 工时 | 状态 |
|------|------|------|
| DynamoDB 新增 `student_memory` 实体 | 4h | 🟡 实为 `LEARNING_MEMORY#` 快照，模型与设计不同 |
| 后端：每次问答后自动提取知识点标签并存入 memory | 8h | ✅ 2.0 前已存在 |
| 后端：提问时附加学生画像到 system prompt | 4h | ✅ 2026-08-10 修复 `actor.user` 后才真正生效 |
| 后端：`/questions/{id}/chat` 多轮追问 API | 6h | ✅ 实为 `/conversations`，2.0 前已存在 |
| 前端：解答页追问输入框 + 对话历史展示 | 8h | ✅ 本次补齐快捷追问 chips |
| 前端：Dashboard「我的薄弱点」卡片（来自 memory） | 4h | ✅ |

---

### Sprint 2（第 5–6 周）：多模态升级 + 流式 UX + LaTeX

**目标**：图片识别更准，AI 回答体验更流畅

| 任务 | 工时 | 状态 |
|------|------|------|
| 后端：Claude Vision 替换 Rekognition OCR | 8h | ⛔ 待确认模型 ID |
| 后端：AI prompt 增加 LaTeX 输出指令 | 2h | ✅ |
| 前端：引入 KaTeX，解答页 LaTeX 渲染 | 4h | ✅ 含降级分支 XSS 修复 |
| 前端：流式答案打字机效果（ReadableStream） | 6h | ✅ 光标 + Thinking 占位 |
| 前端：骨架屏 + 流式过程中的 UX 优化 | 4h | ✅ 接入 Suspense lazy 路由 |
| 测试：10 道真实手写数学题 OCR 对比测试 | 4h | ⬜ 依赖上面的 Vision 决策 |

---

### Sprint 3（第 7–8 周）：自适应练习 + WebSocket

**目标**：AI 主动推荐练习，实时替代轮询

| 任务 | 工时 | 状态 |
|------|------|------|
| 后端：完善 `/adaptive/recommendations` 算法（基于 memory） | 8h | 🟡 算法 2.0 前已存在；`POST /adaptive/feedback` 未实现 |
| 后端：API Gateway WebSocket API 部署 | 8h | ⬜ `websocket_live_deployed=False` |
| 后端：answer_ready / teacher_replied 事件 WebSocket 推送 | 6h | 🟡 服务层可推送，设计的事件名多数未 emit |
| 前端：移除所有 `refetchInterval` 轮询，改为 WebSocket | 8h | 🟡 已优化 2 处间隔，仍剩 3 处轮询；未部署无法真正移除 |
| 前端：Dashboard「今日推荐练习」个性化卡片 | 6h | ✅ |

---

### Sprint 4（第 9–10 周）：AI 叙述式报告 + 老师 AI Tools 2.0

**目标**：提升家长报告质量，提升老师效率

| 任务 | 工时 | 状态 |
|------|------|------|
| 后端：周报生成增加 AI 叙述段落（德语） | 6h | ✅ 2.0 前已实现 |
| 前端：周报顶部 AI 摘要组件 | 4h | 🟡 `ParentReportSummaryCard.tsx` 存在但无页面引用（死代码） |
| 后端：老师智能草稿 2.0（结合学生画像） | 6h | ⬜ 现为字符串模板，未调 Bedrock |
| 后端：相似题推荐 API（Bedrock Embedding） | 8h | ⬜ |
| 前端：老师工作台新增「相似题推荐」面板 | 6h | ⬜ |
| 前端：老师工作台新增「错误模式分析」提示 | 4h | ⬜ |

---

### Sprint 5（第 11–12 周）：Bedrock Knowledge Base + PWA

**目标**：AI 与瑞士课程绑定，支持离线访问

| 任务 | 工时 | 状态 |
|------|------|------|
| 准备 Lehrplan 21 数学知识点文档（PDF/Markdown） | 16h（内容） | ⬜ |
| 后端：部署 Bedrock Knowledge Base（S3 + Titan Embeddings） | 8h | ⛔ 待确认表名对齐 |
| 后端：提问时 RAG 检索相关课程内容 | 6h | ⬜ |
| 前端：引入 `vite-plugin-pwa`，配置 Service Worker | 4h | ⛔ 与 runtime-config 校验冲突 |
| 前端：历史问答离线缓存 | 4h | ⛔ 同上 |
| 测试：PWA 安装到手机主屏幕验证 | 2h | ⛔ 同上 |

---

### Sprint 6（第 13–14 周）：Bedrock Agents（Premium AI）

**目标**：Premium 用户享受更强大的 AI 推理

| 任务 | 工时 | 状态 |
|------|------|------|
| 设计 Agent Action Group（identify_concepts / retrieve_knowledge / generate_steps） | 8h | ⬜ |
| 后端：部署 Bedrock Agent，配置 Lambda action executor | 12h | ⬜ |
| 后端：按订阅 tier 路由（Premium → Agent，Standard/Free → Haiku） | 4h | ⬜ 涉及模型选择，待决策 |
| 前端：Premium 用户解答页增加「深度分析」标记 | 2h | ⬜ |
| 测试：复杂数学题 Agent vs Haiku 质量对比 | 4h | ⬜ |

---

## 五、技术栈变化对比

| 层次 | v1.x（现在） | v2.0（目标） |
|------|-------------|-------------|
| AI 模型 | Claude Haiku（单次调用） | Haiku（Standard/Free）+ Bedrock Agents（Premium） |
| 图片处理 | Rekognition OCR | Claude Vision（多模态直接处理） |
| 知识库 | 无 | Bedrock Knowledge Base（Lehrplan 21） |
| 上下文记忆 | 无 | DynamoDB student_memory + prompt 注入 |
| 实时通信 | HTTP 轮询（refetchInterval） | API Gateway WebSocket |
| 数学渲染 | 纯文本 | KaTeX（LaTeX 渲染） |
| 离线支持 | 无 | PWA + Service Worker |
| AI 工作流 | 单步 invoke | Bedrock Agents（多步推理） |
| 报告生成 | 数据统计 | 数据统计 + Bedrock 叙述生成 |
| 测试 mock | SQLite demo backend | MSW（Mock Service Worker） |

---

## 六、基础设施变化（stoa-infra）

### 新增 CDK Stack / 资源

| 资源 | Stack | 用途 |
|------|-------|------|
| API Gateway WebSocket API | `StoaApiStack` 扩展 | 实时通知 |
| Bedrock Knowledge Base | `StoaAiStack` 扩展 | 课程 RAG |
| Bedrock Agent | `StoaAiStack` 扩展 | Premium AI 工作流 |
| S3 Bucket: `stoa-knowledge-base` | `StoaStorageStack` 扩展 | 知识库文档存储 |
| Lambda: WebSocket connect/disconnect | `StoaApiStack` 扩展 | WebSocket 连接管理 |
| DynamoDB GSI: `student_memory-index` | `StoaDatabaseStack` 扩展 | 学生记忆查询 |

### ADR 新增决策

**ADR-010: Claude Vision 替代 Rekognition OCR**
- 决策：用 Claude Haiku vision 能力直接处理图片，同时完成 OCR + 题目理解
- 原因：手写体识别率更高（95% vs 70%）；消除独立 OCR 步骤；成本相近
- 影响：移除 `ocr_service.py` 中的 Rekognition 调用；图片尺寸压缩策略调整

**ADR-011: API Gateway WebSocket for 实时通知**
- 决策：新建 WebSocket API 替代前端 HTTP 轮询
- 原因：降低 API 调用量（成本 −60%）；改善用户体验；EventBridge + SQS 已有基础
- 影响：新增 connection management Lambda；前端需要 WebSocket 连接管理

**ADR-012: Bedrock Knowledge Base for 课程 RAG**
- 决策：将瑞士 Lehrplan 21 课程文档向量化入 Knowledge Base
- 原因：提升 AI 回答的课程针对性；减少家长"答法不对"的投诉
- 影响：需准备课程内容（非技术工作量）；每次提问增加约 100ms RAG 检索

---

## 七、成功指标（2.0 版本）

| 指标 | v1.x 目标 | v2.0 目标 | 测量方式 |
|------|----------|----------|---------|
| AI 答疑满意度 | ≥ 3.5/5 | **≥ 4.2/5** | 每次回答评星 |
| 老师接管率 | ≤ 30% | **≤ 18%**（AI 更强） | 接管数/总提问数 |
| 学生 7 日留存 | ≥ 40% | **≥ 60%**（自适应推荐） | DAU 追踪 |
| AI 首字节延迟（流式） | < 3s | **< 1s** | P90 latency |
| 图片识别准确率 | 70% | **≥ 90%** | 手动抽查 |
| 家长报告打开率 | 未测量 | **≥ 60%** | SES 打开跟踪 |
| Premium 用户占比 | 未测量 | **≥ 20%** | 订阅数据 |

---

## 八、风险与应对（2.0 新增）

| 风险 | 概率 | 影响 | 应对 |
|------|------|------|------|
| Bedrock Agents 延迟高（>10s） | 中 | 中 | 仅 Premium 用，Standard 用简单 Haiku；Agent 异步化处理 |
| 学生记忆积累数据量大，影响 DynamoDB 查询 | 低 | 中 | 设置 memory TTL（180天）；每次仅检索 Top 10 薄弱点 |
| Lehrplan 21 课程内容版权问题 | 低 | 高 | 使用官方公开文档；联系 EDK（瑞士教育局）授权 |
| WebSocket 连接管理复杂（Lambda 无状态） | 中 | 中 | 使用 DynamoDB 存储 connection_id；连接失效自动降级为轮询 |
| KaTeX 渲染性能（移动端） | 低 | 低 | 仅在题目区域启用 KaTeX；使用 Suspense 延迟渲染 |
| Claude Vision 对非德语数学题处理 | 低 | 低 | prompt 明确指定语言；测试覆盖法语/意大利语题目 |

---

## 九、里程碑

| 里程碑 | 时间节点 | 核心交付 | 状态（2026-08-10） |
|--------|---------|---------|-------------------|
| **M0：瘦身完成 + 全链路联调** | Sprint 0 末（+2周） | 真实环境可用，demo 数据全部移除 | ❌ **未达成** — 联调未做，demo backend 仍在 |
| **M1：AI 记忆 + 多轮对话** | Sprint 1 末（+4周） | AI 认识学生，学生可追问 | 🟡 代码达成，未在真实环境验证 |
| **M2：多模态 + 流式 UX** | Sprint 2 末（+6周） | 图片识别 90%+，LaTeX 渲染，打字机效果 | 🟡 UX 部分达成；多模态阻塞 |
| **M3：自适应 + 实时通知** | Sprint 3 末（+8周） | 个性化推荐，WebSocket 替代轮询 | 🟡 推荐达成；WebSocket 未部署 |
| **M4：报告 + 老师 AI 2.0** | Sprint 4 末（+10周） | 叙述式家长报告，老师效率 3× | 🟡 报告已达成（早于计划）；老师工具未开始 |
| **M5：Knowledge Base + PWA** | Sprint 5 末（+12周） | 课程 RAG，离线访问 | ⛔ 两项均待决策 |
| **M6：Bedrock Agents（Premium）** | Sprint 6 末（+14周） | Premium 多步 AI 推理上线 | ⬜ 未开始 |

> **M0 是关键路径。** M1–M4 的「🟡 代码达成」都建立在未验证的链路上，
> 在 M0 补齐前不应视为可交付。

---

## 十、附录：文件变更清单

> 本节为设计期的预估清单，实际落地路径与之有出入，已在每行标注。

### 需要新增的文件

**后端**：
```
src/stoa/routers/websocket.py          # ✅ 已存在（含 services/websocket_service.py）
src/stoa/services/memory_service.py    # 🟡 实为 services/adaptive_learning_service.py
src/stoa/services/rag_service.py       # ⬜ 未创建
src/stoa/services/agent_service.py     # ⬜ 未创建
src/stoa/models/memory.py              # 🟡 记忆模型并入 adaptive learning 相关模块
src/stoa/db/repositories/memory_repo.py  # 🟡 实为 db/repositories/adaptive_learning_repo.py
```

**前端**：
```
src/hooks/useWebSocket.ts                    # ✅ 已存在
src/components/ui/MathRenderer.tsx           # ✅ 已创建（路径为 ui/ 非 chat/）
src/components/dashboard/WeakTopicsCard.tsx  # ✅ 已创建（名称非 WeakConcepts）
src/components/dashboard/RecommendedPracticeCard.tsx  # ✅ 已创建（名称非 DailyRecommend）
src/services/learning/memoryApi.ts           # ✅ 新增，未在原清单中
src/hooks/learning/useWeakTopicsQuery.ts     # ✅ 新增，未在原清单中
src/mocks/handlers/index.ts                  # 🟡 存在，但仅测试侧消费
src/mocks/browser.ts                         # ❌ 已删除 — 违反发布门禁，见 §2
tests/mswServer.ts                           # ✅ 替代方案：Node 侧 MSW server
```

**测试基础设施（原清单未覆盖）**：
```
stoa-frontend/vitest.config.ts               # ✅ 组件测试配置（jsdom）
stoa-frontend/tests/setup.ts                 # ✅ 含 @/lib/env mock
stoa-frontend/tests/component/*.test.tsx     # ✅ 5 个组件/Hook 测试文件
stoa-backend/conftest.py                     # ✅ 根级 sys.path 修正
stoa-backend/tests/test_conversation_memory_context.py  # ✅ 记忆注入集成测试
```

**基础设施**：
```
stoa-infra/stacks/stoa_websocket_stack.py   # WebSocket API Stack
stoa-infra/stacks/stoa_knowledge_base_stack.py  # Knowledge Base Stack
```

### 需要修改的文件

**后端**：
```
src/stoa/services/ai_service.py        # 注入学生画像到 prompt
src/stoa/services/report_service.py   # 增加叙述式摘要生成
src/stoa/routers/questions.py         # 新增 /chat 端点
src/stoa/routers/teachers.py          # AI draft 2.0 增强
```

**前端**：
```
src/pages/student/AnswerPage.tsx      # 追问输入框 + LaTeX
src/components/chat/MessageBubble.tsx # 流式渲染 + KaTeX
src/components/parent/WeeklyReport.tsx # AI 摘要段落
src/components/tutor/TutorSession.tsx  # 相似题推荐面板
```

---

*本文档为 STOA 2.0 开发的整体设计规划，Sprint 开始前各模块应细化为具体 Task。*
