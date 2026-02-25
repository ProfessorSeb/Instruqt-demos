---
slug: install-agentgateway
id: lhqkmnrhx0yg
type: challenge
title: Install AgentGateway on Kubernetes
teaser: Deploy AgentGateway OSS — a Kubernetes-native gateway built specifically for
  AI agent traffic.
notes:
- type: text
  contents: "# \U0001F680 AgentGateway: Built for AI Agent Traffic\n\nTraditional
    API gateways were built for HTTP microservices. They handle request routing, auth,
    and rate limiting — but they know nothing about:\n\n- **Token counting** — LLM
    responses are billed by tokens, not requests\n- **Prompt inspection** — you need
    to see what's going into the model\n- **MCP tool routing** — a completely different
    protocol from HTTP REST\n- **Model failover** — if GPT-4 is down, fall back to
    GPT-3.5 automatically\n- **Streaming** — LLM responses stream back as chunks,
    not single responses\n\nAgentGateway is built for exactly this. It extends the
    **Kubernetes Gateway API** —\nthe same standard used by Istio, Envoy, and NGINX
    — with AI-specific capabilities.\n\n**Architecture:**\n\n```\n┌─────────────────────────────────────┐\n│
    \        AgentGateway OSS            │\n│                                     │\n│
    \ GatewayClass  →  Gateway           │\n│       ↓               ↓             │\n│
    \ AgentgatewayBackend  AIRoute       │\n│  (your LLM provider)  (routing)     │\n└─────────────────────────────────────┘\n```"
tabs:
- id: 0cuxkbepgsum
  title: Terminal
  type: terminal
  hostname: server
- id: dsjamxnm8z67
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 900
enhanced_loading: null
---

# Install AgentGateway on Kubernetes

A k3d cluster is running with **port 80 mapped to `localhost:8080`** on this host.
That's the port your gateway will listen on — and where you'll point OpenClaw later.

## Verify the Cluster

```bash
kubectl get nodes
```

Gateway API CRDs were pre-installed during setup. These are the standard Kubernetes
types that AgentGateway builds on:

```bash
kubectl get crds | grep gateway.networking.k8s.io
```

You should see `gateways`, `httproutes`, `gatewayclasses`, and more.

## Why Kubernetes Gateway API?

AgentGateway is built on the **Kubernetes Gateway API** — not a proprietary config format.
This means:

- **Role separation** — platform teams manage Gateways, app teams manage Routes
- **Portability** — same patterns as Istio, Envoy, NGINX Gateway Fabric
- **Extensibility** — custom resources for AI-specific needs layered on top

## Install AgentGateway via Helm

AgentGateway uses two Helm charts — install CRDs first, then the control plane:

```bash
# Step 1: Install CRDs
helm install agentgateway-crds \
  oci://ghcr.io/kgateway-dev/charts/agentgateway-crds \
  --version v2.2.0 \
  --namespace agentgateway-system \
  --create-namespace \
  --wait

# Step 2: Install the control plane
helm install agentgateway \
  oci://ghcr.io/kgateway-dev/charts/agentgateway \
  --version v2.2.0 \
  --namespace agentgateway-system \
  --wait --timeout 120s
```

## Verify the Installation

```bash
kubectl get pods -n agentgateway-system
```

Check the `GatewayClass` was registered — this is how Kubernetes knows
AgentGateway is available as a gateway implementation:

```bash
kubectl get gatewayclass
```

You should see `agentgateway` with `ACCEPTED: True`.

## What Just Happened?

AgentGateway deployed a **controller** — a process that watches for Gateway API
resources you create and programs the gateway accordingly.

It's ready, but it's not doing anything yet. No routes. No providers.
No traffic. In the next challenge you'll configure the first LLM provider.

> 💡 **Key insight:** The gateway is a control plane. It does nothing until you
> tell it where to route traffic. That's intentional — explicit configuration
> means explicit control.
