---
slug: configure-llm-provider
id: xkavxawgrix2
type: challenge
title: Configure the OpenAI LLM Provider
teaser: Route LLM traffic through AgentGateway — no API key in your app, full visibility
  into every call.
notes:
- type: text
  contents: "# \U0001F511 Centralized Credential Management\n\nAgentGateway solves
    the API key problem with a **backend + secret** model:\n\n```\nYour App  ──▶  AgentGateway
    \ ──▶  OpenAI\n(no key)       (holds the key)\n```\n\nYou'll create four Kubernetes
    resources:\n\n| Resource | Purpose |\n|---|---|\n| `Secret` | Stores the API key
    — only the gateway reads it |\n| `AgentgatewayBackend` | Defines OpenAI as an
    LLM provider |\n| `Gateway` | Exposes the proxy on port 80 |\n| `HTTPRoute` |
    Routes traffic to the backend |\n\nAfter this challenge, your app never needs
    to know the API key."
tabs:
- id: k6jy6ou9frm6
  title: Terminal
  type: terminal
  hostname: server
- id: jgk1eccktm41
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 900
enhanced_loading: null
---

# Configure the OpenAI LLM Provider

## Step 1 — Store the API Key as a Kubernetes Secret

The API key lives in Kubernetes. Your app never sees it.

```bash
kubectl apply -f- <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: openai-secret
  namespace: agentgateway-system
type: Opaque
stringData:
  Authorization: ${OPENAI_API_KEY}
EOF
```

## Step 2 — Create the AgentgatewayBackend

Tells AgentGateway to route to OpenAI and which secret holds the credentials:

```bash
kubectl apply -f- <<'EOF'
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
        name: openai-secret
EOF
```

## Step 3 — Create the Gateway and HTTPRoute

The `Gateway` listens on port 80. The `HTTPRoute` connects it to the OpenAI backend:

```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - protocol: HTTP
    port: 80
    name: http
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - backendRefs:
    - name: openai
      namespace: agentgateway-system
      group: agentgateway.dev
      kind: AgentgatewayBackend
EOF
```

Wait for the Gateway:

```bash
kubectl wait --for=condition=Programmed gateway/agentgateway-proxy \
  -n agentgateway-system --timeout=60s
```

Save the route manifest for the kill switch challenge:

```bash
kubectl get httproute openai -n agentgateway-system -o yaml > /root/openai-route.yaml
```

## Step 4 — Expose the Gateway and Test

Port-forward to reach the gateway from this host:

```bash
kubectl port-forward deployment/agentgateway-proxy \
  -n agentgateway-system 8080:80 &
sleep 2
```

Test — **no API key in the request**:

```bash
curl localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What is AgentGateway in one sentence?"}]
  }' | jq '.choices[0].message.content'
```

You should get a real OpenAI response — routed through AgentGateway. 🎉

> 💡 No credentials in your app. No API key in the request. The gateway
> authenticated using the Kubernetes secret. One secret, one place.
