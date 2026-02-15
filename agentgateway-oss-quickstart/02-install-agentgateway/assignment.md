---
slug: install-agentgateway
id: ctseayfuds2t
type: challenge
title: Install Agentgateway on Kubernetes
teaser: Set up a Kubernetes cluster and deploy Agentgateway OSS using Helm.
notes:
- type: text
  contents: "# \U0001F680 Install Agentgateway on Kubernetes\n\nAgentgateway is a
    Kubernetes-native AI gateway built on the **Gateway API** standard.\n\n**In this
    challenge, you'll:**\n\n- Verify your Kubernetes cluster is ready\n- Install Agentgateway
    CRDs and control plane via Helm\n- Understand the GatewayClass → Gateway → Route
    model\n"
tabs:
- id: 9debqhafljmq
  title: Terminal
  type: terminal
  hostname: server
- id: oxqn1rffggz5
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Install Agentgateway on Kubernetes

Agentgateway is a Kubernetes-native project built on the **Gateway API** standard. This means it uses the same patterns and CRDs that the broader Kubernetes ecosystem has adopted for ingress and traffic management — but extended for AI agent traffic.

## Why Kubernetes Gateway API?

The Gateway API is the successor to Ingress in Kubernetes. It provides:
- **Role-oriented design** — platform teams manage Gateways, app teams manage Routes
- **Expressive routing** — path, header, and method-based routing built in
- **Extensibility** — custom resources for domain-specific needs (like AI!)

Agentgateway extends Gateway API with AI-specific capabilities while staying compatible with the standard.

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

## Install Agentgateway OSS

Agentgateway installs in two steps: first the CRDs, then the control plane.

**Step 1: Install Agentgateway CRDs**

```bash
helm install agentgateway-crds oci://ghcr.io/kgateway-dev/charts/agentgateway-crds \
  --version v2.2.0 \
  --namespace agentgateway-system \
  --create-namespace \
  --wait
```

**Step 2: Install the Agentgateway control plane**

```bash
helm install agentgateway oci://ghcr.io/kgateway-dev/charts/agentgateway \
  --version v2.2.0 \
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

You should see an `agentgateway` GatewayClass — this tells Kubernetes that Agentgateway is available as a gateway implementation.

## What Just Happened?

You installed:
- **Agentgateway CRDs** — custom resource definitions for AI-specific configuration
- **Agentgateway control plane** — the controller that watches for Gateway/Route resources and configures the data plane
- **GatewayClass** — registers Agentgateway as an available gateway implementation

Next, we'll create our first AI Gateway and route traffic to an LLM provider.
