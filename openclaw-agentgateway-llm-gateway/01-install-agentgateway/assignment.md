---
slug: install-agentgateway
id: ""
type: challenge
title: Install AgentGateway on Kubernetes
teaser: Deploy AgentGateway OSS onto a kind cluster and verify it's running.
notes:
- type: text
  contents: |-
    # 🚀 Install AgentGateway

    **AgentGateway** is a Kubernetes-native gateway built for AI agent traffic.

    Instead of your AI assistant calling OpenAI directly, AgentGateway sits in the middle — giving you:

    - 🔑 **Centralized credential management** — one secret, not scattered API keys
    - 📊 **Observability** — see every LLM call, token count, and latency
    - 🔀 **Routing** — load balance across models, fail over, rate limit

    In this challenge you'll install it on a local kind cluster.
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

# Install AgentGateway on Kubernetes

A **kind** cluster is already running with port 80 mapped to `localhost:8080` on
the host — that's the port AgentGateway will listen on.

## Verify the Cluster

```bash
kubectl get nodes
```

You should see one node in `Ready` state.

Gateway API CRDs were pre-installed during setup. Verify:

```bash
kubectl get crds | grep gateway
```

## Install AgentGateway via Helm

Install the AgentGateway control plane into its own namespace:

```bash
helm upgrade --install agentgateway \
  oci://ghcr.io/agentgateway-dev/helm-charts/agentgateway \
  --namespace agentgateway-system \
  --create-namespace \
  --wait --timeout 180s
```

## Verify the Installation

Check that the AgentGateway pod is running:

```bash
kubectl get pods -n agentgateway-system
```

Check the GatewayClass was registered:

```bash
kubectl get gatewayclass
```

You should see `agentgateway` with `ACCEPTED: True`.

> 🎉 AgentGateway is running! In the next challenge you'll configure it to proxy LLM traffic to OpenAI.
