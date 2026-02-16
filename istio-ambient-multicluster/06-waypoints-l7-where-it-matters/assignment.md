---
slug: waypoints-l7-where-it-matters
type: challenge
title: "Waypoints — L7 Where It Matters"
teaser: "Deploy L7 waypoint proxies for advanced traffic management."
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
---

# Waypoints — L7 Where It Matters

## The L4 vs L7 Tradeoff

So far, all traffic management has been handled by **ztunnel** at L4. This gives you:
- ✅ mTLS encryption
- ✅ Identity-based authorization (source namespace, service account)
- ✅ Near-zero overhead (ztunnel is incredibly lightweight)

But L4 can't see inside HTTP requests. For features like:
- Header-based routing
- Request-level rate limiting
- HTTP-aware authorization (paths, methods, headers)
- Rich L7 telemetry (request duration, status codes)

...you need **waypoint proxies**. A waypoint is an on-demand Envoy proxy that Istio deploys only for the namespaces or services that need L7 features. This is the key innovation of ambient: **L7 only where you need it**.

> 🏢 **Enterprise Value Beat:** Waypoints let you apply L7 policies surgically. Instead of paying the Envoy cost for every pod in the mesh, you deploy L7 processing only for the services that require it.

## Your Task

### Step 1: Deploy a Waypoint for the Accounts Namespace

Use `istioctl` to deploy a waypoint proxy and enroll the entire `accounts` namespace:

**Cluster 1:**

```bash
istioctl waypoint apply -n accounts --enroll-namespace --context=$CTX_CLUSTER1
```

**Cluster 2:**

```bash
istioctl waypoint apply -n accounts --enroll-namespace --context=$CTX_CLUSTER2
```

### Step 2: Verify the Waypoint is Running

```bash
kubectl --context=$CTX_CLUSTER1 get pods -n accounts -l gateway.istio.io/managed=istio.io-mesh-controller
kubectl --context=$CTX_CLUSTER2 get pods -n accounts -l gateway.istio.io/managed=istio.io-mesh-controller
```

You should see a `waypoint` pod running in each cluster's `accounts` namespace.

Also check the Gateway resource:

```bash
kubectl --context=$CTX_CLUSTER1 get gateway -n accounts
kubectl --context=$CTX_CLUSTER2 get gateway -n accounts
```

### Step 3: Verify Connectivity Still Works

The waypoint should be transparent — existing connectivity should still work:

```bash
kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
  curl -s --max-time 10 accounts.accounts.svc.cluster.local:8080
```

### Step 4: Test L7 Authorization (HTTP Method)

Now that we have a waypoint, we can create L7-aware policies. Let's add a policy that only allows GET requests:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx apply -n accounts -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-get-only
spec:
  targetRefs:
  - kind: Service
    group: ""
    name: accounts
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["finance"]
    to:
    - operation:
        methods: ["GET"]
EOF
done
```

Test a GET request (should work):

```bash
kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
  curl -s --max-time 10 accounts.accounts.svc.cluster.local:8080
```

Test a POST request (should be denied by the waypoint):

```bash
kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
  curl -s --max-time 10 -X POST accounts.accounts.svc.cluster.local:8080
```

### Step 5: List Waypoints

```bash
istioctl waypoint list --context=$CTX_CLUSTER1
istioctl waypoint list --context=$CTX_CLUSTER2
```

Click **Check** when waypoints are running in both clusters!
