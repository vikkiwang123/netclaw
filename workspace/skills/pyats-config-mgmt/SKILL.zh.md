---
name: pyats-config-mgmt
description: "网络变更管理：变更前基线、配置下发、变更后验证、回退流程与合规校验。在向设备推送配置、规划网络变更、回滚配置或运行合规检查时使用。"
user-invocable: true
metadata:
  { "openclaw": { "requires": { "bins": ["python3"], "env": ["PYATS_TESTBED_PATH"] } } }
---

# 配置管理

## 黄金准则

**必须先采集基线再下发配置。** 若变更失败，必须清楚回退到什么状态。

## 变更工作流

### 阶段 1：变更前基线

采集变更可能影响到的当前状态。

#### 1A：保存运行配置

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_show_running_config '{"device_name":"R1"}'
```

保存该输出 — 作为回退参照。

#### 1B：采集相关状态

按变更类型采集相应状态：

**接口变更：**
```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip interface brief"}'

PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show interfaces"}'
```

**路由变更：**
```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip route"}'

PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip ospf neighbor"}'

PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip bgp summary"}'
```

**ACL / 安全变更：**
```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip access-lists"}'
```

#### 1C：连通性基线

变更前对关键目标做 ping：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_ping_from_network_device '{"device_name":"R1","command":"ping 8.8.8.8 repeat 10"}'
```

### 阶段 2：规划变更

下发任何配置前须明确说明：
1. **将下发**哪些配置行
2. **每一行**的必要性
3. **预期**效果
4. **可能**出什么问题（风险评估）
5. **如何**验证成功
6. **失败时如何**回退

### 阶段 3：下发配置

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_configure_device '{"device_name":"R1","config_commands":["interface Loopback99","ip address 99.99.99.99 255.255.255.255","description NetClaw-Managed","no shutdown"]}'
```

**配置最佳实践：**
- 每次只做一类逻辑变更（不要批量无关变更）
- **不要**包含 `configure terminal` 或 `end` — 由工具处理
- 切换配置上下文时 **要** 包含 `exit`（例如退出接口）
- 接口与 route-map 使用清晰描述
- 复杂变更（route-map、ACL）先在对象层面拼完整再挂到接口

**常见配置模式：**

**接口：**
```json
["interface GigabitEthernet2", "description WAN-Link-to-ISP", "ip address 203.0.113.1 255.255.255.252", "no shutdown"]
```

**OSPF：**
```json
["router ospf 1", "router-id 1.1.1.1", "network 10.0.0.0 0.0.255.255 area 0", "passive-interface default", "no passive-interface GigabitEthernet1"]
```

**BGP：**
```json
["router bgp 65001", "neighbor 10.1.1.2 remote-as 65002", "neighbor 10.1.1.2 description ISP-Peer", "address-family ipv4 unicast", "neighbor 10.1.1.2 activate", "neighbor 10.1.1.2 route-map ISP-IN in", "neighbor 10.1.1.2 route-map ISP-OUT out", "exit-address-family"]
```

**ACL：**
```json
["ip access-list extended MGMT-ACCESS", "permit tcp 10.0.0.0 0.0.0.255 any eq 22", "permit tcp 10.0.0.0 0.0.0.255 any eq 443", "deny ip any any log"]
```

**Route-map：**
```json
["route-map ISP-IN permit 10", "match ip address prefix-list ALLOWED-IN", "set local-preference 200", "exit", "route-map ISP-IN deny 99"]
```

**静态路由：**
```json
["ip route 0.0.0.0 0.0.0.0 203.0.113.2 name DEFAULT-TO-ISP"]
```

**NTP：**
```json
["ntp server 10.0.0.1 prefer", "ntp server 10.0.0.2", "ntp source Loopback0"]
```

### 阶段 4：变更后验证

下发后立即验证：

#### 4A：检查日志错误

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_show_logging '{"device_name":"R1"}'
```

查找变更时间戳之后新出现的错误信息。

#### 4B：确认配置已生效

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_show_running_config '{"device_name":"R1"}'
```

与变更前配置对比，确认仅有预期变更。

#### 4C：验证预期状态

重跑阶段 1B 中的 show 命令并对比：
- 路由邻接是否仍正常？
- 预期新路由是否出现？
- 接口状态是否正确？
- ACL 计数器是否按预期增长？

#### 4D：连通性验证

对阶段 1C 中所有目标重新 ping：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_ping_from_network_device '{"device_name":"R1","command":"ping 8.8.8.8 repeat 10"}'
```

与基线对比成功率与 RTT。

### 阶段 5：回退（必要时）

若验证失败，通过反向配置回退：

**删除新增配置：**
```json
["no interface Loopback99"]
```

**恢复被改动的配置：**
应用阶段 1A 基线中保存的原始配置行。

**复杂回退：** 从保存的 running config 中应用整个相关段落。

回退后再次验证设备是否回到基线状态。

## 变更文档

每次变更后输出变更报告：

```
Change Report — YYYY-MM-DD HH:MM UTC
Device: R1 (devnetsandboxiosxec8k.cisco.com)
Requestor: [who requested the change]

Change Description:
  Added Loopback99 (99.99.99.99/32) for OSPF router-id migration

Config Applied:
  interface Loopback99
   ip address 99.99.99.99 255.255.255.255
   description OSPF-RID-Migration
   no shutdown

Pre-Change State:
  - Routing table: 47 routes
  - OSPF neighbors: 2 (FULL)
  - Connectivity: 100% to 8.8.8.8

Post-Change State:
  - Routing table: 48 routes (+1 connected 99.99.99.99/32)
  - OSPF neighbors: 2 (FULL) — no change
  - Connectivity: 100% to 8.8.8.8 — no change
  - New log entries: %LINEPROTO-5-UPDOWN: Loopback99 up/up

Verification: PASSED
Rollback Required: No
```

## 合规模板

### 最低安全基线（每台新设备建议应用）

```json
[
  "service timestamps debug datetime msec localtime",
  "service timestamps log datetime msec localtime",
  "service password-encryption",
  "no ip source-route",
  "no ip http server",
  "ip http secure-server",
  "ip ssh version 2",
  "ip ssh time-out 60",
  "ip ssh authentication-retries 3",
  "login on-failure log",
  "login on-success log",
  "banner login ^ Authorized access only. All activity is monitored. ^"
]
```

### VTY 加固

```json
[
  "line vty 0 4",
  "transport input ssh",
  "exec-timeout 15 0",
  "login local",
  "exit",
  "line vty 5 15",
  "transport input ssh",
  "exec-timeout 15 0",
  "login local"
]
```

## 与 ServiceNow 变更请求集成（MISSION02 增强）

当可用 ServiceNow（已设置 `$SERVICENOW_MCP_SCRIPT`）时，**每次**配置变更必须由已批准的变更请求（CR）门禁。

### 变更前：创建 CR

任何配置推送前先创建变更请求：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" create_change_request '{"short_description":"Configure SSH hardening on R1","description":"Apply VTY line hardening: SSH-only transport, exec timeout 15 min, login local. Affects R1 management plane.","category":"Network","priority":"3","risk":"low","impact":"low"}'
```

### 提交审批

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" submit_change_for_approval '{"change_number":"CHG0012345"}'
```

### 审批门禁

继续前检查 CR 是否已批准：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" get_change_request_details '{"change_number":"CHG0012345"}'
```

若状态不是「已批准」，**停止**，告知人工并等待。

### 变更前检查：无未关闭 P1/P2

确认受影响 CI 上无未关闭的 P1/P2 事件：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" list_incidents '{"urgency":"1","state":"open"}'
```

若受影响设备存在 P1/P2，**不要**继续，上报人工。

### 变更后：关闭或升级 CR

验证通过时：
```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" update_change_request '{"change_number":"CHG0012345","updates":{"state":"closed","close_code":"successful","close_notes":"Change applied and verified. Post-change baseline matches expected state."}}'
```

验证失败时：
```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" update_change_request '{"change_number":"CHG0012345","updates":{"state":"review","close_notes":"Post-change verification FAILED. Rollback initiated. Human review required."}}'
```

### 紧急变更

紧急变更（网络中断、安全事件）时：
1. 立即通知人工
2. 创建类别为 Emergency 的 CR
3. 执行变更（可绕过审批门禁）
4. 须在 24 小时内补批

## GAIT 审计轨迹

将变更各阶段记入 GAIT：

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"input":{"role":"assistant","content":"Config change on R1: Phase 1 baseline captured. Phase 2 plan approved. Phase 3 config applied. Phase 4 verification PASSED. ServiceNow CR CHG0012345 closed successful.","artifacts":[]}}'
```

五阶段工作流配合 GAIT 形成不可篡改记录：
1. **基线** → GAIT 提交变更前状态
2. **规划** → GAIT 提交变更方案与 CR 编号
3. **下发** → GAIT 提交实际推送命令
4. **验证** → GAIT 提交变更后状态与差异
5. **文档** → GAIT 提交最终摘要与 CR 关闭
