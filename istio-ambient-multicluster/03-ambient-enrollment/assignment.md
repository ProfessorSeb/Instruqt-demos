---
slug: ambient-enrollment
id: 3g4ktfq16awe
type: challenge
title: Ambient Enrollment — L4 Zero Trust Everywhere
teaser: Enroll namespaces into the ambient mesh and verify automatic mTLS.
tabs:
- id: b3bypb6nxjey
  title: Terminal
  type: terminal
  hostname: server
- id: lbeuluff3scp
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Ambient Enrollment — L4 Zero Trust Everywhere

## Why Ambient Mode Changes Everything

Traditional Istio uses sidecar proxies — a full Envoy instance injected into every pod. That works, but it means:
- Double the containers in every pod
- Significant memory and CPU overhead
- Complex upgrade workflows (restart every pod)

**Ambient mode** takes a different approach. Instead of sidecars, it uses **ztunnel** — a lightweight, per-node L4 proxy that's already running as a DaemonSet. When you enroll a namespace, ztunnel automatically:

1. **Encrypts all traffic** with mTLS (mutual TLS)
2. **Authenticates workload identity** via SPIFFE certificates
3. **Does this with zero application changes** — no restarts, no sidecars

All you need is a single label on the namespace.

> 🏢 **Enterprise Value Beat:** Ambient enrollment gives your security team what they want (zero-trust mTLS everywhere) without the operational burden that made platform teams push back on service mesh adoption.

## Your Task

### Step 1: Enroll Namespaces in Ambient Mode

Label both `finance` and `accounts` namespaces on both clusters:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx label namespace finance istio.io/dataplane-mode=ambient
  kubectl --context=$ctx label namespace accounts istio.io/dataplane-mode=ambient
done
```

Verify the labels:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  echo "=== $ctx ==="
  kubectl --context=$ctx get namespace finance accounts --show-labels
done
```

### Step 2: Test Local Connectivity

Now test that services within the same cluster can communicate. From the `curl` pod in `finance`, reach the `accounts` service:

**Cluster 1:**

```bash
kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
  curl -s accounts.accounts.svc.cluster.local:8080
```

You should see: `accounts-cluster-1`

**Cluster 2:**

```bash
kubectl --context=$CTX_CLUSTER2 exec -n finance deploy/curl -- \
  curl -s accounts.accounts.svc.cluster.local:8080
```

You should see: `accounts-cluster-2`

### Step 3: Verify mTLS with ztunnel Logs

The magic of ambient mode is that mTLS happened automatically. Let's verify by checking ztunnel logs:

```bash
kubectl --context=$CTX_CLUSTER1 -n istio-system logs -l app=ztunnel --tail=20 | grep -i "tls" | tail -5
```

You should see log entries showing TLS connections being established between workloads.

### Step 4: Check Workload Identity

Verify that Istio assigned SPIFFE identities to your workloads:

```bash
istioctl --context=$CTX_CLUSTER1 proxy-config workload -n istio-system deploy/ztunnel 2>/dev/null | grep -E "accounts|finance" || \
  echo "Check ztunnel is proxying traffic for your workloads"
```

Click **Check** when all namespaces are enrolled and connectivity works!
