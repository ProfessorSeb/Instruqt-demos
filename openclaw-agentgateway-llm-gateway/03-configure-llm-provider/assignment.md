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

    You'll create four Kubernetes resources:

    | Resource | Purpose |
    |---|---|
    | `Secret` | Stores the API key — only the gateway reads it |
    | `AgentgatewayBackend` | Defines OpenAI as an LLM provider |
    | `Gateway` | Exposes the proxy on port 8080 |
    | `HTTPRoute` | Routes `/openai` traffic to the backend |

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

The API key lives in Kubernetes. Your app never sees it.

```bash
kubectl create namespace llm-gateway

kubectl create secret generic openai-secret \
  --namespace agentgateway-system \
  --from-literal="Authorization=Bearer ${OPENAI_API_KEY}"
```

## Step 2 — Create the AgentgatewayBackend

Defines OpenAI as an LLM provider. References the secret — no key in the config:

```bash
kubectl apply -f - <<'EOF'
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
        name: openai-secret
EOF
```

## Step 3 — Create the Gateway and HTTPRoute

The `Gateway` listens on port 8080. The `HTTPRoute` maps `/openai` traffic to your backend:

```bash
kubectl apply -f - <<'EOF'
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

Wait for the Gateway to be ready:

```bash
kubectl wait --for=condition=Programmed gateway/ai-gateway \
  -n agentgateway-system --timeout=60s
```

Save the route for the kill switch challenge:

```bash
kubectl get httproute openai-route -n agentgateway-system -o yaml > /root/openai-route.yaml
```

## Step 4 — Expose the Gateway and Test

Port-forward to reach the gateway from this host:

```bash
kubectl port-forward -n agentgateway-system svc/ai-gateway 8080:8080 &
sleep 2
```

Now test — **no API key in the request**:

```bash
curl -s http://localhost:8080/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What is AgentGateway in one sentence?"}]
  }' | jq '.choices[0].message.content'
```

You should get a real OpenAI response — routed through AgentGateway. 🎉

> 💡 **What changed?** Your app sent no credentials. The gateway authenticated
> the request using the Kubernetes secret you created. One secret, one place,
> managed by the platform — not scattered across every app.
