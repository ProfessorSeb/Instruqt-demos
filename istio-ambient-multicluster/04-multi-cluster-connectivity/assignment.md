---
slug: multi-cluster-connectivity
type: challenge
title: "Multi-Cluster Connectivity"
teaser: "Verify cross-cluster service discovery and load balancing."
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
---

# Multi-Cluster Connectivity

## Why Cross-Cluster Service Discovery Matters

With a single cluster, services find each other through Kubernetes DNS. But what about identical services running in another cluster? This is where Istio's multi-cluster magic kicks in.

Because we configured:
1. **Shared trust** (same root CA) — clusters trust each other's identities
2. **Remote secrets** — each cluster's istiod can read the other's API server
3. **East-west gateways** — encrypted tunnels between clusters

Istio automatically merges the service registries. When you call `accounts.accounts.svc.cluster.local`, Istio knows about endpoints in **both** clusters and load-balances across them.

> 🏢 **Enterprise Value Beat:** Cross-cluster service discovery means your applications don't need to know about cluster topology. They just call services by name, and the mesh handles routing — even across regions.

## Your Task

### Step 1: Test Cross-Cluster Connectivity

From cluster-1's finance namespace, curl the accounts service. Because Istio merges endpoints from both clusters, you should get responses from both `accounts-cluster-1` AND `accounts-cluster-2`:

```bash
kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
  curl -s accounts.accounts.svc.cluster.local:8080
```

Run it a few times — you might get either response!

### Step 2: Prove Load Balancing with a Loop

Run 10 requests and observe the distribution across clusters:

```bash
for i in $(seq 1 10); do
  kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
    curl -s accounts.accounts.svc.cluster.local:8080
done
```

You should see a mix of `accounts-cluster-1` and `accounts-cluster-2` responses. This proves that Istio is load-balancing across both clusters transparently.

### Step 3: Save the Results

Save the output of a load-balancing test to a file:

```bash
for i in $(seq 1 10); do
  kubectl --context=$CTX_CLUSTER1 exec -n finance deploy/curl -- \
    curl -s accounts.accounts.svc.cluster.local:8080
done > /root/multicluster-results.txt

cat /root/multicluster-results.txt
```

### Step 4: Test from Cluster 2

Verify the same works from cluster-2:

```bash
for i in $(seq 1 5); do
  kubectl --context=$CTX_CLUSTER2 exec -n finance deploy/curl -- \
    curl -s accounts.accounts.svc.cluster.local:8080
done
```

Click **Check** when you've saved the results file!
