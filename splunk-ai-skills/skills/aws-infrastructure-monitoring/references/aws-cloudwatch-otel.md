# AWS CloudWatch & OpenTelemetry Reference

## Official Documentation

### Splunk + AWS Integration
- [Monitor AWS in Observability Cloud](https://docs.splunk.com/observability/en/infrastructure/monitor/aws.html)
- [Connect AWS — All Methods](https://help.splunk.com/en/splunk-observability-cloud/manage-data/connect-to-your-cloud-service-provider/connect-to-aws)
- [Connect via Polling](https://help.splunk.com/en/splunk-observability-cloud/manage-data/connect-to-your-cloud-service-provider/connect-to-aws/connect-via-polling)
- [AWS-Managed Metric Streams](https://help.splunk.com/en/splunk-observability-cloud/manage-data/connect-to-your-cloud-service-provider/connect-to-aws/connect-with-aws-managed-metric-streams)
- [Splunk-Managed Metric Streams](https://help.splunk.com/en/splunk-observability-cloud/manage-data/connect-to-your-cloud-service-provider/connect-to-aws/connect-with-splunk-managed-metrics-streams)
- [Manage AWS Data Import](https://help.splunk.com/en/splunk-observability-cloud/monitor-infrastructure/monitor-services-and-hosts/monitor-amazon-web-services/manage-aws-data-import)
- [Send AWS Logs to Splunk](https://help.splunk.com/en/splunk-observability-cloud/manage-data/connect-to-your-cloud-service-provider/connect-to-aws/send-aws-logs-to-splunk-platform)
- [Metric Naming Conventions](https://help.splunk.com/en/splunk-observability-cloud/manage-data/metrics-metadata-and-events/metrics-in-splunk-observability-cloud/naming-conventions)

### Splunk Add-on for AWS
- [Add-on Documentation](https://splunk.github.io/splunk-add-on-for-amazon-web-services/)
- [Splunkbase Download](https://splunkbase.splunk.com/app/1876)
- [Data Types & Sourcetypes](https://splunk.github.io/splunk-add-on-for-amazon-web-services/DataTypes/)
- [Configure Inputs](https://docs.splunk.com/Documentation/AddOns/released/AWS/ConfigureInputs)
- [SQS-Based S3 Ingestion](https://splunk.github.io/splunk-add-on-for-amazon-web-services/SQS-basedS3/)
- [CloudTrail Config](https://splunk.github.io/splunk-add-on-for-amazon-web-services/CloudTrail/)
- [VPC Flow Logs Config](https://splunk.github.io/splunk-add-on-for-amazon-web-services/VPCFlowLogs/)
- [Transit Gateway Flow Logs](https://splunk.github.io/splunk-add-on-for-amazon-web-services/TransitGatewayFlowLogs/)

### AWS OpenTelemetry (ADOT)
- [ADOT Main Docs](https://aws-otel.github.io/)
- [ADOT Getting Started](https://aws-otel.github.io/docs/getting-started/collector/)
- [ADOT Lambda (Java)](https://aws-otel.github.io/docs/getting-started/lambda/lambda-java/)
- [ADOT ECS Setup](https://aws-otel.github.io/docs/setup/ecs/)
- [ADOT CloudWatch Metrics](https://aws-otel.github.io/docs/getting-started/cloudwatch-metrics/)
- [ADOT GitHub Repository](https://github.com/aws-observability/aws-otel-collector)
- [CloudWatch Receiver (OTel Contrib)](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/awscloudwatchreceiver/README.md)

### AWS Native Docs
- [CloudWatch Metrics Viewer](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/viewing_metrics_with_cloudwatch.html)
- [AWS Services with CloudWatch Metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/aws-services-cloudwatch-metrics.html)
- [CloudWatch OTLP Setup (Simple)](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-OTLPSimplesetup.html)
- [CloudWatch OTLP Setup (Advanced)](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-OTLPAdvancedsetup.html)

## Ingestion Architecture Patterns

### Pattern 1: Observability Cloud (Metrics + APM)
```
AWS CloudWatch → Metric Streams → Kinesis Firehose → Splunk Observability Cloud
AWS Compute (EC2/ECS/EKS) → OTel Collector → Splunk Observability Cloud
AWS Lambda → OTel Lambda Layer → Splunk Observability Cloud
```

### Pattern 2: Splunk Enterprise (Logs + Events)
```
CloudTrail → S3 → SNS → SQS → Splunk Add-on for AWS
VPC Flow Logs → CloudWatch Logs → Kinesis Firehose → Splunk HEC
GuardDuty → CloudWatch Events → Lambda → Splunk HEC
```

### Pattern 3: Hybrid (Both)
```
Metrics → Observability Cloud (Metric Streams)
Logs → Splunk Enterprise (Kinesis Firehose / SQS-S3)
Traces → Observability Cloud APM (OTel / Lambda Layer)
```

## OTel Collector on AWS Compute

### EC2 Agent Config

```yaml
receivers:
  hostmetrics:
    collection_interval: 10s
    scrapers: {cpu: {}, memory: {}, disk: {}, filesystem: {}, network: {}}
  filelog:
    include: ["/var/log/messages", "/var/log/secure", "/opt/app/logs/*.log"]

processors:
  resourcedetection:
    detectors: [ec2, system]  # Auto-tags instance_id, region, AZ
  batch: {}
  memory_limiter: {check_interval: 1s, limit_mib: 512}

exporters:
  signalfx:
    access_token: "${SPLUNK_ACCESS_TOKEN}"
    realm: "${SPLUNK_REALM}"
  splunk_hec:
    token: "${SPLUNK_HEC_TOKEN}"
    endpoint: "${SPLUNK_HEC_URL}"
    index: "aws_infra"

service:
  pipelines:
    metrics: {receivers: [hostmetrics], processors: [memory_limiter, resourcedetection, batch], exporters: [signalfx]}
    logs: {receivers: [filelog], processors: [memory_limiter, batch], exporters: [splunk_hec]}
```

### ECS Fargate Sidecar Config

```yaml
# Add OTel Collector as sidecar in task definition
receivers:
  otlp: {protocols: {grpc: {endpoint: "0.0.0.0:4317"}}}
  awsecscontainermetrics: {}  # ECS-specific container metrics

processors:
  resourcedetection: {detectors: [ecs, env]}
  batch: {}

exporters:
  signalfx: {access_token: "${SPLUNK_ACCESS_TOKEN}", realm: "${SPLUNK_REALM}"}
  otlp: {endpoint: "https://ingest.${SPLUNK_REALM}.signalfx.com:443"}
```

Docs: [ECS EC2 deployment](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/other-deployment-tools-ecs-ec2-fargate-nomad-pcf/amazon-ecs-ec2) · [Fargate deployment](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/other-deployment-tools-ecs-ec2-fargate-nomad-pcf/amazon-fargate)

## CloudWatch Polling vs Metric Streams

| Feature | Polling | Metric Streams |
|---------|---------|---------------|
| Latency | 60-600s | ~1 min |
| Setup | Simple (API creds) | Kinesis Firehose required |
| Cost | API calls | Firehose + stream charges |
| Stats | SampleCount, Avg, Sum, Min, Max | All stats |
| Custom metrics | Supported | Supported |
| Best for | Getting started, low volume | Production, high volume, low latency |
