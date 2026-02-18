---
slug: api-key-management
type: challenge
title: API Key Management
teaser: Replace raw LLM provider keys with org-specific vanity API keys.
notes:
- type: text
  contents: "# 🔑 API Key Management\n\nYou don't want every developer using the
    raw OpenAI API key. Instead, issue **vanity API keys** per team:\n\n- Teams use
    their own key (e.g., `team1-key`)\n- The gateway validates it and swaps in the
    real provider key\n- You can track usage per team and revoke access instantly\n-
    Additional headers (like `x-org`) get injected for observability\n\nNo more shared
    API keys. No more credential sprawl.\n"
tabs:
- title: Terminal
  type: terminal
  hostname: server
- title: Code Editor
  type: code
  hostname: server
  path: /root
- title: Grafana
  type: service
  hostname: server
  port: 3000
difficulty: ""
enhanced_loading: null
---

# API Key Management

Right now, anyone who can reach the gateway can send requests. Let's fix that by requiring org-specific API keys.

## Step 1: Ensure the OpenAI Route Exists

The route from the previous challenge should still be active. Verify:

```bash
kubectl get httproute openai -n enterprise-agentgateway
```

If it's not there, recreate it:

```bash
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

## Step 2: Create the API Key, AuthConfig, and Policy

This creates:
1. A Kubernetes secret holding the vanity API key (`team1-key`) and an `x-org` header to inject
2. An `AuthConfig` that validates the key and injects the org header
3. An `EnterpriseAgentgatewayPolicy` that enforces auth on the Gateway

```bash
kubectl apply -f- <<EOF
apiVersion: v1
data:
  api-key: dGVhbTEta2V5
  x-org: ZGV2ZWxvcGVycw==
kind: Secret
metadata:
  labels:
    llm-provider: openai
  name: team1-apikey
  namespace: enterprise-agentgateway
type: extauth.solo.io/apikey
---
apiVersion: extauth.solo.io/v1
kind: AuthConfig
metadata:
  name: apikey-auth
  namespace: enterprise-agentgateway
spec:
  configs:
    - apiKeyAuth:
        headerName: vanity-auth
        k8sSecretApikeyStorage:
          apiKeySecretRefs:
            - name: team1-apikey
              namespace: enterprise-agentgateway
        headersFromMetadataEntry:
          x-org:
            name: x-org
---
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: api-key-auth
  namespace: enterprise-agentgateway
spec:
  targetRefs:
    - name: agentgateway
      group: gateway.networking.k8s.io
      kind: Gateway
  traffic:
    entExtAuth:
      authConfigRef:
        name: apikey-auth
        namespace: enterprise-agentgateway
EOF
```

## Step 3: Test Without an API Key (Should Fail)

```bash
source /root/.bashrc

curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

You should get a **401 Unauthorized** response. The gateway now requires authentication.

## Step 4: Test With the Vanity API Key (Should Succeed)

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "vanity-auth: team1-key" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello from team1!"}]
  }' | jq .
```

The request succeeds. The gateway:
1. Validated `team1-key` against the secret
2. Injected the `x-org: developers` header
3. Swapped in the real OpenAI API key
4. Forwarded the request

## Step 5: Check the Access Logs

```bash
kubectl logs deploy/agentgateway -n enterprise-agentgateway --tail 1 | jq .
```

Notice the `x-org` header is captured in the logs — this gives you per-team attribution.

## Cleanup

```bash
kubectl delete enterpriseagentgatewaypolicy -n enterprise-agentgateway api-key-auth
kubectl delete authconfig -n enterprise-agentgateway apikey-auth
kubectl delete secret -n enterprise-agentgateway team1-apikey
```

## ✅ What You've Learned

- `AuthConfig` (extauth.solo.io) defines API key validation rules
- `EnterpriseAgentgatewayPolicy` enforces auth on the Gateway
- Vanity keys decouple team access from provider credentials
- Header injection enables per-team tracking in logs and metrics

**Next up:** JWT Authentication & RBAC — for fine-grained identity-based access control.
