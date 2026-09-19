# STOA 学习平台功能测试错误清单

- 测试对象：app.stoaedu.ch（STOA | Guided learning support）
- 测试账号：Demo Student（student@test.stoaedu.ch），学科主修 Mathematics，Grade 8 · Swiss Gymnasium，回复语言（Antwortsprache）设置为 Deutsch
- 测试方式：人工点击 + 页面文本/控制台/网络请求检查
- 测试日期：2026-09-16
- 测试范围：首页新建对话、学科选择、聊天问答、Learn 模块（Exercises / Guided path / Mistakes / Progress）、Profile 页、通知面板、多语言切换（EN/DE）、角色切换面板、登录页
- 未覆盖范围（第一轮）：测试进行到"角色切换"步骤时账号被强制登出（见 BUG-01），且没有可用凭证重新登录，因此"继续阅读 Reading and reducing fractions"课程内容、"Review 1 question"复习流程、"Ask a Tutor"真实转人工、"Report answer"举报流程均未能测完。
- 第二轮补充测试：账号仍处于登出状态（无可用凭证找回登录），因此本轮转为测试登录页/注册流程（不涉及真实提交注册、不输入真实密码）、公开营销页与页脚链接、帮助中心与 Onboarding 页面、隐私政策与使用条款页面。新发现的问题见 BUG-11 ~ BUG-17。移动端响应式布局因运行环境限制（浏览器侧边栏无法真实改变视口宽度）本轮未能有效验证，暂不下结论。

---

## BUG-01（严重）角色切换会直接把用户登出，会话丢失

**现象**：Profile 页右下角悬浮的"student"徽章点开后会展示一个"Testing as"面板，列出当前身份 `student@test.stoaedu.ch · this tab only`，下方有一个可选身份 `teacher · teacher`，以及"+ Add a role"。点击 `teacher · teacher` 后，页面没有切换成 teacher 视角，而是整个跳转到了登录页（`/login`），原有 Demo Student 会话完全失效。用浏览器"后退"也无法恢复，再次访问 `/chat` 依然被重定向到登录页。

**复现步骤**：
1. 以 Demo Student 身份登录，进入任意页面（如 Profile 页 `/learn` 下的个人资料）。
2. 点击右下角悬浮的"student"徽章，弹出"Testing as"面板。
3. 点击列表里的 `teacher · teacher` 一行。
4. 观察结果：页面跳转到 `Bei STOA anmelden`（登录）页面，E-Mail/Passwort 均为空，之前的登录状态消失。
5. 点击浏览器后退按钮，或直接访问 `https://app.stoaedu.ch/chat`，均仍停留在登录页，无法恢复原会话。

**预期行为**：点击切换到 teacher 角色，应该在保留会话的前提下切换到 teacher 对应的视角/权限，而不是登出整个应用。

**影响**：如果这个"Testing as"面板只出现在测试/预发环境，影响有限；但只要它在生产环境对真实用户可见（哪怕只对拥有多重角色的账号可见，例如同时是学生和家长/老师的账号），就会导致用户在使用中途被强制登出、丢失当前会话上下文，属于阻断性问题，建议优先修复或确认其可见范围。

---

## BUG-02（高）文本框留空也能创建对话，产生大量空/重复记录

**现象**：首页"New conversation"卡片的输入框（placeholder："Type your learning question here..."）不填写任何内容，直接点击"Start conversation"按钮，没有任何校验提示（既不置灰按钮，也不弹出"请输入问题"之类的提示），而是会成功创建一条新对话，标题形如 `math – Grade 8`（内容为空，助手没有任何实际回复，只是打开了一个空的聊天窗口）。

**复现步骤**：
1. 进入首页 `/chat`，确认"Subject for this question"选中"Mathematics"（默认选项）。
2. 不在文本框中输入任何文字。
3. 点击"Start conversation"按钮。
4. 观察结果：页面跳转到一个新对话页，标题为"math – Grade 8"（注意是全小写"math"，见 BUG-04），对话内容为空，没有任何用户消息或助手回复。
5. 返回首页下滑到"Conversations"列表，可以看到该空对话已经被记录进历史（日期显示为当天，如"Sep 16"）。

**佐证（历史数据）**：查看 Demo Student 的历史对话列表（`/learn/progress` → Fragen 区块，或 `/chat` 下方 Conversations 列表），发现存在大量标题完全相同、时间戳间隔仅几分钟到十几分钟的重复条目，例如：
- "Mathematics – 9" 在 2026/8/27 当天出现 5 次，时间分别为 12:20:50、12:33:36、12:41:10、12:43:38、12:51:51
- "Wie loese ich 2x + 3 = 11?" 在 2026/8/26 一天内出现 4 次（14:33:27、14:40:32、14:41:42、14:43:23），在 8/27 又出现多次

这些重复记录的产生模式（同一标题、短时间内反复出现）与"空输入也能创建对话"这个漏洞高度吻合，推测是用户（或测试脚本）反复点击"Start conversation"时被无意中多次触发所致。

**预期行为**：输入框为空时应禁用提交按钮，或点击后给出明确的"请输入你的问题"提示，不应该创建空对话记录；同时建议增加去重逻辑，避免短时间内重复的相同问题被记为多条独立历史。

---

## BUG-03（高）AI 回复语言与账号设置不一致，且出现中英/德英混杂

**现象**：Profile 页明确显示"ANTWORTSPRACHE（回复语言）：Deutsch"，但实测发现回复语言并不稳定：
- 用德语提问数学题"Wie loese ich 2x + 3 = 11?"时，助手全程用德语正确作答。
- 用德语提问物理题"Was ist die Beschleunigung?"（什么是加速度）时，助手却整段用英语作答，并且英文回答里混入了一个未翻译的德语提示词"**Hinweis:**"（应为英文"Hint:"），形成了英德混杂的答案。

**复现步骤**：
1. 确认 Profile 页"ANTWORTSPRACHE"字段为"Deutsch"。
2. 回到首页 `/chat`，在"Subject for this question"选择"Physics"。
3. 在输入框中输入德语问题："Was ist die Beschleunigung?"，点击"Start conversation"。
4. 等待助手回复，观察语言：回复主体为英文（"Step 1: Let's think about..."），但结尾提示语固定写成"**Hinweis:** Think if velocity doesn't change, what is the acceleration?"，"Hinweis"一词未被翻译成英文。
5. 对比：用同样是德语提问的数学题（历史记录"Wie loese ich 2x + 3 = 11?"），助手回复全程为德语，行为不一致。

**预期行为**：AI 回复语言应始终遵循账号"Antwortsprache"设置或跟随用户提问所用语言，不应该出现"某学科用错误语言回答"或"一句话里两种语言混用"的情况。

---

## BUG-04（中，出现频率高）AI 回复中的 Markdown 加粗语法未被渲染

**现象**：几乎每一条 AI 回复中，用双星号标记的强调文字都原样显示成带 `**` 符号的纯文本，没有被解析成粗体，例如：
- "**Hinweis:** Denke daran: Was du auf einer Seite machst..."
- "**Hinweis:** Think if velocity doesn't change..."
- "acceleration can also be **negative** — that means the object is slowing down"
- "In physics, **acceleration** tells us how quickly the velocity... **changes over time**"
- "The **unit** of acceleration is meters per second squared"

与之形成对比的是：同一条回复里的数学公式（LaTeX/KaTeX，如 `a = Δv/Δt`）渲染完全正常、清晰美观，说明问题只出在 Markdown 的加粗语法解析上，公式渲染管线是正常的。

**复现步骤**：
1. 打开任意一条包含"提示/Hinweis"或强调重点的历史对话（例如"Wie loese ich 2x + 3 = 11?"），或新提一个数学/物理问题。
2. 阅读助手回复的最后一段"提示"文字，或正文中被强调的术语。
3. 观察结果：本应加粗显示的词前后会带有两个星号 `**词语**`，星号本身也被显示出来，不是加粗效果。

**预期行为**：`**文字**` 应被渲染为 **粗体**，不应显示原始星号。

---

## BUG-05（中）切换界面语言后，部分动态内容不跟随翻译，造成语言混杂

**现象**：将右上角语言切换器从 EN 切到 DE 后，导航栏、按钮、页面大标题等静态文案都正确地变成了德语（如"Startseite""Übungen""Lernpfad""Wiederholen""Fortschritt""Übung starten"），但以下内容始终保持英文，未随界面语言切换：
- 学科/主题名称："mathematics"、"Brueche"（应为"Brüche"）、"Math"、"Physics"
- 优先级标签："High" / "Low"
- 多处卡片描述文字，例如"Recent learning evidence points to this topic as the next reviewed remediation area."、"A parent account is linked and can follow this student's progress."、"Access, family binding, and latest profile update."
- 账户状态板块的字段值："PLAN"标题下的"Family plan"、"STATUS"标题下的"Active"
- 历史记录卡片标题："Question asked"、"Teacher help requested"

**复现步骤**：
1. 打开任意页面（建议用 `/learn/progress` 或 `/profile`），点击右上角"EN"下拉，选择"DE"。
2. 观察页面：确认导航栏文字、区块标题（如"Heute empfohlen""Schwache Themen""Lernverlauf"）已变成德语。
3. 继续观察"Recommended for Today / Heute empfohlen"卡片内部：科目名"Brueche"、"Math"、优先级"High"/"Low"、说明文字"Recent learning evidence points to..."仍为英文。
4. 前往 Profile 页，观察"HAUPTFÄCHER"字段值为"mathematics"（英文小写），"Plan"区块的"Family plan""Active"均为英文，页面副标题"Account, family, billing, and learning context used to keep STOA support accurate."也未翻译。
5. 前往历史记录列表（Fragen 区块），确认每条记录的类型标签"Question asked"仍为英文。

**预期行为**：切换到德语后，页面上所有面向用户的文案（包括后台返回的学科名、状态值、说明文字、记录类型标签）都应该跟随界面语言一起显示为德语，不应该出现德英混排。

---

## BUG-06（低）学科名称大小写不统一

**现象**：首页"Subject for this question"选择区展示的是首字母大写的"Mathematics / Physics / German / English"，但通过这些入口新建的对话，标题却使用全小写形式，例如"math – Grade 8""physics – Grade 8"；而更早期的对话标题又是首字母大写的"Mathematics – Grade 8""Mathematics – 9"。同一个概念在不同时间/不同入口生成的展示形式不一致。

**复现步骤**：
1. 在首页选择"Physics"学科，输入任意问题并提交。
2. 观察新对话页顶部标题：显示为"physics – Grade 8"（小写）。
3. 对比历史记录里较早的对话，如"Mathematics – Grade 8"（大写开头）。

**预期行为**：同一学科在任何位置的展示名称大小写应保持一致，建议统一在展示层做首字母大写处理，与后端存储的原始值解耦。

---

## BUG-07（低）通知面板点击外部区域无法关闭

**现象**：点击顶部导航栏的铃铛图标会弹出"Mitteilungen（通知）"面板，面板内容为"Sucht nach Ne...（文字被截断）"和"0 ungelesen（0条未读）"、"Noch keine Mitteilungen.（暂无通知）"。打开后，无论点击页面上面板以外的哪个区域，面板都不会自动关闭，唯一能关闭它的方式是再点一次铃铛图标本身。

**复现步骤**：
1. 点击右上角铃铛图标，弹出通知面板。
2. 点击面板范围以外的任意空白处（例如页面中间的卡片区域）。
3. 观察结果：面板依然保持展开状态，没有关闭。
4. 再次点击铃铛图标，面板才会关闭。

**附带发现**：面板中的状态文字"Sucht nach Neuem"（正在搜索新内容）在气泡内被截断显示为"Sucht nach Ne..."，说明该气泡的固定宽度没有考虑到德语文本通常比英语更长，存在文本溢出/裁切的风险。

**预期行为**：点击弹层以外的任意区域应能自动关闭该弹层，这是下拉/弹出组件的通用交互预期；同时状态提示文字容器宽度应根据内容自适应或允许换行，避免被截断。

---

## BUG-08（低）悬浮的角色徽章会遮挡按钮和文字内容

**现象**：右下角固定悬浮的"student"徽章（即 BUG-01 中提到的"Testing as"入口）没有做避让处理，会遮挡多处正常界面元素，包括：
- 首页/Learn 页"Continue Practice"卡片的"Resume"续学按钮
- Progress 页第二个推荐练习卡片的"Start practice"按钮下半部分
- 聊天页"Would a tutor help clarify this?"卡片里"Ask a Tutor"按钮的文字说明
- 通知面板展开时与"0 ungelesen"标签部分重叠

**复现步骤**：
1. 进入 `/learn`（Exercises 标签），下滑页面到"Continue Practice"卡片附近的"Resume"按钮位置。
2. 观察：右下角悬浮的"student"徽章与"Resume"按钮出现视觉重叠。
3. 类似地，在任意聊天对话页面等待助手回复后出现的"Ask a Tutor"提示卡片处也能看到同样的重叠。

**预期行为**：悬浮元素应具备避让逻辑（例如在滚动到关键交互按钮附近时收起或位移），不应遮挡任何可点击元素或关键文字信息。

---

## BUG-09（低，交互反馈问题，非阻断）学科选中状态视觉反馈弱，与"Active"标签语义冲突

**现象**：在首页"Subject for this question"区域点击"Physics/German/English"后，实际提交是按选中的学科生效的（功能正确），但视觉上的选中反馈非常弱，仅表现为文字略微加粗和背景色轻微变化；而"Mathematics"卡片上常驻的红色"Active"标签并不会因为切换选择而消失或转移到新选中的学科上。"Active"字面容易被理解为"当前选中项"，但实际语义似乎是"当前主修/默认学科"，两者用了同一套视觉表达，容易造成用户误解——尤其是不确定自己刚才点选的学科是否真的生效。

**复现步骤**：
1. 进入首页，默认"Mathematics"带红色"Active"标签。
2. 点击"Physics"卡片。
3. 观察：Physics 卡片文字变为加粗、底色略深，但 Mathematics 卡片上的"Active"标签没有任何变化。
4. 输入问题并提交后，实际生成的对话标题确认是"physics – ..."，说明选择确实生效，只是界面提示不够明确。

**预期行为**：建议为"当前选中学科"设计更明显的选中态（如描边高亮、勾选图标），并将"Active"标签的文案改为更明确地表达"主修/默认学科"含义的措辞，避免与"当前选中"混淆。

---

## BUG-10（低）登录页缺少"忘记密码"入口

**现象**：登录页（`/login`）只提供"E-Mail""Passwort""Einloggen（登录）"和"Noch kein Konto? Registrieren（还没有账号？注册）"，没有找到任何"忘记密码/重置密码"相关的链接或按钮。

**复现步骤**：
1. 访问 `https://app.stoaedu.ch/login`。
2. 完整浏览登录表单区域及下方内容。
3. 未发现"忘记密码"或类似入口。

**预期行为**：如果产品确实没有设计找回密码流程，用户一旦忘记密码将无法自助恢复账号访问，建议补充该入口（不确定是否为遗漏，也可能入口位于本次未能访问到的其他位置，建议产品侧确认）。

---

## BUG-11（高）隐私政策与使用条款页面，德语变音字符系统性丢失（ä/ö/ü/ß 全部被转写成 ae/oe/ue/ss）

**现象**：`/privacy`（Datenschutzerklärung，隐私政策）和 `/terms`（Nutzungsbedingungen，使用条款）这两个法律文本页面，整篇正文里所有德语变音字符都被替换成了对应的 ASCII 转写形式，即 `ä→ae`、`ö→oe`、`ü→ue`、`ß→ss`，覆盖全文、没有例外。例如：

- 标题本身："Datenschutzerklaerung"（应为 Datenschutzerklärung）
- "Lerndaten von Schuelerinnen und Schuelern"（应为 Schülerinnen und Schülern）
- "Lerndaten koennen ... umfassen, die den Schueler unterstuetzen"（应为 können / Schüler / unterstützen）
- "Sichtbarkeit fuer Eltern und Lehrpersonen"（应为 für）
- "Nutzung, Aufbewahrung und Loeschung"（应为 Löschung）
- "Zuverlaessigkeit zu ueberwachen"、"gemaess der operativen STOA-Richtlinie"（应为 Zuverlässigkeit / überwachen / gemäß）
- `/terms` 页同样通篇如此："Lernunterstuetzung"、"Erklaerungen"、"Uebung"、"Pruefungsergebnisse"、"Zulaessige Nutzung"、"duerfen"、"beschraenkt"、"Kuendigungen, Rueckerstattungen"、"Aenderungen des Dienstes"、"Supportqualitaet" 等（正确应为 Lernunterstützung / Erklärungen / Übung / Prüfungsergebnisse / Zulässige / dürfen / beschränkt / Kündigungen, Rückerstattungen / Änderungen / Supportqualität）

对比之下，产品内其他德语界面文案（导航栏、按钮、标题等，如"Übungen""Wählen Sie Ihren""Schülerprofil"）的变音字符都是正常的，只有这两个法律文本页面存在系统性丢失，推测是这两页内容单独走了某个文本处理/翻译/生成流程，该流程的输出编码或转写逻辑有问题。

**复现步骤**：
1. 直接访问 `https://app.stoaedu.ch/privacy`，通读正文，可见标题及每一段落都存在 ae/oe/ue/ss 替代变音字符的情况。
2. 访问 `https://app.stoaedu.ch/terms`，同样通读正文，问题同样存在且更密集。
3. 对比同一时间界面语言为 DE 时的其他页面（如 Profile、Progress），确认那些页面的变音字符是正常显示的，从而排除是浏览器/字体渲染问题。

**预期行为**：法律文本作为对用户最正式、最需要专业感和可信度的内容，理应与产品其余部分一样正确显示德语变音字符，不应出现拼写错误观感的转写文本。

**影响**：这是本轮测试中除 BUG-01（登出）之外最值得优先修复的问题之一——面向瑞士德语区用户的隐私政策和使用条款，如果通篇存在明显拼写"错误"，会直接影响用户对平台专业度和可信度的观感，也可能引发对法律文本准确性的怀疑。

---

## BUG-12（高）帮助中心（/support）与 Pilot 引导页（/onboarding）完全没有德语翻译，且暴露了面向内部/试点用户的运营文案

**现象**：将界面语言切换为 DE 后，`/support`（Hilfe und Rückmeldung）页面的大标题是德语，但页面主体内容——FAQ 分类标签（"FAQ""Bug feedback""Teacher-help distinction""Contact path"）以及全部 FAQ 正文、"Bug feedback""Teacher help vs support""Pilot expectations"等板块——全部是英文，完全没有跟随语言切换。`/onboarding`（Get ready for the STOA pilot）页面更彻底，从大标题、按钮到全部正文均为纯英文，语言切换器选了 DE 也不生效。

更值得注意的是，这两页的文案内容读起来像是写给"内部试点管理者/客服"看的运营说明，而不是面向学生/家长的用户文案，例如：
- "The pilot prioritizes reliability, clear learning support, and fast issue discovery over complete feature breadth."（试点优先保证可靠性和问题发现速度，而非完整功能覆盖）
- "Some flows may use lightweight operations while the team validates demand and support volume."（部分流程会先用简化实现，等团队验证需求量）
- "Do not paste passwords, tokens, full private chat transcripts, or file contents into support messages."（这条对最终用户其实是有用的安全提示，但语气和上下文明显是内部培训材料风格）
- "Pilot feedback is reviewed by the STOA team and used to decide what should be fixed, documented, or expanded before a broader launch."

这些内容目前未登录即可直接访问（`/support` 和 `/onboarding` 都不需要登录），任何访客都能看到"这是一个小范围试点、部分功能是简化实现"等内部运营信息。

**复现步骤**：
1. 在任意页面把语言切换为 DE。
2. 直接访问 `https://app.stoaedu.ch/support`，查看页面除大标题外的所有正文内容，确认为英文。
3. 在 `/support` 页面点击右上角"Kontoeinrichtung ansehen"按钮，跳转到 `https://app.stoaedu.ch/onboarding`，确认整页无论标题还是正文都是纯英文，且未登录也可直接访问。

**预期行为**：建议至少把面向真实用户（学生/家长/老师）的部分补充德语翻译；同时建议重新审视这两页的文案定位——如果内容本意是写给内部试点团队或客服参考的操作说明，不建议直接以这种口吻呈现给外部访客，可以拆分成"面向用户的帮助文案"和"内部试点运营文档"两套内容。

---

## BUG-13（高）帮助中心"Kontakt"按钮是空的 mailto 链接，点击后会跳离应用、打开完全空白的邮件撰写页面

**现象**：`/support` 页面顶部和底部都有"Kontakt"按钮，点击后浏览器**当前标签页**直接跳转到用户自己的 Gmail 网页版撰写新邮件界面（"Compose Mail"），而且"收件人（Recipients）"和"主题（Subject）"两个字段都是**完全空白**的——也就是说这个按钮背后的链接是一个没有指定收件地址（`mailto:` 缺少邮箱地址）的空 mailto 链接。

**复现步骤**：
1. 访问 `https://app.stoaedu.ch/support`。
2. 点击页面顶部的"Kontakt"按钮（页面底部"STOA kontaktieren"板块里也有一个功能相同的"Kontakt"按钮）。
3. 观察结果：当前标签页离开 STOA 应用，跳转到 Gmail 的"新邮件"撰写页面，收件人和主题均为空，用户完全不知道应该填写谁的邮箱地址，"联系我们"这个功能实际上没有传达任何有效信息。

**预期行为**：点击"Kontakt"应该打开一个带有正确收件地址（例如 support@stoaedu.ch 或类似地址）、理想情况下带有预填主题的邮件草稿，或者跳转到一个真正的在线联系表单；无论哪种方式，都不应该在当前标签页内直接跳转离开应用去往一个完全空白、不知道该发给谁的邮件撰写页面。另外，即便修复了收件地址问题，也建议改成新开标签页而不是替换当前页面，避免用户"联系我们"点一下就把正在使用的 STOA 页面弄丢了。

---

## BUG-14（中）注册向导（Registrieren）第 2 步：必填字段校验只提示一条通用错误，不逐项标注

**现象**：注册流程"Schritt 2 von 4"页面包含 Name、E-Mail、Passwort 三个必填输入框和一个"我同意隐私政策和条款"的必选复选框。全部留空点击"Weiter"后，页面只在复选框下方出现了一条通用提示"Dieses Feld ist erforderlich."（此字段为必填项，用的是单数"字段"），Name、E-Mail、Passwort 三个输入框本身没有任何红框、图标或各自的提示文字，用户光看界面完全无法判断到底是哪个字段没填对，只能靠猜测或者依次尝试。

**复现步骤**：
1. 访问登录页，点击"Registrieren"进入注册向导，选择任意账号类型点击"Weiter"进入 Schritt 2。
2. 不填写 Name / E-Mail / Passwort 任何一项，也不勾选同意条款复选框，直接点击"Weiter"。
3. 观察：页面停留在原地，仅复选框下方出现一条"Dieses Feld ist erforderlich."提示；Name/E-Mail/Passwort 三个输入框均无任何视觉上的错误提示。
4. 勾选复选框后再次点击"Weiter"，提示消失但依然停留在当前页（说明 Name/E-Mail/Passwort 校验在生效拦截提交），但界面上从始至终没有对这三个字段给出任何单独的错误反馈。

**预期行为**：建议为每个必填字段单独显示校验状态（红色边框 + 该字段下方对应的提示文字），并在提示文案中标明具体是哪个字段的问题，而不是一条通用语句让用户自行排查。

---

## BUG-15（低）注册向导各步骤共用同一个大标题/说明文字，未随步骤内容更新

**现象**：注册向导 Schritt 1（选择账号类型）的页面大标题是"Wählen Sie Ihren STOA-Kontotyp."（选择您的 STOA 账号类型），副标题是"Wir passen die Lernumgebung an Schülerinnen und Schüler, Eltern und Lehrpersonen an."。进入 Schritt 2（填写 Name/E-Mail/Passwort）后，这个大标题和副标题完全没有变化，仍然显示"选择您的账号类型"，与当前页面实际展示的表单内容（姓名/邮箱/密码）不匹配。

**复现步骤**：
1. 进入注册向导 Schritt 1，记录大标题"Wählen Sie Ihren STOA-Kontotyp."。
2. 选择任意账号类型，点击"Weiter"进入 Schritt 2。
3. 观察：大标题和副标题文字与 Schritt 1 完全相同，只有下方"Schritt 2 von 4"字样和表单内容变了。

**预期行为**：建议每一步都配上与当前步骤内容相符的标题/说明文字（例如 Schritt 2 可以是"创建您的登录信息"之类），帮助用户理解自己当前处于流程的哪个阶段、需要做什么。

---

## BUG-16（低）注册向导"已注册？"链接没有可点击的视觉样式

**现象**：注册向导 Schritt 1 底部有一行"Bereits registriert?"文字，检查页面结构确认它确实是一个指向 `/login` 的超链接，功能正常，但视觉上它就是一段普通黑色文字，没有下划线、没有品牌色高亮、字重也和登录页"Registrieren"链接（加粗深色）不一致，用户很容易把它当成纯文本而忽略，不知道可以点击它返回登录。

**复现步骤**：
1. 进入注册向导 Schritt 1，下滑到底部。
2. 观察"Bereits registriert?"文字样式：与周围说明文字几乎没有区别。
3. 用元素查找确认它其实是 `<a href="/login">`，点击后能正常跳转到登录页。

**预期行为**：建议给这类"跳转到另一流程"的链接文字加上下划线、品牌色或加粗等视觉样式，与登录页"Registrieren"的呈现方式保持一致，让用户能一眼识别出这是可点击的链接。

---

## 附注：浏览器原生表单校验提示不跟随页面语言（供参考，非产品直接可控）

在登录页把 E-Mail 或 Passwort 留空直接点击"Einloggen"，会触发浏览器原生的"required"校验提示（如"Please fill out this field."/"Please include an '@' in the email address."），这类提示语言跟随的是浏览器/系统语言设置，而不是页面当前选择的 DE。这不是应用代码可以直接控制的部分（除非改为完全自定义的 JS 校验并接管默认的 HTML5 校验行为），但如果希望在校验提示上也做到完全德语化的一致体验，可以考虑用自定义校验替换原生 `required` 属性的提示。

---

## 附注：页脚"Über STOA"与"Zur STOA Website"两个链接功能完全重复

登录页/公开页脚部里，"PLATTFORM"分组下的"Über STOA"和底部版权栏的"Zur STOA Website"两个链接指向完全相同的地址（`https://stoaedu.ch`），文案不同但目的地相同，属于轻微的信息重复，可以考虑合并或让其中一个指向更具体的子页面（如"关于我们"介绍页）。

---

## 附：本次测试中确认正常、值得肯定的部分

- 界面多语言切换（EN/DE 已验证，另有 FR/IT 选项未逐一验证）覆盖面广，绝大多数静态文案翻译到位。
- 数学公式（LaTeX/KaTeX）渲染清晰美观，未发现渲染错误。
- AI 能正确识别提问是否超出当前年级/学科范围，并礼貌引导用户回到课程范围内提问（例如物理对话中拒绝回答数论问题"Was ist eine Primzahl?"）。
- 追问快捷按钮（"I don't understand one of the steps""Can you explain it more simply?"）功能正常，点击后能正确发出对应追问并获得新回复。
- 注册向导的必填字段+复选框校验在后端逻辑上是可靠的：留空或未勾选条款时始终会拦截提交，不会让用户跳过必填项（只是前端提示不够清晰，见 BUG-14）。
- 登录页对空字段、非法邮箱格式的原生校验能正确拦截提交（虽然提示语言跟随浏览器而非页面语言，见附注）。
- 两轮测试全程未在浏览器控制台发现任何 JavaScript 报错，网络请求也未发现失败（4xx/5xx）的接口调用，说明后端接口层比较稳定，本次发现的问题集中在前端渲染、内容本地化与交互细节层面。

---

*本清单基于两轮人工黑盒测试整理。第一轮测试因 BUG-01（角色切换导致强制登出）提前终止；第二轮在无法重新登录的情况下，把测试范围转向登录/注册、公开页面、帮助中心与法律文本页面，又发现 BUG-11 ~ BUG-16。仍未覆盖的功能点（学生端"继续阅读""复习"流程、真实的"Ask a Tutor"转人工、"Report answer"举报流程、移动端真实响应式布局）需要有效登录凭证或不同的测试环境才能继续验证，建议后续补充测试后再合并进正式缺陷跟踪系统。*
