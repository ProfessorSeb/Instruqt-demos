---
slug: install-openclaw
id: y9rcfluxngzm
type: challenge
title: Observe Your Gateway — Logs, Metrics, and Traffic Visibility
teaser: A gateway is only as good as its visibility. Learn how to inspect what's flowing
  through AgentGateway in real time.
notes:
- type: text
  contents: "# \U0001F50D Visibility Is the Whole Point\n\nYou've built a gateway.
    But a gateway without observability is just a\nrouting hop — it's not giving you
    the governance you need.\n\nThe reason we proxy AI traffic is so we can **see**
    it:\n\n- Which models are being called?\n- How often?\n- By what?\n- Did any calls
    fail?\n- Is there unusual traffic?\n\nWithout a gateway, you're flying blind.
    With AgentGateway, every\nrequest leaves a trace.\n\nIn this challenge, you'll
    inspect what's flowing through your gateway\nusing Kubernetes-native tooling —
    no external observability stack needed."
tabs:
- id: zsxwxh8nrfr1
  title: Terminal
  type: terminal
  hostname: server
- id: addbb2eg4snn
  title: Editor
  type: code
  hostname: server
  path: /root
difficulty: basic
timelimit: 900
enhanced_loading: null
---

# Observe Your Gateway

## View Live Gateway Logs

The gateway logs every request. Watch them in real time:

```bash
kubectl logs -n agentgateway-system deployment/agentgateway-proxy --tail=50
```

Send a test request in another terminal tab, then check the logs again:

```bash
curl -s -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello from the gateway!"}]}' \
  | jq '.choices[0].message.content'
```

## Inspect the Control Plane State

Check the full state of your gateway configuration:

```bash
# All registered AI backends
kubectl get agentgatewaybackend -n agentgateway-system -o wide

# Active routing rules
kubectl get httproute -n agentgateway-system

# The gateway resource itself
kubectl describe gateway agentgateway-proxy -n agentgateway-system
```

## Check Pod Health

Verify the gateway pod is healthy:

```bash
kubectl get pods -n agentgateway-system

# Detailed status
kubectl describe pod -n agentgateway-system \
  -l app.kubernetes.io/name=agentgateway | grep -A5 "Conditions:"
```

## Save Logs to a File

Capture recent traffic for analysis:

```bash
kubectl logs -n agentgateway-system deployment/agentgateway-proxy \
  --tail=100 > /tmp/gateway-logs.txt

echo "Log lines captured: $(wc -l < /tmp/gateway-logs.txt)"
cat /tmp/gateway-logs.txt | tail -20
```

## Make One More Test Request

Confirm end-to-end health:

```bash
curl -s -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Summarise what AgentGateway does in one sentence."}]}' \
  | jq '.choices[0].message.content'
```

> 💡 **What you're seeing:** This is the audit trail for your AI traffic.
> Every call to every model, logged, inspectable, and governable — all from
> standard Kubernetes tooling you already know.
