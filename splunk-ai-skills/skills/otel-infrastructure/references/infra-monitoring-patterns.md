# Infrastructure Monitoring Patterns

## Host Metrics Alert Thresholds

| Metric | Warning | Critical | Collection Interval |
|--------|---------|----------|-------------------|
| system.cpu.utilization | > 70% | > 90% | 10s |
| system.memory.utilization | > 80% | > 95% | 10s |
| system.filesystem.utilization | > 80% | > 95% | 60s |
| system.disk.io (sustained) | > 80% capacity | > 95% capacity | 10s |
| system.network.errors | > 0.1% | > 1% | 10s |
| system.cpu.load_average.5m | > CPU_count | > 2x CPU_count | 10s |
| system.paging.operations | Increasing trend | Sustained high rate | 30s |

## Deployment Patterns

### Pattern 1: Direct to Splunk Enterprise

```
Hosts (OTel Agent) → Splunk HEC → Splunk Indexers
```

Best for: On-prem Splunk, simple infrastructure, < 100 hosts.

### Pattern 2: Gateway Aggregation

```
Hosts (OTel Agent) → OTel Gateway → Splunk HEC
                                   → Observability Cloud
```

Best for: Centralized processing, dual destinations, filtering before ingestion.

### Pattern 3: Kubernetes Full Stack

```
DaemonSet Agent (per node) → Gateway (Deployment) → Splunk HEC
                                                   → Observability Cloud
OTel Operator (auto-instrument pods) ↗
```

Best for: Kubernetes environments needing metrics + logs + traces.

## Sizing Guidelines

| Scale | Agent Memory | Gateway Memory | Gateway Replicas |
|-------|-------------|---------------|-----------------|
| < 50 hosts | 256 MB | 512 MB | 1 |
| 50-200 hosts | 512 MB | 1 GB | 2 |
| 200-1000 hosts | 512 MB | 2 GB | 3+ |
| 1000+ hosts | 512 MB | 4 GB | 5+ (load balanced) |

## Resource Attribute Best Practices

Always tag telemetry with environment context:

```yaml
processors:
  resource:
    attributes:
      - key: deployment.environment
        value: production
        action: upsert
      - key: service.namespace
        value: platform-team
        action: upsert
      - key: host.group
        value: web-servers
        action: upsert
```

## Log Collection Patterns

### Syslog → OTel → Splunk

```yaml
receivers:
  filelog/syslog:
    include: ["/var/log/syslog"]
    operators:
      - type: regex_parser
        regex: '^(?P<pri><\d+>)?(?P<timestamp>\w+ \d+ \d+:\d+:\d+) (?P<host>\S+) (?P<ident>\S+?)(\[(?P<pid>\d+)\])?: (?P<message>.*)'
```

### Application Logs (JSON)

```yaml
receivers:
  filelog/app:
    include: ["/opt/app/logs/*.log"]
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%dT%H:%M:%S.%LZ'
```

### Multi-Line Stack Traces

```yaml
receivers:
  filelog/java:
    include: ["/opt/app/logs/app.log"]
    multiline:
      line_start_pattern: '^\d{4}-\d{2}-\d{2}'
```

## Security Best Practices

1. **Never hardcode tokens** — use `${ENV_VAR}` expansion in config
2. **Use TLS everywhere** — `tls.insecure_skip_verify: false`
3. **Redact sensitive data** — use the redaction processor for PII
4. **Restrict OTLP listener** — bind to localhost unless needed: `endpoint: 127.0.0.1:4317`
5. **Limit collector permissions** — run as non-root with minimal filesystem access
6. **Rotate tokens** — use Splunk token management for regular rotation
7. **Monitor collector health** — expose internal metrics, alert on drops/errors

Reference: [OTel Security — Handling Sensitive Data](https://opentelemetry.io/docs/security/handling-sensitive-data/)
