---
slug: configure-llm-gateway
id: ""
type: challenge
title: Configure the OpenAI LLM Provider
teaser: Create the OpenAI backend, expose a Gateway, and verify LLM traffic flows through AgentGateway.
notes:
- type: text
  contents: |-
    # 🔑 Configure the LLM Gateway

    AgentGateway is running — now let's tell it where to send LLM traffic.

    You'll create three Kubernetes resources:

    - **Secret** — stores your OpenAI API key
    - **AgentgatewayBackend** — defines OpenAI as the LLM provider
    - **Gateway + HTTPRoute** — exposes the proxy on port 80 (mapped to `localhost:8080`)

    After this challenge, you'll be able to `curl localhost:8080` and get a real OpenAI response — with AgentGateway in the middle.
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

## Store the OpenAI API Key as a Kubernetes Secret

The API key is already available as `$OPENAI_API_KEY` in your environment (provided by the lab).
Store it as a Kubernetes secret that AgentGateway will use:

```bash
kubectl create namespace llm-gateway

kubectl create secret generic openai-api-key \
  --namespace agentgateway-system \
  --from-literal=apiKey="${OPENAI_API_KEY}"
```

## Create the AgentgatewayBackend

This resource tells AgentGateway to route traffic to OpenAI using the secret above:

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

## Create the Gateway and HTTPRoute

Expose AgentGateway on port 80 (mapped to `localhost:8080` on this host):

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

Wait for the Gateway to be ready:

```bash
kubectl wait gateway llm-gateway \
  -n agentgateway-system \
  --for=condition=Programmed \
  --timeout=60s
```

## Test the Gateway

Send a test request through AgentGateway to OpenAI:

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Say hello in one sentence."}]
  }' | jq '.choices[0].message.content'
```

You should get a response from OpenAI — routed through your gateway. 🎉

> Notice: you didn't pass an API key in the curl request. AgentGateway handled authentication internally using the Kubernetes secret.
