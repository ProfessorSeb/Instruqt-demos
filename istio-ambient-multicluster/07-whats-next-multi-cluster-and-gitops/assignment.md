---
slug: whats-next-multi-cluster-and-gitops
id: ws9i3voecbxy
type: challenge
title: What's Next — Multi-Cluster & GitOps
teaser: Structure your mesh config for GitOps and learn how to extend to multi-cluster.
tabs:
- id: 8l1f8n9fghps
  title: Terminal
  type: terminal
  hostname: server
- id: 3nu7wozj5vqf
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# What's Next — Multi-Cluster & GitOps

## From Lab to Production

You've built a complete ambient mesh on a single cluster:
- ✅ Automatic mTLS via ztunnel (L4)
- ✅ Zero-trust authorization policies
- ✅ L7 waypoint proxies where needed
- ✅ Custom production telemetry

Now let's structure this for GitOps and understand how to scale to multi-cluster when you're ready.

## Your Tasks

### 1. Create a GitOps Directory Structure

Organize your mesh configuration the way a platform team would:

```bash
mkdir -p /root/mesh-lab/{base,overlays/{dev,staging,prod},policies,telemetry,waypoints}

# Base Istio config
cat > /root/mesh-lab/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- namespace-finance.yaml
- namespace-accounts.yaml
EOF

cat > /root/mesh-lab/base/namespace-finance.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: finance
  labels:
    istio.io/dataplane-mode: ambient
EOF

cat > /root/mesh-lab/base/namespace-accounts.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: accounts
  labels:
    istio.io/dataplane-mode: ambient
EOF

# Policies
cat > /root/mesh-lab/policies/deny-all.yaml << 'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: accounts
spec: {}
EOF

cat > /root/mesh-lab/policies/allow-finance.yaml << 'EOF'
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

# Telemetry
cat > /root/mesh-lab/telemetry/accounts-telemetry.yaml << 'EOF'
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
        mode: SERVER
      tagOverrides:
        error_type:
          operation: UPSERT
          value: "response.code >= 500 ? 'server_error' : response.code >= 400 ? 'client_error' : 'none'"
EOF

echo "GitOps structure created!"
```

### 2. Review the Structure

```bash
find /root/mesh-lab -type f | sort
```

### 3. Multi-Cluster — The Next Step

When you're ready to expand to multiple clusters, here's what you'll need:

**Shared Trust (Certificate Authority)**
- Both clusters need certificates from the same root CA
- Istio provides `tools/certs/Makefile.selfsigned.mk` for generating these
- In production, use cert-manager or Vault

**Service Discovery (Remote Secrets)**
- Each cluster needs a `kubeconfig` secret for the other cluster
- `istioctl create-remote-secret` generates these
- This lets istiod discover services across clusters

**East-West Gateways**
- Cross-cluster traffic flows through dedicated east-west gateways
- These handle mTLS tunneling between cluster networks
- Deploy with `samples/multicluster/gen-eastwest-gateway.sh`

**Network Configuration**
- Label each cluster's `istio-system` with `topology.istio.io/network`
- Configure istiod with `global.meshID`, `global.multiCluster.clusterName`, and `global.network`

```bash
# Document the multi-cluster roadmap
cat > /root/mesh-lab/MULTI-CLUSTER-ROADMAP.md << 'MEOF'
# Multi-Cluster Ambient Mesh Roadmap

## Prerequisites
1. Shared root CA across clusters
2. Network connectivity between cluster nodes (or east-west gateways)
3. Unique cluster names in the mesh

## Steps
1. Generate shared CA certificates
2. Install Istio ambient on each cluster with multi-cluster config
3. Deploy east-west gateways
4. Exchange remote secrets
5. Verify cross-cluster service discovery

## Key Helm Values
- global.meshID=mesh1
- global.multiCluster.clusterName=<unique-name>
- global.network=<network-name>
- pilot.env.AMBIENT_ENABLE_MULTI_NETWORK=true
MEOF

echo "Roadmap documented!"
```

## Recap

In this track, you've learned the complete ambient mesh workflow:

| Layer | Component | What It Does |
|-------|-----------|-------------|
| L4 | ztunnel | Per-node mTLS, TCP auth policies, basic telemetry |
| L7 | Waypoint | HTTP routing, method-based auth, rich telemetry |
| Control | istiod | Config distribution, certificate management |
| Infra | CNI | Traffic redirection to ztunnel |

The ambient architecture lets you start simple (L4 everywhere) and add complexity (L7 waypoints) only where business logic demands it. That's not just a technical choice — it's an operational philosophy.

> 🏢 **Enterprise Value Beat:** You've gone from zero to production-grade mesh in under an hour. The GitOps structure means your platform team can version-control every policy, every telemetry config, every waypoint decision. Audit trail included.
