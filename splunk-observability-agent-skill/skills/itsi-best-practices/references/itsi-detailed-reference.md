# ITSI Detailed Reference

## Official Documentation Links

### Services & KPIs
- [Create Services](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/create-services/overview-of-creating-services-in-itsi)
- [Create KPI Base Searches](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/create-kpis/create-kpi-base-searches-in-itsi)
- [Monitor KPI Data Drift](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/create-kpis/monitor-kpi-data-drift-in-itsi)
- [Backfill Health Scores](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.18/create-services/backfill-service-health-scores-in-itsi)

### Service Templates
- [Create a Service Template](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/service-templates/create-a-service-template-in-itsi)
- [Create Service from Template](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/service-templates/create-a-service-from-a-service-template-in-itsi)

### Entity Management
- [Create Entities](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/discover-and-integrate-it-components/4.21/create-and-manage-entities/create-a-single-entity-in-itsi)
- [Configure Entity Thresholds](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/discover-and-integrate-it-components/4.21/create-and-manage-entities/configure-entity-thresholds-in-itsi)

### Thresholds
- [Adaptive KPI Thresholds](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/advanced-thresholding/create-adaptive-kpi-thresholds-in-itsi)
- [Time-Based Static Thresholds](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.20/advanced-thresholding/create-time-based-static-kpi-thresholds-in-itsi)

### Glass Tables & Visualization
- [Glass Table Editor](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/glass-tables/overview-of-the-glass-table-editor-in-itsi)
- [Service Analyzer](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.20/service-analyzer/overview-of-the-service-analyzer-in-itsi)
- [Deep Dives](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/deep-dives/overview-of-deep-dives-in-itsi)

### Predictive Analytics
- [Predictive Analytics Overview](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/predictive-analytics/overview-of-predictive-analytics-in-itsi)
- [Train a Predictive Model](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/visualize-and-assess-service-health/4.21/predictive-analytics/train-a-predictive-model-in-itsi)

### Event Analytics
- [Correlation Searches](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/detect-and-act-on-notable-events/4.21/event-correlation/overview-of-correlation-searches-in-itsi)
- [Aggregation Policies](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/detect-and-act-on-notable-events/4.20/event-aggregation/overview-of-aggregation-policies-in-itsi)
- [Notable Events](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/detect-and-act-on-notable-events/4.20/notable-events/overview-of-notable-events-in-itsi)
- [Investigate Episodes](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/detect-and-act-on-notable-events/4.21/episode-review/investigate-episodes-in-itsi)

### REST API & Content Packs
- [ITSI REST API Reference](https://help.splunk.com/en/splunk-it-service-intelligence/splunk-it-service-intelligence/leverage-rest-apis/4.18/itsi-rest-api-reference/itsi-rest-api-reference)
- [Content Packs for ITSI](https://help.splunk.com/en/splunk-it-service-intelligence/content-packs-for-itsi/1.9/overview/content-packs-for-itsi)

## Quick Reference

### Service Hierarchy Pattern
```
Business Service > Application Service > Infrastructure Service > Component
```

### Entity Fields
Identifying (hostname, IP, MAC), Informational (location, owner, OS), Alias (matching fields for KPI searches).

### KPI Metric Functions
count, sum, avg, max, min, stdev, perc90/95/99, median, range, dc

### Threshold Defaults
`Normal: <70, Low: 70-80, Medium: 80-90, High: 90-95, Critical: >=95`

### Episode Lifecycle
New > In Progress > Pending > Resolved > Closed (or Escalated)

### Scaling
< 200 KPIs: single search head · 200-1000: SH cluster · 1000+: dedicated KPI processing
