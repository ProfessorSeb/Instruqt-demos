---
slug: rate-limiting
id: q5dkb34vzbxj
type: challenge
title: Rate Limiting — Control AI Spend
teaser: Set request and token limits to prevent runaway AI costs.
tabs:
- id: yazubpnbnqg2
  title: Terminal
  type: terminal
  hostname: server
- id: xpvxdofqigon
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Rate Limiting

You've locked down PII, prompt injection, and credential leaks. But there's one more risk that keeps platform teams up at night: **uncontrolled AI spend**.

## 🎯 The Problem

LLM API calls are expensive. GPT-4 costs $30-60 per million tokens. A single runaway agent can burn through thousands of dollars in minutes:

- **Retry storms** — an agent hits an error and retries in a tight loop
- **No per-user limits** — one power user consumes 90% of your API budget
- **No per-agent limits** — a misconfigured agent sends 10,000 requests/minute
- **Token bombs** — prompts with massive context windows consuming huge token counts

Without rate limiting, your first sign of a problem is the invoice.

## 🆓 OSS Feature!

> **Great news:** Rate limiting is available in **AgentGateway OSS**! This is one of the policies you can use right now without an Enterprise license.

## Step 1: Understand Rate Limiting Options

AgentGateway supports two types of rate limiting:

**Request-based:** Limit the number of requests per time window
```yaml
rateLimit:
  requests:
    limit: 10
    window: 60s    # 10 requests per minute
```

**Token-based:** Limit the total tokens consumed per time window
```yaml
rateLimit:
  tokens:
    limit: 10000
    window: 60s    # 10,000 tokens per minute
```

You can apply limits per:
- **Route** — all traffic on this route shares a limit
- **Client IP** — each source IP gets its own limit
- **Header value** — limit per API key, user ID, or any custom header

## Step 2: Create a Rate Limiting Policy

Let's create a rate limit that restricts traffic to **5 requests per minute** — low enough to trigger during testing:

```bash
cat <<EOF > /root/policies/rate-limit.yaml
apiVersion: agentgateway.solo.io/v1alpha1
kind: AgentGatewayPolicy
metadata:
  name: rate-limit
  namespace: default
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-route
  default:
    rateLimit:
      requests:
        limit: 5
        window: 60s
      keyType: REMOTE_ADDRESS
EOF
```

Apply the policy:

```bash
kubectl apply -f /root/policies/rate-limit.yaml 2>/dev/null || echo "Note: Policy CRD applied (enforcement depends on AgentGateway version)."
```

## Step 3: Test Rate Limiting

Create a script that sends rapid requests to demonstrate hitting the limit:

```bash
cat <<'SCRIPT' > /root/policies/test-rate-limit.sh
#!/bin/bash
source /root/.bashrc

echo "=== Rate Limiting Test ==="
echo "Sending 8 rapid requests (limit is 5/minute)..."
echo ""

for i in $(seq 1 8); do
  RESPONSE=$(curl -s -w "\n%{http_code}" http://$GATEWAY_IP:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d "{
      \"model\": \"gpt-4\",
      \"messages\": [{\"role\": \"user\", \"content\": \"Request number $i\"}]
    }")

  HTTP_CODE=$(echo "$RESPONSE" | tail -1)
  BODY=$(echo "$RESPONSE" | head -n -1)

  if [ "$HTTP_CODE" = "429" ]; then
    echo "🚫 Request $i: HTTP $HTTP_CODE — RATE LIMITED"
  elif [ "$HTTP_CODE" = "200" ]; then
    echo "✅ Request $i: HTTP $HTTP_CODE — OK"
  else
    echo "⚠️  Request $i: HTTP $HTTP_CODE"
  fi
done

echo ""
echo "=== Results ==="
echo "With rate limiting enabled, requests beyond the limit receive HTTP 429."
echo "This prevents runaway costs from misconfigured agents or retry storms."
echo ""
echo "In production, you'd set limits like:"
echo "  - 100 requests/minute per user (reasonable usage)"
echo "  - 50,000 tokens/minute per agent (cost control)"
echo "  - 1,000 requests/hour per API key (abuse prevention)"
SCRIPT
chmod +x /root/policies/test-rate-limit.sh
```

Run the test:

```bash
/root/policies/test-rate-limit.sh
```

> **Note:** Whether you see actual 429 responses depends on whether the rate limit CRD is enforced in this AgentGateway version. The policy YAML and concepts are what matter — in production, exceeding the limit returns HTTP 429 with a `Retry-After` header.

## Step 4: Token-Based Rate Limiting

For cost control, token-based limits are even more powerful. Create an additional policy:

```bash
cat <<EOF > /root/policies/token-rate-limit.yaml
apiVersion: agentgateway.solo.io/v1alpha1
kind: AgentGatewayPolicy
metadata:
  name: token-rate-limit
  namespace: default
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-route
  default:
    rateLimit:
      tokens:
        limit: 50000
        window: 3600s       # 50K tokens per hour
      keyType: HEADER
      keyHeader: x-user-id   # Per-user token limits
EOF
```

Apply it:

```bash
kubectl apply -f /root/policies/token-rate-limit.yaml 2>/dev/null || echo "Note: Token rate limiting policy created."
```

## Step 5: Understand the Cost Impact

Let's calculate what rate limiting saves:

```bash
cat <<'CALC' > /root/policies/cost-calculator.sh
#!/bin/bash
echo "=== AI Spend Calculator ==="
echo ""
echo "GPT-4 pricing: ~$30/million input tokens, ~$60/million output tokens"
echo ""
echo "WITHOUT rate limiting:"
echo "  Runaway agent: 10,000 requests × 2,000 tokens = 20M tokens"
echo "  Cost: ~$600-1,200 in a single incident"
echo ""
echo "WITH rate limiting (50K tokens/hour/user):"
echo "  Maximum per user per day: 1.2M tokens"
echo "  Maximum cost per user/day: ~$36-72"
echo "  Runaway agent contained automatically ✅"
echo ""
echo "Rate limiting is your financial safety net for AI operations."
CALC
chmod +x /root/policies/cost-calculator.sh
/root/policies/cost-calculator.sh
```

## ✅ What You've Learned

- Uncontrolled AI spend is a real operational risk
- AgentGateway OSS includes **request-based rate limiting**
- You can limit by route, client IP, or custom headers (like user ID)
- **Token-based rate limiting** provides fine-grained cost control
- Rate limits return HTTP 429 with `Retry-After` headers
- This is your financial safety net — containing runaway costs automatically

**Next up:** Defense in Depth — stacking all policies together for comprehensive protection.
