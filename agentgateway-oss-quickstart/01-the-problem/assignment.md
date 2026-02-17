---
slug: the-problem
id: 9ixvzqdyuqbh
type: challenge
title: The Problem — Agents Without Guardrails
teaser: See what happens when AI agents call LLMs directly with no gateway in between.
notes:
- type: text
  contents: "# \U0001F6A8 The Problem: Agents Without Guardrails\n\nEvery AI agent
    today calls LLMs directly — scattered API keys, no audit trail, no rate limits.\n\n**In
    this challenge, you'll see why this is dangerous.**\n\n- Explore what happens
    when agents call LLMs with no gateway\n- Understand the security, cost, and visibility
    gaps\n- Learn why we need a purpose-built AI gateway\n"
tabs:
- id: lfkce2mxhdfv
  title: Terminal
  type: terminal
  hostname: server
- id: gyurmbwdu2ae
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# The Problem: Agents Without Guardrails

Right now, most AI agents look like this:

```
Agent → Direct HTTPS → OpenAI API
Agent → Direct HTTPS → Anthropic API
Agent → Direct HTTPS → MCP Tool Server
```

Every agent has its own API key. Every call goes directly to the provider. There's no central point of control.

Let's see why this is a problem.

## Why This Matters

Imagine you're running 20 agents across 5 teams. Each agent has hardcoded API keys. One team racks up $50K in OpenAI costs overnight because of a retry loop. Another team's agent leaks customer PII in prompts. You find out a week later from your cloud bill.

**There's no audit trail. No rate limiting. No visibility. No guardrails.**

This is the exact same problem we solved for microservices with API gateways and service meshes. Now we need to solve it for AI agents.

## Hands-On: Call an LLM Directly

Let's simulate what a typical agent does — call an LLM directly with no gateway.

Your environment has a real OpenAI API key pre-configured as `$OPENAI_API_KEY`. Let's use it to call OpenAI directly:

```bash
curl -s https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Say hello"}],
    "max_tokens": 50
  }' | jq .
```

You should get a real response from OpenAI. But notice what just happened:

1. **The API key was embedded directly in the request** — if this agent's code is in a repo, the key is in the repo
2. **There's no record of this call** anywhere except OpenAI's dashboard
3. **No rate limiting** — this agent could fire 10,000 requests per second
4. **No content filtering** — anything goes in the prompt and response

## The Problems with Direct LLM Access

Think about what just happened. Scale that to 20 agents across 5 teams:

1. **API keys scattered** across agents and repos
2. **No centralized audit trail** or logging
3. **No rate limiting** or cost controls
4. **No content filtering** or PII protection
5. **No unified observability** across providers
6. **Each team solves these problems independently**

## The Solution

What if there was a single gateway that all your agents talked through? One place to:

- **Manage credentials** (agents never see raw API keys)
- **Audit every request** (who called what, when, with what prompt)
- **Enforce policies** (rate limits, content filtering, cost controls)
- **Observe everything** (latency, tokens, errors — across all providers)

That's exactly what **Agentgateway** does. Let's install it.
