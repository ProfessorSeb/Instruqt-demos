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
- id: jysftkbisgsy
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

> **What's happening:** Removing the prompt enrichment and guardrail policies from the previous challenge to start fresh for rate limiting.

```bash
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system openai-prompt-enrichment 2>/dev/null || true
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system openai-prompt-guard 2>/dev/null || true
```

## Step 2: Ensure the OpenAI Route Exists

> **What's happening:** Confirming the LLM backend and route are in place. Rate limiting is applied *on top of* routing — you need traffic flowing before you can limit it.

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

> **What's happening:** The `RateLimitConfig` defines the actual rate limiting rules. This configuration uses a "generic key" descriptor — a simple global counter that increments for every request regardless of who sends it. The limit is set to **5 requests per hour**, which is intentionally low so you can hit it quickly during the demo. The `type: REQUEST` means this counts HTTP requests (as opposed to token-based limiting, which counts LLM tokens). The rate-limiter service stores counters in Redis (`ext-cache`), so limits are enforced consistently even if you scale to multiple gateway pods.

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

> **What's happening:** The `EnterpriseAgentgatewayPolicy` connects the rate limit rules to the gateway. It targets the Gateway resource (so it applies to all routes) and references the `RateLimitConfig` we just created. When a request comes in, the data plane sends a check to the rate-limiter service, which looks up the counter in Redis, increments it, and returns allow/deny. If the counter exceeds the limit, the gateway returns `429 Too Many Requests` without ever calling OpenAI.

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

> **What's happening:** This loop sends 7 requests in quick succession. The first 5 succeed (HTTP 200) because they're within the 5-requests-per-hour limit. Requests 6 and 7 fail with HTTP 429 (Too Many Requests) because the counter has exceeded the limit. Each request is separated by 1 second to make the output readable. Notice that the 429 responses are returned instantly — the gateway doesn't call OpenAI at all, so rate-limited requests cost you nothing.

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

> **What's happening:** Rate-limited responses include standard HTTP headers that tell the client how many requests remain in the current window and when the limit resets. The `x-ratelimit-limit` header shows the total allowed requests, `x-ratelimit-remaining` shows how many are left, and `x-ratelimit-reset` shows when the counter resets. These headers follow industry conventions, so most HTTP clients and libraries know how to interpret them for automatic retry/backoff logic.

```bash
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

## Step 7: View in Grafana and Solo UI

> **What's happening:** Both 200 and 429 responses appear in traces and metrics. In Grafana, you can see the spike in 429s alongside the successful requests. This visibility is crucial in production — you want to know when rate limits are being hit, by whom (if combined with JWT auth), and how often. It helps you tune limits to balance cost control with user experience.

Switch to the **Grafana** or **Solo UI** tab to see the 429 responses alongside the 200s.

## ✅ What You've Learned

- `RateLimitConfig` defines rate limit rules
- `EnterpriseAgentgatewayPolicy` with `entRateLimit` enforces limits
- 429 responses are returned when limits are exceeded

**Next up:** MCP Server Connectivity.
