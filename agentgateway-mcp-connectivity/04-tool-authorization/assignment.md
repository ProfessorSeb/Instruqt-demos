---
slug: tool-authorization
id: p1xiekxiecht
type: challenge
title: "MCP Tool Authorization \U0001F512"
teaser: Control which tools AI agents can call with fine-grained allow/deny policies.
notes:
- type: text
  contents: "# \U0001F512 MCP Tool Authorization\n\nJust because a tool exists doesn't
    mean every agent should use it. `delete_repo`? `drop_table`? No thanks.\n\nIn
    this challenge, you'll use AgentgatewayPolicy to control exactly which MCP tools
    agents can call using CEL expressions.\n"
tabs:
- id: gaycyxorx6zq
  title: Terminal
  type: terminal
  hostname: server
- id: u7x0soe8jx2y
  title: Editor
  type: code
  hostname: server
  path: /root
- id: gpfibdtuwyyb
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: ""
enhanced_loading: null
---

# MCP Tool Authorization 🔒

Your MCP servers expose tools through the gateway. But should every agent be able to call every tool? **Absolutely not.** 🚫

Agentgateway's `AgentgatewayPolicy` lets you define **MCP authorization rules** using CEL (Common Expression Language) to control which tools agents can access.

## How It Works

There are two approaches:
- **Allow list** — only explicitly matched tools are accessible (deny by default)
- **Deny list** — matched tools are blocked, everything else is allowed

CEL expressions evaluate against `tool.name` for each tool. You can use:
- `tool.name == "fetch"` — exact match
- `tool.name.startsWith('delete')` — prefix match
- `tool.name.contains('admin')` — substring match

## Step 1: See All Available Tools

First, let's confirm what tools are currently accessible:

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq '.result.tools[].name'
```

You should see the `fetch` tool listed. Try calling it:

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"fetch","arguments":{"url":"https://example.com"}},"id":2}' | jq .
```

## Step 2: Create a Deny Policy (Block by Pattern)

Let's create a policy that **denies** any tools matching dangerous patterns. While our MCP server only has `fetch`, this demonstrates how you'd protect against destructive operations in production:

```bash
cat > /root/mcp-tool-auth.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: mcp-tool-auth
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: mcp-route
  backend:
    mcp:
      authorization:
        action: Deny
        policy:
          matchExpressions:
            - "tool.name.startsWith('delete')"
            - "tool.name.startsWith('drop')"
YAML

kubectl apply -f /root/mcp-tool-auth.yaml
```

**What this does:**
- `action: Deny` — matching tools are **blocked**
- Any tool starting with `delete` or `drop` would be filtered from `tools/list` and rejected from `tools/call`
- Non-matching tools (like `fetch`) remain accessible

## Step 3: Verify `fetch` Still Works

Since `fetch` doesn't match any deny pattern, it should still work:

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":3}' | jq '.result.tools[].name'
```

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"fetch","arguments":{"url":"https://example.com"}},"id":4}' | jq .
```

✅ The `fetch` tool works because it doesn't match the deny patterns.

## Step 4: Switch to an Allow List (More Restrictive)

Now let's flip it — instead of denying bad tools, only **allow** the tools we explicitly trust:

```bash
cat > /root/mcp-tool-auth.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: mcp-tool-auth
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: mcp-route
  backend:
    mcp:
      authorization:
        action: Allow
        policy:
          matchExpressions:
            - "tool.name == 'fetch'"
YAML

kubectl apply -f /root/mcp-tool-auth.yaml
```

**What changed:**
- `action: Allow` — **only** matching tools are accessible
- Only `fetch` is allowed — everything else is blocked by default
- This is the **zero-trust approach** — deny by default, allow explicitly

## Step 5: Test the Allow List

`fetch` should still work:

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":5}' | jq '.result.tools[].name'
```

But trying to call any non-allowed tool would be rejected:

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"unauthorized_tool","arguments":{}},"id":6}' | jq .
```

The gateway blocks the call before it ever reaches the MCP server! 🛡️

## Step 6: Verify the Policy

```bash
kubectl -n agentgateway-system get agentgatewaypolicies
kubectl -n agentgateway-system get agentgatewaypolicy mcp-tool-auth -o yaml
```

🎉 **Tool authorization in action!** You've seen both approaches:
- **Deny list** — block known-bad patterns (good for broad protection)
- **Allow list** — only permit known-good tools (zero-trust, more secure)

> **💡 Production recommendation:** Use allow lists in production. It's safer to explicitly allow the tools you need than to try to predict all dangerous ones.
