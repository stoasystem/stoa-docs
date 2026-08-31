# STOA 项目瘦身规划

> 生成时间：2026-08-04  
> 分析范围：stoa-frontend / stoa-backend / stoa-infra / stoa-docs  
> 目标：精简代码体积、移除废弃功能、统一技术路线、降低维护成本

---

## 一、执行摘要

STOA 已从「前端 Demo + SQLite」演进为 AWS 上可运行的生产级学习平台。然而在快速迭代过程中积累了大量技术债：**双后端并存、双 Store 并存、未挂路由的遗留页面、783 + 2267 个规划文件混入工作区、并行的双首页实验**等。

本规划将瘦身工作分为 **5 个层次**：

| 层次 | 工作内容 | 预估收益 |
|------|---------|---------|
| L1 死代码清理 | 删除未引用 Store、遗留页面、废弃组件 | 代码行数 −15% |
| L2 功能收敛 | 合并双首页、合并双 Mistakes 页、明确 Demo Surface 边界 | 维护复杂度 −30% |
| L3 Demo 后端清退 | 移除/隔离 `stoa-frontend/backend`（SQLite demo） | 消除契约漂移风险 |
| L4 文档/规划文件归档 | 将 `.planning/`、`evidence/`、phase 文档移出工作区 | Git 体积 −80%+ |
| L5 依赖与架构优化 | 收敛重复样式 token、优化 Lambda 包体积 | 构建速度提升 |

---

## 二、前端瘦身

### 2.1 死代码：废弃 Store 目录

**问题**：存在两个 Zustand 目录，其中 `src/stores/` 完全无引用。

| 文件 | 状态 | 动作 |
|------|------|------|
| `src/store/authStore.ts` | ✅ 使用中 | 保留 |
| `src/store/uiStore.ts` | ❌ 无引用 | **删除** |
| `src/stores/authStore.ts` | ❌ 无引用 | **删除** |
| `src/stores/questionStore.ts` | ❌ 无引用 | **删除** |

**操作**：
```bash
rm -rf src/stores/
rm src/store/uiStore.ts
```

---

### 2.2 死代码：未挂路由的遗留页面

以下页面在 `AppRouter.tsx` 中**无对应路由**，属于历史遗留，应删除：

**学生旧版页面**：
- `src/pages/student/AskPage.tsx`
- `src/pages/student/AnswerPage.tsx`
- `src/pages/student/HomePage.tsx`
- `src/pages/student/HistoryPage.tsx`

**老师旧版页面**：
- `src/pages/teacher/QueuePage.tsx`
- `src/pages/teacher/SessionPage.tsx`

**家长旧版页面**：
- `src/pages/parent/DashboardPage.tsx`
- `src/pages/parent/ReportPage.tsx`

**计费旧版**：
- `src/pages/VirtualCheckoutPage.tsx`（在 routeConfig 有记录但未挂路由）

**操作**：逐一确认无动态 `import()` 后统一删除。

---

### 2.3 功能收敛：合并双首页

**问题**：`/` (HomePage) 与 `/home-v2` (HomeV2Page) 并行存在，各自有独立的组件目录、样式文件和 i18n key，造成双倍维护成本。

**建议**：
1. 产品侧确认最终采用哪个版本（建议采用 `/home-v2` 的 Premium 风格）
2. 将选定版本的组件移至 `src/components/home/`，统一命名
3. 删除被淘汰版本的所有相关资产

**涉及文件**：
```
# 若保留 home-v2，需删除：
src/components/home/           # 旧版 home 组件
src/styles/stoa-theme.css      # 与旧 home 对应的主题（确认后）
src/pages/student/HomePage.tsx

# 若保留 home，需删除：
src/components/home-v2/        # home-v2 专属组件
src/styles/home-v2-premium.css
src/styles/premium-theme.css
src/pages/HomeV2Page.tsx
```

**并删除 `/home-v2` 路由入口**。

---

### 2.4 功能收敛：合并双 Mistakes 页面

**问题**：`/practice/mistakes` 与 `/question-bank/mistakes` 是两套功能重叠的错题本实现。

**建议**：
1. 评估两者的差异（数据源是否不同）
2. 若数据源相同，合并为统一的 `MistakesPage`，在侧边栏从两处入口指向同一路由
3. 若数据源不同，明确命名区分（如 `PracticeMistakesPage` vs `QuestionBankMistakesPage`）并添加说明注释

---

### 2.5 功能收敛：明确 Demo Surface 边界

以下功能被 `DemoSurfaceRoute` 包裹，意味着它们**在生产环境对普通用户不可见**，但代码仍在主代码库中维护：

- `/organization/*` 多租户组织页（全部）
- `/students/:id/diagnosis`
- `/students/:id/curriculum-graph`
- `/curriculum-graph`

**建议选项**：

| 选项 | 描述 | 适用场景 |
|------|------|---------|
| A. 保持现状 | 维持 DemoSurfaceRoute 包裹 | 近期有 B2B 销售演示需求 |
| B. 迁移到独立分支 | 功能移至 `feature/organization-demo` 分支 | 无近期演示需求 |
| C. 提升为正式功能 | 移除 DemoSurfaceRoute，完成真实数据对接 | 已有付费学校客户 |

**推荐**：若 6 个月内无学校客户签约计划，选 B（迁移分支）。

---

### 2.6 功能裁剪：Live Classroom

**现状**：`src/features/live-classroom/` 存在 UI 组件，但文档**明确声明无真实视频能力**，仅为 UI 框架。

**建议**：
- 短期：在组件目录添加 `README.md` 说明"此功能为规划中，暂未实现"，防止误以为可用
- 中期：若无明确上线计划，将整个目录迁移到 `feature/live-classroom` 分支或归档

---

### 2.7 Mock 数据清理

以下 mock 数据文件在生产 API 已可用的情况下应逐步移除：

| 文件 | 建议 |
|------|------|
| `src/data/phase11MockData.ts` | 确认无生产引用后删除 |
| `src/data/phase12MockData.ts` | 确认无生产引用后删除 |
| `src/data/mockPractice.ts` | 替换为真实 API 调用 |

**操作**：全局搜索 `import.*phase11Mock\|import.*phase12Mock\|import.*mockPractice`，确认引用来源后删除。

---

### 2.8 样式 Token 收敛

**现状**：存在多个主题 CSS 文件，部分重叠：

| 文件 | 用途 |
|------|------|
| `stoa-theme.css` | 主品牌主题 |
| `premium-theme.css` | Premium 版主题 |
| `brand-tokens.css` | 品牌 token |
| `platform-theme.css` | 平台主题 |
| `home-v2-premium.css` | home-v2 专属 |

**建议**：与双首页合并工作同步，确定一套主题后删除其余，保留 `stoa-theme.css` 为唯一入口，在其中通过 CSS `@layer` 或 data-attribute 切换 Premium 变体。

---

### 2.9 角色命名统一

**问题**：前端代码中 `tutor` 与 `teacher` 两套名称混用（authStore 做了 alias，但遗留 store 仍用 `teacher`）。

**建议**：删除废弃 `src/stores/` 后，在现用代码中统一使用 `tutor`（与后端 `/teachers` 路由对应，但注意更新注释与文档）。

---

## 三、后端瘦身

### 3.1 移除 Demo Backend（高优先级）

**问题**：`stoa-frontend/backend/` 是一个基于 SQLite 的本地 demo FastAPI 服务，与生产 `stoa-backend`（DynamoDB/Lambda）并行存在。

**风险**：
- 前端开发者可能在 demo 后端验证通过后误以为生产后端也支持该功能
- API 契约持续漂移
- 开发者需要同时理解两套后端

**建议**：
```
# 步骤 1：确认 stoa-frontend 的 src/services/ 全部指向生产 API（VITE_API_BASE_URL）
# 步骤 2：将 demo backend 的独特测试场景迁移为 mock service worker (MSW) 文件
# 步骤 3：删除 stoa-frontend/backend/ 目录
# 步骤 4：删除 package.json 中的 "demo:backend" 和 "demo:reset" scripts
```

**替代方案**：用 [MSW (Mock Service Worker)](https://mswjs.io/) 替代 demo 后端，保持前端可离线开发的同时消除双后端问题。

---

### 3.2 Admin 路由瘦身

**现状**：`/admin` 前缀下已有 **111+ 路由**，其中包含大量 `placeholder` 路由和部分仅用于 phased testing 的端点。

**建议**：
1. 审查所有 `placeholder` 端点，删除确认永远不上线的
2. 将纯内部运维脚本（如 billing migration jobs）移至独立的管理脚本，而非暴露为 HTTP 端点
3. 考虑为 admin 路由增加 feature flag，生产关闭未完成的端点

---

### 3.3 测试文件清理

**现状**：`stoa-backend/tests/` 有 ~131 个测试文件，其中大量命名为 `test_phase473_*`、`test_phase474_*` 等 **phase 证据测试**。

**问题**：phase 证据测试与业务逻辑测试混杂，难以区分哪些是持续维护的回归测试，哪些是一次性验收证明。

**建议**：
```
tests/
├── unit/          # 单元测试（持续维护）
├── integration/   # 集成测试（持续维护）
├── e2e/           # 端到端测试（持续维护）
└── archive/       # 已归档的 phase 证据测试（不参与 CI）
```

将所有 `test_phase*` 文件移至 `archive/`，在 `pytest.ini` 中排除该目录。

---

### 3.4 Lambda 包体积优化

**现状**：单 Lambda monolith 包含所有路由（admin/billing/AI/notifications/files 等），冷启动和包体积风险较高。

**建议**（按优先级）：

| 优先级 | 方案 | 描述 |
|--------|------|------|
| 短期 | 依赖分层（Lambda Layer） | 将 `boto3`、`pillow`、`pypdf` 等大包放入 Layer |
| 中期 | 拆分 admin 路由为独立 Lambda | admin 使用频率低但路由多，独立部署减少主 Lambda 体积 |
| 长期 | 评估微服务边界 | billing / notifications 可作为候选 |

---

### 3.5 移动端仓库边界重新划定

**现状**：`stoa-backend/mobile/`（Expo React Native）嵌套在后端仓库内，造成仓库边界混乱。

**建议**：
- 将 `mobile/` 移出为独立仓库 `stoa-mobile`
- 或移至 monorepo 顶层 `stoa-mobile/`
- 不应与 Python 后端代码共享仓库

---

## 四、文档与规划文件整理

### 4.1 `.planning/` 目录归档（高优先级，高收益）

**现状**：
- `stoa-frontend/.planning/`：~783 文件
- `stoa-backend/.planning/`：~2267 文件
- 合计约 **3050 个规划产物**混入工作区

**影响**：
- IDE 索引慢、搜索噪音大
- Git 历史膨胀
- 新团队成员困惑

**建议**：
```bash
# 方案 A：移至独立归档仓库
git archive HEAD stoa-frontend/.planning stoa-backend/.planning | \
  gzip > stoa-planning-archive-$(date +%Y%m).tar.gz

# 添加到 .gitignore
echo ".planning/" >> .gitignore

# 方案 B：保留最新 5 个 phase 的规划，其余移至 archive 分支
```

---

### 4.2 `docs/` 目录整理

| 目录 | 文件数 | 建议 |
|------|--------|------|
| `stoa-frontend/docs/` | 234+ | 保留架构决策和 API 文档；删除 phase QA/release note（已有 git log） |
| `stoa-backend/docs/build-week-2026/` | OpenAI Build Week 材料 | 迁移至 `stoa-docs/` 或独立归档 |
| `stoa-backend/evidence/` | Phase 证据包 | 归档到外部存储（S3/语雀/Notion） |
| `stoa-docs/` | 核心文档 | **保留并更新**（HLD/PRD/ADR） |

---

### 4.3 `stoa-docs/` 文档更新

当前 `HLD.md` 中的 API 清单（约 2026-05 版本）已远落后于实际实现（admin 已 100+ 路由）。

**需要同步更新**：
- [ ] HLD.md：更新 API 路由清单
- [ ] HLD.md：补充 billing/notifications/adaptive 的数据模型
- [ ] PRD.md：标注已上线功能 vs 规划功能
- [ ] ADR.md：补充近期关键架构决策（如单表 DynamoDB 设计）

---

## 五、依赖优化

### 5.1 前端依赖审查

| 依赖 | 版本 | 风险/建议 |
|------|------|----------|
| `zod` | v4.4.3 | 较新版本，确认 `@hookform/resolvers` 兼容 v4 |
| `aws-amplify` | v6 | 若仅用 Cognito Auth，考虑替换为更轻量的 `@aws-sdk/client-cognito-identity-provider`，减少 bundle 体积 |
| `react-error-boundary` | - | React 19 已内置 ErrorBoundary 能力，可评估是否仍需此包 |

**建议运行**：
```bash
cd stoa-frontend
npx depcheck   # 检测未使用依赖
npx bundlemon  # 监控 bundle 体积
```

### 5.2 后端依赖审查

```bash
cd stoa-backend
pip-audit              # 检查安全漏洞
uv pip list --outdated # 检查过期包
```

重点关注：
- `python-jose`（JWT）：确认无 CVE
- `cryptography`：保持最新版
- `pillow`：图像处理库，频繁有 CVE，需定期更新

---

## 六、优先级矩阵与执行路线图

### 优先级矩阵

| 任务 | 影响 | 工作量 | 优先级 |
|------|------|--------|--------|
| 删除 `src/stores/` 和 `uiStore.ts` | 中 | 极低 | ~~**P0 立即执行**~~ ✅ 已完成 |
| 归档 `.planning/` 目录 | 高 | 低 | ~~**P0 立即执行**~~ ✅ 已完成 |
| 删除未挂路由的遗留页面 | 中 | 低 | ~~**P1 本周**~~ ✅ 已完成 |
| 合并双首页 | 高 | 中 | ~~**P1 本周**~~ ✅ 已完成 |
| 修复 VirtualCheckoutPage 缺失路由 | 中 | 极低 | ✅ 已完成（顺带修复） |
| 移除 demo backend | 高 | 中 | **P1 待完成** |
| 合并双 Mistakes 页 | 低 | 低 | **P2 本月**（数据源不同，暂不合并） |
| 清理 mock 数据文件 | 中 | 低 | **P2 本月** |
| 整理 docs/ 目录 | 中 | 中 | **P2 本月** |
| 样式 Token 收敛 | 中 | 高 | **P3 下季度** |
| Live Classroom 归档/明确 | 低 | 低 | **P2 本月** |
| Organization demo 边界决策 | 中 | 中 | **P2 本月** |
| Lambda 包体积优化 | 高 | 高 | **P3 下季度** |
| 移动端仓库分离 | 中 | 中 | **P3 下季度** |
| 更新 stoa-docs HLD/PRD | 中 | 中 | **P2 本月** |
| 后端测试文件分类归档 | 中 | 低 | **P2 本月** |

---

### 执行路线图

```
Week 1（立即）
├── [frontend] 删除 src/stores/ 目录和 uiStore.ts
├── [both]     将 .planning/ 加入 .gitignore 并本地归档
└── [backend]  将 test_phase* 文件分类移至 tests/archive/

Week 2-3（短期）
├── [frontend] 删除未挂路由的 8 个遗留页面
├── [frontend] 产品确认保留哪个首页版本
├── [frontend] 删除确认无用的 mock 数据文件
└── [backend]  评估并规划 demo backend 的 MSW 替代方案

Week 4（月底前）
├── [frontend] 执行首页合并（删除被淘汰版本的全部资产）
├── [frontend] 合并双 Mistakes 页面
├── [backend]  完成 demo backend 移除
├── [docs]     整理 stoa-frontend/docs/ 保留核心文档
└── [docs]     更新 stoa-docs/HLD.md API 清单

Month 2-3（下季度）
├── [frontend] 样式 Token 收敛至单一主题文件
├── [backend]  Lambda Layer 优化（boto3/pillow）
├── [mobile]   移动端仓库独立
└── [infra]    评估 admin Lambda 拆分可行性
```

---

## 七、预期收益

| 指标 | 当前估算 | 瘦身后估算 |
|------|---------|-----------|
| 前端源文件数 | ~1013 | ~800（−20%） |
| 后端测试文件数（CI参与） | ~131 | ~60（−55%） |
| 工作区文件总数（含规划） | ~4000+ | ~900（−78%） |
| IDE 索引文件数 | ~4000+ | ~900 |
| 前端 bundle 体积（aws-amplify优化后） | - | 预估 −30% |
| 新开发者 onboarding 理解时间 | - | 预估缩短 50% |

---

## 八、注意事项与风险

1. **删除前必须确认**：所有删除操作前请在 `git` 中创建 `chore/slim-cleanup` 分支，确保可回滚
2. **E2E 测试兜底**：在删除页面/组件后运行 `npm run test:e2e` 确保 26 个 Playwright 规格全部通过
3. **首页合并需产品决策**：技术团队不应单方面决定删除哪个首页版本
4. **Demo Surface 决策需业务输入**：Organization 功能是否保留取决于 B2B 销售策略
5. **移动端分仓库前**：确保 CI/CD 流水线已针对新仓库位置更新

---

*本规划由 AI 分析生成，执行前请团队评审确认。*
