---
slug: first-ai-gateway
id: tuokfmqtlbfn
type: challenge
title: Create Your First AI Gateway
teaser: Deploy a Gateway resource and route traffic to OpenAI through AgentGateway.
tabs:
- id: xyyfq8ho9qsz
  title: Terminal
  type: terminal
  hostname: server
- id: q7rcjvnxo1e0
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Create Your First AI Gateway

Now for the exciting part — we'll create a Gateway that sits between your agents and OpenAI. Instead of agents calling OpenAI directly, they'll call the gateway, which proxies the request with full visibility and control.

## Why This Matters

With a gateway in the middle, you get:
- **Credential isolation** — API keys live in Kubernetes Secrets, not in agent code
- **Centralized logging** — every request is logged at the gateway
- **A single endpoint** — agents call `gateway:8080` instead of memorizing provider URLs

## Step 1: Store Your API Key as a Secret

First, store your OpenAI API key as a Kubernetes Secret. For this demo, we'll use a placeholder:

```bash
kubectl create secret generic openai-api-key \
  --namespace agentgateway-system \
  --from-literal=api-key="sk-demo-placeholder-key"
```

> 💡 In production, you'd use a real key here. The important thing is that **agents never see this key** — only the gateway does.

## Step 2: Create a Gateway

Create the Gateway resource that will listen for AI agent traffic:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ai-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
    - name: http
      protocol: HTTP
      port: 8080
EOF
```

Check its status:

```bash
kubectl get gateway -n agentgateway-system
```

Wait until the gateway shows `Programmed: True`:

```bash
kubectl wait --for=condition=Programmed gateway/ai-gateway \
  -n agentgateway-system --timeout=60s
```

## Step 3: Create an OpenAI Backend and Route

Now create a Backend that tells AgentGateway how to reach OpenAI, and an HTTPRoute that sends traffic there:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-backend
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai:
        model: gpt-4o-mini
  policies:
    auth:
      secretRef:
        name: openai-api-key
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-route
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: ai-gateway
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai
      backendRefs:
        - group: agentgateway.dev
          kind: AgentgatewayBackend
          name: openai-backend
EOF
```

Verify both resources:

```bash
kubectl get agentgatewaybackend,httproute -n agentgateway-system
```

## Step 4: Test Your Gateway

Port-forward to the gateway:

```bash
kubectl port-forward -n agentgateway-system svc/ai-gateway 8080:8080 &
sleep 3
```

Now test it! (This will get an auth error since we used a placeholder key, but it proves the gateway is routing correctly):

```bash
curl -s http://localhost:8080/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }' | jq .
```

You should see a response from OpenAI (even if it's an auth error). The important thing: **the request went through AgentGateway**, not directly to OpenAI.

## What Changed?

Before: `Agent → OpenAI (direct, unmonitored)`
After: `Agent → AgentGateway → OpenAI (proxied, logged, controlled)`

The agent doesn't need the API key. It doesn't even need to know it's talking to OpenAI. It just calls the gateway.
