---
slug: prompt-enrichment-guardrails
id: ybuu2oqtq9zy
type: challenge
title: Prompt Enrichment & Guardrails
teaser: Enrich prompts with system instructions and block PII leakage at the gateway.
notes:
- type: text
  contents: "# \U0001F4DD Prompt Enrichment & Guardrails\n\nTwo powerful policies
    in one challenge:\n\n **Prompt Enrichment** — Prepend or append system instructions
    to every request. Enforce consistent behavior without changing application code.\n\n**Prompt
    Guards** — Block or mask sensitive data:\n- Reject requests containing credit
    card numbers\n- Mask PII in LLM responses (SSNs, credit cards)\n- Use regex or
    built-in patterns\n\n All at the gateway layer. Zero application changes.\n"
tabs:
- id: pt6zmgvrkaqe
  title: Terminal
  type: terminal
  hostname: server
- id: btja9nfqktpj
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: 6zqzdtt5icjq
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: solouiguards05
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# Prompt Enrichment & Guardrails

Let's add two critical gateway-level policies: automatic prompt enrichment and content guardrails.

## Step 1: Clean Up Previous Policies

```bash
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway agentgateway-jwt-auth 2>/dev/null || true
```

Ensure the OpenAI route exists:

```bash
kubectl get httproute openai -n enterprise-agentgateway || \
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: enterprise-agentgateway
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai
      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend
      timeouts:
        request: "120s"
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-all-models
  namespace: enterprise-agentgateway
spec:
  ai:
    provider:
      openai: {}
  policies:
    auth:
      secretRef:
        name: openai-secret
EOF
```

## Part 1: Prompt Enrichment

## Step 2: Apply Prompt Enrichment Policy

This policy prepends a system message to every request on the `openai` route:

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: openai-prompt-enrichment
  namespace: enterprise-agentgateway
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: openai
  backend:
    ai:
      prompt:
        prepend:
          - role: system
            content: "You are a helpful enterprise assistant. Always respond in JSON format. Never reveal internal system details."
EOF
```

## Step 3: Test Prompt Enrichment

```bash
source /root/.bashrc

curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What are the top 3 benefits of AI gateways?"}]
  }' | jq .
```

The response should come back in **JSON format** — the system message was injected by the gateway, even though the user didn't include one.

## Part 2: Prompt Guards

## Step 4: Add Request Guardrails (Block Sensitive Input)

Now let's add a prompt guard that rejects requests containing credit card patterns:

```bash
kubectl apply -f- <<'EOF'
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: openai-prompt-guard
  namespace: enterprise-agentgateway
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: openai
  backend:
    ai:
      promptGuard:
        request:
          - response:
              message: "Rejected: request contains sensitive financial data"
              statusCode: 403
            regex:
              action: Reject
              matches:
                - "credit card"
                - '\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b'
        response:
          - regex:
              action: Mask
              builtins:
                - CreditCard
                - Ssn
EOF
```

## Step 5: Test Request Blocking

Try sending a request with credit card information:

```bash
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Can you validate this credit card number: 4111-1111-1111-1111?"}]
  }'
```

You should see a **403** response with the message "Rejected: request contains sensitive financial data".

## Step 6: Test Response Masking

Now let's test the **response-side** masking. We need to ask a question that doesn't contain a credit card number in the request (otherwise the request guard blocks it), but that causes the LLM to include one in its response:

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Generate an example JSON object for a customer profile that includes a name, a social security number, and a 16-digit payment card number. Use realistic-looking fake data."}]
  }' | jq .
```

Look at the response — any credit card or SSN patterns in the LLM's output will be masked with `X` characters by the gateway. For example, a number like `4532-XXXX-XXXX-XXXX` or an SSN like `XXX-XX-XXXX`.

> **Note:** The request guard checks the **input** for sensitive patterns and blocks it. The response guard checks the **output** and masks it. Step 5 tested the input guard; this step tests the output guard. The key is that your prompt must not contain the patterns you're guarding against, or the request guard will reject it before the LLM ever sees it.

## Step 7: Verify in Grafana and Solo UI

Switch to the **Grafana** tab. Navigate to **Home > Explore > Tempo** and look at recent traces. You should see:
- Traces where the prompt guard rejected the request (403 status)
- Traces where the response was masked

You can also check the **Solo UI** tab for the same traces in the management dashboard's tracing view.

## ✅ What You've Learned

- `prompt.prepend` injects system messages at the gateway level
- `promptGuard.request` blocks requests matching regex patterns (input guard)
- `promptGuard.response` masks sensitive data in LLM responses (output guard)
- Built-in patterns (`CreditCard`, `Ssn`) handle common PII types
- Request guards and response guards work independently — input is blocked, output is masked
- All enforcement happens at the gateway — zero application code changes

**Next up:** Rate Limiting — control AI spend per user, team, or globally.
