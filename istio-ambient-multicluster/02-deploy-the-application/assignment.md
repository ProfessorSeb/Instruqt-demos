---
slug: deploy-the-application
id: fyre4qrlzwb7
type: challenge
title: Deploy the Application
teaser: Deploy a multi-namespace application and test baseline connectivity.
tabs:
- id: vaccwlxmenyh
  title: Terminal
  type: terminal
  hostname: server
- id: xvqvwgcipwej
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Deploy the Application

## The Scenario

You're building a platform where a **finance** team needs to call an **accounts** service. Different teams, different namespaces — a common Kubernetes pattern.

Right now, there's no mesh enrollment. Traffic flows in plain text. Let's set that up as a baseline, then we'll lock it down.

## Your Tasks

### 1. Create the Namespaces

```bash
kubectl create namespace finance
kubectl create namespace accounts
```

### 2. Deploy the Accounts Service

```bash
cat <<EOF | kubectl apply -f -
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
      containers:
      - name: accounts
        image: hashicorp/http-echo:1.0
        args: ["-listen=:8080", "-text=accounts-v1 responding"]
        ports:
        - containerPort: 8080
---
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
```

### 3. Deploy the Curl Client

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: curl
  namespace: finance
  labels:
    app: curl
spec:
  containers:
  - name: curl
    image: curlimages/curl:8.5.0
    command: ["sleep", "infinity"]
EOF
```

### 4. Wait for Pods and Test Connectivity

```bash
kubectl wait --for=condition=Ready pod -l app=accounts -n accounts --timeout=60s
kubectl wait --for=condition=Ready pod/curl -n finance --timeout=60s
```

Now test that finance can reach accounts:

```bash
kubectl exec curl -n finance -- curl -sS accounts.accounts:8080
```

You should see: `accounts-v1 responding`

> 🏢 **Enterprise Value Beat:** This is your "day zero" baseline — plain HTTP, no encryption, no identity, no authorization. Any pod in the cluster can call any service. In the next challenge, we'll fix that with a single label.

### 5. Verify Pod Count

```bash
kubectl get pods -n accounts
kubectl get pods -n finance
```

Notice: **one container per pod**. No sidecars. That won't change even after we enroll in the mesh.
