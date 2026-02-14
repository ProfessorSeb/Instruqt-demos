---
slug: mcp-authentication
id: osvbswxcvrmg
type: challenge
title: "MCP Authentication with OAuth \U0001F511"
teaser: Secure MCP endpoints with Keycloak OAuth — verify agent identity before granting
  tool access.
notes:
- type: text
  contents: "# \U0001F511 MCP Authentication\n\nRouting ✅ Authorization ✅ Rate limiting
    ✅\n\nBut who's actually calling? Without authentication, any client can hit your
    MCP endpoints. In this final challenge, you'll secure MCP with OAuth using Keycloak.\n"
tabs:
- id: gfwi5qx4d6qr
  title: Terminal
  type: terminal
  hostname: server
- id: 2tyadbequqed
  title: Editor
  type: code
  hostname: server
  path: /root
- id: cwcmb6typ2mk
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: ""
enhanced_loading: null
---

# MCP Authentication with OAuth 🔑

So far, anyone who can reach the gateway can call MCP tools. In production, you need to verify **who** is calling — is it an authorized agent? Which team does it belong to?

AgentGateway supports OAuth authentication for MCP endpoints. We'll use **Keycloak** as the identity provider (IdP).

## The OAuth Flow for MCP

```
1. Agent → Gateway: "I want to call tools"
2. Gateway → Agent: "Authenticate first" (401)
3. Agent → Keycloak: "Here are my credentials"
4. Keycloak → Agent: "Here's your access token"
5. Agent → Gateway: "Here's my token + tool call"
6. Gateway → Keycloak: "Is this token valid?"
7. Gateway → MCP Server: (forwards the call)
```

## Step 1: Deploy Keycloak

```bash
cat > /root/keycloak.yaml << 'YAML'
apiVersion: v1
kind: Namespace
metadata:
  name: keycloak
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:24.0
        args: ["start-dev"]
        env:
        - name: KEYCLOAK_ADMIN
          value: admin
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: admin
        - name: KC_HTTP_PORT
          value: "8180"
        ports:
        - containerPort: 8180
        readinessProbe:
          httpGet:
            path: /realms/master
            port: 8180
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
spec:
  selector:
    app: keycloak
  ports:
  - port: 8180
    targetPort: 8180
YAML

kubectl apply -f /root/keycloak.yaml
```

Wait for Keycloak to be ready (this may take a minute):

```bash
kubectl -n keycloak wait --for=condition=Ready pod -l app=keycloak --timeout=180s
```

## Step 2: Configure Keycloak

Set up port-forward to Keycloak and create a realm and client:

```bash
kubectl -n keycloak port-forward svc/keycloak 8180:8180 &
sleep 3
```

Get an admin token:

```bash
ADMIN_TOKEN=$(curl -s http://localhost:8180/realms/master/protocol/openid-connect/token \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=admin" \
  -d "grant_type=password" | jq -r '.access_token')

echo "Admin token obtained: ${ADMIN_TOKEN:0:20}..."
```

Create an `agentgateway` realm:

```bash
curl -s -X POST http://localhost:8180/admin/realms \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "realm": "agentgateway",
    "enabled": true
  }'
echo "Realm created"
```

Create a client for agents:

```bash
curl -s -X POST http://localhost:8180/admin/realms/agentgateway/clients \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "clientId": "mcp-agent",
    "enabled": true,
    "clientAuthenticatorType": "client-secret",
    "secret": "agent-secret-123",
    "directAccessGrantsEnabled": true,
    "serviceAccountsEnabled": true
  }'
echo "Client created"
```

Create a test user:

```bash
curl -s -X POST http://localhost:8180/admin/realms/agentgateway/users \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "agent-user",
    "enabled": true,
    "credentials": [{"type": "password", "value": "agent-pass", "temporary": false}]
  }'
echo "User created"
```

## Step 3: Create Authentication Policy

Now configure AgentGateway to require OAuth for the fetch MCP route:

```bash
cat > /root/mcp-auth-policy.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: mcp-oauth-auth
  namespace: agentgateway-system
spec:
  targetRefs:
  - kind: HTTPRoute
    name: mcp-route
  auth:
    oauth:
      issuerUrl: http://keycloak.keycloak.svc.cluster.local:8180/realms/agentgateway
      clientId: mcp-agent
YAML

kubectl apply -f /root/mcp-auth-policy.yaml
```

## Step 4: Test Authentication 🧪

**Without a token** (should be rejected ❌):

```bash
curl -s -w "\nHTTP Status: %{http_code}\n" http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

**Get an access token from Keycloak**:

```bash
TOKEN=$(curl -s http://localhost:8180/realms/agentgateway/protocol/openid-connect/token \
  -d "client_id=mcp-agent" \
  -d "client_secret=agent-secret-123" \
  -d "username=agent-user" \
  -d "password=agent-pass" \
  -d "grant_type=password" | jq -r '.access_token')

echo "Agent token: ${TOKEN:0:30}..."
```

**With a valid token** (should work ✅):

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

🎉 **MCP endpoints are now secured with OAuth!**

## Summary 🏆

In this track, you've built a complete MCP security stack:

| Layer | What | Resource |
|-------|------|----------|
| 🔌 Routing | Direct MCP traffic to the right server | `HTTPRoute` + `AgentgatewayBackend` |
| 🌐 Federation | Multiple MCP servers, one gateway | Path-based `HTTPRoute` matching |
| 🔒 Authorization | Control which tools agents can call | `AgentgatewayPolicy` with `toolAuth` |
| ⏱️ Rate Limiting | Prevent runaway agents | `AgentgatewayPolicy` with `rateLimit` |
| 🔑 Authentication | Verify agent identity | `AgentgatewayPolicy` with `auth.oauth` |

**AgentGateway gives you enterprise-grade security for AI agent tool access — all on Kubernetes, all using the Gateway API.**

Want to learn more? Visit [agentgateway.dev](https://agentgateway.dev) 🚀
