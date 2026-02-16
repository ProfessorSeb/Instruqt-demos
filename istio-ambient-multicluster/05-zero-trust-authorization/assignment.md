---
slug: zero-trust-authorization
type: challenge
title: "Zero Trust Authorization Policies"
teaser: "Enforce deny-by-default policies and allow only authorized traffic."
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
---

# Zero Trust Authorization Policies

## Why Deny-by-Default?

mTLS encryption is table stakes. True zero trust means **authorization** — explicitly allowing only the traffic that should flow, and denying everything else.

In ambient mode, authorization policies at L4 are enforced by **ztunnel** directly. No waypoint proxy needed. This means you get identity-based access control with virtually zero overhead.

The pattern:
1. **Deny all** — nothing gets through by default
2. **Allow specific** — explicitly permit only the flows you want

> 🏢 **Enterprise Value Beat:** Deny-by-default authorization policies give your security and compliance teams auditable proof that only authorized services can communicate — across cluster boundaries.

## Your Task

### Step 1: Apply Deny-by-Default Policy

Apply an `AuthorizationPolicy` that denies all traffic to the `accounts` namespace on **both clusters**:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx apply -n accounts -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
spec:
  {}
EOF
done
```

### Step 2: Verify Traffic is Blocked

Try calling the accounts service — it should fail:

```bash
kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
  curl -s --max-time 5 accounts.accounts.svc.cluster.local:8080
```

You should get a connection reset or timeout. **This is correct!** The deny-all policy is working.

### Step 3: Allow Traffic from Finance

Now create a policy that allows traffic specifically from the `finance` namespace:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx apply -n accounts -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-from-finance
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["finance"]
EOF
done
```

### Step 4: Verify Finance Can Access Accounts

```bash
kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
  curl -s --max-time 10 accounts.accounts.svc.cluster.local:8080
```

You should see `accounts-cluster-1` or `accounts-cluster-2` again. ✅

### Step 5: Prove Unauthorized Access is Blocked

Create a `test` namespace with a curl pod and verify it **cannot** access accounts:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx create namespace test
  kubectl --context=$ctx apply -n test -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl-test
  template:
    metadata:
      labels:
        app: curl-test
    spec:
      containers:
      - name: curl
        image: curlimages/curl:8.5.0
        command: ["sleep", "infinity"]
EOF
done
```

Wait for pods to be ready, then test:

```bash
kubectl --context=$CTX_CLUSTER1 -n test wait --for=condition=Ready pod -l app=curl-test --timeout=60s
kubectl --context=$CTX_CLUSTER1 exec -n test deploy/curl-test -- \
  curl -s --max-time 5 accounts.accounts.svc.cluster.local:8080
```

This should fail — the `test` namespace is not authorized. 🔒

### Step 6: Run istioctl analyze

Validate your configuration on both clusters:

```bash
istioctl analyze --context=$CTX_CLUSTER1 -n accounts
istioctl analyze --context=$CTX_CLUSTER2 -n accounts
```

Click **Check** when your authorization policies are in place!
