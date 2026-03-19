# DNS & DHCP Monitoring Reference

## Official Documentation Links

### Splunk Add-ons
- [ISC BIND Sourcetypes](https://docs.splunk.com/Documentation/AddOns/released/ISCBIND/Sourcetypes)
- [ISC DHCP Sourcetypes](https://docs.splunk.com/Documentation/AddOns/released/ISCDHCP/Sourcetypes)
- [DHCP Insight App](https://splunkbase.splunk.com/app/1837)
- [CIM Network Resolution Model](https://help.splunk.com/en/data-management/common-information-model/6.3)

### Splunk Security Research
- [Rogue DHCP Server Detection](https://research.splunk.com/network/6e1ada88-7a0d-4ac1-92c6-03d354686079/)
- [DNS Exfiltration Detection](https://www.splunk.com/en_us/blog/security/detect-hunt-dns-exfiltration.html)

## DNS Data Sources

| Source | Sourcetype | Collection |
|--------|-----------|------------|
| ISC BIND | `bind` | File monitor / syslog |
| Windows DNS | `WinEventLog:DNS` | Event Log |
| Infoblox | `infoblox:dns` | Syslog |
| Zeek Passive DNS | `zeek:dns` | Network tap |
| PAN DNS Proxy | `pan:threat` (subtype=dns) | Syslog |

**CIM fields**: query, query_type, src, dest, answer, reply_code, message_type, transport, ttl

## DNS Threat Detection Searches

```spl
# DGA detection (entropy-based)
index=dns
| rex field=query "^(?<subdomain>[^.]+)\."
| where len(subdomain) > 12
| eval unique_chars = len(replace(subdomain, "(.)\1*", "X"))
| eval entropy_score = round(unique_chars * log(len(subdomain), 2) / len(subdomain), 2)
| where entropy_score > 3.2
| stats count as dga_hits, dc(query) as unique_domains by src
| where dga_hits > 20

# DNS tunneling
index=dns query_type IN ("TXT", "NULL", "CNAME", "MX")
| where len(query) > 50
| stats count, avg(len(query)) as avg_len by src
| where count > 30 AND avg_len > 40

# DNS exfiltration
index=dns
| rex field=query "\.(?<parent_domain>[^.]+\.[^.]+)$"
| stats dc(query) as unique_subs by src, parent_domain
| where unique_subs > 100

# NXDOMAIN flood
index=dns reply_code=NXDOMAIN
| timechart span=5m count by src
| where count > 100
```

## DHCP Security Searches

```spl
# Rogue DHCP server detection
index=network sourcetype=cisco:ios facility=DHCP_SNOOPING
| stats count by host, interface, src_mac, vlan

# DHCP starvation attack
index=dhcp action=lease
| bin _time span=5m
| stats count as leases by mac_address, _time
| where leases > 20

# Scope exhaustion
index=dhcp action=lease
| timechart span=1h dc(lease_address) as active_leases by scope

# Unknown devices
index=dhcp action=lease
| lookup asset_inventory mac_address OUTPUT authorized
| where isnull(authorized)

# IP conflict
index=dhcp action=lease
| stats dc(mac_address) as mac_count, values(mac_address) as macs by lease_address
| where mac_count > 1
```
