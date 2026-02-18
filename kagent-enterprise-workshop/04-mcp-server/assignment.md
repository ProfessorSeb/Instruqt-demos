---
slug: mcp-server
id: koub9navde6j
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
- id: clmdwkzq7kjk
  title: Terminal
  type: terminal
  hostname: server
- id: 4n9apsnfjjex
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: ugr56lxkgxpi
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

Create an `AgentgatewayBackend` that discovers MCP servers by label, and an `HTTPRoute` that exposes them on the `/mcp` path through the existing `agentgateway-proxy`:

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
      selector:
        services:
          matchLabels:
            app: mcp-server-everything
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

## Step 4: Test MCP Tool Discovery

Get the proxy service IP and test tool discovery via the MCP protocol:

```bash
export GATEWAY_IP=$(kubectl get svc -n agentgateway-system -l gateway.networking.k8s.io/gateway-name=agentgateway-proxy -o jsonpath='{.items[0].spec.clusterIP}')

curl -s "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}' | jq .
```

You should see a list of tools exposed by the MCP server.

## Step 5: Call an MCP Tool

Test calling one of the tools through the gateway:

```bash
curl -s "http://$GATEWAY_IP:8080/mcp" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/call", "params": {"name": "echo", "arguments": {"message": "Hello from AgentGateway!"}}}' | jq .
```

## Step 6: Verify in the UI

Click the **KAgent UI** tab and navigate to **AgentGateway → Routes**. You should see the `mcp` route listed alongside the `openai` route. Check the traces to see the MCP tool call.

## ✅ What You've Learned

- `AgentgatewayBackend` with `mcp.targets` routes to MCP servers
- The `selector` field discovers MCP servers by Kubernetes service labels
- `appProtocol: agentgateway.dev/mcp` on Services enables MCP protocol handling
- MCP traffic gets full observability (logs, metrics, traces) through the gateway
- Tool discovery (`tools/list`) and tool calls (`tools/call`) work through the gateway
