---
slug: kill-switch
id: q7ax1dcb9y0g
type: challenge
title: 'The Kill Switch: Real-Time LLM Access Control'
teaser: Cut off LLM access instantly without touching your application — and restore
  it just as fast.
notes:
- type: text
  contents: |-
    # ☠️ The Kill Switch

    Your AI agent is in a runaway loop. It's been running for 10 minutes,
    burning tokens, and you have no idea what it's sending.

    Without a gateway: you kill the entire application.
    With AgentGateway: you flip a switch.

    Because **all** LLM traffic flows through one gateway, you have a single
    point of control. Delete a route — every agent behind it goes dark instantly.
    Restore it — they're back up.

    No application changes. No redeployments. No credential rotation.

    **This challenge shows you exactly how it works.**
tabs:
- id: q6kslpxrgaxg
  title: Terminal
  type: terminal
  hostname: server
- id: ssrdde3x1eyr
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 600
enhanced_loading: null
---

# The Kill Switch: Real-Time LLM Access Control

## Verify Traffic is Flowing

First, confirm your LLM proxy is working:

```bash
curl -s http://localhost:8080/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Am I connected?"}]}' \
  | jq '.choices[0].message.content'
```

You should get a response. Good — now watch what happens next.

## Kill It

Delete the `AIRoute`. This removes the path from the gateway to OpenAI:

```bash
kubectl delete httproute openai-route -n agentgateway-system
```

## Try Again — Immediately

```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" \
  -X POST http://localhost:8080/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Are you there?"}]}' \
  --max-time 5
```

The request fails. No route exists. **Every agent behind this gateway is now blocked.**
Your application is still running. You changed nothing in your code.

## Inspect What's Left

The gateway is still up. The secret still exists. The backend is still configured:

```bash
kubectl get gateway,agentgatewaybackend -n agentgateway-system
```

You removed only the **route** — the explicit permission for traffic to flow.
Everything else is intact and ready to restore.

## Restore Access

Apply the saved route manifest from the previous challenge:

```bash
kubectl apply -f /root/openai-route.yaml
```

Wait a moment, then test again:

```bash
curl -s http://localhost:8080/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Back online?"}]}' \
  | jq '.choices[0].message.content'
```

Access restored. Zero downtime for anything else. 🎉

## Create Convenience Aliases

Add these to your toolkit for fast kill/restore from anywhere:

```bash
cat >> ~/.bashrc << 'EOF'

# AgentGateway Kill Switch
alias llm-kill="kubectl delete httproute openai-route -n agentgateway-system --ignore-not-found"
alias llm-restore="kubectl apply -f /root/openai-route.yaml"
alias llm-status="kubectl get httproute,gateway -n agentgateway-system"
EOF

source ~/.bashrc
```

Now test:

```bash
llm-kill     # Cut off LLM access
llm-restore  # Restore LLM access
llm-status   # Check current state
```

---

> 💡 **Why this matters in production:**
> - Runaway agent loop burning budget? `llm-kill`
> - Suspected prompt injection? `llm-kill` while you investigate
> - Rolling deployment where you want zero LLM traffic temporarily? `llm-kill`
> - Compliance requirement to disable AI features? `llm-kill`
>
> This is governance. Not hope.
