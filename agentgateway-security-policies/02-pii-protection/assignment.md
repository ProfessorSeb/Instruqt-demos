---
slug: pii-protection
id: 3o8dxkorzzke
type: challenge
title: PII Protection — Stop Sensitive Data from Reaching LLMs
teaser: Block personally identifiable information at the gateway before it reaches
  third-party LLMs.
notes:
- type: text
  contents: "# \U0001F6E1️ PII Protection\n\nSSNs, emails, credit cards — all flowing
    to third-party LLMs unfiltered.\n\n**In this challenge, you'll:**\n\n- Create
    an EnterpriseAgentgatewayPolicy with prompt guards\n- Block requests containing
    PII using built-in regex matchers\n- Mask credit card numbers in LLM responses\n-
    See the gateway enforce protection in real time\n"
tabs:
- id: kffafiugzg3y
  title: Terminal
  type: terminal
  hostname: server
- id: 38syuxqfqttb
  title: Code Editor
  type: code
  hostname: server
  path: /root
- title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# PII Protection

In the last challenge, you sent a customer record with an SSN, email, and credit card number straight through to an LLM. In production, that's a **data breach waiting to happen**.

## 🎯 The Problem

Every time an AI agent processes user data, there's a risk of PII leaking to third-party LLM providers:

- **Customer support agents** — users paste account details, SSNs, medical info
- **Code assistants** — developers include API keys, database credentials, customer data in prompts
- **Data analysis agents** — raw datasets with emails, phone numbers, addresses

Third-party LLM providers may log, train on, or store this data. Even if they promise not to — **the data still leaves your network**.

The fix? **Block PII at the gateway** — before it ever reaches the LLM.

## Step 1: Understand the Policy

Enterprise Agentgateway uses `EnterpriseAgentgatewayPolicy` resources with **prompt guards** to detect and block PII. The policy uses built-in regex matchers for common PII types:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: pii-protection
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: llm-route
  backend:
    ai:
      promptGuard:
        request:
        - response:
            message: "🚫 Rejected - PII detected in request"
          regex:
            action: Reject
            builtins:
            - CreditCard
            - Ssn
        response:
        - regex:
            builtins:
            - CreditCard
            action: Mask
```

Key concepts:
- **`targetRefs`** — attaches the policy to a specific HTTPRoute
- **`promptGuard.request`** — scans incoming requests for PII patterns
- **`action: Reject`** — blocks the request entirely if PII is detected
- **`action: Mask`** — replaces detected PII in responses with `X` characters
- **`builtins`** — pre-built regex patterns: `CreditCard`, `Ssn`, `PhoneNumber`, `Email`, `CaSin`

## Step 2: Create the PII Protection Policy

Create and apply the policy:

```bash
cat <<EOF > /root/policies/pii-protection.yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: pii-protection
  namespace: default
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: llm-route
  backend:
    ai:
      promptGuard:
        request:
        - response:
            message: "🚫 Rejected - PII detected in request"
          regex:
            action: Reject
            builtins:
            - CreditCard
            - Ssn
        response:
        - regex:
            builtins:
            - CreditCard
            action: Mask
EOF

kubectl apply -f /root/policies/pii-protection.yaml
```

Verify the policy is accepted:

```bash
kubectl get enterpriseagentgatewaypolicies -n default
```

## Step 3: Test PII Blocking — Request Side

Now try sending the same PII-laden request from Challenge 1:

```bash
source /root/.bashrc
curl -v http://$GATEWAY_IP:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {
        "role": "user",
        "content": "Process this customer: Jane Doe, SSN 987-65-4321, card 4532-1234-5678-9012"
      }
    ]
  }'
```

💥 **The request is rejected!** You should see a `403` response with the message "🚫 Rejected - PII detected in request". The SSN and credit card patterns triggered the built-in matchers.

Now try a clean request without PII:

```bash
curl -s http://$GATEWAY_IP:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {
        "role": "user",
        "content": "What is the weather like today?"
      }
    ]
  }' | jq .
```

✅ This request passes through — no PII detected.

## Step 4: Test PII Masking — Response Side

The policy also masks credit card numbers in LLM responses. If the LLM returns a credit card number, the gateway replaces it before it reaches the user:

```bash
curl -s http://$GATEWAY_IP:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {
        "role": "user",
        "content": "What type of number is 5105105105105100?"
      }
    ]
  }' | jq .
```

If the LLM response contains a credit card number, the gateway masks it automatically.

## Step 5: Verify the Policy

```bash
kubectl get enterpriseagentgatewaypolicies -n default -o yaml
```

## ✅ What You've Learned

- PII routinely leaks through AI agent traffic to third-party LLMs
- Enterprise Agentgateway's prompt guards **reject requests containing sensitive data**
- Built-in regex matchers detect SSNs, credit cards, emails, and phone numbers
- Response-side masking catches PII in LLM outputs
- No application code changes needed — the gateway handles it transparently

**Next up:** Prompt Injection Guard — blocking jailbreak attempts.
