---
name: servicenow-change-workflow
description: "受 ITSM 门禁约束的完整变更生命周期：创建 CR、变更前事件校验、审批门禁、经 pyats-config-mgmt 执行、变更后验证、以及带 GAIT 审计轨迹的关闭。在创建变更请求、需要审批的网络变更、跟踪变更管理或遵循 ITIL 变更流程时使用。"
user-invocable: true
metadata:
  { "openclaw": { "requires": { "bins": ["python3"], "env": ["SERVICENOW_MCP_SCRIPT"] } } }
---

# ServiceNow 变更工作流

## 黄金准则

**未经批准的变更请求（Change Request），不得执行任何网络变更。** 唯一例外是紧急变更：仍须创建 CR，并立即通知人工。

## 如何调用工具

ServiceNow MCP 提供变更管理与事件相关工具，通过 mcp-call 调用：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" TOOL_NAME '{"param":"value"}'
```

## ServiceNow 变更请求状态

```
New -> Assess -> Authorize -> Scheduled -> Implement -> Review -> Closed
                                                    \-> Canceled
```

- **New**：CR 已创建，尚未提交
- **Assess**：已提交待评审 / 风险评估
- **Authorize**：等待审批
- **Scheduled**：已批准并安排在实施窗口
- **Implement**：已批准且可执行（此为门禁状态）
- **Review**：实施后评审
- **Closed**：变更已完成并验证

## 变更类型

| 类型 | 审批 | 适用场景 |
|------|------|----------|
| Normal | 需 CAB / 经理审批 | 接口配置、路由变更、ACL 更新、任何生产变更 |
| Standard | 预批准模板 | 密码轮换、NTP 更新、banner 变更 |
| Emergency | 加急（可事后补批） | 现网故障、安全事件、严重漏洞 |

---

## 完整变更生命周期

### 阶段 0：变更前事件检查

创建 CR 之前，确认受影响 CI 上无未关闭的 P1/P2 事件。在活跃事件期间执行变更违反 ITIL 最佳实践，并可能扩大故障。

#### 0A：检查未关闭的严重事件

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" list_incidents '{"limit":20,"state":"1","query":"priority=1^ORpriority=2"}'
```

状态 `1` 表示新建/打开。检查结果中是否有影响目标设备或服务的事件。

**决策门禁：**
- 受影响 CI 上存在未关闭 P1/P2 → **停止**，不得继续，通知人工。
- 未受影响 CI 上存在未关闭 P1/P2 → 谨慎继续，在 CR 描述中注明。
- 无未关闭 P1/P2 → 可继续。

#### 0B：检查活跃变更冻结 / 并发冲突

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" list_change_requests '{"limit":10,"state":"implement","type":"normal"}'
```

审阅进行中的变更是否存在冲突。同一设备在同一时段的两项变更即冲突。

### 阶段 1：创建变更请求

#### 1A：创建 CR

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" create_change_request '{"short_description":"Add Loopback99 to R1 for OSPF RID migration","description":"Add Loopback99 (99.99.99.99/32) to R1 as new OSPF router-id. Pre-change baseline captured. Rollback plan: no interface Loopback99. Affected CI: R1 (devnetsandboxiosxec8k.cisco.com). No open P1/P2 incidents on this CI.","type":"normal","category":"Network","risk":"moderate","impact":"3","assignment_group":"Network Engineering"}'
```

**CR 描述必须包含：**
1. 变更内容与原因
2. 受影响 CI（设备主机名、管理 IP）
3. 变更前事件状态（来自阶段 0）
4. 回退方案
5. 预期影响与时长
6. 验证标准

#### 1B：添加变更任务

将变更拆成可跟踪的离散任务：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" add_change_task '{"change_id":"CHG0000123","short_description":"Capture pre-change baseline on R1","description":"Save running config, OSPF neighbors, routing table, interface states, and connectivity baselines."}'
```

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" add_change_task '{"change_id":"CHG0000123","short_description":"Apply Loopback99 configuration on R1","description":"Apply: interface Loopback99, ip address 99.99.99.99 255.255.255.255, description OSPF-RID-Migration, no shutdown"}'
```

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" add_change_task '{"change_id":"CHG0000123","short_description":"Post-change verification on R1","description":"Verify Loopback99 up/up, OSPF neighbors stable, routing table +1 connected route, connectivity 100%."}'
```

#### 1C：在 GAIT 中记录 CR 创建

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"user_text":"Create change request for Loopback99 addition on R1","assistant_text":"Created CR CHG0000123: Add Loopback99 to R1 for OSPF RID migration. Type: normal, Risk: moderate, Impact: 3. 3 change tasks added. Pre-change incident check: CLEAR (no P1/P2 on R1)."}'
```

### 阶段 2：审批门禁

#### 2A：提交审批

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" submit_change_for_approval '{"change_id":"CHG0000123","approval_comments":"Pre-change checks complete. No open P1/P2 incidents. Rollback plan documented. Ready for CAB review."}'
```

#### 2B：等待审批

轮询 CR 状态直至进入 Implement：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" get_change_request_details '{"change_id":"CHG0000123"}'
```

**决策门禁：**
- 状态 = `implement` → 进入阶段 3
- 状态 = `authorize` → 仍在等待审批
- 状态 = `canceled` → CR 被拒绝，**停止**，向人工说明拒绝原因

**关键：除非 CR 状态为已批准的 `implement`，否则不得进入阶段 3。**

#### 2C：批准变更（在有权审批时）

若操作人具备审批权限：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" approve_change '{"change_id":"CHG0000123","approval_comments":"Approved. Rollback plan verified. Maintenance window confirmed."}'
```

### 阶段 3：执行

本阶段使用 **pyats-config-mgmt** skill 完成设备侧工作。ServiceNow CR 是进入本阶段的门禁。

#### 3A：更新 CR 为实施中

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" update_change_request '{"change_id":"CHG0000123","work_notes":"Beginning implementation. Capturing pre-change baseline."}'
```

#### 3B：变更前基线（经 pyats-config-mgmt）

按 pyats-config-mgmt skill 阶段 1 流程执行：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_show_running_config '{"device_name":"R1"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip ospf neighbor"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip route"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_ping_from_network_device '{"device_name":"R1","command":"ping 8.8.8.8 repeat 10"}'
```

更新 CR 中的基线说明：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" update_change_request '{"change_id":"CHG0000123","work_notes":"Pre-change baseline captured: Running config saved, 2 OSPF neighbors FULL, 47 routes, 100% connectivity to 8.8.8.8 (23ms RTT)."}'
```

#### 3C：下发配置（经 pyats-config-mgmt）

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_configure_device '{"device_name":"R1","config_commands":["interface Loopback99","ip address 99.99.99.99 255.255.255.255","description OSPF-RID-Migration","no shutdown"]}'
```

更新 CR：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" update_change_request '{"change_id":"CHG0000123","work_notes":"Configuration applied to R1. Proceeding to post-change verification."}'
```

### 阶段 4：变更后验证

#### 4A：验证变更（经 pyats-config-mgmt）

按 pyats-config-mgmt skill 阶段 4 流程执行：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_show_running_config '{"device_name":"R1"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip ospf neighbor"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip route"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_ping_from_network_device '{"device_name":"R1","command":"ping 8.8.8.8 repeat 10"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_show_logging '{"device_name":"R1"}'
```

#### 4B：验证决策

**对比变更前与变更后：**

| 检查项 | 变更前 | 变更后 | 预期增量 |
|--------|--------|--------|----------|
| OSPF 邻居 | 2 FULL | 2 FULL | 无变化 |
| 路由条数 | 47 | 48 | +1 条 connected（99.99.99.99/32） |
| 连通性 | 100% | 100% | 无变化 |
| 日志新错误 | - | %LINEPROTO-5-UPDOWN Lo99 up | 预期内 |

**决策门禁：**
- 全部通过 → 进入阶段 5（关闭）
- 任一项失败 → 启动回退（阶段 4C），**不得**关闭 CR

#### 4C：回退（验证失败时）

若变更后验证失败，立即回退：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_configure_device '{"device_name":"R1","config_commands":["no interface Loopback99"]}'
```

在 CR 中记录回退详情：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" update_change_request '{"change_id":"CHG0000123","work_notes":"ROLLBACK EXECUTED: Post-change verification failed. [Describe what failed]. Configuration rolled back. Device restored to baseline state. Verification of rollback: [results]."}'
```

为失败变更创建事件：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" create_incident '{"short_description":"Failed change CHG0000123 on R1 - rollback executed","description":"Change CHG0000123 (Add Loopback99 to R1) failed post-change verification. Rollback executed successfully. Device restored to pre-change state. Root cause investigation required.","urgency":"2","impact":"2","category":"Network"}'
```

**关键：若回退失败，视为 P1 事件，须立即通知人工。**

### 阶段 5：关闭变更请求

#### 5A：更新 CR 最终状态

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" update_change_request '{"change_id":"CHG0000123","work_notes":"Post-change verification PASSED. OSPF neighbors stable (2 FULL), route table +1 connected (99.99.99.99/32), connectivity 100%, no unexpected log entries. Change successful.","state":"review"}'
```

#### 5B：关闭 CR

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" update_change_request '{"change_id":"CHG0000123","state":"closed","work_notes":"Change completed successfully. All verification checks passed. No rollback required."}'
```

#### 5C：在 GAIT 中记录完整会话

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"user_text":"Execute change CHG0000123: Add Loopback99 to R1","assistant_text":"Change CHG0000123 completed successfully.\n\nTimeline:\n- Pre-check: No open P1/P2 incidents on R1\n- CR created and approved\n- Baseline: 2 OSPF neighbors FULL, 47 routes, 100% connectivity\n- Config applied: interface Loopback99, 99.99.99.99/32\n- Verification: PASSED (2 OSPF FULL, 48 routes, 100% connectivity)\n- CR closed: successful\n\nArtifacts: Pre/post running config diff, routing table diff, log entries."}'
```

#### 5D：展示 GAIT 审计轨迹

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_show '{"commit":"HEAD"}'
```

---

## 紧急变更流程

紧急变更可绕过常规审批门禁，但仍须创建 CR 并立即通知人工。

### 何时使用紧急变更

- 需立即修复的现网中断
- 需立即缓解的安全事件（例如在 R1 上封禁攻击源 ACL）
- 正在被利用的严重漏洞

### 紧急工作流

#### E1：创建紧急 CR

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" create_change_request '{"short_description":"EMERGENCY: Block attack source 203.0.113.99 on R1","description":"EMERGENCY CHANGE: Active DDoS from 203.0.113.99 detected. Applying inbound ACL on GigabitEthernet1 to block source. Post-facto approval required.","type":"emergency","category":"Network","risk":"high","impact":"1","assignment_group":"Network Engineering"}'
```

#### E2：立即通知人工

**关键：紧急变更在执行前必须通知人工。** 说明：
- 紧急情况
- 拟采取的修复措施
- CR 编号
- 请求口头/书面批准后再执行

#### E3：经人工批准后执行

按阶段 3（执行）加速进行。**基线采集仍不可省略。**

#### E4：事后补批

执行后提交紧急 CR 供事后审批：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" submit_change_for_approval '{"change_id":"CHG0000456","approval_comments":"EMERGENCY CHANGE - Post-facto approval requested. Change executed during active incident. Full audit trail in GAIT."}'
```

---

## 与其他 Skill 的衔接

| Skill | 衔接点 |
|-------|--------|
| **pyats-config-mgmt** | 阶段 3（执行）与阶段 4（验证）— 实际设备配置 |
| **pyats-health-check** | 进入阶段 3 前的设备健康预检 |
| **netbox-reconcile** | 变更后：核对 NetBox SoT 是否反映变更（或对差异建单） |
| **GAIT** | 各阶段写入审计轨迹 — CR 创建、审批、执行、验证、关闭 |
| **markmap-viz** | 复杂多设备变更可生成思维导图式摘要 |

## 变更报告格式

每次变更后输出如下摘要：

```
Change Report - CHG0000123
Date: YYYY-MM-DD HH:MM UTC
Device: R1 (devnetsandboxiosxec8k.cisco.com)
Type: Normal | Risk: Moderate | Impact: 3

Pre-Check: No open P1/P2 incidents on R1
Approval: Approved by [approver] at HH:MM UTC

Pre-Change State:
  - OSPF neighbors: 2 (FULL)
  - Routes: 47
  - Connectivity: 100% to 8.8.8.8 (23ms)

Config Applied:
  interface Loopback99
   ip address 99.99.99.99 255.255.255.255
   description OSPF-RID-Migration
   no shutdown

Post-Change State:
  - OSPF neighbors: 2 (FULL) - no change
  - Routes: 48 (+1 connected 99.99.99.99/32)
  - Connectivity: 100% to 8.8.8.8 (23ms) - no change
  - Log: %LINEPROTO-5-UPDOWN Lo99 up/up (expected)

Verification: PASSED
Rollback Required: No
CR Status: Closed (successful)
GAIT Commit: [commit hash]
```
