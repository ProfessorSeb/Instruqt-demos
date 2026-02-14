---
slug: mcp-federation
id: 0qy2amitryky
type: challenge
title: "MCP Federation \U0001F310 Multi-Server Routing"
teaser: Route to multiple MCP servers based on path — one gateway, many tools.
notes:
- type: text
  contents: "# \U0001F310 MCP Federation\n\nReal agents don't use just one tool —
    they use many. GitHub for code, Slack for messaging, databases for data.\n\nIn
    this challenge, you'll federate multiple MCP servers behind a single gateway endpoint.\n"
tabs:
- id: bizdzqq7zeyg
  title: Terminal
  type: terminal
  hostname: server
- id: cnlmna088gix
  title: Editor
  type: code
  hostname: server
  path: /root
- id: 0huxufksqenm
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: ""
enhanced_loading: null
---

# MCP Federation 🌐 Multi-Server Routing

Real-world agents use many tools. A coding agent might need GitHub for repositories, Slack for notifications, and a web fetcher for research. Instead of configuring each tool separately, AgentGateway can **federate** them behind a single gateway.

```
Agent → /mcp/github  → GitHub MCP Server
Agent → /mcp/slack   → Slack MCP Server
Agent → /mcp         → Fetch MCP Server (from previous challenge)
```

## Step 1: Deploy Additional MCP Servers

We'll deploy two more instances of the MCP website fetcher with different names to simulate different tool domains (GitHub and Slack):

```bash
cat > /root/mcp-github.yaml << 'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-github
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-github
  template:
    metadata:
      labels:
        app: mcp-github
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
  name: mcp-github
  namespace: default
spec:
  selector:
    app: mcp-github
  ports:
  - port: 80
    targetPort: 8000
    appProtocol: kgateway.dev/mcp
YAML

kubectl apply -f /root/mcp-github.yaml
```

Now the Slack MCP server:

```bash
cat > /root/mcp-slack.yaml << 'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-slack
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-slack
  template:
    metadata:
      labels:
        app: mcp-slack
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
  name: mcp-slack
  namespace: default
spec:
  selector:
    app: mcp-slack
  ports:
  - port: 80
    targetPort: 8000
    appProtocol: kgateway.dev/mcp
YAML

kubectl apply -f /root/mcp-slack.yaml
```

Wait for all servers:

```bash
kubectl wait --for=condition=Ready pod -l app=mcp-github --timeout=120s
kubectl wait --for=condition=Ready pod -l app=mcp-slack --timeout=120s
```

## Step 2: Create AgentgatewayBackends

```bash
cat > /root/federation-backends.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: github-mcp
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: github-target
      static:
        host: mcp-github.default.svc.cluster.local
        port: 80
        protocol: SSE
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: slack-mcp
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: slack-target
      static:
        host: mcp-slack.default.svc.cluster.local
        port: 80
        protocol: SSE
YAML

kubectl apply -f /root/federation-backends.yaml
```

## Step 3: Create Path-Based HTTPRoutes

Now create routes that direct traffic based on the path:

```bash
cat > /root/federation-routes.yaml << 'YAML'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: github-mcp-route
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp/github
    backendRefs:
    - name: github-mcp
      group: agentgateway.dev
      kind: AgentgatewayBackend
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: slack-mcp-route
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp/slack
    backendRefs:
    - name: slack-mcp
      group: agentgateway.dev
      kind: AgentgatewayBackend
YAML

kubectl apply -f /root/federation-routes.yaml
```

## Step 4: Test Federation 🧪

Test GitHub tools via the gateway:

```bash
curl -s http://localhost:8080/mcp/github -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

Test Slack tools:

```bash
curl -s http://localhost:8080/mcp/slack -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

The original fetch route still works too:

```bash
curl -s http://localhost:8080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

You can also check these in the **MCP Inspector** tab for visual verification.

🎉 **One gateway, three MCP servers, path-based routing!** The agent just needs to know the gateway address and the path prefix for each tool domain.
