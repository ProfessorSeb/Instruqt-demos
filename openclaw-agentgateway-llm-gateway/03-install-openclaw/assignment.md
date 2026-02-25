---
slug: install-openclaw
id: ""
type: challenge
title: Install & Configure OpenClaw
teaser: Install OpenClaw and configure it to use AgentGateway as its LLM provider.
notes:
- type: text
  contents: |-
    # 🤖 Install OpenClaw

    **OpenClaw** is a self-hosted AI assistant platform. It runs as a background gateway service and lets you talk to your AI assistant via messaging apps like Telegram or WhatsApp.

    By default, OpenClaw talks directly to OpenAI. In this challenge, you'll configure it to use your **AgentGateway** instead — making every AI interaction go through your gateway.

    ```
    You (Telegram/WhatsApp)
        ↓
    OpenClaw  →  AgentGateway  →  OpenAI
    ```
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

# Install & Configure OpenClaw

## Install OpenClaw

OpenClaw is distributed as an npm package:

```bash
npm install -g openclaw
```

Verify it installed:

```bash
openclaw --version
```

## Run the Onboarding Wizard

OpenClaw has an interactive onboarding wizard. Run it and follow the prompts:

```bash
openclaw onboard
```

When the wizard asks you to choose a model provider, select **Custom / OpenAI-compatible endpoint**.

Use these settings:

| Field | Value |
|-------|-------|
| Base URL | `http://localhost:8080/v1` |
| API Key | `agentgateway` (any value — the gateway handles auth) |
| Model | `gpt-4o-mini` |

> Skip the channel (Telegram/WhatsApp) setup for now — we'll connect a channel in the next challenge.

## Verify the Configuration

After the wizard completes, check your config:

```bash
cat ~/.openclaw/openclaw.json | jq '.agents.defaults.model'
```

You should see your custom provider configured.

## Test the Provider Directly

Before starting the gateway, test that OpenClaw can reach AgentGateway:

```bash
openclaw chat --message "Say hello in one sentence."
```

You should get a response — routed through AgentGateway → OpenAI. 🎉

## Start the OpenClaw Gateway

Start the gateway as a background service:

```bash
openclaw gateway start
openclaw gateway status
```

> Your AI assistant is now running and routing all LLM traffic through AgentGateway.
