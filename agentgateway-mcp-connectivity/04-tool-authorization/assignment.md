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
    agents can call.\n"
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

AgentGateway's `AgentgatewayPolicy` with `toolAuth` lets you control exactly which tools agents can call.

## Step 1: Create a Read-Only Policy for GitHub

Let's create a policy that only allows read-only tools on the GitHub route:

```bash
cat > /root/github-tool-policy.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: github-read-only
  namespace: agentgateway-system
spec:
  targetRefs:
  - kind: HTTPRoute
    name: github-mcp-route
  backend:
    mcp:
      toolAuth:
        defaultAction: Deny
        rules:
        - tools:
          - "get_*"
          - "list_*"
          - "fetch"
          action: Allow
YAML

kubectl apply -f /root/github-tool-policy.yaml
```

This policy:
- **Denies all tools by default** (`defaultAction: Deny`)
- **Allows** only tools matching `get_*`, `list_*`, or `fetch` patterns
- Any other tools are **blocked** ❌

## Step 2: Create a Messaging Policy for Slack

For Slack, let's allow listing and reading but block destructive operations:

```bash
cat > /root/slack-tool-policy.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: slack-messaging
  namespace: agentgateway-system
spec:
  targetRefs:
  - kind: HTTPRoute
    name: slack-mcp-route
  backend:
    mcp:
      toolAuth:
        defaultAction: Deny
        rules:
        - tools:
          - "list_*"
          - "get_*"
          - "send_message"
          - "fetch"
          action: Allow
        - tools:
          - "delete_*"
          action: Deny
YAML

kubectl apply -f /root/slack-tool-policy.yaml
```

## Step 3: Test the Policies 🧪

**Test allowed tools** — `fetch` on GitHub (should work ✅):

```bash
curl -s http://localhost:8080/mcp/github -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

**Test the tool authorization** by calling a tool:

```bash
curl -s http://localhost:8080/mcp/github -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"fetch","arguments":{"url":"https://example.com"}},"id":2}' | jq .
```

You can also verify the policies using the **MCP Inspector** tab.

🎉 **Tool authorization in action!** Agents can use the tools they need, but dangerous operations are blocked at the gateway level — before they ever reach the MCP server.
