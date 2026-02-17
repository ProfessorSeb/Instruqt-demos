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
    you'll:**\n\n- Create a credential leak detection policy\n- Scan for API keys,
    AWS keys, JWTs, and passwords in responses\n- Understand response-side scanning
    vs request-side filtering\n\n> ℹ️ Credential leak prevention is an **Enterprise**
    feature.\n"
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

## 🏢 OSS vs Enterprise

> **Important:** Credential leak prevention is an **Agentgateway Enterprise** feature. We'll create the policy and simulate the detection behavior.

## Step 1: Understand the Policy

Credential leak prevention scans **LLM responses** (not requests) for secret patterns:

```yaml
apiVersion: agentgateway.solo.io/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: credential-leak-prevention
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-route
  default:
    security:
      credentialLeak:
        action: REDACT          # REDACT or BLOCK
        detectors:
          - type: API_KEY        # Generic API key patterns
          - type: AWS_KEY        # AWS access key IDs
          - type: PRIVATE_KEY    # RSA/SSH private keys
          - type: JWT            # JSON Web Tokens
          - type: PASSWORD_IN_URL # Passwords in connection strings
```

This is **response-side scanning** — it inspects what the LLM sends back, not what goes in.

## Step 2: Create the Policy

```bash
cat <<EOF > /root/policies/credential-leak.yaml
apiVersion: agentgateway.solo.io/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: credential-leak-prevention
  namespace: default
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-route
  default:
    security:
      credentialLeak:
        action: REDACT
        detectors:
          - type: API_KEY
          - type: AWS_KEY
          - type: PRIVATE_KEY
          - type: JWT
          - type: PASSWORD_IN_URL
EOF
```

Apply it:

```bash
kubectl apply -f /root/policies/credential-leak.yaml 2>/dev/null || echo "Note: Credential leak prevention requires Enterprise. Policy file created for reference."
```

## Step 3: Show the Current Gap

Send a request that would cause credential echo-back through our unprotected gateway:

```bash
source /root/.bashrc

curl -s http://$GATEWAY_IP:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {
        "role": "user",
        "content": "Review my config: DATABASE_URL=postgresql://admin:P@ssw0rd123@db.prod.internal:5432/app OPENAI_KEY=sk-proj-abc123xyz"
      }
    ]
  }' | jq -r '.choices[0].message.content'
```

The credentials pass through in both directions. With Enterprise, the response would have them redacted.

## ✅ What You've Learned

- LLM responses can leak credentials through echo-back, RAG context, and training data
- Agentgateway Enterprise scans **responses** for API keys, AWS keys, JWTs, and more
- Detected credentials are **redacted** before reaching the client
- This is the response-side complement to request-side PII protection

**Next up:** Rate Limiting — controlling AI spend before it controls you.
