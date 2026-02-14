---
slug: static-mcp
id: q9n8iov0ygxm
type: challenge
title: "Static MCP Routing \U0001F3AF"
teaser: Deploy a mock MCP server and route traffic to it through AgentGateway.
notes:
- type: text
  contents: "# \U0001F3AF Static MCP Routing\n\nTime to get hands-on! You'll deploy
    a mock MCP tool server, create an AgentGateway backend, and route MCP traffic
    through the gateway.\n\nThis is the foundation for everything else in this track.\n"
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
difficulty: ""
enhanced_loading: null
---

# Static MCP Routing 🎯

In this challenge, you'll deploy a mock MCP server and route traffic to it through AgentGateway. This is the fundamental building block for MCP connectivity.

## Step 1: Deploy a Mock MCP Server

First, let's create a mock MCP server that exposes a `fetch` tool. This is a simple Python HTTP server that speaks the MCP JSON-RPC protocol:

```bash
cat > /root/fetch-mcp-server.yaml << 'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: fetch-mcp-server-code
  namespace: mcp
data:
  server.py: |
    from http.server import HTTPServer, BaseHTTPRequestHandler
    import json

    TOOLS = [
        {
            "name": "fetch",
            "description": "Fetch content from a URL",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "url": {"type": "string", "description": "The URL to fetch"}
                },
                "required": ["url"]
            }
        }
    ]

    class MCPHandler(BaseHTTPRequestHandler):
        def do_POST(self):
            length = int(self.headers.get('Content-Length', 0))
            body = json.loads(self.rfile.read(length))
            method = body.get('method', '')

            if method == 'tools/list':
                result = {"tools": TOOLS}
            elif method == 'tools/call':
                tool_name = body.get('params', {}).get('name', '')
                args = body.get('params', {}).get('arguments', {})
                url = args.get('url', 'unknown')
                result = {"content": [{"type": "text", "text": f"Fetched content from {url}: <html>Mock page content</html>"}]}
            elif method == 'initialize':
                result = {"protocolVersion": "2024-11-05", "capabilities": {"tools": {"listChanged": False}}, "serverInfo": {"name": "fetch-mcp", "version": "1.0.0"}}
            else:
                result = {"error": {"code": -32601, "message": f"Method not found: {method}"}}

            response = json.dumps({"jsonrpc": "2.0", "result": result, "id": body.get('id', 1)})
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(response.encode())

        def log_message(self, format, *args):
            print(f"MCP Server: {args[0]}")

    print("Starting Fetch MCP Server on port 8080...")
    HTTPServer(('0.0.0.0', 8080), MCPHandler).serve_forever()
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fetch-mcp-server
  namespace: mcp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fetch-mcp-server
  template:
    metadata:
      labels:
        app: fetch-mcp-server
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
          name: fetch-mcp-server-code
---
apiVersion: v1
kind: Service
metadata:
  name: fetch-mcp-server
  namespace: mcp
spec:
  selector:
    app: fetch-mcp-server
  ports:
  - port: 8080
    targetPort: 8080
    appProtocol: kgateway.dev/mcp
YAML
```

Apply it:

```bash
kubectl apply -f /root/fetch-mcp-server.yaml
```

Wait for the pod to be ready:

```bash
kubectl -n mcp wait --for=condition=Ready pod -l app=fetch-mcp-server --timeout=120s
```

## Step 2: Create the Gateway

Create a Gateway resource that AgentGateway will implement:

```bash
cat > /root/mcp-gateway.yaml << 'YAML'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: mcp-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - name: http
    port: 8080
    protocol: HTTP
YAML

kubectl apply -f /root/mcp-gateway.yaml
```

## Step 3: Create the AgentgatewayBackend

This tells AgentGateway where your MCP server lives:

```bash
cat > /root/fetch-backend.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: fetch-mcp
  namespace: agentgateway-system
spec:
  mcp:
    target:
      host: fetch-mcp-server.mcp.svc.cluster.local
      port: 8080
YAML

kubectl apply -f /root/fetch-backend.yaml
```

## Step 4: Create the HTTPRoute

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
  - name: mcp-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp
    backendRefs:
    - group: agentgateway.dev
      kind: AgentgatewayBackend
      name: fetch-mcp
YAML

kubectl apply -f /root/mcp-route.yaml
```

## Step 5: Test It! 🧪

Get the gateway service IP and port-forward to it:

```bash
kubectl -n agentgateway-system port-forward svc/mcp-gateway 9080:8080 &
sleep 2
```

Now test the MCP protocol — list available tools:

```bash
curl -s http://localhost:9080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

You should see the `fetch` tool listed! Now call it:

```bash
curl -s http://localhost:9080/mcp -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"fetch","arguments":{"url":"https://example.com"}},"id":2}' | jq .
```

🎉 You just routed MCP traffic through AgentGateway! The agent sends standard JSON-RPC, the gateway routes it to the right MCP server.
