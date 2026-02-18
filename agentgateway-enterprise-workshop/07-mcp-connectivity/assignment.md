---
slug: mcp-connectivity
id: yzldrxdmsnom
type: challenge
title: MCP Server Connectivity
teaser: Route AI agent tool calls to MCP servers through the gateway with JWT security.
notes:
- type: text
  contents: "# \U0001F527 MCP Server Connectivity\n\nMCP (Model Context Protocol)
    is how AI agents call tools — file access, web search, database queries, and more.\n\n
    Without a gateway, agents connect directly to MCP servers. With AgentGateway:\n\n-
    **Centralized routing** — one entry point for all tool calls\n- **JWT authentication**
    — control who can use which tools\n- **Access logging** — full audit trail of
    tool usage\n- **Metrics and traces** — MCP-specific observability\n\nLet's deploy
    an MCP server, route traffic through the gateway, and lock it down with JWT auth.\n"
tabs:
- id: x2eyyhxrjjvm
  title: Terminal
  type: terminal
  hostname: server
- id: kl1gn9squuli
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: efrastrznxup
  title: Grafana
  type: service
  hostname: server
  port: 3000
difficulty: ""
enhanced_loading: null
---

# MCP Server Connectivity

Let's deploy an MCP server, route it through AgentGateway, and secure it with JWT authentication.

## Step 1: Clean Up Previous Policies

```bash
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway global-request-rate-limit 2>/dev/null || true
kubectl delete ratelimitconfig -n enterprise-agentgateway global-request-rate-limit 2>/dev/null || true
```

## Step 2: Deploy an MCP Server

Deploy the `mcp-website-fetcher` — an MCP server that fetches web page content:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-website-fetcher
  namespace: enterprise-agentgateway
spec:
  selector:
    matchLabels:
      app: mcp-website-fetcher
  template:
    metadata:
      labels:
        app: mcp-website-fetcher
    spec:
      containers:
      - name: mcp-website-fetcher
        image: ghcr.io/peterj/mcp-website-fetcher:main
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-website-fetcher
  namespace: enterprise-agentgateway
  labels:
    app: mcp-website-fetcher
spec:
  selector:
    app: mcp-website-fetcher
  ports:
  - port: 80
    targetPort: 8000
    appProtocol: agentgateway.dev/mcp
EOF
```

Wait for it to be ready:

```bash
kubectl rollout status deployment/mcp-website-fetcher -n enterprise-agentgateway --timeout=120s
```

## Step 3: Create MCP Backend and Route

```bash
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mcp-backend
  namespace: enterprise-agentgateway
spec:
  mcp:
    targets:
    - name: mcp-target
      static:
        host: mcp-website-fetcher.enterprise-agentgateway.svc.cluster.local
        port: 80
        protocol: SSE
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp
  namespace: enterprise-agentgateway
spec:
  parentRefs:
  - name: agentgateway
  rules:
    - backendRefs:
      - name: mcp-backend
        group: agentgateway.dev
        kind: AgentgatewayBackend
EOF
```

## Step 4: Test MCP Connectivity

The MCP protocol requires a session to be initialized before making tool calls. First, initialize a session and capture the session ID:

```bash
source /root/.bashrc

SESSION_ID=$(curl -s -D - "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {},
      "clientInfo": {
        "name": "curl-client",
        "version": "1.0.0"
      }
    }
  }' 2>&1 | grep -i "mcp-session-id" | awk '{print $2}' | tr -d '\r')

echo "Session ID: $SESSION_ID"
```

Now list the available tools through the gateway using the session ID:

```bash
curl -s "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list"
  }' | jq .
```

You should see the `fetch` tool listed. Now call it:

```bash
curl -s "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "fetch",
      "arguments": {
        "url": "https://httpbin.org/get"
      }
    }
  }' | jq .
```

## Step 5: View MCP Access Logs

```bash
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail 2 | jq .
```

Notice MCP-specific fields: `mcp.method`, `mcp.resource`, `mcp.target`.

## Step 6: Secure MCP with JWT Auth

Now let's require JWT authentication to access MCP servers:

```bash
kubectl apply -f - <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: mcp-jwt-auth
  namespace: enterprise-agentgateway
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agentgateway
  traffic:
    jwtAuthentication:
      mode: Strict
      providers:
        - issuer: solo.io
          jwks:
            inline: |
                {
                  "keys": [
                    {
                      "kty": "RSA",
                      "kid": "solo-public-key-001",
                      "n": "vlmc5pb-jYaOq75Y4r91AC2iuS9B0sm6sxzRm3oOG7nIt2F1hHd4AKll2jd6BZg437qvsLdREnbnVrr8kU0drmJNPHL-xbsTz_cQa95GuKb6AI6osAaUAEL3dPjuoqkGNRe1sAJyOi48qtcbV0kPWcwFmCV0-OiqliCms12jrd1PSI_LYiNc3GcutpxY6BiHkbxxNeIuWDxE-i_Obq8EhhGkwha1KVUvLHV-EwD4M_AY8BegGsX-sjoChXOxyueu_ReqWV227I-FTKwMnjwWW0BQkeI6g1w1WqADmtKZ2sLamwGUJgWt4ZgIyhQ-iQfeN1WN2iupTWa5JAsw--CQJw",
                      "e": "AQAB",
                      "use": "sig",
                      "alg": "RS256"
                    }
                  ]
                }
EOF
```

## Step 7: Test MCP Without JWT (Should Fail)

```bash
curl -i "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {},
      "clientInfo": {
        "name": "curl-client",
        "version": "1.0.0"
      }
    }
  }'
```

You should see `403` — `authentication failure: no bearer token found`.

## Step 8: Test MCP With JWT (Should Succeed)

```bash
export DEV_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6InVzZXItaWQiLCJ0ZWFtIjoidGVhbS1pZCIsImV4cCI6MjA3OTU1NjEwNCwibGxtcyI6eyJvcGVuYWkiOlsiZ3B0LTRvIl19fQ.e49g9XE6yrttR9gQAPpT_qcWVKe-bO6A7yJarMDCMCh8PhYs67br00wT6v0Wt8QXMMN09dd8UUEjTunhXqdkF5oeRMXiyVjpTPY4CJeoF1LfKhgebVkJeX8kLhqBYbMXp3cxr2GAmc3gkNfS2XnL2j-bowtVzwNqVI5D8L0heCpYO96xsci37pFP8jz6r5pRNZ597AT5bnYaeu7dHO0a5VGJqiClSyX9lwgVCXaK03zD1EthwPoq34a7MwtGy2mFS_pD1MTnPK86QfW10LCHxtahzGHSQ4jfiL-zp13s8MyDgTkbtanCk_dxURIyynwX54QJC_o5X7ooDc3dxbd8Cw"

# Initialize session with JWT
SESSION_ID=$(curl -s -D - "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {},
      "clientInfo": {
        "name": "curl-client",
        "version": "1.0.0"
      }
    }
  }' 2>&1 | grep -i "mcp-session-id" | awk '{print $2}' | tr -d '\r')

echo "Session ID: $SESSION_ID"

# List tools with session ID and JWT
curl -s "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/list"}' | jq .
```

## Step 9: Add RBAC for Tool Access

Restrict MCP access to users with `org=solo.io`:

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: mcp-rbac
  namespace: enterprise-agentgateway
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agentgateway
  traffic:
    authorization:
      policy:
        matchExpressions:
          - 'jwt.org == "solo.io"'
EOF
```

Test with the JWT (should succeed since org=solo.io). Note: reuse the session from Step 8, or initialize a new one:

```bash
curl -s "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "tools/call", "params": {"name": "fetch", "arguments": {"url": "https://httpbin.org/get"}}}' | jq .
```

## ✅ What You've Learned

- `AgentgatewayBackend` with `mcp.targets` routes to MCP servers
- MCP Streamable HTTP requires session initialization (`initialize` method) before making tool calls
- The `Mcp-Session-Id` header must be included on all requests after initialization
- MCP traffic gets full observability (logs, metrics, traces)
- JWT auth secures MCP endpoints the same way as LLM routes
- CEL-based RBAC controls who can access which tools
- `appProtocol: agentgateway.dev/mcp` on Services enables MCP protocol handling

**Next up:** LLM Failover — build resilient AI architectures with priority groups.
