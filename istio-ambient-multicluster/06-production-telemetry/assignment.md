---
slug: production-telemetry
id: 9thh8ktlbfct
type: challenge
title: Production Telemetry
teaser: Configure custom telemetry with tag overrides for production observability.
tabs:
- id: qsxbmaa9lxzn
  title: Terminal
  type: terminal
  hostname: server
- id: r8rzbkydwchw
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Production Telemetry

## Beyond Default Metrics

Istio generates metrics out of the box — request count, latency, error rates. But in production, you often need **custom tags** to slice data by business dimensions: error types, SLO categories, deployment rings.

The `Telemetry` API lets you add custom tag overrides to Istio's metrics without modifying application code.

## Your Tasks

### 1. Apply a Telemetry CR

Create a Telemetry resource with custom tag overrides:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: accounts-telemetry
  namespace: accounts
spec:
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: REQUEST_COUNT
        mode: SERVER
      tagOverrides:
        error_type:
          operation: UPSERT
          value: "response.code >= 500 ? 'server_error' : response.code >= 400 ? 'client_error' : 'none'"
        slow_request:
          operation: UPSERT
          value: "request.duration > '500ms' ? 'true' : 'false'"
EOF
```

### 2. Generate Some Traffic

Send a burst of requests so metrics are populated:

```bash
for i in $(seq 1 20); do
  kubectl exec curl -n finance -- curl -sS accounts.accounts:8080 > /dev/null 2>&1
done
echo "Traffic sent!"
```

### 3. Check Metrics

Query the waypoint's Prometheus metrics endpoint to see the custom tags:

```bash
WAYPOINT_POD=$(kubectl get pods -n accounts -l gateway.networking.k8s.io/gateway-name=waypoint -o jsonpath='{.items[0].metadata.name}')
kubectl exec $WAYPOINT_POD -n accounts -- curl -sS localhost:15020/stats/prometheus | grep istio_requests_total | head -5
```

Look for `error_type` and `slow_request` tags in the output.

### 4. Understand Cardinality

> ⚠️ **Cardinality Warning:** Every unique combination of tag values creates a new time series. Adding a tag with high cardinality (like user IDs) can explode your metrics storage. Stick to low-cardinality dimensions: error categories, boolean flags, environment labels.

Good patterns:
- `error_type: server_error | client_error | none` (3 values)
- `slow_request: true | false` (2 values)

Bad patterns:
- `user_id: <unique per request>` (millions of values 💥)

### 5. Verify the Telemetry CR

```bash
kubectl get telemetry -n accounts
```

> 🏢 **Enterprise Value Beat:** Custom telemetry tags let SRE teams build dashboards that answer business questions — "How many 5xx errors in the payments flow?" — without touching application code. The mesh handles instrumentation; your apps stay clean.
