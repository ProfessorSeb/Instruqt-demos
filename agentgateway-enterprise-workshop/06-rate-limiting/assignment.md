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
difficulty: ""
enhanced_loading: null
---

# Rate Limiting

Let's add request-based rate limiting to prevent abuse and control costs.

## Step 1: Clean Up Previous Policies

```bash
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway openai-prompt-enrichment 2>/dev/null || true
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway openai-prompt-guard 2>/dev/null || true
```

## Step 2: Ensure the OpenAI Route Exists

```bash
kubectl get httproute openai -n enterprise-agentgateway || \
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
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
  namespace: enterprise-agentgateway
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

This creates a global rate limit of **5 requests per hour** using a `RateLimitConfig`:

```bash
kubectl apply -f- <<EOF
apiVersion: ratelimit.solo.io/v1alpha1
kind: RateLimitConfig
metadata:
  name: global-request-rate-limit
  namespace: enterprise-agentgateway
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
  namespace: enterprise-agentgateway
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

Send requests and watch for the rate limit to kick in:

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

You should see:
- Requests 1-5: `HTTP 200` (allowed)
- Requests 6-7: `HTTP 429` (rate limited)

## Step 6: Check Rate Limit Headers

Send a request and look at the response headers:

```bash
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

You'll see rate limit headers like `x-ratelimit-limit` and `x-ratelimit-remaining`.

## Step 7: View in Grafana

Switch to the **Grafana** tab. In the AgentGateway dashboard, you should see the 429 responses in the HTTP status code distribution.

## Cleanup

```bash
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway global-request-rate-limit
kubectl delete ratelimitconfig -n enterprise-agentgateway global-request-rate-limit
```

## ✅ What You've Learned

- `RateLimitConfig` defines rate limit rules (requests per unit)
- `EnterpriseAgentgatewayPolicy` with `entRateLimit` enforces limits on the Gateway
- Rate limiting uses the shared Redis cache + rate limiter service
- 429 responses are returned when limits are exceeded
- Rate limit headers provide transparency to clients

**Next up:** MCP Server Connectivity — route AI agent tool calls through the gateway.
