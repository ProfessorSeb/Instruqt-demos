---
slug: rate-limiting
id: heyxcz0pc4vb
type: challenge
title: MCP Rate Limiting ⏱️
teaser: Prevent runaway agents from exhausting upstream APIs with rate limiting.
notes:
- type: text
  contents: |
    # ⏱️ MCP Rate Limiting

    AI agents can be... enthusiastic. A coding agent might call GitHub's API hundreds of times in a loop. A data agent might hammer your database.

    Rate limiting prevents runaway agents from exhausting upstream APIs and running up costs.
tabs:
- id: vj37bxd9jtzq
  title: Terminal
  type: terminal
  hostname: server
- id: uxy4jdxa5ukm
  title: Editor
  type: code
  hostname: server
  path: /root
- title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: ""
enhanced_loading: null
---

# MCP Rate Limiting ⏱️

AI agents can enter loops. A coding agent debugging a test might call tools hundreds of times. Without rate limiting, your upstream APIs get hammered, costs spike, and you might get rate-limited by the provider itself.

AgentGateway lets you set rate limits on MCP routes so runaway agents are throttled at the gateway.

## Step 1: Create a Rate Limiting Policy

Let's add rate limiting to the GitHub MCP route — allow 5 requests per minute:

```bash
cat > /root/github-rate-limit.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: github-rate-limit
  namespace: agentgateway-system
spec:
  targetRefs:
  - kind: HTTPRoute
    name: github-mcp-route
  rateLimit:
    requests: 5
    unit: Minute
YAML

kubectl apply -f /root/github-rate-limit.yaml
```

And a more generous limit for Slack — 10 requests per minute:

```bash
cat > /root/slack-rate-limit.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: slack-rate-limit
  namespace: agentgateway-system
spec:
  targetRefs:
  - kind: HTTPRoute
    name: slack-mcp-route
  rateLimit:
    requests: 10
    unit: Minute
YAML

kubectl apply -f /root/slack-rate-limit.yaml
```

## Step 2: Test Rate Limiting 🧪

Let's hit the GitHub endpoint rapidly and see what happens:

```bash
echo "=== Sending 8 rapid requests to GitHub MCP ==="
for i in $(seq 1 8); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/mcp/github \
    -H "Content-Type: application/json" \
    -d "{\"jsonrpc\":\"2.0\",\"method\":\"tools/list\",\"id\":$i}")
  echo "Request $i: HTTP $STATUS"
done
```

The first 5 requests should return `200`. After that, you should see `429 Too Many Requests` — the gateway is protecting your upstream!

## Step 3: Verify the Policies

Check all policies on the cluster:

```bash
kubectl -n agentgateway-system get agentgatewaypolicies
```

You should see four policies:
- `github-read-only` — Tool authorization for GitHub
- `slack-messaging` — Tool authorization for Slack
- `github-rate-limit` — Rate limiting for GitHub
- `slack-rate-limit` — Rate limiting for Slack

🎉 **Rate limiting protects your upstream APIs!** Even if an agent goes into a loop, the gateway throttles it before it can do damage.

> **💡 Pro tip:** In production, you'd combine rate limiting with per-client identification (using auth tokens) so each agent gets its own quota instead of sharing a global limit.
