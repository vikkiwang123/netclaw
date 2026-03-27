# NetClaw 12页技术汇报版

## 第1页：标题页

**标题**
NetClaw：面向企业网络运维与变更治理的 AI 工程协作体

**副标题**
基于 OpenClaw + Skills + MCP 的网络自动化执行框架

**讲述重点**
- NetClaw 不是单一问答机器人，而是可执行网络工程流程的 AI coworker
- 定位是把“人、流程、工具、审计”统一到一个操作平面

---

## 第2页：一句话定义与定位

**一句话定义**
NetClaw 是一个 CCIE 级别的网络工程 AI，同步具备排障、变更、审计、可观测性、可视化和协同交付能力。

**技术定位**
- 上层：Slack / WebEx / WebChat 等交互入口
- 中层：Skills 负责方法论与 SOP 编排
- 下层：MCP Servers 负责实际系统和平台能力接入

**关键区别**
- 不是“把很多 API 接进来”
- 而是“把工程方法固化为技能，再由技能调用工具完成闭环”

---

## 第3页：总体架构

**架构分层**
1. 交互层
   - Slack
   - WebEx
   - WebChat / TUI

2. Agent 层
   - OpenClaw Agent
   - NetClaw 角色设定
   - 会话上下文、记忆、规则注入

3. Skill 编排层
   - 排障技能
   - 变更技能
   - SoT 对账技能
   - 可观测性技能
   - 可视化与协同交付技能

4. MCP 接入层
   - pyATS
   - NetBox / Nautobot / Infrahub
   - ServiceNow
   - Grafana / Prometheus / SuzieQ
   - gNMI
   - GitHub
   - Teams / SharePoint / Visio
   - ACI / ISE / F5 / Meraki / SD-WAN / AWS / GCP 等

**讲述重点**
- Skill 决定“怎么干”
- MCP 决定“能连谁、能做什么”

---

## 第4页：为什么需要 NetClaw

**现网常见痛点**
- 网络工程师需要在 CLI、监控、CMDB、ITSM、文档平台之间来回切换
- 故障定位依赖个人经验，标准化程度不高
- 变更流程分散，缺少统一 pre-check / post-check / 审计记录
- SoT 与设备实际状态持续漂移
- 多域协作复杂：传统网络、云网络、K8s、SD-WAN、F5、Meraki、可观测性平台彼此割裂

**NetClaw 的切入点**
- 统一入口
- 统一方法论
- 统一治理
- 统一审计

---

## 第5页：业务流程案例

**案例主题**
某分支站点访问总部 ERP 变慢，需要快速判断是链路、路由、策略、还是近期变更引起。

**目标**
- 快速收集现状
- 交叉验证路径、协议和资源状态
- 如果需要，发起受控变更并验证
- 输出可审计结果

**传统处理方式**
- 人工登录设备查接口、BGP、OSPF、日志
- 再去 Grafana、NetBox、ServiceNow、GitHub、Teams 整理信息

**NetClaw 处理方式**
- 在聊天入口发起自然语言请求
- 由 Skills 自动串联多个 MCP 完成整条链路

---

## 第6页：流程拆解一：排障阶段

**输入示例**
`帮我排查 Site-B 到总部 ERP 访问变慢`

**第一阶段动作**
- `pyats-network`：采集接口、日志、show 命令结果
- `pyats-routing`：检查 BGP/OSPF 邻接、路由选择和路径变化
- `pyats-parallel-ops`：并行拉取多设备状态
- `gtrace-path-analysis`：看链路路径、抖动、丢包、ECMP、MPLS/NAT
- `te-path-analysis`：从外部/多探针视角验证路径质量
- `grafana-observability` / `prometheus-monitoring`：查看时序指标和告警
- `suzieq-observability`：看网络状态快照与断言

**产出**
- 当前故障范围
- 可能影响点
- 初步根因候选集

---

## 第7页：流程拆解二：校验与定界

**第二阶段动作**
- `netbox-reconcile`：对比设备真实状态与 NetBox 设计意图
- `nautobot-sot`：如果用 Nautobot，做 IP/Prefix/VRF 维度校验
- `infrahub-sot`：如果是 schema-driven SoT，做关系型对象追踪
- `nvd-cve`：必要时检查版本脆弱性风险
- `rfc-lookup`：需要解释协议行为时引用标准依据

**这一阶段回答的问题**
- 当前状态是不是本来就偏离设计
- 路由/接口/链路/地址是否发生漂移
- 是配置偏差、路径异常，还是平台侧资源瓶颈

**产出**
- 更清晰的根因判断
- 是否需要变更
- 是否需要升级告警或转入事件管理

---

## 第8页：流程拆解三：变更执行与验证

**第三阶段动作**
- `servicenow-change-workflow`
  - 检查相关 CI 是否存在 P1/P2
  - 创建或核查 CR
  - 等待审批状态进入 Implement

- `pyats-config-mgmt` 或 `gnmi-telemetry`
  - 采集 pre-change baseline
  - 执行配置变更
  - 做 post-change verification

- `gait-session-tracking`
  - 记录会话、动作、结论、产物

**NetClaw 的治理特点**
- 先查现状，再做变更
- 没有 CR 不执行正式变更
- 变更后必须验证

---

## 第9页：这一流程里用到的核心 Skills

**排障与状态采集**
- `pyats-network`
- `pyats-routing`
- `pyats-troubleshoot`
- `pyats-parallel-ops`

**路径与可观测性**
- `gtrace-path-analysis`
- `te-path-analysis`
- `grafana-observability`
- `prometheus-monitoring`
- `suzieq-observability`

**SoT 与治理**
- `netbox-reconcile`
- `nautobot-sot`
- `servicenow-change-workflow`
- `gait-session-tracking`

**展示与交付**
- `canvas-network-viz`
- `drawio-diagram`
- `slack-report-delivery`
- `msgraph-teams`
- `github-ops`

**讲述重点**
- Skills 让 AI 不只是“能调用工具”，而是“会按网络工程方法论做事”

---

## 第10页：这一流程里用到的 MCP / Tools

**典型 MCP**
- `pyATS MCP`
- `ServiceNow MCP`
- `NetBox MCP`
- `Nautobot MCP`
- `Grafana MCP`
- `Prometheus MCP`
- `SuzieQ MCP`
- `gNMI MCP`
- `gtrace MCP`
- `ThousandEyes MCP`
- `GitHub MCP`
- `Microsoft Graph MCP`

**这些 tools 在流程中的职责**
- 设备状态采集
- 指标和日志查询
- 路径探测
- 源数据对账
- 变更审批
- 配置下发
- 审计记录
- 报告和通知分发

**推荐讲法**
- NetClaw 的竞争力不在某一个 MCP，而在“跨 MCP 编排能力”

---

## 第11页：NetClaw 的核心技术亮点

**1. Skill-first**
- 把排障、变更、审计的专家经验沉淀为可复用技能

**2. Real-state-first**
- 强调“Never guess device state”
- 一切结论建立在实时采集之上

**3. Governance-by-design**
- ServiceNow CR
- baseline
- verification
- escalation / rollback 思路

**4. Audit-by-default**
- GAIT 全程记录 AI 做过的动作
- 回答“AI 做了什么、为什么做、产出了什么”

**5. Multi-domain integration**
- 网络设备
- 负载均衡
- SoT
- ITSM
- 可观测性
- 云与 K8s
- 协同平台

---

## 第12页：规模、团队与价值总结

**当前可用于汇报的规模口径**
- 主口径：`101 skills`
- 主口径：`46 MCP integrations`
- 当前 `README` MCP 清单已扩展到 `55` 个 server 条目
- 当前工作区中有 `108` 个 `SKILL.md`
- 按文档里明确可数的条目保守统计：`1224+ tools/endpoints`

**团队规模**
- 从 git 历史可见 `6` 个 author identity
- 如果合并同一核心作者的不同身份，约 `5~6` 位贡献者
- 结构上属于“核心作者主导 + 多位协作者扩展”

**NetClaw 解决的业务问题**
- 缩短故障定位时间
- 提高变更标准化和成功率
- 降低多人协作成本
- 提升 AI 可审计、可治理、可规模化落地能力

**结束语**
NetClaw 的价值，不在于“接入了多少工具”，而在于“把网络工程流程真正做成了可执行闭环”。
