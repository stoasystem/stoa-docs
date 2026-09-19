# 001 认证系统改造：开放注册 → 邀请制，Cognito → Authentik

- 状态：**未采纳**（2026-09-19 需求方否决，未动任何代码）

> ## ⛔ 否决记录（2026-09-19）
>
> **否决理由**：需求方原以为「开源 = 免费」，看到月成本后否决。
> 澄清：Authentik 软件本身免费，$117~280/月**全部是 AWS 基础设施费**
> （Fargate $39 + RDS $15 + ALB $28 + EFS + 公网 IP + Secrets），与软件是否开源无关。
> Cognito 现在 $0 是因为 10,000 MAU 以内 AWS 免费，等于白嫖一个托管 IdP。
> 自托管省掉的是 SaaS 授权费，换来的是基础设施费 + 运维时间——
> **而 Cognito 在当前量级的授权费本来就是零**，所以这笔交换在今天是净亏。
>
> **什么条件下重新评估**（任一成立即应重开此卡）：
> 1. 真的要接第二个系统做跨系统 SSO，且该系统无法接 Cognito
> 2. 出现合规或数据主权的硬要求，身份数据不允许放在 AWS 托管服务里
> 3. MAU 超出 Cognito 免费额度且费用逼近自建成本（约 10,000 MAU 之后才需要算这笔账）
> 4. 需要 Cognito 做不到的复杂认证流（多步条件化注册、按角色分支、内嵌审核）
>
> **本文件其余内容保留**，因为选型依据、成本明细、风险分析在上述条件触发时可直接复用。
> 若重开，注意核对：AWS 定价、Authentik 版本与 CVE 状况都会过期。
- 日期：2026-09-19
- 提出：「登录注册系统之前是全范围注册没有限制，改为邀请制或管理员制，因为业务目前对内，
  暂不对 C 端开放。框架改为开源 SSO 框架，Keycloak / Authentik 调研一下哪个更适合。
  要求对之前的注册、登录系统推翻改造。」

---

## 一、结论

**选 Authentik。** 不选 Keycloak，也不留在 Cognito。

### 这个结论是怎么得出的

调研 agent 的默认推荐其实是**「继续用 Cognito」**，理由是邀请制 Cognito 已具备 90%、月成本 $0、零运维。
它的推荐在两个前提下成立，而需求方的回答把这两个前提都推翻了：

| 调研的前提 | 实际答复 | 影响 |
| --- | --- | --- |
| 「只是要邀请制」 | 要的是**接多个系统做真 SSO、自托管掌控数据、多端登录、更正规** | 邀请制只是触发点，不是目标 |
| 「迁移代价大（Cognito 不导出密码哈希，要全员改密）」 | **只有 5 个账号，全是测试号，可以重做** | 迁移代价 ≈ 0 |

调研自己写了改选条件的原文：**「已经决定要自建，但没有惰性迁移的硬约束——那就选 authentik 而不是
Keycloak：Keycloak 的优势（Java SPI、LDAP、企业级集群规模）STOA 一个都用不上，而它的代价
（JVM 内存、只支持最新版、25→26 那种会话清空 + DB 不可降级的升级）要全额承担。」**
本项目正落在这一条上。

### 为什么不是 Keycloak

- Keycloak 唯一压倒 Authentik 的能力是 **User Storage SPI 做惰性密码迁移**（首次登录时拿明文回打
  Cognito 验证），但那要写 Java，而**我们根本不需要迁移密码**。
- 上游**只支持最新版**——出新版就必须升，否则拿不到安全补丁。1 人团队扛不动这个节奏。
- 栈是 JVM / Quarkus / Infinispan，与本项目 Python 后端完全异质，排障要读 Java 栈追踪。

### 为什么不留在 Cognito

- 它解决不了需求方真正要的四件事里的三件：自托管、数据主权、跨系统 SSO。
- 锁定只会加深：今天 5 个账号迁移代价为零，用户长起来之后密码仍然导不出来，那时的痛是今天的若干倍。
  **现在动是成本最低的时刻**，这一点本身就是动手的理由。

---

## 二、需要正视的代价（不粉饰）

| 项 | 现状 | 改造后 |
| --- | --- | --- |
| 月成本 | **$0**（Cognito Lite 免费额度内） | **$117 起**（最小可用）／**$190~280**（生产姿势 Multi-AZ） |
| AWS 资源 | 无 VPC、无容器、无关系库 | **从零建网络**：VPC + 子网 + ALB + ACM + RDS PostgreSQL + ECS Fargate×2 任务 + EFS + Secrets |
| 运维 | 零 | RDS 快照与恢复演练、blueprint 版本化、镜像构建推送、**安全补丁跟进** |
| 升级节奏 | AWS 自己升 | Authentik 3 个月一版，**只支持最近 2 个版本** |

**安全补丁不是可选项**：Authentik 2026 年出过 **CVE-2026-49448（CVSS 9.8，发一个空 POST 即可未认证
绕过认证）**，Keycloak 出过 **CVE-2026-18963（CVSS 9.1，完全账号接管）**。自建等于把这类补丁的响应
时间揽到自己身上。这条不接受就不该自建。

> eu-central-2（苏黎世）**不支持 App Runner**，所以没有「托管容器」这条捷径，只能 ECS Fargate。
> 跨区会破坏 ADR-006 的统一部署区域与瑞士数据驻留，不考虑。

---

## 三、头号风险：push 即部署 × 容器启动自动迁移

**这是本项目特有的、调研点出来但必须由我们自己处置的风险。**

实测：`stoa-infra/.github/workflows/deploy-production.yml` 是 `on: push: branches: [main]`，
与前后端一样 **push 即部署，没有 PR 缓冲**。而 Authentik 容器**启动时自动跑 Django migrations**。

两者相乘的后果：**一次无人值守的 push 可以把线上登录打掉，且数据库已迁到半路，没有回滚路径。**

### 处置（写进卡的硬约束）

1. **IdP 相关的 CDK 变更不走自动部署**。新建的 `IdentityStack` 从 GitHub Actions 的自动 deploy 里
   排除，只允许本地 `cdk diff` 确认后手动 `cdk deploy`。
2. **RDS 开启自动快照 + PITR，并在切换前做一次恢复演练**（演练算验收项，不是建议）。
3. **Authentik 版本固定到具体 tag**，不使用 `latest`；升级单独立卡，先在独立 stack 上演练。

---

## 四、两个会简化改造的既有事实

摸底时发现，实际工作量比预想小，因为：

1. **邀请-激活机制项目里已经有了**，目前只服务教师申请：
   `teacher_application_service.py` 的 `_issue_invitation()` 用 `secrets.token_urlsafe(32)` 生成令牌、
   带 `expires_at`、**令牌只在签发那一刻可读**、支持 `reissue_invitation` 重发；
   端点 `POST /teacher-applications/{id}/invitations`、`/activation/claim`、`/activation/consume`
   都已存在，前端也有 `TeacherActivatePage.tsx`。**这套语义可以直接搬到 Authentik 的 Invitation 上。**
2. **角色不在 IdP 里**。`security/identity.py:152` 是从 DynamoDB 的 account 行读 `role` 的，
   Cognito 只负责认证。**所以换 IdP 不触动授权层**——208 条路由授权清单的判定逻辑不需要重写，
   这是本次改造范围能收窄的最大原因。
3. **多 issuer 天生支持**：`config.py` 的 `cognito_allowed_issuers` / `cognito_access_client_ids`
   本来就是 List，校验层可以同时受理两个 issuer 的 token → **灰度并存可行，不必一次性硬切**。

---

## 五、不做清单

> 格式按流程要求：不做什么 / 为什么 / 什么条件下重新评估。
> 没有恢复条件的「不做」会变成「永远不做」，而那通常不是当初的意思。

| 不做什么 | 为什么 | 什么条件下重新评估 |
| --- | --- | --- |
| **不做密码迁移** | 只有 5 个测试账号，需求方明确说可以重做。Cognito 不导出密码哈希，做惰性迁移要引入 Keycloak + 写 Java | 切换前发现存在**真实用户**（非测试号）且不可要求其重设密码 |
| **不做 LDAP / AD 联邦** | 当前没有学校 IT 对接需求，Authentik 支持但配置与测试成本不小 | 有瑞士学校要求用其目录登录 |
| **不改授权层（208 条路由清单）** | 角色存在 DynamoDB，不在 IdP。换认证不等于换授权，混在一起做会让这张卡无法独立验收 | 将来要把角色下放到 IdP 做跨系统统一授权 |
| **不上 MFA** | 对内业务、用户可控；MFA 会显著拉长首版的测试面 | 接入真实学生/家长，或处理支付与成绩等敏感操作 |
| **不做 Hosted UI / 托管登录页** | 现有登录是后端代理密码登录（`/auth/login` → `cognito.initiate_auth`），前端自绘表单，四语已适配 | 要支持第三方身份源（Google / Microsoft 学校账号）登录 |
| **首版不上 Multi-AZ** | $117 vs $259，对内业务可接受一次几分钟的不可用；省下的钱先验证方案可行 | 开始接真实用户，或不可用会造成业务损失 |
| **不重写多标签页多角色机制**（`devSessions.ts` / `tabToken()`） | 那是测试账号专用的开发便利，与生产认证无关，且上一轮刚修过它的登出缺陷 | 它与新 token 模型产生冲突 |
| **不在本卡修前端 refresh 缺口** | 后端**已有** `/auth/refresh`（refresh token 30 天），前端从未调用——这是独立的既有缺口，混进来会让本卡验收范围失焦 | **单独立卡，且应在本卡之前或并行做**，见下 |

---

## 六、分卡计划

一张卡 = 一个可独立验收的业务点。**每条 DoD 都要可判**。

### 卡 A（前置，独立于本次改造）补上前端 refresh 链路

后端 `/auth/refresh` 早就有，前端一行没调用，导致 token 1 小时过期即掉线——这正是上一轮 BUG-01
证据链的源头。**换不换 IdP 都要修，而且换 IdP 会把它重做一遍，所以先做。**

DoD：
- [ ] access token 过期后，前端自动用 refresh token 换新 token，用户无感知
- [ ] 判据：把 access token 有效期调到 60 秒跑一次，**61 秒后调任意受保护接口仍返回 200**（不是 401）
- [ ] refresh 失败（refresh token 也过期）时，行为与现在一致：清会话 → 跳 `/login`
- [ ] 投毒：摘掉 refresh 调用 → 上述测试必须变红

### 卡 B 基础设施：IdentityStack

DoD：
- [ ] `cdk diff` 能看到完整资源清单，且**不含 NAT Gateway**（用公网子网 + `assignPublicIp` 规避 $41.76/月）
- [ ] Authentik 版本**固定到具体 tag**，`grep -c "latest" stacks/identity_stack.py` **等于 0**
- [ ] 该 stack **不在** GitHub Actions 自动 deploy 的目标里（判据：workflow 文件里有显式排除，且故意 push 一次验证它没被部署）
- [ ] RDS 自动快照已开启，**并完成一次真实的恢复演练**，演练输出留档
- [ ] `https://auth.stoaedu.ch/` 返回 Authentik 登录页（HTTP 200）

### 卡 C 后端：接受 Authentik 签发的 token

DoD：
- [ ] `config.py` 的 issuer 列表同时含 Cognito 与 Authentik，**两个 issuer 的 token 都能通过校验**
- [ ] 内容型判据（不只是计数）：用 Authentik token 调 `/auth/me`，返回的 `role` 与 DynamoDB 里该账号的
      `role` 字段**逐字相同**——防止「能登录但角色错了」这种计数断言看不出来的错
- [ ] 208 条路由授权清单的校验**仍然全绿**（`uv run pytest tests/test_route_authorization_inventory.py`）
- [ ] 投毒：把 Authentik issuer 从白名单摘掉 → 用它的 token 必须被拒（401），且**不是 500**

### 卡 D 邀请制：关掉自助注册

DoD：
- [ ] Cognito `self_sign_up_enabled=False`，且**服务端拒绝**：直接 curl `POST /auth/register` 返回 4xx，
      不是靠前端藏入口
- [ ] Authentik 侧无 enrollment flow，或 Invitation Stage 的「Continue flow without invitation」已关闭
- [ ] 管理员能建号并发邀请：邀请链接**带过期时间**、**一次性**，过期后点击报明确原因（不是白屏）
- [ ] 内容型判据：邀请里预置的 `role` 与激活后 DynamoDB 里落的 `role` 一致
- [ ] 投毒：把「一次性」标志去掉 → 同一链接用两次必须仍被拒（若通过则说明校验在别处失效）

### 卡 E 前端：登录接到 Authentik

DoD：
- [ ] 四语登录页可用，错误提示跟随界面语言（**不是浏览器语言**）
- [ ] 前端 7 项部署门禁全绿
- [ ] 邀请激活页可用，四语齐全
- [ ] 旧的 `/register` 自助注册入口移除或改为「请联系管理员」

### 卡 F 收尾：下线 Cognito

DoD：
- [ ] 后端 issuer 白名单中移除 Cognito，`grep -rc "cognito" src/stoa/` 降到**只剩注释或零**
- [ ] `cdk diff` 显示 Cognito 用户池将被删除，且**已确认 5 个测试账号已在 Authentik 重建**
- [ ] 全量测试通过

---

## 七、红线判定与审计安排

对照项目 CLAUDE.md 的红线触点：**「鉴权与路由授权：`route-authorization-inventory.json` 及其消费方」**

| 卡 | 红线？ | 判定理由 |
| --- | --- | --- |
| A refresh | **是** | 动 token 生命周期与 401 处置路径 |
| B 基础设施 | 否 | 不碰鉴权判定逻辑；但有**数据库与不可逆部署**风险，按运维流程另行把关 |
| C 后端 token 校验 | **是** | 直接动 issuer 白名单与身份解析，是认证的核心 |
| D 邀请制 | **是** | 邀请令牌的生成／常数时间比较／过期／一次性／枚举防护全是自己的代码 |
| E 前端 | 否 | 展示层 |
| F 下线 Cognito | **是** | 不可逆，且一旦白名单删错会让所有人登不上 |

**A / C / D / F 四张卡完成后，各自另起一个与实现 agent 上下文不共享的 agent 做独立审计，出具意见书。**
成因：主会话既派活又验收就是「自己检查自己」，失败形态是全绿——上一轮 001 卡里，
注册接口的 500 加孤儿账号、BUG-06 只有 math 会炸、BUG-04 跨公式完全失效，
**三个都是独立审计抓的，主会话自己一个都没复核出来**。

---

## 八、待确认（需求方拍板，不替你决定）

1. **域名**：IdP 用 `auth.stoaedu.ch` 可以吗？需要新建 Route53 记录与 ACM 证书。
2. **首版是否接受非 Multi-AZ**（$117/月 vs $259/月）。我方倾向先单 AZ 验证可行性，写进不做清单了。
3. **上一轮遗留的那条**：后端 `age` 字段可选，导致不传 age 就能绕过「未成年必须填家长信息」。
   本次改造会重做注册流程，**正好一起定**：age 改成必填，还是认可「缺失按成年处理」？
4. **SSO 的第二个系统是什么、什么时候接**？这决定卡 C 要不要一开始就按多 client 设计，
   还是先单 client 跑通。

---

## 九、我没验证的部分

- **成本数字来自调研 agent 查的 AWS Price List API（eu-central-2，2026-09-17/18）**，
  其中 ALB 的 LCU 用量、CloudWatch Logs 量是**估的**，不是官方标价。
- Authentik 在 ECS 上**是否仍需要 Redis/Valkey** 未逐版确认；若需要，月成本再加 $15~30。
- 线上实际用户数**没查**（SSO profile 未配好，aws-mcp 这次连接失败）。
  「5 个测试账号」来自需求方口述，**未经我方核实**。切换前必须核实一次，这是卡 F 的前置。
