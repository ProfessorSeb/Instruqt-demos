---
slug: explore-the-ambient-environment
type: challenge
title: Explore the Ambient Environment
teaser: Verify your Istio ambient mesh is running and understand its architecture.
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Explore the Ambient Environment

## What Is Ambient Mesh?

Traditional Istio injects a sidecar proxy into every pod. That works — but it means doubling your pod count, dealing with sidecar lifecycle, and burning CPU/memory on proxies that most workloads barely use.

**Istio ambient mesh** takes a different approach. It splits the data plane into two layers:

- **ztunnel** — a per-node L4 proxy (DaemonSet). Handles mTLS, L4 authorization, and telemetry for *every* pod on the node. Zero config, zero sidecars.
- **Waypoint proxies** — opt-in L7 proxies you deploy only where you need HTTP routing, header-based auth, or traffic shaping.

The result? Zero-trust networking by default, with L7 capabilities only where they add value.

> 🏢 **Enterprise Value Beat:** Ambient mesh gives you the security posture of a full service mesh at a fraction of the resource cost. No more "sidecar tax" on every microservice.

## What's Already Set Up

Your environment has been pre-configured with:

- **A k3d cluster** (`my-k8s-cluster`) with Istio ambient mode installed
- **istiod** — the control plane
- **ztunnel** — L4 per-node proxy (DaemonSet)
- **istio-cni** — CNI plugin that redirects traffic through ztunnel
- **istioctl** — CLI for mesh management

## Your Tasks

### 1. Check Istio Components

List the pods in the `istio-system` namespace:

```bash
kubectl get pods -n istio-system
```

You should see:
- `istiod-*` — the control plane
- `ztunnel-*` — one per node (DaemonSet)
- `istio-cni-node-*` — one per node (DaemonSet)

### 2. Verify the DaemonSets

```bash
kubectl get daemonsets -n istio-system
```

Both `ztunnel` and `istio-cni-node` should show the same DESIRED and READY count (matching your node count).

### 3. Check Istio Version

```bash
istioctl version
```

### 4. Verify No Sidecars

Look at a system namespace — notice there are NO istio-proxy sidecar containers:

```bash
kubectl get pods -n kube-system -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
```

No `istio-proxy` containers anywhere. That's ambient at work.

### 5. Mark Complete

Once you've explored the environment, create the verification file:

```bash
echo "ambient environment verified" > /root/environment-verified.txt
```

> 💡 **Key Insight:** In ambient mode, mTLS happens at the *node level* via ztunnel — not inside your pods. Your applications don't need to change at all.
