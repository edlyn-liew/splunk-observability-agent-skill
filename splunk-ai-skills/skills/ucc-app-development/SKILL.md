---
name: ucc-app-development
description: >
  Guide for building Splunk Technology Add-ons using the UCC (Universal Configuration Console)
  Generator framework. Use when the user asks to "create a Splunk app", "build a Splunk add-on",
  "configure globalConfig.json", "set up UCC inputs", "create modular inputs", "build a TA",
  "scaffold a Splunk technology add-on", "add OAuth to a Splunk app", "create alert actions",
  "build a network monitoring add-on", "create a security data connector", "build an APM add-on",
  or needs guidance on UCC framework best practices, entity types, validators, or the
  ucc-gen build process for any monitoring perspective (network, security, application).
metadata:
  version: "0.2.0"
---

# UCC Generator App Development

Build professional Splunk Technology Add-ons using the Universal Configuration Console (UCC) framework. UCC auto-generates UI components, REST handlers, modular inputs, monitoring dashboards, and OpenAPI documentation from a single `globalConfig.json` configuration file.

## Core Workflow

### 1. Initialize

```bash
ucc-gen init \
  --addon-name "my_addon_for_splunk" \
  --addon-display-name "My Add-on for Splunk" \
  --addon-input-name my_input
```

### 2. Build

```bash
ucc-gen build --source my_addon_for_splunk/package --ta-version 1.0.0
```

### 3. Package

```bash
ucc-gen package --path output/my_addon_for_splunk
```

Output: a `.tar.gz` ready for installation on any Splunk instance.

## Prerequisites

- Python 3.9+
- Git
- Virtual environment: `python3 -m venv .venv && source .venv/bin/activate`
- Install: `pip install splunk-add-on-ucc-framework`

## globalConfig.json Structure

The central configuration file with three main sections:

```json
{
  "inputs": [],
  "configuration": {
    "tabs": []
  },
  "dashboard": {
    "settings": {}
  }
}
```

### Inputs Section

Define modular inputs with entity fields:

```json
{
  "inputs": [{
    "name": "my_input",
    "title": "My Input",
    "description": "Collects data from API",
    "use_single_instance": false,
    "entity": [
      {
        "type": "text",
        "field": "name",
        "label": "Name",
        "help": "Unique input name",
        "required": true,
        "validators": [{"type": "string", "minLength": 1, "maxLength": 50}]
      },
      {
        "type": "text",
        "field": "interval",
        "label": "Interval",
        "help": "Collection interval in seconds",
        "required": true,
        "defaultValue": "300"
      },
      {
        "type": "singleSelect",
        "field": "index",
        "label": "Index",
        "required": true,
        "defaultValue": "default",
        "options": {
          "endpointUrl": "data/indexes",
          "denyList": "^_.*$",
          "createSearchChoice": true
        }
      }
    ]
  }]
}
```

### Configuration Tabs

Built-in tab types: Account, Proxy, Logging. Custom tabs also supported.

```json
{
  "configuration": {
    "tabs": [
      {
        "name": "account",
        "title": "Account",
        "table": {
          "header": [{"label": "Name", "field": "name"}],
          "actions": ["edit", "delete"]
        },
        "entity": [
          {"type": "text", "field": "name", "label": "Account Name", "required": true},
          {"type": "text", "field": "api_key", "label": "API Key", "required": true, "encrypted": true}
        ]
      }
    ]
  }
}
```

## Entity Component Types

| Type | Description | Key Properties |
|------|-------------|----------------|
| `text` | Single-line input | `validators`, `placeholder`, `encrypted` |
| `textarea` | Multi-line input | `rowsMin`, `rowsMax` |
| `singleSelect` | Dropdown | `options`, `createSearchChoice` |
| `multiSelect` | Multi-select | `options`, `delimiter` |
| `checkbox` | Single checkbox | `defaultValue` |
| `checkboxGroup` | Grouped checkboxes | `options` |
| `checkboxTree` | Hierarchical checkboxes | `expandable`, `expanded` |
| `radioButton` | Radio selection | `options` |
| `custom` | Custom React component | `src` path to JS module |

## Validators

```json
{"type": "string", "minLength": 1, "maxLength": 200}
{"type": "regex", "pattern": "^[a-zA-Z0-9_-]+$", "errorMsg": "Invalid characters"}
{"type": "number", "range": [1, 65535]}
```

Save validator for form-level validation:
```json
{"saveValidator": "function(dataDict) { if (!dataDict.url) return 'URL is required'; }"}
```

## OAuth Configuration

Two grant types: authorization_code (interactive) and client_credentials (server-to-server):

```json
{
  "type": "oauth",
  "field": "oauth_config",
  "options": {
    "client_id": "",
    "client_secret": "",
    "redirect_url": "http://localhost:8000/...",
    "scope": "read write",
    "endpoint": "https://auth.example.com/authorize",
    "endpoint_token": "https://auth.example.com/token",
    "oauth_popup_width": 600,
    "oauth_popup_height": 600,
    "oauth_timeout": 180
  }
}
```

## Monitoring Perspective Use Cases

When building a TA, always consider which monitoring perspective it serves. This determines CIM data model compliance, sourcetype naming, and field mappings.

### Network Engineering TAs

Common patterns for network device TAs (firewalls, switches, load balancers):

- **Sourcetypes**: Use vendor-prefixed naming (e.g., `pan:traffic`, `cisco:asa`, `f5:bigip_ltm`)
- **CIM compliance**: Map to `Network_Traffic` data model (src, dest, action, transport, bytes, packets)
- **Collection methods**: Syslog (UDP/TCP), SNMP traps, REST API polling, NetFlow/sFlow/IPFIX
- **Key fields**: `src_ip`, `dest_ip`, `src_port`, `dest_port`, `action` (allowed/blocked), `bytes_in`, `bytes_out`, `protocol`, `rule`

### Network Security TAs

For IDS/IPS, DNS security, threat detection appliances:

- **Sourcetypes**: `ids`, `ips`, `suricata:eve`, `zeek:*`
- **CIM compliance**: Map to `Intrusion_Detection` (signature, severity, category) and `Network_Resolution` (DNS)
- **Key fields**: `signature`, `severity`, `category`, `src`, `dest`, `threat_name`, `action`

### Application Monitoring TAs

For web servers, APM agents, application log collectors:

- **Sourcetypes**: `access_combined`, `nginx_access`, `apache_access`, `otel:traces`, `otel:metrics`
- **CIM compliance**: Map to `Web` data model (status, url, http_method, response_time)
- **Key fields**: `status`, `uri`, `method`, `response_time`, `bytes`, `user_agent`, `trace_id`, `span_id`

### Security Monitoring TAs

For identity, endpoint, email, and compliance sources:

- **Sourcetypes**: `WinEventLog:Security`, `linux_secure`, `crowdstrike:event`
- **CIM compliance**: Map to `Authentication` (action, user, src, dest, app) and `Change` models
- **Key fields**: `user`, `src`, `dest`, `action`, `result`, `EventCode`, `signature`

## Best Practices

1. **Always map to CIM** — ensure your TA populates CIM data model fields via props.conf/transforms.conf
2. **Use built-in tabs** (account, proxy, logging) before creating custom ones
3. **Set `use_single_instance: false`** for proper scheduling of modular inputs
4. **Pin dependency versions** in `package/lib/requirements.txt` (e.g., `urllib3 < 2`)
5. **Never hardcode credentials** — use encrypted fields and OAuth
6. **Provide descriptive help text** for every entity field
7. **Set sensible defaults** to reduce user configuration effort
8. **Validate inputs** with string, regex, and number validators
9. **Use vendor-prefixed sourcetypes** (e.g., `vendor:product:type`)
10. **Test with multiple Splunk versions** before publishing
11. **Include CIM field mapping documentation** in your app's README
12. **Use the monitoring dashboard** generated by UCC for health checks

## Debugging

Search `_internal` for errors:
```
index=_internal source=*splunkd* ERROR
index=_internal source=*splunkd* stderr
```

For detailed reference on globalConfig schema, entity types, advanced patterns, and perspective-specific examples, read:
- `references/ucc-detailed-reference.md` — Full globalConfig schema and advanced UCC patterns
- `references/monitoring-perspectives.md` — CIM mappings, sourcetypes, and field references by perspective
