---
slug: explore-enterprise-environment
id: explore-enterprise-environment
type: challenge
title: Explore the Enterprise Environment
teaser: Orient yourself with the Enterprise AgentGateway installation and monitoring stack.
notes:
- type: text
  contents: |-
    # Welcome to Enterprise AgentGateway + Okta! 🔐

    In this workshop, you'll build an **identity-aware AI gateway** using Enterprise AgentGateway and Okta.

    Your environment comes pre-installed with:
    - **Kubernetes cluster** (k3d)
    - **Enterprise AgentGateway controller** (v2.1.1)
    - **Monitoring stack** (Grafana + Tempo + Prometheus)
    - **OpenAI API key** (as a Kubernetes secret)

    Your job: configure the gateway, route traffic, and lock it down with Okta identity.

    Let's start by exploring what's already running!
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 600
---

# Explore the Enterprise Environment

Before building anything, let's understand what's already deployed.

## Check the Enterprise AgentGateway Controller

The controller is installed in the `enterprise-agentgateway` namespace:

```bash
kubectl get pods -n enterprise-agentgateway
```

You should see the controller pod running.

## Discover Enterprise CRDs

Enterprise AgentGateway extends the OSS version with additional CRDs:

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

## Record Your Findings

Create a file to confirm you've explored the environment:

```bash
cat > /root/environment-verified.txt << 'EOF'
Enterprise AgentGateway Environment Verified
- Controller: running in enterprise-agentgateway namespace
- GatewayClass: enterprise-agentgateway
- CRDs: AgentgatewayBackend, EnterpriseAgentgatewayParameters, EnterpriseAgentgatewayPolicy
- Monitoring: Grafana + Tempo + Prometheus in monitoring namespace
- OpenAI secret: configured
EOF
```

Click **Check** when you're done exploring!
