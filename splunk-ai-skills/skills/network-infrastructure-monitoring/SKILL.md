---
name: network-infrastructure-monitoring
description: >
  Comprehensive network infrastructure monitoring in Splunk — DNS/DHCP, routers, switches,
  firewalls, wireless, SD-WAN, internet circuits, SNMP, NetFlow, and Layer 2 security.
  Use when the user asks to "monitor DNS", "track DHCP leases", "monitor routers", "monitor
  switches", "set up SNMP monitoring", "analyze NetFlow", "monitor internet circuits",
  "detect rogue DHCP", "monitor wireless access points", "track bandwidth utilization",
  "detect ARP poisoning", "monitor BGP sessions", "track interface errors", "set up syslog
  parsing", "monitor WAN performance", "detect port security violations", "track MAC addresses",
  "monitor SD-WAN", "calculate network SLA", "detect DNS tunneling", "monitor DHCP scope
  exhaustion", "track network configuration changes", "monitor OSPF adjacency", "parse
  Cisco syslog", "set up SC4S", or needs guidance on network device monitoring, SNMP OIDs,
  syslog field extraction, flow analysis, network SLA calculation, or vendor-specific
  monitoring patterns (Cisco, Juniper, Palo Alto, Fortinet, Arista, Meraki).
metadata:
  version: "0.1.0"
---

# Network Infrastructure Monitoring

Monitor, analyze, and secure network infrastructure in Splunk — covering DNS, DHCP, routers, switches, firewalls, wireless, SD-WAN, internet circuits, and Layer 2 security.

## Data Collection Architecture

### Collection Methods

| Method | Use Case | Tools |
|--------|----------|-------|
| **Syslog** | Device events, state changes, auth | Splunk Connect for Syslog (SC4S), HEC |
| **SNMP Polling** | Interface metrics, CPU/mem, counters | SC4SNMP, scripted inputs |
| **SNMP Traps** | Real-time alerts from devices | SC4SNMP trap receiver |
| **NetFlow/sFlow/IPFIX** | Traffic flow analysis, bandwidth | NetFlow add-on, NetFlow Logic |
| **REST API** | Cloud-managed devices (Meraki, SD-WAN) | Modular inputs, UCC TAs |
| **Streaming** | Packet-level analysis | Splunk Stream |

### Recommended Index Strategy

```
index=network          # Syslog from network devices
index=netflow          # Flow data (NetFlow, sFlow, IPFIX)
index=netmetrics       # SNMP metric data (SC4SNMP)
index=dns              # DNS query/response logs
index=dhcp             # DHCP lease events
index=wireless         # Wireless AP and controller logs
index=infrastructure   # Ping, availability, synthetic tests
```

### Vendor-Specific Sourcetypes

| Vendor | Sourcetype | Add-on |
|--------|-----------|--------|
| Cisco IOS/IOS-XE/NX-OS | `cisco:ios` | TA-cisco-ios |
| Cisco ASA | `cisco:asa` | Splunk Add-on for Cisco ASA |
| Cisco Meraki | `meraki:*` | Splunk Add-on for Cisco Meraki |
| Palo Alto Networks | `pan:traffic`, `pan:threat`, `pan:system` | Splunk Add-on for PAN |
| Fortinet FortiGate | `fortinet:fortigate` | Fortinet FortiGate Add-on |
| Juniper | `juniper:junos`, `juniper:srx` | Splunk Add-on for Juniper |
| Arista | `arista:eos` | Arista EOS Add-on |
| F5 BIG-IP | `f5:bigip_ltm`, `f5:bigip_gtm` | Splunk Add-on for F5 |
| ISC BIND (DNS) | `bind` | Splunk Add-on for ISC BIND |
| ISC DHCP | `isc_dhcp` | Splunk Add-on for ISC DHCP |
| Windows DNS | `WinEventLog:DNS` | Splunk Add-on for Windows |
| Windows DHCP | `WinEventLog:DHCP` | Splunk Add-on for Windows |

## DNS Monitoring

### Key Metrics

| KPI | Search Pattern | Alert Threshold |
|-----|---------------|-----------------|
| Query Volume | `index=dns \| timechart span=5m count` | > 3x baseline |
| Response Time | `index=dns \| stats avg(response_time) by server` | > 100ms |
| NXDOMAIN Rate | `index=dns reply_code=NXDOMAIN \| timechart span=5m count` | > 100/5min per src |
| Query Type Distribution | `index=dns \| stats count by query_type` | Unusual spikes in TXT/NULL |
| Unique Domains per Host | `index=dns \| stats dc(query) by src` | > 500/hour (DGA indicator) |
| DNS Server Availability | `index=infrastructure dest_port=53 \| stats latest(status) by dest` | status != up |

### DNS Threat Detection

```spl
# DGA domain detection (entropy-based)
index=dns
| rex field=query "(?<sld>[^.]+)\.[^.]+$"
| eval sld_len = len(sld)
| eval char_set = replace(sld, "(.)\1+", "\1")
| eval entropy_approx = len(char_set) / sld_len
| where sld_len > 12 AND entropy_approx > 0.7
| stats count as suspicious_queries, dc(query) as unique_domains by src
| where suspicious_queries > 20

# DNS tunneling detection (high TXT/NULL query volume)
index=dns query_type IN ("TXT", "NULL", "CNAME")
| stats count as queries, avg(len(query)) as avg_query_len by src
| where queries > 50 AND avg_query_len > 40

# NXDOMAIN flood (DGA cycling through domains)
index=dns reply_code=NXDOMAIN
| timechart span=5m count by src
| where count > 100
```

### DNS Operational Monitoring

```spl
# DNS server response time degradation
index=dns
| timechart span=5m avg(response_time) as avg_rt, perc95(response_time) as p95_rt by server
| where p95_rt > 100

# Zone transfer monitoring
index=dns query_type=AXFR OR query_type=IXFR
| stats count by src, dest, query
| where src NOT IN (known_secondary_dns_servers)

# Top queried domains
index=dns
| stats count by query
| sort -count
| head 50
```

## DHCP Monitoring

### Key Metrics

| KPI | Search Pattern | Alert Threshold |
|-----|---------------|-----------------|
| Scope Utilization % | Leased addresses / Total scope | > 80% warning, > 90% critical |
| Lease Rate | `index=dhcp action=lease \| timechart span=1h count` | Anomaly detection |
| DHCP Server Availability | Response to DISCOVER | No OFFER within 5s |
| Rogue DHCP Servers | Unauthorized OFFER responses | Any detection |
| MAC Address Anomalies | New/unknown MACs requesting leases | Alert on non-inventory MACs |

### DHCP Searches

```spl
# Scope utilization tracking
index=dhcp action=lease
| stats dc(lease_address) as leased by scope
| lookup dhcp_scope_sizes scope OUTPUT total_addresses
| eval utilization = round(leased / total_addresses * 100, 1)
| where utilization > 80
| sort -utilization

# Rogue DHCP server detection
index=network sourcetype=cisco:ios facility=DHCP mnemonic=DHCP_SNOOPING_UNTRUST
| stats count by src_mac, src_ip, vlan, interface
| table _time, src_mac, src_ip, vlan, interface

# Rapid lease churn (possible reconnaissance or DoS)
index=dhcp
| stats count by mac_address
| where count > 50
| sort -count

# DHCP lease correlation with DNS
index=dhcp action=lease
| join lease_address [search index=dns | rename src as lease_address | fields lease_address, query]
| stats values(query) as dns_queries by lease_address, mac_address, hostname
```

## Router & Switch Monitoring

### SNMP OID Reference

| OID | Name | Description | Type |
|-----|------|-------------|------|
| 1.3.6.1.2.1.2.2.1.8 | `ifOperStatus` | Interface operational state (1=up, 2=down) | Gauge |
| 1.3.6.1.2.1.2.2.1.10 | `ifInOctets` | Total octets received | Counter |
| 1.3.6.1.2.1.2.2.1.16 | `ifOutOctets` | Total octets transmitted | Counter |
| 1.3.6.1.2.1.2.2.1.13 | `ifInDiscards` | Inbound packets discarded | Counter |
| 1.3.6.1.2.1.2.2.1.14 | `ifInErrors` | Inbound packets with errors | Counter |
| 1.3.6.1.2.1.2.2.1.19 | `ifOutDiscards` | Outbound packets discarded | Counter |
| 1.3.6.1.2.1.2.2.1.20 | `ifOutErrors` | Outbound packets with errors | Counter |
| 1.3.6.1.2.1.1.3.0 | `sysUpTime` | Time since last reboot | TimeTicks |
| 1.3.6.1.4.1.9.2.1.57.0 | `avgBusy5` | Cisco CPU 5-min avg | Gauge |
| 1.3.6.1.4.1.9.9.48.1.1.1.5 | `ciscoMemoryPoolUsed` | Cisco memory used | Gauge |
| 1.3.6.1.4.1.9.9.48.1.1.1.6 | `ciscoMemoryPoolFree` | Cisco memory free | Gauge |

### Interface Monitoring Searches

```spl
# Interface status changes (syslog)
index=network sourcetype=cisco:ios mnemonic=LINK* OR mnemonic=LINEPROTO*
| rex "Interface (?<interface>\S+), changed state to (?<state>\w+)"
| stats count by host, interface, state
| sort -_time

# Bandwidth utilization from SNMP
index=netmetrics sourcetype=sc4snmp:metric
| eval in_mbps = round(sc4snmp.IF-MIB.ifInOctets * 8 / 1000000, 2)
| eval out_mbps = round(sc4snmp.IF-MIB.ifOutOctets * 8 / 1000000, 2)
| timechart span=5m avg(in_mbps) as inbound, avg(out_mbps) as outbound by host

# Interface error rate
index=netmetrics sourcetype=sc4snmp:metric
| timechart span=5m sum(sc4snmp.IF-MIB.ifInErrors) as in_errors, sum(sc4snmp.IF-MIB.ifOutErrors) as out_errors by host
| where in_errors > 0 OR out_errors > 0

# Device CPU/Memory health
index=network sourcetype=cisco:ios
| rex "CPU utilization for five seconds: (?<cpu_5s>\d+)%"
| stats latest(cpu_5s) as cpu by host
| where cpu > 80
```

### Cisco Syslog Parsing

Cisco syslog format: `%FACILITY-SEVERITY-MNEMONIC: Message`

| Facility | Examples | Monitoring Focus |
|----------|----------|-----------------|
| `LINK` | LINK-3-UPDOWN | Interface state changes |
| `LINEPROTO` | LINEPROTO-5-UPDOWN | Line protocol changes |
| `OSPF` | OSPF-5-ADJCHG | OSPF adjacency changes |
| `BGP` | BGP-5-ADJCHANGE | BGP peer state changes |
| `DHCP` | DHCP_SNOOPING | DHCP snooping violations |
| `PORT_SECURITY` | PSECURE_VIOLATION | MAC address security |
| `SYS` | SYS-5-RESTART | Device restarts |
| `SNMP` | SNMP-3-AUTHFAIL | SNMP auth failures |
| `SEC_LOGIN` | SEC_LOGIN-4-LOGIN_FAILED | Login failures |

## Internet & WAN Monitoring

### Circuit Performance KPIs

| KPI | Measurement | Warning | Critical |
|-----|------------|---------|----------|
| Circuit Availability % | Uptime / total time | < 99.9% | < 99.5% |
| Round-Trip Latency | ICMP ping avg RTT | > 80ms | > 150ms |
| Jitter | Stddev of RTT samples | > 15ms | > 30ms |
| Packet Loss % | Lost / total sent | > 0.5% | > 2% |
| Bandwidth Utilization | Used / total capacity | > 70% | > 85% |

### SLA Calculation

```spl
# Monthly SLA calculation per circuit
index=infrastructure sourcetype=ping earliest=-30d
| eval is_up = if(status="success", 1, 0)
| stats count as total_checks, sum(is_up) as successful by circuit_id
| eval availability_pct = round(successful / total_checks * 100, 4)
| eval sla_met = if(availability_pct >= 99.9, "Yes", "NO - BREACH")
| eval downtime_minutes = round((total_checks - successful) * 5, 0)
| table circuit_id, availability_pct, sla_met, downtime_minutes
```

### Routing Protocol Monitoring

```spl
# BGP session state changes
index=network sourcetype=cisco:ios mnemonic=ADJCHANGE facility=BGP
| rex "neighbor (?<peer_ip>\S+)\s+.*state (?<new_state>\w+)"
| stats count by host, peer_ip, new_state
| where new_state != "Established"

# OSPF adjacency changes
index=network sourcetype=cisco:ios mnemonic=ADJCHG facility=OSPF
| rex "Nbr (?<neighbor>\S+) on (?<interface>\S+).*to (?<state>\w+)"
| stats count by host, neighbor, interface, state
| where state != "FULL"

# Route flapping detection
index=network sourcetype=cisco:ios (mnemonic=ADJCHANGE OR mnemonic=ADJCHG)
| timechart span=5m count by host
| where count > 5
```

## Wireless Monitoring

### Access Point Health

```spl
# AP availability
index=wireless sourcetype=cisco:wlc
| stats latest(ap_status) as status, latest(_time) as last_seen by ap_name, ap_mac
| eval hours_since = round((now() - last_seen) / 3600, 1)
| where status != "online" OR hours_since > 0.5

# Client count per AP
index=wireless sourcetype=cisco:wlc event_type=association
| stats dc(client_mac) as connected_clients by ap_name, ssid
| sort -connected_clients

# Rogue AP detection
index=wireless sourcetype=cisco:wlc event_type=rogue_ap
| stats count, values(channel) as channels, values(rssi) as signal_strength by rogue_mac, rogue_ssid
```

## Layer 2 Security

### Port Security & MAC Tracking

```spl
# Port security violations
index=network sourcetype=cisco:ios facility=PORT_SECURITY mnemonic=PSECURE_VIOLATION
| stats count by host, interface, src_mac, vlan
| sort -count

# ARP poisoning detection (DAI violations)
index=network sourcetype=cisco:ios facility=SW_DAI mnemonic=DHCP_SNOOPING_MATCH_MAC_FAIL
| stats count by src_mac, src_ip, vlan, interface

# Unauthorized device detection
index=dhcp action=lease
| lookup authorized_devices mac_address OUTPUT device_name, authorized
| where NOT authorized="yes"
| table _time, mac_address, lease_address, hostname

# MAC address flapping (same MAC on multiple ports)
index=network sourcetype=cisco:ios mnemonic=MAC_MOVE_NOTIFICATION
| rex "from (?<old_port>\S+) to (?<new_port>\S+)"
| stats count by host, mac_address, old_port, new_port, vlan
| where count > 3
```

## Network Change Management

```spl
# Configuration change events
index=network sourcetype=cisco:ios mnemonic=CONFIG_I OR mnemonic=SYS_CONFIG
| stats count by host, user, _time
| sort -_time

# Unauthorized change detection (outside maintenance window)
index=network sourcetype=cisco:ios mnemonic=CONFIG_I
| eval hour = strftime(_time, "%H"), dow = strftime(_time, "%w")
| where (hour < 6 OR hour > 22) OR (dow = 0 OR dow = 6)
| table _time, host, user
```

## SLA Targets Reference

| SLA Level | Annual Downtime | Monthly Downtime |
|-----------|----------------|-----------------|
| 99.0% | 3 days 15.6 hrs | 7 hrs 18 min |
| 99.5% | 1 day 19.8 hrs | 3 hrs 39 min |
| 99.9% | 8 hrs 45.6 min | 43.8 min |
| 99.95% | 4 hrs 22.8 min | 21.9 min |
| 99.99% | 52.6 min | 4.38 min |
| 99.999% | 5.26 min | 26.3 sec |

For detailed reference files covering specific topics in depth, read:
- `references/dns-dhcp-monitoring.md` — DNS/DHCP deep dive with threat detection, scope management, and vendor-specific patterns
- `references/router-switch-snmp.md` — Router/switch monitoring, SNMP OID catalog, syslog parsing, and vendor configuration
- `references/wan-internet-wireless.md` — WAN circuits, BGP/OSPF, SD-WAN, wireless, and SLA calculations
- `references/layer2-security-changes.md` — ARP/MAC security, port security, network change management, and compliance
