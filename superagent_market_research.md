# Super-agent 类产品调研报告（2026-11）

> 调研时间：2026-11-03 ｜ 参照产品：DeepSeek Super-agent（开源原型 `dsh-superagent`）｜ 信息口径：公开资料，价格与功能以各厂商当期页面为准

## 0. 结论先行

1. **在本次调研范围内，没有完全同构的竞品。** 主流产品是"派单式"异步编码 Agent：人指定一个任务，Agent 在云端沙箱或 CI 里执行，最后提一个 PR。super-agent 的差异是"常驻 + 多源事件驱动 + 团队级工单编排"。
2. **最接近的是 Devin、Factory、GitHub Copilot coding agent 和 OpenAI Codex；国内形态最接近的是阿里 QoderWake。** 它们已经覆盖"异步委托 → 自主执行 → 提 PR"或"常驻数字员工"的主链路，但入口仍是人明确派单/人配规则，缺少长期在线的策略门控与多工单生命周期管理。
3. **GitHub Agent HQ 是控制平面，不是执行器。** 它把多家 Agent 收进统一的 GitHub 任务入口，验证了 super-agent 路由层的价值，同时也是最大的平台级威胁。
4. **super-agent 的差异化在治理与上下文，不在模型能力。** Ticket / Mandate / Policy / Subscription 这套抽象，加上"只在需要人时说话"的注意力预算，是通用编码 Agent 没有的。
5. **短板同样明显。** 研究原型、单仓库、无企业身份/审计/计费、以高权限运行，离企业采购还有明显距离。

## 1. 调研范围与方法

### 1.1 什么算"相近"

"相近"不按界面或模型口径判断，而按工作方式。本文采用七条判定标准，满足四条以上即纳入：

| # | 判定标准 | 说明 |
| --- | --- | --- |
| 1 | 以工作单元为中心 | 接收自然语言任务，而不是做代码补全或单轮问答 |
| 2 | 多步自治 | 能自主规划、改代码、跑测试、根据结果修正 |
| 3 | 真实工程环境 | 在真实仓库、沙箱或 CI 中操作，产出可评审的代码变更 |
| 4 | 异步无人值守 | 不需要人在会话里持续回车，可离开后回来收结果 |
| 5 | 事件驱动或常驻 | 由 IM、Issue、PR 评论、CI、定时任务触发，能 7×24 在线 |
| 6 | 团队级 | 支持多用户、多任务并行、权限与策略、通知与追责 |
| 7 | 交付闭环 | 从需求到 PR，再到评审回复、修复、合并或交接 |

按与 super-agent 的距离，把市场分成四层：

| 层级 | 特征 | 代表产品 | 与 super-agent 的关系 |
| --- | --- | --- | --- |
| A. 常驻/主动式 Agent | 长期在线，监听事件，自己决定是否开工，管理多个工单 | super-agent、Devin、Factory | 同构，直接竞品 |
| B. 异步委托式编码 Agent | 人明确派单，云端异步执行，交付 PR | OpenAI Codex、Claude Code（web/云端）、Google Jules、Copilot coding agent、Cursor Cloud Agents、OpenHands | 近邻，能力底座最接近 |
| C. 交互式编码 Agent | 在 IDE/CLI 里与人结对，实时协作 | Claude Code、Codex CLI、Cursor、Amp、通义灵码、Trae | 上游能力供给，不是同一交付方式 |
| D. 通用任务 Agent | 跨软件完成通用任务，不专攻代码仓库 | Manus、Genspark、扣子空间 | 相邻品类，争夺"AI 员工"预算 |

### 1.2 方法

- 桌面研究：官方文档、定价页、产品博客优先，媒体与第三方评测交叉验证；访问时间统一为 2026-11-03。
- 不做实测：本文比较产品形态与交付方式，不比较模型绝对能力；不同厂商的基准分数口径差异大，不直接横比。
- 价格只作量级参考：多数产品按用量计费，实际账单取决于任务复杂度、模型和并发。
- 已知局限：部分企业版只提供销售报价；中国市场的信息披露不充分；super-agent 自身只有合成负载实验，没有生产客户数据。

## 2. 参照物：super-agent 的设计选择

### 2.1 一句话定义

super-agent（开源实现 `dsh-superagent`）是一个常驻在单个 GitHub 仓库上的主动式编码 Agent，可选接入飞书。允许的人提出需求后，它评估是否接单，创建工单，在独立 Session 和目录里执行，最后回到原会话交付，例如提一个 PR，并在 PR 线程里继续响应评审。

### 2.2 八个关键设计（也是后文的比较维度）

| 设计选择 | 具体机制 | 为什么重要 |
| --- | --- | --- |
| 事件源抽象 | Source → Trigger → Thread；GitHub 与飞书都翻译成统一事件 | 换渠道不改核心逻辑；同一件事跨渠道可关联 |
| 两层评估 | 程序层 Gate 先判"谁可以提、强度多大"，模型层 Assessment 再判"要不要接" | 授权与语义判断分离；默认 `observe` 只记录不执行 |
| 工单化 | Ticket 承载 Mandate（策略版本、范围、强度）、Assignment、Subscription | 工作可授权、可审计、可恢复、可交接；不等同于一个 Session |
| 双层执行 | 常驻 Router 只分派与回复；每个 Ticket 一个 Worker Session | 上下文隔离，多任务并行；Router 可用轻量模型 |
| 订阅直通 | PR、Issue 等线程的后续事件直接送给对应 Worker，不绕 Router | 省 Token、降延迟，且多个工单能看到彼此动作 |
| 上下文缓存化 | 高频信息内联，长内容截断留续读入口，对象间保留跳转 ID | 控制上下文成本，支持跨会话追溯 |
| 注意力预算 | 确认一次、工作期间安静、只在有结果或需决策时说话；结构化提问 | 把"打扰人"当成成本，而不是把消息量当产出 |
| 持久化与恢复 | 状态落库、存算分离、外部 Watchdog 巡检 | 进程重启、任务卡死、模型空转时可恢复 |

### 2.3 公开数据与边界

- 仓库：`deepseek-ai/dsh-superagent-prototype`，MIT，2026-10-31 开源；截至 2026-11-03 约 14,800 stars、1,580 forks。
- 官方实验（合成负载，约 300 条消息、20 条工作请求）：任务完成时间 p50 12.4 分钟、p95 15.7 分钟；单 Session + Subagent 方案在中性提示下 p95 为 29.5 分钟。数据来自团队自述，仅作初步参考。
- 定位：研究原型，明确不是 DeepSeek 正式产品；代码大量由编码 Agent 完成，不建议直接上生产。
- 能力边界：单仓库、仅 GitHub.com（不支持 GitHub Enterprise Server）、可选飞书、本地进程运行、单租户；无 SSO、审计、计费、配额；Worker 与 Router 以 `danger-full-access` 运行。

## 3. 市场全景：三层结构，三条路线

### 3.1 三层结构

2026 年的"AI 编码 Agent"已经不是一个单层市场，而是分成三层。理解 super-agent 的位置，先要看清每一层卖的是什么：

| 层级 | 卖什么 | 代表产品 | 竞争焦点 |
| --- | --- | --- | --- |
| 执行层：编码 Agent 运行时 | 模型 + 工具 + 代码库理解，把任务做出来 | Claude Code、Codex、Cursor、Amp、OpenHands、Trae、Qoder CN、CodeBuddy | 代码质量、工具生态、上下文长度、模型能力 |
| 委托层：异步任务外包 | 人派单，云端/沙箱执行，交付 PR | Codex Cloud、Claude Code 云会话、Copilot coding agent、Devin、Factory、Cursor Cloud Agents、Augment | 并行度、无人值守时长、PR 可评审性、价格 |
| 控制层：团队级调度与治理 | 事件入口、工单、路由、策略、审计、注意力管理 | super-agent、GitHub Agent HQ、Devin/Factory 的 Mission Control、Claude Projects/Tag | 多 Agent 编排、权限与审计、跨渠道上下文、成本治理 |

super-agent 的特殊之处，是它在**一个开源包里同时占了三层**：用 DSH Session 当执行层，用 Ticket/Assignment 当委托层，用 Source/Policy/Eval 当控制层。代价是每一层都不如专做一层的产品深；收益是它是少数能把"事件 → 工单 → 执行 → 评审 → 回复"完整闭环跑在自有环境里的方案。

### 3.2 三条产品路线

1. **把会话做长**：Claude Code、Codex、Cursor、Amp 等。核心价值是让一个开发者在一个会话里完成更多工作，人在环内，信任建立快，但吞吐受人的注意力限制。
2. **把任务外包**：Devin、Factory、Copilot coding agent、Claude 云会话、Codex Cloud 等。核心价值是"派单后离开"，用云端并行换吞吐，按任务/用量计费。
3. **把团队接进来**：super-agent、GitHub Agent HQ 等。核心价值不是替一个人写代码，而是把群聊、Issue、PR、CI 的事件流接进一个统一的工单系统，让 Agent 在团队层面工作，并控制它对人注意力的消耗。

### 3.3 2026 年的三个变化

**变化一：主动性正在从差异化变成标配，但"可调度的自动化"不等于"常驻判断"。**

- Anthropic 的 Claude Code 推出云端的 [scheduled tasks（原 routines）](https://code.claude.com/docs/en/routines)，支持定时、API 调用和 GitHub 事件触发；
- GitHub Copilot 的 [Automations](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/create-automations) 支持定时、仓库事件和 API，能自动开 PR；
- Cursor 的 [Cloud Agent Automations](https://cursor.com/docs/cloud-agent/automations) 支持 cron、GitHub/GitLab/Bitbucket、Slack、Linear、webhook 等触发器；
- Google Jules 甚至上线过"Proactivity"（主动扫描仓库、建议任务），但只做到"建议"，仍要人确认；
- OpenAI 的 [dots](https://learn.chatgpt.com/docs/dots) 被官方定义为 always-on agent，有云端电脑和浏览器，会主动找人。

这些能力说明"不用被点名也能跑"已经不难。难的是**判断什么值得做、做之前有没有授权、做完之后如何交付和追责**。多数产品的主动性仍停在"人先配一条规则/一个定时任务"，而不是"常驻监听、按策略判断、自己开工单"。这恰好是 super-agent 的 Eval/Gate/Mandate 设计要解决的问题。

**变化二：控制平面成为大厂的必争之地。**

GitHub 在 2025-10 发布 [Agent HQ](https://github.blog/news-insights/company-news/welcome-home-agents/)，把 Copilot、Claude、Codex、Jules、Devin 等收进同一个"任务控制台"；2026-02，Claude 和 Codex 正式进入 Agent HQ，GitHub 明确把自己定位成"分配、驾驶、观察和治理 Agent 的驾驶舱"（[IT for Business，2026-02-09](https://www.itforbusiness.fr/claude-et-codex-debarquent-dans-agent-hq-github-multiplie-les-agents-100149)）。Anthropic 则用 Claude Tag 提供组织级共享身份和 admin 管控，用 Projects 协调多个并行会话。竞争焦点从"谁的 Agent 更会写代码"上移到"谁掌握任务入口和治理权"。

**变化三：独立的"异步编码 Agent"层正在被收编。**

- Google Jules 官方公告 2026-12-31 关停，2026-11-11 后转只读，能力并入 Antigravity（[Jules changelog](https://jules.google/docs/changelog)）；
- Codegen 被 ClickUp 收购，独立服务 2026-01-09 停止（[Codegen 公告](https://codegen.com/an-update-on-codegen/)）；
- Sweep 的 issue-to-PR 产品退场，仓库改为指向 JetBrains 插件（[Sweep README](https://github.com/sweepai/sweep)）；
- Cursor 的订阅改为由 X Subscription 统一售卖，Cloud Agents 与 Automations 需要 Premium 及以上档位（[Cursor 定价](https://cursor.com/pricing)）。

被收编的原因不难理解：单点 Agent 的模型能力可以外购，工具链可以复制，真正难的是分发、治理和企业信任。这对 super-agent 是个警告，也是个提示：**如果只做"更主动的编码 Agent"，很容易被平台吸收；如果做"团队级 Agent 控制平面"，反而有独立存在的空间。**

## 4. 代表产品逐个看

> 以下价格均为 2026-11-03 公开列表价，按美元/人民币月付口径，实际账单受用量、年付折扣和企业合同影响。

### 4.1 OpenAI Codex：最完整的平台化选手

Codex 已经从"云端编码 Agent"扩展成覆盖 Web、CLI、IDE、iOS、Slack、GitHub 的多端产品：每个云任务有独立 workspace，电脑关机后继续跑；支持 subagents、自动代码评审、GitHub issue/PR 委派和 Slack 对话式委派（[Codex Cloud 文档](https://learn.chatgpt.com/docs/cloud)）。

- **主动性**：两条线。一是 **scheduled tasks**，支持 Gmail、Slack 频道、GitHub PR 事件触发，可设 `approval_policy="never"` 无人值守（[Automations 文档](https://learn.chatgpt.com/docs/automations)）；二是 **dots**，官方定义为常驻云端的 always-on agent，有自己的电脑和浏览器，能主动推进并在需要时找人（[dots 文档](https://learn.chatgpt.com/docs/dots)）。但 dots 是通用 Agent，不是仓库级团队 Agent；事件触发也不覆盖 GitHub Issue。
- **交付**：diff、commit、PR；自动评审只报 P0/P1，刻意收敛噪音。
- **价格**：Free $0、Go $8、Plus $20、Pro $100/$200/$500、Business $20/人/月（年付）或 $25（月付）、Enterprise 面谈（[Codex 定价](https://learn.chatgpt.com/docs/pricing)）。
- **判断**：平台能力最全，分发最强。它的"主动性"是通用助手的主动性，不是"盯一个团队仓库、按策略接单"的主动性；对 super-agent 的威胁主要在模型和通用入口，而不是工单治理。

### 4.2 Anthropic Claude Code：开发者心智最强，企业控制补得最快

Claude Code 现在有终端、IDE、桌面、Web、Slack 五个入口，云会话可以在浏览器、手机、桌面、终端之间迁移；Claude Tag 提供组织级共享身份，Projects 负责协调多个并行会话，Agent SDK 允许自建 Agent（[Claude Code 概览](https://code.claude.com/docs/en/overview)、[云会话文档](https://code.claude.com/docs/en/claude-code-on-the-web)）。

- **主动性**：云端 scheduled tasks 支持定时、API、GitHub 事件三种触发器；在 Slack 里 @Claude 会直接创建一个云会话并把结果回到对话（[Slack 集成](https://code.claude.com/docs/en/slack)）。这已经非常接近"团队 IM 入口 + 异步执行"，但任务仍由人逐条发起，没有 Ticket 生命周期和策略评估层。
- **多 Agent**：subagents 已可用；agent teams 仍是 experimental（共享任务列表、mailbox、team lead），官方也提示顺序任务和同文件改动不适合用（[agent teams 文档](https://code.claude.com/docs/en/agent-teams)）。
- **价格**：Pro 年付 $17/月（月付 $20）、Max $100/$200、Team Standard $20/人/月（月付 $25）、Team Premium $100（月付 $125）、Enterprise $20/席位/月 + API 用量（[Claude 定价](https://claude.com/pricing)）。
- **判断**：执行层最强之一，企业治理（SSO、审计、数据保留、私有环境）补得最快。缺的是"允许谁在什么范围内主动开工"的策略层，以及工单化的多任务管理。

### 4.3 GitHub Copilot coding agent + Agent HQ：分发与治理的平台重力

把 Issue 分配给 Copilot，或用 Agents 标签页、`@copilot`、`/task` 发起任务；它在云 sandbox、GitHub Actions 或自建 remote environment 里执行，推分支、开 PR，完成后自动加 reviewer，但不能自己合并。Automations 支持定时、仓库事件、API 触发，可自动开 PR，但仅限私有/内部仓库（[coding agent 文档](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)、[Automations 文档](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/create-automations)）。

Agent HQ 把 Copilot、Claude、Codex 和第三方 agent apps 收进同一个 Agents 入口，第三方产物同样过 CodeQL/secret scanning（[第三方 Agent 文档](https://docs.github.com/en/copilot/concepts/agent-capabilities/about-agents/about-third-party-coding-agents)）。这意味着：**团队不需要选边，GitHub 直接做路由和治理层。**

- **价格**：Free、Pro $10（1,500 credits）、Pro+ $39（7,000）、Max $100（20,000）、Business $19/席位（1,900）、Enterprise $39（3,900）；2026-06-01 起 premium requests 改为按 token 计费的 AI credits（[Copilot 计划](https://docs.github.com/en/copilot/get-started/plans)、[CloudZero 分析](https://www.cloudzero.com/blog/github-copilot-cost/)）。
- **判断**：这是 super-agent 最强悍的对手——不是因为它更主动，而是因为它已经长在团队已有的 GitHub 工作流里，边际采用成本极低。super-agent 的差异化必须落在 GitHub 不会做的事上：跨 IM 的上下文、私有化、自定义策略，或者与 Agent HQ 互补而不是对抗。

### 4.4 Devin（Cognition）：目前最接近"数字工程师"的产品

Devin 在云端 VM 里规划、编码、测试、开 PR，产品线扩展到 Desktop、CLI、Devin Review、DeepWiki、Auto-Triage、Security Swarm。它支持从 Slack、Linear、Jira、Issue、CLI、API 发起任务，更重要的是有一套很宽的**事件自动化矩阵**：Slack 消息/表情、GitHub Issue/PR/CI、GitLab、Linear、Jira、PagerDuty、cron、webhook，动作可以是"开新会话、给已有会话发消息、持续盯一个频道"（[Devin Automations 文档](https://docs.devin.ai/product-guides/automations.md)）。官方博客称 Cognition 内部一周合并了 659 个 Devin PR，并用它做每日设计规范扫描、Linear 工单分诊、自动修复 CI（[Cognition 博客](https://cognition.com/blog/how-cognition-uses-devin-to-build-devin)）。

- **部署**：默认 Cognition 云；Desktop 本地；Outposts 把命令和文件操作放到客户 VM/K8s，推理仍在云上，属于混合而非完全自托管（[Outposts 文档](https://docs.devin.ai/cloud/outposts/overview.md)）。
- **价格**：Free、Pro $20/月、Max $200/月、Teams $80/月最低 + $40/全席位、Enterprise 定制；Automations 和 Review 消耗预付费 credits，额度耗尽即停（[Devin 自助计费](https://docs.devin.ai/admin/billing/self-serve.md)、[Devin 定价页](https://devin.ai/pricing)）。
- **体量**：2026-09 完成超过 20 亿美元融资、估值 480 亿美元；run-rate 从 5 月的 4.92 亿美元增长到 9 月的近 9 亿美元；客户包括 NVIDIA、GE Aerospace、Citi、Mercedes-Benz（[Cognition Series E](https://cognition.com/blog/series-e)）。
- **判断**：功能上最接近 super-agent 的"watch → ticket → PR"闭环，尤其是持续频道监听和多触发器。差距在开放性（闭源、无飞书、数据必须过 Cognition 云）和价格（credits 制，重度使用不便宜）。

### 4.5 Factory Droids：最像"企业级 Agent 软件工厂"

Factory 把 Droid 放在桌面、CLI、Web、SDK 上，外面包一层"软件工厂"：Mission Control 跟踪长任务，Automations 跑定时/事件任务，Code Review 评审每个 PR，AutoWiki 维护仓库知识，Agent Readiness 评估仓库对 Agent 的友好度；模型路由可选 GPT、Claude、Gemini 等（[Factory 官网](https://factory.com/)、[第三方评测](https://theaiagentindex.com/agents/factory-ai)）。客户包括 RBC、T-Mobile、Blackstone、Adyen、HP、Wipro；部署支持本地、云、CI、Droid Computers，Factory Private 提供 VPC/on-prem，但仍是 private preview。

- **价格**：Pro $20、Plus $100、Max $200、Teams $60/团队 + $40/席位（最多 10 席）、Business/Enterprise 定制；无免费档，按 5 小时/7 天/30 天滚动限额计量（[Factory 定价](https://factory.com/pricing)）。
- **判断**：产品形态和企业治理最接近"super-agent 商业化后的样子"。它的优势是模型无关、部署灵活、企业客户验证；弱点是 on-prem 未 GA、价格门槛高、主动性仍以"人配 Automations"为主。

### 4.6 OpenHands：开源阵营中功能最完整的对手之一

OpenHands 是 MIT 协议的开源 Agent 平台，GitHub 约 92,500 stars，模型无关，支持本地/自托管（VM/K8s）和云端入口。Automations 支持 GitHub、Slack、Jira、Linear、定时和 webhook，内置"PR 打开自动评审""CI 失败自动修复并开 PR""安全修复""Slack 频道监听"等模板（[OpenHands 官网](https://www.openhands.dev/)、[OpenHands 仓库](https://github.com/OpenHands/OpenHands)）。

- **价格**：自托管免费；Teams $500/月含 10 用户 + $50/额外席位，5,000 会话/月上限；Enterprise 定制、全自托管（[OpenHands 定价](https://www.openhands.dev/pricing)）。
- **判断**：架构上最接近 super-agent 的开源方案，也是"数据不能出内网"场景的直接替代。差距在开箱即用的团队工单/策略层、飞书原生集成和运维成本。

### 4.7 Amp、Cursor、Augment：三条不同的中间路线

- **Amp**：以 "orbs"（每线程一台持久远程机器）为核心，笔记本合上后继续跑；支持每线程定时任务和 GitHub CI/Slack 事件订阅，按 orb 分钟计费，Hobby 免费、Individual $20/月含 45,000 分钟、自托管 orb 免费（[Amp Automations](https://ampcode.com/docs/orbs/automations)、[Amp 定价](https://ampcode.com/pricing)）。它的"常驻感"最强，但没有团队工单收件箱，自动化是"每线程"而非"每仓库策略"。
- **Cursor**：Cloud Agents 在隔离 VM 里跑，支持多仓库、artifacts、远程桌面接管；Automations 触发器覆盖 cron、GitHub/GitLab/Bitbucket、Slack、Linear、webhook，Bugbot 自动评审 PR 并支持 autofix（[Cloud Agent](https://cursor.com/docs/cloud-agent)、[Automations](https://cursor.com/docs/cloud-agent/automations)、[Bugbot](https://cursor.com/docs/bugbot)）。2026 年订阅改由 X Subscription 售卖后，Plus $8、Premium $30、Super $100、Ultra $200，Cloud Agents/Automations 需 Premium 以上（[Cursor 定价](https://cursor.com/pricing)）。触发器最全，但仍是开发者工具而非团队控制平面。
- **Augment Code**：Cosmos 软件工厂提供 Cloud Agents、Ticket-to-PR、Automations、PR 评审和 Context Engine；Standard $20/月（最多 50 席）、Business $100/月、Enterprise 定制含 SSO/SCIM/CMEK/数据驻留（[Augment 定价](https://www.augmentcode.com/pricing)）。企业上下文强，但触发器广度和公开证据不如 Devin/Factory。

### 4.8 Google Jules → Antigravity：一个品类退场的样本

Jules 2025-05 上线、2025-08 进入 beta，支持 GitHub、VM 内执行、先规划后改码、直接开 PR，后来补了 Jules Tools CLI、REST API、Scheduled Tasks、Suggested Tasks（主动扫描仓库找可修的问题，2025-12）和 MCP（[Jules changelog](https://jules.google/docs/changelog)）。但官方已公告 2026-12-31 关停，2026-11-11 后只读，官方称相关经验将用于 Antigravity。它说明：**"异步编码 Agent"作为独立产品，很难同时打赢模型厂商的免费捆绑和平台厂商的分发。**

### 4.9 中国编程 Agent：卖套件，不卖单点

| 产品 | 定位与能力 | 价格（公开） | 与 super-agent 的关系 |
| --- | --- | --- | --- |
| 字节 Trae | IDE + SOLO 自主模式，2026 年与 Trae Work 融合；2026-02 改为按 Token 计费；Free/Lite/Pro/Pro+/Ultra 五档，SOLO 从 Pro+ 起 | Free；Lite $3、Pro $10、Pro+ $30、Ultra $100（连续包月，[IT之家/网易](https://www.163.com/dy/article/KMI3BNI90511B8LM.html)） | 交互式为主，SOLO 接近异步任务，但无团队工单/事件治理；规模最大、价格最低 |
| 阿里 Qoder CN（原通义灵码） | 从编码扩展到办公、数字员工等子产品；Credits 制；Quest/Subagent/Experts；企业专属部署 | 个人免费 300 Credits；Pro ¥59、Pro+ ¥169、旗舰 ¥559；团队 ¥99/人；企业标准 ¥149/人、专属 ¥199/人（[阿里云社区](https://developer.aliyun.com/article/1757140)） | 企业套件路线，VPC/专属部署是强项；缺公开的 GitHub/IM 事件矩阵 |
| 腾讯 CodeBuddy | IDE/插件/CLI；智能体模式、任务管理、Subagents；企业 SaaS 与专有云；企微打通 | SaaS 企业版 ¥198/人/月；专有云 ¥316/人/月（[CodeBuddy 定价](https://www.codebuddy.cn/docs/ide/Codebuddy-enterprise-edition/Codebuddy-enterprise-edition-Pricing)） | 国内最接近"IDE + 企业交付"的组合；IM 生态是优势 |
| 百度 Comate | 企业 License + 智能体代金券；专业版/旗舰版；与办公 Agent DuMate 联合售卖 | 专业版 ¥150/月或 ¥1,500/年；旗舰版 ¥358/月或 ¥3,998/年（[Comate 文档](https://cloud.baidu.com/doc/COMATE/s/bmk2eqq4h)） | 企业采购导向，Agent 能力与生态弱于前三家 |

### 4.10 中国另一条线：IM 原生的"数字员工"

这一条线比"编程 Agent"更接近 super-agent 的形态：常驻、接 IM 事件、有身份/记忆/权限、按任务汇报。它们大多不做 GitHub PR，但抢的是同一个"团队 AI 员工"预算。

| 产品 | 形态 | 关键能力 | 公开价格 | 与 super-agent 的关系 |
| --- | --- | --- | --- | --- |
| 阿里 QoderWake | Always-on 数字员工，2026-04-30 邀测 | 身份、记忆、技能、分工、权限红线五层；员工与工位分离（本地或云沙箱）；任务队列、变更简报、故障初诊、告警分诊；有审批红线 | 未公开（邀测） | **中国最接近 super-agent 的产品形态**：常驻 + 权限边界 + 汇报；但更偏"数字程序员/运维"，不是开源、也未见 PR 工作流（[QoderWake 发布](https://qoder.com/zh/blog/qoderwake)） |
| 腾讯 WorkBuddy | 企微侧边栏机器人 + 企业 Agent Runtime | 7×24 接单、回传；Skills 市场；统一账号/权限/审计；官方称覆盖 50+ 行业、20 万 Skills、90 亿次任务 | 与 CodeBuddy 企业版同页报价（SaaS ¥198/人/月，专有云 ¥316） | 入口和治理最像：企微内常驻、可观测、可审计；但以办公任务为主，GitHub/PR 能力弱（[央广网报道](https://tech.cnr.cn/techph/20261103/t20261103_527860926.shtml)） |
| 百度 DuMate | 企业"AI 工作团队" | 数字员工、企业技能市场、角色权限、调用审计；与 Comate 联合席位 | 原价 ¥189/席位/月；联合席位另计，资源不包含在席位内（[DuMate 企业版](https://cloud.baidu.com/product/dumate.html)） | 企业套件路线，强调"可管控、可审计"；缺开源和开发者工作流 |
| Kimi Claw | 云端 OpenClaw，Kimi 会员产品 | 5,000+ Skills；接微信、飞书、企微、钉钉机器人；收发文件；桌面/移动端 | 随 Kimi 会员档位计费（[Kimi Claw 文档](https://www.kimi.com/help/kimi-claw)） | IM 入口最全的"个人数字员工"；面向个人/小团队，不是仓库级工单 |
| 智谱 AutoClaw | 本地 OpenClaw 一键安装 | 50+ Skills、接飞书、免费额度 + 积分 | 免费额度 + 积分（[证券时报](https://www.stcn.com/article/detail/3669382.html)） | 本地部署 + IM 常驻，偏个人；企业治理弱 |
| 扣子 2.0 / Coze Space | 通用 Agent 与应用平台 | Agent Skill/Plan/Coding/Office；长期计划可持续/定时执行；多平台发布 | 个人进阶 ¥34、高阶 ¥89、旗舰 ¥179；企业 ¥166/月起、旗舰 ¥3,360/月（20 人）（[扣子公告](https://docs.coze.cn/guides_20260119_coze_premium_upgraded)） | 低代码编排，抢"AI 员工"预算；不是开箱即用的常驻团队 Agent |

中国市场的四个特点：

1. **卖"套件"而不是卖"单点 Agent"**：阿里、腾讯、字节、百度都把编码/办公 Agent 放进更大的云、办公、数字员工套件里，靠既有企业客户分发。
2. **私有化与合规是硬需求**：专有云、专属版、信创适配、审计是标准配置；这和 super-agent 的开源自托管方向一致。
3. **IM 是天然入口，"数字员工"是主流话术**：企微、飞书、钉钉机器人 + 定时巡检 + 权限红线 + 人工审批，是中国式"主动性"的常见组合；真正无人值守的主动发起仍集中在告警、变更、日程等窄场景。
4. **纯编程工具被低价和免费压缩**：Trae 把价格打到 $3–$100，Qoder/CodeBuddy 用 Credits 和企业席位打包；单卖"更会写代码"很难收费，收费点在私有化、治理、行业方案。

### 4.11 相邻品类与退出者

- **通用常驻 Agent**：OpenAI dots（GPT-6 Astra、云端电脑/浏览器、主动找人）、Manus（主要市场在海外）、Genspark、扣子。它们争夺"AI 员工"心智，但缺少仓库级工具链和 PR 交付。
- **评审/质量类**：Qodo（多 Agent PR 评审、规则生命周期、on-prem，$0.012/credit，[Qodo 定价](https://www.qodo.ai/pricing/)）。它不实现 Issue，只做评审，和 super-agent 是互补关系。
- **退出者**：Sweep（issue-to-PR 退场）、Codegen（被 ClickUp 收购）、Jules（关停，经验转向 Antigravity）。它们说明单点"issue-to-PR"的护城河很浅。
- **未单独展开的相邻产品**：Replit Agent（应用生成）、Lovable/Bolt（应用生成）、Cline/Roo Code（开源 IDE Agent）、JetBrains Junie、Amazon Q Developer/Kiro。它们更偏应用生成或 IDE 内协作，不改变本文的竞争格局判断。

## 5. 能力对比矩阵

### 5.1 国际市场

符号说明：● 原生支持且较完整；◐ 部分支持或需要额外配置；○ 基本不支持。

| 产品 | 触发方式 | 运行环境 | 常驻/主动 | 交付物 | 多任务/编排 | IM/工单 | 开源/私有化 | 价格量级 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **super-agent** | GitHub/飞书事件、@、规则（observe 可只记录） | 本地 DSH Session | ● 常驻 + 策略门控 | PR、评论、IM 回复 | ● Ticket 并行、Router/Worker | 飞书、GitHub | ● MIT、自托管 | 免费，模型自付 |
| **Devin** | @、Issue、Slack/Linear/Jira、GitHub/GitLab/CI、cron、webhook | Cognition 云；Outposts 混合 | ◐ 事件自动化 + 持续频道 | PR、Slack、评审 | ● Sessions、多仓库 | Slack/Teams、Linear/Jira、GitHub | ○ 闭源；混合部署 | $20–$200/席位 + credits |
| **Factory** | Slack/Linear/Jira、Automations、GitHub 事件、webhook | 本地、云、CI、Droid Computers | ◐ 定时/事件 | PR、评审、文档 | ● Missions/Workstreams | Slack/Linear/Jira、GitHub | ◐ 企业 on-prem（preview） | $20–$200；Teams $60 + $40/席 |
| **Copilot + Agent HQ** | assign Issue、`@copilot`、Agents 页、Automations | 云 sandbox、Actions、remote | ◐ 定时/事件（Automations 仅私有仓库） | PR、评审（不可自合并） | ● 多 Agent 统一入口 | GitHub 为主 | ○ 云端；GHES 另计 | $10–$39/席位 |
| **Codex** | 显式任务、Slack、GitHub `@codex`、定时、dots | 云 sandbox、本地 | ◐ dots always-on（通用） | PR、评审、commit | ● 多任务、subagents | Slack、GitHub、Linear | ○ 闭源；支持 API Key | $8–$500/席位 |
| **Claude Code** | CLI/IDE/Web/Slack、`@claude`、scheduled tasks | 本地、云、Actions | ◐ 定时/事件 | PR、评论 | ● sessions；teams 实验 | Slack、GitHub/GitLab | ○ 闭源；企业自托管环境 | $17–$200/席位 |
| **Cursor** | `@cursor`、Automations（cron/GitHub/GitLab/Slack/Linear） | 云 VM、本地 | ◐ 事件 | PR、评审、Bugbot | ● 多 Agent | Slack、GitHub、Linear | ○ 闭源 | $30–$200（X 订阅） |
| **Amp** | 线程、每线程定时、事件订阅、webhook | Orbs（云/自托管） | ◐ 每线程常驻 | 变更、PR | ◐ 线程级 | Slack、GitHub CI | ◐ 自托管 orb | 免费–$20 |
| **OpenHands** | GitHub、Slack、Jira、Linear、定时、webhook | 本地、VM/K8s、云 | ◐ 事件自动化 | PR、评论 | ● 多会话 | Slack/Jira/Linear、GitHub | ● MIT、自托管 | 自托管免费；Teams $500 + $50/席 |
| **Jules（退出）** | 任务、Scheduled Tasks、Suggested Tasks | Google VM | ◐ 主动建议（需确认） | PR、评论 | ◐ 单任务为主 | GitHub | ○ 闭源 | 2026-12-31 关停 |

### 5.2 中国市场

| 产品 | 触发方式 | 运行环境 | 常驻/主动 | 交付物 | 多任务/编排 | IM/工单 | 开源/私有化 | 价格量级 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **QoderWake** | 任务、巡检（邀测） | 本地或云"工位" | ● always-on | 简报、诊断、建议/修复 | ● 任务队列 | 待公开 | ○ 闭源 | 未公开 |
| **WorkBuddy** | 企微会话、7×24 接单 | 腾讯云/企业部署 | ● 常驻 | 办公任务、回传 | ● 多任务 | 企微为主 | ◐ 专有云 | ¥198–¥316/席 |
| **DuMate** | 企业任务、数字员工 | 百度智能云 | ● 企业常驻 | 办公任务、流程 | ● 多数字员工 | 百度生态 | ◐ 企业版 | ¥189/席起（原价）+ 资源 |
| **Kimi Claw** | IM 机器人、群聊、文件 | 云端 OpenClaw | ● 常驻 | 文件、任务结果 | ◐ 个人/小团队 | 微信、飞书、企微、钉钉 | ○ 云端 | Kimi 会员内 |
| **Trae** | 人发起、SOLO 模式 | 本地/云 | ○ 以人为主 | 代码、文档、网页 | ◐ Subagent | 无原生飞书/钉钉 | ◐ 企业私有化 | $3–$100/月 |
| **Qoder CN** | 人发起、Quest/Experts | 本地/企业云 | ○ 以人为主 | 代码、文档 | ◐ Subagent | 有限 | ◐ 专属/VPC | ¥59–¥559/月 |
| **CodeBuddy** | 人发起、智能体模式 | 本地/专有云 | ○ 以人为主 | 代码、任务 | ◐ Subagent | 企微生态 | ◐ 专有云 | ¥198–¥316/席 |
| **Comate** | 人发起、Plan 模式 | 本地/云 | ○ 以人为主 | 代码、文档 | ◐ Subagent | 有限 | ◐ 企业版 | ¥150–¥358/月 |
| **扣子 2.0** | 用户任务、长期计划 | 火山引擎 | ◐ 定时计划 | 应用、报告、办公 | ● 工作流/多 Agent | 多平台发布 | ○ SaaS | 免费–¥3,360/月 |

## 6. 与 super-agent 的逐项对比

### 6.1 谁最像它

| 产品 | 常驻/主动 | 工单化/授权 | 跨渠道（IM+代码） | 开源自托管 | PR 闭环 | 一句话 |
| --- | --- | --- | --- | --- | --- | --- |
| **super-agent** | ● | ● | ● | ● | ● | 参照物 |
| **Devin** | ● | ◐ | ◐ | ○ | ● | 功能最全的闭源"数字工程师" |
| **Factory** | ◐ | ◐ | ◐ | ◐ | ● | 企业治理最强的 Agent 软件工厂 |
| **QoderWake** | ● | ◐ | ◐ | ○ | ◐ | 中国最像的"数字员工"，但偏运维/办公 |
| **OpenHands** | ◐ | ◐ | ◐ | ● | ● | 开源阵营最完整的替代 |
| **Copilot + Agent HQ** | ○ | ○ | ◐ | ○ | ● | 分发和治理的平台重力，不是常驻 Agent |
| **Claude Code** | ◐ | ○ | ◐ | ○ | ● | 执行与开发者心智最强，缺工单层 |
| **Codex** | ◐ | ○ | ◐ | ○ | ● | 平台化最完整，主动性偏通用助手 |
| **WorkBuddy** | ● | ◐ | ◐ | ○ | ○ | IM 原生 + 企业治理，缺代码仓库闭环 |

结论：**Devin 是国际市场上功能最接近的产品；QoderWake 是中国市场形态最接近的产品；OpenHands 是开源/自托管方向最接近的产品。** 但三者都缺 super-agent 的一个核心组合：**把"是否接单"做成可版本化的策略，把"一条工作"做成可恢复、可审计、可交接的 Ticket，而不是一个会话或一次自动化。**

### 6.2 super-agent 真正独有的组合

1. **策略门控的主动性**：Gate（程序层）+ Assessment（模型层）两层评估，`observe` 模式让团队先看"Agent 本来会做什么"再放开。多数产品的主动性是"人配规则/定时"，没有"这条规则允许谁、做到什么程度、依据哪个策略版本"的授权模型。
2. **Ticket 作为一等公民**：Mandate（授权）、Assignment（派工）、Subscription（订阅）、Resume（恢复）、Handoff（交接）都在领域模型里。Session 只是执行载体，挂了可以重派；这比"一个 Agent 会话"更接近真实团队的工作方式。
3. **注意力预算**：确认一次、工作期间安静、只在有结果或需要决策时说话，问题用结构化字段提出。这是产品原则，不是功能；多数竞品仍在用"消息越多越显得在干活"的方式。
4. **开源 + 自托管 + 本地执行 + 模型无关**：代码和数据不出内网，模型可以换（DeepSeek、GLM、Kimi、MiniMax 或任意 OpenAI 兼容网关）。OpenHands 也有这些，但没有 super-agent 的工单/策略层。
5. **飞书 + GitHub 的统一事件/线程模型**：同一个 Ticket 可以同时挂在飞书话题和 GitHub PR 上，后续消息按订阅扇入，不要求用户搬上下文。这是本土化最实用的一点。

### 6.3 差距：从"原型"到"企业可用"

| 维度 | super-agent 现状 | 主流企业产品 | 影响 |
| --- | --- | --- | --- |
| 身份与权限 | 无 SSO/SCIM/RBAC；白名单是 Source 级 | Claude/Copilot/Factory 均有企业身份与角色 | 大团队无法采购 |
| 审计与合规 | 状态落库，但无审计导出、保留策略 | 审计日志、ZDR、数据驻留、SOC 2/ISO | 金融、政企过不了合规 |
| 隔离与安全 | Worker 以 `danger-full-access` 运行 | 云 sandbox/VM、最小权限、网络白名单 | 误操作和供应链风险高 |
| 多仓库/多租户 | 单仓库、单实例、单租户 | 多仓库、组织级、租户隔离 | 无法服务平台团队 |
| 成本治理 | 无配额、无计费、无预算告警 | credits/ACU、限额、用量报表 | 无法控制账单 |
| 评测 | 合成负载初步实验 | 公开基准 + 客户案例/SLA | 难以证明 ROI |
| 生态 | GitHub + 飞书两个 Source | Slack/Teams/Jira/Linear/GitLab/CI 等 | 接入成本高 |
| 可靠性 | 研究原型，无 SLA | 企业 SLA、值班、支持 | 生产不敢用 |
| 交互 | 本地面板 | Web/移动/IDE/IM 多端 | 体验差距 |

### 6.4 威胁：三条线同时挤压

1. **平台向下**：GitHub Agent HQ 把多 Agent 收进 Copilot 订阅内的 GitHub 工作流；Copilot Automations 直接覆盖"定时/事件 → PR"。团队没有理由再自建一个只服务单仓库的调度器。
2. **模型厂商向上**：OpenAI dots 是 always-on 通用 Agent，Anthropic 用 Claude Tag/Projects 做组织级入口。模型厂商一旦把"常驻 + 记忆 + 权限"做成平台能力，单纯的路由层会被吸走。
3. **开源与低价替代**：OpenHands 提供 MIT、自托管、事件触发和 PR 模板；Trae 把价格打到 $3/月，Qoder/CodeBuddy 用企业套件打包。super-agent 的"免费开源"不是独占优势。

### 6.5 机会：三个可守的位置

1. **治理型控制平面**：不和执行层比模型，做"谁可以指派、能做什么、花了多少、谁批准、怎么审计"的层。企业愿意为治理付费，且大厂产品很难为单个团队深度定制。
2. **IM 原生的团队入口**：飞书/企微/钉钉是中国团队的默认工作界面。把 IM 事件变成工单、把结果回到 IM，是本土化的真需求；Kimi Claw/WorkBuddy 已验证入口价值，但缺 GitHub/PR 闭环。
3. **"触发 + 工单"层与执行层组合**：不替代 Claude Code/Codex/Copilot，而是把它们的产出纳入统一工单和评审流。对已经买了 Copilot/Claude 的团队，这比"再换一个 Agent"更容易接受。

## 7. 趋势判断

1. **主动性会变成标配，"可授权的主动性"才是差异。** 2026 年各家都补了定时/事件触发；下一步竞争的是"谁来判断这件事该不该做、做到什么程度、出了事谁负责"。
2. **控制平面战争已经开始。** GitHub（Agent HQ）、Anthropic（Claude Tag/Projects）、OpenAI（dots/Agents）、Devin/Factory（Mission Control）都在抢"任务入口"。入口的胜者会挤压单点 Agent 的生存空间。
3. **独立 Agent 层继续整合。** Jules、Codegen、Sweep 的路径说明：没有分发或治理的单点产品，最终会被平台、IDE 或企业套件收编。
4. **定价从席位转向"席位 + 用量 + 企业合同"。** Copilot 的 AI credits、Devin 的 ACU/credits、Cursor 的订阅池、国内厂商的 Credits/积分，都是同一方向；成本治理会成为采购的硬指标。
5. **交付终点从 PR 延伸到"可合并"。** 评审、CI 修复、合并冲突、部署、回滚正在被纳入 Agent 闭环；只会开 PR 不够，能对"合并质量"负责才有价值。
6. **评测从基准分数转向端到端指标。** SWE-bench 之类的分数不能回答"这个团队每周能少花多少小时"。任务完成率、PR 接受率、人工干预次数、单位任务成本、注意力成本会成为新的对比口径。
7. **IM 正在变成 Agent 的操作系统。** Slack/Teams/飞书/企微不只是通知渠道，而是身份、记忆、权限、审批和协作的入口；原生 IM 集成会比 IDE 插件更有黏性。
8. **开源与闭源分层。** 闭源厂商拼模型、沙箱和生态；开源项目拼自托管、可定制和数据主权。中间态（开源核心 + 企业版）会越来越多。

## 8. 对 super-agent 的启示

### 8.1 如果它是内部工具

- **先别急着替换，先做"触发器 + 工单层"**：把执行交给团队已经在用的 Claude Code/Codex/Copilot，super-agent 负责从飞书/GitHub 收事件、评估、开工单、跟踪、回复。这样既保留现有投入，又补上"没人盯盘"的空白。
- **优先补三个企业能力**：审计导出（谁在什么时候让 Agent 做了什么）、配额/预算（每个 Ticket 的 Token/费用上限）、沙箱隔离（不要把 `danger-full-access` 暴露给生产仓库）。
- **用三个指标证明价值**：每个任务的人工干预次数、从请求到 PR 的端到端时延、每周节省的开发者小时数。不要用"发了多少消息/开了多少 PR"当 KPI。
- **选 1–2 个低风险仓库试点**：先开 `observe`，观察"本来会做什么"；再放开显式请求；最后才开规则驱动的主动工作。

### 8.2 如果它是产品

- **不要做"更会写代码的 Agent"**：这条赛道有 Claude Code、Codex、Cursor、Copilot，且价格战已经开始。
- **做"团队级 Agent 控制平面"**：多仓库、多 IM、策略/审批/审计、成本治理、与主流执行 Agent 互操作。开源核心 + 企业版（SSO/SCIM/审计/多租户/支持）是常见且可行的商业模式。
- **本土化是护城河**：飞书/企微/钉钉原生、私有化/信创、等保合规、中文工作流。这些是大厂 SaaS 和海外产品短期难补的。
- **谨慎对待"主动性"叙事**：客户真正怕的是"Agent 乱来"，不是"Agent 不够主动"。把"权限红线 + 审批 + 可回滚"讲清楚，比"7×24 自主干活"更能促成采购。

## 9. 结论

1. **市场验证了 super-agent 的方向**：常驻、事件驱动、团队级、从 IM/GitHub 到 PR 的闭环，是 2026 年 Agent 产品共同演进的方向。
2. **但本次调研范围内没有完全同构的竞品**：Devin/Factory 最接近功能，QoderWake 最接近形态，OpenHands 最接近开源/自托管；它们都没有把"可授权、可审计、可恢复的工单"做成核心抽象。
3. **"主动性"本身不再是护城河**：定时任务、事件触发、常驻 Agent 正在被平台免费化；真正稀缺的是治理、上下文连续性和注意力控制。
4. **super-agent 的最大价值不是替代编码 Agent，而是把它们组织起来**：做团队的"触发器 + 工单 + 治理"层，让 Claude Code/Codex/Copilot 在明确的授权和预算内工作。
5. **从原型到产品，优先级是治理 > 生态 > 模型 > 主动性**：先解决 SSO/审计/配额/沙箱，再谈多仓库和多渠道；否则能力越强，越难被企业放心采用。

## 10. 参考来源

> 访问时间均为 2026-11-03。价格为公开列表价，可能随版本调整；带"第三方"标注的数据未获厂商官方确认。

**官方文档与定价**

- DeepSeek dsh-superagent：https://github.com/deepseek-ai/dsh-superagent-prototype
- DeepSeek Harness：https://www.deepseek.com/en/harness/
- OpenAI Codex Cloud：https://learn.chatgpt.com/docs/cloud
- OpenAI Codex 定价：https://learn.chatgpt.com/docs/pricing
- OpenAI Codex Automations：https://learn.chatgpt.com/docs/automations
- OpenAI dots：https://learn.chatgpt.com/docs/dots
- Anthropic Claude Code 概览：https://code.claude.com/docs/en/overview
- Anthropic Claude Code 云会话：https://code.claude.com/docs/en/claude-code-on-the-web
- Anthropic Claude Code scheduled tasks：https://code.claude.com/docs/en/routines
- Anthropic Claude Code in Slack：https://code.claude.com/docs/en/slack
- Anthropic agent teams：https://code.claude.com/docs/en/agent-teams
- Anthropic 定价：https://claude.com/pricing
- GitHub Copilot coding agent：https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent
- GitHub Copilot plans：https://docs.github.com/en/copilot/get-started/plans
- GitHub Copilot Automations：https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/create-automations
- GitHub 第三方 Agent：https://docs.github.com/en/copilot/concepts/agent-capabilities/about-agents/about-third-party-coding-agents
- GitHub Agent HQ 发布：https://github.blog/news-insights/company-news/welcome-home-agents/
- Devin Automations：https://docs.devin.ai/product-guides/automations.md
- Devin Outposts：https://docs.devin.ai/cloud/outposts/overview.md
- Devin 自助计费：https://docs.devin.ai/admin/billing/self-serve.md
- Devin 定价：https://devin.ai/pricing
- Cognition Series E：https://cognition.com/blog/series-e
- Cognition 内部使用 Devin：https://cognition.com/blog/how-cognition-uses-devin-to-build-devin
- Factory：https://factory.com/
- Factory 定价：https://factory.com/pricing
- OpenHands：https://www.openhands.dev/
- OpenHands 仓库：https://github.com/OpenHands/OpenHands
- OpenHands 定价：https://www.openhands.dev/pricing
- Amp Automations：https://ampcode.com/docs/orbs/automations
- Amp 定价：https://ampcode.com/pricing
- Cursor Cloud Agent：https://cursor.com/docs/cloud-agent
- Cursor Automations：https://cursor.com/docs/cloud-agent/automations
- Cursor Bugbot：https://cursor.com/docs/bugbot
- Cursor 定价：https://cursor.com/pricing
- Augment 定价：https://www.augmentcode.com/pricing
- Google Jules changelog：https://jules.google/docs/changelog
- Qodo 定价：https://www.qodo.ai/pricing/
- Codegen 公告：https://codegen.com/an-update-on-codegen/

**中国厂商**

- 阿里 Qoder CN 个人版价格：https://developer.aliyun.com/article/1757140
- 阿里 Qoder CN 计费公告：https://cn.aliyun.com/notice/118264
- 阿里 QoderWake 发布：https://qoder.com/zh/blog/qoderwake
- 腾讯 CodeBuddy 企业版定价：https://www.codebuddy.cn/docs/ide/Codebuddy-enterprise-edition/Codebuddy-enterprise-edition-Pricing
- 腾讯 CodeBuddy 企业版概述：https://www.codebuddy.cn/docs/ide/Codebuddy-enterprise-edition/Codebuddy-enterprise-edition-Overview
- 腾讯 WorkBuddy 报道：https://tech.cnr.cn/techph/20261103/t20261103_527860926.shtml
- 百度 Comate 付费订阅：https://cloud.baidu.com/doc/COMATE/s/bmk2eqq4h
- 百度 DuMate 企业版：https://cloud.baidu.com/product/dumate.html
- 字节 Trae 计费调整：https://www.163.com/dy/article/KMI3BNI90511B8LM.html
- 扣子订阅套餐：https://docs.coze.cn/guides_20260119_coze_premium_upgraded
- Kimi Claw：https://www.kimi.com/help/kimi-claw
- 智谱 AutoClaw 报道：https://www.stcn.com/article/detail/3669382.html
- Manus 定价：https://help.manus.im/es/articles/11711111-cual-es-el-precio-actual-de-la-membresia-de-manus

**媒体与第三方分析**

- IT for Business（Agent HQ，2026-02-09）：https://www.itforbusiness.fr/claude-et-codex-debarquent-dans-agent-hq-github-multiplie-les-agents-100149
- CloudZero（Copilot 成本，2026）：https://www.cloudzero.com/blog/github-copilot-cost/
- The AI Agent Index（Factory 评测，2026-11-02）：https://theaiagentindex.com/agents/factory-ai
- 七牛云（DeepSeek Super-agent 架构解读，2026-11-01）：https://news.qiniu.com/archives/1793499698327

---
