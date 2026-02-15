# 🎓 Agentgateway Instruqt Demos

Interactive hands-on labs for [Agentgateway](https://agentgateway.dev/) — the open-source, AI-first gateway for routing to LLMs, MCP tools, and agents on Kubernetes.

Built for the [Instruqt](https://instruqt.com/) platform by the Solo.io GTM team.

---

## 📚 Tracks

### Track 1: [Agentgateway OSS — Your First AI Gateway](https://play.instruqt.com/soloio/tracks/agentgateway-oss-quickstart)

> Deploy a purpose-built gateway for AI agent traffic on Kubernetes.

| # | Challenge | What You'll Learn |
|---|-----------|-------------------|
| 1 | The Problem — Agents Without Guardrails | Why direct agent-to-LLM calls are a problem |
| 2 | Install Agentgateway on Kubernetes | Helm install with Gateway API CRDs |
| 3 | Create Your First AI Gateway | Gateway + HTTPRoute + LLM backend |
| 4 | Add Multiple LLM Providers | Multi-provider routing (OpenAI + Anthropic) |
| 5 | See What Your Agents Are Doing | Built-in observability and metrics |
| 6 | What's Next — Security & Governance | Overview of enterprise capabilities |

**Time:** ~45 min · **Level:** Beginner · **Prerequisites:** Basic Kubernetes (kubectl, pods, services)

---

### Track 2: [Securing AI Agents — PII, Prompt Injection & Rate Limiting](https://play.instruqt.com/soloio/tracks/agentgateway-security-policies)

> Protect your AI agent traffic with gateway-level security policies.

| # | Challenge | What You'll Learn |
|---|-----------|-------------------|
| 1 | The Security Gap | What happens when agents run without guardrails |
| 2 | PII Protection | Stop sensitive data from reaching LLMs |
| 3 | Prompt Injection Guard | Block jailbreak attempts at the gateway |
| 4 | Credential Leak Prevention | Keep secrets out of LLM responses |
| 5 | Rate Limiting | Control AI spend with request/token limits |
| 6 | Defense in Depth | Combine all policies into layered security |

**Time:** ~45 min · **Level:** Intermediate · **Prerequisites:** Track 1 or Agentgateway basics

---

### Track 3: [Agentgateway MCP — Secure Tool Access for AI Agents](https://play.instruqt.com/soloio/tracks/agentgateway-mcp-connectivity)

> Connect AI agents to external tools securely using MCP and Agentgateway.

| # | Challenge | What You'll Learn |
|---|-----------|-------------------|
| 1 | Why MCP? 🤔 | The tool integration problem MCP solves |
| 2 | Static MCP Routing 🎯 | Deploy an MCP server, route through Agentgateway |
| 3 | MCP Federation 🌐 | Multi-server routing with path-based matching |
| 4 | MCP Tool Authorization 🔒 | CEL-based policies to control tool access |
| 5 | MCP Rate Limiting ⏱️ | Prevent runaway agents from hammering APIs |
| 6 | MCP Authentication with OAuth 🔑 | Keycloak integration for authenticated tool access |

**Time:** ~60 min · **Level:** Intermediate · **Prerequisites:** Track 1 or Agentgateway basics

---

### Track 4: [Enterprise Agentgateway + Okta: Identity-Aware AI Agents](https://play.instruqt.com/soloio/tracks/agentgateway-enterprise-okta)

> Build a production-grade AI gateway with real OAuth authentication, tracing, and scope-based authorization.

| # | Challenge | What You'll Learn |
|---|-----------|-------------------|
| 1 | Explore the Enterprise Environment | Enterprise CRDs, monitoring stack, GatewayClass |
| 2 | Configure the Gateway with Tracing | EnterpriseAgentgatewayParameters + OpenTelemetry to Grafana/Tempo |
| 3 | Route Real LLM Traffic to OpenAI | AgentgatewayBackend + HTTPRoute with real API responses |
| 4 | Connect an MCP Tool Server | Deploy MCP server, test with MCP Inspector |
| 5 | Secure with Okta JWT Authentication | Dynamic JWKS validation with Okta as IdP |
| 6 | Scope-Based RBAC Authorization | CEL expressions on JWT claims (mcp:read/write/admin) |

**Time:** ~90 min · **Level:** Advanced · **Prerequisites:** Tracks 1-3 recommended, basic OAuth/JWT understanding

---

## 🏗️ Architecture

All tracks run on Instruqt with:
- **VM:** Solo.io pre-baked GCP image with k3d, kubectl, helm pre-installed
- **Cluster:** k3d single-node Kubernetes cluster
- **Helm Charts:** OCI-based from `ghcr.io/kgateway-dev/charts/`
- **No API keys required** — all demos use mock LLM servers and MCP servers

## 🔗 Resources

- [Agentgateway Docs](https://agentgateway.dev/)
- [Agentgateway GitHub](https://github.com/agentgateway/agentgateway)
- [Solo.io](https://solo.io)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)

## 📝 License

These demos are maintained by the Solo.io GTM team. See individual track directories for content.
