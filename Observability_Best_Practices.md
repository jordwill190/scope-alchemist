# Observability Best Practices 2025  

### 1. The Golden Rule
> **If it moves, instrument it. If it doesn’t move, instrument it anyway.**

### 2. The 4 Golden Signals + 1 (2025 version)
| Signal | Critical / Pri-1 | Pri-2 / Pri-3 | Tool |
|--------|------------------|---------------|------|
| Latency | p50 / p95 / p99.9 | p50 / p99 | Datadog APM / OpenTelemetry |
| Traffic | requests/sec + 4xx/5xx split by route | requests/sec | NGINX → Prometheus |
| Errors | error % + semantic errors (e.g. `business_error`) | error % | Sentry + custom tags |
| Saturation | CPU / memory / connection pool / queue depth | CPU / memory | cAdvisor → Prometheus |
| **Business** | $ processed / claims submitted / logins | — | Custom OTel semantic conventions |

### 3. One Dashboard to Rule Them All (save as “FIREWALL”)
```yaml
# Datadog dashboard JSON – paste at https://app.datadoghq.com/dashboard/new
{
  "title": "FIREWALL – All Critical & Pri-1 Services",
  "widgets": [
    { "type": "toplist", "query": "avg:system.cpu.user{service:*} by {service}" },
    { "type": "timeseries", "query": "sum:trace.servlet.request.errors{service:payment} by {error_type}.as_rate()" },
    { "type": "heatmap", "query": "p99:trace.servlet.request.duration{service:checkout} by {region}" },
    { "type": "query_value", "query": "100 - avg:error_budget.remaining{service:*} by {service}" }
  ]
}
```
Pin this to every TV in the office. If anything is red → drop everything.

### 4. The 2025 Instrumentation Standard (enforce in CI)
```yaml
# .otelcol/config.yaml – one collector per cluster
receivers:
  otlp:
    protocols: { grpc: { endpoint: 0.0.0.0:4317 }, http: { endpoint: 0.0.0.0:4318 } }
  hostmetrics: { collection_interval: 15s }
  prometheus: { config: { scrape_configs: [ { job_name: k8s, kubernetes_sd_configs: [{role: pod}] } ] } }

processors:
  batch: { send_batch_size: 8192 }
  memory_limiter: { limit_percentage: 75 }
  resourcedetection: { detectors: [env, gcp, aws, azure] }

exporters:
  prometheusremotewrite:
    endpoint: "https://prometheus-prod-10-prod-us-east-0.grafana.net/api/prom/push"
    headers: { Authorization: "Bearer YOUR_ID:YOUR_KEY" }
  datadog:
    api: { key: "${DD_API_KEY}", site: "datadoghq.com" }

service:
  pipelines:
    traces: { receivers: [otlp], processors: [memory_limiter,batch], exporters: [datadog] }
    metrics: { receivers: [otlp,hostmetrics,prometheus], exporters: [prometheusremotewrite,datadog] }
    logs: { receivers: [otlp], exporters: [datadog] }
```

### 5. Mandatory Semantic Conventions (fail PR if missing)
```java
// Java – Spring Boot 3 + Micrometer
@Timed(value = "checkout.process", percentile = {0.95, 0.99, 0.999},
      extraTags = {"priority", "critical", "customer_tier", "platinum"})

Span.current().setAttribute("business.transaction_id", orderId)
          .setAttribute("business.amount_usd", amount)
          .setAttribute("business.error", "INSUFFICIENT_FUNDS"); // not 500!
```

### 6. Alerting Hierarchy That Doesn’t Wake You at 3 AM
| Priority | Alert | Paging | Example |
|---------|-------|--------|--------|
| Critical | Error budget burn >30% in 30 min | PagerDuty → on-call + VP | `error_budget_burn{service="payment"} > 0.3` |
| Pri-1    | p99 latency > 800 ms for 5 min | PagerDuty → on-call | `p99:trace.servlet.request.duration{service="login"} > 0.8` |
| Pri-2    | 5xx > 0.5% for 15 min | Slack #alerts-prod | ticket auto-created |
| Pri-3    | Anything | Email digest 9 AM | — |

### 7. The “No Human Ever Looks Here” Rule
- Logs → only for postmortems (ship via OTLP, never parse with grep in prod)
- Metrics → 15s scrape interval minimum
- Traces → 100% sampling for Critical/Pri-1, 5% for everything else (use Tail Sampling)

### 8. Weekly 15-Minute Observability Review (add to calendar)
1. Who burned error budget this week?  
2. Which alert fired >3 times? → mute or fix  
3. Did we ship any service without OTel? → revert

### 9. One Command Every Engineer Must Memorize
```bash
# “Why is login slow right now?”
ddtrace status && \
datadog-ci trace search --query 'service:login @http.status_code:5* duration:>2s' --from now-15m
