---
slug: pii-protection
id: 3o8dxkorzzke
type: challenge
title: PII Protection — Stop Sensitive Data from Reaching LLMs
teaser: Redact personally identifiable information at the gateway before it reaches
  third-party LLMs.
notes:
- type: text
  contents: "# \U0001F6E1️ PII Protection\n\nSSNs, emails, credit cards — all flowing
    to third-party LLMs unfiltered.\n\n**In this challenge, you'll:**\n\n- Create
    an AgentGatewayPolicy for PII redaction\n- Understand REDACT vs BLOCK vs LOG actions\n-
    Simulate what PII protection looks like in practice\n- See how the gateway transforms
    requests transparently\n\n> ℹ️ PII protection is an **Enterprise** feature. You'll
    learn the concepts and create the policy YAML.\n"
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

The fix? **Redact PII at the gateway** — before it ever reaches the LLM.

## 🏢 OSS vs Enterprise

> **Important:** PII protection is an **AgentGateway Enterprise** feature. In this challenge, we'll create the policy YAML and understand how it works conceptually. The policy structure is real — it just requires an Enterprise license to enforce.
>
> We'll use our mock LLM to simulate the behavior so you can see the concept in action.

## Step 1: Understand the Policy

AgentGateway uses `AgentGatewayPolicy` resources to define security rules. Here's what a PII protection policy looks like:

```yaml
apiVersion: agentgateway.solo.io/v1alpha1
kind: AgentGatewayPolicy
metadata:
  name: pii-protection
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-route
  default:
    security:
      pii:
        action: REDACT          # REDACT, BLOCK, or LOG
        detectors:
          - type: SSN            # Social Security Numbers
          - type: EMAIL          # Email addresses
          - type: PHONE          # Phone numbers
          - type: CREDIT_CARD    # Credit card numbers
          - type: ADDRESS        # Physical addresses
```

Key concepts:
- **`targetRefs`** — attaches the policy to a specific route (or gateway)
- **`action: REDACT`** — replaces detected PII with placeholder tokens (e.g., `[SSN_REDACTED]`)
- **`action: BLOCK`** — rejects the entire request if PII is detected
- **`action: LOG`** — allows the request but logs the PII detection
- **`detectors`** — which types of PII to look for

## Step 2: Create the PII Protection Policy

Create the policy file:

```bash
cat <<EOF > /root/policies/pii-protection.yaml
apiVersion: agentgateway.solo.io/v1alpha1
kind: AgentGatewayPolicy
metadata:
  name: pii-protection
  namespace: default
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-route
  default:
    security:
      pii:
        action: REDACT
        detectors:
          - type: SSN
          - type: EMAIL
          - type: PHONE
          - type: CREDIT_CARD
EOF
```

Apply it (this will create the CRD but won't be enforced without Enterprise):

```bash
kubectl apply -f /root/policies/pii-protection.yaml 2>/dev/null || echo "Note: AgentGatewayPolicy CRD requires Enterprise. Policy file created for reference."
```

## Step 3: Simulate PII Redaction

To see what PII protection **would** do, let's simulate it. With Enterprise, the gateway would automatically transform this request:

**Before (what the agent sends):**
```json
{
  "content": "Summarize: John Smith, SSN 123-45-6789, email john@example.com"
}
```

**After (what the LLM receives):**
```json
{
  "content": "Summarize: [NAME_REDACTED], SSN [SSN_REDACTED], email [EMAIL_REDACTED]"
}
```

Let's demonstrate this by sending a PII-laden request and examining what would happen:

```bash
# This is what currently goes through (no protection):
source /root/.bashrc
curl -s http://$GATEWAY_IP:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {
        "role": "user",
        "content": "Process this customer: Jane Doe, SSN 987-65-4321, email jane.doe@company.com, phone 415-555-0199, card 4532-1234-5678-9012"
      }
    ]
  }' | jq .
```

Notice all the PII passes through. With Enterprise PII protection enabled, the LLM would instead receive:

```
Process this customer: [NAME], SSN [SSN_REDACTED], email [EMAIL_REDACTED], phone [PHONE_REDACTED], card [CREDIT_CARD_REDACTED]
```

## Step 4: Create a Test Script

Create a script that demonstrates PII detection (what Enterprise would catch):

```bash
cat <<'SCRIPT' > /root/policies/test-pii.sh
#!/bin/bash
# PII Detection Test — shows what AgentGateway Enterprise would catch

echo "=== PII Detection Test ==="
echo ""

INPUT="Process this customer: Jane Doe, SSN 987-65-4321, email jane.doe@company.com, phone 415-555-0199, card 4532-1234-5678-9012"

echo "📥 Original input:"
echo "   $INPUT"
echo ""

# Simulate PII detection
echo "🔍 PII Detected:"
echo "$INPUT" | grep -oP '\d{3}-\d{2}-\d{4}' && echo "   ↳ SSN found — would be REDACTED"
echo "$INPUT" | grep -oP '[\w.]+@[\w.]+\.\w+' && echo "   ↳ Email found — would be REDACTED"
echo "$INPUT" | grep -oP '\d{3}-\d{3}-\d{4}' && echo "   ↳ Phone found — would be REDACTED"
echo "$INPUT" | grep -oP '\d{4}-\d{4}-\d{4}-\d{4}' && echo "   ↳ Credit card found — would be REDACTED"
echo ""

# Show what would be sent to LLM
REDACTED=$(echo "$INPUT" | \
  sed 's/[0-9]\{3\}-[0-9]\{2\}-[0-9]\{4\}/[SSN_REDACTED]/g' | \
  sed 's/[a-zA-Z0-9.]*@[a-zA-Z0-9.]*\.[a-zA-Z]*/[EMAIL_REDACTED]/g' | \
  sed 's/[0-9]\{3\}-[0-9]\{3\}-[0-9]\{4\}/[PHONE_REDACTED]/g' | \
  sed 's/[0-9]\{4\}-[0-9]\{4\}-[0-9]\{4\}-[0-9]\{4\}/[CREDIT_CARD_REDACTED]/g')

echo "📤 What the LLM would receive (with Enterprise PII protection):"
echo "   $REDACTED"
echo ""
echo "✅ PII protection test complete"
SCRIPT
chmod +x /root/policies/test-pii.sh
```

Run the test:

```bash
/root/policies/test-pii.sh
```

## Step 5: Verify Your Work

Confirm the policy file exists and is valid YAML:

```bash
cat /root/policies/pii-protection.yaml
```

## ✅ What You've Learned

- PII routinely leaks through AI agent traffic to third-party LLMs
- AgentGateway Enterprise's PII protection **redacts sensitive data at the gateway layer**
- Policies use `AgentGatewayPolicy` CRDs attached to specific routes
- You can choose to **REDACT**, **BLOCK**, or **LOG** PII detections
- No application code changes needed — the gateway handles it transparently

**Next up:** Prompt Injection Guard — blocking jailbreak attempts.
