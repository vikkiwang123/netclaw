# NetClaw PPT 大纲

## Slide 1. 封面

**标题：** NetClaw：面向网络运维与变更的 AI 数字同事

**副标题：** 围绕一个真实业务流程，看 NetClaw 如何把排障、变更、审计和协同串成闭环

**可讲稿：**
- NetClaw 不是单点工具，而是一个基于 OpenClaw 的网络工程 AI coworker。
- 它把网络设备、ITSM、SoT、可观测性、协同平台和审计系统统一到同一个操作入口中。

---

## Slide 2. NetClaw 是什么

**一句话定义：**
NetClaw 是一个 CCIE 级别的网络工程 AI 协作体，面向“监控、排障、变更、审计、可视化、协同交付”全流程工作。

**核心特征：**
- 统一入口：Slack / WebEx / WebChat 发起请求
- 统一编排：通过 Skills 把不同 MCP 工具串成标准作业流程
- 统一治理：变更前基线、ServiceNow CR、变更后验证、GAIT 审计闭环
- 统一可视化：支持 Canvas/A2UI、Draw.io、Visio、UML 等结果输出

**可讲稿：**
- NetClaw 的价值不只是“能调一个工具”，而是“能完成一段业务流程”。

---

## Slide 3. 为什么需要 NetClaw

**它解决的业务问题：**
- 工具割裂：设备 CLI、NetBox、ServiceNow、Grafana、GitHub 各自独立
- 排障慢：人工切系统、切页面、找日志、对配置，链路长
- 变更风险高：没有统一基线、审批、验证和回滚流程
- 审计困难：出了问题很难回答“AI 做了什么、为什么这么做”
- SoT 漂移：NetBox/Nautobot 与设备真实状态长期不一致
- 跨域复杂：园区、数据中心、云、K8s、SD-WAN、负载均衡、可观测性平台都要一起看

**可讲稿：**
- 传统自动化解决的是“某一步快一点”。
- NetClaw 解决的是“从发现问题到闭环交付”的整段效率和可靠性。

---

## Slide 4. 具体业务流程：门店访问总部业务变慢

**场景设定：**
某分支门店反馈访问总部业务系统变慢，怀疑是 WAN 路径抖动、BGP 邻接异常或策略变更导致。

**传统处理方式：**
- 工程师登录设备查接口、查路由、查日志
- 再切到 Grafana / ThousandEyes / NetBox / ServiceNow
- 最后人工整理结论、提变更、执行验证、发通知

**NetClaw 处理方式：**
- 在聊天窗口直接发起请求
- 自动按标准技能链路做“排障 -> 定位 -> 变更 -> 验证 -> 通知 -> 审计”

**可讲稿：**
- 这个场景适合展示 NetClaw，因为它同时覆盖监控、拓扑、路径、协议、ITSM 和协同交付。

---

## Slide 5. 这条流程里 NetClaw 怎么工作

**第 1 步：接收问题**
- 在 Slack / WebEx / WebChat 中输入自然语言
- 例如：`帮我排查 Site-B 到总部 ERP 访问变慢`

**第 2 步：自动采集现状**
- 用 `pyats-network`、`pyats-routing` 拉设备状态
- 用 `gtrace-path-analysis`、`te-path-analysis` 看路径和质量
- 用 `grafana-observability`、`prometheus-monitoring` 看指标和告警
- 用 `suzieq-observability` 看网络状态和历史断言

**第 3 步：交叉验证**
- 用 `netbox-reconcile` 或 `nautobot-sot` 对比设计意图和真实状态
- 用 `rfc-lookup`、`nvd-cve` 做协议和风险辅助判断

**第 4 步：需要变更时走受控流程**
- 用 `servicenow-change-workflow` 创建或核查 CR
- 用 `pyats-config-mgmt` 或 `gnmi-telemetry` 执行变更
- 用 `gait-session-tracking` 记录全过程

**第 5 步：验证与输出**
- 自动做 post-check
- 输出报告、图表、通知、审计记录

---

## Slide 6. 这条流程里会用到哪些 Skills

**排障主链路：**
- `pyats-network`
- `pyats-routing`
- `pyats-troubleshoot`
- `pyats-parallel-ops`

**路径与可观测性：**
- `gtrace-path-analysis`
- `te-path-analysis`
- `grafana-observability`
- `prometheus-monitoring`
- `suzieq-observability`

**源数据与治理：**
- `netbox-reconcile`
- `nautobot-sot`
- `servicenow-change-workflow`
- `gait-session-tracking`

**展示与协同：**
- `canvas-network-viz`
- `drawio-diagram`
- `msgraph-teams`
- `slack-report-delivery`
- `github-ops`

**可讲稿：**
- Skill 的本质不是“又一个插件”，而是“把工具变成 SOP”。
- NetClaw 的优势在于会按技能方法论执行，而不是随意调用工具。

---

## Slide 7. 这条流程里会用到哪些 Tools / MCP

**典型 MCP 组合：**
- `pyATS MCP`：设备 show/config/验证
- `ServiceNow MCP`：变更审批与闭环
- `NetBox / Nautobot MCP`：SoT 校验
- `Grafana / Prometheus / SuzieQ MCP`：指标、日志、状态、断言
- `gtrace / ThousandEyes MCP`：路径质量与全球视角探测
- `gNMI MCP`：模型驱动配置与订阅
- `GitHub MCP`：问题单、PR、配置归档
- `Microsoft Graph MCP`：Teams、SharePoint、Visio

**对业务最关键的不是单个 tool，而是组合能力：**
- 采集真实状态
- 对照设计意图
- 在审批下执行变更
- 自动验证结果
- 输出审计证据

---

## Slide 8. NetClaw 的核心点在哪里

**核心点 1：Skill 驱动，而不是裸 Tool 调用**
- 把专家经验固化为 SOP
- 保证不同会话、不同工程师、不同场景下行为一致

**核心点 2：真实状态优先**
- 先拉状态，再下结论
- 强调“Never guess device state”

**核心点 3：变更治理内建**
- ServiceNow CR
- 变更前基线
- 变更后验证
- 失败升级与回滚思路

**核心点 4：GAIT 审计**
- 记录“问了什么、查了什么、改了什么、产出了什么”
- 回答管理者最关心的问题：AI 到底做了什么

**核心点 5：跨平台统一编排**
- 设备、SoT、ITSM、观测、协同、代码仓统一起来

---

## Slide 9. NetClaw 解决了哪些业务问题

**效率问题：**
- 缩短 MTTR
- 减少工程师在多个系统之间切换
- 把重复排障和标准检查自动化

**质量问题：**
- 减少误判和漏检
- 提高配置变更的标准化程度
- 提高验证覆盖率

**治理问题：**
- 让 AI 工作可审计、可回溯、可解释
- 把 ITSM、SoT、合规、通知纳入同一个闭环

**扩展性问题：**
- 从传统网络扩展到云、K8s、SD-WAN、F5、Meraki、ACI、Grafana 等多域场景

---

## Slide 10. 团队与生态规模

**从仓库主叙事看：**
- `101 skills`
- `46 MCP integrations`

**从当前仓库清单看：**
- `README` 的 MCP 清单已枚举到 `55` 个服务器条目
- `workspace/skills` 目录当前有 `108` 个 `SKILL.md`

**关于 tools 的口径：**
- 按 `README` 当前 MCP 表中“明确写出数量”的条目做保守统计，已能确认 `1224+` 个 tools / endpoints
- 实际总数高于这个数字，因为仍有不少 MCP 没在文档中写出完整 tool 数

**可讲稿：**
- 对外汇报建议采用 `101 skills / 46 MCP integrations / 1224+ tools(endpoints)` 这组数字。
- 如果面对技术评审，可以补一句：当前仓库清单已继续扩展，文档口径正在追平。

---

## Slide 11. NetClaw 是多少人构建的

**基于 git 提交作者身份的保守判断：**
- 当前仓库可见 `6` 个 author identity
- 如果把 `automateyournetwork` 与 `John Capobianco` 视为同一核心作者，则约为 `5` 位实际贡献者

**团队结构特征：**
- 明显是“1 位核心作者主导 + 多位专项贡献者协同”的建设模式
- 核心作者贡献占比非常高

**建议 PPT 讲法：**
- `这是一个由核心作者主导、多人协作扩展的项目，当前从 git 历史可见约 5~6 位贡献者。`

---

## Slide 12. 总结页

**一句话总结：**
NetClaw 的价值，不只是把很多 MCP 接进来，而是把“监控、排障、变更、验证、通知、审计”组织成一个可执行、可治理、可复用的网络工程流程。

**结束金句：**
- 对工程师：少切系统，少做重复劳动
- 对团队：把专家经验沉淀成 Skill
- 对管理者：让 AI 可审计、可治理、可规模化

---

## 附：演讲时可直接引用的数据口径

- 项目定位：CCIE-level AI network engineering coworker
- 主口径：`101 skills`、`46 MCP integrations`
- 当前仓库扩展清单：`55 MCP server entries`
- 当前工作区技能文件：`108 SKILL.md`
- 明确可数的 tools/endpoints：`1224+`
- Git 可见贡献者身份：`6` 个，折算约 `5~6` 位贡献者

## 附：一句话版本

NetClaw 是一个把网络设备、SoT、ITSM、可观测性和协同平台统一起来的 AI 网络工程同事，围绕真实业务流程把排障、变更、验证和审计做成闭环。
