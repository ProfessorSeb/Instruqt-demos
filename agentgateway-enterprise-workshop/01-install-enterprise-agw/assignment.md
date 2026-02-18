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
- id: solouiinstall01
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

Check that the Enterprise AgentGateway controller and shared extensions are running:

```bash
kubectl get pods -n enterprise-agentgateway
```

You should see:
- **enterprise-agentgateway** — the control plane
- **agentgateway** — the data plane proxy
- **ext-auth-service** — external auth for API key and JWT validation
- **rate-limiter** — rate limiting service
- **ext-cache** — Redis cache for rate limit counters

## Step 2: Verify the CRDs

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

Check that the Gateway resource is ready and has an address:

```bash
kubectl get gateway -n enterprise-agentgateway
```

Save the gateway IP for later challenges:

```bash
export GATEWAY_IP=$(kubectl get svc -n enterprise-agentgateway --selector=gateway.networking.k8s.io/gateway-name=agentgateway -o jsonpath='{.items[*].status.loadBalancer.ingress[0].ip}{.items[*].status.loadBalancer.ingress[0].hostname}')
echo "Gateway IP: $GATEWAY_IP"
echo "export GATEWAY_IP=$GATEWAY_IP" >> /root/.bashrc
```

## Step 4: Verify the Monitoring Stack

Check that Grafana, Prometheus, and Tempo are running:

```bash
kubectl get pods -n monitoring
```

You should see pods for:
- **grafana-prometheus** — Grafana with Prometheus data source
- **prometheus** — metrics collection
- **tempo** — distributed tracing

## Step 5: Verify the Solo Enterprise Management UI

The Solo Enterprise Management UI provides a visual dashboard for managing and observing your AgentGateway deployment, including tracing, gateway configuration, and policy management.

Check that the Solo UI components are running:

```bash
kubectl get pods -n agentgateway-system
```

You should see pods for the management UI frontend, backend, and supporting services (telemetry collector, ClickHouse, etc.).

Switch to the **Solo UI** tab to open the management dashboard. From here you can:
- View gateway topology and configuration
- Explore distributed traces for LLM and MCP traffic
- Monitor agent activity across your deployment

## Step 6: Review the Gateway Configuration

The `EnterpriseAgentgatewayParameters` resource configures the gateway's behavior. Let's look at what was deployed:

```bash
kubectl get enterpriseagentgatewayparameters agentgateway-params -n enterprise-agentgateway -o yaml
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
