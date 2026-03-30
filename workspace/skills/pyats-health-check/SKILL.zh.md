---
name: pyats-health-check
description: "全面的网络设备健康监控：CPU、内存、接口、硬件、NTP、日志、环境与运行时间分析。在执行设备健康检查、监控 CPU/内存、检查接口错误或校验 NTP 同步时使用。"
user-invocable: true
metadata:
  { "openclaw": { "requires": { "bins": ["python3"], "env": ["PYATS_TESTBED_PATH"] } } }
---

# 设备健康检查

## 何时使用

- 主动的每日/每周健康巡检
- 变更前与变更后验证
- 事件响应 — 告警后首先执行
- 容量规划与趋势
- 运行就绪合规检查

## 健康检查流程

始终按以下**固定顺序**执行；每一节依赖前一节结果。

### 步骤 1：设备身份与运行时间

运行 `show version` 建立身份基线。

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show version"}'
```

**提取并报告：**
- 主机名、型号、序列号
- IOS-XE 版本与镜像文件名
- 运行时间（若 < 24 小时 — 可能近期重启）
- 上次重启原因（若非预期：崩溃、断电则标出）
- 总/可用内存
- 许可状态

**阈值：**
- 运行时间 < 24h → 警告：近期重启
- 运行时间 < 1h → 严重：极近期重启，检查是否崩溃
- 上次重启原因含 "crash" 或 "error" → 严重

### 步骤 2：CPU 利用率

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show processes cpu sorted"}'
```

**阈值（5 秒 / 1 分钟 / 5 分钟平均）：**
- < 50% → 健康
- 50–75% → 警告：CPU 偏高
- 75–90% → 高：排查占用最高的进程
- > 90% → 严重：需立即排查

**需关注的进程：**
- `IP Input` — 流量大或路由环路
- `BGP Router` / `BGP I/O` — 大表或不稳定
- `OSPF-1 Hello` — OSPF 邻接问题
- `Crypto IKMP` / `Crypto Engine` — IPsec 开销
- `SNMP ENGINE` — 轮询风暴
- `ARP Input` — ARP 风暴或二层环路

### 步骤 3：内存利用率

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show processes memory sorted"}'
```

同时执行：
```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show platform resources"}'
```

**阈值：**
- 已用 < 70% → 健康
- 70–85% → 警告：内存压力
- 85–95% → 高：可能影响路由表更新
- > 95% → 严重：进程崩溃或 OOM 风险

**需关注的内存占用：**
- `BGP Router` — 大 BGP 表（全表约 ~100 万路由）
- `CEF process` — 大 FIB
- `OSPF Router` — 大 OSPF LSDB
- `HTTP CORE` — Web/RESTCONF 开销
- `IOSD iomem` — 包缓冲 I/O 内存

### 步骤 4：接口状态

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ip interface brief"}'
```

对每个活动接口再取详细计数器：
```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show interfaces"}'
```

**每个接口报告：**
- 管理状态（up/down）与协议状态（up/down）
- IP 与子网
- 速率、双工、MTU
- 输入/输出速率（bps 与 pps）
- 错误计数：CRC、入向/出向错误、丢包、溢出
- Resets 计数（若持续增长 — 接口抖动）
- 最后入/出时间戳

**标记：**
- 接口 up/down → 警告：检查物理或协议
- CRC > 0 → 警告：物理层（线缆、光模块、双工不匹配）
- 入向错误增长 → 警告：报文损坏
- 出向丢包 > 0 → 警告：拥塞或 QoS
- Resets 增长 → 严重：接口抖动
- 已配置接口 line protocol down → 严重

### 步骤 5：硬件与环境

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show inventory"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show platform"}'
```

**报告：** 模块状态（ok/fail）、序列号、PID、光模块类型与 DOM 读数。

### 步骤 6：NTP 同步

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show ntp associations"}'
```

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_run_show_command '{"device_name":"R1","command":"show clock"}'
```

**标记：**
- 无 NTP 对等同步（associations 中无 `*`）→ 对日志/取证为严重
- 时钟偏移 > 100ms → 警告
- 时钟偏移 > 1s → 严重
- 完全未配置 NTP → 严重

### 步骤 7：系统日志

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_show_logging '{"device_name":"R1"}'
```

**扫描以下模式：**
- `%SYS-*-RELOAD` — 重启事件
- `%LINEPROTO-5-UPDOWN` — 接口抖动
- `%OSPF-*-ADJCHG` — OSPF 邻接变化
- `%BGP-*-ADJCHANGE` — BGP 对等体状态变化
- `%DUAL-*-NBRCHANGE` — EIGRP 邻居变化
- `%SYS-2-MALLOCFAIL` — 内存分配失败（严重）
- `%SYS-3-CPUHOG` — 进程占满 CPU（高）
- `%TRACKING-*` — IP SLA 或对象跟踪变化
- `%SEC-*` / `%AUTHMGR-*` — 安全事件
- `%PLATFORM-*-CRASH` — 崩溃事件（严重）
- `Traceback` — 软件缺陷（严重 — 开 TAC case）

### 步骤 8：连通性验证

测试到关键基础设施的可达性：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_ping_from_network_device '{"device_name":"R1","command":"ping 8.8.8.8 repeat 5"}'
```

**阈值：**
- 100% 成功、RTT < 50ms → 健康
- 100% 成功、RTT > 100ms → 警告：高时延
- 80–99% 成功 → 警告：丢包
- < 80% 成功 → 严重：明显丢包
- 0% 成功 → 严重：不可达

## 健康报告格式

始终输出摘要表：

```
Device: R1 (devnetsandboxiosxec8k.cisco.com)
Model: C8000V | IOS-XE: 17.x.x | Uptime: XXd XXh

┌──────────────────┬──────────┬─────────────────────────┐
│ Check            │ Status   │ Details                 │
├──────────────────┼──────────┼─────────────────────────┤
│ CPU (5min avg)   │ HEALTHY  │ 12%                     │
│ Memory           │ HEALTHY  │ 45% used (1.2G/2.6G)   │
│ Interfaces       │ WARNING  │ Gi2 down/down           │
│ Hardware         │ HEALTHY  │ All modules OK          │
│ NTP              │ HEALTHY  │ Synced, offset 2ms      │
│ Logs             │ WARNING  │ 3 OSPF adjacency flaps  │
│ Connectivity     │ HEALTHY  │ 100% to 8.8.8.8, 23ms  │
└──────────────────┴──────────┴─────────────────────────┘

Overall: WARNING — 2 items need attention
```

严重度顺序：CRITICAL > HIGH > WARNING > HEALTHY。总体状态取各单项中最差者。

## NetBox 交叉引用（MISSION02 增强）

当 NetBox 可用（已设置 `$NETBOX_MCP_SCRIPT`）时，在步骤 1 与步骤 4 之后将设备状态与 SoT 交叉核对：

### 接口状态校验

查询 NetBox 中的预期接口状态：

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_objects '{"object_type":"dcim.interfaces","filters":{"device":"R1"},"brief":true}'
```

**对比 NetBox 意图与设备现实：**
- NetBox 显示启用但设备为 down → 严重：非预期中断
- NetBox 显示禁用但设备为 up → 警告：未备案的启用
- 设备存在但 NetBox 无记录 → 警告：未备案接口
- NetBox 有记录但设备无此接口 → 警告：NetBox 数据陈旧

### IP 地址校验

查询 NetBox 中的 IP 分配：

```bash
python3 $MCP_CALL "python3 -u $NETBOX_MCP_SCRIPT" netbox_get_objects '{"object_type":"ipam.ip-addresses","filters":{"device":"R1"}}'
```

**对比：** 设备 IP 与 NetBox 不一致时标为 IP_DRIFT。

## 全网健康（并行调用）

要对**所有**设备同时健康检查，先列出设备：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" pyats_list_devices
```

然后对每个设备并行执行步骤 1–8。汇总结果并生成 fleet 摘要：

```
┌──────────┬──────────┬──────┬────────┬──────────┬─────────────┐
│ Device   │ CPU      │ Mem  │ Intf   │ NTP      │ Overall     │
├──────────┼──────────┼──────┼────────┼──────────┼─────────────┤
│ R1       │ HEALTHY  │ WARN │ HEALTHY│ HEALTHY  │ WARNING     │
│ R2       │ HEALTHY  │ OK   │ CRIT   │ HEALTHY  │ CRITICAL    │
│ SW1      │ HIGH     │ OK   │ HEALTHY│ CRIT     │ CRITICAL    │
└──────────┴──────────┴──────┴────────┴──────────┴─────────────┘
```

按严重度排序（严重优先）以便分诊。

## GAIT 审计轨迹

健康检查完成后将会话记入 GAIT：

```bash
python3 $MCP_CALL "python3 -u $GAIT_MCP_SCRIPT" gait_record_turn '{"input":{"role":"assistant","content":"Health check completed on R1: CPU HEALTHY (12%), Memory WARNING (78%), Interfaces HEALTHY, NTP HEALTHY. Overall: WARNING.","artifacts":[]}}'
```
