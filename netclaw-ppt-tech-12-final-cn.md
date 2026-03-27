# NetClaw 12页技术汇报版（正式稿）

## 第1页：标题页

**标题**
NetClaw：Skill 驱动的网络工程 AI 执行体系

**副标题**
基于 OpenClaw、Skills 与 MCP 的网络运维闭环平台

**页面正文**
- NetClaw 的目标不是替代工程师，而是把高级网络工程经验沉淀为可执行流程
- 它把排障、变更、验证、通知、审计统一到一个 AI 操作平面

**建议配图**
- 用 `uml-diagram` 生成一个 `component` 或 `structurizr` 架构图

---

## 第2页：NetClaw 是什么

**页面正文**
- NetClaw 是一个 CCIE 级别的网络工程 AI coworker
- 上层通过 Slack / WebEx / WebChat 接收自然语言请求
- 中间用 Skills 固化方法论与 SOP
- 下层通过 MCP 连接网络设备、SoT、ITSM、可观测性和协同平台

**讲稿**
- 传统自动化更多是“命令级自动化”
- NetClaw 更像“流程级自动化 + 治理 + 审计”

**建议配图**
- 用 `uml-diagram` 生成 `c4plantuml` 或 `structurizr` 系统上下文图

---

## 第3页：为什么需要 NetClaw

**页面正文**
- 工具割裂：CLI、NetBox、Grafana、ServiceNow、GitHub、Teams 分散
- 排障低效：工程师要在多个系统来回切换
- 变更风险高：缺少统一 baseline、approval、verification
- 审计困难：很难回答“AI 做了什么、为什么这么做”
- SoT 漂移：设计意图和设备真实状态不一致

**讲稿**
- 真正难的不是“调用一个 API”，而是“按正确顺序完成整段工作”

**建议配图**
- 用 `markmap-viz` 生成一张“传统网络运维痛点树”

---

## 第4页：技术架构分层

**页面正文**
1. 交互层：Slack / WebEx / WebChat
2. Agent 层：OpenClaw + NetClaw 规则、身份、上下文
3. Skill 层：排障、变更、对账、可观测性、可视化、审计
4. MCP 层：pyATS、ServiceNow、NetBox、Grafana、Prometheus、gNMI 等

**讲稿**
- Skill 决定“做事的方法”
- MCP 决定“连接哪些系统”
- Tool 决定“每一步具体怎么执行”

**建议配图**
- 用 `uml-diagram` 生成 `component` 图

---

## 第5页：业务流程案例

**页面正文**
- 案例：分支站点访问总部 ERP 变慢
- 目标：快速定位问题、判断是否需要变更、完成验证并输出审计结果
- 涉及对象：路由设备、监控平台、SoT、ServiceNow、通知渠道

**讲稿**
- 这是一个很典型的真实场景，因为它横跨网络状态、路径质量、配置治理和结果交付

**建议配图**
- 用 `drawio-diagram` 生成一个简单的物理/逻辑拓扑图

---

## 第6页：第一阶段，排障与状态采集

**核心 Skill**
- `pyats-troubleshoot`
- `pyats-network`
- `pyats-routing`
- `pyats-parallel-ops`

**页面正文**
- 先定义问题，再收集事实，禁止猜测设备状态
- 按 L1/L2/L3/L4+ 分层检查接口、ARP、路由、ACL、NAT
- 对 OSPF/BGP 邻接做专项检查
- 对慢性能问题检查 CPU、内存、接口利用率和 QoS
- 对多设备路径做并行状态采集

**讲稿**
- `pyats-troubleshoot` 里明确写的是：Define problem -> Gather facts -> Consider possibilities -> Implement and verify -> Document

**建议配图**
- 用 `markmap-viz` 生成“故障排查树”

---

## 第7页：第二阶段，路径与可观测性交叉验证

**核心 Skill**
- `gtrace-path-analysis`
- `te-path-analysis`
- `grafana-observability`
- `prometheus-monitoring`
- `suzieq-observability`

**页面正文**
- `gtrace` / `ThousandEyes` 用来检查路径、丢包、抖动、全球视角
- `Grafana` 用来查 dashboard、PromQL、LogQL、alert、incident
- `Prometheus` 用来直接查询指标
- `SuzieQ` 用来检查网络状态与断言

**讲稿**
- `grafana-observability` 不是只看图，它还可以直接做 alert investigation、incident correlation、log analysis

**建议配图**
- 用 `canvas-network-viz` 表达 health dashboard / path trace 的展示思路

---

## 第8页：第三阶段，SoT 对账与根因收敛

**核心 Skill**
- `netbox-reconcile`
- `nautobot-sot`
- `infrahub-sot`

**页面正文**
- `netbox-reconcile` 会正式检查：
  - `IP_DRIFT`
  - `MISSING_INTERFACE`
  - `UNDOCUMENTED_LINK`
  - `CABLE_MISMATCH`
  - `VLAN_MISMATCH`
  - `STATUS_MISMATCH`
  - `MTU_MISMATCH`
- 它不是简单查配置，而是把“设计意图”和“真实网络”做对比

**讲稿**
- 很多看起来像故障的问题，根因其实是 SoT 漂移
- NetClaw 把这一步内建进正式排障链路

**建议配图**
- 用 `markmap-viz` 做“drift summary”
- 用 `drawio-diagram` 对链路按 reconciled / mismatch / undocumented 着色

---

## 第9页：第四阶段，受控变更执行

**核心 Skill**
- `servicenow-change-workflow`
- `pyats-config-mgmt`
- `gnmi-telemetry`

**页面正文**
- 先检查目标 CI 是否有 P1/P2 事件
- 创建 CR，明确风险、影响、回滚计划
- 等待状态到 `Implement`
- 才允许进入执行阶段
- 执行后必须做 post-change verification
- 验证失败触发 rollback / incident

**讲稿**
- `servicenow-change-workflow` 的 golden rule 是：没有批准的 CR，不做正式网络变更

**建议配图**
- 用 `uml-diagram` 生成 `sequence` 图，描述 Engineer -> NetClaw -> ServiceNow -> Device -> GAIT 的变更链路

---

## 第10页：第五阶段，GAIT 审计闭环

**核心 Skill**
- `gait-session-tracking`

**页面正文**
- 每次会话必须 `gait_branch`
- 每个关键动作都要 `gait_record_turn`
- 会话结束必须 `gait_log`
- GAIT 记录：
  - 问了什么
  - 查了什么
  - 改了什么
  - 验证结果如何
  - 产出了哪些工单、图、报告

**讲稿**
- GAIT 是 NetClaw 区别于普通 agent 的关键能力之一
- 它让 AI 的行为变得可追踪、可解释、可复盘

**建议配图**
- 用 `uml-diagram` 生成 `activity` 或 `sequence` 图表示 audit trail lifecycle

---

## 第11页：结果交付与可视化

**核心 Skill**
- `canvas-network-viz`
- `drawio-diagram`
- `uml-diagram`
- `markmap-viz`
- `slack-report-delivery`
- `msgraph-teams`

**页面正文**
- `canvas-network-viz`：内嵌 topology、dashboard、alert card、change timeline、path trace
- `drawio-diagram`：生成网络拓扑和分享图
- `uml-diagram`：生成 sequence、component、C4、nwdiag、state 图
- `markmap-viz`：生成层级化思维导图

**讲稿**
- NetClaw 的最终交付物不是一段文字，而是“工程师可消费的多种形式结果”

**建议配图**
- 本页可直接放 3 张缩略图：
  - 架构图
  - 变更时序图
  - drift / troubleshooting mindmap

---

## 第12页：规模、团队与业务价值

**规模数据**
- 官方主口径：`101 skills`
- 官方主口径：`46 MCP integrations`
- 当前 `README` 已枚举 `55` 个 MCP server 条目
- 当前工作区中有 `108` 个 `SKILL.md`
- 按文档中明确可数条目保守统计：`1224+ tools/endpoints`

**团队情况**
- 从 git 历史可见 `6` 个 author identity
- 如果合并同一核心作者的不同身份，约 `5~6` 位贡献者
- 整体呈现“核心作者主导 + 多人协作扩展”特征

**业务价值**
- 缩短 MTTR
- 提高变更标准化和成功率
- 降低跨平台协作成本
- 提升 AI 的可审计、可治理、可规模化落地能力

**结束语**
NetClaw 的关键价值，不是接入了多少工具，而是把网络工程流程真正做成了可执行闭环。

---

## 附：建议放进 PPT 的 3 张图

**图1：总体架构图**
- 推荐用 `uml-diagram`
- 类型：`component` 或 `structurizr`

**图2：变更执行时序图**
- 推荐用 `uml-diagram`
- 类型：`sequence`

**图3：故障排查 / Drift 思维导图**
- 推荐用 `markmap-viz`

## 附：汇报时的推荐一句话

NetClaw 不是“会调用很多工具的 AI”，而是“把网络工程师的 SOP 变成 Skills，再由 MCP 和 Tools 落地执行”的工程平台。
