---
slug: llm-routing
id: xnmpczhy75k7
type: challenge
title: LLM Routing & Observability
teaser: Route your first LLM request through the gateway and explore metrics and traces.
notes:
- type: text
  contents: "# \U0001F310 LLM Routing\n\nThe core value of an AI gateway: **one entry
    point for all your LLM traffic**.\n\nInstead of every agent calling OpenAI directly
    with its own API key, they all go through the gateway. You get:\n\n- Centralized
    credential management\n- Per-model metrics and token tracking\n- Distributed traces
    showing prompt → response\n- A single place to add policies later\n\nLet's route
    traffic to OpenAI and see the observability.\n"
tabs:
- id: rfoqyrehs9zo
  title: Terminal
  type: terminal
  hostname: server
- id: qjxngsqzpfjd
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: ptw0xxk0ldit
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: ylwlqnwkfj7c
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# LLM Routing & Observability

Time to route your first LLM request through the gateway. You'll create an OpenAI backend and HTTPRoute, send a request, and see it show up in Grafana.

## Step 1: Create the OpenAI Backend and Route

The OpenAI API key secret was already created during track setup. Now create the backend and route:

```bash
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

## Step 2: Send a Request Through the Gateway

```bash
source /root/.bashrc

curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "What is an AI gateway and why do enterprises need one? Answer in 2 sentences."
      }
    ]
  }' | jq .
```

You should get a response from OpenAI — but now it went through your gateway, which means metrics and traces were captured.

## Step 3: View Access Logs

AgentGateway logs every request with LLM-specific metadata:

```bash
kubectl logs deploy/agentgateway -n agentgateway-system --tail 5
```

Notice the log includes:
- `gen_ai.request.model` and `gen_ai.response.model`
- `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens`
- `request.body` and `response.body` for full prompt visibility
- Trace ID for correlation with distributed traces

## Step 4: View Metrics Endpoint

AgentGateway exposes Prometheus-compatible metrics:

```bash
kubectl port-forward -n agentgateway-system deployment/agentgateway 15020:15020 &
sleep 2
curl -s http://localhost:15020/metrics | grep -E "agentgateway_(request|token)" | head -20
kill %1 2>/dev/null
```

## Step 5: Explore Grafana

Switch to the **Grafana** tab and log in:
- Username: `admin`
- Password: `admin`

Navigate to **Home > Explore** and select **Tempo** to see distributed traces for your request. You'll see LLM-specific spans with prompt content, token counts, and response data.

## Step 6: Explore the Solo UI

Switch to the **Solo UI** tab. The Solo Enterprise Management UI also collects traces from the gateway. Navigate to the **Tracing** section to see your LLM requests with full execution details. This gives you a second view of the same traffic — useful for teams that prefer a dedicated AI gateway dashboard over Grafana.

## Step 7: Send a Few More Requests

Generate some traffic for better metrics:

```bash
for i in {1..3}; do
  curl -s "$GATEWAY_IP:8080/openai" \
    -H "content-type: application/json" \
    -d "{
      \"model\": \"gpt-4o-mini\",
      \"messages\": [{\"role\": \"user\", \"content\": \"Count to $i\"}]
    }" | jq -r '.choices[0].message.content' &
done
wait
```

## ✅ What You've Learned

- `AgentgatewayBackend` defines where to route LLM traffic (provider + credentials)
- `HTTPRoute` maps URL paths to backends (standard Gateway API)
- Every request gets: structured logs, Prometheus metrics, OTLP traces
- Token usage is tracked automatically per model
- Traces are available in both Grafana (Tempo) and the Solo UI

**Next up:** API Key Management — control who can access the gateway.
