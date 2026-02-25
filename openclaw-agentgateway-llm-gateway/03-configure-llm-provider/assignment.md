---
slug: configure-llm-provider
id: ""
type: challenge
title: Configure the OpenAI LLM Provider
teaser: Route LLM traffic through AgentGateway — no API key in your app, full visibility into every call.
notes:
- type: text
  contents: |-
    # 🔑 Centralized Credential Management

    In the last challenge you saw the problem: your OpenAI API key was sitting in an
    environment variable, accessible to anything on the machine.

    AgentGateway solves this with a **backend + secret** model:

    ```
    Your App  ──▶  AgentGateway  ──▶  OpenAI
    (no key)       (holds the key)    (gets the key)
    ```

    You'll create three Kubernetes resources:

    | Resource | Purpose |
    |---|---|
    | `Secret` | Stores the API key — only the gateway reads it |
    | `AgentgatewayBackend` | Defines OpenAI as an LLM provider |
    | `Gateway` + `AIRoute` | Exposes the proxy and routes traffic |

    After this challenge, your app never needs to know the API key.
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 900
---

# Configure the OpenAI LLM Provider

## Step 1 — Store the API Key as a Kubernetes Secret

This is the key shift: the API key lives in Kubernetes, not in your app.
Only AgentGateway reads it. Your app doesn't need it.

```bash
kubectl create namespace llm-gateway

kubectl create secret generic openai-api-key \
  --namespace agentgateway-system \
  --from-literal=apiKey="${OPENAI_API_KEY}"
```

Verify it's stored (the value is never shown):

```bash
kubectl get secret openai-api-key -n agentgateway-system
```

## Step 2 — Create the AgentgatewayBackend

The `AgentgatewayBackend` tells AgentGateway *what* to proxy to — in this case, OpenAI.
Notice there's no API key here. It references the secret by name:

```bash
kubectl apply -f - <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
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
        namespace: agentgateway-system
EOF
```

## Step 3 — Create the Gateway and Route

The `Gateway` listens on port 80 (mapped to `localhost:8080`).
The `AIRoute` connects the Gateway to the OpenAI backend:

```bash
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: llm-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - name: http
    port: 80
    protocol: HTTP
---
apiVersion: agentgateway.dev/v1alpha1
kind: AIRoute
metadata:
  name: openai-route
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: llm-gateway
    namespace: agentgateway-system
  rules:
  - backendRefs:
    - name: openai
      namespace: agentgateway-system
EOF
```

Save the route manifest for later — you'll need it for the kill switch:

```bash
kubectl get aiRoute openai-route -n agentgateway-system -o yaml > /root/openai-route.yaml
```

Wait for the Gateway to be programmed:

```bash
kubectl wait gateway llm-gateway \
  -n agentgateway-system \
  --for=condition=Programmed \
  --timeout=60s
```

## Step 4 — Test It (No API Key Required)

Make a request through the gateway. **Notice: no `Authorization` header** — the
gateway handles auth internally:

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What is AgentGateway in one sentence?"}]
  }' | jq '.choices[0].message.content'
```

You should get a real OpenAI response. Your app sent no credentials. 🎉

## What Changed?

Compare the two approaches:

| | Direct Call | Via AgentGateway |
|---|---|---|
| API key location | Environment variable | Kubernetes Secret |
| Who holds the key | Your app | The gateway |
| Visibility | None | Full (logs, metrics) |
| Control | None | Routes, policies |
| Revocation | Rotate key everywhere | Update one secret |

> 💡 If you need to rotate your OpenAI key, you update **one Kubernetes secret**.
> Every app behind the gateway picks it up automatically. No redeployments.
