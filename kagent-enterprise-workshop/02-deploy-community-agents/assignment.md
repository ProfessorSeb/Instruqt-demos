---
slug: deploy-community-agents
id: y3vtyfqynnrp
type: challenge
title: Deploy Community Agents
teaser: Install KAgent community agents and tools — Kubernetes, Helm, kGateway, Grafana
  MCP, and QueryDoc.
notes:
- type: text
  contents: "# \U0001F916 Community Agents & Tools\n\nKAgent ships with a library
    of community agents and tools that provide ready-made capabilities:\n\n- **k8s-agent**
    — Kubernetes cluster operations and troubleshooting\n- **helm-agent** — Helm chart
    management and deployments\n- **kgateway-agent** — kGateway/AgentGateway configuration\n-
    **grafana-mcp** — Query Grafana dashboards and metrics via MCP\n- **querydoc**
    — Search and query documentation\n\nLet's deploy them and verify they're running!\n"
tabs:
- id: bh7pxzvahjfz
  title: Terminal
  type: terminal
  hostname: server
- id: egjw4kolwey2
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: rcfqdvp2d5mv
  title: KAgent UI
  type: service
  hostname: server
  port: 8080
difficulty: ""
enhanced_loading: null
---

# Deploy Community Agents

KAgent Enterprise includes community agents and tools that you can enable with a simple Helm upgrade. In this challenge, you'll deploy agents for Kubernetes, Helm, and kGateway operations, plus MCP tools for Grafana and documentation search.

## Step 1: Create the Community Agents Values File

```bash
cat << 'EOF' > /root/community-agents.yaml
kagent-tools:
  enabled: true
agents:
  k8s-agent:
    enabled: true
  kgateway-agent:
    enabled: true
  helm-agent:
    enabled: true
tools:
  grafana-mcp:
    enabled: true
  querydoc:
    enabled: true
EOF
```

## Step 2: Upgrade KAgent with Community Agents

```bash
helm upgrade kagent \
  oci://us-docker.pkg.dev/solo-public/kagent-enterprise-helm/charts/kagent-enterprise \
  --version $KAGENT_ENT_VERSION \
  -n kagent \
  --reuse-values \
  -f /root/community-agents.yaml \
  --wait --timeout 300s
```

This adds the community agents and tools to your existing KAgent installation without changing any other settings.

## Step 3: Verify Agents Are Running

Check that the agent pods are deployed:

```bash
kubectl get pods -n kagent -l app.kubernetes.io/part-of=kagent
```

You should see pods for:
- `k8s-agent`
- `kgateway-agent`
- `helm-agent`

Check the agent custom resources:

```bash
kubectl get agents -n kagent
```

## Step 4: Verify Tools Are Running

```bash
kubectl get pods -n kagent -l app.kubernetes.io/name=grafana-mcp
kubectl get pods -n kagent -l app.kubernetes.io/name=querydoc
```

You should see:
- `kagent-grafana-mcp`
- `kagent-querydoc`

Check the tool server resources:

```bash
kubectl get remotemcpservers -n kagent
```

## Step 5: Verify in the UI

Click the **KAgent UI** tab. You should now see the deployed agents listed in the dashboard. Click on any agent to see its configuration, tools, and status.

## Step 6: Test an Agent

Let's verify the k8s-agent is working by checking its status:

```bash
kubectl get agent k8s-agent -n kagent -o yaml | grep -A5 "status:"
```

## ✅ What You've Learned

- Community agents are enabled via Helm values with `--reuse-values`
- `k8s-agent`, `helm-agent`, and `kgateway-agent` provide Kubernetes operational capabilities
- `grafana-mcp` and `querydoc` are MCP tool servers that agents can use
- All agents and tools are visible in the KAgent UI
- Agents are represented as Kubernetes custom resources (`agents.kagent.dev`)

**Next up:** Use the agents to perform real tasks on your cluster!
