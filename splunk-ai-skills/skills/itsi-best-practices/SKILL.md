---
name: itsi-best-practices
description: >
  Best practices for Splunk IT Service Intelligence (ITSI) — creating and configuring services,
  KPIs, service templates, predictive analytics, glass tables, event analytics, and correlation
  searches. Use when the user asks to "create an ITSI service", "configure KPIs", "build a glass
  table", "set up predictive analytics", "configure ITSI thresholds", "create service templates",
  "set up correlation searches", "configure event analytics", "manage episodes", "design ITSI
  dashboards", "model network services in ITSI", "monitor application health in ITSI",
  "create security services in ITSI", "build infrastructure service trees",
  "troubleshoot ITSI", or needs guidance on ITSI architecture, entity management,
  service health scoring, anomaly detection, or KPI drift detection for network engineering,
  network security, application monitoring, or security monitoring use cases.
metadata:
  version: "0.2.0"
---

# ITSI Best Practices

Comprehensive guide for configuring and optimizing Splunk IT Service Intelligence (ITSI) — services, KPIs, glass tables, predictive analytics, event analytics, and correlation searches across all monitoring perspectives.

## Service Creation Workflow

### Step 1: Plan Services

Before deployment, compile a list of services, KPIs, and glass table views. Align services with business outcomes, not just infrastructure components.

**Service hierarchy pattern**: Business Service > Application Service > Infrastructure Service > Component

### Step 2: Create a Service

1. Navigate to **Configuration > Services** in ITSI
2. Click **Create Service**
3. Provide a title, description, and select a security group
4. Define entity rules to dynamically associate entities

### Step 3: Define Entity Rules

Entity rules dynamically filter KPI searches based on entity alias matches.

- Use entity alias fields (e.g., `host`, `ip`, `service_name`) for matching
- Combine multiple rules with AND/OR logic
- Test rules to verify correct entity association before saving

### Step 4: Add KPIs

Best practice: **20 or fewer KPIs per service** — enough for CPU, IO, disk, response time, etc.

## KPI Configuration

### KPI Types

- **Generic KPI**: Created from scratch with a custom search
- **KPI from Template**: Pre-configured for specific use cases
- **KPI from Base Search**: Shares a search definition across multiple KPIs to reduce search load

### KPI Base Searches

Base searches consolidate similar KPIs, reduce search load, and improve performance.

**Performance warning**: If the same KPI base search is used by too many services (e.g., a template with 10 KPIs linked to 12 services = 120 base search instances), performance degrades.

### Threshold Configuration

Severity levels: **Critical, High, Medium, Low, Normal**

- **Static thresholds**: Fixed values. Use time-based static for daily/weekly patterns.
- **Adaptive thresholds**: ML-driven, auto-adjusting. Preferred for variable baselines.
- **Time-based static**: Different thresholds for business hours vs. off-hours.

## Service Health Score

Weighted average of KPI severity levels and dependency health scores.

- KPI severity: critical=4, high=3, medium=2, low=1, normal=0
- Higher importance values = more weight in the health score
- Assign highest importance to KPIs that directly impact end-user experience

## Service Templates

1. **Create from a working service** (UI preferred over REST API)
2. Keep templates focused on a single technology or use case
3. Use content packs for common technologies
4. Linked KPIs show a lock icon and inherit template changes
5. Bulk import services and auto-link to templates

## Glass Tables

### Creation Workflow

1. Navigate to **Glass Tables** > **Create Glass Table**
2. Choose layout: **Absolute** (pixel-positioned) or **Grid** (auto-filling)
3. Set canvas size and background
4. Add visualizations: KPI widgets, ad hoc search widgets, charts, shapes, icons, text

### Best Practices

1. Use background images representing your infrastructure topology
2. Place KPI widgets on corresponding infrastructure components
3. Use absolute layout with auto-scale for responsive display
4. Configure service swapping for multi-service views
5. Edit source definition directly for advanced customization

## Predictive Analytics

- Requires 24+ hours of historical data and Java 8-11
- Predicts service health 30 minutes ahead
- **KPI Drift Detection** catches slow-trending deviations
- **Entity Cohesion** detects outlier entities (minimum 4 entities)

## Event Analytics

**Workflow**: Data sources > Correlation searches > Notable events > Aggregation policies > Episodes

- Select **5-10 fields** for aggregation (fewer = overly broad, more = many single-event episodes)
- Normalize fields before creating aggregation policies
- Use clearing events to auto-close episodes
- Test correlation searches in ad hoc mode before scheduling

## Monitoring Perspectives in ITSI

ITSI services should be organized by monitoring perspective. Each perspective has characteristic service hierarchies, KPIs, and thresholds.

### Network Engineering Services

**Typical service tree**:
```
WAN Network Service
├── Core Router Health
├── Edge Router Health
├── Firewall Health
├── Load Balancer Health
└── WAN Link Performance
```

**Essential KPIs**:

| KPI | Search Pattern | Threshold (Critical) |
|-----|---------------|---------------------|
| Interface Availability | `index=network sourcetype=snmp \| stats latest(ifOperStatus) by interface` | = 0 (down) |
| Bandwidth Utilization % | `index=netflow \| eval util=bytes*8/link_capacity*100` | > 90% |
| Network Latency (ms) | `index=network \| stats avg(round_trip_time) by dest` | > 200ms |
| Packet Loss % | `index=network \| eval loss=lost/sent*100` | > 2% |
| Error Rate (CRC/Collisions) | `index=snmp \| stats sum(ifInErrors) by host, interface` | > 100/5min |
| BGP Session Status | `index=network sourcetype=bgp \| stats latest(state) by peer` | != Established |

### Network Security Services

**Typical service tree**:
```
Perimeter Security Service
├── Firewall Threat Detection
├── IDS/IPS Health
├── DNS Security
├── Web Proxy Health
└── VPN Gateway Health
```

**Essential KPIs**:

| KPI | Search Pattern | Threshold (Critical) |
|-----|---------------|---------------------|
| Threat Events/Hour | `index=security category=threat \| timechart span=1h count` | > 500 |
| IDS Alert Rate | `index=ids severity=critical \| timechart span=5m count` | > 50/5min |
| Blocked Connection Rate | `index=firewall action=blocked \| timechart span=5m count` | Anomaly detection |
| DNS Query Anomaly | `index=dns \| stats dc(query) by src \| where queries > 200` | > 200 unique/hr |
| SSL Certificate Expiry | `\| inputlookup cert_inventory \| eval days_left=...` | < 7 days |
| Firewall CPU/Memory | `index=firewall sourcetype=pan:system \| stats latest(cpu_pct)` | > 90% |

### Application Monitoring Services

**Typical service tree**:
```
E-Commerce Platform
├── Web Frontend Service
│   ├── Response Time KPI
│   ├── Error Rate KPI
│   └── Request Rate KPI
├── API Backend Service
│   ├── API Latency KPI
│   ├── API Error Rate KPI
│   └── API Throughput KPI
├── Database Service
│   ├── Query Latency KPI
│   ├── Connection Pool KPI
│   └── Replication Lag KPI
└── Infrastructure Service
    ├── CPU Utilization KPI
    ├── Memory Usage KPI
    └── Disk I/O KPI
```

**Essential KPIs (RED Method)**:

| KPI | Search Pattern | Threshold (Critical) |
|-----|---------------|---------------------|
| Error Rate % | `index=web \| eval err=if(status>=500,1,0) \| stats avg(err)` | > 5% |
| P95 Response Time | `index=web \| stats perc95(response_time) by service` | > 2000ms |
| Request Rate | `index=web \| timechart span=1m count` | < 50% of baseline |
| Apdex Score | Custom eval with satisfied/tolerating/frustrated buckets | < 0.7 |
| Active Sessions | `index=app \| stats dc(session_id)` | > capacity threshold |
| Database Connection Pool | `index=db \| stats latest(active_connections)/max_connections*100` | > 90% |

### Security Monitoring Services

**Typical service tree**:
```
SOC Operations Service
├── Authentication Health
│   ├── Failed Login Rate
│   ├── Brute Force Incidents
│   └── MFA Adoption Rate
├── Endpoint Security
│   ├── EDR Agent Health
│   ├── Malware Detection Rate
│   └── Vulnerability Score
├── Threat Intelligence
│   ├── IOC Match Rate
│   ├── Feed Freshness
│   └── False Positive Rate
└── Compliance
    ├── Policy Violations
    ├── Audit Log Coverage
    └── Configuration Drift
```

**Essential KPIs**:

| KPI | Search Pattern | Threshold (Critical) |
|-----|---------------|---------------------|
| Failed Login Rate | `index=auth action=failure \| timechart span=1h count` | > 100/hr |
| Active Brute Force | `index=auth action=failure \| stats count by src \| where count>10` | Any match |
| Risk Score (Entity) | `index=risk \| stats sum(risk_score) by risk_object` | > 100 |
| MTTD (Hours) | `\| eval mttd=(detect_time-event_time)/3600 \| stats avg(mttd)` | > 4 hours |
| MTTR (Hours) | `\| eval mttr=(resolve_time-detect_time)/3600 \| stats avg(mttr)` | > 24 hours |
| EDR Coverage % | `\| stats dc(hosts_with_agent)/dc(all_hosts)*100` | < 95% |
| Unpatched Critical Vulns | `index=vuln severity=critical status=open \| stats count` | > 0 |

## Glass Table Templates by Perspective

### Network Topology Glass Table

- Background: network diagram (data center layout, WAN topology)
- Widgets: interface status (green/red), bandwidth gauges, latency heatmaps
- Service swapping: swap between data center locations

### Application Health Glass Table

- Background: application architecture diagram (microservices map)
- Widgets: RED metrics per service, dependency arrows, database health
- Real-time: request flow visualization with error highlighting

### Security Operations Glass Table

- Background: security zone diagram (DMZ, internal, external)
- Widgets: threat event counters, risk score gauges, authentication heatmaps
- Real-time: active incident counters, compliance scores

## Deployment Planning

- For >200 discrete KPIs, use a **search head cluster**
- Forward all internal data from search heads to indexers
- Start with critical services and expand iteratively
- Use content packs for common technology stacks

For detailed reference on thresholds, entity management, and advanced configurations, read:
- `references/itsi-detailed-reference.md` — Full ITSI configuration patterns
- `references/itsi-monitoring-perspectives.md` — Complete KPI definitions and service templates by perspective
