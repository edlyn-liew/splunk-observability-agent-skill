# ITSI Monitoring Perspectives Reference

Complete service definitions, KPI specifications, and correlation search patterns for each monitoring perspective.

---

## Network Engineering — Full Service Specifications

### Service Template: Core Network Infrastructure

```
Service: Core Network Infrastructure
Description: Monitors availability, performance, and capacity of core network devices
Entity Rules: host=router-* OR host=switch-* OR host=fw-*
Dependencies: None (root service)
```

**KPIs:**

| KPI Name | Base Search | Metric | Aggregate | Entity Split | Importance | Critical Threshold |
|----------|-------------|--------|-----------|--------------|------------|-------------------|
| Device Availability | `index=infrastructure sourcetype=ping \| stats latest(status) by host` | status | count where status=1 / count | host | 11 (highest) | < 99% |
| CPU Utilization | `index=infrastructure sourcetype=snmp:cpu \| stats latest(cpu_pct) by host` | cpu_pct | avg | host | 8 | > 90% |
| Memory Utilization | `index=infrastructure sourcetype=snmp:memory \| stats latest(mem_pct) by host` | mem_pct | avg | host | 7 | > 90% |
| Interface Utilization | `index=netflow \| eval util_pct=bytes*8/link_bps*100 \| stats avg(util_pct) by host, interface` | util_pct | avg | host, interface | 9 | > 85% |
| Interface Errors | `index=infrastructure sourcetype=snmp:if \| stats sum(ifInErrors) as errors by host, interface` | errors | sum | host, interface | 8 | > 100 in 5min |
| BGP Peer Status | `index=network sourcetype=bgp \| stats latest(peer_state) by host, peer` | peer_state | count where state!=Established | host | 10 | Any non-Established |

### Service Template: WAN Performance

```
Service: WAN Link Performance
Description: Monitors WAN circuit utilization, latency, jitter, and loss
Entity Rules: circuit=wan-* OR link_type=mpls OR link_type=internet
Dependencies: Core Network Infrastructure
```

**KPIs:**

| KPI Name | Metric | Critical | High | Medium |
|----------|--------|----------|------|--------|
| Circuit Utilization % | bandwidth used / total capacity | > 90% | > 80% | > 70% |
| Round-Trip Latency (ms) | avg round-trip time | > 200ms | > 150ms | > 100ms |
| Jitter (ms) | stddev of latency samples | > 30ms | > 20ms | > 10ms |
| Packet Loss % | lost packets / total sent | > 2% | > 1% | > 0.5% |
| Circuit Availability | uptime / total time | < 99% | < 99.5% | < 99.9% |

### Correlation Searches: Network Engineering

```spl
# Interface Down Alert
index=infrastructure sourcetype=snmp:if ifOperStatus=2
| stats count by host, ifDescr, ifAlias
| where count > 0
| eval severity="critical", title="Interface Down: ".host." - ".ifDescr

# Bandwidth Saturation
index=netflow
| timechart span=5m sum(bytes) as total_bytes by src_interface
| eval mbps = total_bytes * 8 / 300 / 1000000
| where mbps > (capacity_mbps * 0.85)
| eval severity="high", title="Bandwidth saturation on ".src_interface

# Device Unreachable
index=infrastructure sourcetype=ping status=0
| stats count as failures, latest(_time) as last_fail by host
| where failures >= 3
| eval severity="critical", title="Device unreachable: ".host
```

---

## Network Security — Full Service Specifications

### Service Template: Perimeter Security

```
Service: Perimeter Security
Description: Monitors firewall health, IDS/IPS detection rates, and perimeter threat posture
Entity Rules: host=fw-* OR host=ids-* OR host=ips-*
Dependencies: Core Network Infrastructure
```

**KPIs:**

| KPI Name | Base Search | Critical Threshold |
|----------|-------------|-------------------|
| Firewall Throughput | `index=firewall \| timechart span=5m sum(bytes) by host` | < 50% of baseline (degraded) |
| Threat Detection Rate | `index=firewall category=threat \| timechart span=1h count` | > 500/hr (under attack) |
| IDS Alert Volume | `index=ids \| timechart span=5m count by severity` | > 100 critical/hr |
| Blocked Connections/min | `index=firewall action=blocked \| timechart span=1m count` | Adaptive threshold |
| Firewall Session Table Usage | `index=firewall sourcetype=pan:system \| stats latest(session_pct) by host` | > 90% |
| Policy Violation Rate | `index=firewall \| stats count(eval(action="blocked" AND policy_violation=1))` | Any in prod zones |

### Service Template: DNS Security

```
Service: DNS Security
Description: Monitors DNS infrastructure for DGA, tunneling, and poisoning
Entity Rules: host=dns-* OR service=dns
Dependencies: Core Network Infrastructure
```

**KPIs:**

| KPI Name | Search Logic | Critical |
|----------|-------------|----------|
| DNS Query Volume | `index=dns \| timechart span=5m count` | > 3x baseline |
| Unique Domain Rate | `index=dns \| stats dc(query) as unique by src` | > 500/hr per src |
| NXDOMAIN Rate | `index=dns reply_code=NXDOMAIN \| timechart span=5m count` | > 100/5min |
| High Entropy Domains | `index=dns \| eval entropy=... \| where entropy > 3.5` | Any detection |
| DNS Response Latency | `index=dns \| stats avg(response_time) by server` | > 100ms |
| Threat Intel DNS Matches | `index=dns \| lookup threat_domains domain as query OUTPUT threat` | Any match |

### Correlation Searches: Network Security

```spl
# DGA Domain Detection
index=dns
| eval domain_len=len(query), char_variety=dc(split(query,""))
| eval entropy_score=log(char_variety) * domain_len / log(domain_len+1)
| where entropy_score > 3.5 AND domain_len > 15
| stats count as dga_hits, values(query) as domains by src
| where dga_hits > 5
| eval severity="high", title="Possible DGA activity from ".src

# Port Scan Detection
index=firewall action=blocked
| stats dc(dest_port) as unique_ports, count as attempts by src, dest
| where unique_ports > 20 AND attempts > 50
| eval severity="high", title="Port scan detected: ".src." -> ".dest

# Data Exfiltration Indicator
index=firewall action=allowed direction=outbound
| stats sum(bytes_out) as total_out by src
| where total_out > 1073741824
| eval gb=round(total_out/1073741824,2)
| eval severity="medium", title=src." transferred ".gb."GB outbound"
```

---

## Application Monitoring — Full Service Specifications

### Service Template: Web Application

```
Service: Web Application
Description: Monitors web application using RED method (Rate, Errors, Duration)
Entity Rules: app_name=myapp AND environment=production
Dependencies: Database Service, Cache Service
```

**KPIs (RED Method):**

| KPI Name | Base Search | Critical | High | Medium |
|----------|-------------|----------|------|--------|
| Request Rate (req/s) | `index=web sourcetype=access_combined \| timechart span=1m count` | < 50% baseline | < 70% baseline | < 80% baseline |
| Error Rate % | `index=web \| eval err=if(status>=500,1,0) \| stats avg(err)*100` | > 5% | > 2% | > 1% |
| P50 Response Time | `index=web \| stats median(response_time) by service` | > 500ms | > 300ms | > 200ms |
| P95 Response Time | `index=web \| stats perc95(response_time) by service` | > 2000ms | > 1000ms | > 500ms |
| P99 Response Time | `index=web \| stats perc99(response_time) by service` | > 5000ms | > 3000ms | > 1500ms |
| 4xx Error Rate | `index=web \| eval c4xx=if(status>=400 AND status<500,1,0) \| stats avg(c4xx)*100` | > 10% | > 5% | > 2% |

### Service Template: Database Service

```
Service: Database Service
Description: Monitors database performance, connections, and replication
Entity Rules: host=db-* OR service_type=database
Dependencies: Infrastructure Service
```

**KPIs:**

| KPI Name | Critical | High |
|----------|----------|------|
| Query Latency (avg ms) | > 500ms | > 200ms |
| Slow Query Count | > 50/5min | > 20/5min |
| Connection Pool Utilization % | > 90% | > 80% |
| Replication Lag (seconds) | > 60s | > 30s |
| Deadlock Count | > 5/5min | > 1/5min |
| Cache Hit Ratio % | < 80% | < 90% |

### Correlation Searches: Application Monitoring

```spl
# Error Rate Spike
index=web sourcetype=access_combined
| timechart span=5m count(eval(status>=500)) as errors, count as total
| eval error_rate = round(errors/total*100, 2)
| where error_rate > 5
| eval severity="critical", title="Error rate spike: ".error_rate."%"

# Response Time Degradation
index=web sourcetype=access_combined
| timechart span=5m perc95(response_time) as p95
| predict p95 as predicted algorithm=LLP5
| eval deviation = abs(p95 - predicted) / predicted * 100
| where deviation > 50 AND p95 > 1000
| eval severity="high", title="Response time degradation: P95=".p95."ms"

# Throughput Drop
index=web sourcetype=access_combined
| timechart span=5m count as requests
| streamstats window=24 avg(requests) as baseline
| eval drop_pct = round((baseline - requests) / baseline * 100, 1)
| where drop_pct > 50
| eval severity="critical", title="Throughput dropped ".drop_pct."% below baseline"
```

---

## Security Monitoring — Full Service Specifications

### Service Template: Authentication Health

```
Service: Authentication Health
Description: Monitors authentication success/failure patterns and brute force indicators
Entity Rules: sourcetype=WinEventLog:Security OR sourcetype=linux_secure
Dependencies: None
```

**KPIs:**

| KPI Name | Base Search | Critical |
|----------|-------------|----------|
| Failed Login Rate | `index=auth action=failure \| timechart span=1h count` | > 100/hr |
| Brute Force Attempts | `index=auth action=failure \| stats count by src \| where count>10` | Any detection |
| Successful Brute Force | `index=auth \| transaction src maxspan=1h \| where mvcount(action)>5 AND mvindex(action,-1)="success"` | Any detection |
| After-Hours Logins | `index=auth action=success \| where date_hour<6 OR date_hour>22` | Anomaly detection |
| Privileged Account Logins | `index=auth user IN (admin_list) action=success \| stats count` | Baseline + anomaly |
| MFA Bypass Attempts | `index=auth mfa_status=bypassed \| stats count by user` | Any occurrence |

### Service Template: Threat Intelligence Operations

```
Service: Threat Intelligence
Description: Monitors IOC feed health, match rates, and false positive management
Entity Rules: sourcetype=threat_intel* OR service=threat_platform
Dependencies: None
```

**KPIs:**

| KPI Name | Critical |
|----------|----------|
| IOC Feed Freshness (hours since update) | > 24 hours |
| IOC Match Rate (matches/hour) | Adaptive threshold |
| False Positive Rate % | > 30% |
| Coverage (active IOCs count) | < 1000 |
| Threat Intel Lookup Performance (ms) | > 5000ms |

### Correlation Searches: Security Monitoring

```spl
# Brute Force with Successful Login
index=auth
| stats count(eval(action="failure")) as failures, count(eval(action="success")) as successes by src, user
| where failures > 5 AND successes > 0
| eval severity="critical", title="Brute force success: ".user." from ".src." (".failures." failures)"

# Lateral Movement Indicator
index=auth action=success LogonType=3
| stats dc(dest) as unique_targets, values(dest) as targets by src, user
| where unique_targets > 5
| eval severity="high", title=user." logged into ".unique_targets." hosts from ".src

# Privilege Escalation
index=windows EventCode=4732 OR EventCode=4756
| stats count by user, TargetUserName, GroupName
| eval severity="high", title=user." added ".TargetUserName." to ".GroupName

# Risk Score Aggregation
index=risk
| stats sum(risk_score) as total_risk, dc(search_name) as detection_count, values(search_name) as detections by risk_object, risk_object_type
| where total_risk > 100
| eval severity=case(total_risk>200,"critical",total_risk>100,"high",true(),"medium")
| eval title="High risk entity: ".risk_object." (score=".total_risk.", ".detection_count." detections)"
```

---

## Service Template Reuse Patterns

### Template Linking Strategy

| Perspective | Template | Typical Services Linked |
|-------------|----------|------------------------|
| Network Engineering | Core Network Device | 50-200 (one per device) |
| Network Security | Firewall Security | 5-20 (one per firewall cluster) |
| Application Monitoring | Web Application (RED) | 10-50 (one per app/microservice) |
| Security Monitoring | Authentication Health | 3-10 (one per auth source) |

### Performance Guidelines

- 10 KPIs per template x 50 linked services = 500 KPI instances (manageable)
- 10 KPIs per template x 200 linked services = 2000 KPI instances (consider search head cluster)
- Reduce base search count by sharing base searches across KPIs
- Use entity splitting to reduce per-entity search overhead
- Schedule non-critical KPIs at longer intervals (5-15 min vs. 1-5 min)

### Adaptive Threshold Recommendations by Perspective

| Perspective | KPIs Best for Adaptive | KPIs Best for Static |
|-------------|----------------------|---------------------|
| Network Engineering | Bandwidth utilization, latency (variable by time of day) | Interface status (binary up/down), BGP state |
| Network Security | Threat volume, blocked connections (variable by threat landscape) | SSL cert expiry (fixed dates), policy violations |
| Application Monitoring | Request rate, response time (variable by usage patterns) | Error rate SLO (fixed contractual), uptime SLA |
| Security Monitoring | Login volume, risk scores (variable by user behavior) | Compliance violations (zero tolerance), brute force (fixed threshold) |
