# NetClaw Readiness Summary

> Generated: 2026-03-27
> Scope: current local environment under `/home/test_chatpdf_use/netclaw`

---

## Executive Summary

**Conclusion:** NetClaw is **not yet ready for direct operational use** in the current environment.

The repository, OpenClaw binary, and core NetClaw content are present, but the runtime is still blocked at the device-connection layer. The environment can already:

- initialize and use GAIT locally
- read the pyATS testbed and enumerate devices
- reach the configured lab device SSH ports at the TCP layer

The environment cannot yet:

- log in to `R1` via pyATS and execute even a single read-only show command
- run any example in `examples/` end-to-end as written
- perform topology discovery from `R1`

---

## What Was Verified

### Working

| Area | Status | Notes |
|---|---|---|
| Repository layout | OK | `SOUL.md`, `USER.md`, `TOOLS.md`, `AGENTS.md`, `examples/`, `workspace/skills/` all present |
| OpenClaw binary | OK | `openclaw 2026.2.26` installed |
| Node.js / npm | OK | Present and usable |
| GAIT repo init | OK | GAIT can initialize inside this repo when called with the live MCP schema |
| pyATS testbed loading | OK | `pyats_list_devices` succeeds once runtime deps and writable artifact paths are supplied |
| Testbed inventory | OK | 9 devices discovered: `R1`, `R2`, `SW1`, `SW2`, `PC1`, `PC2`, `Server1`, `Server2`, `NetClaw` |
| Basic TCP reachability | OK | SSH `22/tcp` reachable to `10.10.20.171` through `10.10.20.174` |

### Not Working

| Area | Status | Notes |
|---|---|---|
| Default runtime env | FAIL | `~/.openclaw/.env` is empty |
| Default Python runtime deps | FAIL | `pyats`, `mcp`, `fastmcp`, GAIT-related packages were not available in the default runtime |
| pyATS artifact default path | FAIL | default `~/.pyats-mcp/artifacts` is not writable in this sandbox |
| pyATS device login | FAIL | actual connect to `R1` fails before prompt acquisition |
| End-to-end examples | FAIL | all current examples depend on successful pyATS login or other unconfigured systems |

---

## Primary Blockers

### 1. Environment Variables Are Not Preloaded

`~/.openclaw/.env` currently exists but is empty.

That means the following expected values are not present by default:

- `MCP_CALL`
- `PYATS_TESTBED_PATH`
- `PYATS_MCP_SCRIPT`
- `GAIT_MCP_SCRIPT`

### 2. Default Python Runtime Was Incomplete

The stock shell environment did not have the minimum packages needed for GAIT and pyATS MCP execution.

Observed missing modules in the default runtime:

- `pyats`
- `mcp`
- `fastmcp`

### 3. pyATS Can Read the Testbed but Cannot Actually Connect

The key operational failure is at the unicon/pyATS connection layer.

Observed result when attempting a simple read-only command on `R1`:

```text
ConnectionError: failed to connect to R1
generic state is not registered
```

This is the current hard blocker for all workflows that start with:

- `show version`
- `show cdp neighbors detail`
- `show lldp neighbors detail`
- `show ip ospf neighbor`
- `show ip interface brief`

### 4. Sandbox Path Assumptions Do Not Match Reality

Two live mismatches were confirmed:

- `pyATS_MCP` defaults to `~/.pyats-mcp/artifacts`, which is not writable here
- the GAIT skill examples do not exactly match the live GAIT MCP schema

Working live calls observed during validation:

```json
{"path":"."}
```

for `gait_init`, and

```json
{"name":"readiness-check-netclaw-2026-03-27"}
```

for `gait_branch`.

---

## Example Status

No file under `examples/` can currently run end-to-end as written.

| Example | Current Status | Blocking Reason |
|---|---|---|
| `01_health_check.md` | Not runnable | first step needs pyATS login to `R1` |
| `02_vulnerability_audit.md` | Not runnable | needs `show version` and later running-config collection |
| `03_topology_diagram.md` | Not runnable | topology discovery depends on CDP/LLDP/OSPF data from `R1` |
| `04_ospf_mindmap.md` | Not runnable | OSPF collection depends on pyATS login |
| `05_rfc_config.md` | Not runnable | blocked by pyATS and also by missing ServiceNow change workflow setup |
| `06_full_audit.md` | Not runnable | composite workflow built on the blocked steps above |
| `07_cisco_devnet_cml.md` | Not runnable | requires CML plus additional systems such as NetBox and ServiceNow |

### Partial Capability

Some visualization back-ends are conceptually usable if discovery data is supplied manually:

- Draw.io rendering
- Markmap rendering

But the current examples are written as device-driven workflows, so they still do not complete as-is.

---

## Situation for a User Without Cisco Hardware

The current repo assumes network data is collected from devices or lab nodes.

There are two realistic paths for a user who does **not** own Cisco routers:

### Path A. Mock / Demo Mode

Build a local mock workflow that pretends `R1` exists and feeds stored outputs such as:

- `show cdp neighbors detail`
- `show lldp neighbors detail`
- `show ip interface brief`
- `show ip ospf neighbor`

This is the fastest way to make the prompt

> “Discover the network topology from R1 and draw a diagram”

demonstrable on this machine.

### Path B. Virtual Lab Mode

Use a software lab instead of real hardware.

Relevant repo assets already present:

- `lab/frr-testbed/`

This is a real virtual routing lab based on FRR, but it requires container infrastructure to run. The current machine does **not** have the following available:

- `docker`
- `containerlab`
- `podman`
- `qemu`

So the FRR lab is available in the repo as a design, but **not runnable on this host right now**.

---

## Recommended Next Step

If the goal is to get to a usable demo as fast as possible, the recommended order is:

1. **Do not start with real pyATS repair** unless device-driven operation is strictly required.
2. Build a **mock Cisco topology discovery demo** that uses stored command outputs for `R1`.
3. Wire that mock data into the topology-to-diagram flow so the user can run:
   - “Discover the network topology from R1 and draw a diagram”
4. Only after that, decide whether to invest in:
   - fixing the pyATS/unicon runtime, or
   - adding Docker/CML/ContainerLab for a real virtual lab

If the goal is true device-driven readiness, the next technical task is:

- isolate and fix the `unicon` connection failure on `R1`

---

## Files Updated During This Investigation

- `TOOLS.md`
- `memory/2026-03-27.md`
- `.gait/` initialized locally

This document is the higher-level human-readable summary of those findings.
