---
slug: chat-through-gateway
id: ""
type: challenge
title: Chat Through the Gateway
teaser: Send your first message through OpenClaw and watch it flow through AgentGateway to OpenAI.
notes:
- type: text
  contents: |-
    # 💬 Chat Through the Gateway

    Everything is connected:

    ```
    OpenClaw  →  AgentGateway  →  OpenAI
    ```

    In this challenge you'll:

    - Send a message through the full stack
    - Inspect AgentGateway logs to see your request being proxied
    - Connect OpenClaw to **Telegram** so you can chat from your phone

    This is the "aha moment" — your AI assistant with full gateway visibility.
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 900
---

# Chat Through the Gateway

## Send a Message via OpenClaw CLI

```bash
openclaw chat --message "What is AgentGateway and why does it matter for AI agents?"
```

You should get a response from OpenAI, routed through AgentGateway.

## Watch the Traffic in AgentGateway Logs

In a second terminal (or use the **+** button to open a new tab), watch the AgentGateway logs:

```bash
kubectl logs -n agentgateway-system \
  -l app.kubernetes.io/name=agentgateway \
  --follow
```

Send another message and watch the request flow through in real time.

## Connect OpenClaw to Telegram

To chat from your phone, connect OpenClaw to Telegram:

**1. Create a Telegram Bot**

Open Telegram and message `@BotFather`:
```
/newbot
```
Follow the prompts and copy your bot token.

**2. Configure OpenClaw**

Run the configuration wizard and select Telegram:

```bash
openclaw configure --section channels
```

Or add it directly to your config:

```bash
openclaw configure --set channels.telegram.enabled=true
openclaw configure --set channels.telegram.botToken=YOUR_BOT_TOKEN
openclaw configure --set channels.telegram.dmPolicy=pairing
```

**3. Restart and connect**

```bash
openclaw gateway restart
```

Open your Telegram bot and send it a message — you'll be prompted to pair.
Approve the pairing:

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

**4. Test it**

Send a message to your bot on Telegram. Every LLM call routes through AgentGateway. 🎉

## Inspect Your Gateway's Observability

See what AgentGateway knows about your LLM traffic:

```bash
# Check the backend status
kubectl get agentgatewaybackend -n agentgateway-system -o wide

# Describe the gateway
kubectl describe gateway llm-gateway -n agentgateway-system
```

> You now have a fully observable, gateway-controlled AI assistant. All LLM calls go through one controlled point — no scattered API keys, no blind traffic.
