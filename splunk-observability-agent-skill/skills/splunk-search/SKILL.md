---
name: splunk-search
description: >
  Best practices for writing effective, optimized Splunk searches using SPL (Search Processing
  Language). Use when the user asks to "write a Splunk search", "optimize SPL", "create a
  Splunk query", "improve search performance", "build a dashboard search", "use stats/eval/where",
  "extract fields with rex", "create a tstats search", "write a correlation search", "build
  a data model", "tune Splunk search performance", "debug slow searches", "use subsearches",
  "create alerts", "write macros", "detect brute force attacks", "monitor network bandwidth",
  "find error rate spikes", "write firewall searches", "analyze DNS logs", "monitor authentication",
  "detect lateral movement", "build RED method dashboards", or needs guidance on SPL commands,
  search patterns, field extraction, data model acceleration, Splunk search architecture,
  or monitoring-specific search patterns for network, security, or application use cases.
metadata:
  version: "0.1.0"
---

# Splunk Search Best Practices

Write effective, optimized SPL (Search Processing Language) queries. This guide covers command fundamentals, optimization techniques, common patterns, and anti-patterns.

## Search Optimization Fundamentals

### Golden Rules

1. **Filter early, filter often** — restrict data before the first pipe
2. **Be specific with index, source, sourcetype** — always specify when known
3. **Avoid wildcards** where possible — they prevent index-time optimization
4. **Place non-streaming commands last** — `stats`, `sort`, `dedup` require full result sets
5. **Use fields command** to limit extracted fields
6. **Disable field discovery** in Search Mode when unnecessary

### Command Pipeline Order

Optimal ordering for performance:

```
index=main sourcetype=access_combined status=500
| fields host, uri, status, clientip
| where len(uri) > 0
| stats count by host, uri
| sort -count
| head 20
```

Principle: streaming commands first (where, eval, rename, fields), then non-streaming (stats, sort, dedup, head).

## Essential SPL Commands

### Filtering Commands

```spl
| search status=404                    # Filter by field value
| where status >= 400 AND status < 500 # Conditional filtering with expressions
| dedup host                           # Remove duplicate events
| head 100                             # First N results
| tail 50                              # Last N results
```

### Statistical Commands

```spl
| stats count, avg(response_time), max(bytes) by host
| stats dc(clientip) as unique_visitors by sourcetype
| chart count over status by host
| timechart span=1h count by sourcetype
| eventstats avg(duration) as avg_duration   # Adds stats without collapsing rows
| streamstats count as running_count         # Running/cumulative stats
```

**stats vs chart vs timechart**:
- `stats`: Pure aggregation, no time axis
- `chart`: Aggregation with any field as X-axis
- `timechart`: Aggregation with `_time` as X-axis (always)

### Eval Command

```spl
| eval duration_sec = duration / 1000
| eval status_category = case(status<300, "success", status<400, "redirect", status<500, "client_error", true(), "server_error")
| eval is_slow = if(response_time > 5000, "yes", "no")
| eval domain = upper(host) . ":" . port
```

Common eval functions: `if()`, `case()`, `coalesce()`, `len()`, `lower()`, `upper()`, `split()`, `mvindex()`, `tonumber()`, `tostring()`, `strftime()`, `strptime()`, `relative_time()`, `now()`, `null()`, `isnull()`, `isnotnull()`, `cidrmatch()`, `match()`, `replace()`.

### Field Extraction

```spl
| rex field=_raw "user=(?<username>\w+)"                    # Named capture group
| rex field=uri "\/api\/v\d+\/(?<endpoint>[^\/\?]+)"        # Extract from URI
| rex mode=sed field=message "s/password=[^&]*/password=REDACTED/g"  # Sed mode
| spath input=json_field output=value path=data.items{}.name # JSON/XML extraction
```

### Lookups

```spl
| lookup geo_lookup ip as clientip OUTPUT city, country     # Automatic lookup
| inputlookup my_reference_table.csv                        # Load CSV as events
| outputlookup results_cache.csv                            # Write results to CSV
```

### Subsearches

```spl
index=main [search index=alerts severity=critical | fields host | format]
```

**Warning**: Subsearches have a 60-second timeout and 10,000 result limit by default. Prefer `join` or `stats` for large datasets.

### Join and Append

```spl
| join type=left host [search index=cmdb | fields host, owner, team]
| append [search index=yesterday sourcetype=web]
```

**Prefer stats over join** — `stats` scales better:
```spl
index=main OR index=cmdb
| stats values(owner) as owner, values(team) as team, count by host
```

### Transaction

```spl
| transaction session_id maxspan=30m maxpause=5m
| where duration > 300
```

**Prefer stats over transaction** when possible — `stats` is significantly faster:
```spl
| stats min(_time) as start, max(_time) as end, count, values(action) as actions by session_id
| eval duration = end - start
```

## Performance Optimization

### Use the TERM Directive

```spl
index=firewall TERM(192.168.1.100)    # Treats IP as single indexed term
TERM(error_code=E5001)                 # Exact term lookup in index
```

### Use tstats for Acceleration

```spl
| tstats count WHERE index=main sourcetype=syslog BY host, sourcetype
| tstats summariesonly=true count FROM datamodel=Web BY Web.src, Web.dest
```

`tstats` queries indexed fields and data model summaries — orders of magnitude faster than raw search.

### Data Model Acceleration

Enable acceleration on data models for fast `tstats` queries. Configure acceleration time range based on query patterns (e.g., 7 days for operational, 90 days for compliance).

### Avoid Anti-Patterns

| Anti-Pattern | Better Alternative |
|---|---|
| `NOT field=value` | Specify what you want, not what you don't |
| `*` as the only search term | Always specify index/sourcetype |
| `| sort` on millions of events | Filter first, sort last |
| `| join` on large datasets | Use `stats` with combined index queries |
| `| transaction` on high-cardinality fields | Use `stats` with min/max `_time` |
| Field extraction with `rex` on every event | Use indexed field extraction or `spath` |
| Unbounded time ranges | Always specify earliest/latest |

### Search Job Management

- Use `| head` during development to limit results
- Set appropriate time ranges — shorter = faster
- Schedule heavy searches during off-peak hours
- Use summary indexing for recurring expensive searches

## Common Patterns

### Error Rate Calculation

```spl
index=web sourcetype=access_combined
| eval is_error = if(status >= 500, 1, 0)
| timechart span=5m avg(is_error) as error_rate
| eval error_rate = round(error_rate * 100, 2)
```

### Top Talkers

```spl
index=firewall sourcetype=firewall_traffic
| stats sum(bytes) as total_bytes by src_ip
| sort -total_bytes
| head 10
| eval total_gb = round(total_bytes/1073741824, 2)
```

### Anomaly Detection (Simple)

```spl
index=web sourcetype=access_combined
| timechart span=1h count as requests
| predict requests as predicted_requests algorithm=LLP5
| eval deviation = abs(requests - predicted_requests)
| where deviation > predicted_requests * 0.5
```

### Multi-Value Field Handling

```spl
| makemv delim="," categories
| mvexpand categories
| stats count by categories
```

## Alerting Best Practices

1. Use the most specific and efficient search possible
2. Set appropriate time windows (avoid scanning more data than needed)
3. Use throttling to prevent alert storms
4. Test alert searches manually before scheduling
5. Use `| where` instead of `| search` for post-filter conditions

## Knowledge Objects

- **Event types**: Categorize events with saved search criteria
- **Tags**: Label event types, hosts, sources with human-readable names
- **Field aliases**: Map field names across different sourcetypes
- **Calculated fields**: Auto-compute fields using eval expressions
- **Macros**: Reusable search snippets with optional arguments

For detailed references, read:
- `references/spl-command-reference.md` — Full command, function, and regex reference
- `references/search-by-perspective.md` — Production-ready searches for network engineering, network security, application monitoring, and security monitoring
