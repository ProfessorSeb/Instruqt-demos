---
slug: ambient-enrollment-automatic-mtls
type: challenge
title: Ambient Enrollment — Automatic mTLS
teaser: Enroll namespaces in the ambient mesh and get automatic mTLS with zero app changes.
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

# Ambient Enrollment — Automatic mTLS

## The Magic Label

In ambient mode, enrolling workloads in the mesh is a single label. No restarts, no sidecars injected, no YAML surgery. The ztunnel on each node automatically starts handling mTLS for enrolled namespaces.

## Your Tasks

### 1. Enroll Both Namespaces

```bash
kubectl label namespace finance istio.io/dataplane-mode=ambient
kubectl label namespace accounts istio.io/dataplane-mode=ambient
```

That's it. No pod restarts needed.

### 2. Verify Connectivity Still Works

```bash
kubectl exec curl -n finance -- curl -sS accounts.accounts:8080
```

You should still see `accounts-v1 responding`. The mesh is completely transparent to your applications.

### 3. Verify mTLS Is Active

Check the ztunnel logs for HBONE CONNECT activity — this shows ztunnel is handling the connection with mTLS:

```bash
kubectl logs -n istio-system -l app=ztunnel --tail=50 | grep -i "inbound" | tail -5
```

You should see log entries showing ztunnel proxying traffic between the finance and accounts workloads.

### 4. Confirm No Sidecars Were Injected

```bash
kubectl get pods -n accounts -o jsonpath='{range .items[*]}{.metadata.name}: {range .spec.containers[*]}{.name} {end}{"\n"}{end}'
kubectl get pods -n finance -o jsonpath='{range .items[*]}{.metadata.name}: {range .spec.containers[*]}{.name} {end}{"\n"}{end}'
```

Still one container per pod. The mTLS is happening at the node level via ztunnel — your pods never knew anything changed.

### 5. Check Namespace Labels

```bash
kubectl get namespace finance accounts --show-labels
```

You should see `istio.io/dataplane-mode=ambient` on both.

> 🏢 **Enterprise Value Beat:** You just enabled mTLS encryption for all traffic between these namespaces — without touching a single application manifest, without restarting pods, without adding sidecars. That's the ambient promise: zero-trust networking as infrastructure, not application concern.

> 💡 **Key Insight:** With sidecar mode, enrolling means restarting every pod (to inject the proxy). With ambient, it's instant — ztunnel is already running on every node, just waiting for the label.
