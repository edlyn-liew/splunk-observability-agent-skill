# Router, Switch & SNMP Monitoring Reference

## Official Documentation Links

### Splunk Data Collection
- [Splunk Connect for Syslog (SC4S)](https://splunk.github.io/splunk-connect-for-syslog/main/)
- [SC4S Getting Started](https://splunk.github.io/splunk-connect-for-syslog/main/gettingstarted/)
- [SC4S Configuration](https://splunk.github.io/splunk-connect-for-syslog/main/configuration/)
- [SC4SNMP](https://splunk.github.io/splunk-connect-for-snmp/)
- [SC4SNMP Poller Config](https://splunk.github.io/splunk-connect-for-snmp/v1.9.0/configuration/poller-configuration/)
- [SC4SNMP Trap Config](https://splunk.github.io/splunk-connect-for-snmp/v1.8.5/configuration/trap-configuration/)
- [SC4SNMP Data Format](https://splunk.github.io/splunk-connect-for-snmp/v1.9.0/configuration/snmp-data-format/)

### Splunk Add-ons
- [Cisco Catalyst Add-on](https://splunkbase.splunk.com/app/1352)
- [Cisco ASA Add-on Data Types](https://docs.splunk.com/Documentation/AddOns/released/CiscoASA/DataTypes)
- [Cisco ISE Data Types](https://docs.splunk.com/Documentation/AddOns/released/CiscoISE/Datatypes)
- [Cisco UCS Sourcetypes](https://docs.splunk.com/Documentation/AddOns/released/CiscoUCS/Sourcetypes)
- [Meraki Sourcetypes](https://docs.splunk.com/Documentation/AddOns/released/Meraki/Sourcetypes)

### Vendor Syslog Configuration
- [Palo Alto — Configure Syslog](https://docs.paloaltonetworks.com/pan-os/11-1/pan-os-admin/monitoring/use-syslog-for-monitoring/configure-syslog-monitoring)
- [Palo Alto — Syslog Field Descriptions](https://docs.paloaltonetworks.com/ngfw/administration/monitoring/use-syslog-for-monitoring/syslog-field-descriptions)
- [Fortinet — Log Settings](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/250999/log-settings-and-targets)
- [Fortinet — Log Checking](https://docs.fortinet.com/document/fortigate/7.6.5/administration-guide/458906/checking-the-logs)
- [Meraki — Syslog Overview](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Syslog_Server_Overview_and_Configuration)
- [Meraki — Syslog Event Types](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Syslog_Event_Types_and_Log_Samples)
- [Juniper — Syslog Config](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/syslog-edit-system.html)
- [Arista — System Event Logging](https://www.arista.com/en/um-eos/eos-system-event-logging)

### SNMP OID References
- [ifInOctets (1.3.6.1.2.1.2.2.1.10)](http://oidref.com/1.3.6.1.2.1.2.2.1.10)
- [ifOutOctets (1.3.6.1.2.1.2.2.1.16)](https://oidref.com/1.3.6.1.2.1.2.2.1.16)
- [OID Reference Browser](http://oidref.com/)

## SNMP OID Quick Reference

### Interface Metrics (IF-MIB, base: 1.3.6.1.2.1.2.2.1)

| .suffix | Object | Description |
|---------|--------|-------------|
| .8 | `ifOperStatus` | 1=up, 2=down |
| .10 | `ifInOctets` | Inbound bytes |
| .14 | `ifInErrors` | Inbound errors |
| .16 | `ifOutOctets` | Outbound bytes |
| .20 | `ifOutErrors` | Outbound errors |

64-bit: `ifHCInOctets` (.31.1.1.1.6), `ifHCOutOctets` (.31.1.1.1.10)

### Cisco-Specific
| OID | Object |
|-----|--------|
| 1.3.6.1.4.1.9.2.1.57.0 | `avgBusy5` (CPU 5-min %) |
| 1.3.6.1.4.1.9.9.48.1.1.1.5 | `ciscoMemoryPoolUsed` |
| 1.3.6.1.4.1.9.9.48.1.1.1.6 | `ciscoMemoryPoolFree` |
| 1.3.6.1.4.1.9.9.13.1.3.1.3 | `ciscoEnvMonTemperatureValue` |

## Critical Cisco Syslog Mnemonics

| Mnemonic | Facility | Meaning |
|----------|----------|---------|
| `UPDOWN` | LINK | Interface state change |
| `ADJCHG` | OSPF | OSPF neighbor change |
| `ADJCHANGE` | BGP | BGP peer state change |
| `RESTART` | SYS | Device restarted |
| `CONFIG_I` | SYS | Config changed |
| `PSECURE_VIOLATION` | PORT_SECURITY | Port security violation |
| `DHCP_SNOOPING_UNTRUST` | DHCP | Rogue DHCP |
| `LOGIN_FAILED` | SEC_LOGIN | Login failure |

## Key Monitoring Searches

```spl
# Interface down
index=network sourcetype=cisco:ios (mnemonic=UPDOWN OR mnemonic=LINEPROTO*) | rex "Interface (?<interface>\S+),? changed state to (?<state>\w+)" | where state="down"

# Bandwidth from SNMP
index=netmetrics sourcetype=sc4snmp:metric | streamstats current=f last('sc4snmp.IF-MIB.ifInOctets') as prev_in, last(_time) as prev_time by host, interface | eval time_delta=_time-prev_time | where time_delta>0 | eval in_mbps=round(('sc4snmp.IF-MIB.ifInOctets'-prev_in)*8/time_delta/1000000,2) | timechart span=5m avg(in_mbps) by interface

# CPU/Memory health
index=netmetrics sourcetype=sc4snmp:metric | eval cpu='sc4snmp.CISCO-PROCESS-MIB.cpmCPUTotal5minRev' | eval mem_used='sc4snmp.CISCO-MEMORY-POOL-MIB.ciscoMemoryPoolUsed' | eval mem_free='sc4snmp.CISCO-MEMORY-POOL-MIB.ciscoMemoryPoolFree' | eval mem_pct=round(mem_used/(mem_used+mem_free)*100,1) | timechart span=5m avg(cpu), avg(mem_pct) by host
```
