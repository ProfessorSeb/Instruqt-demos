---
slug: configure-agentgateway
id: bage8ylcwgbl
type: challenge
title: Configure AgentGateway
teaser: Set up the AgentGateway proxy with tracing, OpenAI routing, and observability.
notes:
- type: text
  contents: "# \U0001F310 Configure AgentGateway\n\nNow that KAgent and the community
    agents are running, we need to configure the AgentGateway proxy — the data plane
    that handles all LLM and MCP traffic.\n\nYou'll:\n\n- Enable distributed tracing
    via the Solo telemetry collector\n- Create a Gateway proxy with tracing enabled\n-
    Route LLM traffic to OpenAI through the gateway\n\nThis gives you full visibility
    into every request agents make to LLMs.\n"
tabs:
- id: emmylkbwdsyf
  title: Terminal
  type: terminal
  hostname: server
- id: hnoc2bj73sgy
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: l1garesuvxy3
  title: KAgent UI
  type: service
  hostname: server
  port: 8080
difficulty: ""
enhanced_loading: null
---

# Configure AgentGateway

With KAgent and the community agents deployed, let's configure the AgentGateway proxy to route LLM traffic with full tracing and observability.

## Step 1: Configure Tracing Parameters

Create an `EnterpriseAgentgatewayParameters` resource that configures OTLP tracing to the Solo telemetry collector:

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: tracing
  namespace: agentgateway-system
spec:
  rawConfig:
    config:
      tracing:
        otlpEndpoint: grpc://solo-enterprise-telemetry-collector.kagent.svc.cluster.local:4317
        otlpProtocol: grpc
        randomSampling: true
EOF
```

This sends all traces from AgentGateway to the Solo Enterprise telemetry collector, which stores them in ClickHouse for the UI.

## Step 2: Create the Gateway Proxy

Deploy a Gateway resource that references the tracing parameters:

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: enterprise-agentgateway
  infrastructure:
    parametersRef:
      name: tracing
      group: enterpriseagentgateway.solo.io
      kind: EnterpriseAgentgatewayParameters
  listeners:
    - protocol: HTTP
      port: 8080
      name: http
      allowedRoutes:
        namespaces:
          from: All
EOF
```

Wait for the proxy pod to be ready:

```bash
kubectl -n agentgateway-system wait --for=condition=Available deployment -l gateway.networking.k8s.io/gateway-name=agentgateway-proxy --timeout=120s
```

## Step 3: Create the OpenAI Secret

Store your OpenAI API key as a Kubernetes secret:

```bash
kubectl apply -f- <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: openai-secret
  namespace: agentgateway-system
type: Opaque
stringData:
  Authorization: $OPENAI_API_KEY
EOF
```

## Step 4: Create the OpenAI Backend and Route

Deploy the `AgentgatewayBackend` and `HTTPRoute` to route LLM traffic to OpenAI:

```bash
kubectl apply -f- <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai:
        model: gpt-4o-mini
  policies:
    auth:
      secretRef:
        name: openai-secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway-system
  labels:
    route-type: llm-provider
spec:
  parentRefs:
    - name: agentgateway-proxy
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai
      backendRefs:
        - name: openai
          namespace: agentgateway-system
          group: agentgateway.dev
          kind: AgentgatewayBackend
EOF
```

## Step 5: Verify the Configuration

Check that all resources are created:

```bash
echo "=== Gateway ==="
kubectl get gateway -n agentgateway-system

echo "=== Backend ==="
kubectl get agentgatewaybackend -n agentgateway-system

echo "=== Route ==="
kubectl get httproute -n agentgateway-system

echo "=== Tracing Parameters ==="
kubectl get enterpriseagentgatewayparameters -n agentgateway-system
```

## Step 6: Test the Route via the UI Playground

1. Click the **KAgent UI** tab
2. Navigate to **AgentGateway → Playground**
3. Select the **openai** route
4. Click **Connect**
5. Type a message and click **Send**

You should see a response from OpenAI routed through the AgentGateway proxy.

## Step 7: Check Traces in the UI

After sending a request via the playground, navigate to **AgentGateway → Traces** in the UI. You should see the request trace showing the full path through AgentGateway to OpenAI, including token counts and latency.

## ✅ What You've Learned

- `EnterpriseAgentgatewayParameters` configures proxy behavior (tracing, logging, etc.)
- The Gateway resource references parameters via `infrastructure.parametersRef`
- `AgentgatewayBackend` defines LLM provider connections with auth
- `HTTPRoute` maps URL paths to backends
- All traffic through the gateway is traced to ClickHouse via the telemetry collector
