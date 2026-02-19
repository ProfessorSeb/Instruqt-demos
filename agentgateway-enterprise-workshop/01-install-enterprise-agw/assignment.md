---
slug: install-enterprise-agw
id: vkoltz9xqdli
type: challenge
title: Install Enterprise AgentGateway
teaser: Deploy the enterprise-grade AI gateway with full observability on Kubernetes.
notes:
- type: text
  contents: "# \U0001F680 Enterprise AgentGateway\n\nEnterprise AgentGateway is a
    purpose-built gateway for AI agent traffic — LLMs, MCP tools, and A2A protocols.\n\nUnlike
    generic API gateways, it understands AI-native protocols and provides:\n\n- **Token-aware
    metrics** — not just request counts\n- **Prompt-level tracing** — see what agents
    actually send\n- **Built-in guardrails** — PII protection, prompt injection blocking\n-
    **JWT + API key auth** — with CEL-based RBAC\n- **Rate limiting** — per-request
    and per-token\n\nLet's get it installed and verify everything is running.\n"
tabs:
- id: zcxchzupgjej
  title: Terminal
  type: terminal
  hostname: server
- id: vij0iztm8y9r
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: 6pxjcx1rhoaj
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# Install Enterprise AgentGateway

The track setup has already installed Enterprise AgentGateway, the monitoring stack (Grafana, Prometheus, Tempo), the Solo Enterprise Management UI, and created the Gateway resource. Let's verify everything is running and understand what was deployed.

## Step 1: Verify Enterprise AgentGateway Components

> **What's happening:** The Enterprise AgentGateway runs as a set of pods in Kubernetes. The *control plane* (`enterprise-agentgateway`) watches for configuration changes and programs the *data plane* (`agentgateway`) — the proxy that actually handles LLM and MCP traffic. Supporting services like `ext-auth-service` (authentication), `rate-limiter`, and `ext-cache` (Redis) are shared extensions that the data plane calls out to when policies are enforced. This step confirms all of these components started successfully.

Check that the Enterprise AgentGateway controller and shared extensions are running:

```bash
kubectl get pods -n agentgateway-system
```

You should see:
- **enterprise-agentgateway** — the control plane
- **agentgateway** — the data plane proxy
- **ext-auth-service** — external auth for API key and JWT validation
- **rate-limiter** — rate limiting service
- **ext-cache** — Redis cache for rate limit counters

## Step 2: Verify the CRDs

> **What's happening:** Enterprise AgentGateway extends the Kubernetes API with Custom Resource Definitions (CRDs). These CRDs let you define AI gateway behavior — like which LLM providers to route to, what authentication to require, and what guardrails to enforce — as native Kubernetes YAML resources. This is the "infrastructure as code" model: your entire AI gateway policy lives in version-controlled Kubernetes manifests.

Enterprise AgentGateway extends Kubernetes with custom resources for AI gateway configuration:

```bash
kubectl api-resources | awk 'NR==1 || /enterpriseagentgateway\.solo\.io|agentgateway\.dev|ratelimit\.solo\.io|extauth\.solo\.io/'
```

Key CRDs:
- `AgentgatewayBackend` — defines LLM providers and MCP servers
- `AgentgatewayPolicy` — OSS-level policies
- `EnterpriseAgentgatewayPolicy` — enterprise policies (auth, rate limiting, guardrails)
- `EnterpriseAgentgatewayParameters` — gateway configuration (tracing, logging, extensions)
- `AuthConfig` — authentication configuration
- `RateLimitConfig` — rate limiting rules

## Step 3: Verify the Gateway

> **What's happening:** The `Gateway` resource is a standard Kubernetes Gateway API object. It tells the control plane to spin up a listener (the data plane proxy) on a specific port. All LLM and MCP traffic enters through this single Gateway, giving you one control point. A `kubectl port-forward` service is running in the background so you can reach the gateway on `localhost:8080` from the terminal.

Check that the Gateway resource is ready and has an address:

```bash
kubectl get gateway -n agentgateway-system
```

The gateway is accessible via port-forward on `localhost:8080`. Verify it's set up:

```bash
echo "Gateway IP: $GATEWAY_IP"
```

This should show `localhost`. The port-forward service routes traffic from `localhost:8080` to the gateway pod inside the cluster.

## Step 4: Verify the Monitoring Stack

> **What's happening:** A full observability stack was deployed alongside the gateway. **Prometheus** scrapes metrics from the gateway (request counts, token usage, latencies). **Tempo** collects distributed traces — every LLM request gets a trace showing the full journey from client → gateway → provider → response. **Grafana** provides dashboards and an Explore UI to query both metrics and traces. This gives you production-grade visibility into all AI traffic from day one.

Check that Grafana, Prometheus, and Tempo are running:

```bash
kubectl get pods -n monitoring
```

You should see pods for:
- **grafana-prometheus** — Grafana with Prometheus data source
- **prometheus** — metrics collection
- **tempo** — distributed tracing

## Step 5: Verify the Solo Enterprise Management UI

> **What's happening:** The Solo Enterprise Management UI is a dedicated dashboard purpose-built for managing AgentGateway. Unlike Grafana (which is a general-purpose tool), the Solo UI provides gateway-specific views: topology maps showing your backends and routes, AI-specific trace exploration, and policy management. It collects telemetry via its own collector (ClickHouse-backed), giving you a second, independent view of all gateway traffic.

Check that the Solo UI components are running:

```bash
kubectl get pods -n agentgateway-system -l app.kubernetes.io/instance=management
```

You should see pods for the management UI frontend, backend, and supporting services (telemetry collector, ClickHouse, etc.).

Switch to the **Solo UI** tab to open the management dashboard. From here you can:
- View gateway topology and configuration
- Explore distributed traces for LLM and MCP traffic
- Monitor agent activity across your deployment

## Step 6: Review the Gateway Configuration

> **What's happening:** The `EnterpriseAgentgatewayParameters` resource is the master configuration for the gateway's behavior. It tells the data plane where to find the shared extensions (ext-auth, rate-limiter, Redis), how to export traces (OTLP to Tempo and to the Solo collector), and what to include in access logs (request/response bodies, JWT claims, JSON format). This dual-export tracing setup means the same traffic data is available in both Grafana and the Solo UI — useful for different teams or workflows.

The `EnterpriseAgentgatewayParameters` resource configures the gateway's behavior. Let's look at what was deployed:

```bash
kubectl get enterpriseagentgatewayparameters agentgateway-params -n agentgateway-system -o yaml
```

Key configuration:
- **Shared Extensions**: ExtAuth, RateLimiter, and Redis cache are enabled
- **Tracing**: OTLP traces sent to Tempo for distributed tracing
- **Logging**: JSON-formatted access logs with request/response body capture
- **JWT capture**: JWT claims are logged for audit trails

The Gateway also references a tracing configuration that sends traces to the Solo telemetry collector for the management UI:

```bash
kubectl get enterpriseagentgatewayparameters tracing -n agentgateway-system -o yaml
```

This dual-export setup means traces are available in **both** Grafana (via Tempo) and the Solo UI (via the Solo telemetry collector).

## ✅ What You've Learned

- Enterprise AgentGateway is installed with its control plane and shared extensions
- The Gateway resource creates the data plane proxy
- Full observability is configured: metrics (Prometheus), traces (Tempo), dashboards (Grafana)
- Solo Enterprise Management UI provides visual gateway management and tracing
- Dual trace export sends data to both Tempo and the Solo telemetry collector
- Enterprise CRDs extend Kubernetes with AI-native policy resources

**Next up:** Route your first LLM traffic through the gateway.
