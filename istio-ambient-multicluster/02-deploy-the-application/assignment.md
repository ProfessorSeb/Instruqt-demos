---
slug: deploy-the-application
type: challenge
title: "Deploy the Application"
teaser: "Deploy the accounts service and curl clients across both clusters."
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
---

# Deploy the Application

## Why This Architecture?

Before we can demonstrate multi-cluster service mesh capabilities, we need workloads. We'll deploy a simple but realistic pattern:

- An **accounts** service that serves HTTP responses (simulating a backend API)
- A **curl** client in a separate **finance** namespace (simulating a frontend calling the backend)

By deploying the same service to **both clusters**, we set up the foundation for cross-cluster load balancing — a common pattern for high availability.

> 🏢 **Enterprise Value Beat:** Running the same service across multiple clusters gives you automatic failover. If one region goes down, traffic seamlessly shifts to the other.

## Your Task

### Step 1: Create Namespaces on Both Clusters

Create the `finance` and `accounts` namespaces on both clusters:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx create namespace finance
  kubectl --context=$ctx create namespace accounts
done
```

### Step 2: Deploy the Accounts Service

Deploy the `accounts` service on **both clusters**. Note that cluster-1 returns "accounts-cluster-1" and cluster-2 returns "accounts-cluster-2" — this lets us see which cluster responds.

**Cluster 1:**

```bash
kubectl --context=$CTX_CLUSTER1 apply -n accounts -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: accounts
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: accounts
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
        args: ["-text=accounts-cluster-1", "-listen=:8080"]
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: accounts
spec:
  selector:
    app: accounts
  ports:
  - port: 8080
    targetPort: 8080
EOF
```

**Cluster 2:**

```bash
kubectl --context=$CTX_CLUSTER2 apply -n accounts -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: accounts
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: accounts
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
        args: ["-text=accounts-cluster-2", "-listen=:8080"]
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: accounts
spec:
  selector:
    app: accounts
  ports:
  - port: 8080
    targetPort: 8080
EOF
```

### Step 3: Deploy the Curl Client

Deploy a `curl` pod in the `finance` namespace on both clusters:

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx apply -n finance -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: curl
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      serviceAccountName: curl
      containers:
      - name: curl
        image: curlimages/curl:8.5.0
        command: ["sleep", "infinity"]
EOF
done
```

### Step 4: Wait for Pods to Be Ready

```bash
for ctx in $CTX_CLUSTER1 $CTX_CLUSTER2; do
  kubectl --context=$ctx -n accounts wait --for=condition=Ready pod -l app=accounts --timeout=60s
  kubectl --context=$ctx -n finance wait --for=condition=Ready pod -l app=curl --timeout=60s
done
```

### Step 5: Verify Deployments

```bash
echo "=== Cluster 1 ==="
kubectl --context=$CTX_CLUSTER1 get pods -n accounts
kubectl --context=$CTX_CLUSTER1 get pods -n finance
echo "=== Cluster 2 ==="
kubectl --context=$CTX_CLUSTER2 get pods -n accounts
kubectl --context=$CTX_CLUSTER2 get pods -n finance
```

Click **Check** when all pods are running!
