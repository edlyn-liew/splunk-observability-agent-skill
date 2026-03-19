# OTel Collector Component Reference

## Official Documentation

### Splunk Distribution
- [Collector Overview](https://docs.splunk.com/observability/en/gdi/opentelemetry/opentelemetry.html)
- [Installation Guide](https://docs.splunk.com/observability/en/gdi/opentelemetry/install-the-collector.html)
- [Collector Components](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/collector-components)
- [Understand & Use the Collector](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/get-started-understand-and-use-the-collector)
- [Host Metrics Receiver](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/collector-components/receivers/host-metrics-receiver)
- [Splunk HEC Exporter](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/collector-components/exporters/splunk-hec-exporter)

### GitHub Repositories
- [Splunk OTel Collector](https://github.com/signalfx/splunk-otel-collector)
- [Kubernetes Helm Chart](https://github.com/signalfx/splunk-otel-collector-chart)
- [OTel Collector Operator](https://github.com/signalfx/splunk-otel-collector-operator)

### Upstream OpenTelemetry
- [Collector Configuration](https://opentelemetry.io/docs/collector/configuration/)
- [Security & Sensitive Data](https://opentelemetry.io/docs/security/handling-sensitive-data/)

## Receiver Catalog

| Receiver | Data Type | Use Case |
|----------|----------|----------|
| `hostmetrics` | Metrics | CPU, memory, disk, network, load, processes |
| `filelog` | Logs | Tail log files with parsing operators |
| `fluentforward` | Logs | Receive from Fluentd/Fluent Bit |
| `otlp` | All | gRPC/HTTP from instrumented apps |
| `prometheus` | Metrics | Scrape Prometheus endpoints |
| `journald` | Logs | Linux systemd journal |
| `windowseventlog` | Logs | Windows Event Log |
| `receiver_creator` | All | Dynamically create receivers based on discovery |
| `kubeletstats` | Metrics | K8s node/pod/container metrics |
| `k8s_cluster` | Metrics | K8s cluster-level metrics |

## Processor Catalog

| Processor | Purpose |
|-----------|---------|
| `memory_limiter` | Prevent OOM (always first in pipeline) |
| `batch` | Reduce export calls via batching |
| `resourcedetection` | Auto-add host.name, cloud metadata |
| `resource` | Add/update/delete resource attributes |
| `attributes` | Modify span/metric/log attributes |
| `filter` | Include/exclude telemetry by conditions |
| `transform` | Modify telemetry using OTTL statements |
| `redaction` | Remove sensitive data by pattern |
| `k8sattributes` | Enrich with K8s pod/namespace metadata |
| `metricstransform` | Rename, aggregate, combine metrics |
| `tail_sampling` | Sample traces by latency, error, attribute |

## Exporter Catalog

| Exporter | Destination | Data Types |
|----------|------------|-----------|
| `splunk_hec` | Splunk Enterprise/Cloud (HEC) | Metrics, Logs, Traces |
| `signalfx` | Splunk Observability Cloud | Metrics |
| `otlp` | Any OTLP endpoint (Obs Cloud APM) | Traces, Metrics |
| `otlphttp` | OTLP over HTTP | Traces, Metrics, Logs |
| `logging` | Console stdout (debug) | All |
| `file` | Local file output | All |

## Config File Locations

| Platform | Path |
|----------|------|
| Linux (agent) | `/etc/otel/collector/agent_config.yaml` |
| Linux (gateway) | `/etc/otel/collector/gateway_config.yaml` |
| Windows | Collector install directory |
| Kubernetes | Helm values.yaml |
| Docker | Mounted volume |

## Validation

```bash
otelcol validate --config=agent_config.yaml
```
