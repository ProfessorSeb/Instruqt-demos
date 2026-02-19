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

```bash
kubectl logs deploy/agentgateway -n agentgateway-system --tail 20
```

## Step 6: View in Grafana and Solo UI

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
