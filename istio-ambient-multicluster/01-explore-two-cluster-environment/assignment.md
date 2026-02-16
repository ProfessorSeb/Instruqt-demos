---
slug: explore-two-cluster-environment
id: 4lmnj9cvmcin
type: challenge
title: Explore the Two-Cluster Environment
teaser: Verify your two-cluster Istio ambient mesh is up and running.
tabs:
- id: bgsczh2yfjcn
  title: Terminal
  type: terminal
  hostname: server
- id: iauoejysgowq
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Explore the Two-Cluster Environment

## Why Multi-Cluster?

In production, organizations run multiple Kubernetes clusters — for regional availability, blast-radius isolation, or regulatory compliance. The challenge? Services across those clusters need to communicate securely and transparently, as if they lived in a single mesh.

**Istio ambient mesh** solves this without sidecars. Instead of injecting a proxy into every pod, ambient uses **ztunnel** (a per-node L4 proxy) for automatic mTLS and **waypoint proxies** (on-demand L7) only where you need them. This dramatically reduces resource overhead — especially at multi-cluster scale.

> 🏢 **Enterprise Value Beat:** Multi-cluster ambient mesh gives you zero-trust networking across regions with a fraction of the resource cost of sidecar-based approaches.

## What's Already Set Up

Your environment has been pre-configured with:

- **Two k3d clusters**: `cluster-1` and `cluster-2`
- **Istio ambient mode** installed on both clusters
- **Shared certificate authority** — both clusters trust each other
- **East-west gateways** — cross-cluster traffic tunnels
- **Remote secrets** — cross-cluster service discovery enabled

## Your Task

Explore the environment and verify everything is healthy.

### Step 1: Check Cluster Contexts

Two environment variables hold your cluster contexts:

```bash
echo "Cluster 1: $CTX_CLUSTER1"
echo "Cluster 2: $CTX_CLUSTER2"
```

Verify both clusters are reachable:

```bash
kubectl --context=$CTX_CLUSTER1 get nodes
kubectl --context=$CTX_CLUSTER2 get nodes
```

### Step 2: Verify Istio Components

Check that all Istio components are running on **both** clusters:

```bash
kubectl --context=$CTX_CLUSTER1 get pods -n istio-system
kubectl --context=$CTX_CLUSTER2 get pods -n istio-system
```

You should see:
- **istiod** — the control plane
- **ztunnel** — the per-node L4 proxy (DaemonSet)
- **istio-cni** — the CNI plugin that redirects traffic
- **istio-eastwestgateway** — the cross-cluster tunnel

### Step 3: Check the Ambient GatewayClass

Istio ambient registers a `GatewayClass` for waypoint proxies:

```bash
kubectl --context=$CTX_CLUSTER1 get gatewayclass
kubectl --context=$CTX_CLUSTER2 get gatewayclass
```

### Step 4: Verify Cross-Cluster Secrets

Confirm the remote secrets are in place (this is how clusters discover each other's services):

```bash
kubectl --context=$CTX_CLUSTER1 get secrets -n istio-system -l istio/multiCluster=true
kubectl --context=$CTX_CLUSTER2 get secrets -n istio-system -l istio/multiCluster=true
```

### Step 5: Mark Environment as Verified

Once you've confirmed everything looks healthy, create the verification file:

```bash
echo "Two-cluster Istio ambient mesh verified on $(date)" > /root/environment-verified.txt
```

Click **Check** when you're done!
