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
- id: qxwvdkxpymap
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# MCP Server Connectivity

Let's route MCP tool traffic through AgentGateway.

## Step 1: Clean Up Previous Policies

```bash
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system global-request-rate-limit 2>/dev/null || true
kubectl delete ratelimitconfig -n agentgateway-system global-request-rate-limit 2>/dev/null || true
```

## Step 2: Deploy an MCP Server

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-website-fetcher
  namespace: agentgateway-system
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
  namespace: agentgateway-system
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

Wait for it:

```bash
kubectl rollout status deployment/mcp-website-fetcher -n agentgateway-system --timeout=120s
```

## Step 3: Create the MCP Backend and Route

```bash
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mcp-backend
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: mcp-target
      static:
        host: mcp-website-fetcher.agentgateway-system.svc.cluster.local
        port: 80
        protocol: SSE
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp
  namespace: agentgateway-system
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

```bash
source /root/.bashrc

npx -y @modelcontextprotocol/inspector --cli \
  --config /root/mcp-inspector-config.json \
  --server agentgateway \
  --method tools/list
```

Call the tool:

```bash
npx -y @modelcontextprotocol/inspector --cli \
  --config /root/mcp-inspector-config.json \
  --server agentgateway \
  --method tools/call \
  --tool-name fetch \
  --tool-arg url=https://httpbin.org/get
```

## Step 5: View MCP Access Logs

```bash
kubectl logs deploy/agentgateway -n agentgateway-system --tail 3 | jq .
```

---

## Part 2: Securing MCP with JWT Auth and RBAC

## Step 6: Require JWT Authentication

```bash
kubectl apply -f - <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: mcp-jwt-auth
  namespace: agentgateway-system
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

## Step 7: Test Without a JWT (Should Fail)

```bash
npx -y @modelcontextprotocol/inspector --cli \
  --config /root/mcp-inspector-config.json \
  --server agentgateway \
  --method tools/list
```

## Step 8: Test With a JWT (Should Succeed)

```bash
export DEV_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6InVzZXItaWQiLCJ0ZWFtIjoidGVhbS1pZCIsImV4cCI6MjA3OTU1NjEwNCwibGxtcyI6eyJvcGVuYWkiOlsiZ3B0LTRvIl19fQ.e49g9XE6yrttR9gQAPpT_qcWVKe-bO6A7yJarMDCMCh8PhYs67br00wT6v0Wt8QXMMN09dd8UUEjTunhXqdkF5oeRMXiyVjpTPY4CJeoF1LfKhgebVkJeX8kLhqBYbMXp3cxr2GAmc3gkNfS2XnL2j-bowtVzwNqVI5D8L0heCpYO96xsci37pFP8jz6r5pRNZ597AT5bnYaeu7dHO0a5VGJqiClSyX9lwgVCXaK03zD1EthwPoq34a7MwtGy2mFS_pD1MTnPK86QfW10LCHxtahzGHSQ4jfiL-zp13s8MyDgTkbtanCk_dxURIyynwX54QJC_o5X7ooDc3dxbd8Cw"

npx -y @modelcontextprotocol/inspector --cli \
  --config /root/mcp-inspector-config.json \
  --server agentgateway \
  --header "Authorization: Bearer $DEV_TOKEN" \
  --method tools/list
```

Call the tool with auth:

```bash
npx -y @modelcontextprotocol/inspector --cli \
  --config /root/mcp-inspector-config.json \
  --server agentgateway \
  --header "Authorization: Bearer $DEV_TOKEN" \
  --method tools/call \
  --tool-name fetch \
  --tool-arg url=https://httpbin.org/get
```

## Step 9: Add RBAC for Tool Access

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: mcp-rbac
  namespace: agentgateway-system
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

Verify:

```bash
npx -y @modelcontextprotocol/inspector --cli \
  --config /root/mcp-inspector-config.json \
  --server agentgateway \
  --header "Authorization: Bearer $DEV_TOKEN" \
  --method tools/call \
  --tool-name fetch \
  --tool-arg url=https://httpbin.org/get
```

## Step 10: Compare LLM vs MCP Configuration

```bash
echo "=== LLM Backend ==="
kubectl get agentgatewaybackend openai-all-models -n agentgateway-system -o jsonpath='{.spec}' 2>/dev/null | jq . || echo "(not deployed)"

echo ""
echo "=== MCP Backend ==="
kubectl get agentgatewaybackend mcp-backend -n agentgateway-system -o jsonpath='{.spec}' | jq .
```

## ✅ What You've Learned

- **MCP backends** use `AgentgatewayBackend` with `mcp.targets`
- **Same security model** — JWT auth and RBAC apply to MCP traffic the same way
- **Same observability** — MCP requests appear in access logs
- The gateway provides a **single control point** for both LLM and tool traffic

**Next up:** LLM Failover.
