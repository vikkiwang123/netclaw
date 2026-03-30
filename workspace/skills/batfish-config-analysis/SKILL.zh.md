---
name: batfish-config-analysis
description: "Batfish 网络配置分析：部署前校验、可达性测试、ACL/防火墙路径追踪、差异分析、合规检查。在部署前校验配置、测试流量路径、追踪 ACL 规则、对比配置版本或审计合规策略时使用。严格只读。"
user-invocable: true
metadata:
  { "openclaw": { "requires": { "bins": ["python3", "docker"], "env": ["BATFISH_HOST"] } } }
---

# Batfish 配置分析

## MCP 服务

- **源码**：内置（`mcp-servers/batfish-mcp/`）
- **命令**：`python3 -u mcp-servers/batfish-mcp/batfish_mcp_server.py`（stdio 传输）
- **依赖**：Batfish Docker 容器运行中，环境变量 `BATFISH_HOST`、`BATFISH_PORT`
- **Python**：3.10+
- **依赖包**：`pybatfish`、`mcp[cli]`、`python-dotenv`

## 可用工具（8 个）

| 工具 | 参数 | 作用 |
|------|------|------|
| `batfish_upload_snapshot` | `snapshot_name`、`configs`/`config_path`、`network` | 上传设备配置到 Batfish 并创建命名快照 |
| `batfish_validate_config` | `snapshot_name`、`network` | 校验配置：每设备通过/失败、厂商识别、告警 |
| `batfish_test_reachability` | `snapshot_name`、`src_ip`、`dst_ip`、`protocol`、`dst_port` | 测试两端点流量是否可达并给出完整路径 |
| `batfish_trace_acl` | `snapshot_name`、`device`、`filter_name`、`src_ip`、`dst_ip`、`protocol`、`dst_port` | 将数据包在 ACL 规则中追踪到匹配的 permit/deny |
| `batfish_diff_configs` | `reference_snapshot`、`candidate_snapshot`、`include_routes`、`include_reachability` | 对比两快照的路由与可达性差异 |
| `batfish_check_compliance` | `snapshot_name`、`policy_type` | 按合规策略检查配置（内置 6 种策略类型） |
| `batfish_list_snapshots` | `network` | 列出所有快照 |
| `batfish_delete_snapshot` | `snapshot_name`、`network` | 删除快照 |

## 工作流：变更前校验

用户希望在部署前校验配置时：

1. **上传配置**：`batfish_upload_snapshot`，使用内联 `configs` 字典或配置目录路径
2. **校验**：`batfish_validate_config` 检查解析状态、厂商识别、告警/错误
3. **可达性**：`batfish_test_reachability` 覆盖关键业务路径
4. **合规**：`batfish_check_compliance` 对照组织策略
5. **报告**：结构化通过/失败与具体发现
6. **GAIT**：操作自动记录

### 示例：部署前校验

```bash
# 上传拟议配置
batfish_upload_snapshot snapshot_name="pre-change-site-a" config_path="/path/to/configs/"

# 校验解析状态
batfish_validate_config snapshot_name="pre-change-site-a"

# 测试关键路径
batfish_test_reachability snapshot_name="pre-change-site-a" src_ip="10.1.1.1" dst_ip="10.2.2.1" protocol="TCP" dst_port=443

# 合规检查
batfish_check_compliance snapshot_name="pre-change-site-a" policy_type="interface_descriptions"
```

## 工作流：变更影响分析

对比变更前后配置时：

1. **上传「变更前」快照**：`batfish_upload_snapshot` 使用当前配置
2. **上传「变更后」快照**：`batfish_upload_snapshot` 使用拟议配置
3. **差异**：`batfish_diff_configs` 查看路由与可达性差异
4. **深挖**：对新被拒绝的流量使用 `batfish_trace_acl`
5. **报告**：结构化展示增删改路由与流量

## 工作流：ACL 排障

排查访问控制问题时：

1. **上传配置**：`batfish_upload_snapshot` 上传设备配置
2. **追踪数据包**：`batfish_trace_acl` 指定设备、ACL 名与包头
3. **审阅**：确认匹配规则、行号、permit/deny 动作
4. **尝试替代方案**：修改配置、重新上传、再次追踪

## 信任边界 — Batfish 不是现网真值

Batfish 对**你上传的配置**构建模型并给出推理，语气可以很确定，但那是**离线模型里的结论**，不是对现网的观测。业内常说：它会在不完整输入或未覆盖的行为上照样给出「答案」——若把输出当真理，就等于允许它**误导**你。

- **快照 ≠ 现网**：快照不全、配置陈旧、缺设备、版本不对 → 结果照样整齐，但可能错得离谱。
- **建模有边界**：部分特性、有状态行为、动态协议、厂商细节可能被简化或未实现；可达性/ACL 追踪可能与真机不一致。
- **不得压倒现网验证**：若 Batfish 写「放行」而设备上 `show`、抓包或 pyATS 显示相反，**以设备为准**，再查模型与快照缺口。
- **与 NetClaw 准则一致**：不臆测设备状态 — Batfish **不能替代**设备上的基线采集与变更后验证。

## 与其他 Skill 的衔接

| Skill | 衔接方式 |
|-------|----------|
| **pyats-config-mgmt** | 经 pyATS 推送前用 Batfish 校验配置 |
| **gait-session-tracking** | 所有 Batfish 操作自动记录 |
| **servicenow-change-workflow** | 将 Batfish 校验结果作为变更证据引用 |
| **fwrule-analyzer** | 与跨厂商 ACL 重叠分析互补 |
| **cml-lab-lifecycle** | 对 CML 实验配置做 Batfish 分析 |

## 重要规则

- **所有操作严格只读** — Batfish 只分析上传的配置，不修改现网设备
- **GAIT 审计必选** — 所有操作自动记录
- **快照为临时** — 快照生命周期由 Batfish 管理；持久记录用 GAIT
- **多厂商** — 支持 Cisco IOS/IOS-XE/NX-OS、JunOS、Arista EOS、Palo Alto、F5 等

## 错误处理

- **BATFISH_UNREACHABLE**：确认 Docker 容器在运行（`docker ps | grep batfish`）
- **SNAPSHOT_NOT_FOUND**：使用 `batfish_list_snapshots` 查看可用快照
- **INVALID_INPUT**：确认 `configs` 非空或 `config_path` 存在
- **DEVICE_NOT_FOUND**：使用 `batfish_validate_config` 列出快照内设备
- **FILTER_NOT_FOUND**：确认指定设备上存在该 ACL/过滤器名称

## 环境变量

- `BATFISH_HOST` — Batfish 主机名（默认 localhost）
- `BATFISH_PORT` — Batfish 端口（默认 9997）
- `BATFISH_NETWORK` — 默认网络名（默认 netclaw）
