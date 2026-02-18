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
- id: solouijwt04
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# JWT Authentication & RBAC

Let's enforce JWT-based authentication with role-based access control on the gateway.

## Step 1: Clean Up Previous Auth Policy

Remove the API key auth from the previous challenge (if still active):

```bash
kubectl delete enterpriseagentgatewaypolicy api-key-auth -n agentgateway-system 2>/dev/null || true
kubectl delete authconfig apikey-auth -n agentgateway-system 2>/dev/null || true
kubectl delete secret team1-apikey -n agentgateway-system 2>/dev/null || true
```

## Step 2: Ensure the OpenAI Route Exists

```bash
kubectl get httproute openai -n agentgateway-system || \
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system
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
  namespace: agentgateway-system
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

## Step 3: Set the JWT Token

We'll use this JWT throughout the challenge. It contains claims: `iss=solo.io`, `org=solo.io`, `team=team-id`:

```bash
export DEV_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6InVzZXItaWQiLCJ0ZWFtIjoidGVhbS1pZCIsImV4cCI6MjA3OTU1NjEwNCwibGxtcyI6eyJvcGVuYWkiOlsiZ3B0LTRvIl19fQ.e49g9XE6yrttR9gQAPpT_qcWVKe-bO6A7yJarMDCMCh8PhYs67br00wT6v0Wt8QXMMN09dd8UUEjTunhXqdkF5oeRMXiyVjpTPY4CJeoF1LfKhgebVkJeX8kLhqBYbMXp3cxr2GAmc3gkNfS2XnL2j-bowtVzwNqVI5D8L0heCpYO96xsci37pFP8jz6r5pRNZ597AT5bnYaeu7dHO0a5VGJqiClSyX9lwgVCXaK03zD1EthwPoq34a7MwtGy2mFS_pD1MTnPK86QfW10LCHxtahzGHSQ4jfiL-zp13s8MyDgTkbtanCk_dxURIyynwX54QJC_o5X7ooDc3dxbd8Cw"
```

## Step 4: Create JWT Auth with RBAC Policy

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: agentgateway-jwt-auth
  namespace: agentgateway-system
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

## Step 5: Test Without a JWT (Should Fail — 403)

```bash
source /root/.bashrc

curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

## Step 6: Test With a Valid JWT (Should Succeed — 200)

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What claims are in my JWT?"}]
  }' | jq .
```

## Step 7: Experiment with RBAC Rules

Patch the policy to require `org=internal` instead:

```bash
kubectl patch enterpriseagentgatewaypolicy agentgateway-jwt-auth -n agentgateway-system \
  --type=merge -p '{"spec":{"traffic":{"authorization":{"policy":{"matchExpressions":["(jwt.org == \\"internal\\")"]}}}}}'
```

```bash
sleep 3

curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "test"}]}'
```

You should see `403 Forbidden`.

## Step 8: Restore the Original RBAC Rule

```bash
kubectl patch enterpriseagentgatewaypolicy agentgateway-jwt-auth -n agentgateway-system \
  --type=merge -p '{"spec":{"traffic":{"authorization":{"policy":{"matchExpressions":["(jwt.org == \\"solo.io\\") && (jwt.team == \\"team-id\\")"]}}}}}'
```

Verify:

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "RBAC restored!"}]
  }' | jq -r '.choices[0].message.content'
```

## Step 9: Check Access Logs

```bash
kubectl logs deploy/agentgateway -n agentgateway-system --tail 4 | jq '{status: ."http.status", jwt: .jwt}' 2>/dev/null || \
kubectl logs deploy/agentgateway -n agentgateway-system --tail 4
```

You can also explore these traces in the **Solo UI** tab.

## ✅ What You've Learned

- `jwtAuthentication` validates tokens using JWKS (inline or remote)
- `authorization.policy.matchExpressions` uses CEL for fine-grained RBAC
- JWT claims are available in access logs and traces for audit

**Next up:** Prompt Enrichment & Guardrails.
