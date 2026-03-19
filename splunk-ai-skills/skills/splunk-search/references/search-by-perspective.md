# Splunk Search Patterns by Monitoring Perspective

Production-ready SPL patterns organized by monitoring perspective, with common data sources, fields, and alert patterns.

---

## Network Engineering Searches

### Common Sourcetypes & Indexes

```
index=network sourcetype=pan:traffic          # Palo Alto firewall traffic
index=network sourcetype=cisco:asa            # Cisco ASA firewall
index=network sourcetype=fortinet:fortigate   # Fortinet FortiGate
index=netflow sourcetype=netflow              # NetFlow/sFlow/IPFIX
index=infrastructure sourcetype=snmp:*        # SNMP polling data
index=network sourcetype=cisco:ios            # Cisco IOS syslog
index=network sourcetype=juniper:junos        # Juniper syslog
```

### Bandwidth Analysis

```spl
# Top bandwidth consumers (last hour)
index=netflow earliest=-1h
| stats sum(bytes) as total_bytes by src_ip
| eval total_gb = round(total_bytes/1073741824, 2)
| sort -total_bytes
| head 20
| table src_ip, total_gb

# Interface utilization trend
index=netflow
| eval mbps = bytes * 8 / 300 / 1000000
| timechart span=5m avg(mbps) by src_interface
| where mbps > 0

# Bandwidth by protocol
index=netflow earliest=-4h
| stats sum(bytes) as total_bytes by protocol
| eval pct = round(total_bytes / (sum of total) * 100, 1)
| sort -total_bytes
```

### Device Health

```spl
# Devices with high CPU
index=infrastructure sourcetype=snmp:cpu
| stats latest(cpu_pct) as cpu by host
| where cpu > 80
| sort -cpu

# Interface error summary
index=infrastructure sourcetype=snmp:if
| stats sum(ifInErrors) as in_errors, sum(ifOutErrors) as out_errors by host, ifDescr
| where in_errors > 0 OR out_errors > 0
| sort -(in_errors + out_errors)

# Device availability (last 24h)
index=infrastructure sourcetype=ping earliest=-24h
| stats count(eval(status=1)) as up, count as total by host
| eval availability = round(up/total*100, 2)
| where availability < 99.9
| sort availability
```

### Connection Analysis

```spl
# Firewall connection summary
index=network sourcetype=pan:traffic
| stats count by action
| eval pct = round(count / (total count) * 100, 1)

# Top denied connections
index=network sourcetype=pan:traffic action=denied
| stats count by src, dest, dest_port, app
| sort -count
| head 20

# Connection rate over time
index=network sourcetype=pan:traffic
| timechart span=5m count by action
```

### Latency Monitoring

```spl
# Network latency by destination
index=network sourcetype=ping
| stats avg(round_trip_time) as avg_rtt, perc95(round_trip_time) as p95_rtt, max(round_trip_time) as max_rtt by dest
| where avg_rtt > 50
| sort -avg_rtt

# Latency trend with anomaly detection
index=network sourcetype=ping dest=critical-server-*
| timechart span=5m avg(round_trip_time) as rtt
| predict rtt as predicted algorithm=LLP5
| eval anomaly = if(abs(rtt - predicted) > predicted * 0.5, 1, 0)
```

---

## Network Security Searches

### Common Sourcetypes & Indexes

```
index=security sourcetype=pan:threat          # Palo Alto threat logs
index=security sourcetype=ids                  # Generic IDS
index=security sourcetype=suricata:eve         # Suricata IDS/IPS
index=dns sourcetype=dns                       # DNS query logs
index=proxy sourcetype=bluecoat:proxysg        # Web proxy logs
index=security sourcetype=zeek:*               # Zeek/Bro network analysis
```

### Threat Detection

```spl
# Critical threats in last hour
index=security sourcetype=pan:threat severity=critical earliest=-1h
| stats count by src, dest, threat_name, action
| sort -count

# Threat trends by category
index=security sourcetype=pan:threat
| timechart span=1h count by category

# IDS signature hits ranked by severity
index=security sourcetype=ids
| stats count as hits by signature, severity
| sort severity, -hits
| where severity IN ("critical", "high")
```

### DNS Security Analysis

```spl
# Possible DGA domains (high entropy)
index=dns
| eval domain_parts = split(query, ".")
| eval sld = mvindex(domain_parts, -2) . "." . mvindex(domain_parts, -1)
| eval domain_len = len(mvindex(domain_parts, -2))
| where domain_len > 12
| stats count by sld, src
| where count > 10
| sort -count

# DNS tunneling indicators (high query volume per source)
index=dns
| stats dc(query) as unique_domains, count as total_queries by src
| where unique_domains > 200
| eval ratio = round(total_queries / unique_domains, 1)
| sort -unique_domains

# NXDOMAIN spike detection
index=dns reply_code=NXDOMAIN
| timechart span=5m count as nxdomains
| predict nxdomains as expected algorithm=LLP5
| where nxdomains > expected * 2

# DNS queries to suspicious TLDs
index=dns
| rex field=query "\.(?<tld>[^.]+)$"
| where tld IN ("xyz", "top", "club", "work", "gq", "cf", "tk", "ml", "ga")
| stats count by src, tld
| sort -count
```

### Firewall Policy Analysis

```spl
# Unused firewall rules (no hits in 30 days)
index=network sourcetype=pan:traffic earliest=-30d
| stats count by rule
| append [| inputlookup all_firewall_rules.csv | fields rule]
| stats sum(count) as hits by rule
| where hits = 0 OR isnull(hits)

# Overly permissive rules (any-any)
index=network sourcetype=pan:traffic rule=*any*
| stats count, dc(src) as unique_src, dc(dest) as unique_dest by rule
| sort -count

# Geo-suspicious connections
index=network sourcetype=pan:traffic action=allowed
| iplocation dest
| where Country NOT IN ("United States", "Canada", "United Kingdom")
| stats count by Country, dest
| sort -count
```

### Port Scan Detection

```spl
# Horizontal port scan (one source, many ports on one dest)
index=network sourcetype=pan:traffic action=denied earliest=-15m
| stats dc(dest_port) as unique_ports, count as attempts by src, dest
| where unique_ports > 20
| table src, dest, unique_ports, attempts

# Vertical scan (one source, one port, many destinations)
index=network sourcetype=pan:traffic action=denied earliest=-15m
| stats dc(dest) as unique_dests by src, dest_port
| where unique_dests > 20
| table src, dest_port, unique_dests
```

---

## Application Monitoring Searches

### Common Sourcetypes & Indexes

```
index=web sourcetype=access_combined           # Apache/NGINX access logs
index=web sourcetype=nginx_access              # NGINX specific
index=app sourcetype=app_logs                  # Application logs (custom)
index=traces sourcetype=otel:traces            # OpenTelemetry traces
index=metrics sourcetype=otel:metrics          # OpenTelemetry metrics
index=web sourcetype=iis                       # Microsoft IIS
```

### RED Method Dashboard Searches

```spl
# Request Rate (Rate)
index=web sourcetype=access_combined
| timechart span=1m count as requests
| eval rps = round(requests / 60, 1)

# Error Rate (Errors)
index=web sourcetype=access_combined
| timechart span=5m count(eval(status>=500)) as errors, count as total
| eval error_rate = round(errors/total*100, 2)

# Response Time Distribution (Duration)
index=web sourcetype=access_combined
| stats avg(response_time) as avg_rt, median(response_time) as p50, perc95(response_time) as p95, perc99(response_time) as p99 by uri_path
| sort -p95
| head 20
```

### Error Analysis

```spl
# Error breakdown by status code and URI
index=web sourcetype=access_combined status>=400
| stats count by status, uri_path
| sort -count
| head 30

# Error rate by endpoint over time
index=web sourcetype=access_combined
| eval is_error = if(status >= 500, 1, 0)
| timechart span=5m avg(is_error) as error_rate by uri_path
| where error_rate > 0

# Error spike correlation with deployment
index=web sourcetype=access_combined status>=500
| timechart span=5m count as errors
| appendcols [search index=deploy | timechart span=5m count as deploys]
| where errors > 0 OR deploys > 0
```

### Performance Analysis

```spl
# Slow endpoints (P95 > 1s)
index=web sourcetype=access_combined
| stats perc95(response_time) as p95, count by uri_path
| where p95 > 1000 AND count > 100
| sort -p95

# Response time anomaly detection
index=web sourcetype=access_combined
| timechart span=5m perc95(response_time) as p95_rt
| predict p95_rt as predicted algorithm=LLP5
| eval is_anomaly = if(p95_rt > predicted * 1.5 AND p95_rt > 500, 1, 0)
| where is_anomaly = 1

# Apdex Score calculation (T=500ms threshold)
index=web sourcetype=access_combined
| eval satisfaction = case(
    response_time <= 500, "satisfied",
    response_time <= 2000, "tolerating",
    true(), "frustrated"
  )
| stats count(eval(satisfaction="satisfied")) as satisfied,
        count(eval(satisfaction="tolerating")) as tolerating,
        count as total
| eval apdex = round((satisfied + 0.5 * tolerating) / total, 3)
```

### User Experience

```spl
# Slowest user experiences
index=web sourcetype=access_combined
| stats avg(response_time) as avg_rt, count as requests by clientip
| where requests > 10
| sort -avg_rt
| head 20

# Geographic performance
index=web sourcetype=access_combined
| iplocation clientip
| stats avg(response_time) as avg_rt, count by Country
| where count > 100
| sort -avg_rt

# Session analysis
index=web sourcetype=access_combined
| stats min(_time) as start, max(_time) as end, count as page_views, dc(uri_path) as unique_pages by session_id
| eval duration_min = round((end - start) / 60, 1)
| stats avg(duration_min) as avg_session, avg(page_views) as avg_pages, count as sessions
```

---

## Security Monitoring Searches

### Common Sourcetypes & Indexes

```
index=auth sourcetype=WinEventLog:Security     # Windows Security events
index=auth sourcetype=linux_secure              # Linux auth logs
index=endpoint sourcetype=crowdstrike:event     # CrowdStrike EDR
index=email sourcetype=exchange:*               # Exchange email logs
index=proxy sourcetype=proxy                    # Web proxy logs
index=risk                                      # Splunk ES risk index
```

### Authentication Monitoring

```spl
# Failed logins by user (last 4 hours)
index=auth action=failure earliest=-4h
| stats count as failures, dc(src) as unique_sources, values(src) as sources by user
| where failures > 5
| sort -failures

# Brute force detection
index=auth action=failure earliest=-1h
| stats count as failures by src, user
| where failures > 10
| eval alert_msg = "Brute force: ".failures." failures for ".user." from ".src

# Successful login after failures (brute force success)
index=auth earliest=-1h
| sort _time
| stats list(action) as actions, list(_time) as times, count by src, user
| eval last_action = mvindex(actions, -1)
| eval failure_count = mvcount(mvfilter(match(actions, "failure")))
| where last_action = "success" AND failure_count > 5
| table src, user, failure_count

# After-hours authentication
index=auth action=success
| eval hour = strftime(_time, "%H")
| where hour < 6 OR hour > 22
| stats count by user, src, dest
| sort -count

# Impossible travel (login from geographically distant locations)
index=auth action=success
| iplocation src
| sort user, _time
| streamstats current=f window=1 latest(lat) as prev_lat, latest(lon) as prev_lon, latest(_time) as prev_time by user
| where isnotnull(prev_lat)
| eval distance_km = 6371 * acos(sin(lat*pi()/180)*sin(prev_lat*pi()/180) + cos(lat*pi()/180)*cos(prev_lat*pi()/180)*cos((lon-prev_lon)*pi()/180))
| eval time_diff_hr = (_time - prev_time) / 3600
| eval speed_kmh = if(time_diff_hr > 0, distance_km / time_diff_hr, 0)
| where speed_kmh > 1000
```

### Endpoint Security

```spl
# Suspicious process execution
index=endpoint sourcetype=crowdstrike:event event_type=ProcessExecution
| where process_name IN ("powershell.exe", "cmd.exe", "wscript.exe", "cscript.exe", "mshta.exe")
| stats count by host, user, process_name, command_line
| where count > 0

# New services installed (persistence)
index=auth sourcetype=WinEventLog:System EventCode=7045
| stats count by host, ServiceName, ServiceFileName
| sort -_time

# Large file transfers (data exfiltration)
index=proxy
| stats sum(bytes_out) as total_out by src_user, dest
| where total_out > 104857600
| eval total_mb = round(total_out / 1048576, 1)
| sort -total_out
```

### Risk-Based Alerting

```spl
# Entity risk score aggregation
index=risk
| stats sum(risk_score) as total_risk,
        dc(search_name) as detection_count,
        values(search_name) as detections,
        latest(_time) as last_detection
  by risk_object, risk_object_type
| where total_risk > 100
| sort -total_risk

# Risk trend over time
index=risk
| timechart span=1h sum(risk_score) by risk_object
| where max > 0

# Top risky users
index=risk risk_object_type=user
| stats sum(risk_score) as risk, dc(search_name) as detections by risk_object
| sort -risk
| head 10
```

### Threat Intelligence Correlation

```spl
# IP IOC matching against firewall logs
index=network sourcetype=pan:traffic
| lookup threat_intel_ip ip as dest_ip OUTPUT threat_category, confidence, severity
| where isnotnull(threat_category)
| stats count by dest_ip, threat_category, confidence, severity
| sort -severity, -count

# Domain IOC matching against DNS
index=dns
| lookup threat_intel_domain domain as query OUTPUT threat_category
| where isnotnull(threat_category)
| stats count by query, threat_category, src
| sort -count

# File hash IOC matching
index=endpoint
| lookup threat_intel_hash hash as file_hash OUTPUT threat_name, severity
| where isnotnull(threat_name)
| stats count by host, file_hash, threat_name, severity
```

### Windows Security Event Monitoring

```spl
# Kerberoasting detection (Event 4769 with RC4 encryption)
index=auth sourcetype=WinEventLog:Security EventCode=4769 TicketEncryptionType=0x17
| where ServiceName != "krbtgt"
| stats count by user, ServiceName, src
| where count > 3

# Group membership changes (privilege escalation)
index=auth sourcetype=WinEventLog:Security (EventCode=4728 OR EventCode=4732 OR EventCode=4756)
| stats count by user, MemberName, TargetUserName, GroupName
| eval action_desc = user." added ".MemberName." to ".GroupName

# Account creation monitoring
index=auth sourcetype=WinEventLog:Security EventCode=4720
| stats count by user, TargetUserName, src
| table _time, user, TargetUserName, src
```

---

## Cross-Perspective Patterns

### Data Quality Monitoring

```spl
# Index health check (all perspectives)
| tstats count WHERE index=* BY index, sourcetype, _time span=1h
| timechart span=1h count by sourcetype
| foreach * [eval "<<FIELD>>" = if(isnull('<<FIELD>>'), 0, '<<FIELD>>')]

# Missing data sources
| metadata type=sourcetypes index=main
| eval hours_since = round((now() - recentTime) / 3600, 1)
| where hours_since > 1
| table sourcetype, hours_since, totalCount
| sort -hours_since
```

### Ingestion Latency

```spl
# Check indexing delay
index=* earliest=-15m
| eval latency_sec = _indextime - _time
| stats avg(latency_sec) as avg_delay, perc95(latency_sec) as p95_delay by sourcetype
| where avg_delay > 60
| sort -avg_delay
```

### Summary Indexing for Long-Term Trending

```spl
# Hourly summary for any perspective
index=web sourcetype=access_combined
| timechart span=1h count as requests, avg(response_time) as avg_rt, count(eval(status>=500)) as errors
| collect index=summary source="hourly_web_stats" marker="report=web_hourly"
```
