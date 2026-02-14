---
slug: credential-leak
id: ""
type: challenge
title: "Credential Leak Prevention — Keep Secrets Out of Responses"
teaser: Prevent API keys and secrets from leaking through LLM responses.
tabs:
  - title: Terminal
    type: terminal
    hostname: workstation
  - title: Code Editor
    type: code
    hostname: workstation
    path: /root
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

> **Important:** Credential leak prevention is an **AgentGateway Enterprise** feature. We'll create the policy and simulate the detection behavior.

## Step 1: Understand the Policy

Credential leak prevention scans **LLM responses** (not requests) for secret patterns:

```yaml
apiVersion: agentgateway.solo.io/v1alpha1
kind: AgentGatewayPolicy
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
kind: AgentGatewayPolicy
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

## Step 3: Simulate Credential Detection

Create a test that shows what credential leak prevention catches in responses:

```bash
cat <<'SCRIPT' > /root/policies/test-credential-leak.sh
#!/bin/bash
echo "=== Credential Leak Detection Test ==="
echo ""
echo "Scanning simulated LLM responses for leaked credentials..."
echo ""

# Simulated LLM responses that contain credentials
declare -a RESPONSES=(
  "API Key|Here's your config: OPENAI_API_KEY=sk-proj-abc123def456ghi789jkl012mno345pqr678stu901vwx234|sk-proj-[a-zA-Z0-9]+"
  "AWS Key|Your AWS access key is AKIAIOSFODNN7EXAMPLE and secret is wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY|AKIA[0-9A-Z]{16}"
  "JWT Token|Use this token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozjgNryP4J3jVmNHl0w5N_XgL0n3I9PlFUP0THsR8U|eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+"
  "DB Connection|Connect to: postgresql://admin:SuperSecret123!@db.prod.internal:5432/myapp|://[^:]+:[^@]+@"
  "Clean response|The capital of France is Paris. It has a population of about 2 million.|NO_MATCH"
)

LEAKED=0
CLEAN=0

for resp in "${RESPONSES[@]}"; do
  IFS='|' read -r desc content pattern <<< "$resp"
  
  if [ "$pattern" = "NO_MATCH" ]; then
    echo "✅ CLEAN: $desc"
    echo "   Response: \"${content:0:80}...\""
    ((CLEAN++))
  elif echo "$content" | grep -qP "$pattern"; then
    echo "🚫 CREDENTIAL DETECTED: $desc"
    echo "   Response: \"${content:0:80}...\""
    echo "   → Would be REDACTED before reaching the client"
    ((LEAKED++))
  fi
  echo ""
done

echo "=== Results ==="
echo "🚫 Credentials caught: $LEAKED"
echo "✅ Clean responses: $CLEAN"
echo ""
echo "With Enterprise credential leak prevention, detected secrets"
echo "are automatically redacted in the response body."
SCRIPT
chmod +x /root/policies/test-credential-leak.sh
```

Run it:

```bash
/root/policies/test-credential-leak.sh
```

## Step 4: Show the Current Gap

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
- AgentGateway Enterprise scans **responses** for API keys, AWS keys, JWTs, and more
- Detected credentials are **redacted** before reaching the client
- This is the response-side complement to request-side PII protection

**Next up:** Rate Limiting — controlling AI spend before it controls you.
