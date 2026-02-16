---
slug: zero-trust-authorization-policies
id: c3lcp1ud6dya
type: challenge
title: Zero Trust Authorization Policies
teaser: Enforce deny-by-default policies and allow only authorized service-to-service
  traffic.
tabs:
- id: x3qzqrvxry9h
  title: Terminal
  type: terminal
  hostname: server
- id: 4g9ql5yyqzbw
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Zero Trust Authorization Policies

## From "Allow All" to "Deny by Default"

Right now, any pod in the mesh can talk to the accounts service. In a zero-trust model, we flip that: deny everything, then explicitly allow only what's needed.

Istio's `AuthorizationPolicy` works at L4 in ambient mode (via ztunnel) — meaning these policies are enforced without any waypoint proxies.

## Your Tasks

### 1. Apply Deny-by-Default

Create an AuthorizationPolicy with no rules — this denies all traffic to the accounts namespace:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: accounts
spec:
  {}
EOF
```

### 2. Verify Traffic Is Blocked

```bash
kubectl exec curl -n finance -- curl -sS --max-time 5 accounts.accounts:8080 || echo "BLOCKED as expected"
```

You should see a connection refused or timeout. The deny-all policy is working.

### 3. Allow Traffic from Finance

Now create a policy that allows only workloads in the `finance` namespace:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-finance
  namespace: accounts
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["finance"]
EOF
```

### 4. Verify Finance Can Connect Again

```bash
kubectl exec curl -n finance -- curl -sS accounts.accounts:8080
```

You should see `accounts-v1 responding` again.

### 5. Test from a Rogue Namespace

Create an unauthorized namespace and try to access accounts:

```bash
kubectl create namespace rogue
kubectl label namespace rogue istio.io/dataplane-mode=ambient
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rogue-curl
  namespace: rogue
  labels:
    app: rogue-curl
spec:
  containers:
  - name: curl
    image: curlimages/curl:8.5.0
    command: ["sleep", "infinity"]
EOF

kubectl wait --for=condition=Ready pod/rogue-curl -n rogue --timeout=60s
kubectl exec rogue-curl -n rogue -- curl -sS --max-time 5 accounts.accounts:8080 || echo "BLOCKED — rogue namespace denied"
```

The rogue pod should be blocked. Only finance is allowed.

### 6. Run Istio Analyze

```bash
istioctl analyze --namespace accounts
```

This validates your configuration for common issues.

> 🏢 **Enterprise Value Beat:** You just implemented zero-trust network segmentation without any network policies, firewalls, or CNI plugins. The authorization is based on cryptographic identity (mTLS), not IP addresses — which means it works even if pods move nodes or IPs change.

> 💡 **Key Insight:** The deny-all + explicit-allow pattern is the gold standard for zero-trust. With ambient, these L4 policies are enforced by ztunnel — no waypoint needed.
