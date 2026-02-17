---
slug: multiple-providers
id: gr3edurbowtx
type: challenge
title: Load Balance Across OpenAI Models
teaser: Distribute traffic across multiple OpenAI models through a single gateway
  route using weighted load balancing.
notes:
- type: text
  contents: |
    # ⚖️ Model Load Balancing

    Not every request needs the same model. Agentgateway can distribute traffic across models automatically.

    **In this challenge, you'll:**

    - Create multiple OpenAI backends (GPT-4o-mini and GPT-4o)
    - Configure weighted load balancing across models
    - Test traffic distribution through a single route
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

# Load Balance Across OpenAI Models

In production, you often want to distribute traffic across different models. Maybe you send 80% of traffic to a cheaper model (GPT-4o-mini) and 20% to a more capable one (GPT-4o) for quality comparison. Or you want to gradually shift traffic during a model migration.

Agentgateway makes this simple with **weighted backend references** — the same pattern Kubernetes Gateway API uses for canary deployments.

## Why This Matters

Model load balancing gives you:
- **Cost optimization** — route most traffic to cheaper models, expensive models only when needed
- **A/B testing** — compare model quality on real traffic
- **Gradual migration** — shift traffic from one model to another without downtime
- **Resilience** — if one model has issues, traffic flows to the other

## Step 1: Create a Second OpenAI Backend

You already have `openai-backend` pointing to `gpt-4o-mini`. Let's add a second backend for `gpt-4o`:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-gpt4o-backend
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai:
        model: gpt-4o
  policies:
    auth:
      secretRef:
        name: openai-secret
EOF
```

Both backends use the same `openai-secret` — same API key, different models.

## Step 2: Update the Route for Weighted Load Balancing

Now update the OpenAI route to distribute traffic across both backends. We'll send **80% to GPT-4o-mini** (fast and cheap) and **20% to GPT-4o** (more capable):

```bash
cat <<EOF | kubectl apply -f -
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
          weight: 80
        - group: agentgateway.dev
          kind: AgentgatewayBackend
          name: openai-gpt4o-backend
          weight: 20
EOF
```

## Step 3: Verify Your Load Balancing Setup

Check all the resources:

```bash
kubectl get agentgatewaybackend,httproute -n agentgateway-system
```

You should see two backends (`openai-backend` and `openai-gpt4o-backend`) and the updated route.

Inspect the route to confirm the weights:

```bash
kubectl get httproute openai-route -n agentgateway-system -o yaml | grep -A 10 backendRefs
```

## Step 4: Test Load Balancing

Make sure port-forward is running:

```bash
# Kill any existing port-forward
pkill -f "port-forward.*ai-gateway" || true
kubectl port-forward -n agentgateway-system svc/ai-gateway 8080:8080 &
sleep 3
```

Send several requests and observe which model responds:

```bash
for i in $(seq 1 5); do
  echo "--- Request $i ---"
  curl -s http://localhost:8080/openai/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "gpt-4o-mini",
      "messages": [{"role": "user", "content": "What model are you? Reply in 5 words."}],
      "max_tokens": 20
    }' | jq -r '.model // .error.message'
  echo
done
```

You should see responses from both `gpt-4o-mini` and `gpt-4o` — roughly 80/20 split. The gateway overrides the model based on which backend receives the request.

> 💡 **Key insight:** The agent sends the same request every time. The gateway decides which model handles it. This is infrastructure-level control — no agent code changes needed.

## The Big Picture

```
                    ┌─────────────────────────────┐
                    │                             │──→ GPT-4o-mini (80%)
Agent ──→ /openai/* │      Agentgateway           │
                    │      (ai-gateway)           │──→ GPT-4o (20%)
                    │                             │
                    │   Weighted Load Balancing    │
                    └─────────────────────────────┘
```

One route. Multiple models. Automatic traffic distribution. Adjust the weights anytime — no agent restarts, no code changes.
