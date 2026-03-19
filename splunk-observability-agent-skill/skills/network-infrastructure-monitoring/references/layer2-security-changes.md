# Layer 2 Security & Network Change Management Reference

## Official Documentation Links

### Splunk Security Research Detections
- [Port Security Violation Detection](https://research.splunk.com/network/2de3d5b8-a4fa-45c5-8540-6d071c194d24/)
- [ARP Poisoning Detection](https://research.splunk.com/network/b44bebd6-bd39-467b-9321-73971bcd1aac/)
- [Unauthorized Device Detection (DHCP)](https://research.splunk.com/network/dcfd6b40-42f9-469d-a433-2e53f7489ff4/)
- [Rogue DHCP Server Detection](https://research.splunk.com/network/6e1ada88-7a0d-4ac1-92c6-03d354686079/)

### Tools & Add-ons
- [SA-NetOps (MAC Vendor Lookup)](https://github.com/hire-vladimir/SA-NetOps)
- [CIM Data Models](https://help.splunk.com/en/data-management/common-information-model/6.3)

### Vendor Security Feature Docs
- [Palo Alto Syslog Fields](https://docs.paloaltonetworks.com/ngfw/administration/monitoring/use-syslog-for-monitoring/syslog-field-descriptions)
- [Fortinet Log Administration](https://docs.fortinet.com/document/fortigate/7.6.5/administration-guide/458906/checking-the-logs)
- [Arista System Logging](https://www.arista.com/en/um-eos/eos-system-event-logging)

## ARP & Port Security Searches

```spl
# DAI violations
index=network sourcetype=cisco:ios facility=SW_DAI
| stats count by host, interface, src_mac, src_ip, vlan

# Port security violations
index=network sourcetype=cisco:ios facility=PORT_SECURITY mnemonic=PSECURE_VIOLATION
| stats count by host, interface, src_mac, vlan

# MAC flapping
index=network sourcetype=cisco:ios mnemonic=MAC_MOVE_NOTIFICATION
| rex "Host (?<mac>\S+) in vlan (?<vlan>\d+) is flapping between port (?<port1>\S+) and port (?<port2>\S+)"
| stats count by host, mac, vlan | where count > 3

# Unauthorized devices
index=dhcp action=lease
| lookup authorized_devices mac_address OUTPUT authorized
| where isnull(authorized)
```

## 802.1X / RADIUS Searches

```spl
# 802.1X failures
index=network sourcetype=cisco:ios facility=DOT1X mnemonic=FAIL
| stats count by host, interface, mac_address | where count > 3

# RADIUS server status
index=network sourcetype=cisco:ios facility=RADIUS
| eval result = case(mnemonic="DEAD","down", mnemonic="ALIVE","up", true(),mnemonic)
| stats count by host, radius_server, result
```

## Change Management Searches

```spl
# Config changes
index=network sourcetype=cisco:ios mnemonic=CONFIG_I
| rex "by (?<user>\S+).*\((?<source_ip>\S+)\)"
| table _time, host, user, source_ip

# Changes outside maintenance window
index=network sourcetype=cisco:ios mnemonic=CONFIG_I
| eval hour=tonumber(strftime(_time,"%H")), dow=strftime(_time,"%A")
| where (hour<6 OR hour>22) OR dow IN ("Saturday","Sunday")

# Telnet access audit
index=network sourcetype=cisco:ios facility=SEC_LOGIN
| eval protocol=if(like(_raw,"%SSH%"),"SSH","Telnet")
| where protocol="Telnet"
```

## Alert Priority

| Event | Priority |
|-------|----------|
| Port security / ARP poisoning / Rogue AP | Critical — immediate |
| MAC flapping / Config outside window | High — 15 min |
| 802.1X failures / RADIUS down | High — 15 min |
| Telnet access / New unknown device | Medium — 1 hour |
| Software non-compliance | Medium — 24 hours |
