# AWS Monitoring Perspectives Reference

## Official Documentation

### Network
- [VPC Flow Logs Config](https://splunk.github.io/splunk-add-on-for-amazon-web-services/VPCFlowLogs/)
- [VPC Flow Logs via Firehose](https://aws.amazon.com/blogs/big-data/ingest-vpc-flow-logs-into-splunk-using-amazon-kinesis-data-firehose/)
- [Transit Gateway Flow Logs](https://splunk.github.io/splunk-add-on-for-amazon-web-services/TransitGatewayFlowLogs/)
- [Route53 DNS in Splunk](https://aws.amazon.com/blogs/apn/how-to-leverage-amazon-route-53-vpc-dns-queries-in-splunk-on-aws/)
- [WAF Logs to Splunk](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/send-aws-waf-logs-to-splunk-by-using-aws-firewall-manager-and-amazon-data-firehose.html)
- [AWS WAF Add-on](https://splunkbase.splunk.com/app/4714)
- [Network Firewall to Splunk](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/view-aws-network-firewall-logs-and-metrics-by-using-splunk.html)

### Application
- [Lambda Instrumentation](https://help.splunk.com/en/splunk-observability-cloud/manage-data/instrument-serverless-functions/instrument-serverless-functions/instrument-aws-lambda-functions)
- [Lambda Advanced Config](https://help.splunk.com/en/splunk-observability-cloud/manage-data/instrument-serverless-functions/instrument-serverless-functions/instrument-aws-lambda-functions/advanced-configuration)
- [Lambda Metrics](https://docs.splunk.com/observability/gdi/get-data-in/serverless/aws/otel-lambda-layer/lambda-metrics.html)
- [EKS Monitoring Guide](https://lantern.splunk.com/Observability/Product_Tips/Infrastructure_Monitoring/Monitoring_Amazon_Elastic_Kubernetes_Services_(EKS)_with_Splunk_Observability_Cloud)
- [ECS Container Metrics Receiver](https://help.splunk.com/en/splunk-observability-cloud/manage-data/available-data-sources/supported-integrations-in-splunk-observability-cloud/opentelemetry-receivers/aws-ecs-container-metrics-receiver)
- [RDS Logs in Splunk](https://aws.amazon.com/blogs/database/view-amazon-cloudwatch-logs-for-amazon-rds-in-splunk-cloud-platform/)

### Security
- [CloudTrail Config](https://splunk.github.io/splunk-add-on-for-amazon-web-services/CloudTrail/)
- [CloudTrail for Security Ops](https://www.splunk.com/en_us/blog/security/cloudtrail-data-security-operations.html)
- [GuardDuty Integration](https://repost.aws/articles/ARhXA6njHGRzKEXQ20BKO4lA/how-to-integrate-amazon-guardduty-findings-with-on-premises-splunk)
- [Security Hub Integration](https://github.com/splunk/splunk-for-securityHub)
- [IAM Privilege Escalation Research](https://www.splunk.com/en_us/blog/security/aws-iam-privilege-escalation-threat-research-release-march-2021.html)
- [AWS Config Rules](https://docs.splunk.com/Documentation/AddOns/released/AWS/ConfigRules)
- [AWS Config Ingestion](https://aws.amazon.com/blogs/mt/ingest-aws-config-data-into-splunk-with-ease/)

### Splunkbase Apps
- [CloudTrail App](https://splunkbase.splunk.com/app/5760)
- [GuardDuty App](https://splunkbase.splunk.com/app/5762)
- [Security Hub App](https://splunkbase.splunk.com/app/5767)
- [IAM App](https://splunkbase.splunk.com/app/5763)
- [EC2 App](https://splunkbase.splunk.com/app/5761)
- [AWS Security Dashboards](https://splunkbase.splunk.com/app/6311)

## Network Monitoring Searches

### VPC Flow Logs Deep Dive

```spl
# Security group deny analysis (which rules are blocking)
index=aws sourcetype=aws:vpc:flow-logs action=REJECT
| stats count as denies, dc(srcaddr) as unique_src by dstaddr, dstport
| sort -denies | head 20

# Detect port scanning
index=aws sourcetype=aws:vpc:flow-logs action=REJECT
| stats dc(dstport) as ports_scanned, count as attempts by srcaddr, dstaddr
| where ports_scanned > 20
| eval alert = srcaddr." scanned ".ports_scanned." ports on ".dstaddr

# East-west traffic analysis (internal lateral movement)
index=aws sourcetype=aws:vpc:flow-logs action=ACCEPT
| where cidrmatch("10.0.0.0/8", srcaddr) AND cidrmatch("10.0.0.0/8", dstaddr)
| stats sum(bytes) as total_bytes, count as flows by srcaddr, dstaddr
| sort -total_bytes | head 30

# Data exfiltration indicator (large outbound to external)
index=aws sourcetype=aws:vpc:flow-logs action=ACCEPT
| where NOT cidrmatch("10.0.0.0/8", dstaddr) AND NOT cidrmatch("172.16.0.0/12", dstaddr)
| stats sum(bytes) as bytes_out by srcaddr
| where bytes_out > 1073741824
| eval gb = round(bytes_out/1073741824, 2)
```

### WAF Analysis

```spl
# WAF blocked requests by rule
index=aws sourcetype=aws:waf action=BLOCK
| stats count by terminatingRuleId, ruleGroupList{}.ruleGroupId
| sort -count

# WAF top blocked IPs
index=aws sourcetype=aws:waf action=BLOCK
| stats count by httpRequest.clientIp, httpRequest.country
| sort -count | head 20
```

## Application Monitoring Searches

### Lambda Performance

```spl
# Lambda cold start analysis
index=aws sourcetype=aws:cloudwatch namespace="AWS/Lambda"
| where metric_name="Duration"
| stats avg(value) as avg_duration, perc95(value) as p95_duration, max(value) as max_duration by FunctionName
| eval timeout_risk = if(max_duration > 0.8 * timeout_ms, "HIGH", "OK")

# Lambda error correlation
index=aws sourcetype=aws:cloudwatch namespace="AWS/Lambda" metric_name="Errors"
| timechart span=5m sum(value) as errors by FunctionName
| where errors > 0
```

### ECS/Fargate Health

```spl
# ECS service CPU/memory
index=aws sourcetype=aws:cloudwatch namespace="AWS/ECS"
| timechart span=5m avg(CPUUtilization) as cpu, avg(MemoryUtilization) as mem by ServiceName

# ECS task failures
index=aws sourcetype=aws:cloudwatch:log logGroup="/ecs/*"
| search "STOPPED" OR "OOMKilled" OR "Essential container exited"
| stats count by taskArn, stoppedReason
```

### RDS Health

```spl
# RDS performance overview
index=aws sourcetype=aws:cloudwatch namespace="AWS/RDS"
| timechart span=5m avg(CPUUtilization) as cpu, avg(ReadLatency) as read_ms, avg(WriteLatency) as write_ms, avg(DatabaseConnections) as conns by DBInstanceIdentifier

# RDS storage trending
index=aws sourcetype=aws:cloudwatch namespace="AWS/RDS" metric_name="FreeStorageSpace"
| timechart span=1h min(value) as free_bytes by DBInstanceIdentifier
| eval free_gb = round(free_bytes/1073741824, 1)
```

### API Gateway

```spl
# API Gateway error rates
index=aws sourcetype=aws:cloudwatch namespace="AWS/ApiGateway"
| timechart span=5m sum(4XXError) as client_errors, sum(5XXError) as server_errors, sum(Count) as total by ApiName
| eval error_pct = round((client_errors+server_errors)/total*100, 2)
```

## Security Monitoring Searches

### CloudTrail Threat Hunting

```spl
# Unusual API activity by service (recon indicator)
index=aws sourcetype=aws:cloudtrail
| stats dc(eventName) as unique_apis, count as total_calls by userIdentity.arn, eventSource
| where unique_apis > 20
| sort -unique_apis

# Cross-account activity
index=aws sourcetype=aws:cloudtrail
| where 'userIdentity.accountId' != "123456789012"
| stats count by userIdentity.accountId, userIdentity.arn, eventName

# Security group changes
index=aws sourcetype=aws:cloudtrail eventName IN ("AuthorizeSecurityGroupIngress","AuthorizeSecurityGroupEgress","RevokeSecurityGroupIngress","CreateSecurityGroup")
| stats count by userIdentity.arn, eventName, requestParameters.groupId

# S3 bucket policy changes (data exposure risk)
index=aws sourcetype=aws:cloudtrail eventName IN ("PutBucketPolicy","PutBucketAcl","PutBucketPublicAccessBlock","DeleteBucketPolicy")
| stats count by userIdentity.arn, eventName, requestParameters.bucketName

# KMS key deletion (critical)
index=aws sourcetype=aws:cloudtrail eventName IN ("ScheduleKeyDeletion","DisableKey")
| eval severity="critical"
```

### GuardDuty Triage

```spl
# GuardDuty findings by severity
index=aws sourcetype=aws:cloudwatch:guardduty
| stats count by severity, type
| sort -severity, -count

# GuardDuty high/critical findings timeline
index=aws sourcetype=aws:cloudwatch:guardduty severity>=7
| timechart span=1h count by type
```

### IAM Risk Analysis

```spl
# Users with most permissions
index=aws sourcetype=aws:cloudtrail eventName="AttachUserPolicy" OR eventName="PutUserPolicy"
| stats count as policy_attachments, values(requestParameters.policyArn) as policies by requestParameters.userName
| sort -policy_attachments

# Access key age (keys > 90 days = risk)
index=aws sourcetype=aws:cloudtrail eventName="CreateAccessKey"
| stats earliest(_time) as created by userIdentity.arn, responseElements.accessKey.accessKeyId
| eval age_days = round((now() - created)/86400, 0)
| where age_days > 90
```

## AWS Cost Monitoring in Splunk

```spl
# Estimate data transfer costs from flow logs
index=aws sourcetype=aws:vpc:flow-logs action=ACCEPT
| where NOT cidrmatch("10.0.0.0/8", dstaddr)
| stats sum(bytes) as outbound_bytes
| eval outbound_gb = round(outbound_bytes/1073741824, 2)
| eval estimated_cost = round(outbound_gb * 0.09, 2)
| eval alert = "Estimated outbound transfer: ".outbound_gb."GB ($".estimated_cost.")"
```
