---
slug: the-problem
id: kcx1ptiuxszq
type: challenge
title: 'The Problem: Direct LLM Calls'
teaser: Understand why calling LLMs directly from your agent is a serious problem
  in production.
notes:
- type: text
  contents: |-
    # ❌ The Problem with Direct LLM Calls

    Before we build the solution, let's understand the problem.

    Most AI agents look like this:

    ```
    Your Agent  ──────────────────▶  OpenAI API
                  OPENAI_API_KEY
    ```

    This works great on your laptop. It falls apart in production.

    **In this challenge, you'll see exactly what goes wrong — and why it matters.**
tabs:
- id: 3ij92wohaw6q
  title: Terminal
  type: terminal
  hostname: server
- id: tpefhunisexy
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 600
enhanced_loading: null
---

# The Problem: Direct LLM Calls

## What the "Happy Path" Looks Like

Most developers start here. Set an API key, call the API:

```bash
curl -s https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello!"}]
  }' | jq '.choices[0].message.content'
```

It works. Ship it. ✅

## Now Ask Yourself These Questions

**1. Where is your API key?**

```bash
echo $OPENAI_API_KEY
```

It's in your environment. Every process on this machine can read it. Every container
that inherits env vars has it. Every developer on your team needs a copy.

**2. How much has this agent spent today?**

```bash
# How would you even check this?
# There's no built-in answer.
echo "You don't know."
```

There's no counter. No budget. No alert when you've burned $500 in a runaway loop.

**3. What did your agent actually send to OpenAI?**

```bash
# Can you see the exact prompt? The system message? What was injected?
echo "You can't. It's gone."
```

No logs. No traces. No audit trail. If something goes wrong — a prompt injection,
unexpected behavior, a data leak — you have no visibility into what happened.

**4. Can you stop it right now?**

```bash
# Your agent is in a runaway loop, burning tokens.
# How do you stop it without killing the whole application?
echo "You can't. Not without killing the process."
```

**5. What about MCP tools?**

Modern AI agents don't just call LLMs — they call **MCP tools** (filesystem, GitHub,
databases, APIs). Every tool call is a direct connection from your agent to an
external service. Same problems: no visibility, no control, no governance.

---

## The Pattern We've Seen Before

> *"AI agents calling LLMs directly is the same chaos we saw with microservices in 2016,
> before service meshes and API gateways brought order."*

The solution then was: **put a gateway in the middle**.

The solution now is the same.

```
Before:
  Agent  ──▶  OpenAI (direct, invisible, uncontrolled)

After:
  Agent  ──▶  AgentGateway  ──▶  OpenAI
              (observe, control, govern)
```

In the next challenge, you'll deploy that gateway.

> 💡 **Key insight:** AgentGateway isn't just a proxy. It's a control plane for your
> entire AI agent traffic — LLMs *and* MCP tools.
