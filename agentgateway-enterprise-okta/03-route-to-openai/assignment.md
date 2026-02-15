---
slug: route-llm-traffic-to-openai
id: route-llm-traffic-to-openai
type: challenge
title: Route Real LLM Traffic to OpenAI
teaser: Create an OpenAI backend and route, send real requests, and see traces in Grafana.
notes:
- type: text
  contents: |-
    # Real LLM Traffic 🤖

    Now that your gateway is running with tracing enabled, let's route actual LLM requests through it.

    You'll create an **AgentgatewayBackend** pointing to OpenAI and an **HTTPRoute** to expose it, then watch the traces flow into Grafana/Tempo.
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
- title: Grafana
  type: service
  hostname: server
  port: 3000
difficulty: basic
timelimit: 900
---

# Route Real LLM Traffic to OpenAI

## Step 1: Create the OpenAI Backend and Route

Create `/root/openai.yaml` with both the backend and route:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-backend
  namespace: enterprise-agentgateway
spec:
  ai:
    provider:
      openai: {}
  policies:
    auth:
      secretRef:
        name: openai-secret
---
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
    - name: openai-backend
      group: agentgateway.dev
      kind: AgentgatewayBackend
    timeouts:
      request: "120s"
```

Apply it:

```bash
kubectl apply -f /root/openai.yaml
```

**What this does:**
- **AgentgatewayBackend** — points to OpenAI's API, using the `openai-secret` for authentication
- **HTTPRoute** — routes `/openai` requests through the gateway to the backend

## Step 2: Test with a Real Request

Send a request to OpenAI through the gateway:

```bash
curl -s "localhost:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What is AgentGateway in one sentence?"}]
  }' | jq
```

You should get a real response from OpenAI! 🎉

## Step 3: Check Traces in Grafana

Open the **Grafana** tab (credentials: `admin` / `solo-demo`).

1. Navigate to **Explore** (compass icon in the left sidebar)
2. Select **Tempo** as the data source
3. Click **Search** and hit **Run query**
4. You should see traces from your request with GenAI attributes like model name, token counts, and more

Click **Check** when you've verified the route works.
