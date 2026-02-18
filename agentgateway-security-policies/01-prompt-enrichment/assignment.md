---
slug: prompt-enrichment
id: prompt-enrichment-01
type: challenge
title: "Prompt Enrichment 🎭"
teaser: Add system prompts to every LLM request at the gateway level — no app changes needed.
notes:
- type: text
  contents: "# Prompt Enrichment

**Challenge**: Inject consistent system prompts (policies, guidelines, context) into every LLM request — centrally at the gateway.

**Why?**
- Enforce consistent behavior across all agents/apps
- Add corporate policies, safety guardrails, formatting rules
- No code changes — works for any client (curl, LangChain, OpenAI SDK)

**AgentGateway supports:**
- `prepend` — add before user messages
- `append` — add after user messages
- `role: system` or `role: assistant`

tabs:
- id: terminal-prompt-enrichment
  title: Terminal
  type: terminal
  hostname: server
- id: editor-prompt-enrichment
  title: Editor
  type: code
  hostname: server
  path: /root
- id: solo-ui-prompt-enrichment
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: 1
---

## Step 1: Verify Environment

```bash
kubectl get gateway ai-gateway -n default
source /root/.bashrc
echo \"Gateway IP: \$GATEWAY_IP\"
```

## Step 2: Create AgentgatewayBackend for Mock LLM

```bash
cat > /root/openai-backend.yaml << 'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mock-llm
  namespace: default
spec:
  http:
    target:
      host: mock-llm.default.svc.cluster.local
      port: 8080
EOF

kubectl apply -f /root/openai-backend.yaml
```

## Step 3: Create HTTPRoute

```bash
cat > /root/llm-route.yaml << 'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: llm-route
  namespace: default
spec:
  parentRefs:
  - name: ai-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /openai
    backendRefs:
    - name: mock-llm
      group: agentgateway.dev
      kind: AgentgatewayBackend
EOF

kubectl apply -f /root/llm-route.yaml
```

## Step 4: Test Baseline (No Enrichment)

```bash
curl -s http://\$GATEWAY_IP:8080/openai/v1/chat/completions \\
  -H \"Content-Type: application/json\" \\
  -d '{\"model\":\"gpt-4\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}]}' | jq .
```

**Mock LLM echoes**: `"You said: Hello"`

## Step 5: Add Prompt Enrichment Policy

Create an `EnterpriseAgentgatewayPolicy` that **prepends a system prompt**:

```bash
cat > /root/prompt-enrichment.yaml << 'EOF'
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: prompt-enrichment
  namespace: default
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: llm-route
  backend:
    ai:
      promptEnrichment:
        prepend:
        - role: system
          content: |
            You are Acme Corp's AI assistant.
            Always respond professionally and concisely.
            Never mention pricing, competitors, or internal processes.
EOF

kubectl apply -f /root/prompt-enrichment.yaml
```

## Step 6: Test Enrichment

Send the same request:

```bash
curl -s http://\$GATEWAY_IP:8080/openai/v1/chat/completions \\
  -H \"Content-Type: application/json\" \\
  -d '{\"model\":\"gpt-4\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}]}' | jq .
```

**Now the LLM sees**: System prompt + your message, even though you sent none!

**Check Solo UI**: Switch to Solo UI tab → see the enriched prompt in traces.

## ✅ Success Criteria

```bash
# Policy applied
kubectl get enterpriseagentgatewaypolicy prompt-enrichment -n default

# Enrichment visible in response
curl -s http://\$GATEWAY_IP:8080/openai/v1/chat/completions \\
  -H \"Content-Type: application/json\" \\
  -d '{\"model\":\"gpt-4\",\"messages\":[{\"role\":\"user\",\"content\":\"What is AgentGateway?\"}]}' | jq -r '.choices[0].message.content' | grep -i "Acme Corp"
```

**Expected**: Response mentions "Acme Corp" (from injected system prompt).

**Next**: Prompt Guards — block harmful requests.
