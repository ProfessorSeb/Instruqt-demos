---
slug: connect-mcp-tool-server
id: dzm9kytvd2yo
type: challenge
title: Connect an MCP Tool Server
teaser: Deploy an MCP tool server, route it through the gateway, and test with MCP
  Inspector.
notes:
- type: text
  contents: "# MCP Tool Server \U0001F50C\n\nThe Model Context Protocol (MCP) lets
    AI agents discover and use tools dynamically. You'll deploy a real MCP server
    that can fetch website content, then route it through AgentGateway.\n\nThe **MCP
    Inspector** tab lets you interactively test tool discovery and invocation."
tabs:
- id: f8dmttnf9i1m
  title: Terminal
  type: terminal
  hostname: server
- id: dodbjf3qzo2t
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: 8ysqqjfoe2kk
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: lbylxrxxglcr
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: basic
timelimit: 900
enhanced_loading: null
---

# Connect an MCP Tool Server

## Step 1: Deploy the MCP Website Fetcher

Create `/root/mcp-server.yaml` with the deployment and service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-website-fetcher
  namespace: enterprise-agentgateway
spec:
  selector:
    matchLabels:
      app: mcp-website-fetcher
  template:
    metadata:
      labels:
        app: mcp-website-fetcher
    spec:
      containers:
      - name: mcp-website-fetcher
        image: ghcr.io/peterj/mcp-website-fetcher:main
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-website-fetcher
  namespace: enterprise-agentgateway
spec:
  selector:
    app: mcp-website-fetcher
  ports:
  - port: 80
    targetPort: 8000
    appProtocol: agentgateway.dev/mcp
```

> **Note:** The `appProtocol: agentgateway.dev/mcp` tells Enterprise AgentGateway this is an MCP service.

Apply it:

```bash
kubectl apply -f /root/mcp-server.yaml
kubectl -n enterprise-agentgateway wait --for=condition=Ready pod -l app=mcp-website-fetcher --timeout=120s
```

## Step 2: Create the MCP Backend and Route

Create `/root/mcp-route.yaml`:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mcp-backend
  namespace: enterprise-agentgateway
spec:
  mcp:
    targets:
    - name: mcp-target
      static:
        host: mcp-website-fetcher.enterprise-agentgateway.svc.cluster.local
        port: 80
        protocol: SSE
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp
  namespace: enterprise-agentgateway
spec:
  parentRefs:
  - name: agentgateway
  rules:
  - backendRefs:
    - name: mcp-backend
      group: agentgateway.dev
      kind: AgentgatewayBackend
```

Apply it:

```bash
kubectl apply -f /root/mcp-route.yaml
```

## Step 3: Test with MCP Inspector

Open the **MCP Inspector** tab:

1. Set the URL to: `http://localhost:8080/mcp`
2. Set the transport type to: **Streamable HTTP**
3. Click **Connect**
4. Navigate to **Tools** and click **List Tools**
5. You should see the `fetch_website` tool!
6. Try calling it with a URL like `https://solo.io`

You can also check Grafana for MCP traces — every tool call is instrumented.

Click **Check** when your MCP server is connected and working.
