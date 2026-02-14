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
difficulty: ""
enhanced_loading: null
---

# MCP Federation 🌐 Multi-Server Routing

Real-world agents use many tools. A coding agent might need GitHub for repositories, Slack for notifications, and PostgreSQL for data. Instead of configuring each tool separately, AgentGateway can **federate** them behind a single gateway.

```
Agent → /mcp/github  → GitHub MCP Server
Agent → /mcp/slack   → Slack MCP Server
Agent → /mcp/db      → Database MCP Server
```

## Step 1: Deploy Additional Mock MCP Servers

Let's create GitHub and Slack mock MCP servers alongside the existing fetch server:

```bash
cat > /root/github-mcp-server.yaml << 'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: github-mcp-server-code
  namespace: mcp
data:
  server.py: |
    from http.server import HTTPServer, BaseHTTPRequestHandler
    import json
    TOOLS = [
        {"name": "get_repo", "description": "Get repository information", "inputSchema": {"type": "object", "properties": {"owner": {"type": "string"}, "repo": {"type": "string"}}, "required": ["owner", "repo"]}},
        {"name": "list_issues", "description": "List repository issues", "inputSchema": {"type": "object", "properties": {"owner": {"type": "string"}, "repo": {"type": "string"}}, "required": ["owner", "repo"]}},
        {"name": "create_issue", "description": "Create a new issue", "inputSchema": {"type": "object", "properties": {"owner": {"type": "string"}, "repo": {"type": "string"}, "title": {"type": "string"}, "body": {"type": "string"}}, "required": ["owner", "repo", "title"]}},
        {"name": "delete_repo", "description": "Delete a repository (DANGEROUS)", "inputSchema": {"type": "object", "properties": {"owner": {"type": "string"}, "repo": {"type": "string"}}, "required": ["owner", "repo"]}}
    ]
    class MCPHandler(BaseHTTPRequestHandler):
        def do_POST(self):
            length = int(self.headers.get('Content-Length', 0))
            body = json.loads(self.rfile.read(length))
            method = body.get('method', '')
            if method == 'tools/list':
                result = {"tools": TOOLS}
            elif method == 'tools/call':
                name = body.get('params', {}).get('name', '')
                args = body.get('params', {}).get('arguments', {})
                result = {"content": [{"type": "text", "text": f"Mock GitHub result for {name}: owner={args.get('owner','?')}, repo={args.get('repo','?')}"}]}
            elif method == 'initialize':
                result = {"protocolVersion": "2024-11-05", "capabilities": {"tools": {"listChanged": False}}, "serverInfo": {"name": "github-mcp", "version": "1.0.0"}}
            else:
                result = {"error": {"code": -32601, "message": "Method not found"}}
            response = json.dumps({"jsonrpc": "2.0", "result": result, "id": body.get('id', 1)})
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(response.encode())
        def log_message(self, format, *args):
            pass
    HTTPServer(('0.0.0.0', 8080), MCPHandler).serve_forever()
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-mcp-server
  namespace: mcp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: github-mcp-server
  template:
    metadata:
      labels:
        app: github-mcp-server
    spec:
      containers:
      - name: server
        image: python:3.11-slim
        command: ["python", "/app/server.py"]
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: code
          mountPath: /app
      volumes:
      - name: code
        configMap:
          name: github-mcp-server-code
---
apiVersion: v1
kind: Service
metadata:
  name: github-mcp-server
  namespace: mcp
spec:
  selector:
    app: github-mcp-server
  ports:
  - port: 8080
    targetPort: 8080
    appProtocol: kgateway.dev/mcp
YAML

kubectl apply -f /root/github-mcp-server.yaml
```

Now the Slack MCP server:

```bash
cat > /root/slack-mcp-server.yaml << 'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: slack-mcp-server-code
  namespace: mcp
data:
  server.py: |
    from http.server import HTTPServer, BaseHTTPRequestHandler
    import json
    TOOLS = [
        {"name": "list_channels", "description": "List Slack channels", "inputSchema": {"type": "object", "properties": {}}},
        {"name": "send_message", "description": "Send a message to a channel", "inputSchema": {"type": "object", "properties": {"channel": {"type": "string"}, "text": {"type": "string"}}, "required": ["channel", "text"]}},
        {"name": "get_messages", "description": "Get recent messages from a channel", "inputSchema": {"type": "object", "properties": {"channel": {"type": "string"}}, "required": ["channel"]}},
        {"name": "delete_channel", "description": "Delete a Slack channel (DANGEROUS)", "inputSchema": {"type": "object", "properties": {"channel": {"type": "string"}}, "required": ["channel"]}}
    ]
    class MCPHandler(BaseHTTPRequestHandler):
        def do_POST(self):
            length = int(self.headers.get('Content-Length', 0))
            body = json.loads(self.rfile.read(length))
            method = body.get('method', '')
            if method == 'tools/list':
                result = {"tools": TOOLS}
            elif method == 'tools/call':
                name = body.get('params', {}).get('name', '')
                result = {"content": [{"type": "text", "text": f"Mock Slack result for {name}"}]}
            elif method == 'initialize':
                result = {"protocolVersion": "2024-11-05", "capabilities": {"tools": {"listChanged": False}}, "serverInfo": {"name": "slack-mcp", "version": "1.0.0"}}
            else:
                result = {"error": {"code": -32601, "message": "Method not found"}}
            response = json.dumps({"jsonrpc": "2.0", "result": result, "id": body.get('id', 1)})
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(response.encode())
        def log_message(self, format, *args):
            pass
    HTTPServer(('0.0.0.0', 8080), MCPHandler).serve_forever()
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slack-mcp-server
  namespace: mcp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: slack-mcp-server
  template:
    metadata:
      labels:
        app: slack-mcp-server
    spec:
      containers:
      - name: server
        image: python:3.11-slim
        command: ["python", "/app/server.py"]
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: code
          mountPath: /app
      volumes:
      - name: code
        configMap:
          name: slack-mcp-server-code
---
apiVersion: v1
kind: Service
metadata:
  name: slack-mcp-server
  namespace: mcp
spec:
  selector:
    app: slack-mcp-server
  ports:
  - port: 8080
    targetPort: 8080
    appProtocol: kgateway.dev/mcp
YAML

kubectl apply -f /root/slack-mcp-server.yaml
```

Wait for all servers:

```bash
kubectl -n mcp wait --for=condition=Ready pod -l app=github-mcp-server --timeout=120s
kubectl -n mcp wait --for=condition=Ready pod -l app=slack-mcp-server --timeout=120s
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
    target:
      host: github-mcp-server.mcp.svc.cluster.local
      port: 8080
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: slack-mcp
  namespace: agentgateway-system
spec:
  mcp:
    target:
      host: slack-mcp-server.mcp.svc.cluster.local
      port: 8080
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
  - name: mcp-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp/github
    backendRefs:
    - group: agentgateway.dev
      kind: AgentgatewayBackend
      name: github-mcp
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: slack-mcp-route
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: mcp-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp/slack
    backendRefs:
    - group: agentgateway.dev
      kind: AgentgatewayBackend
      name: slack-mcp
YAML

kubectl apply -f /root/federation-routes.yaml
```

## Step 4: Test Federation 🧪

Make sure port-forward is running:

```bash
# Kill any existing port-forward
pkill -f "port-forward.*mcp-gateway" || true
kubectl -n agentgateway-system port-forward svc/mcp-gateway 9080:8080 &
sleep 2
```

Test GitHub tools:

```bash
curl -s http://localhost:9080/mcp/github -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

Test Slack tools:

```bash
curl -s http://localhost:9080/mcp/slack -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

Call a GitHub tool:

```bash
curl -s http://localhost:9080/mcp/github -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"list_issues","arguments":{"owner":"solo-io","repo":"agentgateway"}},"id":2}' | jq .
```

🎉 **One gateway, three MCP servers, path-based routing!** The agent just needs to know the gateway address and the path prefix for each tool domain.
