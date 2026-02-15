# Track 4: Enterprise AgentGateway + Okta — Identity-Aware AI Agents

## Design Notes

### What's pre-built (track setup):
- k3d cluster
- Enterprise AgentGateway installed (Helm v2.1.1 from us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/)
- Gateway API CRDs (experimental channel for mTLS support)
- Monitoring stack (Tempo + Prometheus + Grafana + Loki)
- Enterprise UI + port-forward
- MCP Inspector (npx)
- Namespace: `enterprise-agentgateway`
- GatewayClass: `enterprise-agentgateway`

### What the learner builds:
1. EnterpriseAgentgatewayParameters (with tracing, logging, extauth, ratelimiter)
2. Gateway resource (with parametersRef for tracing)
3. OpenAI backend + route (with API key secret)
4. MCP server deployment + backend + route
5. JWT auth policy with Okta (EnterpriseAgentgatewayPolicy)
6. CEL-based RBAC authorization
7. MCP tool authorization with scopes

### Instruqt secrets needed:
- AGENTGATEWAY_LICENSE_KEY (Seb to create)
- OPENAI_API_KEY (existing on soloio team, or Seb updates)
- OKTA_ISSUER, OKTA_CLIENT_ID, OKTA_SERVICE_CLIENT_ID, OKTA_SERVICE_CLIENT_SECRET (Seb's Okta dev org)

### Enterprise-specific resources:
- `EnterpriseAgentgatewayParameters` (not `AgentgatewayParameters`)
- `EnterpriseAgentgatewayPolicy` (not `AgentgatewayPolicy`)
- Helm: `oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway`
- CRDs: `oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway-crds`
- GatewayClass: `enterprise-agentgateway`
- Namespace: `enterprise-agentgateway`

### Key patterns from FE workshop (solo-io/fe-enterprise-agentgateway-workshop):
- OpenAI secret: `kubectl create secret generic openai-secret --from-literal="Authorization=Bearer $OPENAI_API_KEY"`
- Backend auth: `spec.policies.auth.secretRef.name: openai-secret`
- MCP service appProtocol: `agentgateway.dev/mcp` (Enterprise uses this, not `kgateway.dev/mcp`)
- MCP backend format: `spec.mcp.targets[].static` with `protocol: SSE`
- JWT auth: `EnterpriseAgentgatewayPolicy` with `traffic.jwtAuthentication`
- RBAC: `traffic.authorization.policy.matchExpressions` with CEL
- Dynamic JWT with Okta: `AgentgatewayBackend` for JWKS endpoint + `jwks.remote.backendRef`
- Gateway with tracing: `spec.infrastructure.parametersRef` pointing to EnterpriseAgentgatewayParameters

### Tracing config in EnterpriseAgentgatewayParameters:
```yaml
spec:
  rawConfig:
    config:
      tracing:
        otlpProtocol: grpc
        otlpEndpoint: http://tempo-distributor.monitoring.svc.cluster.local:4317
        randomSampling: 'true'
        fields:
          add:
            gen_ai.operation.name: '"chat"'
            gen_ai.system: "llm.provider"
            gen_ai.prompt: 'llm.prompt'
            gen_ai.completion: 'llm.completion.map(c, {"role":"assistant", "content": c})'
            gen_ai.request.model: "llm.requestModel"
            gen_ai.response.model: "llm.responseModel"
            gen_ai.usage.completion_tokens: "llm.outputTokens"
            gen_ai.usage.prompt_tokens: "llm.inputTokens"
            jwt: 'jwt'
```

### Gateway with parametersRef:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway
  namespace: enterprise-agentgateway
spec:
  gatewayClassName: enterprise-agentgateway
  infrastructure:
    parametersRef:
      name: tracing
      group: enterpriseagentgateway.solo.io
      kind: EnterpriseAgentgatewayParameters
  listeners:
  - name: http
    port: 8080
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
```

### Okta Dynamic JWT flow:
1. Create AgentgatewayBackend for Okta JWKS endpoint (static host + TLS)
2. Create EnterpriseAgentgatewayPolicy with jwtAuthentication pointing to Okta issuer + remote JWKS
3. Create authorization policy with CEL matching Okta JWT claims (aud, scp, etc.)

### Reference Okta config (from aws-agentcore-demo):
- Issuer: https://integrator-7147223.okta.com/oauth2/aus104zseyg64swj3698
- JWKS: /oauth2/aus104zseyg64swj3698/v1/keys
- Client app (PKCE): devops-copilot-client
- Service app (client_credentials): devops-copilot-service
- Custom scopes: mcp:read, mcp:write, mcp:admin
- Test users in Okta org
