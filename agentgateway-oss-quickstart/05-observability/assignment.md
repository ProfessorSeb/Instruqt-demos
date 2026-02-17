---
slug: observability
id: nclrrhvfeyya
type: challenge
title: See What Your Agents Are Doing
teaser: Explore the observability data Agentgateway captures for every LLM request.
notes:
- type: text
  contents: "# \U0001F4CA Observability — See What Your Agents Are Doing\n\nThe biggest
    reason to use a gateway: **knowing what your agents actually do**.\n\n**In this
    challenge, you'll:**\n\n- Explore Agentgateway's structured logs\n- Check Prometheus-compatible
    metrics\n- Learn about token tracking, latency, and error rates\n- Preview Enterprise
    observability with Langfuse\n"
tabs:
- id: xcnpwifmfrke
  title: Terminal
  type: terminal
  hostname: server
- id: ubyhrompdgts
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# See What Your Agents Are Doing

You've got a gateway routing traffic to OpenAI models with load balancing. But one of the biggest reasons to use a gateway is **observability** — knowing what your agents are actually doing.

## The Observability Gap

When agents call LLMs directly, you're blind:
- Which agent made the call?
- What model did it use?
- How many tokens did it consume?
- How long did the request take?
- Did it fail? How often?

You might get some of this from your LLM provider's dashboard, but it's fragmented, delayed, and doesn't tie back to your agents.

**Agentgateway captures all of this at the gateway layer** — in real time, across all providers.

## Step 1: Check Agentgateway Logs

First, let's see what pods are running:

```bash
kubectl get pods -n agentgateway-system --show-labels
```

Now check the gateway proxy logs — this is where request-level data lives:

```bash
kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway --tail=50
```

You'll see structured log entries for each request that passed through the gateway, including:
- Request path and method
- Target backend/provider
- Response status code
- Latency

> 💡 If logs are empty, send a few requests first (from the previous challenge) and check again.

## Step 2: Generate Some Traffic

Let's send a few requests so we have data to observe:

```bash
# Make sure port-forward is running
pkill -f "port-forward.*ai-gateway" || true
kubectl port-forward -n agentgateway-system svc/ai-gateway 8080:8080 &
sleep 3

# Send 5 requests
for i in $(seq 1 5); do
  curl -s http://localhost:8080/openai/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "gpt-4o-mini",
      "messages": [{"role": "user", "content": "Count to 3"}],
      "max_tokens": 20
    }' | jq -r '.choices[0].message.content' 2>/dev/null
done
```

Now check the logs again:

```bash
kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway --tail=20
```

## Step 3: Explore Gateway Metrics

Agentgateway exposes Prometheus-compatible metrics. Let's check what's available:

```bash
# Find the proxy pod
AG_POD=$(kubectl get pods -n agentgateway-system -l app.kubernetes.io/name=agentgateway -o jsonpath='{.items[0].metadata.name}')

# Port-forward to the metrics endpoint
kubectl port-forward -n agentgateway-system $AG_POD 9091:9091 &
sleep 2

# Fetch metrics
curl -s http://localhost:9091/metrics | grep -i "agentgateway\|llm\|ai_\|envoy" | head -30
```

These metrics include:
- **Request counts** by provider, model, and status
- **Latency histograms** for LLM calls
- **Token usage** (input and output tokens)
- **Error rates** by provider

## What You Get Out of the Box

**Structured Logs:**
- Every request logged with provider, model, status, latency
- Accessible via `kubectl logs` or any log aggregator (Datadog, Splunk, ELK)

**Prometheus Metrics:**
- Request count by provider/model/status
- Latency histograms (p50, p95, p99)
- Token usage (input/output)
- Error rates

With these, you can answer questions like:
- "Which agent is consuming the most tokens?" → Cost attribution
- "Is GPT-4o slower than GPT-4o-mini for our use case?" → Model comparison
- "How many requests are failing?" → Reliability monitoring
- "What's our total LLM spend this week?" → Budget tracking

All from a single dashboard, regardless of how many models you use.

## Going Further: Langfuse Integration

For production deployments, **Agentgateway Enterprise** integrates with [Langfuse](https://langfuse.com) — an open-source LLM observability platform. This gives you:
- Full prompt and response tracing
- Cost tracking per conversation/session
- Quality evaluation and scoring
- User-level analytics

We'll talk more about Enterprise capabilities in the next challenge.
