---
slug: secure-with-okta-jwt-authentication
id: xtaqh8lfmbng
type: challenge
title: Secure with Okta JWT Authentication
teaser: Integrate Okta as your identity provider with dynamic JWKS validation and
  strict JWT enforcement.
notes:
- type: text
  contents: "# Identity-Aware Gateway \U0001F511\n\nYour gateway is routing LLM and
    MCP traffic — but anyone can access it! In production, you need to know WHO is
    calling your agents.\n\nYou'll integrate **Okta** as the identity provider using:\n-
    A **JWKS backend** that dynamically fetches Okta's public signing keys\n- An **EnterpriseAgentgatewayPolicy**
    with strict JWT authentication\n- Real **OAuth2 client_credentials** flow to obtain
    tokens\n\nAfter this, every request must carry a valid Okta JWT."
tabs:
- id: xcrbqqgqbuen
  title: Terminal
  type: terminal
  hostname: server
- id: rt91g9amsy7k
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: p4jmnxweevqj
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: 9alsyski05ls
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: intermediate
timelimit: 900
enhanced_loading: null
---

# Secure with Okta JWT Authentication

## Step 1: Create the Okta JWKS Backend

AgentGateway needs to fetch Okta's public keys to validate JWTs. Create `/root/okta-jwks.yaml`:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: okta-jwks
  namespace: enterprise-agentgateway
spec:
  static:
    host: integrator-7147223.okta.com
    port: 443
  policies:
    tls: {}
```

Apply it:

```bash
kubectl apply -f /root/okta-jwks.yaml
```

This creates a backend that points to Okta's HTTPS endpoint. The `tls: {}` ensures TLS is used for the connection.

## Step 2: Create the JWT Authentication Policy

Now create the policy that enforces JWT validation. Create `/root/okta-jwt-auth.yaml`:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: okta-jwt-auth
  namespace: enterprise-agentgateway
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: agentgateway
  traffic:
    jwtAuthentication:
      mode: Strict
      providers:
      - issuer: https://integrator-7147223.okta.com/oauth2/aus104zseyg64swj3698
        jwks:
          remote:
            backendRef:
              name: okta-jwks
              namespace: enterprise-agentgateway
              kind: AgentgatewayBackend
              group: agentgateway.dev
            jwksPath: /oauth2/aus104zseyg64swj3698/v1/keys
```

Apply it:

```bash
kubectl apply -f /root/okta-jwt-auth.yaml
```

**What this does:**
- **mode: Strict** — ALL requests must have a valid JWT (no anonymous access)
- **issuer** — only tokens from this specific Okta authorization server are accepted
- **jwks.remote** — dynamically fetches Okta's public keys via the JWKS backend
- **jwksPath** — the specific path on Okta to fetch the JWKS

## Step 3: Test Without a Token

Try your previous curl command — it should now be rejected:

```bash
curl -i "localhost:8080/openai" \
  -H "content-type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'
```

You should see a **403 Forbidden** response. The gateway is now locked down! 🔒

## Step 4: Get a Token from Okta

The Okta credentials are pre-configured in your environment. Get a token using the OAuth2 client_credentials flow:

```bash
export TOKEN=$(curl -s -X POST \
  "https://integrator-7147223.okta.com/oauth2/aus104zseyg64swj3698/v1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$OKTA_SERVICE_CLIENT_ID" \
  -d "client_secret=$OKTA_SERVICE_CLIENT_SECRET" \
  -d "scope=mcp:read mcp:write" | jq -r '.access_token')
echo $TOKEN
```

## Step 5: Test With a Token

Now send the same request with the JWT:

```bash
curl -s "localhost:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello from an authenticated agent!"}]}' | jq
```

Success! The request goes through because the JWT is valid. 🎉

## Step 6: Decode the Token

Inspect what claims Okta put in the token:

```bash
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq
```

Notice the `scp` (scopes) claim — it contains `mcp:read` and `mcp:write`. We'll use these for RBAC in the next challenge.

Click **Check** when your JWT authentication is working.
