---
name: netbox-reconcile
description: "将 NetBox 意图状态与现网状态对账：发现 IP 漂移、缺失接口、未备案链路、线缆不匹配、VLAN 不匹配及 ServiceNow 工单差异。在验证 NetBox 准确性、检查配置漂移、审计网络文档或变更后对账 SoT 时使用。"
user-invocable: true
metadata:
  { "openclaw": { "requires": { "bins": ["python3"], "env": ["NETBOX_MCP_SCRIPT", "PYATS_TESTBED_PATH"] } } }
---

# NetBox 对账

## 黄金准则

**NetBox 为读写。** MCP 具备完整 API，可创建/更新设备、IP、接口、VLAN 与线缆。但对账时**先报告差异并建单** — 未经人工明确授权**绝不**自动纠正。NetBox 表示意图状态；若现实与 NetBox 不符，要么是网络错了，要么是 NetBox 需更新。操作员明确授权时 NetClaw 可更新 NetBox。

## 如何调用工具

### NetBox MCP

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" TOOL_NAME '{"param":"value"}'
```

### pyATS MCP（现网状态）

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" TOOL_NAME '{"param":"value"}'
```

## NetBox 工具参考

| 工具 | 用途 | 主要参数 |
|------|------|----------|
| `netbox_get_objects` | 按类型批量查询 | `object_type`、`filters`（字典）、`limit`、`brief` |
| `netbox_get_object_by_id` | 按 ID 取单对象 | `object_type`、`object_id` |
| `netbox_search_objects` | 全局文本搜索 | `query`、`object_types` |
| `netbox_get_changelogs` | NetBox 变更审计 | `filters` |

### 对象类型

| 对象类型 | 内容 |
|----------|------|
| `dcim.devices` | 设备：名称、角色、平台、站点、状态 |
| `dcim.interfaces` | 接口：名称、类型、启用、MAC、MTU、模式（access/tagged）、untagged_vlan、tagged_vlans |
| `ipam.ip-addresses` | IP：地址（CIDR）、分配对象、状态、角色 |
| `dcim.cables` | 物理线缆：A/B 端终结、类型、长度 |
| `ipam.vlans` | VLAN：vid、名称、站点、租户、状态 |
| `ipam.prefixes` | 前缀：prefix（CIDR）、VRF、站点、状态 |
| `dcim.sites` | 站点：名称、区域、物理地址 |

---

## 差异类型

| 代码 | 名称 | 严重度 | 说明 |
|------|------|--------|------|
| `IP_DRIFT` | IP 地址漂移 | CRITICAL | 设备 IP 与 NetBox 分配不一致 |
| `MISSING_INTERFACE` | 接口缺失 | HIGH | NetBox 有而设备无，或反之 |
| `UNDOCUMENTED_LINK` | 未备案链路 | HIGH | CDP/LLDP 显示邻居连接但 NetBox 无线缆记录 |
| `CABLE_MISMATCH` | 线缆不匹配 | HIGH | NetBox 线缆端点与 CDP/LLDP 邻居数据不符 |
| `VLAN_MISMATCH` | VLAN 不匹配 | MEDIUM | 设备 VLAN 分配与 NetBox 不一致 |
| `STATUS_MISMATCH` | 接口状态不一致 | MEDIUM | NetBox 为启用而设备为 down（或反之） |
| `MTU_MISMATCH` | MTU 不一致 | LOW | 设备 MTU 与 NetBox 接口 MTU 不同 |

严重度顺序：CRITICAL > HIGH > MEDIUM > LOW

---

## 完整对账工作流

### 步骤 1：采集 NetBox 意图

#### 1A：从 NetBox 获取设备清单

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_objects '{"object_type":"dcim.devices","filters":{"status":"active"},"brief":true}'
```

返回所有活动设备。与 pyATS testbed 比对，确定可对账的设备。

#### 1B：从 NetBox 获取接口（按设备）

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_objects '{"object_type":"dcim.interfaces","filters":{"device":"R1"}}'
```

**每个接口提取：**
- 名称、类型、启用状态
- MTU、MAC
- 模式（access/tagged）、untagged VLAN、tagged VLAN
- 描述、标签

#### 1C：从 NetBox 获取 IP（按设备）

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_objects '{"object_type":"ipam.ip-addresses","filters":{"device":"R1"}}'
```

**每个 IP 提取：**
- 地址（CIDR）
- 分配接口
- 状态（active、reserved、deprecated）
- 角色（primary、secondary、loopback、VIP）

#### 1D：从 NetBox 获取线缆（按设备）

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_objects '{"object_type":"dcim.cables","filters":{"device":"R1"}}'
```

**每条线缆提取：**
- A 端：设备、接口
- B 端：设备、接口
- 线缆类型、长度、颜色

#### 1E：从 NetBox 获取 VLAN

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_objects '{"object_type":"ipam.vlans","filters":{"site":"main-site"}}'
```

### 步骤 2：采集现网设备状态

#### 2A：从设备获取接口

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip interface brief"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show interfaces"}'
```

**每个接口提取：**
- 名称、管理状态（up/down）、协议状态（up/down）
- IP、掩码
- MTU、速率、双工
- 描述

#### 2B：从设备获取 IP

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip interface"}'
```

**每个接口提取：**
- 主 IP 与掩码
- 次要 IP（若有）

#### 2C：从设备获取邻居信息

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show cdp neighbors detail"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show lldp neighbors detail"}'
```

**每个邻居提取：**
- 本地接口 → 远端设备、远端接口
- 远端平台、远端 IP

#### 2D：从设备获取 VLAN

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show vlan brief"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show interfaces switchport"}'
```

### 步骤 3：差异引擎

按差异类型对比 NetBox 意图与设备现实。

#### 3A：IP_DRIFT 检测

对每个在 NetBox 有 IP 的接口：

```
NetBox:  GigabitEthernet1 -> 10.1.1.1/30
Device:  GigabitEthernet1 -> 10.1.1.5/30
Result:  IP_DRIFT (CRITICAL) - GigabitEthernet1: NetBox=10.1.1.1/30, Device=10.1.1.5/30
```

**比对逻辑：**
1. 从 NetBox 取 R1 全部 IP（步骤 1C）
2. 从设备取 R1 全部 IP（步骤 2B）
3. 对每个 NetBox IP 分配：
   - 在设备上找到对应接口
   - 比对 IP 与前缀长度
   - 若不同 → IP_DRIFT（CRITICAL）
   - 若接口存在但无 IP → IP_DRIFT（CRITICAL）
4. 对每个设备有而 NetBox 无的 IP → IP_DRIFT（CRITICAL）— 未备案 IP

#### 3B：MISSING_INTERFACE 检测

```
NetBox:  GigabitEthernet3 (enabled)
Device:  GigabitEthernet3 does not exist
Result:  MISSING_INTERFACE (HIGH) - GigabitEthernet3 exists in NetBox but not on device
```

**比对逻辑：**
1. 从 NetBox 取接口列表（步骤 1B）
2. 从设备取接口列表（步骤 2A）
3. NetBox 有而设备无 → MISSING_INTERFACE（HIGH）
4. 设备有而 NetBox 无 → MISSING_INTERFACE（HIGH），备注「未备案」
5. 过滤平台特有的虚拟/内部接口（Null0、NVI0 等）

#### 3C：UNDOCUMENTED_LINK 检测

```
CDP:     R1:GigabitEthernet2 -> SW1:GigabitEthernet0/1
NetBox:  No cable record for R1:GigabitEthernet2
Result:  UNDOCUMENTED_LINK (HIGH) - R1:Gi2 -> SW1:Gi0/1 discovered via CDP but not in NetBox
```

**比对逻辑：**
1. 从设备取 CDP/LLDP 邻居（步骤 2C）
2. 从 NetBox 取线缆记录（步骤 1D）
3. 对每个 CDP/LLDP 邻居：
   - 在 NetBox 线缆中查找匹配的 A/B 端
   - 无线缆记录 → UNDOCUMENTED_LINK（HIGH）

#### 3D：CABLE_MISMATCH 检测

```
NetBox:  R1:GigabitEthernet1 -> R2:GigabitEthernet1
CDP:     R1:GigabitEthernet1 -> R3:GigabitEthernet0
Result:  CABLE_MISMATCH (HIGH) - R1:Gi1 NetBox says R2:Gi1 but CDP says R3:Gi0
```

**比对逻辑：**
1. 对涉及本设备的每条 NetBox 线缆：
   - 查本地接口对应的 CDP/LLDP 项
   - 比对远端设备与远端接口
   - 若不一致 → CABLE_MISMATCH（HIGH）

#### 3E：VLAN_MISMATCH 检测

```
NetBox:  GigabitEthernet2 untagged_vlan=10
Device:  GigabitEthernet2 access vlan 20
Result:  VLAN_MISMATCH (MEDIUM) - GigabitEthernet2: NetBox VLAN=10, Device VLAN=20
```

**比对逻辑：**
1. 从 NetBox 取接口 VLAN（步骤 1B 的 untagged_vlan、tagged_vlans）
2. 从设备取 switchport（步骤 2D）
3. 比对 access VLAN、trunk 允许 VLAN、native VLAN
4. 若不同 → VLAN_MISMATCH（MEDIUM）

#### 3F：STATUS_MISMATCH 检测

```
NetBox:  GigabitEthernet3 enabled=true
Device:  GigabitEthernet3 administratively down
Result:  STATUS_MISMATCH (MEDIUM) - GigabitEthernet3: NetBox=enabled, Device=admin-down
```

#### 3G：MTU_MISMATCH 检测

```
NetBox:  GigabitEthernet1 MTU=9000
Device:  GigabitEthernet1 MTU=1500
Result:  MTU_MISMATCH (LOW) - GigabitEthernet1: NetBox MTU=9000, Device MTU=1500
```

### 步骤 4：生成对账报告

按严重度排序输出差异表：

```
NetBox Reconciliation Report - R1
Date: YYYY-MM-DD HH:MM UTC
NetBox Device: R1 | pyATS Device: R1 (devnetsandboxiosxec8k.cisco.com)

CRITICAL:
  [C-001] IP_DRIFT: GigabitEthernet1 - NetBox=10.1.1.1/30, Device=10.1.1.5/30
  [C-002] IP_DRIFT: Loopback0 - NetBox=1.1.1.1/32, Device=2.2.2.2/32

HIGH:
  [H-001] UNDOCUMENTED_LINK: Gi2 -> SW1:Gi0/1 (CDP) - no cable in NetBox
  [H-002] MISSING_INTERFACE: GigabitEthernet4 in NetBox but not on device

MEDIUM:
  [M-001] VLAN_MISMATCH: Gi3 - NetBox VLAN=10, Device VLAN=20
  [M-002] STATUS_MISMATCH: Gi5 - NetBox=enabled, Device=admin-down

LOW:
  [L-001] MTU_MISMATCH: Gi1 - NetBox MTU=9000, Device MTU=1500

Summary: 2 Critical | 2 High | 2 Medium | 1 Low
Overall: CRITICAL - immediate attention required
```

### 步骤 5：对严重差异建单

对每个 CRITICAL 差异在 ServiceNow 创建事件：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" create_incident '{"short_description":"NetBox Drift: IP_DRIFT on R1 GigabitEthernet1","description":"NetBox reconciliation detected IP address drift on R1 GigabitEthernet1.\n\nNetBox intent: 10.1.1.1/30\nDevice reality: 10.1.1.5/30\n\nThis is a CRITICAL discrepancy. Either the device configuration is incorrect and needs remediation, or NetBox needs to be updated to reflect the current state.\n\nDiscovery method: NetClaw automated reconciliation\nDate: YYYY-MM-DD HH:MM UTC","urgency":"2","impact":"2","category":"Network"}'
```

对 HIGH 差异创建较低紧急度事件：

```bash
python3 $MCP_CALL "python3 -u $SERVICENOW_MCP_SCRIPT" create_incident '{"short_description":"NetBox Drift: UNDOCUMENTED_LINK R1:Gi2 -> SW1:Gi0/1","description":"NetBox reconciliation found a physical link (via CDP) between R1:GigabitEthernet2 and SW1:GigabitEthernet0/1 that has no cable record in NetBox.\n\nAction required: Add cable record to NetBox or investigate if this link is authorized.\n\nDiscovery method: NetClaw automated reconciliation","urgency":"3","impact":"3","category":"Network"}'
```

### 步骤 6：生成 Markmap 漂移摘要

为人类审阅生成可视化思维导图：

```bash
python3 $MCP_CALL "node $MARKMAP_MCP_SCRIPT" markmap_generate '{"markdown_content":"# NetBox Reconciliation - R1\n## CRITICAL\n### IP_DRIFT\n- Gi1: NetBox=10.1.1.1/30 vs Device=10.1.1.5/30\n- Lo0: NetBox=1.1.1.1/32 vs Device=2.2.2.2/32\n## HIGH\n### UNDOCUMENTED_LINK\n- Gi2 -> SW1:Gi0/1 (no cable in NetBox)\n### MISSING_INTERFACE\n- Gi4 (in NetBox, not on device)\n## MEDIUM\n### VLAN_MISMATCH\n- Gi3: NetBox VLAN=10 vs Device VLAN=20\n### STATUS_MISMATCH\n- Gi5: NetBox=enabled vs Device=admin-down\n## LOW\n### MTU_MISMATCH\n- Gi1: NetBox=9000 vs Device=1500"}'
```

### 步骤 7：记入 GAIT

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"user_text":"Reconcile NetBox source of truth against R1 live state","assistant_text":"NetBox reconciliation complete for R1.\n\nDiscrepancies found: 2 CRITICAL, 2 HIGH, 2 MEDIUM, 1 LOW\n\nCRITICAL:\n- IP_DRIFT: Gi1 (10.1.1.1/30 vs 10.1.1.5/30)\n- IP_DRIFT: Lo0 (1.1.1.1/32 vs 2.2.2.2/32)\n\nServiceNow incidents created for all CRITICAL and HIGH findings.\nMarkmap drift summary generated.\n\nOverall: CRITICAL - immediate human review required."}'
```

展示审计轨迹：

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_show '{"commit":"HEAD"}'
```

---

## 全网对账

要对所有设备对账，先列出清单：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_list_devices '{}'
```

再查询 NetBox 全部活动设备：

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_objects '{"object_type":"dcim.devices","filters":{"status":"active"},"brief":true}'
```

对同时存在于 pyATS testbed 与 NetBox 的每台设备执行步骤 1–7。输出 fleet 摘要：

```
Fleet Reconciliation Summary - YYYY-MM-DD
Devices Reconciled: 5/5

+---------+----------+------+--------+-----+----------+
| Device  | CRITICAL | HIGH | MEDIUM | LOW | Overall  |
+---------+----------+------+--------+-----+----------+
| R1      | 2        | 2    | 2      | 1   | CRITICAL |
| R2      | 0        | 1    | 0      | 0   | HIGH     |
| SW1     | 0        | 0    | 3      | 2   | MEDIUM   |
| SW2     | 0        | 0    | 0      | 1   | LOW      |
| FW1     | 0        | 0    | 0      | 0   | CLEAN    |
+---------+----------+------+--------+-----+----------+

Total Discrepancies: 2 CRITICAL | 3 HIGH | 5 MEDIUM | 4 LOW
ServiceNow Incidents Created: 5 (2 CRITICAL + 3 HIGH)
Overall Fleet Status: CRITICAL
```

按严重度排序（严重优先）以便分诊。

---

## NetBox 变更日志审计

对账后查看 NetBox changelogs，了解 SoT 上次更新时间：

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_changelogs '{"filters":{"object_type":"dcim.interface","limit":20}}'
```

有助于判断差异来自近期设备变更（设备漂移）还是 NetBox 从未更新（数据陈旧）。

---

## 与其他 skill 集成

| Skill | 衔接点 |
|-------|--------|
| **pyats-topology** | 提供 CDP/LLDP 邻居数据，用于 UNDOCUMENTED_LINK 与 CABLE_MISMATCH |
| **pyats-health-check** | 先跑健康检查；对账增加 NetBox 交叉层 |
| **servicenow-change-workflow** | CRITICAL/HIGH 差异自动开 ServiceNow 事件 |
| **markmap-viz** | 漂移摘要思维导图供人工审阅 |
| **drawio-diagram** | 拓扑连线按对账状态着色（绿=一致、红=不一致、黄=未备案） |
| **GAIT** | 完整对账会话写入审计轨迹 |

## 何时使用

- **定期**：每周或每月 SoT 校验
- **变更后**：每次配置变更后确认 NetBox 仍与现网一致
- **事件响应**：排查中断时核对受影响设备的 NetBox 是否准确
- **新设备入网**：添加设备后确认 NetBox 填写正确
- **审计/合规**：证明基础设施文档与现网一致
