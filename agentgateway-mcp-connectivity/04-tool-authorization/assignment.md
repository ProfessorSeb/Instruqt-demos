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
difficulty: ""
enhanced_loading: null
---

# MCP Tool Authorization 🔒

Your GitHub MCP server exposes `get_repo`, `list_issues`, `create_issue`, and `delete_repo`. Your Slack server has `list_channels`, `send_message`, `get_messages`, and `delete_channel`.

Should every agent be able to call `delete_repo`? **Absolutely not.** 🚫

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
          action: Allow
YAML

kubectl apply -f /root/github-tool-policy.yaml
```

This policy:
- **Denies all tools by default** (`defaultAction: Deny`)
- **Allows** only tools matching `get_*` or `list_*` patterns
- `create_issue` and `delete_repo` are **blocked** ❌

## Step 2: Create a Messaging Policy for Slack

For Slack, let's allow listing and reading but also sending messages — but block deletions:

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
          action: Allow
        - tools:
          - "delete_*"
          action: Deny
YAML

kubectl apply -f /root/slack-tool-policy.yaml
```

## Step 3: Test the Policies 🧪

Make sure port-forward is running:

```bash
pkill -f "port-forward.*mcp-gateway" || true
kubectl -n agentgateway-system port-forward svc/mcp-gateway 9080:8080 &
sleep 2
```

**Test allowed tools** — `list_issues` on GitHub (should work ✅):

```bash
curl -s http://localhost:9080/mcp/github -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"list_issues","arguments":{"owner":"solo-io","repo":"agentgateway"}},"id":1}' | jq .
```

**Test blocked tools** — `delete_repo` on GitHub (should be denied ❌):

```bash
curl -s http://localhost:9080/mcp/github -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"delete_repo","arguments":{"owner":"solo-io","repo":"agentgateway"}},"id":2}' | jq .
```

**Test Slack** — `send_message` (should work ✅):

```bash
curl -s http://localhost:9080/mcp/slack -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"send_message","arguments":{"channel":"general","text":"Hello from agent!"}},"id":3}' | jq .
```

**Test Slack** — `delete_channel` (should be denied ❌):

```bash
curl -s http://localhost:9080/mcp/slack -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"delete_channel","arguments":{"channel":"general"}},"id":4}' | jq .
```

🎉 **Tool authorization in action!** Agents can use the tools they need, but dangerous operations are blocked at the gateway level — before they ever reach the MCP server.
