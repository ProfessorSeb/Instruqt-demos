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
- id: uqvv7xqd3spj
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

> **What's happening:** Removing the JWT auth policy from the previous challenge so we can focus on prompt-level policies. The gateway returns to unauthenticated mode temporarily.

```bash
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system agentgateway-jwt-auth 2>/dev/null || true
```

Ensure the OpenAI route exists:

```bash
kubectl get httproute openai -n agentgateway-system || \
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system
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
  namespace: agentgateway-system
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

> **What's happening:** This policy targets the `openai` HTTPRoute (not the Gateway itself — note the `targetRefs` difference from auth policies). The `prompt.prepend` field tells the gateway to inject a system message at the *beginning* of every request's message array before forwarding to OpenAI. The application sends `[{role: "user", content: "..."}]`, but OpenAI actually receives `[{role: "system", content: "You are a helpful enterprise assistant..."}, {role: "user", content: "..."}]`. This lets platform teams enforce consistent LLM behavior (JSON output, no internal detail leakage) across all applications without touching any application code.

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: openai-prompt-enrichment
  namespace: agentgateway-system
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

> **What's happening:** The user's prompt just asks about AI gateways — it says nothing about JSON format. But because the gateway prepended the system instruction, OpenAI's response will come back in JSON format. If you check the access logs, you'll see the `request.body` field contains both the injected system message and the original user message, confirming the enrichment happened at the gateway layer.

```bash
source /root/.bashrc

curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What are the top 3 benefits of AI gateways?"}]
  }' | jq .
```

## Part 2: Prompt Guards

## Step 4: Add Request Guardrails

> **What's happening:** This policy adds two layers of content protection. **Request guards** inspect incoming prompts *before* they reach the LLM: any request containing "credit card" (literal match) or a 16-digit card number pattern (regex) is rejected with a 403 and a custom error message — the request never reaches OpenAI, saving you money and preventing sensitive data from leaving your network. **Response guards** inspect the LLM's output *after* it comes back: any credit card or SSN patterns in the response are masked with `X` characters before the client sees them. The `builtins` shortcuts (`CreditCard`, `Ssn`) use pre-defined regex patterns so you don't have to write them yourself.

```bash
kubectl apply -f- <<'EOF'
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: openai-prompt-guard
  namespace: agentgateway-system
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

> **What's happening:** This prompt contains both the phrase "credit card" and an actual card number pattern (`4111-1111-1111-1111`). The gateway's request guard matches these patterns *before* the request ever reaches OpenAI, immediately returning a `403 Forbidden` with the custom message "Rejected: request contains sensitive financial data." This is important for compliance: the sensitive data never leaves your infrastructure, and you don't incur any LLM token costs for blocked requests.

```bash
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Can you validate this credit card number: 4111-1111-1111-1111?"}]
  }'
```

You should see a **403** response.

## Step 6: Test Response Masking

> **What's happening:** This prompt asks the LLM to *generate* fake PII data. The request itself doesn't contain any card numbers or SSNs, so it passes the request guard. OpenAI generates a response containing realistic-looking numbers. But before the response reaches the client, the gateway's response guard scans the output and replaces any patterns matching credit card or SSN formats with `X` characters (e.g., `123-45-6789` becomes `XXX-XX-XXXX`). This protects against data leakage even when the LLM generates sensitive-looking content.

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Generate an example JSON object for a customer profile that includes a name, a social security number, and a 16-digit payment card number. Use realistic-looking fake data."}]
  }' | jq .
```

Any credit card or SSN patterns in the LLM's output will be masked with `X` characters.

## Step 7: Verify in Grafana and Solo UI

> **What's happening:** Both the blocked requests (403s) and the masked responses show up in traces and logs. In Grafana/Tempo, you'll see traces where the guardrail fired — the span will show the rejection or masking action. This gives security teams visibility into how often guardrails are triggered, which helps tune the rules and demonstrate compliance.

Switch to the **Grafana** tab. Navigate to **Home > Explore > Tempo** and look at recent traces. You can also check the **Solo UI** tab for the same traces.

## ✅ What You've Learned

- `prompt.prepend` injects system messages at the gateway level
- `promptGuard.request` blocks requests matching regex patterns
- `promptGuard.response` masks sensitive data in LLM responses
- Built-in patterns (`CreditCard`, `Ssn`) handle common PII types
- All enforcement happens at the gateway — zero application code changes

**Next up:** Rate Limiting — control AI spend per user, team, or globally.
