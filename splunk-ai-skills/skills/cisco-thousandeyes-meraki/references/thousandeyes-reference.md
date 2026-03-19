# ThousandEyes Reference

## Official Documentation

### Product Docs
- [ThousandEyes Documentation](https://docs.thousandeyes.com/)
- [Getting Started with Agents](https://docs.thousandeyes.com/product-documentation/getting-started/getting-started-with-cloud-and-enterprise-agents)
- [Network Tests](https://docs.thousandeyes.com/product-documentation/tests/network-tests)
- [DNS Tests](https://docs.thousandeyes.com/product-documentation/tests/dns-tests)
- [BGP Tests](https://docs.thousandeyes.com/product-documentation/tests/bgp-tests)
- [Web Transaction Tests](https://docs.thousandeyes.com/product-documentation/browser-synthetics/transaction-tests)
- [Path Visualization](https://docs.thousandeyes.com/product-documentation/internet-and-wan-monitoring/path-visualization)
- [Internet Insights](https://docs.thousandeyes.com/product-documentation/internet-insights)
- [Endpoint Agents](https://docs.thousandeyes.com/product-documentation/global-vantage-points/endpoint-agents)
- [Metrics Explained](https://docs.thousandeyes.com/product-documentation/tests/thousandeyes-metrics-what-do-your-results-mean)

### API & Integration
- [API v7 (Cisco DevNet)](https://developer.cisco.com/docs/thousandeyes/)
- [API Getting Started](https://developer.cisco.com/docs/thousandeyes/getting-started/)
- [Splunk App Guide](https://docs.thousandeyes.com/product-documentation/integration-guides/custom-built-integrations/splunk-app)
- [Splunkbase App](https://splunkbase.splunk.com/app/7719)
- [Webhook → Splunk Alerts](https://docs.thousandeyes.com/product-documentation/integration-guides/custom-webhook-examples/splunk-alert-notifs)

## Test Types Deep Dive

### Network Tests

| Test | Protocol | Metrics | Use Case |
|------|----------|---------|----------|
| Agent-to-Server | ICMP or TCP | Latency, loss, jitter, MTU | WAN circuit, ISP, SaaS reachability |
| Agent-to-Agent | Bidirectional | Latency, loss, jitter (both directions) | Site-to-site, VPN tunnel quality |
| Network — Path Trace | ICMP TTL | Hop-by-hop latency, IP, ASN, geo | Routing analysis, bottleneck detection |

### DNS Tests

| Test | What It Does | Detects |
|------|-------------|---------|
| DNS Server | Queries a specific DNS server | Resolution time, availability, SERVFAIL |
| DNS Trace | Traces query through hierarchy | Delegation issues, poisoning, slow authority |
| DNSSEC | Validates DNSSEC chain | Broken DNSSEC, expired signatures |
| DNS+ | Global DNS resolution from cloud | Regional DNS manipulation, hijacking |

### Web Tests

| Test | What It Does | Metrics |
|------|-------------|---------|
| HTTP Server | Tests URL response | Response code, time to first byte, headers |
| Page Load | Full browser page load | DOM load, page load time, waterfall, component count |
| Transaction | Scripted browser actions | Step timing, screenshots, custom assertions |

### BGP Tests

Monitor prefix reachability from 300+ BGP monitors worldwide. Detect route leaks, hijacks, origin changes, AS path changes.

## API v7 Key Endpoints

Base URL: `https://api.thousandeyes.com/v7/`

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/tests` | GET | List all tests |
| `/test-results/{testId}/network` | GET | Network test results |
| `/test-results/{testId}/dns/server` | GET | DNS test results |
| `/test-results/{testId}/web/http-server` | GET | HTTP test results |
| `/test-results/{testId}/web/page-load` | GET | Page load results |
| `/test-results/{testId}/net/path-vis` | GET | Path visualization data |
| `/test-results/{testId}/net/bgp-metrics` | GET | BGP metrics |
| `/alerts` | GET | Active alerts |
| `/agents` | GET | List agents |
| `/endpoint-agents` | GET | Endpoint agents |
| `/internet-insights/outages` | GET | Internet outages |

Auth: OAuth2 Bearer token in header.

## Internet Insights

Global outage detection updated every 5 minutes. Two types:
- **Network Outages**: ISP/transit provider issues affecting reachability
- **Application Outages**: SaaS/CDN service degradation

```spl
# Internet Insights outage tracking
index=thousandeyes sourcetype=thousandeyes:insights
| stats count by outageType, affectedProvider, affectedRegion, startTime
| sort -startTime
```

## Path Visualization Analysis

Path trace uses TTL manipulation (ICMP/SYN) to map each hop with: IP address, reverse DNS, BGP prefix, ASN, geographic location, per-hop latency.

```spl
# Identify high-latency hops
index=thousandeyes test_type="agent-to-server"
| spath input=pathTrace
| where hop_latency > 50
| stats avg(hop_latency) as avg_hop_ms by hop_ip, hop_asn, hop_location
| sort -avg_hop_ms
```

## Endpoint Agent Use Cases

- **WiFi monitoring**: Signal strength, BSSID, channel, interference
- **Browser sessions**: Real user page loads correlated with network path
- **VPN performance**: Split tunnel vs full tunnel comparison
- **Scheduled tests**: Run network tests from user desktops

## ITSI Integration

ThousandEyes data can feed ITSI services as KPIs:
- Service: "Internet Connectivity" → KPI: ThousandEyes availability per test
- Service: "SaaS Application Health" → KPI: HTTP response time, page load
- Service: "DNS Infrastructure" → KPI: DNS resolution time per server
