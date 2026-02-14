---
slug: why-mcp
id: wukxyfx2hypj
type: challenge
title: "Why MCP? \U0001F914 The Tool Integration Problem"
teaser: Understand why AI agents need a standard protocol for connecting to external
  tools.
notes:
- type: text
  contents: "# \U0001F50C The Tool Integration Problem\n\nAI agents don't just chat
    — they **use tools**. They fetch web pages, query databases, create tickets, send
    messages.\n\nBut every tool has a different API. Every integration is custom.
    Sound familiar?\n\n**MCP (Model Context Protocol)** fixes this — and AgentGateway
    makes it secure.\n"
tabs:
- id: 04scbzltkksg
  title: Terminal
  type: terminal
  hostname: server
- id: s6hegfcuen6t
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Why MCP? The Tool Integration Problem 🔌

## The World Before MCP

AI agents need to interact with the real world. They fetch web pages, query databases, search codebases, create issues, and send messages. But **every tool has a different API**:

```
Agent → Custom code → GitHub REST API
Agent → Custom code → Slack API
Agent → Custom code → PostgreSQL driver
Agent → Custom code → Web scraper library
```

Every new tool means new integration code. Every agent reimplements the same connectors. There's no standard way for an agent to discover what tools are available or how to call them.

## Enter MCP: Model Context Protocol 🎯

**MCP** is an open standard that gives AI agents a universal way to connect to tools. Think of it as "USB for AI agents" — one protocol, any tool:

```
Agent → MCP Protocol → Any MCP Server
```

An MCP server exposes:
- **Tools** — Functions the agent can call (e.g., `fetch`, `query_db`, `create_issue`)
- **Resources** — Data the agent can read (e.g., files, database schemas)
- **Prompts** — Pre-built templates for common tasks

The agent uses standard JSON-RPC messages:
- `tools/list` — "What tools do you have?"
- `tools/call` — "Run this tool with these parameters"

## Why AgentGateway for MCP? 🛡️

MCP solves the **integration** problem. But who controls:
- Which agents can call which tools?
- How many calls per minute?
- Who authenticates the requests?
- Where's the audit trail?

**AgentGateway** is the answer. It sits between agents and MCP servers, providing:
- ✅ **Routing** — Direct traffic to the right MCP server
- ✅ **Authorization** — Control which tools agents can use
- ✅ **Rate limiting** — Prevent runaway agents from exhausting APIs
- ✅ **Authentication** — Verify agent identity with OAuth
- ✅ **Observability** — See every tool call, every response

## Your Task 📝

Let's make sure you understand the "why" before we dive into the "how."

**Create a file** at `/root/mcp-benefits.txt` that lists at least 4 benefits of using AgentGateway as an MCP gateway (one per line):

```bash
cat > /root/mcp-benefits.txt << 'EOF'
Benefits of AgentGateway for MCP:
1. Centralized routing to multiple MCP tool servers
2. Fine-grained tool authorization (allow/deny specific tools)
3. Rate limiting to protect upstream tool APIs
4. OAuth authentication for agent identity verification
5. Observability and audit trail for all tool calls
EOF
```

Also, let's verify your cluster is ready and AgentGateway is installed:

```bash
kubectl get pods -n agentgateway-system
```

You should see the AgentGateway pods running. In the next challenge, we'll deploy our first MCP server and route traffic to it! 🚀
