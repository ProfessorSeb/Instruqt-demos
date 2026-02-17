---
slug: whats-next
id: yss5ivalys9m
type: challenge
title: What's Next — Security & Governance
teaser: Discover what Agentgateway Enterprise adds for production AI agent deployments.
notes:
- type: text
  contents: "# \U0001F5FA️ What's Next — Security & Governance\n\nYou've built a working
    AI gateway with multi-provider routing and observability.\n\n**In this final challenge,
    you'll:**\n\n- Recap everything you built\n- Learn about Enterprise security features\n-
    Explore MCP gateway capabilities\n- Get resources to continue your journey\n"
tabs:
- id: u0nkuycn4lch
  title: Terminal
  type: terminal
  hostname: server
- id: 8qsvpe3hamxk
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# What's Next: Security & Governance

Congratulations! You've built a working AI gateway that routes traffic to multiple LLM providers with full observability. That's a huge step forward from "every agent calls LLMs directly."

But for production deployments, you need more. Let's talk about what comes next.

## What You Built Today

Let's recap what you accomplished:

✅ **Identified the problem** — direct agent-to-LLM calls create security, cost, and visibility gaps

✅ **Installed Agentgateway OSS** — Kubernetes-native, built on Gateway API

✅ **Created an AI Gateway** — single entry point for all agent traffic

✅ **Model load balancing** — GPT-4o-mini and GPT-4o with weighted routing through one gateway

✅ **Observability** — structured logs and Prometheus metrics for every request

## What Agentgateway Enterprise Adds

For production deployments with strict security and compliance requirements, **Agentgateway Enterprise** adds:

### 🔒 Security
- **Prompt injection detection** — block malicious prompts before they reach the LLM
- **PII redaction** — automatically detect and mask sensitive data in prompts and responses
- **Authentication & authorization** — OIDC, JWT, API key validation per agent/team

### 💰 Cost Controls
- **Rate limiting** — per agent, per team, per model
- **Token budgets** — set daily/weekly/monthly token limits
- **Cost attribution** — track spend by team, agent, or project

### 🔧 MCP Gateway
- **MCP tool routing** — gateway for Model Context Protocol tool calls
- **Tool authorization** — control which agents can call which tools
- **Tool observability** — same logging/metrics for tool calls

### 📊 Advanced Observability
- **Langfuse integration** — full prompt/response tracing
- **OpenTelemetry export** — send traces to your existing observability stack
- **Cost dashboards** — real-time spend tracking across providers

## Create a Recap File

Summarize your learning:

```bash
cat > /root/workshop-recap.txt << 'EOF'
Agentgateway OSS Quickstart - Workshop Recap

What I learned:
1. Direct agent-to-LLM calls create security and visibility gaps
2. Agentgateway provides a Kubernetes-native gateway for AI traffic
3. Built on Gateway API standard (GatewayClass, Gateway, HTTPRoute)
4. Multi-provider routing through a single endpoint
5. Built-in observability with structured logs and Prometheus metrics

Architecture:
  Agent -> Agentgateway (Gateway API) -> LLM Providers

Key Resources:
  - GatewayClass: registers Agentgateway as an implementation
  - Gateway: the actual gateway instance (listeners)
  - Backend: LLM provider connection details + credentials
  - HTTPRoute: routing rules (path-based, header-based)

Next steps:
  - Try with real API keys
  - Explore Agentgateway Enterprise for security features
  - Set up Langfuse for full LLM observability
EOF

cat /root/workshop-recap.txt
```

## Resources

- 📖 [Agentgateway Documentation](https://docs.solo.io/agentgateway/latest/)
- 💻 [Agentgateway GitHub](https://github.com/solo-io/agentgateway)
- 🎓 [Solo.io Academy](https://academy.solo.io)
- 💬 [Solo.io Community Slack](https://slack.solo.io)
- 📧 [Contact Solo.io](https://www.solo.io/contact) — for Enterprise evaluation

## Thank You!

You've seen how Agentgateway brings the same infrastructure patterns we rely on for microservices to the world of AI agents. As agents become a bigger part of your architecture, having a purpose-built gateway isn't optional — it's essential.

**Happy gatewaying! 🚀**
