---
slug: telemetry-and-gitops
type: challenge
title: "Telemetry with Dynamic Tags + GitOps Structure"
teaser: "Configure production telemetry and organize your mesh config for GitOps."
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
---

# Telemetry with Dynamic Tags + GitOps Structure

## Why Custom Telemetry Matters

Istio generates metrics, traces, and access logs out of the box. But production teams need more:

- **Custom tags** on metrics to slice and dice by business attributes
- **Error classification** to distinguish transient from permanent failures
- **SLO tracking** with custom dimensions

The `Telemetry` custom resource lets you configure this declaratively — and it works with both ztunnel (L4) and waypoint (L7) telemetry.

> 🏢 **Enterprise Value Beat:** Custom telemetry tags let your SRE team build dashboards that map to business outcomes, not just infrastructure metrics. "Errors in the accounts service for the finance team" is more actionable than "5xx on pod xyz."

## Your Task

### Step 1: Apply Telemetry Configuration

Apply a `Telemetry` resource with custom tag overrides on both clusters:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx apply -n accounts -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: accounts-telemetry
spec:
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: REQUEST_COUNT
      tagOverrides:
        error_type:
          operation: UPSERT
          value: "response.code >= 500 ? 'server_error' : (response.code >= 400 ? 'client_error' : 'none')"
        slow_request:
          operation: UPSERT
          value: "response.duration > 1000 ? 'true' : 'false'"
        source_namespace:
          operation: UPSERT
          value: "source.namespace | 'unknown'"
  accessLogging:
  - providers:
    - name: envoy
EOF
done
```

### Step 2: Verify the Telemetry Resource

```bash
kubectl --context=$CTX_CLUSTER1 get telemetry -n accounts
kubectl --context=$CTX_CLUSTER2 get telemetry -n accounts
```

### Step 3: Create the GitOps Repository Structure

In production, all mesh configuration should be stored in Git. Create a structured directory that follows GitOps best practices:

```bash
mkdir -p /root/mesh-lab/{base,clusters/{cluster-1,cluster-2},apps/{accounts,finance}}

# Base layer — shared across all clusters
cat > /root/mesh-lab/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- namespace-accounts.yaml
- namespace-finance.yaml
- telemetry.yaml
EOF

cat > /root/mesh-lab/base/namespace-accounts.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: accounts
  labels:
    istio.io/dataplane-mode: ambient
EOF

cat > /root/mesh-lab/base/namespace-finance.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: finance
  labels:
    istio.io/dataplane-mode: ambient
EOF

cat > /root/mesh-lab/base/telemetry.yaml << 'EOF'
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: accounts-telemetry
  namespace: accounts
spec:
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: REQUEST_COUNT
      tagOverrides:
        error_type:
          operation: UPSERT
          value: "response.code >= 500 ? 'server_error' : (response.code >= 400 ? 'client_error' : 'none')"
EOF

# Cluster-specific overlays
for cluster in cluster-1 cluster-2; do
  cat > /root/mesh-lab/clusters/$cluster/kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
- authorization-policies.yaml
EOF

  cat > /root/mesh-lab/clusters/$cluster/authorization-policies.yaml << 'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: accounts
spec:
  {}
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-from-finance
  namespace: accounts
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["finance"]
EOF
done

# App manifests
cat > /root/mesh-lab/apps/accounts/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: accounts
  namespace: accounts
spec:
  replicas: 1
  selector:
    matchLabels:
      app: accounts
  template:
    metadata:
      labels:
        app: accounts
    spec:
      serviceAccountName: accounts
      containers:
      - name: accounts
        image: hashicorp/http-echo:1.0
        args: ["-text=accounts", "-listen=:8080"]
        ports:
        - containerPort: 8080
EOF

cat > /root/mesh-lab/apps/accounts/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: accounts
  namespace: accounts
spec:
  selector:
    app: accounts
  ports:
  - port: 8080
    targetPort: 8080
EOF

# README
cat > /root/mesh-lab/README.md << 'EOF'
# Mesh Lab — Istio Ambient Multi-Cluster GitOps

## Structure

```
mesh-lab/
├── base/           # Shared resources (namespaces, telemetry)
├── clusters/       # Per-cluster overlays (auth policies)
│   ├── cluster-1/
│   └── cluster-2/
└── apps/           # Application manifests
    ├── accounts/
    └── finance/
```

## Usage

```bash
# Apply base + cluster overlay for cluster-1
kustomize build clusters/cluster-1 | kubectl --context=k3d-cluster-1 apply -f -
```
EOF
```

### Step 4: Verify the Structure

```bash
find /root/mesh-lab -type f | sort
```

## 🎉 Congratulations!

You've built a complete Istio ambient multi-cluster mesh! Here's what you accomplished:

| Layer | What You Did |
|-------|-------------|
| **Infrastructure** | Two Kubernetes clusters with shared trust |
| **L4 Zero Trust** | Ambient enrollment with automatic mTLS |
| **Multi-Cluster** | Cross-cluster service discovery and load balancing |
| **Authorization** | Deny-by-default + namespace-scoped allow policies |
| **L7 Waypoints** | On-demand Envoy proxies for HTTP-aware policies |
| **Telemetry** | Custom metrics tags for production observability |
| **GitOps** | Structured repo ready for Argo CD or Flux |

Click **Check** to complete the track!
