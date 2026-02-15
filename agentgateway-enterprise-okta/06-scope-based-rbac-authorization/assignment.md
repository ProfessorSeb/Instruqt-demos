---
slug: scope-based-rbac-authorization
id: 9vbpfq7fmbki
type: challenge
title: Scope-Based RBAC Authorization
teaser: Build CEL-based authorization policies that check JWT scopes for fine-grained
  access control.
notes:
- type: text
  contents: "# Scope-Based RBAC \U0001F6E1️\n\nAuthentication tells you WHO someone
    is. **Authorization** tells you WHAT they can do.\n\nYou'll use **CEL expressions**
    (Common Expression Language) to inspect JWT claims and enforce scope-based access
    control. This is how you build the \"On Behalf Of\" pattern — where AI agents
    act with the user's identity and permissions."
tabs:
- id: xtdbpfyxsc3m
  title: Terminal
  type: terminal
  hostname: server
- id: snokp55ckqct
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: yfagyptuurmk
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: 0rvxzrw0wybx
  title: MCP Inspector
  type: service
  hostname: server
  port: 6274
difficulty: intermediate
timelimit: 900
enhanced_loading: null
---

# Scope-Based RBAC Authorization

## Step 1: Add an Authorization Policy

Create `/root/okta-rbac.yaml` that checks for `mcp:read` or `mcp:write` scopes:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: okta-rbac
  namespace: enterprise-agentgateway
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: agentgateway
  traffic:
    authorization:
      policy:
        matchExpressions:
        - 'jwt.scp.exists(s, s == "mcp:read") || jwt.scp.exists(s, s == "mcp:write")'
```

Apply it:

```bash
kubectl apply -f /root/okta-rbac.yaml
```

**What this does:**
- Uses a **CEL expression** to inspect the JWT's `scp` (scopes) claim
- Allows access if the token has EITHER `mcp:read` OR `mcp:write` scope
- Combined with the JWT auth policy, this means: valid token + correct scopes = access

## Step 2: Test with Valid Scopes

Get a token with the right scopes and test:

```bash
export TOKEN=$(curl -s -X POST \
  "https://integrator-7147223.okta.com/oauth2/aus104zseyg64swj3698/v1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$OKTA_SERVICE_CLIENT_ID" \
  -d "client_secret=$OKTA_SERVICE_CLIENT_SECRET" \
  -d "scope=mcp:read mcp:write" | jq -r '.access_token')

curl -s "localhost:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"What scopes do I have?"}]}' | jq
```

This should succeed — the token has the required scopes! ✅

## Step 3: Tighten the Policy

Now update the policy to require an `mcp:admin` scope that the token does NOT have:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: okta-rbac
  namespace: enterprise-agentgateway
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: agentgateway
  traffic:
    authorization:
      policy:
        matchExpressions:
        - 'jwt.scp.exists(s, s == "mcp:admin")'
```

Apply the updated policy:

```bash
kubectl apply -f /root/okta-rbac.yaml
```

## Step 4: Test Again — Should Fail

```bash
curl -i "localhost:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Am I admin?"}]}'
```

You should get a **403 Forbidden** — the token has `mcp:read` and `mcp:write` but NOT `mcp:admin`. 🔒

## Step 5: Check Traces and Logs

Check Grafana for traces that include JWT claims:

1. Open the **Grafana** tab → **Explore** → **Tempo**
2. Search for recent traces — you'll see JWT claims attached to each span

View access logs to see the authorization decisions:

```bash
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail 5
```

## 🎉 Congratulations!

You've built a complete identity-aware AI gateway! Here's the full chain:

```
User → Okta (authenticate) → JWT (with scopes) → AgentGateway → validates JWT → checks RBAC → LLM/MCP
```

**Key takeaways:**
- **Authentication** (JWT validation) ensures only known identities can access the gateway
- **Authorization** (CEL expressions) enforces fine-grained access based on token claims
- **"On Behalf Of" pattern** — AI agents act with the user's identity and permissions
- **Scope-based gating** — `mcp:read` for listing tools, `mcp:write` for calling them, `mcp:admin` for dangerous operations
- **Full observability** — every request is traced with identity context in Grafana

Click **Check** to complete the workshop!
