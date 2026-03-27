# Cisco MCP Tools Reference

面向 `netclaw` 仓库中 Cisco 相关 MCP Server 的学习参考手册。

目标：
- 快速看清每个 Cisco MCP Server 的用途
- 查到每个 Server 的仓库链接
- 查到运行所需环境变量/调用方式
- 查到主要 tools 的参数格式

说明：
- 本文按仓库当前 `README.md` 与对应 `workspace/skills/*/SKILL.md` 整理。
- 这是“学习/赋能版”文档，优先覆盖常用 Cisco MCP。
- 对于 `Meraki` 这类动态暴露大量 API 的服务器，本文列出高频能力与关键入口，不穷举全部 804 个 endpoint。

---

## 目录

1. `pyATS MCP`
2. `Catalyst Center MCP`
3. `Cisco ACI MCP`
4. `Cisco ISE MCP`
5. `Cisco CML MCP`
6. `Cisco NSO MCP`
7. `Cisco FMC MCP`
8. `Cisco Meraki MCP`
9. `Cisco ThousandEyes MCP`
10. `Cisco RADKit MCP`
11. `Cisco SD-WAN MCP`

---

## 1. pyATS MCP

### 概览

- Repository: [automateyournetwork/pyATS_MCP](https://github.com/automateyournetwork/pyATS_MCP)
- Transport: `stdio (Python)`
- 主要用途：
  - 设备 `show` 命令执行
  - 配置下发
  - 设备侧 `ping`
  - 日志抓取
  - running-config 基线
  - 动态 pyATS AEtest 校验

### 环境变量与调用方式

必需环境变量：

- `PYATS_TESTBED_PATH`
- `PYATS_MCP_SCRIPT`
- `MCP_CALL`

调用模板：

```bash
PYATS_TESTBED_PATH=$PYATS_TESTBED_PATH python3 $MCP_CALL "python3 -u $PYATS_MCP_SCRIPT" TOOL_NAME 'ARGS_JSON'
```

### Tools 参数总表

| Tool | 参数 | 用途 |
|------|------|------|
| `pyats_list_devices` | 无 | 列出 testbed 中全部设备 |
| `pyats_run_show_command` | `device_name: string`, `command: string` | 执行 `show` 命令，优先返回 Genie 结构化 JSON |
| `pyats_configure_device` | `device_name: string`, `config_commands: list[string] \| multiline string` | 下发配置 |
| `pyats_show_running_config` | `device_name: string` | 获取 running-config |
| `pyats_show_logging` | `device_name: string` | 获取最近日志 |
| `pyats_ping_from_network_device` | `device_name: string`, `command: string` | 从设备本机发起 ping |
| `pyats_run_linux_command` | `device_name: string`, `command: string` | 对 testbed 中 Linux 节点执行命令 |
| `pyats_run_dynamic_test` | `test_script_content: string` | 执行内联 pyATS AEtest 脚本 |

### 参数备注

- `pyats_run_show_command.command` 必须以 `show` 开头
- 禁止包含管道、重定向、危险关键字
- `pyats_configure_device` 不要手动传 `conf t` / `end`
- `pyats_ping_from_network_device.command` 可直接传扩展 ping 语法

### 适合学习什么

- Cisco CLI 自动化
- Genie parser/structured output
- 变更前后基线抓取
- 设备级自动化的 MCP 设计方式

---

## 2. Catalyst Center MCP

### 概览

- Repository: [richbibby/catalyst-center-mcp](https://github.com/richbibby/catalyst-center-mcp)
- Transport: `stdio (Python)`
- 主要用途：
  - 设备资产查询
  - Site hierarchy 查询
  - 接口清单
  - 客户端查询与统计
  - 园区网排障

### 环境变量与调用方式

必需环境变量：

- `CCC_HOST`
- `CCC_USER`
- `CCC_PWD`
- `CATC_MCP_SCRIPT`
- `MCP_CALL`

调用模板：

```bash
CCC_HOST=$CCC_HOST CCC_USER=$CCC_USER CCC_PWD=$CCC_PWD python3 $MCP_CALL "python3 -u $CATC_MCP_SCRIPT" TOOL_NAME 'ARGS_JSON'
```

### Tools 参数总表

#### 2.1 Inventory

| Tool | 参数 | 用途 |
|------|------|------|
| `fetch_devices` | 至少一个过滤条件；可选参数包括 `hostname`, `managementIpAddress`, `serialNumber`, `platformId`, `family`, `type`, `role`, `reachabilityStatus`, `collectionStatus`, `softwareVersion`, `softwareType`, `series`, `locationName`, `associatedWlcIp`, `macAddress`, `id`, `roleSource`, `offset`, `limit`, `sortBy`, `sortOrder` | 查询设备清单 |
| `fetch_sites` | 无 | 查询全站点层级 |
| `fetch_interfaces` | `device_id: string` | 查询指定设备接口 |

#### 2.2 Client Operations

| Tool | 参数 | 用途 |
|------|------|------|
| `get_api_compatible_time_range` | `time_window?`, `start_datetime_iso?`, `end_datetime_iso?` | 把人类时间转换为 API 所需 epoch ms |
| `get_clients_list` | `start_time`, `end_time`, `limit?`, `offset?`, `sort_by?`, `order?`, `client_type?`, `os_type?`, `os_version?`, `site_hierarchy?`, `site_hierarchy_id?`, `site_id?`, `ipv4_address?`, `ipv6_address?`, `mac_address?`, `wlc_name?`, `connected_network_device_name?`, `ssid?`, `band?`, `view?`, `attribute?` | 查询客户端列表 |
| `get_client_details_by_mac` | `client_mac_address`, `start_time?`, `end_time?`, `view?`, `attribute?` | 按 MAC 查询单客户端详情 |
| `get_clients_count` | 与 `get_clients_list` 同过滤参数，但不含排序/分页视图参数 | 统计客户端数量 |

### 适合学习什么

- 控制器型网络平台的 API 模型
- inventory / site / client 三类数据面
- 时间窗口和过滤器型 API 的设计方式

---

## 3. Cisco ACI MCP

### 概览

- Repository: [automateyournetwork/ACI_MCP](https://github.com/automateyournetwork/ACI_MCP)
- Transport: `stdio (Python)`
- 主要用途：
  - ACI fabric 健康检查
  - tenant / VRF / BD / EPG 审计
  - fault / health 分析
  - ACI policy 变更

### 环境变量与调用方式

必需环境变量：

- `APIC_URL`
- `ACI_USERNAME`
- `ACI_PASSWORD`
- `ACI_MCP_SCRIPT`
- `MCP_CALL`

调用模板：

```bash
APIC_URL=$APIC_URL USERNAME=$ACI_USERNAME PASSWORD=$ACI_PASSWORD python3 $MCP_CALL "python3 -u $ACI_MCP_SCRIPT" TOOL_NAME '{"param":"value"}'
```

### Tools 参数总表

| Tool | 参数 | 用途 |
|------|------|------|
| `fabric_nodes` | 无 | 查询 fabric 节点 |
| `fabric_pods` | 无 | 查询 pods |
| `fabric_links` | 无 | 查询 fabric 链路 |
| `faults` | 无 | 查询 faults |
| `health` | 无 | 查询整体健康分 |
| `tenants_get` | 无 | 查询 tenants |
| `tenants_post` | `name`, `descr?` | 创建 tenant |
| `fvCtx_get` | 无 | 查询 VRF |
| `fvCtx_post` | `tenant`, `name`, `descr?`, `pcEnfPref?` | 创建 VRF |
| `fvBD_get` | 无 | 查询 Bridge Domain |
| `fvBD_post` | `tenant`, `name`, `descr?`, `vrf` | 创建 BD |
| `fvAp_get` | 无 | 查询 App Profile |
| `fvAp_post` | `tenant`, `name`, `descr?` | 创建 App Profile |
| `fvAEPg_post` | `tenant`, `ap`, `name`, `descr?`, `bd` | 创建 EPG |

### 适合学习什么

- Cisco ACI 对象模型
- policy object 的依赖顺序
- 从只读审计到变更执行的 MCP 设计

---

## 4. Cisco ISE MCP

### 概览

- Repository: [automateyournetwork/ISE_MCP](https://github.com/automateyournetwork/ISE_MCP)
- Transport: `stdio (Python)`
- 主要用途：
  - posture / policy audit
  - endpoint 调查
  - authentication / authorization rule 检查
  - identity / profiling / session 查询

### 环境变量与调用方式

必需环境变量：

- `ISE_BASE`
- `ISE_USERNAME`
- `ISE_PASSWORD`
- `ISE_MCP_SCRIPT`
- `MCP_CALL`

调用模板：

```bash
ISE_BASE=$ISE_BASE USERNAME=$ISE_USERNAME PASSWORD=$ISE_PASSWORD python3 $MCP_CALL "python3 -u $ISE_MCP_SCRIPT" TOOL_NAME '{"param":"value"}'
```

### Tools 参数总表

| Tool | 参数 | 用途 |
|------|------|------|
| `clear_cache` | 无 | 清缓存，确保读取最新数据 |
| `get_cache_stats` | 无 | 查看缓存状态 |
| `network_access_policy_set` | 无 | 查询 policy set |
| `network_access_authorization_rules` | 无 | 查询 authorization rules |
| `network_access_authentication_rules` | 无 | 查询 authentication rules |
| `network_access_conditions` | 无 | 查询条件对象 |
| `endpoints` | 无 | 查询 endpoints |
| `active_sessions` | 无 | 查询活动会话 |
| `profiler_profiles` | 无 | 查询 profiler profiles |
| `identity_groups` | 无 | 查询 identity groups |

### 适合学习什么

- NAC/AAA 系统如何做 MCP 封装
- endpoint/session/policy 三层数据联动
- 安全调查工作流如何映射为 tools

---

## 5. Cisco CML MCP

### 概览

- Repository: [xorrkaz/cml-mcp](https://github.com/xorrkaz/cml-mcp)
- Command: `cml-mcp`
- Transport: `stdio`
- 主要用途：
  - lab 生命周期
  - 拓扑构建
  - 节点操作
  - 抓包
  - 平台管理

### 环境变量与调用方式

必需环境变量：

- `CML_URL`
- `CML_USERNAME`
- `CML_PASSWORD`
- `CML_VERIFY_SSL` 可选

### Tools 参数总表

#### 5.1 Lab Lifecycle

| Tool | 参数 | 用途 |
|------|------|------|
| `create_lab` | `title` | 创建空 lab |
| `get_lab` | `lab_id` 或 `lab_title` | 查询单 lab |
| `get_labs` | 无 | 查询所有 lab |
| `start_lab` | `lab_id` 或 `lab_title` | 启动 lab |
| `stop_lab` | `lab_id` 或 `lab_title` | 停止 lab |
| `wipe_lab` | `lab_id` 或 `lab_title` | 清空 lab 配置 |
| `delete_lab` | `lab_id` 或 `lab_title` | 删除 lab |
| `import_lab` | `topology` | 从 YAML 导入 lab |
| `export_lab` | `lab_id` 或 `lab_title` | 导出 lab YAML |
| `clone_lab` | `lab_id` 或 `lab_title` | 克隆 lab |
| `download_lab_configs` | `lab_id` 或 `lab_title` | 下载 lab 全部配置 |

#### 5.2 Topology Builder

| Tool | 参数 | 用途 |
|------|------|------|
| `get_node_defs` | 无 | 查询支持的节点类型 |
| `create_node` | `lab_id/lab_title`, `node_def`, `label`, `x`, `y` | 创建节点 |
| `get_nodes` | `lab_id/lab_title` | 查询 lab 节点 |
| `get_node` | `lab_id/lab_title`, `node_id/node_label` | 查询节点详情 |
| `create_interface` | `lab_id/lab_title`, `node_id/node_label`, `slot?` | 创建接口 |
| `get_interfaces` | `lab_id/lab_title`, `node_id/node_label` | 查询节点接口 |
| `get_interface` | `lab_id/lab_title`, `node_id/node_label`, `interface_id/interface_label` | 查询接口详情 |
| `create_link` | `lab_id/lab_title`, `src_node/src_label`, `src_intf`, `dst_node/dst_label`, `dst_intf` | 建链 |
| `get_links` | `lab_id/lab_title` | 查询全部链路 |
| `get_link` | `lab_id/lab_title`, `link_id` | 查询链路详情 |
| `set_link_condition` | `lab_id/lab_title`, `link_id`, `bandwidth?`, `latency?`, `jitter?`, `loss?` | 配置链路仿真参数 |
| `set_link_state` | `lab_id/lab_title`, `link_id`, `state` | up/down 链路 |
| `create_annotation` | `lab_id/lab_title`, `annotation_type`, `content`, `x`, `y`, `width?`, `height?` | 增加标注 |
| `get_annotations` | `lab_id/lab_title` | 查询标注 |
| `delete_annotation` | `lab_id/lab_title`, `annotation_id` | 删除标注 |

#### 5.3 Node Operations

| Tool | 参数 | 用途 |
|------|------|------|
| `start_node` | `lab_id/lab_title`, `node_id/node_label` | 启动单节点 |
| `stop_node` | `lab_id/lab_title`, `node_id/node_label` | 停止单节点 |
| `get_node` | `lab_id/lab_title`, `node_id/node_label` | 查询节点状态 |
| `get_nodes` | `lab_id/lab_title` | 查询所有节点 |
| `wipe_node` | `lab_id/lab_title`, `node_id/node_label` | 清除节点配置 |
| `set_node_config` | `lab_id/lab_title`, `node_id/node_label`, `config` | 设置 startup config |
| `get_node_config` | `lab_id/lab_title`, `node_id/node_label` | 获取节点配置 |
| `download_lab_configs` | `lab_id/lab_title` | 下载全 lab 配置 |
| `get_node_console` | `lab_id/lab_title`, `node_id/node_label` | 获取 console 连接信息 |
| `get_node_console_log` | `lab_id/lab_title`, `node_id/node_label`, `lines?` | 获取 console log |
| `execute_command` | `lab_id/lab_title`, `node_id/node_label`, `command` | 对节点执行 CLI |

#### 5.4 Packet Capture

| Tool | 参数 | 用途 |
|------|------|------|
| `start_capture` | `lab_id/lab_title`, `link_id`, `max_packets?`, `pcap_filter?` | 开始抓包 |
| `stop_capture` | `lab_id/lab_title`, `link_id` | 停止抓包 |
| `get_capture_status` | `lab_id/lab_title`, `link_id` | 查询抓包状态 |
| `download_capture` | `lab_id/lab_title`, `link_id`, `file_path?` | 下载 pcap |
| `list_captures` | `lab_id/lab_title` | 查看 lab 中抓包任务 |

#### 5.5 Admin

| Tool | 参数 | 用途 |
|------|------|------|
| `get_users` | 无 | 查询 CML 用户 |
| `create_user` | `username`, `password`, `fullname?`, `email?`, `admin?` | 创建用户 |
| `get_user` | `user_id` 或 `username` | 查单用户 |
| `update_user` | `user_id` 或 `username`, 需要更新的字段 | 更新用户 |
| `delete_user` | `user_id` 或 `username` | 删除用户 |
| `get_groups` | 无 | 查询组 |
| `create_group` | `name`, `description?`, `members?` | 创建组 |
| `update_group` | `group_id` 或 `name`, 需要更新的字段 | 更新组 |
| `delete_group` | `group_id` 或 `name` | 删除组 |
| `get_system_info` | 无 | 系统信息 |
| `get_node_defs` | 无 | 节点类型与资源需求 |
| `get_licensing` | 无 | License 状态 |
| `get_resource_usage` | 无 | 资源使用情况 |

### 适合学习什么

- 实验室类 MCP 的完整设计
- lab / topology / node / capture / admin 五个维度的 tool 设计

---

## 6. Cisco NSO MCP

### 概览

- Repository: [NSO-developer/cisco-nso-mcp-server](https://github.com/NSO-developer/cisco-nso-mcp-server)
- Command: `cisco-nso-mcp-server`
- Transport: `stdio`
- API: `RESTCONF`

### 环境变量与调用方式

必需环境变量：

- `NSO_ADDRESS`
- `NSO_USERNAME`
- `NSO_PASSWORD`

可选环境变量：

- `NSO_SCHEME`，默认 `http`
- `NSO_PORT`，默认 `8080`
- `NSO_VERIFY`
- `NSO_TIMEOUT`

### Tools 参数总表

#### 6.1 Device Operations

| Tool | 参数 | 用途 |
|------|------|------|
| `get_device_config` | `device_name` | 从 NSO CDB 获取设备配置 |
| `get_device_state` | `device_name` | 获取设备 operational state |
| `check_device_sync` | `device_name` | 检查 NSO 与设备是否同步 |
| `sync_from_device` | `device_name` | 从设备回拉配置到 NSO |
| `get_device_platform` | `device_name` | 获取设备平台信息 |
| `get_device_ned_ids` | 无 | 查看支持的 NED |
| `get_device_groups` | 无 | 查看设备组 |

#### 6.2 Service Management

| Tool | 参数 | 用途 |
|------|------|------|
| `get_service_types` | 无 | 获取所有 service type |
| `get_services` | `service_type` | 获取指定服务类型下的实例 |

### 适合学习什么

- NSO 的 device 与 service 双视角
- CDB / NED / sync-from 等核心概念
- 编排平台型 MCP 的接口设计

---

## 7. Cisco FMC MCP

### 概览

- Repository: [CiscoDevNet/CiscoFMC-MCP-server-community](https://github.com/CiscoDevNet/CiscoFMC-MCP-server-community)
- Transport: `HTTP`
- 默认形态：`http://<host>:8000/mcp`
- 主要用途：
  - Secure Firewall / FTD 策略搜索
  - 按 IP/FQDN 查规则
  - 多 FMC 环境查询

### 环境变量

- `FMC_BASE_URL`
- `FMC_USERNAME`
- `FMC_PASSWORD`

### Tools 参数总表

| Tool | 参数 | 用途 |
|------|------|------|
| `list_fmc_profiles` | 无 | 查看全部 FMC profile |
| `find_rules_by_ip_or_fqdn` | 具体字段依实现而定，核心是 `policy` + `IP/FQDN indicator` | 在指定 access policy 内查规则 |
| `find_rules_for_target` | 具体字段依实现而定，核心是 `FTD device / HA cluster target` | 先解析目标设备所属 policy，再查规则 |
| `search_access_rules` | 具体字段依实现而定，核心支持 `network indicators`, `identity indicators`, `policy filters` | FMC 范围内搜索 access rules |

### 适合学习什么

- 安全策略搜索型 MCP
- 多 profile / 多控制器场景下的 tool 设计

---

## 8. Cisco Meraki MCP

### 概览

- Repository: [CiscoDevNet/meraki-magic-mcp-community](https://github.com/CiscoDevNet/meraki-magic-mcp-community)
- Transport: `stdio` 或 `HTTP`
- 主要特点：
  - 动态暴露大量 Dashboard API
  - 推荐脚本：`meraki-mcp-dynamic.py`
  - 规模：约 `804` API endpoints

### 环境变量

- `MERAKI_API_KEY`
- `MERAKI_ORG_ID`

### 调用风格

Meraki 更像“API method catalog”，不是少量固定 tool。

最重要的通用入口：

| Tool / 方法 | 参数 | 用途 |
|------|------|------|
| `call_meraki_api` | `section`, `operation`, 以及对应 API 参数 | 调用任意 Dashboard API method |

### 高频能力总表

#### 8.1 Network & Device

| API Method / Tool | 主要参数 | 用途 |
|------|------|------|
| `getOrganizations` | 无 | 查组织 |
| `getNetworks` | 常见为 `organizationId` 或隐含 org 上下文 | 查网络 |
| `getDevices` | 组织/过滤条件 | 查设备 |
| `getNetworkDevices` | `networkId` | 查某网络设备 |
| `getDeviceDetails` | `serial` | 查单设备详情 |
| `getDeviceStatus` | `serial` | 查设备状态 |
| `getDeviceUplink` | `serial` | 查 uplink 状态 |
| `getDeviceClients` | `serial`, 时间/分页类参数 | 查设备客户端 |
| `getOrganizationConfigurationChanges` | 时间/组织过滤参数 | 配置变更审计 |
| `getOrganizationApiRequests` | 时间/过滤参数 | API 调用审计 |

#### 8.2 Wireless

| API Method / Tool | 主要参数 | 用途 |
|------|------|------|
| `getWirelessSSIDs` | `networkId` | 查 SSID |
| `updateWirelessSSID` | `networkId`, `number`, 配置字段 | 改 SSID |
| `getWirelessSettings` | `networkId` | 查无线全局设置 |
| `getWirelessRFProfiles` | `networkId` | 查 RF Profile |
| `createWirelessRFProfile` | `networkId`, profile 字段 | 建 RF Profile |
| `getWirelessChannelUtilization` | `networkId` 或 AP/time 参数 | 查信道利用率 |
| `getWirelessSignalQuality` | `networkId` 或 AP/time 参数 | 查信号质量 |
| `getWirelessConnectionStats` | `networkId`, 时间参数 | 查连接成功/失败率 |
| `getWirelessClientConnectivityEvents` | `networkId`, `client/mac`, 时间参数 | 查客户端无线事件 |

#### 8.3 Switching

| API Method / Tool | 主要参数 | 用途 |
|------|------|------|
| `getDeviceSwitchPorts` | `serial` | 查端口配置 |
| `updateDeviceSwitchPort` | `serial`, `portId`, 端口配置字段 | 改端口 |
| `getDeviceSwitchPortStatuses` | `serial` | 查端口实时状态 |
| `cycleDeviceSwitchPorts` | `serial`, `ports` | bounce 端口 |
| `getSwitchVlans` | `networkId` | 查 VLAN |
| `createSwitchVlan` | `networkId`, VLAN 字段 | 建 VLAN |
| `getDeviceSwitchAccessControlLists` | `serial` | 查 ACL |
| `updateDeviceSwitchAccessControlLists` | `serial`, ACL 字段 | 改 ACL |
| `getDeviceSwitchQosRules` | `serial` | 查 QoS 规则 |
| `createDeviceSwitchQosRule` | `serial`, QoS 字段 | 新建 QoS 规则 |

#### 8.4 Security Appliance (MX)

| API Method / Tool | 主要参数 | 用途 |
|------|------|------|
| `getNetworkSecurityCenter` | `networkId` | 安全总览 |
| `getNetworkVpnStatus` | `networkId` | VPN 状态 |
| `getNetworkSecurityFirewallRules` | `networkId` | 查 L3 firewall rules |
| `updateNetworkSecurityFirewallRules` | `networkId`, rule set | 改 firewall rules |
| `getNetworkSecurityVpnSiteToSite` | `networkId` | 查站点 VPN |
| `updateNetworkSecurityVpnSiteToSite` | `networkId`, VPN config | 改站点 VPN |
| `getNetworkSecurityContentFiltering` | `networkId` | 查 URL filtering |
| `updateNetworkSecurityContentFiltering` | `networkId`, filtering config | 改 URL filtering |
| `getNetworkSecuritySecurityEvents` | `networkId`, 时间参数 | 查安全事件 |
| `getNetworkSecurityTrafficShaping` | `networkId` | 查 traffic shaping |
| `updateNetworkSecurityTrafficShaping` | `networkId`, shaping config | 改 traffic shaping |

#### 8.5 Monitoring & Diagnostics

| API Method / Tool | 主要参数 | 用途 |
|------|------|------|
| `createDeviceManagementInterfacePing` | `serial`, 目标 IP/host | 从设备发起 ping |
| `getDeviceManagementInterfacePingResults` | `serial`, ping job/id | 获取 ping 结果 |
| `createDeviceLiveToolsCableTest` | `serial`, `port` | 发起线缆测试 |
| `getDeviceLiveToolsCableTestResults` | `serial`, test job/id | 取线缆测试结果 |
| `blinkDeviceLeds` | `serial`, 可选 duration | 闪灯定位设备 |
| `wakeDeviceOnLan` | `serial`, MAC 等 | 发送 WoL |
| `getDeviceCameraAnalyticsLive` | `serial` | 摄像头实时分析 |
| `getDeviceCameraAnalyticsOverview` | `serial`, 时间参数 | 摄像头历史分析 |
| `getDeviceCameraAnalyticsZones` | `serial` | 摄像头区域分析 |
| `generateDeviceCameraSnapshot` | `serial` | 取快照 |
| `getDeviceCameraVideoSettings` | `serial` | 摄像头视频设置 |
| `getNetworkCameraQualityRetentionProfiles` | `networkId` | 摄像头保留策略 |
| `getDeviceCameraSense` | `serial` | 查 camera sense |
| `updateDeviceCameraSense` | `serial`, sense config | 改 camera sense |

### 适合学习什么

- 动态 API 暴露型 MCP
- 同一 Server 内横跨 network / wireless / switch / mx / diagnostics 的设计

---

## 9. Cisco ThousandEyes MCP

### 概览

社区版：
- Repository: [CiscoDevNet/thousandeyes-mcp-community](https://github.com/CiscoDevNet/thousandeyes-mcp-community)
- Transport: `stdio`
- Script: `src/server.py`

官方版：
- Repository: [CiscoDevNet/ThousandEyes-MCP-Server-official](https://github.com/CiscoDevNet/ThousandEyes-MCP-Server-official)
- Endpoint: `https://api.thousandeyes.com/mcp`
- Transport: `Remote HTTP`

### 环境变量

- `TE_TOKEN`

### Tools 参数总表

#### 9.1 Community Tools

| Tool | 参数 | 用途 |
|------|------|------|
| `te_list_tests` | `aid?`, `name_contains?`, `test_type?` | 查测试 |
| `te_list_agents` | `agent_types?`, `aid?` | 查 agents |
| `te_get_test_results` | `test_id`, `test_type`, `window?`, `start?`, `end?`, `aid?`, `agent_id?` | 查测试结果 |
| `te_get_path_vis` | `test_id`, `window?`, `start?`, `end?`, `aid?`, `agent_id?`, `direction?` | 查路径可视化 |
| `te_list_dashboards` | `aid?`, `title_contains?` | 查 dashboards |
| `te_get_dashboard` | `dashboard_id`, `aid?` | 查单 dashboard |
| `te_get_dashboard_widget` | `dashboard_id`, `widget_id`, `window?`, `start?`, `end?`, `aid?` | 查 widget 数据 |
| `te_get_users` | 无 | 查用户 |
| `te_get_account_groups` | 无 | 查 account groups |

#### 9.2 Official Server 能力

官方 server 在 skill 文档里给的是能力集合，不是逐个 JSON 参数清单。常用能力包括：

- `List Tests`
- `Get Test Details`
- `List Events`
- `Get Event Details`
- `List Alerts`
- `Get Alert Details`
- `Search Outages`
- `Instant Tests`
- `Get Anomalies`
- `Get Metrics`
- `Views Explanations`
- `List Endpoint Agents and Tests`
- `Get Endpoint Agent Metrics`
- `Get Path Visualization`
- `Get Full Path Visualization`
- `Get BGP Test Results`
- `Get BGP Route Details`

### 适合学习什么

- 合成监测与外部路径观测
- 双 Server 模式：社区本地版 + 厂商托管官方版

---

## 10. Cisco RADKit MCP

### 概览

- Repository: [CiscoDevNet/radkit-mcp-server-community](https://github.com/CiscoDevNet/radkit-mcp-server-community)
- Transport: `stdio / SSE / HTTPS`
- 主要用途：
  - 通过 Cisco cloud relay 访问内网设备
  - 远程 CLI
  - SNMP GET
  - inventory 查询

### 环境变量

- `RADKIT_IDENTITY`
- `RADKIT_DEFAULT_SERVICE_SERIAL`

### Tools 参数总表

| Tool | 参数 | 用途 |
|------|------|------|
| `get_device_inventory_names` | 无 | 列出可访问设备 |
| `get_device_attributes` | `target_device` | 查设备属性 |
| `exec_cli_commands_in_device` | `target_device`, `commands`, `timeout?`, `max_lines?` | 执行 CLI |
| `snmp_get` | `target_device`, `oid(s)`, `timeout?` | 执行 SNMP GET |
| `exec_command` | `target_device`, `commands` | 结构化命令执行 |

### 适合学习什么

- 云中继型远程设备访问
- 当 Agent 无法直连设备时的替代模式

---

## 11. Cisco SD-WAN MCP

### 概览

- Repository: [siddhartha2303/cisco-sdwan-mcp](https://github.com/siddhartha2303/cisco-sdwan-mcp)
- Transport: `stdio`
- Command: `python3 -u $SDWAN_MCP_SCRIPT --transport stdio`
- 主要用途：
  - vManage 只读运维
  - fabric inventory
  - templates / policies
  - alarms / events
  - BFD / OMP / control connections

### 环境变量与调用方式

必需环境变量：

- `VMANAGE_IP`
- `VMANAGE_USERNAME`
- `VMANAGE_PASSWORD`
- `SDWAN_MCP_SCRIPT`
- `MCP_CALL`

调用模板：

```bash
python3 $MCP_CALL "python3 -u $SDWAN_MCP_SCRIPT --transport stdio" <tool_name> '<args_json>'
```

### Tools 参数总表

| Tool | 参数 | 用途 |
|------|------|------|
| `get_devices` | 无 | 列出全 fabric 设备 |
| `get_wan_edge_inventory` | 无 | 查 WAN Edge 清单 |
| `get_device_templates` | 无 | 查 device templates |
| `get_feature_templates` | 无 | 查 feature templates |
| `get_centralized_policies` | 无 | 查 centralized policies |
| `get_alarms` | 无 | 查 alarms |
| `get_events` | 无 | 查 events |
| `get_interface_stats` | `device_ip` | 查接口统计 |
| `get_bfd_sessions` | `device_ip` | 查 BFD 状态 |
| `get_omp_routes` | `device_ip` | 查 OMP 路由 |
| `get_control_connections` | `device_ip` | 查 control plane 连接 |
| `get_running_config` | `device_ip` | 查运行配置 |

### 适合学习什么

- 控制器型 WAN 平台的 read-only MCP 封装
- fabric / template / policy / runtime 四类数据的组织方式

---

## 推荐学习顺序

如果你是为了“学 Cisco MCP 底层工具怎么设计”，建议顺序：

1. `pyATS MCP`
2. `Catalyst Center MCP`
3. `Cisco ACI MCP`
4. `Cisco ISE MCP`
5. `Cisco NSO MCP`
6. `Cisco SD-WAN MCP`
7. `Cisco Meraki MCP`
8. `Cisco ThousandEyes MCP`
9. `Cisco RADKit MCP`
10. `Cisco CML MCP`

---

## 进一步延伸

如果后续要继续扩展这份文档，建议补：

- 每个 tool 的返回结果样例
- 每个 server 的最小可运行 demo
- 每个 server 的只读/可写风险分级
- 每个 server 与 ServiceNow / GAIT 的联动方式

