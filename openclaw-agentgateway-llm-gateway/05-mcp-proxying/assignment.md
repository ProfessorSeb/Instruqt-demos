---
slug: mcp-proxying
id: uoeonaturrmp
type: challenge
title: Proxy MCP Tools Through AgentGateway
teaser: LLMs aren't the only problem. Learn why MCP tool traffic needs a gateway too
  — and how AgentGateway handles both.
notes:
- type: text
  contents: "# \U0001F527 The MCP Problem\n\nModern AI agents don't just call LLMs.
    They call **MCP tools**:\n\n- Filesystem access\n- GitHub repositories\n- Databases
    and APIs\n- Web search and browsing\n- Internal services\n\nEach MCP tool connection
    is a **direct channel** from your agent to an\nexternal service. Same problem
    as direct LLM calls — no visibility, no\ncontrol, no governance.\n\n```\nAgent
    \ ──▶  OpenAI       (uncontrolled)\nAgent  ──▶  GitHub MCP   (uncontrolled)\nAgent
    \ ──▶  Filesystem   (uncontrolled)\nAgent  ──▶  Database MCP (uncontrolled)\n```\n\nAgentGateway
    handles **both** — it's a unified control plane for all\nyour agent's outbound
    traffic, whether LLM or tool.\n\n```\nAgent  ──▶  AgentGateway  ──▶  OpenAI\n
    \                          ──▶  GitHub MCP\n                           ──▶  Filesystem\n
    \                          ──▶  Database MCP\n```"
tabs:
- id: if4tljwlsspx
  title: Terminal
  type: terminal
  hostname: server
- id: 2kjc4xxnsdkm
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: intermediate
timelimit: 900
enhanced_loading: null
---

# Proxy MCP Tools Through AgentGateway

## What is MCP?

**Model Context Protocol (MCP)** is an open standard (from Anthropic) for connecting
AI agents to tools and data sources. Think of it as a USB-C port for AI tools —
one standard protocol that any agent can use to call any tool.

An MCP server exposes **tools** (functions an agent can call), **resources**
(data an agent can read), and **prompts** (reusable templates).

## Start a Local MCP Server

You'll run a simple filesystem MCP server — it lets an agent read and write files:

```bash
# Install the filesystem MCP server
npm install -g @modelcontextprotocol/server-filesystem

# Start it in the background, allowing access to /root/lab-files only
mkdir -p /root/lab-files
echo "This is a test file for MCP access." > /root/lab-files/hello.txt

npx @modelcontextprotocol/server-filesystem /root/lab-files &
MCP_PID=$!
echo "MCP server PID: $MCP_PID"
echo $MCP_PID > /tmp/mcp-server.pid
```

## Configure AgentGateway to Proxy MCP Traffic

Create an `MCPBackend` — the AgentGateway resource for MCP tool servers:

```bash
kubectl apply -f - <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: MCPBackend
metadata:
  name: filesystem
  namespace: agentgateway-system
spec:
  mcp:
    stdio:
      cmd: npx
      args:
        - "@modelcontextprotocol/server-filesystem"
        - "/root/lab-files"
    policies: {}
EOF
```

Verify it was accepted:

```bash
kubectl get mcpbackend -n agentgateway-system
```

## Why Proxy MCP Tools?

The same benefits apply as with LLM proxying:

| Without Gateway | With Gateway |
|---|---|
| Agent connects directly to tool | Gateway mediates the connection |
| Tool credentials in the agent | Credentials in Kubernetes Secrets |
| No visibility into tool calls | Full audit log of every tool invocation |
| Agent can call any tool freely | Gateway enforces which tools are allowed |
| Can't revoke tool access | Delete the MCPBackend — tool goes dark |

## The Kill Switch Applies Here Too

Just like with LLM routes, you can cut off MCP tool access instantly:

```bash
# Revoke filesystem access
kubectl delete mcpbackend filesystem -n agentgateway-system

# Restore it
kubectl apply -f /root/filesystem-mcp.yaml
```

## Inspect Your Full Gateway Config

Now you have both LLM and MCP backends configured. See your full control plane:

```bash
# All backends (LLM + MCP)
kubectl get agentgatewaybackend,mcpbackend -n agentgateway-system

# The gateway routing
kubectl get gateway,aiRoute -n agentgateway-system
```

You now have **one control plane** for all your agent's outbound traffic.

> 💡 **The big picture:** AgentGateway is not an LLM proxy. It's a
> **unified gateway for AI agent traffic** — whatever protocol your agent
> uses to talk to the outside world, AgentGateway can mediate it.
