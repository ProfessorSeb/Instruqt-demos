---
slug: mcp-proxying
id: uoeonaturrmp
type: challenge
title: Route Multiple Models Through AgentGateway
teaser: One gateway, many models. Add a second AI backend and see how AgentGateway
  centralises routing across your entire model fleet.
notes:
- type: text
  contents: "# \U0001F9E0 One Gateway, Many Models\n\nYou've wired up `gpt-4o-mini`
    through AgentGateway. But in production, you'll\nrarely stop at one model.\n\nDifferent
    tasks call for different models:\n\n| Task | Model |\n|---|---|\n| Fast, cheap
    Q&A | `gpt-4o-mini` |\n| Complex reasoning | `gpt-4o` |\n| Code generation |
    `gpt-4o` |\n| Simple classification | `gpt-4o-mini` |\n\nWithout a gateway, each
    model requires a separate integration — separate\nAPI keys, separate endpoints,
    separate config in every app.\n\n**With AgentGateway**, you register all your
    models once. Apps just call\nthe gateway and specify which model they want.\n\n```\nApp
    \ ──▶  AgentGateway  ──▶  gpt-4o-mini  (fast, cheap)\n                       ──▶
    \ gpt-4o       (smart, powerful)\n                       ──▶  claude-3   (future)\n
    \                      ──▶  gemini     (future)\n```\n\nOne API key vault. One
    audit log. One kill switch for everything."
tabs:
- id: if4tljwlsspx
  title: Terminal
  type: terminal
  hostname: server
- id: 2kjc4xxnsdkm
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: intermediate
timelimit: 900
enhanced_loading: null
---

# Route Multiple Models Through AgentGateway

## Why Multiple Backends?

You already have `gpt-4o-mini` registered as a backend. Now let's add `gpt-4o` — the
more capable (and more expensive) sibling.

This is a common production pattern:
- **Default traffic**: `gpt-4o-mini` (low cost, fast)
- **Complex requests**: `gpt-4o` (higher quality, higher cost)

AgentGateway manages both from a single control plane. No app-level config changes needed.

## Add the gpt-4o Backend

Create a second `AgentgatewayBackend` pointing to `gpt-4o`:

```bash
cat > /root/openai-smart-backend.yaml << 'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-smart
  namespace: agentgateway-system
spec:
  ai:
    openAI:
      authHeaderRef:
        name: openai-api-key
        key: Authorization
      model: gpt-4o
EOF

kubectl apply -f /root/openai-smart-backend.yaml
```

## Verify Both Backends Are Registered

```bash
kubectl get agentgatewaybackend -n agentgateway-system
```

You should see both `openai` (gpt-4o-mini) and `openai-smart` (gpt-4o) listed.

## Test the Smart Backend Directly

Create a second route that points to `openai-smart`:

```bash
cat > /root/openai-smart-route.yaml << 'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-smart
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
  rules:
  - matches:
    - headers:
      - name: X-Model-Tier
        value: smart
    backendRefs:
    - group: agentgateway.dev
      kind: AgentgatewayBackend
      name: openai-smart
      namespace: agentgateway-system
EOF

kubectl apply -f /root/openai-smart-route.yaml
```

Test it by sending a request:

```bash
curl -s -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "X-Model-Tier: smart" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"In one word: name the most powerful OpenAI model."}]}' \
  | jq '.choices[0].message.content'
```

## Your Multi-Model Registry

List everything AgentGateway is managing:

```bash
# All AI backends
kubectl get agentgatewaybackend -n agentgateway-system

# All active routes
kubectl get httproute -n agentgateway-system

# The gateway itself
kubectl get gateway -n agentgateway-system
```

> 💡 **One API key vault.** Both backends share the same `openai-api-key` Secret.
> Add 10 more models — they all use the same Secret, the same audit log, the same kill switch.
