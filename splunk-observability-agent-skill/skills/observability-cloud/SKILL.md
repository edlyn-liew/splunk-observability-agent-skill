---
name: observability-cloud
description: >
  Splunk Observability Cloud best practices — APM tracing, Kubernetes monitoring, service maps,
  RUM, Log Observer, detectors, and platform/security team operational patterns. Use when the
  user asks to "set up Splunk APM", "instrument an application for tracing", "auto-instrument
  with OpenTelemetry", "monitor Kubernetes in Splunk", "set up K8s monitoring", "configure
  detectors and alerts", "use Splunk Observability dashboards", "analyze traces", "set up
  service maps", "configure span tags", "enable always-on profiling", "set up RUM",
  "configure Log Observer", "manage Observability Cloud teams and tokens", "set up RBAC
  for observability", "reduce observability costs", "configure data retention", "auto-instrument
  Java Python Node.js", "deploy OTel operator in Kubernetes", "monitor microservices",
  "troubleshoot application performance", "set up RED metrics", "calculate error budgets",
  or needs guidance on application tracing, distributed tracing, Kubernetes observability,
  or platform team and security team best practices for Splunk Observability Cloud.
metadata:
  version: "0.1.0"
---

# Splunk Observability Cloud

Best practices for APM, Kubernetes monitoring, application tracing, and platform/security team operations in Splunk Observability Cloud.

## Official Documentation

- [Observability Cloud Overview](https://docs.splunk.com/observability/en/get-started/overview.html)
- [APM Setup](https://docs.splunk.com/Observability/apm/set-up-apm/apm.html)
- [Infrastructure Monitoring](https://docs.splunk.com/observability/infrastructure/infrastructure.html)
- [Log Observer](https://docs.splunk.com/Observability/logs/logs.html)
- [RUM Setup](https://docs.splunk.com/observability/rum/set-up-rum.html)

## Platform Components

| Component | What It Does | Data Source |
|-----------|-------------|------------|
| **APM** | Distributed tracing, service maps, RED metrics | OTel traces (OTLP) |
| **Infrastructure Monitoring** | Host, container, cloud metrics | OTel metrics (SignalFx) |
| **Log Observer** | Log query and correlation | OTel logs or Splunk Platform connect |
| **RUM** | Browser/mobile front-end performance | RUM JS/mobile SDKs |
| **Synthetics** | Uptime and API testing | Synthetic probes |
| **On-Call** | Incident routing and escalation | Detector alerts |

## APM & Application Tracing

### Auto-Instrumentation Setup

Zero-code instrumentation for supported languages:

| Language | Package | Env Var |
|----------|---------|---------|
| Java | `splunk-otel-javaagent.jar` | `JAVA_TOOL_OPTIONS=-javaagent:splunk-otel-javaagent.jar` |
| Python | `splunk-opentelemetry[all]` | `splunk-py-trace` wrapper |
| Node.js | `@splunk/otel` | `node --require @splunk/otel/instrument` |
| .NET | `splunk-opentelemetry-dotnet` | Environment variables |
| Go | `splunk-otel-go` | Manual instrumentation |

**Required environment variables** (all languages):

```bash
OTEL_SERVICE_NAME=my-service
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.version=1.2.3
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
SPLUNK_PROFILER_ENABLED=true         # Optional: enable AlwaysOn Profiling
SPLUNK_PROFILER_MEMORY_ENABLED=true  # Optional: memory profiling
```

Docs: [Auto-Instrumentation](https://docs.splunk.com/Observability/gdi/get-data-in/application/auto-instrumentation/auto-instrumentation-java.html)

### RED Metrics

APM automatically computes RED metrics from span data:

| Metric | What It Measures | Alert Threshold |
|--------|-----------------|-----------------|
| **R**equest rate | Requests per second by service/endpoint | < 50% of baseline |
| **E**rror rate | % of requests with error spans | > 5% |
| **D**uration | P50, P90, P99 latency | P99 > 2x baseline |

Docs: [Track Service Performance](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/visualize-and-alert-on-your-application-in-splunk-apm/track-service-performance-using-dashboards-in-splunk-apm)

### Span Tags (Attributes)

Add custom context to traces:

```python
# Python example
from opentelemetry import trace
span = trace.get_current_span()
span.set_attribute("customer.tier", "enterprise")
span.set_attribute("order.id", "ORD-12345")
```

**Indexing scopes**: Service-level (one service), All Services (cross-service), Global (same value across trace).

Docs: [Span Tags](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/analyze-services-with-span-tags-and-metricsets/add-context-to-spans-with-span-tags-in-splunk-apm)

### Service Map

Dynamic service dependency visualization with health status coloring:
- Green = healthy, Yellow = degraded, Red = error
- Click service → drill into RED metrics, endpoints, dependencies
- Group services by indexed span tags (e.g., team, environment)

Docs: [Service Map](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/manage-services-spans-and-traces-in-splunk-apm/view-dependencies-in-the-service-map)

### Trace Analyzer

Full-fidelity trace searching — every trace is stored, not sampled:
- Filter by time, environment, service, tags, duration, error status
- 8-day default retention (extendable to 13 months)
- High-cardinality tag analysis

Docs: [Trace Analyzer](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/manage-services-spans-and-traces-in-splunk-apm/search-traces-using-trace-analyzer)

### Error Analysis

Docs: [Analyze Error Spans](https://help.splunk.com/en/splunk-observability-cloud/monitor-application-performance/manage-services-spans-and-traces-in-splunk-apm/analyze-error-spans)

### AlwaysOn Profiling

Continuous CPU and memory profiling with minimal overhead. Requires APM activation and OTel Collector v0.44.0+.

Docs: [Get Data In — Profiling](https://docs.splunk.com/observability/apm/profiling/get-data-in-profiling.html)

## Kubernetes Monitoring

### Deployment with Helm

```bash
helm repo add splunk-otel-collector-chart https://signalfx.github.io/splunk-otel-collector-chart
helm install splunk-otel-collector splunk-otel-collector-chart/splunk-otel-collector \
  --set="splunkObservability.accessToken=<TOKEN>" \
  --set="splunkObservability.realm=<REALM>" \
  --set="clusterName=my-cluster" \
  --set="splunkObservability.logsEnabled=true" \
  --set="environment=production"
```

Docs: [K8s Helm Install](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/collector-for-kubernetes/install-with-helm)

### What Gets Deployed

| Component | Type | Purpose |
|-----------|------|---------|
| Agent | DaemonSet | Per-node metrics, logs, traces |
| Cluster Receiver | Deployment | Cluster-level K8s metrics |
| Gateway (optional) | Deployment | Centralized processing |
| OTel Operator (optional) | Deployment | Auto-instrumentation injection |

### K8s Navigator Entities

Monitor via Kubernetes Navigator: Clusters, Nodes, Pods, Containers, Deployments, DaemonSets, ReplicaSets, StatefulSets, Jobs, Namespaces, Workloads, Services.

Docs: [Kubernetes Navigator](https://help.splunk.com/en/splunk-observability-cloud/monitor-infrastructure/monitor-services-and-hosts/monitor-kubernetes-entities)

### Auto-Instrumentation via Operator

Inject instrumentation into pods via annotations — no code changes:

```yaml
# Pod or Namespace annotation
instrumentation.opentelemetry.io/inject-java: "true"
# Or for Python
instrumentation.opentelemetry.io/inject-python: "true"
# Or for Node.js
instrumentation.opentelemetry.io/inject-nodejs: "true"
```

Supports: Java, Python, Node.js, .NET, Go.

Docs: [K8s Operator Auto-Instrumentation](https://docs.splunk.com/Observability/gdi/opentelemetry/auto-instrumentation/auto-instrumentation-operator.html)
GitHub: [splunk-otel-collector-operator](https://github.com/signalfx/splunk-otel-collector-operator)

### Key K8s Metrics to Monitor

| Metric | Warning | Critical |
|--------|---------|----------|
| Pod restart count | > 3/hour | > 10/hour |
| Node CPU utilization | > 70% | > 90% |
| Node memory utilization | > 80% | > 95% |
| Pod pending duration | > 5 min | > 15 min |
| Container OOMKilled | Any occurrence | Repeated |
| PVC usage | > 80% | > 95% |

## Detectors & Alerting

### Detector Types

| Type | Use Case |
|------|----------|
| **Static threshold** | Fixed value (e.g., CPU > 90%) |
| **Sudden change** | Rapid deviation from recent baseline |
| **Historical anomaly** | Deviation from same time last week |
| **Resource running out** | Linear projection of exhaustion |
| **AutoDetect** | Automatic for supported integrations |

Severity levels: Critical, Major, Minor, Warning, Info.

Docs: [Detectors & Alerts](https://help.splunk.com/en/splunk-observability-cloud/create-alerts-detectors-and-service-level-objectives/create-alerts-and-detectors/introduction-to-alerts-and-detectors)

## Platform Team Best Practices

### Team Structure & RBAC

| Role | Permissions |
|------|------------|
| Admin | Full access, user management, token management |
| User | Create dashboards, detectors, view all data |
| Read-only | View dashboards and alerts only |
| Usage | Subscription and billing access |

Docs: [Manage Roles](https://help.splunk.com/en/splunk-observability-cloud/administer/user-and-team-management/manage-roles)

### Token Management

| Token Type | Use Case | Best Practice |
|------------|----------|---------------|
| Org access token | Collector data ingest | One per environment, rotate every 90 days |
| User API token | API access, Terraform | Per-user, short expiry (30 days) |
| RUM token | Browser instrumentation | One per app, public-facing |

Docs: [API Access Tokens](https://help.splunk.com/en/splunk-observability-cloud/administer/authentication-and-security/authentication-tokens/api-access-tokens)

### Cost Management

| Billing Unit | How Counted | Cost Reduction |
|-------------|------------|---------------|
| Hosts | Hourly count, monthly average | Right-size collection interval |
| Containers | Hourly count, monthly average | Exclude non-prod namespaces |
| Custom MTS | Metric time series count | Drop unused metrics via filter processor |
| High-res MTS | < 10s resolution | Use 10s+ intervals where possible |
| Archived metrics | 1/10th real-time cost | Archive low-priority metrics |

Docs: [Billing & Usage](https://help.splunk.com/en/splunk-observability-cloud/administer/monitor-subscription-usage-and-billing/manage-infrastructure-billing-host-and-metrics-plans)

### Data Retention

| Data Type | Default | Extended Options |
|-----------|---------|-----------------|
| APM traces | 8 days | 30, 60, 90 days, 13 months |
| Infrastructure metrics | Resolution-dependent | Varies by subscription |

Docs: [Data Retention](https://help.splunk.com/en/splunk-observability-cloud/administer/org-reference-info/data-retention-in-splunk-observability-cloud)

## Security Team Best Practices

1. **Least-privilege tokens** — use Read-only for dashboards, User only where needed
2. **Redact PII from traces** — use OTel Collector redaction processor before export
3. **Restrict OTLP endpoints** — bind to localhost or internal network only
4. **Audit token usage** — monitor for unusual API call patterns
5. **Separate environments** — different tokens for prod/staging/dev
6. **Encrypt in transit** — TLS for all collector-to-backend communication
7. **Review span tags** — ensure no secrets or PII in custom attributes
8. **Use team-scoped dashboards** — limit visibility to relevant teams
9. **Set usage limits** — per-token limits prevent runaway costs from misconfiguration
10. **Rotate tokens regularly** — 90-day rotation cycle, use pre-expiration rotation

For detailed references, read:
- `references/apm-tracing-reference.md` — APM deep dive, instrumentation, and troubleshooting
- `references/kubernetes-observability.md` — K8s deployment patterns, metrics, and Navigator
