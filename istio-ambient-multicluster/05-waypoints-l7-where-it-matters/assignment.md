---
slug: waypoints-l7-where-it-matters
id: gsngygxnqboc
type: challenge
title: Waypoints — L7 Where It Matters
teaser: Deploy a waypoint proxy for HTTP-aware authorization and traffic management.
tabs:
- id: fhff7spj3k2s
  title: Terminal
  type: terminal
  hostname: server
- id: b8tsqhfqqwbu
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Waypoints — L7 Where It Matters

## When L4 Isn't Enough

So far, ztunnel has been handling everything at L4 — TCP connections, mTLS, namespace-based authorization. But what if you need:

- HTTP method-based access control (allow GET but deny POST)?
- Header-based routing?
- Request-level telemetry?

That's where **waypoint proxies** come in. A waypoint is an Envoy-based L7 proxy that you deploy *only* for the namespaces or services that need it. The rest of your mesh stays lightweight on ztunnel.

> 🏢 **Enterprise Value Beat:** Waypoints let you apply L7 features surgically — one namespace at a time. No more "all or nothing" proxy deployment.

## Your Tasks

### 1. Deploy a Waypoint for the Accounts Namespace

```bash
istioctl waypoint apply -n accounts --enroll-namespace
```

### 2. Verify the Waypoint Is Running

```bash
kubectl get pods -n accounts -l gateway.networking.k8s.io/gateway-name=waypoint
```

You should see a waypoint pod running. This is an Envoy proxy dedicated to the accounts namespace.

```bash
kubectl get gateway -n accounts
```

### 3. Apply an L7 Authorization Policy

Now let's create a policy that only allows GET requests — POST, PUT, DELETE will be denied:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-get-only
  namespace: accounts
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
```

### 4. Test GET vs POST

GET should work:

```bash
kubectl exec curl -n finance -- curl -sS accounts.accounts:8080
```

POST should be denied:

```bash
kubectl exec curl -n finance -- curl -sS -X POST accounts.accounts:8080 || echo "POST denied as expected"
```

You should see the GET succeed and the POST get a `403 Forbidden` (or RBAC denied).

### 5. Verify the Traffic Path

Check waypoint logs to see it processing requests:

```bash
kubectl logs -n accounts -l gateway.networking.k8s.io/gateway-name=waypoint --tail=10
```

> 💡 **Key Insight:** The L4 `allow-finance` policy from the previous challenge is still active at the ztunnel level. The L7 `allow-get-only` policy adds *additional* HTTP-level filtering at the waypoint. They work together — ztunnel handles identity, the waypoint handles HTTP semantics.

> 🏢 **Enterprise Value Beat:** In a sidecar mesh, every pod runs an L7 proxy whether it needs one or not. With ambient, you only pay for L7 where you actually use it. Most services only need L4 — save the waypoints for the services that need fine-grained HTTP control.
