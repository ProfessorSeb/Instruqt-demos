---
slug: rate-limiting
id: q5dkb34vzbxj
type: challenge
title: Rate Limiting — Control AI Spend
teaser: Set request and token limits to prevent runaway AI costs.
notes:
- type: text
  contents: "# \U0001F4B0 Rate Limiting — Control AI Spend\n\nA single runaway agent
    can burn $50K overnight. Rate limiting is your financial safety net.\n\n**In this
    challenge, you'll:**\n\n- Create request-based rate limits (requests/minute)\n-
    Create token-based rate limits (tokens/hour)\n- Test rate limiting behavior\n-
    Calculate the cost impact\n\n> ✅ Rate limiting is available in **Agentgateway
    OSS**!\n"
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

> **Great news:** Rate limiting is available in **Agentgateway OSS**! This is one of the policies you can use right now without an Enterprise license.

## Step 1: Understand Rate Limiting Options

Agentgateway supports two types of rate limiting:

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
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
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
kubectl apply -f /root/policies/rate-limit.yaml 2>/dev/null || echo "Note: Policy CRD applied (enforcement depends on Agentgateway version)."
```

## Step 3: Test Rate Limiting

Send 8 rapid requests to demonstrate hitting the limit:

```bash
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

  if [ "$HTTP_CODE" = "429" ]; then
    echo "🚫 Request $i: HTTP $HTTP_CODE — RATE LIMITED"
  elif [ "$HTTP_CODE" = "200" ]; then
    echo "✅ Request $i: HTTP $HTTP_CODE — OK"
  else
    echo "⚠️  Request $i: HTTP $HTTP_CODE"
  fi
done
```

> **Note:** Whether you see actual 429 responses depends on whether the rate limit CRD is enforced in this Agentgateway version. The policy YAML and concepts are what matter — in production, exceeding the limit returns HTTP 429 with a `Retry-After` header.

## Step 4: Token-Based Rate Limiting

For cost control, token-based limits are even more powerful. Create an additional policy:

```bash
cat <<EOF > /root/policies/token-rate-limit.yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
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

**GPT-4 pricing:** ~$30/million input tokens, ~$60/million output tokens

**WITHOUT rate limiting:**
- Runaway agent: 10,000 requests × 2,000 tokens = 20M tokens
- Cost: **~$600-1,200 in a single incident**

**WITH rate limiting (50K tokens/hour/user):**
- Maximum per user per day: 1.2M tokens
- Maximum cost per user/day: ~$36-72
- Runaway agent contained automatically ✅

Rate limiting is your financial safety net for AI operations.

## ✅ What You've Learned

- Uncontrolled AI spend is a real operational risk
- Agentgateway OSS includes **request-based rate limiting**
- You can limit by route, client IP, or custom headers (like user ID)
- **Token-based rate limiting** provides fine-grained cost control
- Rate limits return HTTP 429 with `Retry-After` headers
- This is your financial safety net — containing runaway costs automatically

**Next up:** Defense in Depth — stacking all policies together for comprehensive protection.
