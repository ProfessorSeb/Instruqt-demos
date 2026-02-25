---
slug: bonus-whatsapp
id: ""
type: challenge
title: "BONUS: Connect OpenClaw to WhatsApp"
teaser: Link WhatsApp so you can chat with your gateway-controlled AI assistant from your phone.
notes:
- type: text
  contents: |-
    # 📱 BONUS: Chat from Your Phone

    You've built a fully governed AI assistant stack:

    ```
    OpenClaw  ──▶  AgentGateway  ──▶  OpenAI
    ```

    Now let's add WhatsApp as a front-end.

    ```
    WhatsApp (your phone)
         ↓
    OpenClaw  ──▶  AgentGateway  ──▶  OpenAI
    ```

    Every message you send from WhatsApp flows through your gateway.
    The kill switch still works. Credentials never leave Kubernetes.
    You'll need your phone to scan a QR code.
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Editor
  type: code
  hostname: server
  path: /root
difficulty: intermediate
timelimit: 900
---

# BONUS: Connect OpenClaw to WhatsApp

> **You'll need your phone for this challenge.**

## Configure the WhatsApp Channel

```bash
openclaw configure --set channels.whatsapp.dmPolicy=pairing
```

`pairing` mode means: anyone who messages the linked account is shown a pairing
code. You approve it from the CLI before they can interact with your AI assistant.
This is the security layer at the channel level.

## Link Your WhatsApp Account

```bash
openclaw channels login --channel whatsapp
```

A QR code appears in the terminal.

**On your phone:**
1. WhatsApp → **Settings → Linked Devices → Link a Device**
2. Scan the QR code

When linked, you'll see: `✓ WhatsApp linked successfully`

## Restart and Verify

```bash
openclaw gateway restart
openclaw channels status
```

WhatsApp should show as `linked`.

## Send Your First Message

Message the WhatsApp account you just linked from your phone.

OpenClaw will put the sender in a pairing queue — approve it:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

Send a message and get a response from your AI assistant — routed through
AgentGateway — from your phone.

## Verify the Gateway is Still in Control

While chatting on WhatsApp, watch the gateway logs:

```bash
kubectl logs -n agentgateway-system \
  -l app.kubernetes.io/name=agentgateway \
  --follow
```

Every WhatsApp message that triggers an LLM call shows up here.

Now test the kill switch from WhatsApp:

```bash
llm-kill
```

Send a message from your phone. The AI can't respond — AgentGateway is blocking it.

```bash
llm-restore
```

Send again — it works.

---

## 🎓 What You've Built

Congratulations. You now have:

| Layer | What | Governed by |
|---|---|---|
| Channel | WhatsApp / Telegram | OpenClaw pairing |
| AI Assistant | OpenClaw | Self-hosted, your infrastructure |
| LLM Access | AgentGateway route | Kill switch in 1 command |
| Credentials | Kubernetes Secret | Never in your app |
| MCP Tools | AgentGateway MCPBackend | Same kill switch pattern |
| Observability | Gateway logs | Every call visible |

**This is what a governed, production-ready AI agent stack looks like.**
