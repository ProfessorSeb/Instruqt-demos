---
slug: install-openclaw
id: ""
type: challenge
title: Wire OpenClaw Through AgentGateway
teaser: Install OpenClaw and configure it to use AgentGateway as its LLM provider — completing the full stack.
notes:
- type: text
  contents: |-
    # 🤖 Completing the Stack

    You've built the gateway. Now let's put an AI assistant behind it.

    **OpenClaw** is a self-hosted AI assistant platform. It runs as a background
    service and lets you talk to your AI via Telegram, WhatsApp, Signal, and more.

    By default, OpenClaw would call OpenAI directly. You're going to change that.

    Instead of:
    ```
    OpenClaw  ──▶  OpenAI  (direct, uncontrolled)
    ```

    You'll configure:
    ```
    OpenClaw  ──▶  AgentGateway  ──▶  OpenAI
    ```

    Every message you send to your AI assistant will flow through the gateway.
    You'll have full visibility. You'll have the kill switch. You'll have control.

    **This is what governed AI looks like.**
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

# Wire OpenClaw Through AgentGateway

## Install OpenClaw

```bash
npm install -g openclaw
openclaw --version
```

## Run the Onboarding Wizard

```bash
openclaw onboard
```

When prompted for a model provider, select **Custom / OpenAI-compatible endpoint**.

| Setting | Value |
|---|---|
| Base URL | `http://localhost:8080/v1` |
| API Key | `agentgateway` *(any value — the gateway handles auth)* |
| Model | `gpt-4o-mini` |

> Skip channel setup for now — you'll connect Telegram or WhatsApp in the next step.

## Verify the Configuration

```bash
cat ~/.openclaw/openclaw.json | jq '.models.providers'
```

You should see your custom provider pointing to `localhost:8080`.

## Test the Full Stack

Send a message through OpenClaw CLI:

```bash
openclaw chat --message "You are running through AgentGateway. Tell me what that means."
```

**Watch traffic in AgentGateway at the same time** (open a second terminal tab):

```bash
kubectl logs -n agentgateway-system \
  -l app.kubernetes.io/name=agentgateway \
  --follow
```

Send another message and watch it appear in the gateway logs. 🎉

## Start the OpenClaw Gateway

```bash
openclaw gateway start
openclaw gateway status
```

## Test the Kill Switch Against OpenClaw

Now use the aliases you created in Challenge 4:

```bash
# Cut off OpenClaw's LLM access
llm-kill

# Try to chat — it should fail
openclaw chat --message "Hello?" || echo "Blocked by AgentGateway — as expected."

# Restore access
llm-restore

# Chat again — works
openclaw chat --message "Back online!"
```

> 💡 **This is the full picture:** Your AI assistant has no credentials.
> It has no direct path to any LLM. Everything goes through your gateway.
> You control access. You can see everything. You can stop anything.
