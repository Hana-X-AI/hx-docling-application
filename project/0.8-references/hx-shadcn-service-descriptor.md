# Service Descriptor: hx-shadcn (shadcn/ui MCP Server)

**Service Name**: hx-shadcn
**Node**: hx-shadcn.hx.dev.local
**IP Address**: 192.168.10.229
**Port**: 7423
**Protocol**: Dual (SSE + stdio), MCP (Model Context Protocol)
**Status**: OPERATIONAL
**Operational Date**: 2025-12-09

---

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Service Architecture](#service-architecture)
4. [MCP Tools Reference](#mcp-tools-reference)
5. [MCP Prompts Reference](#mcp-prompts-reference)
6. [Framework Support](#framework-support)
7. [Configuration](#configuration)
8. [Administration](#administration)
9. [Monitoring & Health](#monitoring--health)
10. [Troubleshooting](#troubleshooting)
11. [Related Documentation](#related-documentation)

---

## Overview

### Description

MCP (Model Context Protocol) server providing AI assistants with programmatic access to shadcn/ui components across four frontend frameworks. The service bridges AI assistants (Claude Desktop, Open WebUI, Claude Code) with the shadcn/ui component ecosystem, offering structured access to component source code, metadata, documentation, and pre-built UI blocks.

### Primary Value Proposition

- **Multi-Framework Support**: Access React, Svelte, Vue, and React Native shadcn/ui implementations
- **Component Discovery**: List and retrieve UI components with full source code
- **Block Support**: Pre-built UI blocks for rapid prototyping (React, Svelte, Vue only)
- **AI-Optimized**: MCP protocol integration for seamless AI assistant workflows
- **Caching**: 1-hour TTL cache reduces GitHub API load and improves response times
- **Circuit Breaker**: Graceful degradation when GitHub API is unavailable

### Key Capabilities

| Capability | Description |
|------------|-------------|
| list_components | List all available components for current framework |
| get_component | Retrieve component source code with dependencies |
| get_component_demo | Get usage examples and demo code |
| get_component_metadata | Get props, installation instructions, docs |
| list_blocks | List UI blocks by category |
| get_block | Retrieve block source with all components |
| get_directory_structure | Browse repository file structure |

---

## Quick Start

### Health Check

```bash
# Check service status
ssh agent0@192.168.10.229 "systemctl status shadcn-mcp"

# Run health check script
ssh agent0@192.168.10.229 "/opt/shadcn-mcp/health-check.sh"
```

Expected response:
```
shadcn-mcp health check
=======================
Service Status: active
Port 7423: LISTENING
Process: RUNNING
GitHub API: REACHABLE
Rate Limit: 5000/hour
All checks PASSED
```

### SSE Endpoint Test

```bash
# Verify SSE endpoint is responding
curl -N http://192.168.10.229:7423/sse 2>&1 | head -5
```

---

## Service Architecture

### Deployment Model

- **Type**: Bare-metal Ubuntu 24.04 LTS
- **Runtime**: Node.js 22.x LTS
- **Package**: @jpisnice/shadcn-ui-mcp-server v1.1.4
- **Process Manager**: systemd
- **Service User**: shadcn (system account, nologin)

### Component Stack

```
                            HX-Infrastructure Network
                               192.168.10.0/24
+--------------------------------------------------------------------+
|                                                                    |
|  +------------------+         +--------------------------------+   |
|  |  MCP Clients     |         |   hx-shadcn                    |   |
|  |  - Claude Code   |   SSE   |   192.168.10.229:7423          |   |
|  |  - Open WebUI    |<------->|                                |   |
|  |  - Claude Desktop|         |   @jpisnice/shadcn-ui-mcp      |   |
|  +------------------+         |   v1.1.4                       |   |
|                               |   Node.js 22.x LTS             |   |
|                               |   systemd managed              |   |
|                               +---------------+----------------+   |
|                                               |                    |
+-----------------------------------------------+--------------------+
                                                | HTTPS (443)
                                                v
+--------------------------------------------------------------------+
|                          GitHub API                                 |
|  +----------------+ +------------------+ +------------------+       |
|  | shadcn-ui/ui   | | huntabyte/       | | unovue/          |       |
|  | (React)        | | shadcn-svelte    | | shadcn-vue       |       |
|  | branch: main   | | branch: main     | | branch: dev      |       |
|  +----------------+ +------------------+ +------------------+       |
|  +-------------------------------------------------------------+   |
|  | founded-labs/react-native-reusables (branch: main)          |   |
|  | NOTE: No blocks support for React Native                    |   |
|  +-------------------------------------------------------------+   |
+--------------------------------------------------------------------+
```

### Directory Structure

```
/usr/lib/node_modules/@jpisnice/shadcn-ui-mcp-server/
+-- build/
|   +-- index.js           # Entry point (ExecStart target)
|   +-- server/
|   |   +-- handler.js     # MCP protocol handler
|   +-- tools/
|   |   +-- index.js       # 7 MCP tool definitions
|   +-- prompts/
|   |   +-- index.ts       # 5 MCP prompt definitions
|   +-- utils/
|       +-- axios.js           # React framework handler
|       +-- axios-svelte.js    # Svelte framework handler
|       +-- axios-vue.js       # Vue framework handler
|       +-- axios-react-native.js  # React Native handler

/etc/shadcn-mcp/
+-- env                    # GitHub PAT (mode 400, shadcn:shadcn)
+-- env.backup             # PAT backup (mode 400, shadcn:shadcn)

/etc/systemd/system/
+-- shadcn-mcp.service     # systemd unit file

/opt/shadcn-mcp/
+-- health-check.sh        # Health monitoring script (mode 755)
```

---

## MCP Tools Reference

### list_components

List all available components for the configured framework.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| (none) | - | - | Uses server-configured framework |

**Example Response:**
```json
{
  "components": ["accordion", "alert", "alert-dialog", "avatar", "badge", "button", ...]
}
```

### get_component

Retrieve component source code with dependencies.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| componentName | string | Yes | Name of the component |

**Example:**
```json
{
  "name": "get_component",
  "arguments": {"componentName": "button"}
}
```

### get_component_demo

Get usage examples and demo code.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| componentName | string | Yes | Name of the component |

### get_component_metadata

Get props, installation instructions, and documentation.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| componentName | string | Yes | Name of the component |

### list_blocks

List UI blocks by category. **Note:** Not available for React Native.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| category | string | No | Filter by block category |

**React Native Response:**
```json
{
  "content": [{
    "type": "text",
    "text": "{\"categories\":{},\"totalBlocks\":0,\"availableCategories\":[],\"message\":\"Blocks are not available for React Native framework.\"}"
  }]
}
```

### get_block

Retrieve block source code with all components. **Note:** Not available for React Native.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| blockName | string | Yes | Name of the block |

### get_directory_structure

Browse repository file and folder structure.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| path | string | No | Repository path to browse |

---

## MCP Prompts Reference

The service provides 5 prompts for UI generation workflows.

| Prompt Name | Description | Parameters |
|-------------|-------------|------------|
| build-shadcn-page | Generate complete shadcn/ui page | pageType, features, layout, style |
| create-dashboard | Create comprehensive dashboard UI | dashboardType, widgets, navigation |
| create-auth-flow | Generate authentication pages | authType, providers, features |
| optimize-shadcn-component | Optimize/enhance existing components | component, optimization, useCase |
| create-data-table | Create advanced data tables | dataType, features, actions |

**Prompt Usage Example:**
```json
{
  "method": "prompts/get",
  "params": {
    "name": "build-shadcn-page",
    "arguments": {
      "pageType": "landing",
      "features": ["hero", "pricing", "testimonials"],
      "layout": "single-column",
      "style": "modern"
    }
  }
}
```

---

## Framework Support

### Supported Frameworks

| Framework | Repository | Branch | Blocks Support | Notes |
|-----------|------------|--------|----------------|-------|
| React | shadcn-ui/ui | main | Yes | Primary, stable |
| Svelte | huntabyte/shadcn-svelte | main | Yes | Stable |
| Vue | unovue/shadcn-vue | **dev** | Yes | CAUTION: dev branch, may have breaking changes |
| React Native | founded-labs/react-native-reusables | main | **No** | Limited functionality |

### Framework Selection

Framework is configured server-wide via the `FRAMEWORK` environment variable in the systemd unit file. Default is `react`. To change frameworks:

1. Edit the systemd unit file or environment file
2. Change `FRAMEWORK=react` to desired framework
3. Restart the service: `sudo systemctl restart shadcn-mcp`

---

## Configuration

### Environment Variables

| Variable | Value | Description |
|----------|-------|-------------|
| GITHUB_PERSONAL_ACCESS_TOKEN | (from vault) | GitHub PAT for API authentication |
| MCP_PORT | 7423 | Server listen port |
| MCP_TRANSPORT_MODE | dual | MCP transport protocol (supports: stdio, sse, dual) |
| MCP_HOST | 0.0.0.0 | Server bind address |
| FRAMEWORK | react | Default framework |
| NODE_ENV | production | Runtime environment |
| CACHE_TTL | 3600 | Cache time-to-live (seconds) |

### Transport Modes

The service supports three transport modes via `MCP_TRANSPORT_MODE`:

| Mode | Description | Use Case |
|------|-------------|----------|
| **stdio** | Standard input/output | Claude Desktop local spawning |
| **sse** | Server-Sent Events on port 7423 | Network MCP clients (Open WebUI, Claude Code CLI) |
| **dual** | Both stdio AND SSE simultaneously | **Current configuration** - supports all clients |

**Current Mode: `dual`** - Provides SSE endpoint on port 7423 for network clients while also supporting stdio for local process spawning.

### Claude Desktop Configuration

For Claude Desktop to use this server via stdio (local spawning):

```json
{
  "mcpServers": {
    "shadcn-ui": {
      "command": "node",
      "args": ["/usr/lib/node_modules/@jpisnice/shadcn-ui-mcp-server/build/index.js"],
      "env": {
        "MCP_TRANSPORT_MODE": "stdio",
        "FRAMEWORK": "react",
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<PAT_FROM_VAULT>"
      }
    }
  }
}
```

**Note:** Claude Desktop spawns MCP servers locally, so this configuration runs on the machine with Claude Desktop installed, not on hx-shadcn.

### Network Client Configuration (SSE)

For network clients (Open WebUI, Claude Code), connect via SSE:

```
Endpoint: http://hx-shadcn.hx.dev.local:7423/sse
Health: http://hx-shadcn.hx.dev.local:7423/health
```

**CRITICAL:** The package requires `GITHUB_PERSONAL_ACCESS_TOKEN` (NOT `GITHUB_TOKEN`). Using the wrong variable name results in 60 req/hr rate limit instead of 5000 req/hr.

### Service Account

| Property | Value |
|----------|-------|
| User | shadcn |
| Group | shadcn |
| Shell | /usr/sbin/nologin |
| Home | /nonexistent |
| Type | System account |

### External Dependencies

| Service | Endpoint | Purpose | Fallback |
|---------|----------|---------|----------|
| GitHub API | api.github.com:443 | Component source | Circuit breaker, cache |
| DNS | hx-dc-server (192.168.10.200) | Name resolution | Required |

### Credentials Location

```
Ansible Vault: /home/agent0/HX-Infrastructure/credentials/hx-shadcn-vault.yml
Server Location: /etc/shadcn-mcp/env (mode 400, shadcn:shadcn)
```

---

## Administration

### Service Management

```bash
# Check service status
ssh agent0@192.168.10.229 "systemctl status shadcn-mcp"

# Start service
ssh agent0@192.168.10.229 "sudo systemctl start shadcn-mcp"

# Stop service
ssh agent0@192.168.10.229 "sudo systemctl stop shadcn-mcp"

# Restart service
ssh agent0@192.168.10.229 "sudo systemctl restart shadcn-mcp"

# View logs
ssh agent0@192.168.10.229 "journalctl -u shadcn-mcp -n 50"

# Follow logs in real-time
ssh agent0@192.168.10.229 "journalctl -u shadcn-mcp -f"
```

### Common Operations

**Check GitHub Rate Limit:**
```bash
ssh agent0@192.168.10.229 'source /etc/shadcn-mcp/env && curl -s -H "Authorization: token $GITHUB_PERSONAL_ACCESS_TOKEN" https://api.github.com/rate_limit | jq .rate'
```

**Verify Package Version:**
```bash
ssh agent0@192.168.10.229 "npm list -g @jpisnice/shadcn-ui-mcp-server"
```

**Regenerate PAT from Vault:**
```bash
# On workstation with vault access
ansible-vault view /home/agent0/HX-Infrastructure/credentials/hx-shadcn-vault.yml | grep pat

# Then on server, update /etc/shadcn-mcp/env
```

### Restart Policy

- **Restart**: on-failure
- **RestartSec**: 10 seconds
- **StartLimitBurst**: 3 attempts before giving up

### systemd Unit Files

**Main Service Unit:** `/etc/systemd/system/shadcn-mcp.service`

```ini
[Unit]
Description=shadcn/ui MCP Server
Documentation=https://github.com/jpisnice/shadcn-ui-mcp-server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=shadcn
Group=shadcn

WorkingDirectory=/var/lib/shadcn-mcp
StateDirectory=shadcn-mcp
ConfigurationDirectory=shadcn-mcp
RuntimeDirectory=shadcn-mcp

Environment=NODE_ENV=production
Environment=MCP_PORT=7423
Environment=MCP_TRANSPORT_MODE=sse
Environment=FRAMEWORK=react
EnvironmentFile=/etc/shadcn-mcp/env

ExecStart=/usr/bin/node /usr/lib/node_modules/@jpisnice/shadcn-ui-mcp-server/build/index.js

Restart=always
RestartSec=10
TimeoutStartSec=30
TimeoutStopSec=30

StandardOutput=journal
StandardError=journal
SyslogIdentifier=shadcn-mcp

MemoryMax=512M
CPUQuota=50%
TasksMax=50

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectKernelLogs=true
ProtectControlGroups=true
ProtectClock=true
ProtectHostname=true
RestrictRealtime=true
RestrictSUIDSGID=true
RestrictNamespaces=true
LockPersonality=true
MemoryDenyWriteExecute=false
RemoveIPC=true
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
SystemCallArchitectures=native
CapabilityBoundingSet=
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

[Install]
WantedBy=multi-user.target
```

**Note:** `MemoryDenyWriteExecute=false` is required for Node.js V8 JIT compilation.

**Health Check Service:** `/etc/systemd/system/shadcn-mcp-health.service`

```ini
[Unit]
Description=shadcn-mcp Health Check
After=shadcn-mcp.service

[Service]
Type=oneshot
ExecStart=/opt/shadcn-mcp/health-check.sh
StandardOutput=journal
StandardError=journal
SyslogIdentifier=shadcn-mcp-health
```

**Health Check Timer:** `/etc/systemd/system/shadcn-mcp-health.timer`

```ini
[Unit]
Description=Run shadcn-mcp health check every 5 minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
AccuracySec=1min

[Install]
WantedBy=timers.target
```

**Health Check Script:** `/opt/shadcn-mcp/health-check.sh`

```bash
#!/bin/bash
# shadcn-mcp health check script

# Check 1: Service active
if ! systemctl is-active --quiet shadcn-mcp; then
    echo "FAIL: Service not active"
    exit 1
fi

# Check 2: Port listening
if ! ss -tlnp | grep -q ":7423"; then
    echo "FAIL: Port 7423 not listening"
    exit 1
fi

# Check 3: Health endpoint responds
HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:7423/health 2>/dev/null)
if [ "$HEALTH" != "200" ]; then
    echo "FAIL: Health endpoint returned $HEALTH"
    exit 1
fi

echo "OK: All health checks passed"
exit 0
```

### systemd Hardening Summary

| Directive | Value | Purpose |
|-----------|-------|---------|
| NoNewPrivileges | true | Prevent privilege escalation |
| ProtectSystem | strict | Read-only /usr, /boot, /etc |
| ProtectHome | true | No access to /home |
| PrivateTmp | true | Isolated /tmp |
| PrivateDevices | true | No device access |
| MemoryDenyWriteExecute | false | Required for Node.js JIT |
| SystemCallFilter | @system-service | Allowlist syscalls |
| CapabilityBoundingSet | (empty) | Drop all capabilities |

---

## Monitoring & Health

### Health Check Script

Location: `/opt/shadcn-mcp/health-check.sh`

```bash
# Manual execution
ssh agent0@192.168.10.229 "/opt/shadcn-mcp/health-check.sh"

# Exit codes: 0 = healthy, 1 = unhealthy
```

### Health Check Endpoints

| Check | Command | Success Criteria |
|-------|---------|------------------|
| Service Active | `systemctl is-active shadcn-mcp` | Returns "active" |
| Port Listening | `ss -tlnp \| grep :7423` | Port bound |
| Process Running | `pgrep -f shadcn-ui-mcp-server` | PID returned |

### Key Metrics

| Metric | Threshold | Alert Level |
|--------|-----------|-------------|
| Service status | Not "active" | Critical |
| Port 7423 not listening | Not bound | Critical |
| Response time | >5 seconds | Warning |
| GitHub rate limit | <500 remaining | Warning |
| Memory usage | >400MB | Warning |

### Logging

- **Destination**: systemd journal
- **Identifier**: shadcn-mcp
- **View Command**: `journalctl -u shadcn-mcp`
- **Log Level**: Standard Node.js console output

---

## Troubleshooting

### Service Won't Start

```bash
# Check service status and logs
systemctl status shadcn-mcp
journalctl -u shadcn-mcp -n 100 --no-pager

# Verify Node.js installation
node --version

# Verify package installation
ls -la /usr/lib/node_modules/@jpisnice/shadcn-ui-mcp-server/build/index.js

# Check environment file permissions
ls -la /etc/shadcn-mcp/env
```

### GitHub API Rate Limit Issues

```bash
# Check current rate limit (should show 5000)
source /etc/shadcn-mcp/env
curl -s -H "Authorization: token $GITHUB_PERSONAL_ACCESS_TOKEN" https://api.github.com/rate_limit | jq .rate

# If showing 60, the PAT is not being used correctly
# Verify environment variable name in env file
cat /etc/shadcn-mcp/env | grep GITHUB
# Must be: GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxx
# NOT: GITHUB_TOKEN=ghp_xxx
```

### Port Already in Use

```bash
# Check what's using port 7423
ss -tlnp | grep :7423
lsof -i :7423

# Kill conflicting process if needed
# Then restart service
sudo systemctl restart shadcn-mcp
```

### Cache Issues

```bash
# Restart service to clear in-memory cache
sudo systemctl restart shadcn-mcp
```

### Vue Components Breaking

The Vue framework uses the `dev` branch of unovue/shadcn-vue, which may have breaking changes. If Vue components are returning errors:

1. Check the upstream repository for recent commits
2. Report issue via defect tracking
3. Consider switching to a stable framework temporarily

### React Native Blocks Error

React Native does not support blocks. The service returns an informational message rather than an error. This is expected behavior.

---

## Related Documentation

### Node Documentation

| Document | Location |
|----------|----------|
| Charter | `nodes/hx-shadcn/charter/charter.md` |
| Specification | `nodes/hx-shadcn/specification/node-spec.md` |
| Test Plan | `nodes/hx-shadcn/tests/test-plan.md` |
| Test Suite | `nodes/hx-shadcn/tests/test-suite/` |
| Task Breakdown | `nodes/hx-shadcn/tasks/task-breakdown-summary.md` |

### HX-Infrastructure References

| Document | Location |
|----------|----------|
| Node Inventory | `inventory/nodes.md` |
| Naming Standards | `standards/naming-conventions.md` |
| MCP Standards | `standards/mcp-standards.md` |
| Testing Requirements | `standards/testing-requirements.md` |

### External Documentation

| Resource | URL |
|----------|-----|
| shadcn/ui Documentation | https://ui.shadcn.com |
| MCP Protocol Spec | https://modelcontextprotocol.io |
| Package npm Page | https://www.npmjs.com/package/@jpisnice/shadcn-ui-mcp-server |

### Support Escalation

| Level | Contact | Scope |
|-------|---------|-------|
| L1 | Self-Service | Health checks, basic troubleshooting |
| L2 | william-chen | Infrastructure, systemd, deployment |
| L3 | gordon-zain | MCP protocol, package issues |
| L3 | frank-lucas | Security, credentials, PAT issues |
| L4 | CAIO | Critical issues, architectural decisions |

---

## Known Issues & Limitations

### Vue Framework Development Branch

The Vue framework (unovue/shadcn-vue) uses the `dev` branch which may contain breaking changes. Monitor for issues when using Vue components.

### React Native Blocks Limitation

React Native framework does not support UI blocks. The `list_blocks` and `get_block` tools return an informational message instead of block data.

### No Client Authentication (Phase 1)

Phase 1 deployment relies on network-level isolation. No application-level authentication is implemented. All clients on the HX network can access the service.

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-12-09 | Initial service descriptor | HX-Infrastructure Team |

---

**Last Updated**: 2025-12-09
**Maintained By**: HX-Infrastructure Team (William Chen - Infrastructure Lead)
**Approved By**: CAIO (Jarvis Richardson)
