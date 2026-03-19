# UCC Monitoring Perspectives Reference

## CIM Data Model Compliance

Every TA should map its fields to the appropriate CIM data models. This ensures compatibility with Splunk Enterprise Security, ITSI, and cross-vendor correlation.

### Field Mapping via props.conf

```ini
[my_sourcetype]
FIELDALIAS-src = source_address AS src
FIELDALIAS-dest = destination_address AS dest
FIELDALIAS-action = policy_action AS action
EVAL-vendor_product = "MyVendor MyProduct"
LOOKUP-action = my_action_lookup vendor_action AS raw_action OUTPUT action
```

### Field Mapping via transforms.conf

```ini
[my_action_lookup]
filename = my_action_lookup.csv
```

---

## Perspective 1: Network Engineering

### Popular Splunkbase Apps to Study

| App | What It Does |
|-----|-------------|
| Splunk Add-on for Palo Alto Networks | PAN firewall/Panorama/Cortex data collection |
| Splunk Add-on for Cisco ASA | Cisco ASA firewall log parsing |
| Splunk Add-on for F5 BIG-IP | F5 LTM/GTM load balancer data |
| NetFlow and SNMP Analytics | NetFlow v5/v9, sFlow, IPFIX ingestion |
| Network Monitoring App | Self-service network dashboards |
| Network Toolkit | Troubleshooting (ping, traceroute, DNS, whois) |

### Data Sources & Collection Methods

| Source | Collection | Sourcetype Pattern |
|--------|------------|-------------------|
| Firewalls | Syslog (UDP 514/TCP 6514) | `vendor:product` (e.g., `pan:traffic`) |
| Switches/Routers | SNMP traps, syslog | `cisco:ios`, `juniper:junos` |
| Load Balancers | Syslog, REST API | `f5:bigip_ltm`, `f5:bigip_gtm` |
| Flow Data | NetFlow/sFlow/IPFIX collectors | `netflow`, `sflow`, `ipfix` |
| Wireless Controllers | Syslog, API | `cisco:wlc`, `aruba:controller` |

### CIM: Network_Traffic Data Model

Required field mappings:

| CIM Field | Description | Example Mapping |
|-----------|-------------|----------------|
| `src` | Source IP | `FIELDALIAS-src = srcip AS src` |
| `dest` | Destination IP | `FIELDALIAS-dest = dstip AS dest` |
| `src_port` | Source port | `FIELDALIAS-sport = srcport AS src_port` |
| `dest_port` | Destination port | `FIELDALIAS-dport = dstport AS dest_port` |
| `action` | allowed/blocked/dropped | `LOOKUP-action = action_lookup ...` |
| `transport` | Protocol (tcp/udp/icmp) | `FIELDALIAS-transport = proto AS transport` |
| `bytes_in` | Inbound bytes | `FIELDALIAS-bin = rcvdbytes AS bytes_in` |
| `bytes_out` | Outbound bytes | `FIELDALIAS-bout = sentbytes AS bytes_out` |
| `packets_in` | Inbound packets | `FIELDALIAS-pin = rcvdpkts AS packets_in` |
| `packets_out` | Outbound packets | `FIELDALIAS-pout = sentpkts AS packets_out` |
| `rule` | Firewall rule name/ID | `FIELDALIAS-rule = policy_name AS rule` |
| `vendor_product` | Vendor identification | `EVAL-vendor_product = "Palo Alto Networks Firewall"` |

### Key KPIs for Network Engineering

| KPI | SPL Pattern |
|-----|------------|
| Availability % | `\| stats count(eval(status="up")) as up, count as total \| eval avail=round(up/total*100,2)` |
| Bandwidth Utilization | `\| eval bps=(bytes*8)/(last_switched-first_switched) \| stats avg(bps) by interface` |
| Network Latency (ms) | `\| stats avg(latency) as avg_latency, perc95(latency) as p95_latency by dest` |
| Packet Loss % | `\| eval loss_pct=round(packets_lost/packets_total*100,2)` |
| Interface Error Rate | `\| stats sum(crc_errors) as errors, sum(packets) as total \| eval error_pct=round(errors/total*100,4)` |
| Top Talkers | `\| stats sum(bytes) as total_bytes by src \| sort -total_bytes \| head 10` |
| Connection Rate | `\| timechart span=5m count by action` |

### Example globalConfig Input: Syslog Network Device

```json
{
  "name": "network_syslog",
  "title": "Network Device Syslog",
  "entity": [
    {"type": "text", "field": "name", "label": "Input Name", "required": true},
    {"type": "text", "field": "syslog_port", "label": "Syslog Port", "defaultValue": "514",
     "validators": [{"type": "number", "range": [1, 65535]}]},
    {"type": "singleSelect", "field": "protocol", "label": "Protocol",
     "options": {"items": [{"value": "udp", "label": "UDP"}, {"value": "tcp", "label": "TCP"}]},
     "defaultValue": "udp"},
    {"type": "singleSelect", "field": "vendor", "label": "Device Vendor",
     "options": {"items": [
       {"value": "paloalto", "label": "Palo Alto Networks"},
       {"value": "cisco", "label": "Cisco"},
       {"value": "fortinet", "label": "Fortinet"},
       {"value": "juniper", "label": "Juniper"}
     ]}}
  ]
}
```

---

## Perspective 2: Network Security

### Popular Splunkbase Apps

| App | What It Does |
|-----|-------------|
| Splunk Enterprise Security (ES) | Core SIEM with correlation searches |
| Network Behavior Analytics | DNS/HTTP/TLS anomaly detection, DGA, C2 detection |
| Splunk Add-on for Palo Alto Networks | Threat logs, WildFire, URL filtering |
| Fortinet FortiGate Add-on | FortiGate IPS/IDS/firewall logs |
| OT Security Add-on | Operational technology threat detection |

### CIM: Intrusion_Detection Data Model

| CIM Field | Description |
|-----------|-------------|
| `signature` | IDS/IPS rule name or signature ID |
| `severity` | Alert severity (informational, low, medium, high, critical) |
| `category` | Attack category |
| `src` | Source IP of attacker |
| `dest` | Target IP |
| `action` | allowed/blocked |
| `vendor_product` | Source system identifier |

### CIM: Network_Resolution (DNS)

| CIM Field | Description |
|-----------|-------------|
| `query` | DNS query domain |
| `query_type` | A, AAAA, MX, TXT, etc. |
| `answer` | DNS response IP/value |
| `src` | Client making the query |
| `dest` | DNS server |
| `message_type` | query/response |

### Key KPIs for Network Security

| KPI | SPL Pattern |
|-----|------------|
| Threat Events/Hour | `\| timechart span=1h count where category="threat"` |
| Blocked Connection % | `\| stats count(eval(action="blocked")) as blocked, count as total \| eval blocked_pct=round(blocked/total*100,2)` |
| DGA Domain Score | `\| eval entropy=-1*sum(p*log2(p)) per domain_chars \| where entropy > 3.5 AND len(query) > 15` |
| DNS Tunnel Indicator | `\| stats dc(query) as unique_queries by src \| where unique_queries > 200` |
| C2 Callback Ratio | `\| stats sum(bytes_out) as out, sum(bytes_in) as in by dest \| eval ratio=out/in \| where ratio > 10` |
| IDS Signature Hits | `\| stats count by signature, severity \| sort -count` |
| Firewall Rule Coverage | `\| stats dc(rule) as active_rules \| appendcols [inputlookup all_rules.csv \| stats count as total_rules]` |

---

## Perspective 3: Application Monitoring

### Popular Splunkbase Apps

| App | What It Does |
|-----|-------------|
| Splunk APM | OpenTelemetry-native trace collection, RED metrics |
| Splunk Infrastructure Monitoring | Infrastructure metrics integrated with APM |
| Splunk Add-on for Apache Web Server | Apache access/error log parsing |
| Splunk Add-on for NGINX | NGINX access/error log parsing |
| Content Pack for Observability Cloud | Pre-built ITSI dashboards for app health |
| Log Observer Connect | Centralized log analysis alongside metrics |

### CIM: Web Data Model

| CIM Field | Description |
|-----------|-------------|
| `status` | HTTP status code |
| `url` | Full URL path |
| `http_method` | GET, POST, PUT, DELETE |
| `http_user_agent` | Client user agent string |
| `response_time` | Response time in ms |
| `bytes` | Response body size |
| `src` | Client IP |
| `dest` | Server IP |
| `http_referrer` | Referrer URL |
| `user` | Authenticated user |

### Key KPIs for Application Monitoring (RED Method)

| KPI | SPL Pattern |
|-----|------------|
| **R**equest Rate (req/sec) | `\| timechart span=1m count as requests \| eval rps=requests/60` |
| **E**rror Rate % | `\| eval is_error=if(status>=500,1,0) \| stats avg(is_error) as error_rate \| eval error_pct=round(error_rate*100,2)` |
| **D**uration (Latency) | `\| stats avg(response_time) as avg_rt, perc95(response_time) as p95_rt, perc99(response_time) as p99_rt by uri` |
| Apdex Score | `\| eval bucket=case(rt<=T,"satisfied",rt<=4*T,"tolerating",true(),"frustrated") \| stats count(eval(bucket="satisfied")) as s, count(eval(bucket="tolerating")) as t, count as total \| eval apdex=(s+0.5*t)/total` |
| Throughput (bytes/sec) | `\| timechart span=1m sum(bytes) as total_bytes \| eval mbps=round(total_bytes/1048576/60,2)` |
| Error Budget | `\| eval target=0.999 \| eval actual=1-(errors/total) \| eval budget_remaining=round((actual-target)/target*100,2)` |

### Example globalConfig Input: Web Application

```json
{
  "name": "webapp_monitor",
  "title": "Web Application Monitor",
  "entity": [
    {"type": "text", "field": "name", "label": "Application Name", "required": true},
    {"type": "text", "field": "endpoint_url", "label": "Health Check URL", "required": true,
     "validators": [{"type": "regex", "pattern": "^https?://.*", "errorMsg": "Must be a valid URL"}]},
    {"type": "text", "field": "interval", "label": "Check Interval (seconds)", "defaultValue": "60"},
    {"type": "text", "field": "response_threshold", "label": "Response Time Alert Threshold (ms)", "defaultValue": "500"},
    {"type": "singleSelect", "field": "environment", "label": "Environment",
     "options": {"items": [
       {"value": "production", "label": "Production"},
       {"value": "staging", "label": "Staging"},
       {"value": "development", "label": "Development"}
     ]}}
  ]
}
```

---

## Perspective 4: Security Monitoring

### Popular Splunkbase Apps

| App | What It Does |
|-----|-------------|
| Splunk Enterprise Security (ES) | Comprehensive SIEM with risk-based alerting |
| Splunk ES Essentials | Risk-based alerting, 90% alert reduction |
| Splunk UEBA | User and entity behavior analytics |
| Splunk Security Content Update (ESCU) | Pre-built detections aligned to MITRE ATT&CK |
| Splunk App for PCI Compliance | PCI DSS compliance monitoring |
| Rapid7 InsightVM Add-on | Vulnerability management integration |

### CIM: Authentication Data Model

| CIM Field | Description |
|-----------|-------------|
| `action` | success/failure |
| `user` | Username attempting authentication |
| `src` | Source IP/hostname |
| `dest` | Target system |
| `app` | Application name |
| `authentication_method` | password, certificate, mfa, sso |
| `signature` | Event signature/code (e.g., EventCode 4624/4625) |
| `reason` | Failure reason |

### CIM: Change Data Model

| CIM Field | Description |
|-----------|-------------|
| `action` | created/modified/deleted |
| `object` | What was changed |
| `object_category` | Type of object (user, policy, file) |
| `user` | Who made the change |
| `command` | Command executed |
| `result` | success/failure |

### Key KPIs for Security Monitoring

| KPI | SPL Pattern |
|-----|------------|
| Failed Logins/Hour | `\| where action="failure" \| timechart span=1h count by user` |
| Brute Force Detection | `\| stats count as failures by src, user \| where failures > 5` |
| Successful Brute Force | `\| transaction src maxspan=1h \| where eventcount > 5 AND action="success"` |
| Risk Score (Composite) | `\| stats sum(risk_score) as total_risk by risk_object \| where total_risk > 100` |
| Privileged Account Usage | `\| where user IN ("admin","root","sa") \| stats count by user, src, dest` |
| Threat Intel Matches | `\| lookup threat_intel_ip ip as src OUTPUT threat_category \| where isnotnull(threat_category)` |
| Mean Time to Detect (MTTD) | `\| eval mttd=detection_time - event_time \| stats avg(mttd)` |
| Mean Time to Respond (MTTR) | `\| eval mttr=response_time - detection_time \| stats avg(mttr)` |

### Windows Security Event Codes

| EventCode | Description | Use Case |
|-----------|-------------|----------|
| 4624 | Successful logon | Authentication monitoring |
| 4625 | Failed logon | Brute force detection |
| 4648 | Logon with explicit credentials | Pass-the-hash detection |
| 4720 | User account created | Account management |
| 4722 | User account enabled | Account management |
| 4726 | User account deleted | Account management |
| 4732 | Member added to security group | Privilege escalation |
| 4756 | Member added to universal group | Privilege escalation |
| 4768 | Kerberos TGT requested | Kerberoasting |
| 4769 | Kerberos service ticket requested | Kerberoasting |
| 4771 | Kerberos pre-auth failed | Account lockout |
| 7045 | New service installed | Persistence detection |

---

## Cross-Perspective Best Practices

### CIM Compliance Checklist

1. Map all relevant fields to CIM field names
2. Use `FIELDALIAS` in props.conf for simple renames
3. Use `LOOKUP` for value translations (e.g., vendor action codes to CIM actions)
4. Use `EVAL` for computed fields (e.g., `vendor_product`)
5. Tag events for data model membership (e.g., `tag=network, tag=communicate`)
6. Validate with the CIM validator app from Splunkbase
7. Document all field mappings in your TA's README

### Sourcetype Naming Conventions

```
vendor:product                    # e.g., pan:traffic
vendor:product:type               # e.g., cisco:asa:firewall
vendor:product:component:type     # e.g., aws:cloudtrail:management:event
```

### Alert Tuning Across Perspectives

- Start conservative, tune down over time
- Use suppression lookups for known-good activity
- Implement dynamic throttling for duplicate alerts
- Adopt risk-based alerting to prioritize high-confidence events
- Baseline thresholds per environment (prod vs dev differ)
