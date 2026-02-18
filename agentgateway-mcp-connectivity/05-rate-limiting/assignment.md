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

    AI agents can be... enthusiastic. A coding agent might call tools hundreds of times in a loop. A data agent might hammer your database.

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
- id: yllipuhqq5yj
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: ""
enhanced_loading: null
---

# MCP Rate Limiting ⏱️

AI agents can enter loops. A coding agent debugging a test might call tools hundreds of times. Without rate limiting, your upstream APIs get hammered, costs spike, and you might get rate-limited by the provider itself.

Agentgateway lets you set **rate limits** on MCP routes using the `traffic.rateLimit` section of `AgentgatewayPolicy`.

## Step 1: Create a Rate Limiting Policy

Let's add rate limiting to the MCP route — allow **5 requests per minute**:

```bash
cat > /root/mcp-rate-limit.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: mcp-rate-limit
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: mcp-route
  traffic:
    rateLimit:
      local:
      - requests: 5
        unit: Minutes
YAML

kubectl apply -f /root/mcp-rate-limit.yaml
```

Key things to note:
- Rate limiting lives under `spec.traffic.rateLimit` (not under `backend.mcp`)
- `local` is an array — you can define multiple limits
- Each entry specifies `requests` (or `tokens` for token-based) and a `unit` (Seconds, Minutes, or Hours)

## Step 2: Test Rate Limiting 🧪

Let's hit the MCP endpoint rapidly and see what happens:

```bash
echo "=== Sending 8 rapid requests to MCP endpoint ==="
for i in $(seq 1 8); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/mcp \
    -H "Accept: application/json, text/event-stream" \
    -H "Content-Type: application/json" \
    -d "{\"jsonrpc\":\"2.0\",\"method\":\"tools/list\",\"id\":$i}")
  echo "Request $i: HTTP $STATUS"
  sleep 0.5
done
```

The first 5 requests should return `200`. After that, you should see `429 Too Many Requests` — the gateway is protecting your upstream! 🛡️

## Step 3: Verify the Policy

Check the policy on the cluster:

```bash
kubectl -n agentgateway-system get agentgatewaypolicies
```

You can also inspect the full policy:

```bash
kubectl -n agentgateway-system get agentgatewaypolicy mcp-rate-limit -o yaml
```

🎉 **Rate limiting protects your upstream APIs!** Even if an agent goes into a loop, the gateway throttles it before it can do damage.

> **💡 Pro tip:** You can also use token-based rate limiting with `tokens` instead of `requests` — useful for limiting LLM token consumption. Add `burst` to allow short spikes above the limit.
