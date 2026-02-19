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
- id: 8uhaikdn3kjh
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# LLM Failover & Resilience

Let's build a resilient LLM architecture with priority group failover.

## Step 1: Clean Up Previous Resources

> **What's happening:** This is a full cleanup — removing all resources from previous challenges (auth policies, MCP server, backends, routes). We're starting with a clean slate because the failover configuration uses a completely different backend structure (priority groups) that replaces the single-provider backend from earlier challenges.

```bash
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system mcp-jwt-auth 2>/dev/null || true
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system mcp-rbac 2>/dev/null || true
kubectl delete deployment -n agentgateway-system mcp-website-fetcher 2>/dev/null || true
kubectl delete service -n agentgateway-system mcp-website-fetcher 2>/dev/null || true
kubectl delete agentgatewaybackend -n agentgateway-system mcp-backend 2>/dev/null || true
kubectl delete httproute -n agentgateway-system mcp 2>/dev/null || true
kubectl delete httproute -n agentgateway-system openai 2>/dev/null || true
kubectl delete agentgatewaybackend -n agentgateway-system openai-all-models 2>/dev/null || true
```

## Step 2: Deploy a Mock Server That Always Rate Limits

> **What's happening:** You're deploying a mock LLM server (`llm-d-inference-sim`) that simulates a broken or overloaded provider. The `--failure-injection-rate 100` flag means it returns a `429 Too Many Requests` error for *every single request* — 100% failure rate. This simulates what happens when your primary LLM provider hits rate limits or goes down. The mock server speaks the OpenAI API format, so the gateway treats it like a real provider. Having a deterministic failure source lets you reliably test failover behavior.

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mock-gpt-4o
  namespace: agentgateway-system
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
  namespace: agentgateway-system
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
kubectl rollout status deployment/mock-gpt-4o -n agentgateway-system --timeout=120s
```

## Step 3: Create Priority Group Failover Configuration

> **What's happening:** This is the core of the failover architecture. The `AgentgatewayBackend` defines two **priority groups** as an ordered array under `spec.ai.groups`. **Group 1** (first in the array) is the "preferred" provider — the mock server that always returns 429. **Group 2** (second in the array) is the fallback — real OpenAI with `gpt-4o-mini`. The gateway tries Group 1 first. When it gets a 429 or 5xx, it marks that provider as unhealthy and routes subsequent requests to Group 2. When Group 1 recovers, traffic automatically returns to it. The `model: ""` in the route means the gateway uses whatever model is configured in the backend group. The `passthrough: {}` auth on the mock server means no API key is needed (it's just a local simulator).

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: failover-route
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system
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
  namespace: agentgateway-system
spec:
  ai:
    groups:
      # Priority Group 1: Mock Server (always returns 429)
      - providers:
          - name: mock-ratelimit-provider
            openai:
              model: "mock-gpt-4o"
            host: mock-gpt-4o-svc.agentgateway-system.svc.cluster.local
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

> **What's happening:** You're sending three requests one at a time to observe the failover progression. **Request 1** hits the mock server (Group 1), gets a 429, and the gateway marks Group 1 as unhealthy. Depending on timing, Request 1 may itself be retried to Group 2 and succeed, or it may return the 429. **Requests 2 and 3** are routed directly to Group 2 (real OpenAI) because Group 1 is still marked unhealthy. You'll see successful responses from `gpt-4o-mini`. This is circuit-breaking for AI: the gateway automatically detects provider failures and reroutes traffic, all without any changes to the client application.

```bash
source /root/.bashrc
```

```bash
echo "=== Request 1 ==="
curl -s -w "\nHTTP Status: %{http_code}\n" "$GATEWAY_IP:8080/openai" \
  -H "Content-Type: application/json" \
  -d '{"model": "", "messages": [{"role": "user", "content": "What is 2+2?"}]}'
```

```bash
echo "=== Request 2 ==="
curl -s -w "\nHTTP Status: %{http_code}\n" "$GATEWAY_IP:8080/openai" \
  -H "Content-Type: application/json" \
  -d '{"model": "", "messages": [{"role": "user", "content": "What is 2+2?"}]}'
```

```bash
echo "=== Request 3 ==="
curl -s -w "\nHTTP Status: %{http_code}\n" "$GATEWAY_IP:8080/openai" \
  -H "Content-Type: application/json" \
  -d '{"model": "", "messages": [{"role": "user", "content": "What is 2+2?"}]}'
```

## Step 5: Verify in Access Logs

> **What's happening:** The access logs show the full failover story. You'll see log entries for the failed request to the mock server (429 status, `mock-gpt-4o` model) and successful requests to OpenAI (`gpt-4o-mini` model, 200 status). The `gen_ai.response.model` field in each log entry tells you which provider actually served each request. This is critical for production operations: you can build alerts for when failover activates and dashboards showing traffic distribution across providers.

```bash
kubectl logs deploy/agentgateway -n agentgateway-system --tail 20
```

## Step 6: View in Grafana and Solo UI

> **What's happening:** In Grafana/Tempo and the Solo UI, you'll see traces that show the failover path: initial attempt to Group 1, 429 response, retry to Group 2, successful response. The traces make the failover behavior visually clear — you can see exactly when the circuit opened, which provider served each request, and the end-to-end latency including the failed attempt.

Switch to the **Grafana** or **Solo UI** tab to see failover traces.

## ✅ What You've Learned

- `AgentgatewayBackend.spec.ai.groups` defines priority groups for failover
- Failover works across requests — first request fails, subsequent ones route to fallback
- This pattern enables: cost-tier failover, multi-provider resilience, circuit-breaking

## 🏆 Workshop Complete!

| Capability | What You Did |
|-----------|-------------|
| 🚀 Installation | Enterprise AGW + Solo UI + monitoring + tracing |
| 🌐 LLM Routing | OpenAI backend + HTTPRoute + observability |
| 🔑 API Keys | Vanity key masking with AuthConfig |
| 🛡️ JWT Auth | Token validation + CEL-based RBAC |
| 📝 Guardrails | Prompt enrichment + PII protection |
| 💰 Rate Limiting | Request-based limits with RateLimitConfig |
| 🔧 MCP | Tool routing + JWT security |
| 🔄 Failover | Priority group resilience |

Visit [agentgateway.dev](https://agentgateway.dev) to learn more! 🚀
