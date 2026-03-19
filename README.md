<p align="center">
  <img src="https://www.splunk.com/content/dam/splunk2/images/Planet-workspace-workspace.svg" alt="Splunk" width="80" />
</p>

<h1 align="center">Splunk AI Skills</h1>

<p align="center">
  <strong>Claude plugin for Splunk — expert guidance for monitoring, observability, and app development</strong>
</p>

<p align="center">
  <a href="#installation">Installation</a> ·
  <a href="#skills">Skills</a> ·
  <a href="#usage">Usage</a> ·
  <a href="#examples">Examples</a>
</p>

---

Splunk AI Skills turns Claude into a Splunk Infrastructure expert. Get instant guidance on SPL queries, OpenTelemetry deployments, ITSI service modeling, AWS monitoring, network infrastructure, and more — all without leaving your conversation.

**8 specialized skills** covering the full Splunk ecosystem, from writing your first search to architecting enterprise observability.

## Installation


### Claude Code (CLI)

```bash
claude plugin install splunk-ai-skills
```

### Manual Installation

```bash
# Clone to your Claude skills directory
git clone https://github.com/tufttea/splunk-ai-skills.git ~/.claude/skills/splunk-ai-skills
```

```
claude --plugin-dir ./splunk-ai-skills
```

Then restart Claude.

## Skills

| Skill | What You Can Do |
|-------|-----------------|
| **SPL Search** | Write optimized Splunk queries, tstats acceleration, field extraction, monitoring patterns |
| **UCC App Development** | Build Splunk Technology Add-ons with UCC Generator, CIM compliance, modular inputs |
| **ITSI Best Practices** | Design services & KPIs, glass tables, predictive analytics, correlation searches |
| **Network Infrastructure** | Monitor DNS/DHCP, routers, switches, SNMP, NetFlow, wireless, Layer 2 security |
| **OTel Infrastructure** | Deploy OpenTelemetry collectors, configure pipelines, host metrics, log forwarding |
| **Observability Cloud** | Set up APM tracing, Kubernetes monitoring, service maps, RUM, RED metrics |
| **AWS Monitoring** | CloudWatch, VPC Flow Logs, Lambda, EKS, RDS, CloudTrail, GuardDuty |
| **Cisco ThousandEyes & Meraki** | Synthetic monitoring, BGP visibility, Meraki IDS/IPS, Air Marshal, SD-WAN |

## Usage

Once installed, skills activate automatically when you ask Splunk-related questions. No setup, no API keys, no configuration required.

### Example Prompts

```
Write an SPL query to detect brute force login attempts
```

```
Help me deploy an OpenTelemetry collector on Kubernetes
```

```
How do I monitor AWS Lambda functions in Splunk?
```

```
Create an ITSI service for our payment processing pipeline
```

```
Set up Meraki wireless monitoring with syslog in Splunk
```

```
Build a Splunk TA for collecting data from a REST API
```

## Examples

**SPL Search** — Claude writes optimized, production-ready queries:

```spl
| tstats count where index=security sourcetype=linux_secure "failed password"
  by _time, src_ip, user span=5m
| where count > 10
| sort -count
```

**OTel Deployment** — Claude generates full collector configurations:

```yaml
receivers:
  hostmetrics:
    collection_interval: 10s
    scrapers:
      cpu:
      memory:
      disk:
      network:

exporters:
  splunk_hec:
    token: "${SPLUNK_HEC_TOKEN}"
    endpoint: "https://splunk:8088/services/collector"

service:
  pipelines:
    metrics:
      receivers: [hostmetrics]
      exporters: [splunk_hec]
```

**ITSI** — Claude helps model services with proper KPI hierarchies, threshold configuration, and correlation search patterns.

**AWS** — Claude walks you through CloudWatch metric streams, VPC Flow Log analysis, and multi-account monitoring architectures.

## How It Works

Each skill uses a dual-layer architecture:

1. **Skill layer** (`SKILL.md`) — Quick-start patterns, decision trees, and common workflows
2. **Reference layer** (`references/*.md`) — Deep technical documentation for complex scenarios

Claude reads the relevant skill file when it detects a Splunk topic in your question, giving you expert-level answers grounded in Splunk best practices.

## Plugin Structure

```
splunk-ai-skills/
├── .claude-plugin/
│   └── plugin.json
├── README.md
└── skills/
    ├── splunk-search/
    ├── ucc-app-development/
    ├── itsi-best-practices/
    ├── network-infrastructure-monitoring/
    ├── otel-infrastructure/
    ├── observability-cloud/
    ├── aws-infrastructure-monitoring/
    └── cisco-thousandeyes-meraki/
```

## Requirements

- Claude Desktop or Claude Code
- No API keys or external dependencies
- Works on macOS, Linux, and Windows

## Troubleshooting

**Skills not activating?**

1. Verify the plugin is installed: check `~/.claude/skills/splunk-ai-skills/` exists
2. Ensure `.claude-plugin/plugin.json` is present
3. Restart Claude Desktop or your Claude Code session

**Getting generic answers?**

Be specific with Splunk terminology in your prompts. Mentioning keywords like "SPL", "tstats", "ITSI", "OTel", or specific technologies helps Claude match the right skill.

## Contributing

Built by [Tufttea Studios](https://github.com/tufttea). For issues or feature requests, open an issue in the repository.

## License

MIT

---

<p align="center">
  <sub>v0.6.0 · No API keys required · Works offline</sub>
</p>
