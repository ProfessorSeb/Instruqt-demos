---
slug: deploy-kagent-enterprise
type: challenge
title: Deploy KAgent Enterprise
teaser: Verify the KAgent Enterprise stack with AgentGateway is running on Kubernetes.
notes:
- type: text
  contents: "# \U0001F916 KAgent Enterprise\n\nKAgent Enterprise is Solo.io's AI agent
    orchestration platform. Combined with Enterprise AgentGateway, it provides:\n\n-
    **Agent management** — deploy and monitor AI agents on Kubernetes\n- **MCP tool
    routing** — connect agents to tools through the gateway\n- **Tracing & observability**
    — full visibility into agent behavior\n- **Enterprise security** — JWT auth, RBAC,
    guardrails\n\nYour environment has been pre-deployed with:\n- Enterprise AgentGateway
    in `agentgateway-system`\n- KAgent Enterprise in `kagent`\n- ClickHouse for tracing
    data\n- Solo Enterprise UI\n\nLet's verify everything is running!\n"
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
- title: KAgent UI
  type: service
  hostname: server
  port: 12121
difficulty: ""
enhanced_loading: null
---

# Deploy KAgent Enterprise

Your environment has been pre-deployed with the full KAgent Enterprise stack. Let's verify everything is running correctly.

## Step 1: Check Enterprise AgentGateway

Verify the AgentGateway controller is running:

```bash
kubectl get pods -n agentgateway-system
```

You should see the `enterprise-agentgateway` controller pod in `Running` state.

```bash
kubectl get deployment -n agentgateway-system
```

## Step 2: Check KAgent Enterprise Components

Verify all KAgent components are running:

```bash
kubectl get pods -n kagent
```

You should see pods for:
- `kagent` — the agent controller
- `kagent-mgmt` / `solo-enterprise-ui` — the management UI
- `clickhouse` — tracing backend
- `kmcp` — MCP controller

```bash
kubectl get deployments -n kagent
```

## Step 3: Verify CRDs Are Installed

Check that the KAgent and AgentGateway CRDs are available:

```bash
kubectl get crds | grep -E "kagent|agentgateway"
```

You should see CRDs for agents, tools, MCP servers, and gateway resources.

## Step 4: Access the KAgent UI

Click the **KAgent UI** tab above. You should see the Solo Enterprise UI dashboard.

This UI provides:
- Agent management and monitoring
- AgentGateway configuration
- Tracing and observability data from ClickHouse

## Step 5: Save Environment Variables

```bash
export KAGENT_ENT_VERSION=0.3.4
echo "export KAGENT_ENT_VERSION=$KAGENT_ENT_VERSION" >> ~/.bashrc
```

## ✅ What You've Learned

- Enterprise AgentGateway runs in `agentgateway-system` as the AI-native gateway
- KAgent Enterprise runs in `kagent` with controller, MCP, and management UI
- ClickHouse provides the tracing backend for agent observability
- The Solo Enterprise UI gives a unified dashboard for both products
