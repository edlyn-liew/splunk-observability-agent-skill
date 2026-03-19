# APM & Tracing Reference

## Official Documentation

### APM Setup & Features
- [APM Setup Guide](https://docs.splunk.com/Observability/apm/set-up-apm/apm.html)
- [Track Service Performance (RED)](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/visualize-and-alert-on-your-application-in-splunk-apm/track-service-performance-using-dashboards-in-splunk-apm)
- [Service Map](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/manage-services-spans-and-traces-in-splunk-apm/view-dependencies-in-the-service-map)
- [Trace Analyzer](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/manage-services-spans-and-traces-in-splunk-apm/search-traces-using-trace-analyzer)
- [Analyze Error Spans](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/manage-services-spans-and-traces-in-splunk-apm/analyze-error-spans)
- [Span Tags](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/analyze-services-with-span-tags-and-metricsets/add-context-to-spans-with-span-tags-in-splunk-apm)

### Instrumentation
- [Auto-Instrumentation (Java)](https://docs.splunk.com/Observability/gdi/get-data-in/application/auto-instrumentation/auto-instrumentation-java.html)
- [K8s Operator Auto-Instrumentation](https://docs.splunk.com/Observability/gdi/opentelemetry/auto-instrumentation/auto-instrumentation-operator.html)
- [AlwaysOn Profiling](https://docs.splunk.com/observability/apm/profiling/get-data-in-profiling.html)

### GitHub
- [Splunk OTel Java Agent](https://github.com/signalfx/splunk-otel-java)
- [Splunk OTel Collector](https://github.com/signalfx/splunk-otel-collector)

## Instrumentation Quick Reference

### Java

```bash
# Download agent
curl -L https://github.com/signalfx/splunk-otel-java/releases/latest/download/splunk-otel-javaagent.jar -o splunk-otel-javaagent.jar

# Run with agent
java -javaagent:splunk-otel-javaagent.jar \
  -Dotel.service.name=my-service \
  -Dotel.resource.attributes=deployment.environment=production \
  -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
  -Dsplunk.profiler.enabled=true \
  -jar my-app.jar
```

### Python

```bash
pip install splunk-opentelemetry[all]
splunk-py-trace-bootstrap

# Run with auto-instrumentation
OTEL_SERVICE_NAME=my-service \
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
splunk-py-trace python my_app.py
```

### Node.js

```bash
npm install @splunk/otel

# Run with auto-instrumentation
OTEL_SERVICE_NAME=my-service \
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production \
node --require @splunk/otel/instrument app.js
```

### Manual Span Creation (Any Language)

```python
from opentelemetry import trace

tracer = trace.get_tracer("my-module")

with tracer.start_as_current_span("process-order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("customer.tier", "enterprise")
    span.set_attribute("order.total", 299.99)
    # ... business logic
    if error:
        span.set_status(trace.StatusCode.ERROR, "Payment failed")
        span.record_exception(error)
```

## Recommended Span Tags

### Universal Tags (All Services)

| Tag | Example | Purpose |
|-----|---------|---------|
| `deployment.environment` | production, staging | Environment filtering |
| `service.version` | 1.2.3 | Version correlation |
| `service.namespace` | checkout-team | Team ownership |

### Business Context Tags

| Tag | Example | Purpose |
|-----|---------|---------|
| `customer.tier` | enterprise, free | Prioritize by customer |
| `order.id` | ORD-12345 | Transaction tracing |
| `feature.flag` | new-checkout-v2 | Feature flag correlation |
| `tenant.id` | tenant-abc | Multi-tenant isolation |

## APM Alert Patterns

### Service-Level Detectors

| Detector | Signal | Condition |
|----------|--------|-----------|
| Error rate spike | `service.request.count{sf_error=true}` / `service.request.count` | Sudden change > 2x baseline |
| Latency degradation | `service.request.duration.ns.p99` | > 2x historical baseline |
| Throughput drop | `service.request.count` | < 50% of baseline for 5 min |
| Downstream dependency error | Error rate on dependency edge | > 10% on critical path |

### Troubleshooting Workflow

1. **Service Map** → Identify degraded service (red/yellow)
2. **RED Metrics** → Determine if Rate, Error, or Duration issue
3. **Tag Spotlight** → Find which dimension (endpoint, version, customer) is affected
4. **Trace Analyzer** → Find example traces matching the pattern
5. **Span details** → Examine individual spans for root cause
6. **Profiling** → If latency, check CPU/memory profiles for hot methods
7. **Log Observer** → Correlate with application logs
