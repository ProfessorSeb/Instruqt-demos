---
slug: credential-leak
id: toyfp52ggdsd
type: challenge
title: Credential Leak Prevention — Keep Secrets Out of Responses
teaser: Prevent API keys and secrets from leaking through LLM responses.
notes:
- type: text
  contents: "# \U0001F511 Credential Leak Prevention\n\nWe've protected what goes
    **in** to the LLM. Now let's protect what comes **out**.\n\n**In this challenge,
    you'll:**\n\n- Create a response-side prompt guard policy\n- Scan for API keys
    and secrets in LLM responses\n- See the gateway mask credentials before they reach
    users\n"
tabs:
- id: gbokxxmdqy6y
  title: Terminal
  type: terminal
  hostname: server
- id: gm01awwcf18v
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: guuydbt9cprh
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# Credential Leak Prevention

We've been focused on what goes **into** the LLM. But what comes **out** is equally dangerous.

## 🎯 The Problem

LLMs can leak credentials in their responses in several ways:

**Echo-back attacks:**
A user sends: *"Here's my config: API_KEY=sk-abc123..."*
The LLM helpfully includes the key in its response: *"I see your API key is sk-abc123..."*

**Context leakage in RAG:**
Your RAG pipeline retrieves documents containing database passwords. The LLM includes them in its response.

**Training data leakage:**
LLMs occasionally output API keys, passwords, or tokens they memorized during training.

**Agent tool responses:**
An agent calls a tool that returns credentials, and the LLM passes them to the user.

In all cases, secrets end up in API responses — logged, cached, and potentially exposed.

## Step 1: Understand the Policy

Credential leak prevention uses **response-side prompt guards** to scan LLM responses for secret patterns and mask them:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: credential-leak-prevention
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: llm-route
  backend:
    ai:
      promptGuard:
        response:
        - regex:
            action: Mask
            matches:
            - "sk-[a-zA-Z0-9]{20,}"
            - "ghp_[a-zA-Z0-9]{36}"
            - "AKIA[0-9A-Z]{16}"
            - "xoxb-[a-zA-Z0-9-]+"
```

This is **response-side scanning** — it inspects what the LLM sends back, not what goes in. The `Mask` action replaces matched patterns with `X` characters.

## Step 2: Create the Policy

```bash
cat <<EOF > /root/policies/credential-leak.yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: credential-leak-prevention
  namespace: default
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: llm-route
  backend:
    ai:
      promptGuard:
        response:
        - regex:
            action: Mask
            matches:
            - "sk-[a-zA-Z0-9]{20,}"
            - "ghp_[a-zA-Z0-9]{36}"
            - "AKIA[0-9A-Z]{16}"
            - "xoxb-[a-zA-Z0-9-]+"
EOF

kubectl apply -f /root/policies/credential-leak.yaml
```

Verify:

```bash
kubectl get enterpriseagentgatewaypolicies -n default
```

## Step 3: Test Credential Masking

Send a request that would cause credential echo-back:

```bash
source /root/.bashrc

curl -s http://$GATEWAY_IP:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {
        "role": "user",
        "content": "Review my config: OPENAI_KEY=sk-proj-abc123xyz456789012345"
      }
    ]
  }' | jq .
```

With the policy active, any `sk-` prefixed API keys in the response will be masked with `X` characters before reaching the user.

Try with other credential patterns:

```bash
curl -s http://$GATEWAY_IP:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {
        "role": "user",
        "content": "Check this AWS key: AKIAIOSFODNN7EXAMPLE"
      }
    ]
  }' | jq .
```

The gateway masks credentials in responses regardless of how they got there — echo-back, RAG context, or training data leakage.

## ✅ What You've Learned

- LLM responses can leak credentials through echo-back, RAG context, and training data
- Enterprise Agentgateway's response-side prompt guards scan and **mask secrets in responses**
- Custom regex patterns catch API keys, AWS keys, GitHub tokens, and more
- This is the response-side complement to request-side PII protection

**Next up:** Rate Limiting — controlling AI spend before it controls you.
