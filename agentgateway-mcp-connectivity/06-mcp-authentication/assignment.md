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

Agentgateway supports **MCP-native authentication** via the `backend.mcp.authentication` policy. We'll use **Keycloak** as the identity provider (IdP), which was deployed during challenge setup.

## The OAuth Flow for MCP

```
1. Agent → Gateway: "I want to call tools"
2. Gateway → Agent: "Authenticate first" (401)
3. Agent → Keycloak: "Here are my credentials"
4. Keycloak → Agent: "Here's your access token"
5. Agent → Gateway: "Here's my token + tool call"
6. Gateway validates JWT via Keycloak's JWKS endpoint
7. Gateway → MCP Server: (forwards the call)
```

## Step 1: Verify Keycloak is Running

Keycloak was deployed during challenge setup. Let's verify:

```bash
kubectl -n keycloak get pods
kubectl -n keycloak get svc keycloak
```

Set up a port-forward so we can interact with Keycloak:

```bash
kubectl -n keycloak port-forward svc/keycloak 8180:8080 &
sleep 3
```

Test that Keycloak is accessible and the `agentgateway` realm exists:

```bash
curl -s http://localhost:8180/realms/agentgateway | jq .realm
```

You should see `"agentgateway"`.

## Step 2: Test Getting a Token 🎫

A client (`mcp-agent`) and user (`agent-user`) have been pre-configured. Let's get a token:

```bash
TOKEN=$(curl -s http://localhost:8180/realms/agentgateway/protocol/openid-connect/token \
  -d "client_id=mcp-agent" \
  -d "client_secret=agent-secret-123" \
  -d "username=agent-user" \
  -d "password=agent-pass" \
  -d "grant_type=password" | jq -r '.access_token')

echo "Token obtained: ${TOKEN:0:50}..."
```

You can decode this JWT at [jwt.io](https://jwt.io) to see the claims.

## Step 3: Create the AgentgatewayBackend for Keycloak

The authentication policy needs to fetch JWKS keys from Keycloak. We need an `AgentgatewayBackend` that points to the Keycloak service so the gateway can reach it:

```bash
kubectl apply -f- <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: keycloak
  namespace: keycloak
spec:
  static:
    host: keycloak.keycloak.svc.cluster.local
    port: 8080
EOF
```

> **Why is this needed?** The gateway needs to fetch Keycloak's public keys (JWKS) to validate JWT tokens. The `backendRef` in the auth policy tells it where to find the JWKS endpoint.

## Step 4: Create the MCP Authentication Policy 🛡️

Now configure Agentgateway to require JWT authentication on the MCP route:

```bash
cat > /root/mcp-auth-policy.yaml << 'YAML'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: mcp-oauth-auth
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: mcp-route
  backend:
    mcp:
      authentication:
        issuer: "http://keycloak.keycloak.svc.cluster.local:8080/realms/agentgateway"
        audiences:
          - "mcp-agent"
        jwks:
          backendRef:
            name: keycloak
            namespace: keycloak
            port: 8080
          jwksPath: "/realms/agentgateway/protocol/openid-connect/certs"
        mode: Strict
        provider: Keycloak
        resourceMetadata:
          resource: "http://mcp-server.default.svc.cluster.local"
          scopesSupported:
            - "openid"
          bearerMethodsSupported:
            - "header"
YAML

kubectl apply -f /root/mcp-auth-policy.yaml
```

Key configuration:
- **`issuer`** — must match the `iss` claim in JWT tokens (Keycloak realm URL)
- **`audiences`** — required `aud` claim in the token (`mcp-agent` client ID)
- **`jwks.backendRef`** — points to the Keycloak `AgentgatewayBackend` we created
- **`jwks.jwksPath`** — the path to Keycloak's JWKS endpoint
- **`mode: Strict`** — all requests must have a valid token

## Step 5: Test Authentication 🧪

**Without a token** (should be rejected ❌):

```bash
curl -s -w "\nHTTP Status: %{http_code}\n" http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

You should get a `401 Unauthorized` — the gateway rejects unauthenticated requests!

**With a valid token** (should work ✅):

```bash
# Get a fresh token
TOKEN=$(curl -s http://localhost:8180/realms/agentgateway/protocol/openid-connect/token \
  -d "client_id=mcp-agent" \
  -d "client_secret=agent-secret-123" \
  -d "username=agent-user" \
  -d "password=agent-pass" \
  -d "grant_type=password" | jq -r '.access_token')

curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .
```

You should see the tools listed — authenticated access works! 🎉

**Call a tool with authentication:**

```bash
curl -s http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"fetch","arguments":{"url":"https://example.com"}},"id":2}' | jq .
```

## Summary 🏆

In this track, you've built a complete MCP security stack:

| Layer | What | Resource |
|-------|------|----------|
| 🔌 Routing | Direct MCP traffic to the right server | `HTTPRoute` + `AgentgatewayBackend` |
| 🌐 Federation | Multiple MCP servers, one gateway | Path-based `HTTPRoute` matching |
| 🔒 Authorization | Control which tools agents can call | `AgentgatewayPolicy` with `backend.mcp.authorization` |
| ⏱️ Rate Limiting | Prevent runaway agents | `AgentgatewayPolicy` with `traffic.rateLimit` |
| 🔑 Authentication | Verify agent identity via OAuth | `AgentgatewayPolicy` with `backend.mcp.authentication` |

**Key takeaway:** Every policy is applied at the gateway layer — no changes to the MCP servers or agents. Add security incrementally without touching application code.

Want to learn more? Visit [agentgateway.dev](https://agentgateway.dev) 🚀
