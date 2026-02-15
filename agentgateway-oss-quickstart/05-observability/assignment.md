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

You've got a gateway routing traffic to multiple LLM providers. But one of the biggest reasons to use a gateway is **observability** — knowing what your agents are actually doing.

## The Observability Gap

When agents call LLMs directly, you're blind:
- Which agent made the call?
- What model did it use?
- How many tokens did it consume?
- How long did the request take?
- Did it fail? How often?

You might get some of this from your LLM provider's dashboard, but it's fragmented across providers, delayed, and doesn't tie back to your agents.

**Agentgateway captures all of this at the gateway layer** — in real time, across all providers.

## Step 1: Check Agentgateway Logs

Agentgateway logs every proxied request. Let's look:

```bash
kubectl logs -n agentgateway-system -l app=agentgateway --tail=50
```

You'll see structured log entries for each request that passed through the gateway, including:
- Request path and method
- Target backend/provider
- Response status code
- Latency

## Step 2: Explore Gateway Metrics

Agentgateway exposes Prometheus-compatible metrics. Let's check what's available:

```bash
# Find the Agentgateway pod
AG_POD=$(kubectl get pods -n agentgateway-system -l app=agentgateway -o jsonpath='{.items[0].metadata.name}')

# Port-forward to the metrics endpoint
kubectl port-forward -n agentgateway-system $AG_POD 9090:9090 &
sleep 2

# Fetch metrics
curl -s http://localhost:9090/metrics | grep -i "agentgateway\|llm\|ai_" | head -30
```

These metrics include:
- **Request counts** by provider, model, and status
- **Latency histograms** for LLM calls
- **Token usage** (input and output tokens)
- **Error rates** by provider

## Step 3: Create an Observability Summary

Let's document what Agentgateway observability gives you out of the box:

```bash
cat > /root/observability-notes.txt << 'EOF'
Agentgateway OSS Observability:

Structured Logs:
- Every request logged with provider, model, status, latency
- Accessible via kubectl logs or any log aggregator

Prometheus Metrics:
- Request count by provider/model/status
- Latency histograms (p50, p95, p99)
- Token usage (input/output)
- Error rates

What Enterprise Adds:
- Langfuse integration for full LLM observability
- Trace correlation across agent chains
- Cost tracking and attribution by team/agent
- Prompt/response content logging (opt-in)
EOF

cat /root/observability-notes.txt
```

## Why This Is a Big Deal

With these metrics, you can answer questions like:
- "Which agent is consuming the most tokens?" → Cost attribution
- "Is OpenAI slower than Anthropic for our use case?" → Provider comparison
- "How many requests are failing?" → Reliability monitoring
- "What's our total LLM spend this week?" → Budget tracking

All from a single dashboard, regardless of how many providers you use.

## Going Further: Langfuse Integration

For production deployments, **Agentgateway Enterprise** integrates with [Langfuse](https://langfuse.com) — an open-source LLM observability platform. This gives you:
- Full prompt and response tracing
- Cost tracking per conversation/session
- Quality evaluation and scoring
- User-level analytics

We'll talk more about Enterprise capabilities in the next challenge.
