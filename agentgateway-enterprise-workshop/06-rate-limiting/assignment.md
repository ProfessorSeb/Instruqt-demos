---
slug: rate-limiting
id: poickihqdxjf
type: challenge
title: Rate Limiting
teaser: Control AI spend with request-based rate limiting at the gateway.
notes:
- type: text
  contents: "# \U0001F4B0 Rate Limiting\n\nWithout rate limiting, a single runaway
    agent can burn through your entire AI budget overnight.\n\nEnterprise AgentGateway
    supports:\n\n- **Request-based** — limit total requests per time window\n- **Token-based
    (local)** — limit tokens per pod\n- **Token-based (global)** — limit tokens across
    all pods\n\n In this challenge, you'll configure request-based rate limiting and
    see it in action.\n"
tabs:
- id: r3axhy5nfyyk
  title: Terminal
  type: terminal
  hostname: server
- id: 2kapzjkdvsnw
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: s4ihyyuyvext
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: solouirate06
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# Rate Limiting

Let's add request-based rate limiting to prevent abuse and control costs.

## Step 1: Clean Up Previous Policies

```bash
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system openai-prompt-enrichment 2>/dev/null || true
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system openai-prompt-guard 2>/dev/null || true
```

## Step 2: Ensure the OpenAI Route Exists

```bash
kubectl get httproute openai -n agentgateway-system || \
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
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
      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend
      timeouts:
        request: "120s"
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-all-models
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai: {}
  policies:
    auth:
      secretRef:
        name: openai-secret
EOF
```

## Step 3: Create the Rate Limit Config

```bash
kubectl apply -f- <<EOF
apiVersion: ratelimit.solo.io/v1alpha1
kind: RateLimitConfig
metadata:
  name: global-request-rate-limit
  namespace: agentgateway-system
spec:
  raw:
    descriptors:
    - key: generic_key
      value: counter
      rateLimit:
        requestsPerUnit: 5
        unit: HOUR
    rateLimits:
    - actions:
      - genericKey:
          descriptorValue: counter
      type: REQUEST
EOF
```

## Step 4: Create the Rate Limiting Policy

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: global-request-rate-limit
  namespace: agentgateway-system
spec:
  targetRefs:
    - name: agentgateway
      group: gateway.networking.k8s.io
      kind: Gateway
  traffic:
    entRateLimit:
      global:
        rateLimitConfigRefs:
        - name: global-request-rate-limit
EOF
```

## Step 5: Test Rate Limiting

```bash
source /root/.bashrc

for i in $(seq 1 7); do
  echo "--- Request $i ---"
  curl -s -o /dev/null -w "HTTP %{http_code}\n" "$GATEWAY_IP:8080/openai" \
    -H "content-type: application/json" \
    -d '{
      "model": "gpt-4o-mini",
      "messages": [{"role": "user", "content": "Count to 1"}]
    }'
  sleep 1
done
```

Requests 1-5: `HTTP 200`, Requests 6-7: `HTTP 429`.

## Step 6: Check Rate Limit Headers

```bash
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

## Step 7: View in Grafana and Solo UI

Switch to the **Grafana** or **Solo UI** tab to see the 429 responses alongside the 200s.

## ✅ What You've Learned

- `RateLimitConfig` defines rate limit rules
- `EnterpriseAgentgatewayPolicy` with `entRateLimit` enforces limits
- 429 responses are returned when limits are exceeded

**Next up:** MCP Server Connectivity.
