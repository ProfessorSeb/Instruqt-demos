---
slug: jwt-auth-rbac
id: 1xtozeho0e1r
type: challenge
title: JWT Authentication & RBAC
teaser: Enforce identity-based access with JWT validation and CEL-based authorization.
notes:
- type: text
  contents: "# \U0001F6E1️ JWT Authentication & RBAC\n\nAPI keys are great for simple
    use cases. But enterprises need **identity-based access control**:\n\n- Validate
    JWT tokens from your identity provider\n- Extract user/team/org claims\n- Enforce
    RBAC rules using CEL expressions\n- Log identity with every request for audit
    trails\n\n AgentGateway supports both inline JWKS and remote JWKS (like Okta,
    Auth0).\n"
tabs:
- id: ywftt61szlme
  title: Terminal
  type: terminal
  hostname: server
- id: 7cgldtuzl8pg
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: xgstfhqxoqks
  title: Grafana
  type: service
  hostname: server
  port: 3000
difficulty: ""
enhanced_loading: null
---

# JWT Authentication & RBAC

Let's enforce JWT-based authentication with role-based access control on the gateway.

## Step 1: Clean Up Previous Auth Policy

Remove the API key auth from the previous challenge (if still active):

```bash
kubectl delete enterpriseagentgatewaypolicy api-key-auth -n enterprise-agentgateway 2>/dev/null || true
kubectl delete authconfig apikey-auth -n enterprise-agentgateway 2>/dev/null || true
kubectl delete secret team1-apikey -n enterprise-agentgateway 2>/dev/null || true
```

## Step 2: Ensure the OpenAI Route Exists

```bash
kubectl get httproute openai -n enterprise-agentgateway || \
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: enterprise-agentgateway
spec:
  parentRefs:
    - name: agentgateway
      namespace: enterprise-agentgateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai
      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend
      timeouts:
        request: "120s"
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-all-models
  namespace: enterprise-agentgateway
spec:
  ai:
    provider:
      openai: {}
  policies:
    auth:
      secretRef:
        name: openai-secret
EOF
```

## Step 3: Create JWT Auth with RBAC Policy

This policy:
1. Validates JWTs using an inline JWKS (RSA public key)
2. Enforces RBAC: only tokens with `org=solo.io` AND `team=team-id` can access the gateway

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: agentgateway-jwt-auth
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
        - issuer: solo.io
          jwks:
            inline: |
                {
                  "keys": [
                    {
                      "kty": "RSA",
                      "kid": "solo-public-key-001",
                      "n": "vlmc5pb-jYaOq75Y4r91AC2iuS9B0sm6sxzRm3oOG7nIt2F1hHd4AKll2jd6BZg437qvsLdREnbnVrr8kU0drmJNPHL-xbsTz_cQa95GuKb6AI6osAaUAEL3dPjuoqkGNRe1sAJyOi48qtcbV0kPWcwFmCV0-OiqliCms12jrd1PSI_LYiNc3GcutpxY6BiHkbxxNeIuWDxE-i_Obq8EhhGkwha1KVUvLHV-EwD4M_AY8BegGsX-sjoChXOxyueu_ReqWV227I-FTKwMnjwWW0BQkeI6g1w1WqADmtKZ2sLamwGUJgWt4ZgIyhQ-iQfeN1WN2iupTWa5JAsw--CQJw",
                      "e": "AQAB",
                      "use": "sig",
                      "alg": "RS256"
                    }
                  ]
                }
    authorization:
      policy:
        matchExpressions:
          - '(jwt.org == "solo.io") && (jwt.team == "team-id")'
EOF
```

## Step 4: Test Without a JWT (Should Fail)

```bash
source /root/.bashrc

curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

You should see `403` with `authentication failure: no bearer token found`.

## Step 5: Test With a Valid JWT (Should Succeed)

This JWT contains claims: `iss=solo.io`, `org=solo.io`, `team=team-id`:

```bash
export DEV_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6InVzZXItaWQiLCJ0ZWFtIjoidGVhbS1pZCIsImV4cCI6MjA3OTU1NjEwNCwibGxtcyI6eyJvcGVuYWkiOlsiZ3B0LTRvIl19fQ.e49g9XE6yrttR9gQAPpT_qcWVKe-bO6A7yJarMDCMCh8PhYs67br00wT6v0Wt8QXMMN09dd8UUEjTunhXqdkF5oeRMXiyVjpTPY4CJeoF1LfKhgebVkJeX8kLhqBYbMXp3cxr2GAmc3gkNfS2XnL2j-bowtVzwNqVI5D8L0heCpYO96xsci37pFP8jz6r5pRNZ597AT5bnYaeu7dHO0a5VGJqiClSyX9lwgVCXaK03zD1EthwPoq34a7MwtGy2mFS_pD1MTnPK86QfW10LCHxtahzGHSQ4jfiL-zp13s8MyDgTkbtanCk_dxURIyynwX54QJC_o5X7ooDc3dxbd8Cw"

curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What claims are in my JWT?"}]
  }' | jq .
```

The request succeeds because the JWT has the required `org` and `team` claims.

## Step 6: Inspect the JWT

If you decode the JWT at [jwt.io](https://jwt.io), you'll see:

```json
{
  "iss": "solo.io",
  "org": "solo.io",
  "sub": "user-id",
  "team": "team-id",
  "exp": 2079556104,
  "llms": {
    "openai": ["gpt-4o"]
  }
}
```

## Step 7: Experiment with RBAC Rules

Try changing the CEL expression to deny access. Edit the policy to require `org=internal`:

```bash
kubectl patch enterpriseagentgatewaypolicy agentgateway-jwt-auth -n enterprise-agentgateway \
  --type=merge -p '{"spec":{"traffic":{"authorization":{"policy":{"matchExpressions":["(jwt.org == \"internal\")"]}}}}}'
```

Now the same JWT should be rejected (org is `solo.io`, not `internal`):

```bash
curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "test"}]}'
```

## Step 8: Check Access Logs

```bash
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail 2 | jq '.jwt // .message // empty'
```

JWT claims are captured in access logs for audit trails.

## Cleanup

```bash
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway agentgateway-jwt-auth
```

## ✅ What You've Learned

- `jwtAuthentication` validates tokens using JWKS (inline or remote)
- `authorization.policy.matchExpressions` uses CEL for RBAC rules
- JWT claims are available in access logs and traces
- You can enforce org, team, role, or any custom claim

**Next up:** Prompt Enrichment & Guardrails — protect what goes into and comes out of LLMs.
