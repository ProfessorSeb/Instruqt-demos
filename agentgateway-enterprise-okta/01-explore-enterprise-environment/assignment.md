---
slug: explore-enterprise-environment
id: hgelffzgay6j
type: challenge
title: Explore the Enterprise Environment
teaser: Orient yourself with the Enterprise Agentgateway installation and monitoring
  stack.
notes:
- type: text
  contents: "# Welcome to Enterprise Agentgateway + Okta! \U0001F510\n\nIn this workshop,
    you'll build an **identity-aware AI gateway** using Enterprise Agentgateway and
    Okta.\n\nYour environment comes pre-installed with:\n- **Kubernetes cluster**
    (k3d)\n- **Enterprise Agentgateway controller** (v2.1.1)\n- **Monitoring stack**
    (Grafana + Tempo + Prometheus)\n- **OpenAI API key** (as a Kubernetes secret)\n\nYour
    job: configure the gateway, route traffic, and lock it down with Okta identity.\n\nLet's
    start by exploring what's already running!"
tabs:
- id: 9rzwyjirqm2y
  title: Terminal
  type: terminal
  hostname: server
- id: dqp7tvddmgdd
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 600
enhanced_loading: null
---

# Explore the Enterprise Environment

Before building anything, let's understand what's already deployed.

## Check the Enterprise Agentgateway Controller

The controller is installed in the `enterprise-agentgateway` namespace:

```bash
kubectl get pods -n enterprise-agentgateway
```

You should see the controller pod running.

## Discover Enterprise CRDs

Enterprise Agentgateway extends the OSS version with additional CRDs:

```bash
kubectl api-resources | grep agentgateway
```

Notice both **OSS** CRDs (`agentgateway.dev`) and **Enterprise** CRDs (`enterpriseagentgateway.solo.io`):
- `AgentgatewayBackend` — defines LLM and MCP backends
- `EnterpriseAgentgatewayParameters` — Enterprise-specific gateway configuration
- `EnterpriseAgentgatewayPolicy` — JWT auth, RBAC, and more

## Check the GatewayClass

```bash
kubectl get gatewayclass
```

You should see `enterprise-agentgateway` — this is the GatewayClass your Gateway will use.

## Verify the Monitoring Stack

```bash
kubectl get pods -n monitoring
```

You should see Grafana, Prometheus, and Tempo pods running. Grafana is accessible on port 3000 (credentials: `admin` / `solo-demo`).

## Check the OpenAI Secret

```bash
kubectl get secret openai-secret -n enterprise-agentgateway
```

This secret contains the OpenAI API key that the gateway will use to authenticate with OpenAI.

## Summary

Your environment is pre-configured with:
- **Controller** — running in `enterprise-agentgateway` namespace
- **GatewayClass** — `enterprise-agentgateway`
- **CRDs** — `AgentgatewayBackend`, `EnterpriseAgentgatewayParameters`, `EnterpriseAgentgatewayPolicy`
- **Monitoring** — Grafana + Tempo + Prometheus in `monitoring` namespace
- **OpenAI secret** — configured and ready

Click **Check** when you're done exploring!
