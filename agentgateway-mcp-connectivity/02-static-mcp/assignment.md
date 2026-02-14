---
slug: static-mcp
id: q9n8iov0ygxm
type: challenge
title: "Static MCP Routing \U0001F3AF"
teaser: Deploy an MCP server and route traffic to it through AgentGateway.
notes:
- type: text
  contents: "# \U0001F3AF Static MCP Routing\n\nTime to get hands-on! You'll deploy
    an MCP tool server, create an AgentGateway backend, and route MCP traffic through
    the gateway.\n\nThis is the foundation for everything else in this track.\n"
tabs:
- id: snmgz50inhyi
  title: Terminal
  type: terminal
  hostname: server
- id: kz7m479cul1p
  title: Editor
  type: code
  hostname: server
  path: /root
- id: bgvirymqtixg
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: ""
enhanced_loading: null
---

# Static MCP Routing 🎯

In this challenge, you'll deploy an MCP server and route traffic to it through AgentGateway. This is the fundamental building block for MCP connectivity.

## Step 1: Deploy an MCP Server

We'll use the `mcp-website-fetcher` — a real MCP server that exposes a `fetch` tool for fetching web page content:

```bash
cat > /root/mcp-website-fetcher.yaml << 'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-website-fetcher
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-website-fetcher
  template:
    metadata:
      labels:
        app: mcp-website-fetcher
    spec:
      containers:
      - name: server
        image: ghcr.io/peterj/mcp-website-fetcher:main
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-website-fetcher
  namespace: default
spec:
  selector:
    app: mcp-website-fetcher
  ports:
  - port: 80
    targetPort: 8000
    appProtocol: kgateway.dev/mcp
YAML
```

Apply it:

```bash
kubectl apply -f /root/mcp-website-fetcher.yaml
```

Wait for the pod to be ready:

```bash
kubectl wait --for=condition=Ready pod -l app=mcp-website-fetcher --timeout=120s
```

## Step 2: Create the AgentgatewayBackend

This tells AgentGateway where your MCP server lives. Note the `targets` array with `static` block format:

```bash
cat > /root/mcp-backend.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mcp-backend
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: mcp-target
      static:
        host: mcp-website-fetcher.default.svc.cluster.local
        port: 80
        protocol: SSE
YAML

kubectl apply -f /root/mcp-backend.yaml
```

## Step 3: Create the HTTPRoute

Route MCP traffic from the gateway to the backend:

```bash
cat > /root/mcp-route.yaml << 'YAML'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-route
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
      group: agentgateway.dev
      kind: AgentgatewayBackend
YAML

kubectl apply -f /root/mcp-route.yaml
```

## Step 4: Test It! 🧪

The agentgateway-proxy is already port-forwarded to `localhost:8080`. Test the MCP protocol — list available tools:

```bash
curl -s http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

You should see the `fetch` tool listed! Now call it:

```bash
curl -s http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"fetch","arguments":{"url":"https://example.com"}},"id":2}' | jq .
```

You can also use the **MCP Inspector** tab to visually inspect and test your MCP server connection. Point it at `http://localhost:8080/mcp` to see available tools.

🎉 You just routed MCP traffic through AgentGateway! The agent sends standard JSON-RPC, the gateway routes it to the right MCP server.
