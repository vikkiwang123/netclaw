# NetClaw 代码仓规模分析

本文档量化 NetClaw 仓库的技能、MCP、工具与代码体量，用于说明**构建该类专业网络自动化系统需要长期积累**（领域规程、集成、文档与可维护代码）。

**统计基准日期**：2026-03-31  
**仓库路径**：仓库根目录（排除 `.git`、`node_modules`）。

---

## 1. SKILL 数量

| 位置 | 说明 | `SKILL.md` 数量 |
|------|------|-----------------|
| `workspace/skills/` | 面向 NetClaw 的业务/网络技能（pyATS、变更流程、Batfish、厂商与云平台等） | **108** |
| `.claude/skills/` | 通用技能（pptx、docx、mcp-builder、前端等） | **18** |
| **合计** | | **126** |

README 中宣传的「101 skills」通常指面向用户的 **NetClaw 技能子集**或某次发布快照；以仓库内文件为准时，主技能库为 **108**，若含 `.claude/skills` 则为 **126**。

### `workspace/skills` 行数（规程密度）

| 类型 | 文件数 | 行数 |
|------|--------|------|
| `.md` | 108 | 约 **16,060**（均为技能目录内 Markdown，主体为 `SKILL.md`） |
| `.js` | 13 | 约 3,127 |
| `.css` | 7 | 约 1,101 |

**要点**：仅主技能库中 Markdown 规程即超过 **1.6 万行**，体现「知识工程」而非零散提示词。

### `.claude/skills` 行数

- **`SKILL.md`**：18 个文件，合计约 **2,871** 行。

---

## 2. MCP 数量

| 口径 | 数量 | 说明 |
|------|------|------|
| 本仓 **自研 MCP 服务**（`mcp-servers/` 下独立子项目） | **5** | `azure-network-mcp`、`batfish-mcp`、`gnmi-mcp`、`protocol-mcp`、`suzieq-mcp` |
| README **「MCP integrations」** | **46** | OpenClaw/网关侧**配置的集成总数**（含外部服务、多实例等），**不等于** `mcp-servers/` 目录个数 |

对外表述时需区分：**自研服务端数量**用 **5**；**平台可对接能力**可引用 README 的 **46** 并注明为集成配置维度。

---

## 3. MCP Tools 数量（自研服务暴露的工具）

在 5 个自研 MCP 中，通过 FastMCP 的 `@mcp.tool()` / `mcp.tool()(...)` 注册的工具**合计约 53**：

| 服务 | 工具数（约） |
|------|----------------|
| suzieq-mcp | 5 |
| protocol-mcp | 10 |
| batfish-mcp | 8 |
| gnmi-mcp | 10 |
| azure-network-mcp | 20（含订阅列表 + 各网络能力注册） |

**合计**：5 + 10 + 8 + 10 + 20 = **53**

（若 Azure 侧注释与注册行不一致，以 `azure_network_mcp_server.py` 中实际 `mcp.tool()` 注册为准。）

---

## 4. 代码量

### 4.1 Git 跟踪文件

- `git ls-files`：**约 391** 个跟踪文件（随提交变化）。

### 4.2 按后缀汇总（源码 + 文档，PowerShell 行计数）

**范围**：`.py` `.md` `.ts` `.tsx` `.js` `.mjs` `.yaml` `.yml` `.json` `.sh` `.ps1` `.css` `.html`

| 后缀 | 文件数 | 行数 |
|------|--------|------|
| `.js` | 37 | 46,738 |
| `.md` | 265 | 41,881 |
| `.py` | 159 | 31,645 |
| `.sh` | 12 | 5,148 |
| `.json` | 27 | 2,765 |
| `.css` | 8 | 2,115 |
| `.html` | 5 | 2,048 |
| `.yaml` / `.yml` | 3 | 269 |
| **合计** | **516** | **132,609** |

**说明**：

- `.js` 行数含前端（如 `ui/`），偏高属正常。
- **Python 约 3.16 万行**可单独强调为自研自动化、MCP 与脚本核心逻辑。
- 本统计在 **未安装 Perl / 官方 `cloc`** 环境下用 PowerShell 生成；若需与业界口径完全一致，可在具备 `cloc` 的环境再跑一遍作交叉校验。

---

## 5. 专业性归纳（论证「需要积累」）

1. **领域覆盖**：技能库横跨多厂商网络（Cisco/pyATS、Junos、F5、Palo Alto、Meraki 等）、公有云（Azure/AWS/GCP）、可观测与安全（Prometheus、Grafana、ISE、NVD、防火墙策略）、源与编排（NetBox、Nautobot、NSO、Batfish、SuzieQ、gNMI）等，形成**运维闭环**而非单点脚本。
2. **工程与合规**：`AGENTS.md` 与变更类技能强调 ServiceNow、基线、验证、GAIT 审计；部分 MCP（如 Azure）带审计日志思路，贴近**生产级操作习惯**。
3. **实现深度**：例如 `protocol-mcp` 含大量 BGP/OSPF 等控制面相关实现；`batfish-mcp` 为千行级分析服务；体现不仅是「调 API」，还有**分析与协议层**投入。
4. **知识与文档体量**：数万行 Markdown 技能与文档 + 体量很大的 `README`，属于**可复用规程与领域知识**，需持续迭代才能稳定。

**可作结论的表述**：

> NetClaw 在单仓内可量化为：百余条结构化技能（主库 108 个 `SKILL.md`）、5 个自研 MCP、五十余个 MCP 工具、十余万行级的文档与代码（选定后缀合计约 13.3 万行）；并跨网络自动化多栈。此类系统依赖**领域规程、集成与文档的长期沉淀**，难以用短期原型替代。

---

## 6. 复现统计（维护者）

在 Windows PowerShell 仓库根目录可复现「按后缀行数」：

```powershell
$exclude = '(\\node_modules\\|\\.git\\)'
$allowed = '^(\.py|\.md|\.ts|\.tsx|\.js|\.mjs|\.yaml|\.yml|\.json|\.sh|\.ps1|\.css|\.html)$'
Get-ChildItem -Recurse -File |
  Where-Object { $_.FullName -notmatch $exclude -and $_.Extension -match $allowed } |
  ForEach-Object {
    $n = (Get-Content -LiteralPath $_.FullName -ErrorAction SilentlyContinue | Measure-Object -Line).Lines
    [PSCustomObject]@{ Ext=$_.Extension.ToLower(); Lines=$n }
  } | Group-Object Ext | ForEach-Object {
    $sum = ($_.Group | Measure-Object -Property Lines -Sum).Sum
    [PSCustomObject]@{ Ext=$_.Name; Files=$_.Count; Lines=$sum }
  } | Sort-Object Lines -Descending
```

MCP 工具数量可在 `mcp-servers/` 下检索 `@mcp.tool` 与 `mcp.tool(` 注册。

---

## 相关文档

- [NetClaw 就绪摘要（2026-03-27）](./netclaw-readiness-summary-2026-03-27.md)
- 仓库根目录 [README.md](../README.md)
