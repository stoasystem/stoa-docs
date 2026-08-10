# STOA 开发设计 2.0

> **版本：** v2.0  
> **创建日期：** 2026-08-04  
> **状态：** 规划中  
> **前置文档：** [PLAN.md](./PLAN.md) · [PRD.md](./PRD.md) · [ADR.md](./ADR.md) · [PROJECT_SLIM_PLAN.md](../PROJECT_SLIM_PLAN.md)

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

> 详细执行步骤见 [PROJECT_SLIM_PLAN.md](../PROJECT_SLIM_PLAN.md)

### ✅ 已完成（P0）

| 操作 | 完成时间 |
|------|---------|
| 删除 `src/stores/`（死代码） | 2026-08-04 |
| 删除 `src/store/uiStore.ts`（死代码） | 2026-08-04 |
| `.planning/` 加入前后端 `.gitignore` | 2026-08-04 |
| `evidence/` 加入后端 `.gitignore` | 2026-08-04 |

### 📋 2.0 开发前必须完成（P1）

| 操作 | 负责人 | 预计工时 |
|------|--------|---------|
| 删除 8 个未挂路由的遗留页面 | 前端 | 2h |
| 产品确认保留哪个首页版本并合并 | 产品 + 前端 | 4h |
| 移除 `stoa-frontend/backend/` demo backend | 前端 | 3h |
| 用 MSW 替代 demo backend 完成前端离线开发 | 前端 | 8h |
| 完成前后端联调（Cognito → API → UI 全链路） | 全栈 | 16h |

> 🚨 **核心原则**：不在 demo 数据上开发 2.0 功能。2.0 所有新 AI 特性必须基于真实生产 API。

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

#### 功能 A：学生记忆层（Student Memory）

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

#### 功能 B：对话式多轮问答（Conversational AI）

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

#### 功能 C：自适应练习推荐（Adaptive Practice）

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

#### 功能 D：AI 辅助老师工作台（Teacher AI Tools 2.0）

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

#### 功能 E：家长 AI 叙述式报告（Narrative Report）

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

#### 功能 F：流式响应优化（Streaming UX）

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

#### 功能 G：多模态输入增强（Multimodal Input）

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

#### 引入 1：Bedrock Agents（自主 AI 工作流）

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

#### 引入 2：Bedrock Knowledge Base（RAG 知识库）

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

#### 引入 3：WebSocket 实时通知（替代轮询）

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

#### 引入 4：LaTeX / 数学公式渲染

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

#### 引入 5：PWA + 离线能力

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

| 任务 | 工时 |
|------|------|
| 删除 8 个遗留页面 | 2h |
| 合并双首页（产品确认后） | 4h |
| 移除 demo backend，引入 MSW | 8h |
| 前后端全链路联调（Cognito → API → UI） | 16h |
| 创建真实测试账号（学生/家长/老师各 1 个） | 2h |
| 端到端跑通：注册 → 提问 → AI 回答 → 老师接管 | 8h |

**交付物**：真实环境下完整核心流程可用

---

### Sprint 1（第 3–4 周）：学生记忆层 + 对话式追问

**目标**：AI 能记住学生，学生可以追问

| 任务 | 工时 |
|------|------|
| DynamoDB 新增 `student_memory` 实体 | 4h |
| 后端：每次问答后自动提取知识点标签并存入 memory | 8h |
| 后端：提问时附加学生画像到 system prompt | 4h |
| 后端：`/questions/{id}/chat` 多轮追问 API | 6h |
| 前端：解答页追问输入框 + 对话历史展示 | 8h |
| 前端：Dashboard「我的薄弱点」卡片（来自 memory） | 4h |

---

### Sprint 2（第 5–6 周）：多模态升级 + 流式 UX + LaTeX

**目标**：图片识别更准，AI 回答体验更流畅

| 任务 | 工时 |
|------|------|
| 后端：Claude Vision 替换 Rekognition OCR | 8h |
| 后端：AI prompt 增加 LaTeX 输出指令 | 2h |
| 前端：引入 KaTeX，解答页 LaTeX 渲染 | 4h |
| 前端：流式答案打字机效果（ReadableStream） | 6h |
| 前端：骨架屏 + 流式过程中的 UX 优化 | 4h |
| 测试：10 道真实手写数学题 OCR 对比测试 | 4h |

---

### Sprint 3（第 7–8 周）：自适应练习 + WebSocket

**目标**：AI 主动推荐练习，实时替代轮询

| 任务 | 工时 |
|------|------|
| 后端：完善 `/adaptive/recommendations` 算法（基于 memory） | 8h |
| 后端：API Gateway WebSocket API 部署 | 8h |
| 后端：answer_ready / teacher_replied 事件 WebSocket 推送 | 6h |
| 前端：移除所有 `refetchInterval` 轮询，改为 WebSocket | 8h |
| 前端：Dashboard「今日推荐练习」个性化卡片 | 6h |

---

### Sprint 4（第 9–10 周）：AI 叙述式报告 + 老师 AI Tools 2.0

**目标**：提升家长报告质量，提升老师效率

| 任务 | 工时 |
|------|------|
| 后端：周报生成增加 AI 叙述段落（德语） | 6h |
| 前端：周报顶部 AI 摘要组件 | 4h |
| 后端：老师智能草稿 2.0（结合学生画像） | 6h |
| 后端：相似题推荐 API（Bedrock Embedding） | 8h |
| 前端：老师工作台新增「相似题推荐」面板 | 6h |
| 前端：老师工作台新增「错误模式分析」提示 | 4h |

---

### Sprint 5（第 11–12 周）：Bedrock Knowledge Base + PWA

**目标**：AI 与瑞士课程绑定，支持离线访问

| 任务 | 工时 |
|------|------|
| 准备 Lehrplan 21 数学知识点文档（PDF/Markdown） | 16h（内容） |
| 后端：部署 Bedrock Knowledge Base（S3 + Titan Embeddings） | 8h |
| 后端：提问时 RAG 检索相关课程内容 | 6h |
| 前端：引入 `vite-plugin-pwa`，配置 Service Worker | 4h |
| 前端：历史问答离线缓存 | 4h |
| 测试：PWA 安装到手机主屏幕验证 | 2h |

---

### Sprint 6（第 13–14 周）：Bedrock Agents（Premium AI）

**目标**：Premium 用户享受更强大的 AI 推理

| 任务 | 工时 |
|------|------|
| 设计 Agent Action Group（identify_concepts / retrieve_knowledge / generate_steps） | 8h |
| 后端：部署 Bedrock Agent，配置 Lambda action executor | 12h |
| 后端：按订阅 tier 路由（Premium → Agent，Standard/Free → Haiku） | 4h |
| 前端：Premium 用户解答页增加「深度分析」标记 | 2h |
| 测试：复杂数学题 Agent vs Haiku 质量对比 | 4h |

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

| 里程碑 | 时间节点 | 核心交付 |
|--------|---------|---------|
| **M0：瘦身完成 + 全链路联调** | Sprint 0 末（+2周） | 真实环境可用，demo 数据全部移除 |
| **M1：AI 记忆 + 多轮对话** | Sprint 1 末（+4周） | AI 认识学生，学生可追问 |
| **M2：多模态 + 流式 UX** | Sprint 2 末（+6周） | 图片识别 90%+，LaTeX 渲染，打字机效果 |
| **M3：自适应 + 实时通知** | Sprint 3 末（+8周） | 个性化推荐，WebSocket 替代轮询 |
| **M4：报告 + 老师 AI 2.0** | Sprint 4 末（+10周） | 叙述式家长报告，老师效率 3× |
| **M5：Knowledge Base + PWA** | Sprint 5 末（+12周） | 课程 RAG，离线访问 |
| **M6：Bedrock Agents（Premium）** | Sprint 6 末（+14周） | Premium 多步 AI 推理上线 |

---

## 十、附录：文件变更清单

### 需要新增的文件

**后端**：
```
src/stoa/routers/websocket.py          # WebSocket 路由
src/stoa/services/memory_service.py   # 学生记忆层服务
src/stoa/services/rag_service.py       # Bedrock Knowledge Base RAG
src/stoa/services/agent_service.py    # Bedrock Agents 封装
src/stoa/models/memory.py             # 记忆相关 Pydantic 模型
src/stoa/db/repositories/memory_repo.py  # 记忆 DynamoDB 仓库
```

**前端**：
```
src/hooks/useWebSocket.ts              # WebSocket 连接管理 Hook
src/components/chat/MathRenderer.tsx  # KaTeX 数学公式渲染
src/components/dashboard/WeakConcepts.tsx  # 薄弱知识点卡片
src/components/dashboard/DailyRecommend.tsx # 今日推荐卡片
src/mocks/handlers.ts                  # MSW mock handlers（替代 demo backend）
src/mocks/browser.ts                   # MSW 浏览器启动配置
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
