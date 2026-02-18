---
slug: mcp-server
id: w9ki7r8vvn6x
type: challenge
title: MCP Server Connectivity
teaser: Deploy an MCP server behind AgentGateway and test tool discovery through the
  gateway.
notes:
- type: text
  contents: "# \U0001F527 MCP Server Connectivity\n\nMCP (Model Context Protocol)
    lets AI agents discover and call tools dynamically. By routing MCP traffic through
    AgentGateway, you get:\n\n- **Centralized tool access** — agents connect to the
    gateway, not individual servers\n- **Full observability** — every tool call is
    traced\n- **Security** — JWT auth, RBAC, and rate limiting on tool access\n\nYou'll
    deploy the `mcp-server-everything` (a reference MCP server) and route it through
    the AgentGateway proxy.\n"
tabs:
- id: gt5sqrkhvmxa
  title: Terminal
  type: terminal
  hostname: server
- id: vnvu5pryyedt
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: mglxc5xyhp5p
  title: KAgent UI
  type: service
  hostname: server
  port: 8080
difficulty: ""
enhanced_loading: null
---

# MCP Server Connectivity

In this challenge, you'll deploy an MCP server behind the AgentGateway proxy and verify tool discovery works through the gateway.

## Step 1: Deploy the MCP Server

Deploy the `mcp-server-everything` — a reference MCP server that exposes sample tools:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server-everything
  namespace: agentgateway-system
  labels:
    app: mcp-server-everything
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-server-everything
  template:
    metadata:
      labels:
        app: mcp-server-everything
    spec:
      containers:
      - name: mcp-server-everything
        image: node:20-alpine
        command: ["npx"]
        args: ["-y", "@modelcontextprotocol/server-everything", "streamableHttp"]
        ports:
        - containerPort: 3001
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-server-everything
  namespace: agentgateway-system
  labels:
    app: mcp-server-everything
spec:
  selector:
    app: mcp-server-everything
  ports:
  - protocol: TCP
    port: 3001
    targetPort: 3001
    appProtocol: agentgateway.dev/mcp
  type: ClusterIP
EOF
```

Wait for the MCP server to be ready:

```bash
kubectl rollout status deployment/mcp-server-everything -n agentgateway-system --timeout=120s
```

## Step 2: Create the MCP Backend and Route

Create an `AgentgatewayBackend` with a static target pointing to the MCP server, and an `HTTPRoute` that exposes it on the `/mcp` path through the existing `agentgateway-proxy`:

```bash
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mcp-backend
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: mcp-server-everything
      static:
        host: mcp-server-everything.agentgateway-system.svc.cluster.local
        port: 3001
        protocol: StreamableHTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /mcp
      backendRefs:
      - name: mcp-backend
        namespace: agentgateway-system
        group: agentgateway.dev
        kind: AgentgatewayBackend
EOF
```

## Step 3: Verify the Resources

```bash
echo "=== MCP Backend ==="
kubectl get agentgatewaybackend mcp-backend -n agentgateway-system

echo "=== MCP Route ==="
kubectl get httproute mcp -n agentgateway-system

echo "=== MCP Server Pods ==="
kubectl get pods -n agentgateway-system -l app=mcp-server-everything
```

## Step 4: Test MCP via the UI Playground

1. Click the **KAgent UI** tab
2. Navigate to **AgentGateway → Playground**
3. Select the **mcp** route
4. Click **Connect**
5. You should see the tools listed from the MCP server

## Step 5: Verify Traces

Navigate to **AgentGateway → Traces** in the UI. You should see the MCP tool discovery and call traces alongside any LLM traces from the previous challenge.

## ✅ What You've Learned

- `AgentgatewayBackend` with `mcp.targets` routes to MCP servers
- The `static` target type uses explicit host, port, and protocol
- `appProtocol: agentgateway.dev/mcp` on Services enables MCP protocol handling
- MCP traffic gets full observability (logs, metrics, traces) through the gateway
- Tool discovery (`tools/list`) and tool calls work through the gateway
- The UI Playground provides a visual way to test MCP connectivity
