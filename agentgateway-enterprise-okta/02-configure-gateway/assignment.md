---
slug: configure-gateway-with-tracing
id: configure-gateway-with-tracing
type: challenge
title: Configure the Gateway with Tracing
teaser: Create an Enterprise Gateway with full observability — tracing, structured logging, and shared extensions.
notes:
- type: text
  contents: |-
    # Gateway Configuration 🔧

    Enterprise AgentGateway uses **EnterpriseAgentgatewayParameters** to configure the data plane with:
    - **Tracing** — OpenTelemetry traces to Tempo with GenAI semantic conventions
    - **Structured logging** — JSON logs with request/response bodies and JWT claims
    - **Shared extensions** — ext-auth, rate-limiter, and ext-cache sidecars

    This is the foundation everything else builds on.
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

# Configure the Gateway with Tracing

In this challenge, you'll create two resources:
1. **EnterpriseAgentgatewayParameters** — configures tracing, logging, and extensions
2. **Gateway** — the actual gateway instance that references those parameters

## Step 1: Create EnterpriseAgentgatewayParameters

This resource configures the AgentGateway data plane with full observability and shared extensions.

Create the file `/root/agentgateway-params.yaml`:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: agentgateway-params
  namespace: enterprise-agentgateway
spec:
  sharedExtensions:
    extauth:
      enabled: true
      deployment:
        spec:
          replicas: 1
    ratelimiter:
      enabled: true
      deployment:
        spec:
          replicas: 1
    extCache:
      enabled: true
      deployment:
        spec:
          replicas: 1
  logging:
    level: info
  rawConfig:
    config:
      logging:
        fields:
          add:
            jwt: 'jwt'
            request.body: json(request.body)
            response.body: json(response.body)
        format: json
      tracing:
        otlpProtocol: grpc
        otlpEndpoint: http://tempo-distributor.monitoring.svc.cluster.local:4317
        randomSampling: 'true'
        fields:
          add:
            gen_ai.operation.name: '"chat"'
            gen_ai.system: "llm.provider"
            gen_ai.prompt: 'llm.prompt'
            gen_ai.completion: 'llm.completion.map(c, {"role":"assistant", "content": c})'
            gen_ai.request.model: "llm.requestModel"
            gen_ai.response.model: "llm.responseModel"
            gen_ai.usage.completion_tokens: "llm.outputTokens"
            gen_ai.usage.prompt_tokens: "llm.inputTokens"
            jwt: 'jwt'
  deployment:
    spec:
      replicas: 1
```

Apply it:

```bash
kubectl apply -f /root/agentgateway-params.yaml
```

**What this does:**
- **sharedExtensions** — deploys ext-auth, rate-limiter, and ext-cache pods alongside the proxy
- **tracing** — sends OpenTelemetry traces to Tempo with GenAI semantic conventions (model, tokens, prompt/completion)
- **logging** — structured JSON logs with request bodies and JWT claims
- **deployment.replicas: 1** — single proxy replica for the workshop

## Step 2: Create the Gateway

Now create the Gateway that references these parameters. Create `/root/gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway
  namespace: enterprise-agentgateway
spec:
  gatewayClassName: enterprise-agentgateway
  infrastructure:
    parametersRef:
      name: agentgateway-params
      group: enterpriseagentgateway.solo.io
      kind: EnterpriseAgentgatewayParameters
  listeners:
  - name: http
    port: 8080
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
```

Apply it:

```bash
kubectl apply -f /root/gateway.yaml
```

## Step 3: Verify Everything is Running

Wait for all pods to come up:

```bash
kubectl get pods -n enterprise-agentgateway -w
```

You should see **5 pods**: the controller, the agentgateway proxy, ext-auth, ext-cache, and rate-limiter.

Press `Ctrl+C` once they're all Running, then click **Check**.
