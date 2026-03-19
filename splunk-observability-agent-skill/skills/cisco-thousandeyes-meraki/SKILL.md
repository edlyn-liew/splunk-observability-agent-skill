---
name: cisco-thousandeyes-meraki
description: >
  Monitor Cisco ThousandEyes and Cisco Meraki infrastructure in Splunk — synthetic monitoring,
  internet outage detection, path visualization, SD-WAN/VPN health, wireless AP management,
  switch monitoring, IDS/IPS threat protection, and rogue AP detection. Use when the user asks
  to "set up ThousandEyes in Splunk", "monitor internet performance", "detect network outages",
  "analyze ThousandEyes test results", "monitor BGP routes", "set up synthetic monitoring",
  "track DNS resolution", "monitor SD-WAN with ThousandEyes", "set up Meraki in Splunk",
  "monitor Meraki wireless", "monitor Meraki switches", "set up Meraki security monitoring",
  "detect rogue APs with Air Marshal", "monitor Meraki VPN tunnels", "analyze Meraki IDS alerts",
  "track Meraki client health", "set up Meraki webhooks for Splunk", "monitor MX appliance",
  "set up content filtering monitoring", "analyze Meraki event logs", or needs guidance on
  ThousandEyes API integration, Meraki Dashboard API, Meraki syslog configuration, network
  performance monitoring, or Cisco infrastructure monitoring patterns in Splunk.
metadata:
  version: "0.1.0"
---

# Cisco ThousandEyes & Meraki Monitoring in Splunk

Monitor internet performance, branch network health, wireless infrastructure, and security posture using Cisco ThousandEyes and Meraki integrated with Splunk.

## Official Documentation

### ThousandEyes
- [ThousandEyes Docs](https://docs.thousandeyes.com/)
- [ThousandEyes API (Cisco DevNet)](https://developer.cisco.com/docs/thousandeyes/)
- [ThousandEyes Splunk App](https://docs.thousandeyes.com/product-documentation/integration-guides/custom-built-integrations/splunk-app)
- [ThousandEyes Splunkbase](https://splunkbase.splunk.com/app/7719)
- [ThousandEyes Webhook → Splunk](https://docs.thousandeyes.com/product-documentation/integration-guides/custom-webhook-examples/splunk-alert-notifs)

### Meraki
- [Meraki Documentation](https://documentation.meraki.com)
- [Meraki Dashboard API (Cisco DevNet)](https://developer.cisco.com/meraki/api-v1/)
- [Meraki Splunk Add-on Guide](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/How-Tos/Cisco_Meraki_Add-on_for_Splunk)
- [Meraki Splunkbase](https://splunkbase.splunk.com/app/5580)
- [Meraki Webhooks](https://developer.cisco.com/meraki/webhooks/)

---

## ThousandEyes Overview

### What ThousandEyes Monitors

| Category | Test Types | Key Metrics |
|----------|-----------|-------------|
| **Network** | Agent-to-Server, Agent-to-Agent | Latency, loss, jitter, path MTU, throughput |
| **DNS** | DNS Server, DNS Trace, DNSSEC | Resolution time, availability, mapping accuracy |
| **Web** | HTTP Server, Page Load, Transaction | Response time, availability, DOM load, waterfall |
| **BGP** | BGP Route monitoring | AS path changes, reachability, route leaks/hijacks |
| **Voice** | SIP Server, RTP Stream | MOS score, jitter, packet loss, DSCP marking |

### Agent Types

| Agent | Deployment | Use Case |
|-------|-----------|----------|
| **Cloud Agent** | Managed by ThousandEyes (1057 globally) | External vantage point, SaaS monitoring |
| **Enterprise Agent** | On-prem VM or container | Internal network, data center, branch office |
| **Endpoint Agent** | Windows/macOS desktop agent | End-user experience, browser sessions, WiFi |

### Core Metrics Reference

| Metric | Description | Good | Warning | Critical |
|--------|-------------|------|---------|----------|
| Latency (ms) | Round-trip time | < 50ms | 50-150ms | > 150ms |
| Packet Loss (%) | Packets not reaching dest | < 0.5% | 0.5-2% | > 2% |
| Jitter (ms) | Variation in latency | < 5ms | 5-15ms | > 15ms |
| Availability (%) | Successful connections | > 99.9% | 99-99.9% | < 99% |
| DNS Resolution (ms) | DNS lookup time | < 50ms | 50-200ms | > 200ms |
| HTTP Response (ms) | Time to first byte | < 200ms | 200-1000ms | > 1000ms |
| Page Load (s) | Full page load time | < 3s | 3-8s | > 8s |
| MOS Score | Voice quality | > 4.0 | 3.5-4.0 | < 3.5 |

### Splunk Integration

**Data flow**: ThousandEyes → HEC Streaming → Splunk

Configure in ThousandEyes: Integrations > Splunk, provide HEC token and endpoint.

**Webhook alerts**: ThousandEyes → Webhook → Splunk HEC

Search for alerts:
```spl
eventType=THOUSANDEYES_ALERT_NOTIFICATION
| stats count by alertType, testName, agents{}.agentName
```

### ThousandEyes SPL Patterns

```spl
# Network test results — latency and loss by test
index=thousandeyes sourcetype=thousandeyes test_type="agent-to-server"
| timechart span=5m avg(latency) as avg_latency, avg(loss) as avg_loss by testName

# DNS resolution performance
index=thousandeyes sourcetype=thousandeyes test_type="dns-server"
| stats avg(resolutionTime) as avg_dns, perc95(resolutionTime) as p95_dns by domain, server

# HTTP availability trending
index=thousandeyes sourcetype=thousandeyes test_type="http-server"
| timechart span=15m avg(availability) as availability by testName
| where availability < 100

# BGP route changes
index=thousandeyes sourcetype=thousandeyes test_type="bgp"
| where isnotnull(asPathChange)
| stats count by prefix, peerASN, asPathChange

# Path visualization — hop-by-hop analysis
index=thousandeyes sourcetype=thousandeyes test_type="agent-to-server"
| spath input=pathTrace
| stats avg(delay) as hop_latency by hop_ip, hop_asn, hop_prefix

# Active alerts summary
index=thousandeyes eventType=THOUSANDEYES_ALERT_NOTIFICATION
| stats count as active_alerts by alertRule, severity, testName
| sort -severity, -active_alerts

# Internet Insights — outage correlation
index=thousandeyes sourcetype=thousandeyes:insights
| stats count by outageType, affectedProvider, affectedRegion
```

### ThousandEyes Monitoring Perspectives

**Infrastructure**: Agent-to-server tests for WAN circuits, SD-WAN overlay, data center connectivity, DNS infrastructure health, BGP route stability.

**Security**: DNS hijacking detection (unexpected DNS mappings), BGP route leaks/hijacks (AS path anomalies), path deviations through unexpected networks, DNSSEC validation failures.

---

## Meraki Overview

### Product Portfolio

| Product | Model | Monitored Via | Key Metrics |
|---------|-------|--------------|-------------|
| **MX** | Security Appliance / SD-WAN | Syslog, API, Webhooks | VPN status, throughput, IDS/IPS alerts, content filtering |
| **MR** | Wireless Access Points | Syslog, API, Webhooks | Client count, signal quality, channel utilization, onboarding |
| **MS** | Switches | Syslog, API | Port status, traffic, PoE, STP, DHCP snooping |
| **MV** | Cameras | API | Motion alerts, connectivity |
| **MG** | Cellular Gateways | API | Signal strength, data usage, failover |

### Data Collection Methods

| Method | Data Type | Latency | Setup |
|--------|----------|---------|-------|
| **Syslog** | Events, IDS alerts, URLs, Flows | Real-time | MX/MR/MS → Syslog server → Splunk |
| **REST API** | Device status, metrics, config | Polled (rate-limited) | Splunk Add-on scheduled inputs |
| **Webhooks** | Alerts, config changes | Real-time | Dashboard → HEC endpoint |

### Meraki Syslog Event Categories

| Product | Categories Available |
|---------|---------------------|
| MX | Event Log, IDS Alerts, URLs, Flows |
| MR | Event Log, URLs, Flows |
| MS | Event Log only |

Syslog priority levels: 1 (high) through 4 (very low).

Docs: [Syslog Event Types](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Syslog_Event_Types_and_Log_Samples)

### API Rate Limits

Per organization: **10 requests/second** (burst: 30 in first 2 seconds). Per IP: 100 req/sec, 10 concurrent. HTTP 429 on exceeded, use `Retry-After` header.

Docs: [API Rate Limits](https://developer.cisco.com/meraki/api-v1/rate-limit/)

### Meraki SPL Patterns

```spl
# Device availability overview
index=meraki sourcetype=meraki:api:devices
| stats latest(status) as status, latest(lanIp) as ip by name, model, networkId
| eval is_offline = if(status="offline", 1, 0)
| where is_offline = 1

# MX security — IDS/IPS alerts
index=meraki sourcetype=meraki:syslog category="IDS Alerts"
| stats count by signature, priority, src, dst
| sort -count

# MX security — content filtering blocks
index=meraki sourcetype=meraki:syslog category="URLs"
| rex "(?<action>allow|deny)\s+(?<url>\S+)"
| where action="deny"
| stats count by url, src | sort -count

# MR wireless — client count per AP
index=meraki sourcetype=meraki:api:clients
| stats dc(mac) as client_count by apName, ssid
| sort -client_count

# MR wireless — rogue AP detection (Air Marshal)
index=meraki sourcetype=meraki:syslog
| search "rogue" OR "Air Marshal"
| stats count by bssid, ssid, channel
| lookup authorized_aps bssid OUTPUT authorized
| where isnull(authorized)

# MS switch — port security events
index=meraki sourcetype=meraki:syslog
| search "port_security" OR "dhcp_snooping" OR "arp_inspection"
| stats count by type, port, switch_name

# VPN tunnel status
index=meraki sourcetype=meraki:api:vpn
| stats latest(status) as vpn_status by networkName, peerName
| where vpn_status != "active"

# SD-WAN uplink failover events
index=meraki sourcetype=meraki:syslog
| search "uplink" AND ("change" OR "failover")
| table _time, device, interface, status

# Webhook alert summary
index=meraki sourcetype=meraki:webhook
| stats count by alertType, networkName, deviceName
| sort -count

# Client health — onboarding failures
index=meraki sourcetype=meraki:api:health
| where onboardingStep != "success"
| stats count by onboardingStep, ssid, apName
| sort -count
```

### Meraki Security Monitoring

**IDS/IPS (MX)** — Snort 3 engine with Cisco Talos rulesets. Three modes: Connectivity (least restrictive), Balanced, Security (most restrictive). Alerts forwarded via syslog.

**AMP (Advanced Malware Protection)** — File reputation checking against 500M+ known files, 1.5M+ new samples/day via Cisco AMP cloud.

**Content Filtering** — 106 content categories, 20 threat areas. URL + SNI inspection with Talos Intelligence. Local cache (100K records, 20-min TTL).

**Air Marshal** — Rogue AP detection via dedicated radio. BSSID categorization (Rogue/Other). Containment via deauthentication packets.

Docs: [Threat Protection](https://documentation.meraki.com/SASE_and_SD-WAN/MX/Operate_and_Maintain/Content_Filtering_and_Threat_Protection/Threat_Protection) · [Content Filtering](https://documentation.meraki.com/SASE_and_SD-WAN/MX/Operate_and_Maintain/Content_Filtering_and_Threat_Protection/Content_Filtering) · [Air Marshal](https://documentation.meraki.com/MR/Monitoring_and_Reporting/Air_Marshal)

---

## Combined ThousandEyes + Meraki Patterns

### End-to-End Branch Monitoring

```
ThousandEyes Enterprise Agent (branch) → Tests → Internet/SaaS performance
Meraki MX (branch) → VPN, firewall, IDS → Branch security
Meraki MR (branch) → WiFi health → User connectivity
Meraki MS (branch) → Switch ports → Wired infrastructure
```

### Correlation Searches

```spl
# Correlate ThousandEyes outage with Meraki VPN drops
index=thousandeyes eventType=THOUSANDEYES_ALERT_NOTIFICATION alertType="network"
| append [search index=meraki sourcetype=meraki:syslog "vpn" "down"]
| timechart span=5m count by source

# Branch health score — combine wireless + WAN + internet
index=meraki sourcetype=meraki:api:health OR index=thousandeyes test_type="agent-to-server"
| stats avg(healthScore) as meraki_health, avg(availability) as te_availability by site
| eval branch_health = round((meraki_health + te_availability) / 2, 1)
```

### Alert Thresholds

| Source | Alert | Condition | Priority |
|--------|-------|-----------|----------|
| ThousandEyes | Internet outage | Availability < 95% | Critical |
| ThousandEyes | BGP route hijack | Unexpected AS path | Critical |
| ThousandEyes | DNS hijacking | Unexpected resolution | Critical |
| ThousandEyes | Latency degradation | > 2x baseline | Warning |
| Meraki MX | IDS/IPS critical alert | Priority 1 signature | Critical |
| Meraki MX | VPN tunnel down | Status != active | High |
| Meraki MX | Content filter bypass | Blocked category accessed | Medium |
| Meraki MR | Rogue AP detected | Air Marshal alert | Critical |
| Meraki MR | AP offline | Status = offline > 5 min | High |
| Meraki MR | Client onboarding failure | > 10% failure rate | Warning |
| Meraki MS | Port security violation | DHCP snooping / ARP inspection | High |

For detailed reference files, read:
- `references/thousandeyes-reference.md` — ThousandEyes test types, API endpoints, path visualization, Internet Insights
- `references/meraki-reference.md` — Meraki API hierarchy, syslog parsing, MX/MR/MS monitoring, security features
