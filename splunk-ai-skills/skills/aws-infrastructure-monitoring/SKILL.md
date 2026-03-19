---
name: aws-infrastructure-monitoring
description: >
  Monitor AWS infrastructure using OpenTelemetry and Splunk — CloudWatch metrics/logs, VPC Flow
  Logs, EC2, EKS, Lambda, RDS, ECS Fargate, CloudTrail, GuardDuty, and Security Hub. Use when
  the user asks to "monitor AWS in Splunk", "set up CloudWatch metrics in Splunk", "analyze VPC
  flow logs", "monitor EC2 instances", "instrument Lambda with OTel", "monitor EKS clusters",
  "monitor RDS databases", "set up AWS security monitoring", "analyze CloudTrail", "integrate
  GuardDuty with Splunk", "monitor ELB ALB in Splunk", "set up AWS WAF monitoring", "monitor
  API Gateway", "track AWS IAM changes", "monitor S3 access", "set up AWS Config compliance",
  "deploy OTel on ECS Fargate", "use ADOT collector", "set up AWS Metric Streams", "monitor
  SQS queues", "analyze Route53 DNS", "set up Transit Gateway flow logs", or needs guidance
  on AWS monitoring patterns, CloudWatch namespace metrics, AWS sourcetypes, data ingestion
  methods, or network/application/security monitoring for AWS environments.
metadata:
  version: "0.1.0"
---

# AWS Infrastructure Monitoring with Splunk & OpenTelemetry

Monitor AWS infrastructure across network, application, and security perspectives using Splunk Observability Cloud, Splunk Enterprise, and OpenTelemetry.

## Official Documentation

- [Monitor AWS in Splunk Observability](https://docs.splunk.com/observability/en/infrastructure/monitor/aws.html)
- [Connect AWS to Splunk Observability](https://docs.splunk.com/observability/gdi/get-data-in/connect/aws/get-awstoc.html)
- [Splunk Add-on for AWS](https://splunk.github.io/splunk-add-on-for-amazon-web-services/)
- [Splunk Add-on for AWS (Splunkbase)](https://splunkbase.splunk.com/app/1876)
- [AWS CloudWatch Metrics Reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/aws-services-cloudwatch-metrics.html)
- [AWS Distro for OpenTelemetry (ADOT)](https://aws-otel.github.io/)

## Data Ingestion Methods

| Method | Best For | Latency | Docs |
|--------|---------|---------|------|
| **CloudWatch Polling** | Metrics to Observability Cloud | 60-600s | [Polling setup](https://help.splunk.com/en/splunk-observability-cloud/manage-data/connect-to-your-cloud-service-provider/connect-to-aws/connect-via-polling) |
| **Metric Streams** | Low-latency metrics to Obs Cloud | ~1 min | [AWS-managed streams](https://help.splunk.com/en/splunk-observability-cloud/manage-data/connect-to-your-cloud-service-provider/connect-to-aws/connect-with-aws-managed-metric-streams) |
| **Kinesis Firehose → HEC** | Logs (CloudWatch, VPC, WAF) | Near real-time | [Logs to Splunk](https://help.splunk.com/en/splunk-observability-cloud/manage-data/connect-to-your-cloud-service-provider/connect-to-aws/send-aws-logs-to-splunk-platform) |
| **SQS-based S3** | CloudTrail, Config, S3 logs | Minutes | [SQS-based S3](https://splunk.github.io/splunk-add-on-for-amazon-web-services/SQS-basedS3/) |
| **OTel Collector** | Metrics + traces from compute | Seconds | [OTel on ECS](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/other-deployment-tools-ecs-ec2-fargate-nomad-pcf/amazon-ecs-ec2) |
| **Lambda Layer** | Serverless traces + metrics | Seconds | [Lambda instrumentation](https://help.splunk.com/en/splunk-observability-cloud/manage-data/instrument-serverless-functions/instrument-serverless-functions/instrument-aws-lambda-functions) |

## AWS Sourcetypes (Splunk Add-on)

| Sourcetype | Data Source | Key Fields |
|-----------|------------|------------|
| `aws:cloudtrail` | CloudTrail API audit | eventName, userIdentity, sourceIPAddress, errorCode |
| `aws:cloudwatch` | CloudWatch metrics | metric_name, namespace, dimensions |
| `aws:cloudwatch:log` | CloudWatch Logs | logGroup, logStream, message |
| `aws:config` | AWS Config snapshots | resourceType, resourceId, configuration |
| `aws:config:rule` | Config Rule evaluations | complianceType, configRuleName |
| `aws:vpc:flow-logs` | VPC Flow Logs | srcaddr, dstaddr, srcport, dstport, action, protocol, bytes, packets |
| `aws:transitgateway:flowlogs` | Transit Gateway flows | srcaddr, dstaddr, packets, bytes |
| `aws:s3` | S3 access logs | bucket, key, operation, remoteIP |
| `aws:route53` | Route53 query logs | query_name, query_type, response_code |

Full sourcetype reference: [AWS Add-on Data Types](https://splunk.github.io/splunk-add-on-for-amazon-web-services/DataTypes/)

## CloudWatch Namespaces

| Namespace | Service | Key Metrics |
|-----------|---------|------------|
| `AWS/EC2` | EC2 instances | CPUUtilization, NetworkIn/Out, DiskReadOps, StatusCheckFailed |
| `AWS/ELB` | Classic LB | Latency, RequestCount, HTTPCode_Backend_5XX, UnHealthyHostCount |
| `AWS/ApplicationELB` | ALB | TargetResponseTime, RequestCount, HTTPCode_Target_5XX |
| `AWS/RDS` | RDS databases | CPUUtilization, FreeableMemory, ReadLatency, WriteLatency, DatabaseConnections |
| `AWS/Lambda` | Lambda functions | Duration, Errors, Invocations, ConcurrentExecutions, Throttles |
| `AWS/ECS` | ECS services | CPUUtilization, MemoryUtilization |
| `AWS/EKS` | EKS clusters | node_cpu_utilization, pod_cpu_utilization |
| `AWS/ApiGateway` | API Gateway | Latency, Count, 4XXError, 5XXError |
| `AWS/SQS` | SQS queues | ApproximateNumberOfMessagesVisible, ApproximateAgeOfOldestMessage |
| `AWS/DynamoDB` | DynamoDB | ConsumedReadCapacityUnits, ThrottledRequests |
| `AWS/S3` | S3 buckets | BucketSizeBytes, NumberOfObjects |
| `AWS/Route53` | Route53 | HealthCheckStatus, HealthCheckPercentageHealthy |

## Network Perspective

### VPC Flow Logs Analysis

```spl
# Top denied connections (security groups / NACLs)
index=aws sourcetype=aws:vpc:flow-logs action=REJECT
| stats count by srcaddr, dstaddr, dstport, protocol
| sort -count | head 20

# Bandwidth by source IP
index=aws sourcetype=aws:vpc:flow-logs action=ACCEPT
| stats sum(bytes) as total_bytes by srcaddr
| eval total_gb = round(total_bytes/1073741824, 2)
| sort -total_bytes | head 20

# Cross-AZ traffic (cost indicator)
index=aws sourcetype=aws:vpc:flow-logs
| lookup aws_az_lookup eni_id OUTPUT az
| where src_az != dest_az
| stats sum(bytes) as cross_az_bytes by src_az, dest_az

# Unusual port access
index=aws sourcetype=aws:vpc:flow-logs action=ACCEPT dstport!=80 dstport!=443 dstport!=22 dstport!=53
| stats count, dc(srcaddr) as unique_sources by dstport
| where count > 100
| sort -count
```

Docs: [VPC Flow Logs config](https://splunk.github.io/splunk-add-on-for-amazon-web-services/VPCFlowLogs/)

### Load Balancer Monitoring

```spl
# ALB error rate
index=aws sourcetype=aws:cloudwatch namespace="AWS/ApplicationELB"
| timechart span=5m sum(HTTPCode_Target_5XX) as errors, sum(RequestCount) as requests
| eval error_pct = round(errors/requests*100, 2)

# ALB latency P95
index=aws sourcetype=aws:cloudwatch namespace="AWS/ApplicationELB"
| timechart span=5m perc95(TargetResponseTime) as p95_latency
| where p95_latency > 1
```

### Route53 DNS

```spl
# Top queried domains
index=aws sourcetype=aws:route53
| stats count by query_name | sort -count | head 20

# NXDOMAIN responses (potential DGA/misconfiguration)
index=aws sourcetype=aws:route53 response_code=NXDOMAIN
| stats count by query_name, srcaddr | sort -count
```

Docs: [Route53 DNS in Splunk](https://aws.amazon.com/blogs/apn/how-to-leverage-amazon-route-53-vpc-dns-queries-in-splunk-on-aws/)

### AWS WAF

Docs: [WAF logs via Firehose](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/send-aws-waf-logs-to-splunk-by-using-aws-firewall-manager-and-amazon-data-firehose.html) · [WAF Add-on](https://splunkbase.splunk.com/app/4714)

### Transit Gateway

Docs: [Transit Gateway Flow Logs](https://splunk.github.io/splunk-add-on-for-amazon-web-services/TransitGatewayFlowLogs/)

## Application Perspective

### Lambda Monitoring (OTel Layer)

```bash
# Add Splunk OTel Lambda Layer
aws lambda update-function-configuration \
  --function-name my-function \
  --layers arn:aws:lambda:<region>:254067382080:layer:splunk-apm:<version> \
  --environment Variables="{
    SPLUNK_REALM=<realm>,
    SPLUNK_ACCESS_TOKEN=<token>,
    OTEL_SERVICE_NAME=my-function,
    OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production
  }"
```

Key Lambda KPIs:

| Metric | Warning | Critical |
|--------|---------|----------|
| Error rate | > 1% | > 5% |
| Duration vs timeout | > 70% of timeout | > 90% of timeout |
| Throttles | Any sustained | > 10/min |
| Cold start ratio | > 20% | > 50% |
| Concurrent executions | > 80% of limit | > 90% of limit |

Docs: [Lambda instrumentation](https://help.splunk.com/en/splunk-observability-cloud/manage-data/instrument-serverless-functions/instrument-serverless-functions/instrument-aws-lambda-functions) · [Lambda metrics](https://docs.splunk.com/observability/gdi/get-data-in/serverless/aws/otel-lambda-layer/lambda-metrics.html)

### EKS Monitoring

Deploy OTel Collector via Helm with EKS-specific config:

```bash
helm install splunk-otel-collector splunk-otel-collector-chart/splunk-otel-collector \
  --set="splunkObservability.accessToken=<TOKEN>" \
  --set="splunkObservability.realm=<REALM>" \
  --set="clusterName=my-eks-cluster" \
  --set="cloudProvider=aws"
```

Docs: [EKS monitoring](https://lantern.splunk.com/Observability/Product_Tips/Infrastructure_Monitoring/Monitoring_Amazon_Elastic_Kubernetes_Services_(EKS)_with_Splunk_Observability_Cloud)

### ECS Fargate (Sidecar Pattern)

Deploy OTel Collector as a sidecar container in Fargate tasks.

Docs: [Fargate deployment](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/other-deployment-tools-ecs-ec2-fargate-nomad-pcf/amazon-fargate) · [ECS container metrics](https://help.splunk.com/en/splunk-observability-cloud/manage-data/available-data-sources/supported-integrations-in-splunk-observability-cloud/opentelemetry-receivers/aws-ecs-container-metrics-receiver)

### RDS Database Monitoring

Key RDS KPIs:

| Metric | Warning | Critical |
|--------|---------|----------|
| CPUUtilization | > 70% | > 90% |
| FreeableMemory | < 1 GB | < 256 MB |
| ReadLatency / WriteLatency | > 20ms | > 50ms |
| DatabaseConnections | > 80% max | > 95% max |
| FreeStorageSpace | < 20% | < 10% |
| ReplicaLag | > 30s | > 120s |

Docs: [RDS in Splunk](https://aws.amazon.com/blogs/database/view-amazon-cloudwatch-logs-for-amazon-rds-in-splunk-cloud-platform/)

## Security Perspective

### CloudTrail Analysis

```spl
# IAM privilege escalation
index=aws sourcetype=aws:cloudtrail eventName IN ("CreateUser","AttachUserPolicy","PutUserPolicy","CreateAccessKey","AssumeRole")
| stats count by userIdentity.arn, eventName, sourceIPAddress
| sort -count

# Root account usage (should be near zero)
index=aws sourcetype=aws:cloudtrail userIdentity.type=Root
| stats count by eventName, sourceIPAddress

# Failed API calls (recon indicator)
index=aws sourcetype=aws:cloudtrail errorCode IN ("AccessDenied","UnauthorizedAccess","Client.UnauthorizedAccess")
| stats count by userIdentity.arn, eventName, errorCode
| sort -count

# Console login without MFA
index=aws sourcetype=aws:cloudtrail eventName=ConsoleLogin
| spath input=additionalEventData path=MFAUsed output=mfa_used
| where mfa_used="No"
| stats count by userIdentity.arn, sourceIPAddress
```

Docs: [CloudTrail config](https://splunk.github.io/splunk-add-on-for-amazon-web-services/CloudTrail/) · [CloudTrail app](https://splunkbase.splunk.com/app/5760)

### GuardDuty

```spl
# GuardDuty high-severity findings
index=aws sourcetype=aws:cloudwatch:guardduty severity>=7
| stats count by type, title, resource.resourceType
| sort -count
```

Docs: [GuardDuty integration](https://repost.aws/articles/ARhXA6njHGRzKEXQ20BKO4lA/how-to-integrate-amazon-guardduty-findings-with-on-premises-splunk) · [GuardDuty app](https://splunkbase.splunk.com/app/5762)

### Security Hub

Docs: [Security Hub integration](https://github.com/splunk/splunk-for-securityHub) · [Security Hub app](https://splunkbase.splunk.com/app/5767)

### AWS Config Compliance

```spl
# Non-compliant resources
index=aws sourcetype=aws:config:rule complianceType=NON_COMPLIANT
| stats count by configRuleName, resourceType
| sort -count
```

Docs: [AWS Config rules](https://docs.splunk.com/Documentation/AddOns/released/AWS/ConfigRules) · [Config ingestion](https://aws.amazon.com/blogs/mt/ingest-aws-config-data-into-splunk-with-ease/)

### S3 Bucket Security

```spl
# S3 public access attempts
index=aws sourcetype=aws:s3 operation=REST.GET.OBJECT
| where NOT cidrmatch("10.0.0.0/8", remoteIP)
| stats count by bucket, remoteIP, operation
| sort -count
```

### IAM Monitoring

Docs: [IAM privilege escalation research](https://www.splunk.com/en_us/blog/security/aws-iam-privilege-escalation-threat-research-release-march-2021.html) · [IAM app](https://splunkbase.splunk.com/app/5763)

## Alert Thresholds by Perspective

### Network
| Alert | Condition |
|-------|-----------|
| VPC Flow — high deny rate | > 1000 REJECT/5min from single src |
| ALB — 5XX error spike | Error rate > 5% for 5 min |
| ALB — latency degradation | P95 > 2x baseline |
| Route53 health check failure | HealthCheckStatus = 0 |

### Application
| Alert | Condition |
|-------|-----------|
| Lambda errors | Error rate > 1% |
| Lambda throttles | Any sustained throttling |
| RDS CPU | > 90% for 10 min |
| RDS connections | > 80% of max |
| ECS task failures | > 3 task failures in 15 min |
| SQS age | ApproximateAgeOfOldestMessage > 300s |

### Security
| Alert | Condition |
|-------|-----------|
| Root account login | Any usage |
| Console login without MFA | Any occurrence |
| GuardDuty high severity | Severity >= 7 |
| IAM privilege escalation | CreateUser/AttachPolicy by non-admin |
| Failed API calls spike | > 50 AccessDenied in 5 min |
| Config non-compliance | Any NON_COMPLIANT |

For detailed references, read:
- `references/aws-cloudwatch-otel.md` — CloudWatch integration, OTel on AWS, ADOT patterns
- `references/aws-monitoring-perspectives.md` — Network, application, and security monitoring deep dive
