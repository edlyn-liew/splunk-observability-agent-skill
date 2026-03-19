# WAN, Internet, Wireless & SD-WAN Reference

## Official Documentation Links

### Splunk Add-ons & Apps
- [Cisco Meraki Add-on](https://splunkbase.splunk.com/app/5580)
- [Cisco Meraki Integration Guide](https://documentation.meraki.com/General_Administration/Cross-Platform_Content/Cisco_Meraki_Add-on_for_Splunk)
- [VeloCloud SD-WAN App](https://splunkbase.splunk.com/app/6252)
- [Technology Add-on for NetFlow](https://splunkbase.splunk.com/app/1838)
- [NetFlow Logic Integration](https://docs.netflowlogic.com/integrations-and-apps/integrations-with-splunk/)
- [Palo Alto Networks Add-on](https://splunk.github.io/splunk-add-on-for-palo-alto-networks/)
- [PAN CIM Mapping](https://splunk.github.io/splunk-add-on-for-palo-alto-networks/Cimmapping/)
- [PAN Firewall/Panorama Config](https://splunk.github.io/splunk-add-on-for-palo-alto-networks/FirewallsPanorama/)

### Vendor Wireless/SD-WAN Docs
- [Meraki Syslog Event Types](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Syslog_Event_Types_and_Log_Samples)
- [Juniper Syslog Config](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/syslog-edit-system.html)
- [Arista Event Logging](https://www.arista.com/en/um-eos/eos-logging-of-event-notifications)
- [ISP Monitoring Overview](https://www.splunk.com/en_us/blog/learn/internet-isp-monitoring.html)

## WAN Circuit Searches

```spl
# Circuit health (latency + jitter + loss)
index=infrastructure sourcetype=ping
| stats avg(round_trip_time) as latency, stdev(round_trip_time) as jitter, count(eval(status="timeout")) as lost, count as total by dest
| eval loss_pct = round(lost/total*100, 2)
| eval health = case(loss_pct>2 OR latency>150, "Critical", loss_pct>0.5 OR latency>80, "Warning", true(), "Healthy")

# Monthly SLA
index=infrastructure sourcetype=ping earliest=-30d@d
| eval is_up = if(status="success", 1, 0)
| stats count as checks, sum(is_up) as ok by dest
| eval availability = round(ok/checks*100, 4)
| eval sla_status = if(availability >= 99.9, "MET", "BREACH")
```

## Routing Protocol Searches

```spl
# BGP state changes
index=network sourcetype=cisco:ios facility=BGP mnemonic=ADJCHANGE
| rex "neighbor (?<peer_ip>\S+).*state\s+(?<state>\w+)"
| where state != "Established"

# OSPF adjacency changes
index=network sourcetype=cisco:ios facility=OSPF mnemonic=ADJCHG
| rex "Nbr (?<neighbor>\S+) on (?<interface>\S+).*to (?<state>\w+)"
| where state != "FULL"
```

## NetFlow Analysis

```spl
# Top talkers
index=netflow earliest=-1h | stats sum(bytes) as total by src_ip | eval gb=round(total/1073741824,2) | sort -total | head 20

# Protocol distribution
index=netflow | stats sum(bytes) as total by protocol | eventstats sum(total) as grand | eval pct=round(total/grand*100,1) | sort -total
```

## Wireless Searches

```spl
# AP status
index=wireless sourcetype=cisco:wlc | stats latest(ap_status) as status, latest(client_count) as clients by ap_name

# Rogue AP detection
index=wireless event_type=rogue* | stats count by rogue_mac, rogue_ssid, detecting_ap | lookup authorized_aps mac as rogue_mac OUTPUT authorized | where isnull(authorized)
```

## Reference Tables

### SLA Targets
| Level | Annual Downtime | Monthly |
|-------|----------------|---------|
| 99.0% | 3d 15h | 7h 18m |
| 99.9% | 8h 46m | 43.8m |
| 99.99% | 52.6m | 4.38m |

### VoIP Quality
| Metric | Good | Poor |
|--------|------|------|
| Latency | < 80ms | > 150ms |
| Jitter | < 10ms | > 30ms |
| Loss | < 0.5% | > 1% |
| MOS | > 4.0 | < 3.5 |
