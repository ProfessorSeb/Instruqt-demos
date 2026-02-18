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
- id: mcpinspector01
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: ""
enhanced_loading: null
---

# MCP Server Connectivity

So far we've been routing **LLM traffic** (OpenAI chat completions) through AgentGateway. But modern AI agents don't just talk to LLMs — they also call **tools**. That's where MCP comes in.

**MCP (Model Context Protocol)** is an open standard that defines how AI agents discover and invoke tools — things like fetching web pages, querying databases, reading files, or calling APIs. Each tool runs as an MCP server that exposes its capabilities through a standard protocol.

Without a gateway, agents connect directly to every MCP server, creating the same sprawl problem we solved for LLMs. In this challenge, you'll route MCP tool traffic through AgentGateway, getting centralized routing, observability, and security for free.

## Step 1: Clean Up Previous Policies

Remove the rate limiting resources from the previous challenge:

```bash
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway global-request-rate-limit 2>/dev/null || true
kubectl delete ratelimitconfig -n enterprise-agentgateway global-request-rate-limit 2>/dev/null || true
```

## Step 2: Deploy an MCP Server

Deploy `mcp-website-fetcher` — an MCP server that exposes a single tool called `fetch`, which retrieves the content of a given URL.

This creates two resources:
- A **Deployment** running the MCP server container on port 8000
- A **Service** exposing it on port 80 with `appProtocol: agentgateway.dev/mcp` — this annotation tells AgentGateway to handle this service using the MCP protocol rather than treating it as a regular HTTP backend

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

## Step 3: Create the MCP Backend and Route

Just like we created an `AgentgatewayBackend` for OpenAI in earlier challenges, we now create one for MCP. The difference is the `mcp` section instead of `ai`:

- **AgentgatewayBackend** — defines the MCP server target using `mcp.targets` (instead of `ai.provider` for LLMs). The `protocol: SSE` tells the gateway to communicate with the backend using Server-Sent Events.
- **HTTPRoute** — maps the `/mcp` path to this backend, just like `/openai` mapped to the LLM backend

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

At this point, the MCP server is deployed and reachable through the gateway at `/mcp`. Let's test it.

## Step 4: Test MCP Connectivity with MCP Inspector

Testing MCP servers with `curl` is tricky — the protocol requires session initialization and state management. Instead, we'll use the **[MCP Inspector](https://github.com/modelcontextprotocol/inspector)**, a visual testing tool that handles all of this automatically.

The setup script pre-created a config file at `/root/mcp-inspector-config.json` that points the Inspector at the gateway's MCP endpoint. Launch it:

```bash
source /root/.bashrc

HOST=0.0.0.0 npx -y @modelcontextprotocol/inspector \
  --config /root/mcp-inspector-config.json \
  --server agentgateway &
```

Now switch to the **MCP Inspector** tab in the Instruqt UI. You should see the Inspector interface.

> **Tip:** If the tab shows a blank page, wait 5-10 seconds and refresh — the Inspector takes a moment to start.

Once the UI loads:

1. Click **Connect** — the Inspector initializes an MCP session with the gateway and establishes the connection
2. Click **List Tools** — you should see the `fetch` tool appear. This is the tool exposed by the `mcp-website-fetcher` server, now discoverable through the gateway
3. Click the `fetch` tool, enter `https://httpbin.org/get` in the `url` field, and click **Run Tool** — you should see the HTTP response from httpbin, confirming end-to-end connectivity: **Inspector → Gateway → MCP Server → Internet → back**

When you're done testing, stop the Inspector:

```bash
kill %1 2>/dev/null || true
```

## Step 5: View MCP Access Logs

Every MCP request that flows through the gateway is logged, just like LLM requests:

```bash
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail 3 | jq .
```

Look for MCP-specific fields in the logs:
- `mcp.method` — the MCP operation (e.g., `tools/list`, `tools/call`)
- `mcp.resource` — which resource was accessed
- `mcp.target` — which backend MCP server handled the request

This gives you the same observability for tool calls that you already have for LLM traffic.

---

## Part 2: Securing MCP with JWT Auth and RBAC

Having MCP traffic routed through the gateway is great for observability. But right now, **anyone** who can reach the gateway can call any MCP tool. Let's lock it down.

## Step 6: Require JWT Authentication

Apply a JWT authentication policy — the same approach we used to secure LLM traffic in Challenge 4, now applied to all gateway traffic including MCP:

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

## Step 7: Test Without a JWT (Should Fail)

Launch the Inspector again **without** JWT credentials to confirm the gateway blocks unauthenticated access:

```bash
HOST=0.0.0.0 npx -y @modelcontextprotocol/inspector \
  --config /root/mcp-inspector-config.json \
  --server agentgateway &
```

Switch to the **MCP Inspector** tab and click **Connect**.

You should see a connection error — the gateway is now rejecting all requests without a valid JWT. This confirms that authentication is enforced for MCP traffic, not just LLM traffic.

Stop the Inspector:

```bash
kill %1 2>/dev/null || true
```

## Step 8: Test With a JWT (Should Succeed)

The setup script also created a second config file (`mcp-inspector-config-jwt.json`) that includes the `Authorization: Bearer` header. Launch the Inspector with this config:

```bash
export DEV_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6InVzZXItaWQiLCJ0ZWFtIjoidGVhbS1pZCIsImV4cCI6MjA3OTU1NjEwNCwibGxtcyI6eyJvcGVuYWkiOlsiZ3B0LTRvIl19fQ.e49g9XE6yrttR9gQAPpT_qcWVKe-bO6A7yJarMDCMCh8PhYs67br00wT6v0Wt8QXMMN09dd8UUEjTunhXqdkF5oeRMXiyVjpTPY4CJeoF1LfKhgebVkJeX8kLhqBYbMXp3cxr2GAmc3gkNfS2XnL2j-bowtVzwNqVI5D8L0heCpYO96xsci37pFP8jz6r5pRNZ597AT5bnYaeu7dHO0a5VGJqiClSyX9lwgVCXaK03zD1EthwPoq34a7MwtGy2mFS_pD1MTnPK86QfW10LCHxtahzGHSQ4jfiL-zp13s8MyDgTkbtanCk_dxURIyynwX54QJC_o5X7ooDc3dxbd8Cw"

HOST=0.0.0.0 npx -y @modelcontextprotocol/inspector \
  --config /root/mcp-inspector-config-jwt.json \
  --server agentgateway &
```

Switch to the **MCP Inspector** tab and click **Connect**. This time it should succeed — the JWT is validated by the gateway and the MCP session is established.

1. Click **List Tools** — you should see the `fetch` tool
2. Select `fetch`, enter `https://httpbin.org/get` as the URL, and click **Run Tool** — full end-to-end with authentication

Stop the Inspector:

```bash
kill %1 2>/dev/null || true
```

## Step 9: Add RBAC for Tool Access

JWT authentication confirms **who you are**. RBAC controls **what you can do**. Let's add an authorization policy that restricts MCP access to users whose JWT contains `org=solo.io`:

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

Our JWT has `org=solo.io`, so it should still pass. Verify by launching the Inspector one more time:

```bash
HOST=0.0.0.0 npx -y @modelcontextprotocol/inspector \
  --config /root/mcp-inspector-config-jwt.json \
  --server agentgateway &
```

Switch to the **MCP Inspector** tab, click **Connect**, then **List Tools** and call `fetch` with `https://httpbin.org/get`. Everything should work, confirming that our token passes both authentication (valid JWT) and authorization (`org=solo.io` matches the CEL rule).

A token with a different `org` claim would be rejected at the authorization step — authenticated but not authorized.

Stop the Inspector:

```bash
kill %1 2>/dev/null || true
```

## Step 10: Compare LLM vs MCP Configuration

Let's take a moment to see how the two types of backends compare:

```bash
echo "=== LLM Backend ==="
kubectl get agentgatewaybackend openai-all-models -n enterprise-agentgateway -o jsonpath='{.spec}' 2>/dev/null | jq . || echo "(not deployed in this challenge)"

echo ""
echo "=== MCP Backend ==="
kubectl get agentgatewaybackend mcp-backend -n enterprise-agentgateway -o jsonpath='{.spec}' | jq .
```

The key difference: LLM backends use `spec.ai.provider` to define a cloud LLM provider, while MCP backends use `spec.mcp.targets` to point at an MCP server. Everything else — routing via `HTTPRoute`, security via `EnterpriseAgentgatewayPolicy`, observability — works identically for both.

## ✅ What You've Learned

- **MCP backends** use `AgentgatewayBackend` with `mcp.targets` instead of `ai.provider`
- **`appProtocol: agentgateway.dev/mcp`** on the Kubernetes Service tells the gateway to speak MCP protocol to the backend
- **Same security model** — JWT auth and CEL-based RBAC apply to MCP traffic the same way as LLM traffic
- **Same observability** — MCP requests appear in access logs with `mcp.method`, `mcp.resource`, and `mcp.target` fields
- **MCP Inspector** is a visual tool for testing MCP servers that handles session management automatically
- The gateway provides a **single control point** for both LLM and tool traffic

**Next up:** LLM Failover — build resilient AI architectures with priority groups.
