---
name: gait-session-tracking
description: "GAIT 会话生命周期管理：创建分支、记录轮次、为每次 NetClaw 操作保留审计日志。在开启新会话、记录健康检查或配置变更、固定变更前基线、或查看排障会话审计轨迹时使用。"
user-invocable: true
metadata:
  { "openclaw": { "requires": { "bins": ["python3"], "env": ["GAIT_MCP_SCRIPT"] } } }
---

# GAIT 会话跟踪

**本 skill 为强制要求。** 每个 NetClaw 会话必须以 `gait_branch` 开始，以 `gait_log` 结束。

## 如何调用工具

GAIT MCP 提供 9 个工具，通过 mcp-call 调用：

### 检查仓库状态

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_status '{}'
```

返回当前分支、未提交变更与仓库状态。

### 初始化新 GAIT 仓库

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_init '{}'
```

若尚不存在则创建 GAIT 仓库。在 NetClaw 初次搭建时执行一次。

### 创建新分支（会话开始）

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_branch '{"branch_name":"health-check-r1-2026-02-21"}'
```

**每个会话从这里开始。** 分支名应包含动作类型、目标设备与日期，例如：
- `health-check-r1-2026-02-21`
- `ospf-troubleshoot-core-2026-02-21`
- `config-deploy-acl-update-2026-02-21`
- `netbox-reconcile-site-hq-2026-02-21`
- `security-audit-dmz-2026-02-21`

### 切换到已有分支

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_checkout '{"branch_name":"health-check-r1-2026-02-21"}'
```

用于恢复先前会话或在并行排查间切换上下文。

### 记录一轮 AI 交互（主要记录手段）

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"prompt":"User asked to check CPU on R1","response":"Ran show processes cpu sorted. CPU 5-min avg: 12%. Status: HEALTHY.","artifacts":["show_proc_cpu_r1.txt"]}'
```

**每次重要动作后记录一轮。** 每轮包含：
- **prompt**：用户请求或触发动作的原因
- **response**：采集的数据与结论
- **artifacts**：产出文件列表（可选）

### 查看提交历史（会话结束）

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_log '{}'
```

**每个会话在这里结束。** 结束前展示完整审计日志，供用户留存全过程记录。

### 查看某次提交详情

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_show '{"commit_ref":"HEAD"}'
```

检查单次提交全文。可使用 `HEAD`、`HEAD~1` 或提交哈希。

### 固定重要提交

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_pin '{"commit_ref":"HEAD","label":"pre-change-baseline"}'
```

标记会话中的关键时刻以便日后查找。常用标签：
- `pre-change-baseline` — 任何修改前的状态
- `post-change-verified` — 变更验证通过后的状态
- `critical-finding` — 排查中的重要发现
- `rollback-point` — 需要时可回退的安全状态

### 汇总并压缩多轮记录

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_summarize_and_squash '{}'
```

将多轮细粒度记录合并为一条摘要提交。长会话结束时用于生成简洁审计记录。

## 强制会话生命周期

每个 NetClaw 会话严格按以下顺序：

### 1. 会话开始 — 创建分支

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_branch '{"branch_name":"ACTION-TYPE-TARGET-DATE"}'
```

### 2. 会话进行中 — 记录每一轮

每次有意义动作后（show、配置变更、API 调用、验证）记录一轮：

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"prompt":"WHAT_WAS_ASKED","response":"WHAT_DATA_COLLECTED_AND_WHAT_CHANGED_AND_VERIFICATION_RESULT","artifacts":[]}'
```

**记录格式建议：**
- **prompt**：清楚说明用户请求或触发原因
- **response** 包含三部分：
  1. 采集了什么（执行的命令、API 响应）
  2. 改变了什么（下发的配置、创建的工单、NetBox 更新）
  3. 验证结果（HEALTHY/WARNING/CRITICAL、通过/失败、前后差异）
- **artifacts**：生成的文件（日志、配置、图表、报告）

### 3. 会话结束 — 展示日志

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_log '{}'
```

始终向用户展示会话日志，形成完整记录。

## 按 skill 类型的记录示例

### 健康检查轮次

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"prompt":"Run full health check on R1","response":"Collected: show version, show processes cpu sorted, show processes memory sorted, show ip interface brief, show interfaces, show ntp associations, show logging. Results: CPU 12% HEALTHY, Memory 45% HEALTHY, Interfaces 4/5 up WARNING (Gi2 down), NTP synced HEALTHY, no critical log patterns. Overall: WARNING.","artifacts":["health-report-r1.txt"]}'
```

### 配置变更轮次

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"prompt":"Apply ACL update to block 192.168.50.0/24 on R1 Gi1","response":"Pre-change: captured running-config. Applied: ip access-list extended BLOCK-LIST, permit/deny entries. Post-change: verified ACL in show access-lists, tested with ping from blocked subnet -- dropped as expected. Change verified successfully.","artifacts":["pre-change-config-r1.txt","post-change-config-r1.txt","acl-diff.txt"]}'
```

### 排障轮次

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"prompt":"Investigate OSPF adjacency failure between R1 and R3","response":"Checked show ip ospf neighbor on R1 -- R3 missing. Checked show ip ospf interface on both -- area mismatch: R1 area 0, R3 area 1 on shared link. Root cause identified: area misconfiguration on R3 Gi0/1.","artifacts":["ospf-neighbor-r1.txt","ospf-interface-r1.txt","ospf-interface-r3.txt"]}'
```

### NetBox 对账轮次

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"prompt":"Reconcile R1 interfaces against NetBox","response":"Live state: 5 interfaces discovered via show ip interface brief. NetBox state: 4 interfaces documented. Drift detected: Gi5 exists on device but missing from NetBox. Gi2 documented in NetBox but admin-down on device. Reconciliation report generated.","artifacts":["reconcile-report-r1.json"]}'
```

## 与所有其他 skill 集成

GAIT 会话跟踪供其余 NetClaw skill 满足审计合规：

- **pyats-health-check** — 记录各健康检查步骤与总体结果
- **pyats-topology** — 记录发现的邻居与拓扑变化
- **pyats-security** — 按严重度记录审计发现
- **pyats-config-mgmt** — 记录变更前基线、已下发变更、变更后验证
- **pyats-troubleshoot** — 记录排查步骤与根因
- **pyats-routing** — 记录路由表快照与协议状态
- **netbox-reconcile** — 记录漂移检测与修复动作
- **drawio-diagram** — 记录图表生成与源数据引用
- **markmap-viz** — 记录思维导图与底层数据
- **rfc-lookup** — 记录调查引用的 RFC
- **wikipedia-research** — 记录协议调研上下文
- **servicenow-incidents** — 记录工单创建与更新
- **nvd-cve** — 记录漏洞发现与修复跟踪

## 何时使用

始终。每个 NetClaw 会话。无例外。这是系统的审计主干。
