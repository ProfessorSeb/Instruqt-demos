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

## Step 1: See All Available Tools

First, let's confirm that all tools are currently accessible:

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq '.result.tools[].name'
```

You should see all tools listed — including potentially dangerous ones like `delete_*` or `drop_*` operations. 😬

Try calling a tool to confirm it works:

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"fetch","arguments":{"url":"https://example.com"}},"id":2}' | jq .
```

## Step 2: Create a Tool Authorization Policy 🛡️

Let's create a policy that **denies** any tools starting with `delete` or `drop` — blocking destructive operations at the gateway:

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

This policy uses **CEL expressions** to match tools:
- `action: Deny` — matching tools are **blocked**
- `matchExpressions` — array of CEL expressions evaluated against each tool
- `tool.name` — the CEL variable representing the tool's name

## Step 3: Verify the Policy Works 🧪

**Check that denied tools are filtered from `tools/list`:**

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":3}' | jq '.result.tools[].name'
```

Any tools starting with `delete` or `drop` should no longer appear in the list! ✅

**Try calling a denied tool directly (should be rejected):**

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"delete_something","arguments":{}},"id":4}' | jq .
```

The gateway blocks the call before it ever reaches the MCP server! 🛡️

**Verify the policy exists on the cluster:**

```bash
kubectl -n agentgateway-system get agentgatewaypolicies
```

🎉 **Tool authorization in action!** Agents can use the tools they need, but dangerous operations are blocked at the gateway level — before they ever reach the MCP server.

> **💡 Pro tip:** You can also use `action: Allow` with `matchExpressions` to create an allowlist instead — only explicitly matched tools will be accessible. CEL gives you powerful pattern matching beyond just prefixes!
