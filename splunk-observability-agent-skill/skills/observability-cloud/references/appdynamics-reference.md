# Splunk AppDynamics SaaS Reference

## Official Documentation

- [AppDynamics SaaS Overview](https://help.splunk.com/en/appdynamics-saas)
- [Getting Started](https://help.splunk.com/en/appdynamics-saas/get-started)
- [Application Performance Monitoring](https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring)
- [Database Visibility](https://help.splunk.com/en/appdynamics-saas/database-visibility)
- [Infrastructure Visibility](https://help.splunk.com/en/appdynamics-saas/infrastructure-visibility)
- [End User Monitoring](https://help.splunk.com/en/appdynamics-saas/end-user-monitoring)
- [Analytics](https://help.splunk.com/en/appdynamics-saas/analytics)
- [Application Security Monitoring](https://help.splunk.com/en/appdynamics-saas/application-security-monitoring)
- [Extend AppDynamics](https://help.splunk.com/en/appdynamics-saas/extend-splunk-appdynamics)
- [Agent Management](https://help.splunk.com/en/appdynamics-saas/agent-management)
- [Tag Management](https://help.splunk.com/en/appdynamics-saas/tag-management)
- [Licensing](https://help.splunk.com/en/appdynamics-saas/licensing)
- [Accounts & Administration](https://help.splunk.com/en/appdynamics-saas/accounts)

## Platform Architecture

### Core Components

```
Application Agents ─┐
Machine Agent ──────┤
Database Agent ─────┤──→ Controller Tenant (SaaS) ──→ Dashboards / Alerts / Analytics
Analytics Agent ────┤
Browser/Mobile SDK ─┘
```

| Component | Purpose | Deployment Model |
|-----------|---------|-----------------|
| **Controller Tenant** | Central management console — monitoring, configuration, alerting, analytics | SaaS-hosted by Splunk |
| **Application Agents** | Instrument application code, detect business transactions, capture snapshots | One per application tier/node |
| **Machine Agent** | Collect infrastructure metrics (CPU, disk, memory, network) | One per host (DaemonSet for K8s) |
| **Database Agent** | Monitor database performance, queries, wait states | One per database cluster |
| **Analytics Agent** | Process analytics events, log data, custom events | Standalone or co-located |
| **Browser JS Agent** | Capture browser page performance, AJAX, JS errors | Injected into web pages |
| **Mobile SDK** | Capture mobile app performance, crashes, network requests | Bundled with iOS/Android apps |

### Getting Started Sequence

1. Create or access SaaS account via Splunk provisioning
2. Access the Controller Tenant UI (browser-based console)
3. Install appropriate application agents for your technology stack
4. Configure agents with controller host, port, SSL, account name, and access key
5. Validate agent registration in Controller → Applications
6. Set up health rules and alerting policies
7. Configure dashboards and custom reporting

## Application Performance Monitoring (APM)

### Application Model

AppDynamics organizes monitoring around a hierarchy:

| Entity | Description | Example |
|--------|-------------|---------|
| **Application** | Top-level logical grouping | "eCommerce", "Payment-Service" |
| **Tier** | A set of servers performing the same role | "Web-Tier", "API-Tier", "Cart-Service" |
| **Node** | An individual server instance within a tier | "web-server-1", "api-pod-abc123" |
| **Backend** | External system called by monitored tiers | "Oracle-DB", "Redis-Cache", "PaymentGateway-API" |
| **Business Transaction** | A user-initiated workflow through the application | "Login", "Checkout", "Search" |

### Supported Application Agents

| Agent | Languages/Frameworks | Key Capabilities |
|-------|---------------------|-----------------|
| **Java Agent** | Java, Kotlin, Scala (Tomcat, WebLogic, WebSphere, JBoss, Spring Boot, Quarkus) | Auto BT detection, async correlation, thread profiling, memory leak detection |
| **.NET Agent** | C#, VB.NET (IIS, .NET Core, ASP.NET, Azure) | Auto BT detection, .NET compatibility mode, Azure Spring Cloud support |
| **Node.js Agent** | Node.js (Express, Koa, Hapi, Fastify) | Auto BT detection, async tracking, promise correlation |
| **Python Agent** | Python (Django, Flask, FastAPI, WSGI/ASGI) | Auto BT detection, WSGI/ASGI middleware |
| **PHP Agent** | PHP (Apache, Nginx, WordPress, Laravel) | Auto BT detection, request tracing |
| **Go SDK** | Go | Manual instrumentation, BT correlation |
| **C/C++ SDK** | C, C++ | Manual instrumentation, low-overhead |
| **Apache Agent** | Apache HTTP Server | Web server monitoring, upstream correlation |

### Business Transaction Detection

Business transactions are automatically detected at application entry points:

| Entry Point Type | Examples |
|-----------------|----------|
| **HTTP/HTTPS** | Servlet endpoints, ASP.NET routes, Node.js routes, PHP scripts |
| **Web Service** | SOAP endpoints, REST API endpoints |
| **Message Queue** | JMS listener, RabbitMQ consumer, Kafka consumer |
| **Background Task** | Scheduled jobs, cron tasks, worker threads, timer-based tasks |
| **Database** | Stored procedure calls, database triggers |
| **Custom** | User-defined entry points via configuration |

**Configuration options**:
- Custom BT detection rules (match on URL, header, parameter, method signature)
- Naming rules (control how BTs are named from request attributes)
- Exclusion rules (filter out health checks, static resources)
- Splitting rules (separate a single entry point into multiple BTs by parameter)
- Grouping rules (combine related entry points into a single BT)
- Default BT limit: 200 per application — overflow goes to "All Other Traffic"

### Transaction Snapshots

Periodic deep-dive captures of individual transaction executions:

| Snapshot Element | What It Shows |
|-----------------|---------------|
| **Call graph** | Method-level execution path with timing for each method |
| **SQL queries** | Captured queries with bind variables (configurable via data security settings) |
| **HTTP exits** | External service calls with URL, response time, status code |
| **Error details** | Exception type, message, and full stack trace |
| **Data collectors** | Custom business data from method parameters, return values, or HTTP data |
| **Async segments** | Cross-thread and cross-JVM transaction tracking |
| **Thread state** | Thread dumps showing blocked/waiting states |

Snapshot collection frequency is automatic (periodic + slow/error triggers) but can be increased via diagnostic sessions.

### Flow Maps

Visual topology of application components:
- Shows tiers, backends, and call relationships
- Color-coded health status: Green (normal), Yellow (warning), Red (critical)
- Click any component to drill into metrics, BTs, snapshots
- Shows call counts, response times, and error rates on edges

### Health Rules & Baselines

**Baseline types**: Automatic rolling baselines using historical data (hourly, daily, weekly patterns).

| Health Rule Category | Common Conditions |
|---------------------|-------------------|
| **BT Response Time** | > 2x baseline, > absolute threshold (e.g., 3000ms) |
| **BT Error Rate** | > 5% of calls for 3+ consecutive minutes |
| **Slow Calls** | > 10% of BT calls exceeding slow threshold |
| **Overall Application** | Total errors/minute > threshold |
| **Node Health** | JVM heap > 80%, GC time > 10%, thread pool exhaustion |
| **Hardware** | CPU > 90%, memory > 95%, disk I/O saturated |
| **Custom Metric** | Any custom or extension metric with static/baseline threshold |

**Severity levels**: Critical, Warning, Info → mapped to policies → trigger actions.

**Actions**: Email, SMS, HTTP webhook, PagerDuty, Slack, Teams, JIRA ticket, ServiceNow event, custom script.

### Diagnostic Sessions

On-demand deep diagnostics for production troubleshooting:
- Increase snapshot frequency for specific BTs
- Extend call graph depth (capture deeper method calls)
- Enable memory leak detection (object instance tracking)
- Capture thread dumps and lock contention analysis
- Time-limited (auto-expires to prevent overhead)

## Database Visibility

### Supported Databases

| Database | Special Features |
|----------|-----------------|
| **Oracle** (including RAC) | Wait state analysis, execution plans, snapshot correlation, ASH integration |
| **Microsoft SQL Server** (AWS RDS, Azure) | Wait types, blocking sessions, execution plans |
| **MySQL** (Amazon RDS) | Slow query analysis, InnoDB metrics, replication monitoring |
| **PostgreSQL** (including pgvector) | Query analysis, vacuum stats, connection pooling |
| **MongoDB** (Kerberos auth) | Operation analysis, replica set monitoring, index usage |
| **IBM DB2** (LUW, Kerberos) | SQL analysis, bufferpool metrics, lock escalation |
| **Cassandra** (Apache and DSE) | Read/write latency, compaction metrics, node health |
| **Couchbase** | Bucket metrics, N1QL query analysis, XDCR monitoring |
| **SAP HANA** | SQL analysis, memory metrics, row/column store stats |
| **Sybase ASE/IQ** | Query analysis, cache metrics, tempdb monitoring |

### Database Agent Deployment

```bash
# Standalone installation
java -jar db-agent.jar \
  -Ddbagent.name=production-db-agent \
  -Dappdynamics.controller.hostName=<your-controller>.saas.appdynamics.com \
  -Dappdynamics.controller.port=443 \
  -Dappdynamics.controller.ssl.enabled=true \
  -Dappdynamics.agent.accountName=<account-name> \
  -Dappdynamics.agent.accountAccessKey=<access-key>
```

Deployment options: Standalone JAR, container (EKS-compatible), Windows service, Linux systemd.

Supports: SSL/SSH encryption, custom JRE, HashiCorp Vault, Kerberos authentication.

### Database Monitoring Windows

| Window | What It Shows |
|--------|---------------|
| **Dashboard** | Overview metrics, top queries, wait states |
| **Activity** | Real-time and historical query execution |
| **Topology** | Visual map of database connections and dependencies |
| **Live** | Real-time session and query data |
| **Query Analysis** | Top queries by execution time, frequency, wait states |
| **Execution Plans** | Query execution plans with comparison |
| **Blocking Sessions** | Lock holders and waiters |
| **Client/Sessions** | Database connections by application |
| **BT Correlation** | Link database activity to application business transactions |
| **Hardware** | Disk, CPU, memory utilization of database servers |
| **Custom Metrics** | User-defined metrics from database queries |

### Database Permissions

Each database requires specific monitoring user permissions. Key examples:
- **Oracle**: SELECT on V$ views, EXECUTE on DBMS_MONITOR
- **SQL Server**: VIEW SERVER STATE, VIEW DATABASE STATE
- **MySQL**: PROCESS, SELECT on performance_schema
- **PostgreSQL**: pg_monitor role or SELECT on pg_stat views

## Infrastructure Visibility

### Machine Agent

Collects host-level metrics on Linux, Windows, Solaris, and AIX:

| Metric Category | Key Metrics |
|----------------|-------------|
| **CPU** | Utilization %, idle %, I/O wait %, steal %, per-core breakdown |
| **Memory** | Used, free, buffers, cached, swap usage |
| **Disk** | Utilization %, I/O read/write bytes, IOPS, latency |
| **Network** | Bandwidth in/out, packets/sec, errors, TCP connections |
| **Processes** | CPU/memory per process, process count, top processes |

Installation: Standalone JAR with bundled JRE options, configurable via controller-info.xml.

### Server Visibility

- Dashboard with disk/CPU/memory utilization at a glance
- Process monitoring with availability tracking
- Role-based access controls (standard and dynamic roles)
- Health rule configuration per server group
- Tagging for grouping and filtering (AWS, Azure, K8s, ServiceNow CMDB)
- Service availability tracking

### Network Visibility (Requires Additional License)

| Metric | Description |
|--------|-------------|
| **Packet loss** | Percentage of dropped packets |
| **Round-trip time** | Network latency between components |
| **Connection errors** | TCP setup/teardown failures |
| **TCP window size** | Flow control issues |
| **Retransmissions** | Network reliability indicator |

Available on Linux, Windows, AIX. Useful for identifying network-layer issues affecting application performance.

### Kubernetes & Container Monitoring

| Component | Deployment | Purpose |
|-----------|-----------|---------|
| **Cluster Agent** | Helm chart, CLI, or OpenShift OperatorHub | Cluster-level visibility, pod/container metrics, auto-instrumentation |
| **Machine Agent** | DaemonSet | Per-node infrastructure metrics |
| **Docker Visibility** | Machine Agent extension | Container-level metrics (CPU, memory, network per container) |

Auto-instrumentation supported for containerized Java, .NET, Node.js applications.

### GPU Monitoring

- DCGM Exporter and NVIDIA-SMI collector support
- GPU utilization, memory, temperature, power metrics
- Useful for ML/AI workload monitoring

## End User Monitoring (EUM)

### Browser Real User Monitoring (BRUM)

| Feature | Description |
|---------|-------------|
| **Page performance** | Load times, Web Vitals (LCP, FID, CLS), resource timing, navigation timing |
| **AJAX monitoring** | Request/response timing, error rates, correlation to backend BTs |
| **JavaScript errors** | Error capture with source mapping, configurable ignore rules |
| **Session replay** | Visual replay of user sessions |
| **Virtual pages** | SPA (Single Page Application) navigation tracking |
| **GraphQL monitoring** | Operation-level performance tracking |
| **Cookie consent** | GDPR-compliant consent management integration |
| **CSP support** | Content Security Policy compatible injection |

**JavaScript agent injection methods**: Manual script tag, auto-injection via application agent, CDN hosted.

### Mobile Real User Monitoring

| Platform | SDK | Capabilities |
|----------|-----|-------------|
| **iOS** | AppDynamics iOS SDK | Network requests, crashes, UI performance, sessions, custom timers |
| **Android** | AppDynamics Android SDK | Network requests, crashes, ANR detection, sessions, custom events |

### Synthetic Monitoring

- **Browser synthetic**: Scripted browser interactions from public and private locations
- **API synthetic**: HTTP/REST endpoint availability and performance monitoring
- **Private synthetic agent**: Deploy in your own infrastructure (8 deployment options)
- **Credential vault**: Secure storage for test authentication
- **Alerts**: Availability and performance threshold alerting
- **Authentication**: Support for various auth methods in synthetic tests

### Experience Journey Map

Visualize user workflows across browser and mobile:
- Define milestones (e.g., browse → add-to-cart → checkout → payment)
- Traffic segmentation analysis by user attributes
- Conversion funnel visualization
- Custom view creation for different stakeholder perspectives

## Analytics (ADQL)

### Query Language

ADQL (AppDynamics Query Language) — SQL-like syntax for analytics searches:

```sql
-- Find slowest business transactions
SELECT transactionName, avg(responseTime) as avgRT, count(*) as txnCount
FROM transactions
WHERE responseTime > 5000
AND application = 'ecommerce'
SINCE 1 hour ago
GROUP BY transactionName
ORDER BY avgRT DESC
LIMIT 20

-- Analyze error patterns
SELECT transactionName, errorMessage, count(*) as errorCount
FROM transactions
WHERE userExperience = 'ERROR'
SINCE 24 hours ago
GROUP BY transactionName, errorMessage
ORDER BY errorCount DESC

-- Browser analytics
SELECT pageName, avg(domReadyTime), avg(firstByteTime), count(*)
FROM browser_records
WHERE application = 'web-portal'
SINCE 1 hour ago
GROUP BY pageName
```

**Key clauses**: SELECT, FROM, WHERE, GROUP BY, ORDER BY, HAVING, SINCE/UNTIL, LIMIT.
**Operators**: Comparison, logical, REGEXP, math expressions, aggregation functions.

### Analytics Data Sources

| Data Source | What It Contains |
|------------|-----------------|
| **Transaction Analytics** | BT performance data with custom data collector fields |
| **Log Analytics** | Application logs with field extraction and free-text search |
| **Browser Analytics** | Page loads, AJAX requests, JS errors from BRUM |
| **Mobile Analytics** | Mobile app performance, crashes, network requests |
| **Synthetic Analytics** | Synthetic test results and availability data |
| **Connected Devices** | IoT and connected device telemetry |

### Business Journeys

Track multi-step user workflows:
- Define milestones and expected conversion rates
- Set health thresholds per milestone
- Dashboard metrics for journey completion
- Funnel analysis for drop-off identification

### Experience Level Management (XLM)

Monitor compliance against defined service levels:
- Define SLA targets per BT or application
- Exclusion periods for maintenance windows
- Audit trails for SLA violations
- Compliance reporting

## Application Security (Cisco Secure Application)

### Runtime Protection

| Capability | Description |
|-----------|-------------|
| **Vulnerability assessment** | Continuous scanning of code execution for known CVEs |
| **Runtime protection** | Block exploit attempts in real-time |
| **Zero-day detection** | Identify new security gaps post-deployment |
| **Attack monitoring** | Detect and alert on active exploitation attempts |
| **Security observations** | Track security-relevant events and anomalies |

Integrates with APM dashboards via Security Events widget.
Supported with Java, .NET, and Node.js APM agents.
Requires separate Secure Application subscription license.

### Security for Different Teams

| Team | Use Case |
|------|----------|
| **IT Operations** | Real-time security events in existing APM dashboards |
| **AppSec Developers** | Best practice violation insights, vulnerability prioritization |
| **DevOps** | Security integration into CI/CD automation (DevSecOps) |
| **Business** | Reduced risk through constant runtime protection |

## Extensions & Integrations

### REST APIs

| API | Purpose |
|-----|---------|
| **Application Model API** | Query application structure (apps, tiers, nodes, BTs) |
| **Metric & Snapshot API** | Retrieve performance data programmatically |
| **Alert & Respond API** | Manage health rules, schedules, policies, actions |
| **Configuration API** | Import/export application configuration |
| **Database Visibility API** | Database monitoring data access |
| **Analytics Events API** | Query analytics event data |
| **RBAC API** | User and permission management |
| **License API** | Licensing information and usage |
| **Synthetic Monitoring API** | Manage synthetic jobs |
| **Agent Management API** | Agent inventory, deployment, configuration |
| **Anomaly Violation API** | Anomaly detection data |

### Alert & Notification Integrations

| Integration | Type |
|------------|------|
| **Email/SMS** | Built-in notification |
| **HTTP Webhooks** | Custom HTTP-based actions |
| **PagerDuty** | Incident management |
| **Slack** | Channel notifications |
| **Microsoft Teams** | Channel notifications |
| **Atlassian JIRA** | Automatic ticket creation |
| **ServiceNow** | CMDB and Event Management |
| **Splunk Enterprise** | Data forwarding and correlation |

### Community Exchange

Pre-built extensions at [AppDynamics Community Exchange](https://developer.cisco.com/codeexchange/search?q=Appdynamics):
- Custom monitoring extensions for third-party systems
- Platform connectors
- Dashboard templates
- Custom health rule packs

## Administration & Security

### Account Management

- Multi-tenant account structure
- SSO integration support
- Role and permission management
- Audit logging for SaaS deployments
- API access key management

### Data Security

- SQL query literal suppression (mask sensitive query data)
- Sensitive data filtering in snapshots
- SSL/TLS for all agent-to-controller communication
- Agent-to-controller authentication via access keys
- Proxy support for restricted network environments

## Troubleshooting Quick Reference

### Agent Connectivity

1. Verify controller host, port (443 for SaaS), and SSL settings
2. Check account name and access key in agent configuration
3. Ensure firewall allows HTTPS to `<account>.saas.appdynamics.com`
4. Review agent logs at `<agent-install-dir>/logs/`
5. Validate proxy settings if behind corporate proxy
6. Check agent version compatibility with controller version

### Missing Business Transactions

1. Confirm agent is registered: Controller → Applications → your app
2. Check BT detection rules — is the entry point type enabled?
3. Review exclusion rules — is the BT accidentally excluded?
4. Check BT limit (200 default) — is BT in "All Other Traffic"?
5. Verify agent auto-detection configuration for the framework
6. Check agent logs for detection errors

### High Overhead

1. Reduce snapshot collection frequency
2. Limit call graph depth
3. Disable unnecessary data collectors
4. Adjust analytics event volume
5. Review and optimize custom extensions
6. Check for runaway diagnostic sessions (should auto-expire)

### Database Agent Issues

1. Verify JDBC connectivity from agent to database
2. Check database user permissions (platform-specific)
3. Validate controller connectivity and registration
4. Review database agent logs
5. Check for firewall rules blocking database ports

## Unified Observability: AppDynamics + Splunk Observability Cloud

### Comparison

| Aspect | AppDynamics | Splunk Observability Cloud |
|--------|-------------|---------------------------|
| **Instrumentation** | Proprietary agents (deep auto-discovery) | OpenTelemetry-based (open standard) |
| **BT detection** | Automatic business transaction detection | Manual span/trace instrumentation |
| **Database monitoring** | Built-in Database Visibility with dedicated agent | Via OTel database client spans |
| **Infrastructure** | Machine Agent | OTel Collector host metrics |
| **End user** | BRUM + Mobile RUM + Synthetics | Splunk RUM + Synthetics |
| **Analytics** | ADQL, Business Journeys, XLM | SignalFlow, custom dashboards |
| **Security** | Cisco Secure Application (runtime) | N/A (complemented by Splunk SIEM) |
| **Best for** | Deep application diagnostics, legacy/enterprise apps | Cloud-native, microservices, K8s-first |

### When to Use Each

- **AppDynamics**: Legacy enterprise applications, deep BT-level diagnostics, database visibility with dedicated agent, runtime application security, teams already using AppDynamics, need for automatic BT detection
- **Splunk Observability Cloud**: Cloud-native microservices, Kubernetes-first environments, OpenTelemetry standardization, real-time streaming analytics, vendor-neutral instrumentation
- **Both together**: Full-stack unified observability across legacy and modern application estates, with AppDynamics providing deep enterprise app visibility and Observability Cloud providing cloud-native infrastructure and microservices monitoring
