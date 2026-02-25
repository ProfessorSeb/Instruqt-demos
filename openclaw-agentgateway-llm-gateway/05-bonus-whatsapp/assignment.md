---
slug: bonus-whatsapp
id: ""
type: challenge
title: "BONUS: Connect OpenClaw to WhatsApp"
teaser: Link your WhatsApp account so you can chat with your AI assistant from any device.
notes:
- type: text
  contents: |-
    # 📱 BONUS: WhatsApp Channel

    You've built the full stack:

    ```
    OpenClaw  →  AgentGateway  →  OpenAI
    ```

    Now let's add WhatsApp as a channel so you can chat with your AI assistant from your phone — through the gateway.

    ```
    WhatsApp (your phone)
        ↓
    OpenClaw  →  AgentGateway  →  OpenAI
    ```

    **You'll need:** A WhatsApp account and your phone nearby to scan a QR code.
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

> **You'll need your phone for this challenge** — you'll scan a QR code to link WhatsApp.

## Configure WhatsApp in OpenClaw

Add the WhatsApp channel configuration:

```bash
openclaw configure --set channels.whatsapp.dmPolicy=pairing
```

This sets WhatsApp to "pairing" mode — the first person to message the bot
gets prompted to pair (you'll approve it from the CLI).

## Start the WhatsApp Login Flow

This command shows a QR code in your terminal:

```bash
openclaw channels login --channel whatsapp
```

**On your phone:**
1. Open WhatsApp → **Settings → Linked Devices**
2. Tap **Link a Device**
3. Scan the QR code in the terminal

Once linked, you'll see: `✓ WhatsApp linked successfully`

## Restart the Gateway

```bash
openclaw gateway restart
openclaw channels status
```

You should see WhatsApp listed as `linked`.

## Send Your First WhatsApp Message

Open WhatsApp on your phone and message the account you just linked.

Since you're in `pairing` mode, OpenClaw will ask you to approve the connection:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

Once approved, send a message and get a response — routed through AgentGateway → OpenAI. 🎉

## Verify the Full Stack

Check that WhatsApp traffic also flows through your gateway:

```bash
# Watch AgentGateway logs while you send a WhatsApp message
kubectl logs -n agentgateway-system \
  -l app.kubernetes.io/name=agentgateway \
  --follow
```

Every message you send from WhatsApp should appear as a request in the AgentGateway logs.

---

**🎓 Congratulations!**

You've built a complete AI assistant stack:
- ✅ AgentGateway controlling all LLM traffic
- ✅ OpenClaw as your self-hosted AI assistant
- ✅ WhatsApp (and/or Telegram) as your chat interface
- ✅ Full visibility into every LLM call through the gateway

Your API key lives in Kubernetes — not scattered across devices.
