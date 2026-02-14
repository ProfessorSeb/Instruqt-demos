---
slug: the-problem
id: 9ixvzqdyuqbh
type: challenge
title: The Problem — Agents Without Guardrails
teaser: See what happens when AI agents call LLMs directly with no gateway in between.
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

First, let's set a placeholder API key (we'll use a real one later):

```bash
export OPENAI_API_KEY="sk-placeholder-not-a-real-key"
```

Now, try calling OpenAI directly:

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

You'll get an authentication error — that's expected. But notice what just happened:

1. **The API key was embedded directly in the request** — if this agent's code is in a repo, the key is in the repo
2. **There's no record of this call** anywhere except OpenAI's dashboard
3. **No rate limiting** — this agent could fire 10,000 requests per second
4. **No content filtering** — anything goes in the prompt and response

## Create a File to Record What You've Learned

Create a file that captures the problems with direct LLM access:

```bash
cat > /root/problems.txt << 'EOF'
Problems with direct agent-to-LLM communication:
1. API keys scattered across agents and repos
2. No centralized audit trail or logging
3. No rate limiting or cost controls
4. No content filtering or PII protection
5. No unified observability across providers
6. Each team solves these problems independently
EOF
```

Now view it:

```bash
cat /root/problems.txt
```

## The Solution

What if there was a single gateway that all your agents talked through? One place to:

- **Manage credentials** (agents never see raw API keys)
- **Audit every request** (who called what, when, with what prompt)
- **Enforce policies** (rate limits, content filtering, cost controls)
- **Observe everything** (latency, tokens, errors — across all providers)

That's exactly what **AgentGateway** does. Let's install it.
