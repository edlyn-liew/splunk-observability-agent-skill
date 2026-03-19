# Meraki Reference

## Official Documentation

### Product Docs
- [Meraki Documentation](https://documentation.meraki.com)
- [Syslog Event Types & Log Samples](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Syslog_Event_Types_and_Log_Samples)
- [Syslog Server Configuration](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Syslog_Server_Overview_and_Configuration)
- [Event Log](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Meraki_Event_Log)
- [Health Overview](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Meraki_Health_Overview)

### MX Security Appliance
- [Security Center](https://documentation.meraki.com/SASE_and_SD-WAN/MX/Operate_and_Maintain/Monitoring_and_Reporting/Security_Center)
- [Threat Protection (IDS/IPS)](https://documentation.meraki.com/SASE_and_SD-WAN/MX/Operate_and_Maintain/Content_Filtering_and_Threat_Protection/Threat_Protection)
- [Content Filtering](https://documentation.meraki.com/SASE_and_SD-WAN/MX/Operate_and_Maintain/Content_Filtering_and_Threat_Protection/Content_Filtering)
- [SD-WAN Monitoring](https://documentation.meraki.com/SASE_and_SD-WAN/MX/Operate_and_Maintain/Monitoring_and_Reporting/SD-WAN_Monitoring)

### MR Wireless
- [AP Health Details](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Meraki_Health_Overview/Meraki_Health_-_MR_Access_Point_Details)
- [Air Marshal (Rogue AP)](https://documentation.meraki.com/MR/Monitoring_and_Reporting/Air_Marshal)

### MS Switching
- [Switch Overview & Statistics](https://documentation.meraki.com/Switching/MS_-_Switches/Operate_and_Maintain/Monitoring_and_Reporting/Switch_Overview_and_Statistics)

### API & Integration
- [Dashboard API v1](https://developer.cisco.com/meraki/api-v1/)
- [API Index (All Endpoints)](https://developer.cisco.com/meraki/api-v1/api-index/)
- [API Rate Limits](https://developer.cisco.com/meraki/api-v1/rate-limit/)
- [Get Organization Devices](https://developer.cisco.com/meraki/api-v1/get-organization-devices/)
- [Webhooks](https://developer.cisco.com/meraki/webhooks/)
- [Splunk Add-on Guide](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/How-Tos/Cisco_Meraki_Add-on_for_Splunk)
- [Splunkbase App](https://splunkbase.splunk.com/app/5580)

## API Hierarchy

```
Organization (org_id)
├── Network (network_id)
│   ├── MX Appliance (serial)
│   ├── MR Access Point (serial)
│   ├── MS Switch (serial)
│   └── Clients (mac)
├── Network 2
└── ...
```

Key API endpoints:

| Endpoint | Purpose |
|----------|---------|
| `GET /organizations` | List all orgs |
| `GET /organizations/{id}/devices` | All devices in org |
| `GET /organizations/{id}/devices/statuses` | Device online/offline status |
| `GET /networks/{id}/clients` | Clients on a network |
| `GET /devices/{serial}/clients` | Clients on a specific device |
| `GET /organizations/{id}/appliance/vpn/statuses` | VPN tunnel status |
| `GET /networks/{id}/wireless/clientCountHistory` | Wireless client history |
| `GET /networks/{id}/switch/ports/statuses` | Switch port status |
| `GET /organizations/{id}/security/events` | Security events |

Rate limit: 10 req/sec per org. Use `Retry-After` header on 429.

## Syslog Configuration

### MX Syslog Categories

| Category | Content | Priority |
|----------|---------|----------|
| **Event Log** | VPN events, DHCP, firewall rules, uplink changes | 1-4 |
| **IDS Alerts** | Snort 3 signature matches, AMP detections | 1-2 |
| **URLs** | Content filtering allow/deny with URL and category | 2-4 |
| **Flows** | Firewall flow starts and flow ends with src/dst/port | 3-4 |

### MR Syslog Categories

| Category | Content |
|----------|---------|
| **Event Log** | Association, disassociation, auth, deauth, roaming |
| **URLs** | Client web browsing (if configured) |
| **Flows** | Client traffic flows |

### MS Syslog
Event Log only: port up/down, STP changes, DHCP snooping, PoE events.

## MX Security Features

### IDS/IPS Monitoring

Snort 3 engine with Cisco Talos rulesets. Three ruleset modes:

| Mode | Sensitivity | Use Case |
|------|------------|----------|
| Connectivity | Low | Minimize false positives, basic threats |
| Balanced | Medium | General purpose (recommended) |
| Security | High | Maximum protection, may block legitimate traffic |

```spl
# IDS alerts by signature and severity
index=meraki sourcetype=meraki:syslog category="IDS Alerts"
| stats count by signature, priority, src, dst
| sort priority, -count

# Top targeted destinations
index=meraki sourcetype=meraki:syslog category="IDS Alerts"
| stats count by dst, dstport
| sort -count | head 10
```

### Content Filtering

106 content categories, 20 threat areas. Talos Intelligence backend.

```spl
# Blocked content categories
index=meraki sourcetype=meraki:syslog category="URLs" action="deny"
| stats count by content_category, src
| sort -count

# Users hitting blocked categories
index=meraki sourcetype=meraki:syslog category="URLs" action="deny"
| stats dc(url) as unique_urls, count as total_blocks by src
| sort -total_blocks
```

### AMP (Advanced Malware Protection)

File reputation checking: 500M+ known files, 1.5M+ new daily samples.

## MR Wireless Monitoring

### Key Wireless Metrics

| Metric | Good | Warning | Critical |
|--------|------|---------|----------|
| Channel utilization | < 50% | 50-70% | > 70% |
| Client signal (dBm) | > -65 | -65 to -75 | < -75 |
| SNR (dB) | > 25 | 15-25 | < 15 |
| Client onboarding success | > 95% | 85-95% | < 85% |
| AP CPU utilization | < 50% | 50-80% | > 80% |

### Onboarding Steps (Health API)

Client connection journey tracked in steps: Association → Authentication → DHCP → DNS. Failure at any step indicates a specific issue.

```spl
# Onboarding failure analysis
index=meraki sourcetype=meraki:api:health
| where status != "success"
| stats count by failedStep, ssid, apName
| sort -count
```

### Air Marshal — Rogue AP Detection

Dedicated radio scans for unauthorized APs. Categories: Rogue (connected to your wired network), Other (nearby but not connected).

## MS Switch Monitoring

```spl
# Port status overview
index=meraki sourcetype=meraki:api:switchports
| stats latest(status) as status, latest(speed) as speed by switchName, portId
| where status = "Connected"

# PoE power consumption
index=meraki sourcetype=meraki:api:switchports
| where poeEnabled = true
| stats sum(powerUsageInWh) as total_power by switchName
```

## SD-WAN / VPN Monitoring

```spl
# VPN tunnel health
index=meraki sourcetype=meraki:api:vpn
| stats latest(status) as status, latest(latency) as latency_ms by networkName, peerName
| eval health = case(status="active" AND latency_ms<100, "Healthy", status="active", "Degraded", true(), "Down")

# Uplink failover tracking
index=meraki sourcetype=meraki:syslog
| search "uplink" ("change" OR "failover" OR "down")
| timechart span=1h count as failovers by device
```

## Meraki + ITSI Service Templates

| Service | KPIs | Source |
|---------|------|--------|
| Branch Network | VPN status, uplink health, MX CPU | API + syslog |
| Wireless Infrastructure | AP availability, client count, channel util | API + syslog |
| Network Security | IDS alerts, content filter blocks, AMP detections | Syslog |
| Switch Infrastructure | Port availability, PoE health, STP events | API + syslog |
