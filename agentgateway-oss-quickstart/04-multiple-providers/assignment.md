---
slug: multiple-providers
id: gr3edurbowtx
type: challenge
title: Add Multiple LLM Providers
teaser: Route to both OpenAI and Anthropic through a single gateway with path-based
  routing.
notes:
- type: text
  contents: "# \U0001F500 Multi-Provider Routing\n\nMost teams use multiple LLM providers.
    Without a gateway, that's N×M complexity.\n\n**In this challenge, you'll:**\n\n-
    Add Anthropic as a second LLM backend\n- Configure path-based routing (`/openai/*`
    and `/anthropic/*`)\n- Test both routes through a single gateway endpoint\n"
tabs:
- id: yby5cmiejbio
  title: Terminal
  type: terminal
  hostname: server
- id: mkzjmo0elinf
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Add Multiple LLM Providers

Most organizations don't use just one LLM provider. Teams choose different models for different use cases — GPT-4o for general tasks, Claude for long-context analysis, Gemini for multimodal work.

Without a gateway, each team integrates with each provider independently. That's N×M complexity.

With Agentgateway, you add a new provider once, and every agent can use it through the same gateway endpoint.

## Why This Matters

A single gateway for multiple providers means:
- **One endpoint** for agents to call, regardless of provider
- **Path-based routing** — `/openai/*` goes to OpenAI, `/anthropic/*` goes to Anthropic
- **Swap providers** without changing agent code
- **Compare providers** by routing the same traffic to both

## Step 1: Add an Anthropic API Key Secret

For this demo, we'll use a placeholder Anthropic key. The OpenAI route will return real responses, while the Anthropic route demonstrates the multi-provider routing concept:

```bash
kubectl create secret generic anthropic-secret \
  --namespace agentgateway-system \
  --from-literal="Authorization=Bearer sk-ant-demo-placeholder-key"
```

> 💡 In production, you'd use a real Anthropic API key here. The point is that adding a new provider is just one Backend + one Route.

## Step 2: Create the Anthropic Backend and Route

```bash
cat <<EOF | kubectl apply -f -
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: anthropic-backend
  namespace: agentgateway-system
spec:
  ai:
    provider:
      anthropic:
        model: claude-sonnet-4-20250514
  policies:
    auth:
      secretRef:
        name: anthropic-secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: anthropic-route
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: ai-gateway
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /anthropic
      backendRefs:
        - group: agentgateway.dev
          kind: AgentgatewayBackend
          name: anthropic-backend
EOF
```

## Step 3: Verify Your Multi-Provider Setup

Check all the resources:

```bash
kubectl get agentgatewaybackend,httproute -n agentgateway-system
```

You should see two backends (openai-backend, anthropic-backend) and two routes (openai-route, anthropic-route).

## Step 4: Test Both Routes

Make sure port-forward is running (start it if it's not):

```bash
# Kill any existing port-forward
pkill -f "port-forward.*ai-gateway" || true
kubectl port-forward -n agentgateway-system svc/ai-gateway 8080:8080 &
sleep 3
```

Test the OpenAI route:

```bash
echo "--- Testing OpenAI route ---"
curl -s http://localhost:8080/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello from OpenAI route!"}],
    "max_tokens": 50
  }' | jq .
```

Test the Anthropic route:

```bash
echo "--- Testing Anthropic route ---"
curl -s http://localhost:8080/anthropic/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 50,
    "messages": [{"role": "user", "content": "Hello from Anthropic route!"}]
  }' | jq .
```

The OpenAI route should return a real response. The Anthropic route will return an auth error (placeholder key), but it proves the routing works — both go through the **same gateway** to **different providers**. Your agents just need to know one hostname.

## The Big Picture

```
                    ┌─────────────────────┐
Agent A ──→        │                     │──→ OpenAI
                    │   Agentgateway      │
Agent B ──→        │   (ai-gateway)      │──→ Anthropic
                    │                     │
Agent C ──→        │   /openai/*         │──→ OpenAI
                    │   /anthropic/*      │──→ Anthropic
                    └─────────────────────┘
```

One gateway. Multiple providers. Full control.
