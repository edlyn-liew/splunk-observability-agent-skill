---
name: otel-infrastructure
description: >
  OpenTelemetry setup and infrastructure monitoring best practices in Splunk. Use when the
  user asks to "set up OpenTelemetry", "install OTel collector", "configure OTel for Splunk",
  "monitor infrastructure with OpenTelemetry", "set up host metrics", "configure receivers
  and exporters", "deploy OTel on Linux or Windows", "send metrics to Splunk via OTel",
  "configure HEC exporter", "set up log collection with OTel", "monitor CPU memory disk
  with OTel", "configure OTel pipeline", "set up fluentd or filelog receiver", "deploy
  OTel agent vs gateway", "configure OTel processors", "filter or batch telemetry data",
  "send OTel data to Splunk Enterprise", "set up OTel for on-prem infrastructure", or needs
  guidance on OpenTelemetry Collector architecture, pipeline configuration, receiver/processor/exporter
  setup, host metrics collection, log forwarding, or infrastructure monitoring patterns.
metadata:
  version: "0.1.0"
---

# OpenTelemetry Infrastructure Setup in Splunk

Set up the Splunk Distribution of OpenTelemetry Collector for infrastructure monitoring — covering installation, pipeline configuration, host metrics, log collection, and deployment patterns.

## Official Documentation

- [Splunk OTel Collector Overview](https://docs.splunk.com/observability/en/gdi/opentelemetry/opentelemetry.html)
- [Install the Collector](https://docs.splunk.com/observability/en/gdi/opentelemetry/install-the-collector.html)
- [Collector Components](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/collector-components)
- [GitHub: splunk-otel-collector](https://github.com/signalfx/splunk-otel-collector)
- [Upstream OTel Collector Config](https://opentelemetry.io/docs/collector/configuration/)

## Architecture

### Agent vs Gateway Mode

| Mode | Deployment | Use Case |
|------|-----------|----------|
| **Agent** | DaemonSet (K8s) or per-host | Local metrics/logs/traces collection, runs alongside apps |
| **Gateway** | Centralized deployment | Aggregation, routing, advanced processing, buffering |

**Recommended**: Agent on every host → Gateway for centralized processing → Splunk backends.

### Pipeline Model

```yaml
# Every pipeline has: receivers → processors → exporters
service:
  pipelines:
    metrics:
      receivers: [hostmetrics, prometheus]
      processors: [batch, resourcedetection]
      exporters: [splunk_hec, signalfx]
    logs:
      receivers: [filelog, fluentforward]
      processors: [batch, resource]
      exporters: [splunk_hec]
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [otlp/splunk, splunk_hec]
```

## Installation

### Linux (One-Liner)

```bash
curl -sSL https://dl.signalfx.com/splunk-otel-collector.sh > /tmp/splunk-otel-collector.sh
sudo sh /tmp/splunk-otel-collector.sh --realm <REALM> -- <ACCESS_TOKEN>
```

Config location: `/etc/otel/collector/agent_config.yaml`

### Windows

```powershell
& {Set-ExecutionPolicy Bypass -Scope Process -Force
$script = ((New-Object System.Net.WebClient).DownloadString('https://dl.signalfx.com/splunk-otel-collector.ps1'))
$params = @{access_token = "<ACCESS_TOKEN>"; realm = "<REALM>"}
Invoke-Command -ScriptBlock ([scriptblock]::Create(". {$script} $(&{$args} @params)"))}
```

### Docker

```bash
docker run --rm -e SPLUNK_ACCESS_TOKEN=<TOKEN> \
  -e SPLUNK_REALM=<REALM> \
  -v /etc/otel/collector/agent_config.yaml:/etc/otel/collector/agent_config.yaml \
  quay.io/signalfx/splunk-otel-collector:latest
```

## Core Receivers

### Host Metrics Receiver

Collects system-level metrics. Default in agent mode.

```yaml
receivers:
  hostmetrics:
    collection_interval: 10s
    scrapers:
      cpu:
      memory:
      disk:
      filesystem:
      network:
      load:
      paging:
      processes:
```

**Key metrics produced**:

| Scraper | Metrics | Description |
|---------|---------|-------------|
| `cpu` | system.cpu.time, system.cpu.utilization | Per-CPU and aggregate |
| `memory` | system.memory.usage, system.memory.utilization | Used/free/cached/buffered |
| `disk` | system.disk.io, system.disk.operations | Read/write bytes and ops |
| `filesystem` | system.filesystem.usage, system.filesystem.utilization | Mount point usage |
| `network` | system.network.io, system.network.packets, system.network.errors | Per-interface |
| `load` | system.cpu.load_average.1m/5m/15m | System load |
| `paging` | system.paging.usage, system.paging.operations | Swap usage |
| `processes` | system.processes.count, system.processes.created | Process counts by state |

### Filelog Receiver (Recommended for Logs)

```yaml
receivers:
  filelog:
    include:
      - /var/log/syslog
      - /var/log/auth.log
      - /var/log/messages
      - /opt/app/logs/*.log
    start_at: end
    operators:
      - type: regex_parser
        regex: '^(?P<timestamp>\S+ \S+) (?P<hostname>\S+) (?P<process>\S+): (?P<message>.*)'
        timestamp:
          parse_from: attributes.timestamp
          layout: '%b %d %H:%M:%S'
```

### Prometheus Receiver

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'node-exporter'
          scrape_interval: 15s
          static_configs:
            - targets: ['localhost:9100']
```

### OTLP Receiver (for Traces and Metrics from Apps)

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
```

## Core Processors

```yaml
processors:
  # Batching — reduces export calls
  batch:
    send_batch_size: 8192
    timeout: 200ms

  # Resource detection — auto-adds host.name, os.type, cloud metadata
  resourcedetection:
    detectors: [system, env, ec2, gcp, azure]
    system:
      hostname_sources: ["os"]

  # Resource attributes — add custom metadata
  resource:
    attributes:
      - key: deployment.environment
        value: production
        action: upsert
      - key: service.namespace
        value: my-team
        action: upsert

  # Filter — drop unwanted telemetry
  filter:
    metrics:
      exclude:
        match_type: regexp
        metric_names:
          - "system.cpu.time" # Drop if not needed

  # Memory limiter — prevent OOM
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128
```

## Core Exporters

### Splunk HEC Exporter (to Splunk Enterprise/Cloud)

```yaml
exporters:
  splunk_hec:
    token: "${SPLUNK_HEC_TOKEN}"
    endpoint: "https://splunk-hec.example.com:8088/services/collector"
    source: "otel"
    sourcetype: "otel:metrics"
    index: "metrics"
    tls:
      insecure_skip_verify: false
    log_data_enabled: true
    profiling_data_enabled: false
```

### SignalFx Exporter (to Observability Cloud)

```yaml
exporters:
  signalfx:
    access_token: "${SPLUNK_ACCESS_TOKEN}"
    realm: "${SPLUNK_REALM}"
```

### OTLP Exporter (to Observability Cloud APM)

```yaml
exporters:
  otlp:
    endpoint: "https://ingest.${SPLUNK_REALM}.signalfx.com:443"
    headers:
      "X-SF-TOKEN": "${SPLUNK_ACCESS_TOKEN}"
```

## Infrastructure Monitoring Patterns

### Baseline Host Monitoring Config

Minimum viable config for any host:

```yaml
receivers:
  hostmetrics:
    collection_interval: 10s
    scrapers: {cpu: {}, memory: {}, disk: {}, filesystem: {}, network: {}, load: {}}
  filelog:
    include: ["/var/log/syslog", "/var/log/auth.log"]

processors:
  batch: {send_batch_size: 8192, timeout: 200ms}
  resourcedetection: {detectors: [system, env]}
  memory_limiter: {check_interval: 1s, limit_mib: 512}

exporters:
  splunk_hec:
    token: "${SPLUNK_HEC_TOKEN}"
    endpoint: "${SPLUNK_HEC_URL}"
    index: "infrastructure"

service:
  pipelines:
    metrics:
      receivers: [hostmetrics]
      processors: [memory_limiter, batch, resourcedetection]
      exporters: [splunk_hec]
    logs:
      receivers: [filelog]
      processors: [memory_limiter, batch]
      exporters: [splunk_hec]
```

### Best Practices

1. **Always use memory_limiter** — first processor in every pipeline to prevent OOM
2. **Use resourcedetection** — auto-tags host.name, cloud.provider, cloud.region
3. **Batch aggressively** — reduces network overhead, improves throughput
4. **Filter early** — drop metrics you don't need at the processor level, not in Splunk
5. **Separate pipelines** — different receivers/exporters for metrics vs logs vs traces
6. **Use environment variables** — never hardcode tokens in config files
7. **Set collection_interval wisely** — 10s for real-time, 60s for capacity planning
8. **Deploy agents first, then gateway** — start simple, add gateway when you need central processing
9. **Monitor the collector itself** — expose internal metrics via Prometheus endpoint
10. **Test config before deploying** — `otelcol validate --config=agent_config.yaml`

For detailed reference on receivers, exporters, and deployment patterns, read:
- `references/otel-collector-reference.md` — Component catalog with configuration examples
- `references/infra-monitoring-patterns.md` — Infrastructure monitoring patterns and alert thresholds
