---
slug: install-agentgateway
id: ctseayfuds2t
type: challenge
title: Install AgentGateway on Kubernetes
teaser: Set up a Kubernetes cluster and deploy AgentGateway OSS using Helm.
difficulty: ""
enhanced_loading: null
---

# Install AgentGateway on Kubernetes

AgentGateway is a Kubernetes-native project built on the **Gateway API** standard. This means it uses the same patterns and CRDs that the broader Kubernetes ecosystem has adopted for ingress and traffic management — but extended for AI agent traffic.

## Why Kubernetes Gateway API?

The Gateway API is the successor to Ingress in Kubernetes. It provides:
- **Role-oriented design** — platform teams manage Gateways, app teams manage Routes
- **Expressive routing** — path, header, and method-based routing built in
- **Extensibility** — custom resources for domain-specific needs (like AI!)

AgentGateway extends Gateway API with AI-specific capabilities while staying compatible with the standard.

## Verify Your Cluster

A k3s cluster was created during track setup. Let's verify it's ready:

```bash
kubectl cluster-info
```

```bash
kubectl get nodes
```

You should see a single node in `Ready` state.

## Install Gateway API CRDs

The Gateway API CRDs were installed during setup. Verify they're present:

```bash
kubectl get crds | grep gateway
```

You should see CRDs like `gateways.gateway.networking.k8s.io` and `httproutes.gateway.networking.k8s.io`.

## Install AgentGateway OSS

AgentGateway installs in two steps: first the CRDs, then the control plane.

**Step 1: Install AgentGateway CRDs**

```bash
helm install agentgateway-crds agentgateway/agentgateway-crds \
  --version 2.1.0 \
  --namespace agentgateway-system \
  --create-namespace \
  --wait
```

**Step 2: Install the AgentGateway control plane**

```bash
helm install agentgateway agentgateway/agentgateway \
  --version 2.1.0 \
  --namespace agentgateway-system \
  --wait --timeout 120s
```

## Verify the Installation

Check that all pods are running:

```bash
kubectl get pods -n agentgateway-system
```

All pods should show `Running` status with all containers ready.

Verify the GatewayClass was created:

```bash
kubectl get gatewayclass
```

You should see an `agentgateway` GatewayClass — this tells Kubernetes that AgentGateway is available as a gateway implementation.

## What Just Happened?

You installed:
- **AgentGateway CRDs** — custom resource definitions for AI-specific configuration
- **AgentGateway control plane** — the controller that watches for Gateway/Route resources and configures the data plane
- **GatewayClass** — registers AgentGateway as an available gateway implementation

Next, we'll create our first AI Gateway and route traffic to an LLM provider.
