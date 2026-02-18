---
slug: llm-failover
id: ovyqoul9hfvx
type: challenge
title: LLM Failover & Resilience
teaser: Build resilient AI architectures with priority group failover across LLM providers.
notes:
- type: text
  contents: "# \U0001F504 LLM Failover\n\nWhat happens when your primary LLM provider
    hits rate limits or goes down? Without failover, your agents stop working.\n\nEnterprise
    AgentGateway supports **priority group failover**:\n\n- Group 1 (preferred): Primary
    provider\n- Group 2 (fallback): Secondary provider\n- When Group 1 returns 429/5xx,
    traffic fails over to Group 2\n- When Group 1 recovers, traffic returns automatically\n\n
    This is circuit-breaking for AI — without writing any application code.\n"
tabs:
- id: fcpubxaywvly
  title: Terminal
  type: terminal
  hostname: server
- id: 6jt5daksmchf
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: nmvfrcedmkvk
  title: Grafana
  type: service
  hostname: server
  port: 3000
difficulty: ""
enhanced_loading: null
---

# LLM Failover & Resilience

Let's build a resilient LLM architecture with priority group failover. When the primary backend returns rate limit errors, traffic automatically fails over to the secondary.

## Step 1: Clean Up Previous Resources

```bash
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway mcp-jwt-auth 2>/dev/null || true
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway mcp-rbac 2>/dev/null || true
kubectl delete deployment -n enterprise-agentgateway mcp-website-fetcher 2>/dev/null || true
kubectl delete service -n enterprise-agentgateway mcp-website-fetcher 2>/dev/null || true
kubectl delete agentgatewaybackend -n enterprise-agentgateway mcp-backend 2>/dev/null || true
kubectl delete httproute -n enterprise-agentgateway mcp 2>/dev/null || true
kubectl delete httproute -n enterprise-agentgateway openai 2>/dev/null || true
kubectl delete agentgatewaybackend -n enterprise-agentgateway openai-all-models 2>/dev/null || true
```

## Step 2: Deploy a Mock Server That Always Rate Limits

Deploy a mock OpenAI server that returns 429 (rate limit) for every request:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mock-gpt-4o
  namespace: enterprise-agentgateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mock-gpt-4o
  template:
    metadata:
      labels:
        app: mock-gpt-4o
    spec:
      containers:
      - args:
        - --model
        - mock-gpt-4o
        - --port
        - "8000"
        - --failure-injection-rate
        - "100"
        - --failure-types
        - "rate_limit"
        image: ghcr.io/llm-d/llm-d-inference-sim:latest
        imagePullPolicy: IfNotPresent
        name: vllm-sim
        ports:
        - containerPort: 8000
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: mock-gpt-4o-svc
  namespace: enterprise-agentgateway
spec:
  selector:
    app: mock-gpt-4o
  ports:
    - port: 8000
      targetPort: 8000
      name: http
EOF
```

Wait for it:

```bash
kubectl rollout status deployment/mock-gpt-4o -n enterprise-agentgateway --timeout=120s
```

## Step 3: Create Priority Group Failover Configuration

Configure an `AgentgatewayBackend` with two priority groups:
- **Group 1**: Mock server (always returns 429)
- **Group 2**: Real OpenAI (healthy fallback)

The `ResponseHeaderModifier` adds `Retry-After: 60` so the gateway knows how long to mark the provider as unhealthy:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: failover-route
  namespace: enterprise-agentgateway
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai
      filters:
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: Retry-After
                value: "60"
      backendRefs:
        - name: failover-backend
          group: agentgateway.dev
          kind: AgentgatewayBackend
      timeouts:
        request: "120s"
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: failover-backend
  namespace: enterprise-agentgateway
spec:
  ai:
    groups:
      # Priority Group 1: Mock Server (always returns 429)
      - providers:
          - name: mock-ratelimit-provider
            openai:
              model: "mock-gpt-4o"
            host: mock-gpt-4o-svc.enterprise-agentgateway.svc.cluster.local
            port: 8000
            path: "/v1/chat/completions"
            policies:
              auth:
                passthrough: {}
      # Priority Group 2: Real OpenAI (failover)
      - providers:
          - name: openai-fallback
            openai:
              model: "gpt-4o-mini"
            policies:
              auth:
                secretRef:
                  name: openai-secret
EOF
```

## Step 4: Test Failover Behavior

```bash
source /root/.bashrc
```

**Request 1** — hits the mock server, gets 429:

```bash
echo "=== Request 1 ==="
curl -s -w "\nHTTP Status: %{http_code}\n" "$GATEWAY_IP:8080/openai" \
  -H "Content-Type: application/json" \
  -d '{"model": "", "messages": [{"role": "user", "content": "What is 2+2?"}]}'
```

You should see a 429 rate limit error from the mock server.

**Request 2** — gateway has marked mock as unhealthy, routes to OpenAI:

```bash
echo "=== Request 2 ==="
curl -s -w "\nHTTP Status: %{http_code}\n" "$GATEWAY_IP:8080/openai" \
  -H "Content-Type: application/json" \
  -d '{"model": "", "messages": [{"role": "user", "content": "What is 2+2?"}]}'
```

You should see a 200 response from OpenAI! The gateway failed over to Group 2.

**Request 3** — continues using the healthy failover:

```bash
echo "=== Request 3 ==="
curl -s -w "\nHTTP Status: %{http_code}\n" "$GATEWAY_IP:8080/openai" \
  -H "Content-Type: application/json" \
  -d '{"model": "", "messages": [{"role": "user", "content": "What is 2+2?"}]}'
```

Still 200 from OpenAI — the mock provider stays unhealthy for 60 seconds.

## Step 5: Verify in Access Logs

```bash
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail 5 | \
  jq 'select(.scope == "request") | {status: ."http.status", endpoint: .endpoint}'
```

You should see:
- First request: `endpoint: mock-gpt-4o-svc...`, `status: 429`
- Subsequent requests: `endpoint: api.openai.com:443`, `status: 200`

## How Priority Group Failover Works

1. **Priority ordering**: Group 1 is always preferred when healthy
2. **Health-based failover**: When a provider returns 429, the `Retry-After` header tells the gateway how long to mark it unhealthy
3. **Across-request mechanism**: The first request fails, then subsequent requests route to the fallback
4. **Auto-recovery**: After `Retry-After` seconds, the gateway retries the primary to check if it recovered
5. **Per-pod health state**: Each gateway pod maintains its own health state

## Step 6: View in Grafana

Switch to the **Grafana** tab. In the AgentGateway dashboard, you should see:
- HTTP 429 responses from the mock server
- HTTP 200 responses from OpenAI
- The endpoint switching in real-time

## ✅ What You've Learned

- `AgentgatewayBackend.spec.ai.groups` defines priority groups for failover
- `ResponseHeaderModifier` with `Retry-After` controls unhealthy duration
- Failover works across requests — first request fails, subsequent ones route to fallback
- Health state is per-pod — use single replica for deterministic testing
- This pattern enables: cost-tier failover, multi-provider resilience, circuit-breaking behavior

## 🏆 Workshop Complete!

You've built a complete Enterprise AgentGateway deployment with:

| Capability | What You Did |
|-----------|-------------|
| 🚀 Installation | Enterprise AGW + monitoring + tracing |
| 🌐 LLM Routing | OpenAI backend + HTTPRoute + observability |
| 🔑 API Keys | Vanity key masking with AuthConfig |
| 🛡️ JWT Auth | Token validation + CEL-based RBAC |
| 📝 Guardrails | Prompt enrichment + PII protection |
| 💰 Rate Limiting | Request-based limits with RateLimitConfig |
| 🔧 MCP | Tool routing + JWT security |
| 🔄 Failover | Priority group resilience |

Visit [agentgateway.dev](https://agentgateway.dev) to learn more! 🚀
